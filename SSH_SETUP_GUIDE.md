# SSH Setup Guide for VPS Git Access

This guide explains how to use the enhanced Ansible playbook to set up SSH access for GitHub on your VPS.

## Overview

The solution provides a holistic approach to SSH setup through Ansible, which enables secure Git repository access over SSH. The enhanced `setup-vps.yml` playbook configures SSH keys and GitHub access, preparing your VPS for secure Git operations.

## SSH Setup Process

### What the Enhanced setup-vps.yml Does

The playbook now includes improved SSH configuration:

1. **Generates ed25519 SSH key** (more secure than RSA)
   - Key location: `/root/.ssh/id_ed25519_github`
   - Proper file permissions (600 for private, 644 for public)

2. **Creates proper SSH config** 
   - Automatically detects Git hostname (github.com)
   - Uses the correct SSH user (git)
   - Disables host key checking for automation

3. **Tests SSH connection**
   - Validates the SSH setup
   - Provides clear feedback on connection status

4. **Displays setup instructions**
   - Shows the exact public key to add to GitHub
   - Provides step-by-step GitHub configuration

### Running the Initial Setup

```bash
# Run the enhanced setup playbook
ansible-playbook -i deployment/inventories/dev/hosts.yml \
                 deployment/playbooks/setup-vps.yml \
                 --ask-vault-pass

# When prompted, enter your vault password
```

### GitHub Configuration

After running the playbook, you'll see output like:

```
=== GITHUB DEPLOY KEY SETUP ===
Please add the following public key to your GitHub repository's deploy keys:
Repository: git@github.com:KieranVanRealMouth/shift-planning-back-end.git
Settings -> Deploy keys -> Add deploy key
Title: VPS Deploy Key (your-hostname)
Key: ssh-ed25519 AAAAC3NzaC1lZDI1NTE5... (your key)
✓ Allow write access: YES (optional, only needed for push operations)
================================
```

**To add the deploy key to GitHub:**

1. Go to your repository: https://github.com/KieranVanRealMouth/shift-planning-back-end
2. Click **Settings** → **Deploy keys** → **Add deploy key**
3. Title: `VPS Deploy Key (your-hostname)`
4. Key: Paste the SSH key from the playbook output
5. **✓ Check "Allow write access"** (optional, only needed if you plan to push from this server)
6. Click **Add key**

## Testing SSH Setup

After the SSH setup is complete and the deploy key is added to GitHub:

```bash
# SSH into your VPS
ssh root@your-vps-ip

# Test SSH connection to GitHub
ssh -T git@github.com
# Should show: "Hi KieranVanRealMouth! You've successfully authenticated..."
```

If you see the authentication success message, your SSH setup is working correctly and ready for Git operations.

## Configuration Files Changed

### 1. `deployment/inventories/dev/group_vars/all/vars.yml`
- Changed `git_repo` from HTTPS to SSH format
- Updated `ssh_key_path` to use GitHub-specific key

### 2. `deployment/playbooks/setup-vps.yml`
- Enhanced SSH key generation (ed25519 instead of RSA)
- Fixed SSH config generation with proper hostname detection
- Added SSH connection testing
- Improved user feedback and instructions

## Security Features

- Uses ed25519 SSH keys (more secure than RSA)
- Proper file permissions on SSH keys
- Host key checking disabled for automation (safe in controlled environment)
- Uses GitHub deploy keys (repository-specific, not account-wide)

## Troubleshooting

### SSH Connection Issues
```bash
# Test SSH connection manually
ssh -T git@github.com

# Debug SSH connection
ssh -vT git@github.com
```

### Permission Issues
```bash
# Fix SSH key permissions if needed
chmod 600 /root/.ssh/id_ed25519_github
chmod 644 /root/.ssh/id_ed25519_github.pub
chmod 600 /root/.ssh/config
```

### Git Clone Issues
```bash
# Verify SSH config
cat /root/.ssh/config

# Test with verbose output
GIT_SSH_COMMAND="ssh -v" git clone git@github.com:KieranVanRealMouth/shift-planning-back-end.git
```

## Next Steps

1. Run the enhanced `setup-vps.yml` playbook
2. Add the deploy key to GitHub as shown in the output
3. Test SSH connection to verify setup
4. Your VPS is now ready for secure Git operations
