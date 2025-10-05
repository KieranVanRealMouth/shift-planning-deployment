# Multi-Repository Git Management Implementation Summary

## Overview

This document summarizes the implementation of multi-repository Git management for the Shift Planning deployment system.

## Key Changes

### 1. Repository Configuration (Step 1)

Added centralized repository configuration to all environment inventory files:

- `inventories/local/group_vars/all/vars.yml`
- `inventories/dev/group_vars/all/vars.yml`
- `inventories/prod/group_vars/all/vars.yml`

Each repository entry includes:

- `name`: Service identifier
- `git_url`: Repository URL
- `local_path`: Clone destination
- `required_for_deployment`: Boolean flag
- `branch`: Environment-specific branch (main/dev)

### 2. Generic Git Repository Role (Step 2)

Refactored `roles/git_repository/tasks/main.yml` to support two modes:

**SSH Setup Mode** (when `repo_config` is not defined):

- One-time SSH key generation and configuration
- Git user configuration
- SSH host configuration
- Connection testing

**Repository Management Mode** (when `repo_config` is defined):

- Clone repository if not exists
- Pull latest changes if exists
- Branch-aware operations
- Validation and error handling

### 3. Repository Management Playbook (Step 3)

Created `playbooks/manage-repositories.yml`:

- Loops through repositories configuration
- Supports filtering by `required_for_deployment`
- Environment-aware (VPS vs local)
- Supports selective execution via tags

### 4. Environment-Specific Configuration (Step 4)

Updated `app_base_path` for all environments:

- **Local**: `{{ playbook_dir }}/../..` (workspace root)
- **Dev/Prod**: `/home/rocky/shift-planning` (VPS base directory)

### 5. Initial VPS Setup Integration (Step 5 - Refactored)

Modified `initial-setup-vps.yml`:

- **Only** sets up SSH keys
- Does **NOT** clone repositories
- Displays instructions for manual GitHub key addition
- User must run `manage-repositories.yml` separately after adding keys

### 6. Deploy Services Integration (Step 6 - Refactored)

Modified `deploy-services.yml`:

- **Removed** automatic git pull
- Added repository validation before deployment
- User must run `manage-repositories.yml` separately to update code
- Clear messaging about separation of concerns

### 7. Repository Paths (Step 7)

Service paths automatically use the new structure:

- `{{ app_base_path }}/shift-planning-auth`
- `{{ app_base_path }}/shift-planning-database`
- `{{ app_base_path }}/shift-planning-gateway`
- `{{ app_base_path }}/shift-planning-scheduling`

### 8. Repository Validation (Step 8)

Created `roles/repository_validation`:

- Validates all required repositories exist
- Checks repositories are valid git repositories
- Verifies Dockerfiles exist for service repositories
- Fails fast with clear error messages
- Integrated into `deploy-services.yml`

### 9. Documentation (Step 9)

Updated `README.md` with comprehensive documentation:

- Multi-repository architecture overview
- Clear deployment workflow
- Repository management instructions
- Environment-specific branch handling
- Typical update workflow

## Workflow

### Initial VPS Setup

```bash
# 1. Setup VPS and SSH keys
ansible-playbook initial-setup-vps.yml -i inventories/dev/hosts.yml --ask-vault-pass

# 2. Manually add SSH public key to GitHub (displayed in output)

# 3. Clone all repositories
ansible-playbook playbooks/manage-repositories.yml -i inventories/dev/hosts.yml

# 4. Deploy services
ansible-playbook deploy-services.yml -i inventories/dev/hosts.yml --ask-vault-pass
```

### Updating and Redeploying

```bash
# 1. Update repositories
ansible-playbook playbooks/manage-repositories.yml -i inventories/dev/hosts.yml

# 2. Redeploy with updated code
ansible-playbook deploy-services.yml -i inventories/dev/hosts.yml --ask-vault-pass
```

### Local Development

```bash
# Deploy using existing workspace repositories (no git operations)
ansible-playbook deploy-services.yml -i inventories/local/hosts.yml --ask-vault-pass
```

## Design Decisions

1. **Separation of Concerns**: SSH setup, repository management, and deployment are separate steps
2. **Manual Key Addition**: User must manually add SSH key to GitHub for security
3. **Explicit Updates**: No automatic git pull during deployment - must be explicit
4. **Environment-Aware**: Different behavior for local vs VPS deployments
5. **Idempotent**: Safe to run multiple times (clone if missing, pull if exists)
6. **Validation**: Fail fast if repositories are missing or invalid
7. **Selective Execution**: Tags allow fine-grained control of operations

## Files Created/Modified

### Created:

- `playbooks/manage-repositories.yml`
- `roles/repository_validation/tasks/main.yml`
- `roles/repository_validation/defaults/main.yml`
- `MULTI_REPO_IMPLEMENTATION_SUMMARY.md` (this file)

### Modified:

- `inventories/local/group_vars/all/vars.yml`
- `inventories/dev/group_vars/all/vars.yml`
- `inventories/prod/group_vars/all/vars.yml`
- `roles/git_repository/tasks/main.yml`
- `initial-setup-vps.yml`
- `deploy-services.yml`
- `README.md`

## Benefits

1. **Scalability**: Easy to add new repositories
2. **Consistency**: Same process for all repositories
3. **Flexibility**: Support for multiple environments and branches
4. **Safety**: Validation prevents deployment with missing repositories
5. **Clarity**: Clear separation between setup, update, and deployment
6. **Control**: Explicit control over when code is updated

## Tags Available

### manage-repositories.yml:

- `ssh-setup`: Only setup SSH keys
- `repo-management`: Only clone/pull repositories
- `verify`: Only verify repositories exist (local)

### deploy-services.yml:

- `validation`: Only validate repositories
- `infrastructure`: Deploy infrastructure only
- `services`: Deploy services only
- `build`: Only build images
- `--skip-tags validation`: Skip repository validation
- `--skip-tags migrations`: Skip database migrations
