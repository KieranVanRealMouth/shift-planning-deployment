# Ansible Roles Architecture Guide

## Overview

This deployment has been refactored from monolithic playbooks to a clean, role-based architecture with generic, loop-based service deployment. This makes the codebase more maintainable, testable, and scalable.

## Directory Structure

```
shift-planning-deployment/
├── deploy-services.yml           # Main orchestration playbook (30 lines vs 274 before!)
├── initial-setup-vps.yml         # VPS setup orchestration
├── inventories/
│   ├── local/
│   │   └── group_vars/all/
│   │       ├── vars.yml          # Service definitions and configuration
│   │       └── vault.yml         # Encrypted secrets
│   ├── dev/
│   └── prod/
├── roles/                        # All deployment logic lives here
│   ├── app_service/              # Generic service deployment (used in loop)
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   └── defaults/main.yml
│   ├── database/                 # Database container deployment
│   ├── podman_network/           # Podman network setup
│   ├── vps_setup/                # System packages and configuration
│   ├── git_repository/           # Git clone/pull operations
│   ├── firewall/                 # Firewalld configuration
│   └── haproxy/                  # Load balancer setup
└── playbooks/
    ├── single-node.yml           # Legacy playbook (kept for reference)
    └── single-node-roles.yml     # New role-based version
```

## Key Concepts

### 1. Service Definitions in Variables

All service configurations are now defined in `inventories/{env}/group_vars/all/vars.yml`:

```yaml
# Global configuration
external_hostname: "shiftplanning.rowanvi.nl" # Public hostname for this environment

app_services:
  # Gateway - ONLY service exposed to host
  - name: gateway
    image_name: shift-planning-gateway
    container_name: sp-gateway
    path: "{{ app_base_path }}/shift-planning-gateway"
    port: 3000
    internal_port: 3000
    expose_port: true # Exposed to host
    needs_migration: false
    env:
      FRONTEND_URL: "https://{{ external_hostname }}"
      # ... other env vars

  # Auth - Internal only
  - name: auth
    image_name: shift-planning-auth
    container_name: sp-auth
    path: "{{ app_base_path }}/shift-planning-auth"
    internal_port: 3100
    expose_port: false # Internal only
    startup_wait: 10
    needs_migration: true
    migration_db_url: "{{ vault_auth_database_url_external }}"
```

**Benefits:**

- Add new service = add one dictionary to the list
- All service configs in one place
- Easy to compare configurations across services
- **Security:** Only gateway exposed to host, internal services isolated
- **Flexibility:** `expose_port` flag controls port exposure per service

### 2. Generic Service Role with Loops

The `app_service` role deploys ANY service using the configuration passed to it:

```yaml
- name: Deploy Application Services
  hosts: all
  tasks:
    - include_role:
        name: app_service
      vars:
        service_config: "{{ item }}"
      loop: "{{ app_services }}"
      loop_control:
        label: "{{ item.name }}"
```

The role handles:

1. Building the Docker image
2. Stopping existing container
3. Deploying new container with env vars and healthcheck
4. Waiting for service to be ready
5. Running migrations (if `needs_migration: true`)

### 3. Specialized Infrastructure Roles

Infrastructure components that differ significantly get their own roles:

- **podman_network**: Creates the Podman network
- **database**: Deploys PostgreSQL (unique enough to warrant its own role)
- **vps_setup**: Installs system packages, configures SELinux
- **git_repository**: Handles SSH keys, clones/pulls repository
- **firewall**: Configures firewalld rules
- **haproxy**: Sets up load balancer

## Usage Examples

### Complete Deployment (VPS)

```bash
# Step 1: Initial VPS setup (packages, git, firewall)
ansible-playbook initial-setup-vps.yml -i inventories/dev/hosts.yml --ask-vault-pass

# Step 2: Deploy HAProxy load balancer
ansible-playbook deploy-haproxy.yml -i inventories/dev/hosts.yml --ask-vault-pass

# Step 3: Deploy application services
ansible-playbook deploy-services.yml -i inventories/dev/hosts.yml --ask-vault-pass
```

### Deploy Everything (Local)

```bash
ansible-playbook deploy-services.yml -i inventories/local/hosts.yml
```

### Deploy Only Infrastructure

```bash
ansible-playbook deploy-services.yml -i inventories/local/hosts.yml --tags "infrastructure"
```

### Deploy Only Services (skip infrastructure)

```bash
ansible-playbook deploy-services.yml -i inventories/local/hosts.yml --tags "services"
```

### Skip Migrations

```bash
ansible-playbook deploy-services.yml -i inventories/local/hosts.yml --skip-tags "migrations"
```

### Build Images Only (no deployment)

```bash
ansible-playbook deploy-services.yml -i inventories/local/hosts.yml --tags "build"
```

### Deploy Single Service

```bash
ansible-playbook deploy-services.yml -i inventories/local/hosts.yml \
  --extra-vars '{"app_services": [{"name": "gateway", ...}]}'
```

### VPS Setup Only

```bash
ansible-playbook initial-setup-vps.yml -i inventories/dev/hosts.yml --tags "vps_setup"
```

## Available Tags

### Infrastructure Tags

- `infrastructure`: All infrastructure components
- `network`: Podman network only
- `database`: Database only
- `firewall`: Firewall configuration only
- `haproxy`: HAProxy setup only

### Service Tags

- `services`: All application services
- `deploy`: Service deployment
- `build`: Image building
- `migrations`: Database migrations

### VPS Setup Tags

- `vps_setup`: All VPS setup tasks
- `packages`: System package installation
- `git`: Git configuration and repository clone
- `ssh`: SSH key configuration
- `selinux`: SELinux configuration

## Adding a New Service

To add a new service, simply add it to the `app_services` list in `vars.yml`:

```yaml
app_services:
  # ... existing services ...

  - name: notifications
    image_name: shift-planning-notifications
    container_name: "{{ notifications_container_name }}"
    path: "{{ app_base_path }}/notifications"
    port: "{{ notifications_container_port }}"
    internal_port: 3300
    needs_migration: true
    migration_db_url: "{{ vault_notifications_database_url_external }}"
    env:
      DATABASE_URL: "{{ vault_notifications_database_url }}"
      PORT: "3300"
      NODE_ENV: "production"
      # ... other env vars
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:3300/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

Then add the corresponding variables:

```yaml
notifications_container_name: "sp-notifications"
notifications_container_port: 3300
```

That's it! The next deployment will include your new service.

## Testing Roles Independently

You can test individual roles:

```bash
# Test app_service role with gateway
ansible-playbook -i inventories/local/hosts.yml test-role.yml \
  --extra-vars "service_config={{ app_services[0] }}"

# Test database role
ansible-playbook -i inventories/local/hosts.yml test-role.yml \
  --tags "database"
```

## Migration from Old Playbooks

The old monolithic playbooks are kept in `playbooks/` for reference:

- `playbooks/single-node.yml` - Original 274-line playbook
- `playbooks/setup-vps.yml` - Original VPS setup

The new versions are:

- `deploy-services.yml` - Main deployment (uses roles)
- `initial-setup-vps.yml` - VPS setup (uses roles)
- `playbooks/single-node-roles.yml` - Role-based single-node deployment

## Benefits of This Architecture

✅ **DRY (Don't Repeat Yourself)**: Service deployment logic written once  
✅ **Scalable**: Add new service = add one dictionary to list  
✅ **Testable**: Test roles independently  
✅ **Maintainable**: Change logic in one place  
✅ **Reusable**: Roles can be used in other projects  
✅ **Flexible**: Use tags to deploy specific components  
✅ **Clear**: Each role has a single responsibility

## Comparison: Before and After

### Before (Monolithic)

- `single-node.yml`: 274 lines
- Repeated code for each service
- Hard to add new services
- Difficult to test individual components
- No easy way to skip specific parts

### After (Role-Based)

- `deploy-services.yml`: 30 lines
- Service deployment logic in one reusable role
- Add service = add config dictionary
- Test roles independently
- Use tags for selective deployment

## Common Patterns

### Conditional Migrations

The `app_service` role only runs migrations if `needs_migration: true`:

```yaml
- name: Run Prisma migrations
  block:
    # ... migration tasks
  when: service_config.needs_migration | default(false)
```

### Loop Control Labels

Keep output readable with `loop_control.label`:

```yaml
loop_control:
  loop_var: item
  label: "{{ item.name }}"
```

This shows "gateway", "auth", "scheduling" instead of the entire config dictionary.

### Handlers for Restarts

Use handlers for actions that should run once after changes:

```yaml
# In tasks/main.yml
- name: Deploy HAProxy configuration
  template:
    src: haproxy.cfg.j2
    dest: /etc/haproxy/haproxy.cfg
  notify: restart haproxy

# In handlers/main.yml
- name: restart haproxy
  systemd:
    name: haproxy
    state: restarted
```

## Troubleshooting

### Role Not Found

```
ERROR! the role 'app_service' was not found
```

**Solution**: Make sure you're running the playbook from the deployment directory, or use absolute paths.

### Variable Not Defined

```
ERROR! 'app_services' is undefined
```

**Solution**: Check that `inventories/{env}/group_vars/all/vars.yml` is loaded and contains `app_services`.

### Migration Fails

```
Migration for auth service failed
```

**Solution**:

1. Check database is running: `podman ps | grep database`
2. Check database URL is correct in vault.yml
3. Check container logs: `podman logs sp-auth`
4. Run migration manually: `podman exec sp-auth npx prisma migrate deploy`

## Next Steps

1. Test the new deployment in local environment
2. Migrate dev environment to use new playbooks
3. Add monitoring role (Prometheus/Grafana)
4. Add backup role for database
5. Create multi-node deployment playbook using same roles

## Questions?

Refer to:

- Individual role README files (if created)
- Ansible documentation: https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html
- Original playbooks in `playbooks/` directory for reference
