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

## Update packages

From your VPS machine...

```sh
# Update packages.
sudo apt update

# Upgrade packages.
##
## If you get a message saying: `A new version of configuration file
## /etc/ssh/sshd_config is available, but the version installed currently
## has been locally modified.`, select `keep the local version currently
## installed` and then <Ok> to continue.
##
## If you get a message saying: `Newer kernel available… Restarting the
## system to load the new kernel will not be handled automatically, so you
## should consider rebooting.`, select <Ok> to continue.
sudo apt upgrade

# Check if server needs to be rebooted.
## If you see a message similar to `*** System restart required ***` or `yes`,
## reboot the server. If possible, reboot it from the UI, otherwise type `reboot`.
cat /var/run/reboot-required

# After reboot, Upgrade packages again.
sudo apt upgrade

# If you see any pending packages, install them manually.
sudo apt install __PACKAGE__

# Upgrade packages again and expect to see no pending packages.
sudo apt upgrade
```

## Configure firewall

From your VPS machine...

```sh
# Deny all incoming traffic.
sudo ufw default deny incoming

# Allow all outgoing traffic.
sudo ufw default allow outgoing

# Allow SSH, HTTP and HTTPS traffic.
sudo ufw allow 22
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Enable/apply changes.
sudo ufw enable

# Verify status. At this point you should get the following output:
### Status: active
###
### To                         Action      From
### --                         ------      ----
### 22                         ALLOW       Anywhere
### 80/tcp                     ALLOW       Anywhere
### 443/tcp                    ALLOW       Anywhere
### 22 (v6)                    ALLOW       Anywhere (v6)
### 80/tcp (v6)                ALLOW       Anywhere (v6)
### 443/tcp (v6)               ALLOW       Anywhere (v6)
sudo ufw status

# List firewall rules.
sudo ufw app list

# Show rules you added.
sudo ufw show added
```

## Hardening SSH

From your VPS machine...

```sh
# Check your auth logs for lines like below...
## banner exchange: Connection from XXX.XXX.XXX.XXX port XXXXX: invalid format
sudo vim /var/log/auth.log

# Install fail2ban package.
sudo apt install fail2ban

# Create a local config file.
sudo vim /etc/fail2ban/jail.local

# Set the contents in jail.local to below output.
sudo cat /etc/fail2ban/jail.local
# Output (ignore `###`):
### [sshd]
### enabled = true
### port = 22
### maxretry = 5
### bantime = 12h

# Enable and restart service.
sudo systemctl enable --now fail2ban
sudo systemctl restart fail2ban

# Check service status.
sudo systemctl status fail2ban

# Check client service status.
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

## Reverse proxy

From your VPS machine...

```sh
# Install Caddy.
# Visit https://caddyserver.com/docs/install#debian-ubuntu-raspbian
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' |\
  sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' |\
  sudo tee /etc/apt/sources.list.d/caddy-stable.list
chmod o+r /usr/share/keyrings/caddy-stable-archive-keyring.gpg
chmod o+r /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy

# Check if caddy is running.
sudo systemctl status caddy

# If successfully running, you should be able to open your browser at
# http://<your_vps_ip> and see the Caddy welcome page.
```

## (Optional) - Application example

The example below shows how to run an application and use caddy as a reverse proxy.

This example assumes that you have a `go http server` application binary create in your local machine.

You can create the binary running the following command:

```sh
# For linux amd64
GOOS=linux GOARCH=amd64 go build -ldflags="-s" -o=./bin/linux_amd64/api ./cmd/api
```

Copy the binary to you VPS server.

```sh
rsync -P ./bin/linux_amd64/api john@10.0.0.1:~
```

Running the app.

```sh
# For now (we will setup this up with HTTPS later) allow connections
# to your api port that the server is listening to (for example 8090).
sudo ufw allow 8090/tcp

# Run your app (make sure to change to directly where binary exists).
./api

# Visit your api endpoint on the browser (for example if it has an health api)
# http://<your_vps_ip>:8090/api/v1/health
# Make sure the content is showing what your api is serving.
```

## (Optional) - Run the application as a service

Create a service file.

```sh
# cat api.service
[Unit]
Description=My API Service
After=network.target
After=network-online.target
Requires=network-online.target

[Service]
Type=exec
User=__USER__
Group=__GROUP__
WorkingDirectory=/home/__USERNAME__
ExecStart=/home/__USERNAME__/api
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Setup service with systemctl.

```sh
# Move service file to systemd dir.
sudo mv api.service /etc/systemd/system/

# Enable service to start at boot time.
sudo systemctl enable api

# Restart service.
sudo systemctl restart api

# Check service status.
sudo systemctl status api

# Check process that service is running on.
ps -U api

# After checking on the browser that you can still hit the api, disable port
# we allowed for testing.
sudo ufw delete allow 8090/tcp
sudo ufw reload

# Create a Caddyfile backup.
sudo mv /etc/caddy/Caddyfile /etc/caddy/Caddyfile.original

# Create a custom Caddyfile with contents below.
## Example 1
#### __VPS_IP_ADDRESS__ {
####   reverse_proxy 127.0.0.1:8090
#### }

## Example 2
#### http://__VPS_IP_ADDRESS__ {
####   respond /api/* "Not Permitted" 403
####   reverse_proxy 127.0.0.1:8090
#### }

## Example 3
#### http://__DOMAIN_NAME__ {
####   reverse_proxy 127.0.0.1:8090
#### }

## Example 4 - With HTTPS
{
  email you@email.com
}

# Do not add `http` or `https` before domain name
__DOMAIN__NAME__ {
  reverse_proxy 127.0.0.1:8090
}

# Move you custom Caddyfile to caddy's config location.
sudo mv Caddyfile /etc/caddy/

# Reload caddy service.
sudo systemctl reload caddy

# Check caddy service status.
sudo systemctl status caddy

# At this point you should have HTTPS setup.
# All requests to port HTTP should be redirected to HTTPS.
```
