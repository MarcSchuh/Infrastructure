# Infrastructure Repository

## Introduction
This repository serves as an infrastructure example for using Docker Compose to deploy services such as Jitsi, WordPress, and Nextcloud. It aims to provide a lightweight introduction to these tools and demonstrate the benefits of Infrastructure as Code.

## Prerequisites
The following requirements should be met:
- A Linux server (currently tested with Ubuntu 24.04)
- An internet domain
- Control over DNS settings
- Docker Compose (v2.33.1 or later)
- Open ports on your firewall (see Firewall Configuration section)

Ensure that the website where the services should be accessible is properly configured in your domain management.

## Docker
Install Docker using:

```shell
sudo apt install docker docker-compose
```

Enable your user to run Docker without sudo:

```shell
sudo usermod -aG docker $USER
```

## Firewall
Make sure the firewall is up and properly configured:

```shell
sudo apt install ufw
```

Configure the necessary ports:

```shell
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 8000/tcp
sudo ufw allow 8443/tcp
sudo ufw allow 10000/udp
sudo ufw allow 3478/udp
sudo ufw allow 5349/tcp
sudo ufw enable
```

Check the firewall status:

```shell
sudo ufw status
```

### File2ban
Optional but highly recommended:

```shell
sudo apt install fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

The default settings in  ```/etc/fail2ban/jail.local``` are fine, but I recommend:
```text
[sshd]
enabled = true
port    = ssh
logpath = %(sshd_log)s
backend = %(sshd_backend)s
maxretry = 3
findtime = 1200
bantime = 1200
```

Enable und starte `fail2ban` neu
```shell
sudo systemctl enable fail2ban 
sudo systemctl start fail2ban
```

Verify that `fail2ban` is running:
```shell
systemctl status fail2ban
```

And check if bans are occurring:
```shell
fail2ban-client status sshd
```
Generally, it's also advisable to disable password login over SSH and only allow authentication via public key methods.

# Jitsi
For this setup, the `docker-compose.jitsi.yml` and `docker-compose.proxy.yml`are important.
Both assume that a `.env` file is present, similar to the `env-example`.
Ensure that all paths are present and have the access setting `755`.

Start by creating the Docker network:

```shell
docker network create jitsi_network
```

Comment out the SSL-part in the `nginx.conf`

```text
        listen 443 ssl;
        listen [::]:443 ssl;
        server_name jitsi.test.mydomain.de;

        ssl_certificate /etc/letsencrypt/live/jitsi.test.mydomain.de/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/jitsi.test.mydomain.de/privkey.pem;
```
otherwise the nginx server will not start. 

Start the nginx-server
```shell
docker-compose -f docker-compose.proxy.yml up -d --force-recreate --build
```

## Certificate
Get a certificate for your domain via:

```shell
docker-compose -f docker-compose.proxy.yml run --rm certbot certonly \
  --webroot -w /var/www/certbot-challenges \
  -d jitsi.example.com \
  --email your-email@example.com \
  --agree-tos --no-eff-email
```

Remove the comments in the `nginx.conf` and restart the nginx-server:

```shell
docker-compose -f docker-compose.proxy.yml up -d --force-recreate --build
```

## Jitsi user
Before starting your Jitsi, ensure that all passwords are set to proper values.
Start your Jitsi via
```shell
docker-compose -f docker-compose.jitsi.yml up -d --force-recreate --build
```

Add users via
```shell
docker exec -it jitsi_prosody prosodyctl --config /config/prosody.cfg.lua register my_user jitsi.mydomain.de SuperSecretPassword
```

and with that your Jitsi should be up and running