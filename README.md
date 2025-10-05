# Shift Planning Deployment

The deployment is done using Ansible and Podman with support for multi-repository management.

All containers are run on the same machine.

## Deployment Overview

This deployment uses Ansible playbooks with a role-based architecture for deploying the Shift Planning application to a single VPS with HAProxy, firewalld, and multi-repository git-based updates.

### Multi-Repository Architecture

The Shift Planning application is organized into multiple repositories:

- **shift-planning-auth** - Authentication and user management service
- **shift-planning-database** - Database initialization scripts
- **shift-planning-gateway** - API gateway and reverse proxy
- **shift-planning-scheduling** - Scheduling and shift management service
- **shift-planning-front-end** - React frontend application
- **shift-planning-docs** - Documentation
- **shift-planning-deployment** - Ansible deployment configurations (this repository)

### Prerequisites

- Single VPS with CentOS/RHEL/Fedora or Ubuntu
- Ansible installed on deployment machine
- SSH access to VPS with sudo privileges
- Domain name pointing to VPS (optional, can use IP)
- GitHub access configured (SSH keys or tokens)

### Deployment Steps

1. **Initial VPS Setup** - Run `initial-setup-vps.yml`

   - Installs system packages (Podman, Node.js, Git, HAProxy, Firewalld)
   - Configures SSH keys for GitHub access
   - Configures firewall rules
   - **Important**: Displays SSH public key that must be added to GitHub

2. **Add SSH Key to GitHub** - Manual step

   - Copy the SSH public key displayed in step 1
   - Add it to each repository's Deploy Keys in GitHub Settings
   - Read-only access is sufficient

3. **Clone Repositories** - Run `playbooks/manage-repositories.yml`

   - Clones all required repositories to the VPS
   - Checks out the appropriate branch for the environment

4. **Deploy HAProxy** - Run `deploy-haproxy.yml` (optional)

   - Configures HAProxy as reverse proxy
   - Sets up load balancer for gateway service
   - Enables HAProxy stats interface

5. **Deploy Services** - Run `deploy-services.yml`
   - Validates all repositories are present
   - Creates Podman network
   - Deploys database container
   - Deploys application services (gateway, auth, scheduling)
   - Runs database migrations

### Quick Start

#### VPS Deployment (Dev/Prod)

```bash
# Step 1: Initial VPS setup (installs packages, configures SSH keys)
ansible-playbook initial-setup-vps.yml -i inventories/dev/hosts.yml --ask-vault-pass

# Step 2: Add the displayed SSH public key to GitHub
# Go to each repository -> Settings -> Deploy keys -> Add deploy key
# Paste the key and save (read-only access is sufficient)

# Step 3: Clone all repositories
ansible-playbook playbooks/manage-repositories.yml -i inventories/dev/hosts.yml

# Step 4: Deploy HAProxy (optional)
ansible-playbook deploy-haproxy.yml -i inventories/dev/hosts.yml --ask-vault-pass

# Step 5: Deploy application services
ansible-playbook deploy-services.yml -i inventories/dev/hosts.yml --ask-vault-pass

# To update repositories and redeploy:
ansible-playbook playbooks/manage-repositories.yml -i inventories/dev/hosts.yml
ansible-playbook deploy-services.yml -i inventories/dev/hosts.yml --ask-vault-pass
```

#### Local Development Deployment

```bash
# Deploy locally (uses existing checked-out repositories in workspace)
ansible-playbook deploy-services.yml -i inventories/local/hosts.yml --ask-vault-pass

# Note: Local deployment automatically uses your workspace repositories
# No git operations are performed in local environment
```

## Repository Management

### Managing Multiple Repositories

The deployment system includes a dedicated playbook for managing multiple Git repositories. This playbook is used to:

1. **Initial clone**: After VPS setup, clone all repositories for the first time
2. **Updates**: Pull latest changes from Git before redeployment

```bash
# Clone/pull all repositories (use after initial VPS setup or for updates)
ansible-playbook playbooks/manage-repositories.yml -i inventories/dev/hosts.yml

# Only manage repositories required for deployment
ansible-playbook playbooks/manage-repositories.yml -i inventories/dev/hosts.yml -e "deployment_repos_only=true"

# Only setup SSH keys (normally done by initial-setup-vps.yml)
ansible-playbook playbooks/manage-repositories.yml -i inventories/dev/hosts.yml --tags ssh-setup

# Only clone/pull repositories (skip SSH setup)
ansible-playbook playbooks/manage-repositories.yml -i inventories/dev/hosts.yml --tags repo-management
```

### Typical Update Workflow

When you want to deploy new code changes:

```bash
# 1. Update all repositories to latest version
ansible-playbook playbooks/manage-repositories.yml -i inventories/dev/hosts.yml

# 2. Redeploy services with updated code
ansible-playbook deploy-services.yml -i inventories/dev/hosts.yml --ask-vault-pass
```

**Note**: The `deploy-services.yml` playbook does NOT automatically pull from Git. You must explicitly run `manage-repositories.yml` first to update the code.

### Repository Configuration

Repositories are configured in `inventories/{env}/group_vars/all/vars.yml`:

```yaml
repositories:
  - name: auth
    git_url: "git@github.com:KieranVanRealMouth/shift-planning-auth.git"
    local_path: "{{ app_base_path }}/shift-planning-auth"
    required_for_deployment: true
    branch: main # or dev for development environment
```

### Environment-Specific Branches

- **Local**: Uses existing workspace repositories (no git operations)
- **Dev**: Uses `dev` branch for all repositories
- **Prod**: Uses `main` branch for all repositories

### Repository Validation

Before deployment, the system validates:

- ✓ All required repositories exist
- ✓ Repositories are valid git repositories
- ✓ Service repositories contain Dockerfiles
- ✓ Repository branches match environment expectations

```bash
# Run validation only
ansible-playbook deploy-services.yml -i inventories/dev/hosts.yml --tags validation
```

## Implementation Plan for Automated Development Environment Deployment

This plan outlines the step-by-step implementation of automated deployment for the development environment on a single VPS with HAProxy, firewalld, and git-based updates.

### Prerequisites

- Single VPS with CentOS/RHEL/Fedora or Ubuntu
- Ansible installed on deployment machine
- SSH access to VPS with sudo privileges
- Domain name pointing to VPS (optional, can use IP)

### Step 1: Setup Base System Dependencies

**Commit Goal**: Install and configure required system packages and services

- Install Podman and Podman Compose
- Install and configure firewalld
- Install HAProxy
- Install git and other required utilities
- Create deployment user with appropriate permissions
- Verify all services are running and enabled

**Test**: All required packages installed, services running, deployment user can manage containers

### Step 2: Configure firewalld Security Rules

**Commit Goal**: Implement proper firewall configuration to secure the VPS

- Configure firewalld zones and rules
- Allow only HTTP (80) and HTTPS (443) from external networks
- Allow SSH (22) from trusted networks only
- Block direct access to application ports (3000, 3100, 3200, 5432)
- Configure internal network rules for container communication
- Add logging for dropped packets

**Test**: External access only possible on ports 80/443, internal services not accessible externally

### Step 3: Setup Git Repository Synchronization

**Commit Goal**: Implement automated git-based code deployment

- Configure git repository access (SSH keys or token)
- Create deployment script for pulling latest changes
- Implement git hooks or polling mechanism for updates
- Add rollback capability to previous commit
- Configure environment-specific branch handling (dev/staging/prod)
- Add pre-deployment validation checks

**Test**: Git pull updates code, deployment can be triggered, rollback works

### Step 4: Configure HAProxy for Load Balancing and Rate Limiting

**Commit Goal**: Setup HAProxy as reverse proxy with security features

- Configure HAProxy to proxy requests to gateway service
- Implement rate limiting per IP address
- Add SSL/TLS termination (with Let's Encrypt integration)
- Configure health checks for backend services
- Add logging and monitoring capabilities
- Implement request filtering and security headers
- Configure graceful service switching during deployments

**Test**: HAProxy routes traffic correctly, rate limiting works, SSL certificates valid

### Step 5: Enhance Ansible Playbook for Production Deployment

**Commit Goal**: Extend existing Ansible configuration for production use

- Add HAProxy configuration management to playbooks
- Integrate firewalld configuration tasks
- Add git repository management tasks
- Implement zero-downtime deployment strategy
- Add service health checks and validation
- Configure log management and rotation
- Add backup and restore procedures for database

**Test**: Ansible playbook deploys complete stack, health checks pass, logs properly configured

### Step 6: Implement Container Registry and Image Management

**Commit Goal**: Setup proper container image lifecycle management

- Configure local container registry (or external registry integration)
- Add image building and versioning in CI/CD pipeline
- Implement image vulnerability scanning
- Add automated image cleanup policies
- Configure image pull secrets and authentication
- Add container health monitoring

**Test**: Images built and stored properly, old images cleaned up, vulnerability scans run

### Step 7: Add Monitoring and Alerting

**Commit Goal**: Implement comprehensive monitoring solution

- Deploy Prometheus for metrics collection
- Configure Grafana for visualization
- Add container and service health monitoring
- Configure log aggregation (ELK stack or similar)
- Implement alerting for critical issues (email/Slack)
- Add performance monitoring and capacity planning
- Configure backup monitoring and verification

**Test**: Monitoring dashboards show system health, alerts trigger correctly

### Step 8: Implement Automated Deployment Pipeline

**Commit Goal**: Create fully automated deployment process

- Integrate git webhooks or CI/CD triggers
- Add automated testing before deployment
- Implement blue-green or rolling deployment strategy
- Add automated database migrations
- Configure deployment notifications
- Add deployment approval workflow for production
- Implement automated rollback on failure

**Test**: Push to git triggers deployment, tests run automatically, rollback works on failure

### Step 9: Add Backup and Disaster Recovery

**Commit Goal**: Implement comprehensive backup and recovery procedures

- Configure automated database backups
- Implement configuration and volume backups
- Add backup encryption and secure storage
- Create disaster recovery runbooks
- Test backup restoration procedures
- Configure backup monitoring and verification
- Add point-in-time recovery capabilities

**Test**: Backups run automatically, restoration works, recovery procedures validated

### Step 10: Security Hardening and Compliance

**Commit Goal**: Implement additional security measures and compliance

- Configure SELinux/AppArmor policies
- Implement log monitoring for security events
- Add intrusion detection system (fail2ban)
- Configure regular security updates
- Implement secrets management (Vault integration)
- Add security scanning and compliance reporting
- Configure audit logging

**Test**: Security scans pass, compliance requirements met, audit logs generated

### Step 11: Performance Optimization and Scaling Preparation

**Commit Goal**: Optimize performance and prepare for horizontal scaling

- Implement container resource limits and requests
- Add performance monitoring and optimization
- Configure auto-scaling policies (future multi-node)
- Optimize database performance and connection pooling
- Add CDN integration for static assets
- Implement caching strategies (Redis)
- Performance testing and benchmarking

**Test**: Performance benchmarks meet requirements, resource usage optimized

### Step 12: Documentation and Maintenance Procedures

**Commit Goal**: Complete documentation and operational procedures

- Document all deployment procedures and runbooks
- Create troubleshooting guides
- Add operational maintenance schedules
- Document security procedures and incident response
- Create user guides for deployment management
- Add capacity planning and scaling guidelines
- Complete architecture documentation

**Test**: Documentation complete, procedures tested, team can operate system

### Final Architecture

```
Internet -> firewalld -> HAProxy (80/443) -> Gateway Container (3000)
                                         -> Auth Container (3100)
                                         -> Scheduling Container (3200)
                                         -> Database Container (5432)
```

### Key Features Implemented

- ✅ Single VPS deployment with HAProxy reverse proxy
- ✅ firewalld security configuration
- ✅ Git-based automated deployments
- ✅ Rate limiting and SSL termination
- ✅ Container orchestration with Podman
- ✅ Zero-downtime deployments
- ✅ Comprehensive monitoring and alerting
- ✅ Automated backups and disaster recovery
- ✅ Security hardening and compliance

### Security Considerations

- Only HTTP/HTTPS ports exposed externally
- All internal services protected by firewall
- SSL/TLS encryption for all external traffic
- Container isolation and resource limits
- Regular security updates and vulnerability scanning
- Comprehensive audit logging and monitoring
