# vps-setup

This document provides guidence for provisioning and securing a Virtual Private Server (VPS). While this guide uses **DigitalOcean** (where a VPS is called a *Droplet*), the same principles apply to most VPS providers.

---

## Create a Droplet

Provision a VPS using your preferred provider (e.g., DigitalOcean).

After the droplet has been created, proceed with the configuration steps below.

### Example Environment

For demonstration purposes, we will assume:

- **OS Machine:** `Ubuntu`
- **VPS IP Address:** `192.51.100.112`
- **Application/Username:** `greenhouse`

## Connect to Your VPS

```sh
ssh root@192.51.100.112
```

## Create a New User

```sh
# Create a new user with sudo privileges
useradd --create-home --shell "/bin/bash" --groups sudo greenhouse

# Add user to sudo group (if needed, above command already does this)
usermod -aG sudo greenhouse

# Force password reset on first login
passwd --delete greenhouse
chage --lastday 0 greenhouse

# Test sudo access
su - greenhouse
sudo ls /
```

## Create an SSH Keypair

Run the following commands **on your local machine**:

```sh
# Generate SSH keypair
ssh-keygen -t ed25519 -C "admin@greenhouse.com" -f $HOME/.ssh/id_ed25519_greenhouse

# Set permissions
chmod 600 $HOME/.ssh/id_ed25519_greenhouse.pub

# Add key to SSH agent
ssh-add $HOME/.ssh/id_ed25519_greenhouse

# Verify key is added
ssh-add -l
```

## Setup Key-Based Authentication

Run the following commands **on your local machine**:

```sh
# Copy public key to VPS
ssh-copy-id -i $HOME/.ssh/id_ed25519_greenhouse.pub greenhouse@192.51.100.112

# Test SSH login
ssh -i $HOME/.ssh/id_ed25519_greenhouse greenhouse@192.51.100.112
```

### (Optional) Configure SSH Shortcut

Add to `~/.ssh/config`:

```sh
Host greenhouse-prod
  Hostname 192.51.100.112
  User greenhouse
  Port 22
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519_greenhouse
```

Then connect to VPS:

```sh
ssh greenhouse-prod
```

## Enhance SSH Secutiry

On your VPS:

```sh
# Open SSH config fiel
sudo vim /etc/ssh/sshd_config

# Update the following settings (uncomment and/or modify value)
# These settings will, respectively:
# - Disable password authentication
# - Disable root login
# - Disable PAM (recommended for hardening ssh)
PasswordAuthentication no
PermitRootLogin no
UsePAM no

# If applicable, also update below file. Depending on your provider, the file name might differ
sudo vim /etc/ssh/sshd_config.d/50-cloud-init.conf
PasswordAuthentication no

# Apply changes
sudo systemctl reload ssh
```

## Test SSH Secutiry Changes

Run the following commands **on your local machine**:

```sh
# Root login should fail
ssh root@192.51.100.112
# Expected: Permission denied (publickey)

# Password authentication should fail
ssh -o PreferredAuthentications=password greenhouse@192.51.100.112
# Expected: Permission denied (publickey)
```

## Update System Packages

On your VPS:

```sh
sudo apt update
sudo apt upgrade

# If prompted about modified SSH configs, select:
### keep the local version currently installed

# Check if reboot is required
cat /var/run/reboot-required

# If required...
reboot

# After reboot, upgrade packages again
sudo apt upgrade

# Install any remaining pendidng packages manually
sudo apt install __PACKAGE__

# Updrage packages one last time
sudo apt upgrade
```

## Configure Firewall (UFW)

```sh
# Default rules
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow essential services (ports 80 and 443 assume that you will have a service listening for incoming requests)
sudo ufw allow 22
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Enable firewall
sudo ufw enable

# Verify
sudo ufw status
sudo ufw app list
sudo ufw show added
```

Expected status should show only ports 22, 80, and 443 allowed.

```
Status: active

To                         Action      From
--                         ------      ----
22                         ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
443/tcp                    ALLOW       Anywhere
22 (v6)                    ALLOW       Anywhere (v6)
80/tcp (v6)                ALLOW       Anywhere (v6)
443/tcp (v6)               ALLOW       Anywhere (v6)
```

## Harden SSH with Fail2Ban

```sh
sudo apt install fail2ban
sudo vim /etc/fail2ban/jail.local
```

Add to `/etc/fail2ban/jail.local` file:

```
[sshd]
enabled = true
port = 22
maxretry = 5
bantime = 12h
```

Enable and verify changes:

```sh
# Enable and restart service
sudo systemctl enable --now fail2ban
sudo systemctl restart fail2ban

# Check service status
sudo systemctl status fail2ban

# Check client service status
sudo fail2ban-client status
sudo fail2ban-client status sshd

# Check your auth logs for lines like below...
# banner exchange: Connection from XXX.XXX.XXX.XXX port XXXXX: invalid format
sudo vim /var/log/auth.log
```
