# vps-setup

## Create a droplet

This guide uses [DigitalOcean](https://www.digitalocean.com) to provision a Virtual Private Server (VPS), referred to as a *droplet* on their platform.

The same general steps apply if you're using a different VPS provider.

Once the droplet has been created, continue with the steps below to complete the server configuration.

## Connect to your VPS

```sh
ssh root@10.0.0.1
```

## Create a new user

From your VPS machine...

```sh
# Add a new user providing sudo privileges.
useradd --create-home --shell "/bin/bash" --groups sudo john

# If the user was added without sudo privileges, add user to sudo group.
usermod -aG sudo john

# Force a password to be set for the new user the first time this user login.
passwd --delete john
chage --lastday 0 john

# Test if user has sudo privileges.
su - john
sudo ls /
```

## Create an SSH keypair

From your local machine...

```sh
# Generate SSH keypair.
ssh-keygen -t ed25519 -C "john@email.com" -f $HOME/.ssh/id_ed25519_john

# Update permissions.
chmod 600 $HOME/.ssh/id_ed25519_john.pub

# Add SSH keypair to your SSH agent.
ssh-add $HOME/.ssh/id_ed25519_john

# Verify your SSH keypair was added to your SSH agent.
ssh-add -l
```

## Setup key based authentication

From your local machine...

```sh
# Copy the user's SSH keypair to the remote VPS.
ssh-copy-id -i $HOME/.ssh/id_ed25519_john.pub john@10.0.0.1

# Test if you can ssh to the VPS with your SSH keypair.
ssh -i $HOME/.ssh/id_ed25519_john john@10.0.0.1

# (Optional) Add a connection settings to your SSH config file.
# cat $HOME/.ssh/config
Host application-prod
  Hostname 10.0.0.1
  User john
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519_john

# Try to ssh to the remote VPS.
ssh application-prod
```

## Enhance security

From your VPS machine...

```sh
# Open sshd_config file.
sudo vim /etc/ssh/sshd_config

# Make the following changes:

## 1. Disable password authentication
#### From: `#PasswordAuthentication yes`
#### To:   `PasswordAuthentication no`

## 2. Disable the ability to login as the `root` user
#### From: `PermitRootLogin yes`
#### To:   `PermitRootLogin no`

## 3. Disable PAM authentication (recommended for hardening ssh)
#### From: `UsePAM yes`
#### To:   `UsePAM no`

# Also disable password authentication from `cloud-init` config file
# Not all VPS providers will have this config. Check with your provider if it does
sudo vim /etc/ssh/sshd_config.d/50-cloud-init.conf
#### From: `PasswordAuthentication yes`
#### To:   `PasswordAuthentication no`

# Apply changes.
sudo systemctl reload ssh
```

## Test your security changes

From your local machine...

```sh
# Try to ssh with the root user.
# If you don't get an error like below, verify if your changes were applied correctly.
# Expected: `root@10.0.0.1: Permission denied (publickey).`
ssh root@10.0.0.1

# Try to ssh with the user and password.
# If you don't get an error like below, verify if your changes were applied correctly.
# Expected: `john@10.0.0.1: Permission denied (publickey).`
ssh -o PreferredAuthentications=password john@10.0.0.1
```
