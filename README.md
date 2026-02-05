# Multipass Role

Ansible role to install and configure Canonical Multipass on Windows systems.

## Requirements

- Windows 10/11 or Windows Server 2019/2022
- Hyper-V or VirtualBox installed
- Minimum 4GB RAM
- Minimum 10GB free disk space

## Role Variables

### Required Variables

None - all variables have sensible defaults.

### Optional Variables
```yaml
# Version
multipass_version: "1.14.1"

# Hypervisor
multipass_driver: "hyperv"  # or "virtualbox"

# Service configuration
multipass_service_start_mode: "auto"
multipass_ensure_service_running: true

# Installation behavior
multipass_allow_upgrade: false
multipass_force_reinstall: false
multipass_cleanup_installer: true

# Feature flags
multipass_check_hypervisor: true
multipass_add_to_path: true
multipass_verify_installation: true
```

## Dependencies

None

## Example Playbook
```yaml
- hosts: windows
  roles:
    - role: multipass
      vars:
        multipass_version: "1.14.1"
        multipass_driver: "hyperv"
```

## Tags

- `multipass` - All tasks
- `preflight` - Preflight checks only
- `install` - Installation tasks only
- `configure` - Configuration tasks only
- `validate` - Validation tasks only
- `cleanup` - Cleanup tasks only

## Usage

```shell
# Install Multipass
ansible-playbook -i inventories/dev/hosts.ini playbooks/install_multipass.yml --ask-vault-pass

# Install with specific version
ansible-playbook -i inventories/dev/hosts.ini playbooks/install_multipass.yml \
  -e "multipass_version=1.16.1" \
  --ask-vault-pass

# Upgrade existing installation
ansible-playbook -i inventories/dev/hosts.ini playbooks/install_multipass.yml \
  -e "multipass_allow_upgrade=true" \
  --ask-vault-pass

# Run only preflight checks
ansible-playbook -i inventories/dev/hosts.ini playbooks/install_multipass.yml \
  --tags preflight \
  --ask-vault-pass

# Skip validation
ansible-playbook -i inventories/dev/hosts.ini playbooks/install_multipass.yml \
  --skip-tags validate \
  --ask-vault-pass

# Dry run
ansible-playbook -i inventories/dev/hosts.ini playbooks/install_multipass.yml \
  --check \
  --ask-vault-pass

# Manage instances
ansible-playbook -i inventories/dev/hosts.ini playbooks/manage_multipass.yml --ask-vault-pass

# Remove Multipass
ansible-playbook -i inventories/dev/hosts.ini playbooks/remove_multipass.yml --ask-vault-pass
```