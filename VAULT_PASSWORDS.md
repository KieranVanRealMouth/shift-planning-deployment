# Ansible Vault Passwords

This document contains information about the vault passwords used to encrypt sensitive configuration data.

## Environment Vault Passwords

### Local Environment
- **Password**: `local-vault-password`
- **File**: `inventories/local/group_vars/all/vault.yml`

### Development Environment
- **Password**: `dev-vault-password`
- **File**: `inventories/dev/group_vars/all/vault.yml`

### Production Environment
- **Password**: `prod-vault-password`
- **File**: `inventories/prod/group_vars/all/vault.yml`

## Usage

### To decrypt and edit a vault file:
```bash
ansible-vault edit inventories/local/group_vars/all/vault.yml --vault-password-file <(echo "local-vault-password")
```

### To view a vault file:
```bash
ansible-vault view inventories/local/group_vars/all/vault.yml --vault-password-file <(echo "local-vault-password")
```

### To run playbooks:
```bash
`ansible-playbook -i inventories/local deploy-services.yml --vault-password-file <(echo "local-vault-password")
````

## Security Notes

- **IMPORTANT**: In production, use strong, unique passwords and store them securely
- Consider using a password manager or external secret management system
- These example passwords are for demonstration only
- Never commit vault passwords to version control
- Consider using `.vault_pass` files (excluded from git) for each environment

## Changing Vault Passwords

To change a vault password:
```bash
ansible-vault rekey inventories/local/group_vars/all/vault.yml
```