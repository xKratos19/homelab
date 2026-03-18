# Ansible

Manages all LXC containers from the Proxmox host.

## Usage
```bash
# Test connectivity
ansible all -i inventory/hosts.yml -m ping

# Audit all containers
ansible-playbook -i inventory/hosts.yml playbooks/audit.yml

# Install node_exporter on all containers
ansible-playbook -i inventory/hosts.yml playbooks/monitoring.yml

# Setup Prometheus + Grafana
ansible-playbook -i inventory/hosts.yml playbooks/setup-monitoring.yml
```

## Vault

Sensitive variables are encrypted with Ansible Vault.
Password file is stored at `/root/.ansible-vault-password` (never committed).
```bash
# Edit vault secrets
ansible-vault edit group_vars/all/vault.yml \
  --vault-password-file /root/.ansible-vault-password
```
