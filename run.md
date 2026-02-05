# Run ansible playbooks

## Simple version
```shell
ansible-playbook -i inventories/dev/hosts.ini playbooks/install_multipass.yml --ask-vault-pass
```

## With specific version
```shell
ansible-playbook -i inventories/dev/hosts.ini playbooks/install_multipass.yml \
  -e "multipass_version=1.14.1" \
  --ask-vault-pass
```

## Dry run first
```shell
ansible-playbook -i inventories/dev/hosts.ini playbooks/install_multipass.yml \
  --check --diff \
  --ask-vault-pass
```

## Verbose output
```shell
ansible-playbook -i inventories/dev/hosts.ini playbooks/install_multipass.yml \
  -vv \
  --ask-vault-pass
```