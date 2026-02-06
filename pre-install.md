# Windows
This project is mainly to normal users that want to create their homelab. This was build and tested for this scenario. Not for production scenarios neither for enterprise ones.
For those, I recommend using proxmox or any other kind of server OS that is specific to manage VMs.

## Pre-ansible configurations on Localhost
pipx install ansible
pipx inject ansible pywinrm


## Pre-ansible configurations on Windows Host
### Configure WinRM on Windows host (desktop version)

#### Kinds of WinRM connection
- Kerberos:

✓ Most secure (industry standard for enterprise)
✓ No password in plaintext
✓ Mutual authentication
✗ Requires Active Directory domain
✗ Complex setup (Kerberos server, tickets, etc.)
✗ Overkill for home lab/small setup

- NTLM:

✓ Good security over HTTPS
✓ Works without Active Directory
✓ Simple to set up
✗ Less secure than Kerberos
✗ Password in plaintext in Ansible inventory

- Certificate:

✓ Very secure
✓ No password needed
✗ Complex certificate management
✗ Certificate validation hassles


## Before you start

### WinRM important requirement
Your network connection is set to Public, which blocks WinRM. Change it to Private:
Option 1: Via Settings (easiest)

Open Settings → Network & internet → Advanced network settings
Look for your Ethernet connection
Click on it
Scroll down to "Network profile"
Change from "Public" to "Private"

If you desktop is using wifi:
Open Settings → Network & internet → Wifi
Scroll down to "Network profile type"
Change from "Public" to "Private"

### On mac, I had to set environment variable to disable macOS proxy detection 
```shell
export no_proxy='*'
export NO_PROXY='*'
```

## WinRM listener with HTTPS and self-signed certificate
The recommendations for WinRM would be to use Kerberos auth over HTTP if in a domain environment or Basic/NTLM over HTTPS for local accounts. CredSSP should only be used when absolutely necessary as it can be a security risk due to its use of unconstrained delegation.

#### Create a user on windows host (optional)
New-LocalUser -Name "ansible" -Password (ConvertTo-SecureString "12345password*" -AsPlainText -Force) -FullName "Ansible User"

#### Add to Administrators group (optional)

Add-LocalGroupMember -Group "Administrators" -Member "ansible"

### Configure winrm on windows host
```powershell
## Activate WinRM
winrm quickconfig

# Create self signed certificate
$certParams = @{
    CertStoreLocation = 'Cert:\LocalMachine\My'
    DnsName           = $env:COMPUTERNAME
    NotAfter          = (Get-Date).AddYears(1)
    Provider          = 'Microsoft Software Key Storage Provider'
    Subject           = "CN=$env:COMPUTERNAME"
}
$cert = New-SelfSignedCertificate @certParams

# Create HTTPS listener
$httpsParams = @{
    Path                  = 'WSMan:\localhost\Listener'
    Address               = '*'
    CertificateThumbprint = $cert.Thumbprint
    Enabled               = $true
    Port                  = 5986
    Transport             = 'HTTPS'
    Force                 = $true
}
New-Item @httpsParams

# Opens port 5986 for all profiles
$firewallParams = @{
    Action      = 'Allow'
    Description = 'Inbound rule for Windows Remote Management via WS-Management. [TCP 5986]'
    Direction   = 'Inbound'
    DisplayName = 'Windows Remote Management (HTTPS-In)'
    LocalPort   = 5986
    Profile     = 'Any'
    Protocol    = 'TCP'
}
New-NetFirewallRule @firewallParams

# This tells WinRM: "Accept NTLM as a valid authentication method"
Set-Item -Path "WSMan:\localhost\Service\Auth\NTLM" -Value $true
# This tells WinRM: "Accept credential-based authentication (username/password)"
Set-Item -Path "WSMan:\localhost\Service\Auth\Basic" -Value $true 
```

### Verify if Negotiate = true if you will use NTLM
```powershell 
winrm set winrm/config/service/auth '@{Negotiate="true"}'
# output: Auth
#    Basic = true
#    Kerberos = true
#    Negotiate = true
#    Certificate = false
#    CredSSP = false
#    CbtHardeningLevel = Relaxed
```

### Check ansible https port is listening in 5986
```powershell 
Get-Item WSMan:\localhost\Listener
# output: netstat -ano | findstr 5986
```

### Try to connect with your client host (where you will run ansible)

I created a host.ini for test:

#### On my mac
```ini
[windows]
192.168.200.35

[windows:vars]
ansible_connection: winrm
ansible_winrm_transport: ntlm
ansible_port: 5986
ansible_winrm_server_cert_validation: ignore
```

#### On my mac
```shell

ansible windows -i hosts.ini -m win_ping -vvv --ask-vault-pass
```

### Check if connection in your windows host
```powershell 
WSManConfig: Microsoft.WSMan.Management\WSMan::localhost

Type            Name                           SourceOfValue   Value
----            ----                           -------------   -----
Container       Listener
  TCP    0.0.0.0:5986           0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:59869          0.0.0.0:0              LISTENING       11172
  TCP    192.168.1.10:5986    192.168.1.101:53233      ESTABLISHED     4
  TCP    [::]:5986              [::]:0                 LISTENING       4
```

## Encapsulate you windows username and password variables using ansible-vault:
ansible-vault create group_vars/windows/vault.yml
ansible-vault edit group_vars/windows/vault.yml


# TROUBLESHOOTING:

## Some troubleshooting commands I used during this process:

```powershell
PS C:\Users\ansible> Get-Service WinRM
Status   Name               DisplayName
------   ----               -----------
Running  WinRM              Windows Remote Management (WS-Manag...

PS C:\Users\ansible> winrm get winrm/config/service/auth
Auth
    Basic = true
    Kerberos = true
    Negotiate = true
    Certificate = false
    CredSSP = false
    CbtHardeningLevel = Relaxed

PS C:\Users\ansible> winrm get winrm/config/service
Service
    RootSDDL = O:NSG:BAD:P(A;;GA;;;BA)(A;;GR;;;IU)S:P(AU;FA;GA;;;WD)(AU;SA;GXGW;;;WD)
    MaxConcurrentOperations = 4294967295
    MaxConcurrentOperationsPerUser = 1500
    EnumerationTimeoutms = 240000
    MaxConnections = 300
    MaxPacketRetrievalTimeSeconds = 120
    AllowUnencrypted = false
    Auth
        Basic = true
        Kerberos = true
        Negotiate = true
        Certificate = false
        CredSSP = false
        CbtHardeningLevel = Relaxed
    DefaultPorts
        HTTP = 5985
        HTTPS = 5986
    IPv4Filter = *
    IPv6Filter = *
    EnableCompatibilityHttpListener = false
    EnableCompatibilityHttpsListener = false
    CertificateThumbprint
    AllowRemoteAccess = true

PS C:\Users\ansible> winrm get winrm/config/client/auth
Auth
    Basic = true
    Digest = true
    Kerberos = true
    Negotiate = true
    Certificate = true
    CredSSP = false

PS C:\Users\ansible> Test-WSMan -ComputerName localhost -Port 5986 -UseSSL
Test-WSMan : <f:WSManFault xmlns:f="http://schemas.microsoft.com/wbem/wsman/1/wsmanfault" Code="12175"
Machine="viriatis-desktop"><f:Message>The server certificate on the destination computer (localhost:5986) has the
following errors:
The SSL certificate is signed by an unknown certificate authority.
The SSL certificate contains a common name (CN) that does not match the hostname.     </f:Message></f:WSManFault>
At line:1 char:1
+ Test-WSMan -ComputerName localhost -Port 5986 -UseSSL
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidOperation: (localhost:String) [Test-WSMan], InvalidOperationException
    + FullyQualifiedErrorId : WsManError,Microsoft.WSMan.Management.TestWSManCommand

PS C:\Users\ansible> Get-WinEvent -LogName "Microsoft-Windows-WinRM/Operational" -MaxEvents 10 | Format-List
TimeCreated  : 04/02/2026 17:35:37
ProviderName : Microsoft-Windows-WinRM
Id           : 142
Message      : WSMan operation Identify failed, error code 12175
TimeCreated  : 04/02/2026 17:35:37
ProviderName : Microsoft-Windows-WinRM
Id           : 172
Message      : The server certificate on the destination computer (localhost:ᝢ) has the following errors:
               The SSL certificate is signed by an unknown certificate authority.
               The SSL certificate contains a common name (CN) that does not match the hostname.    . Fix the server
               certificate and try again.
TimeCreated  : 04/02/2026 17:35:37
ProviderName : Microsoft-Windows-WinRM
Id           : 254
Message      : Activity Transfer
TimeCreated  : 04/02/2026 17:35:37
ProviderName : Microsoft-Windows-WinRM
Id           : 145
Message      : WSMan operation Identify started with resourceUri NotSpecified
TimeCreated  : 04/02/2026 17:35:25
ProviderName : Microsoft-Windows-WinRM
Id           : 132
Message      : WSMan operation Get completed successfully
TimeCreated  : 04/02/2026 17:35:25
ProviderName : Microsoft-Windows-WinRM
Id           : 145
Message      : WSMan operation Get started with resourceUri
               http://schemas.microsoft.com/wbem/wsman/1/config/client/auth
TimeCreated  : 04/02/2026 17:35:12
ProviderName : Microsoft-Windows-WinRM
Id           : 132
Message      : WSMan operation Get completed successfully
TimeCreated  : 04/02/2026 17:35:12
ProviderName : Microsoft-Windows-WinRM
Id           : 145
Message      : WSMan operation Get started with resourceUri http://schemas.microsoft.com/wbem/wsman/1/config/service
TimeCreated  : 04/02/2026 17:34:58
ProviderName : Microsoft-Windows-WinRM
Id           : 132
Message      : WSMan operation Get completed successfully
TimeCreated  : 04/02/2026 17:34:58
ProviderName : Microsoft-Windows-WinRM
Id           : 145
Message      : WSMan operation Get started with resourceUri
               http://schemas.microsoft.com/wbem/wsman/1/config/service/auth

PS C:\Users\ansible> Get-LocalUser -Name ansible
Name    Enabled Description
----    ------- -----------
ansible True

PS C:\Users\ansible> Get-LocalGroupMember -Group "Administrators" | Where-Object {$_.Name -like "*ansible*"}
ObjectClass Name                  PrincipalSource
----------- ----                  ---------------
User        viriatis-desktop\ansible Local

PS C:\Users\ansible> Get-LocalGroupMember -Group "Remote Management Users" | Where-Object {$_.Name -like "*ansible*"}
PS C:\Users\ansible>

# On mac:
So the issue was never Python 3.14, SSL certificates, pywinrm, or WinRM configuration. It was macOS's proxy detection mechanism crashing during Python's fork operation.
```
