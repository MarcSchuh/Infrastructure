# Infrastructure Repository

## General remarks
Please be aware, that this repository is in the beta status. 
Especially the nginx-example is not in a state, where it can be rolled out as is, because the SSL-part depends on certificates that can only be created, if the http-part is enabled.
It is recommended to first disable all SSL-parts in the nginx config, carry out the necessary certificate creation steps and then enable it again.

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

## WordPress
Create the necessary folders for WordPress:

```shell
mkdir -p ${DATA_DIR}/wordpress/html
mkdir -p ${DATA_DIR}/wordpress/db
chmod -R 755 ${DATA_DIR}/wordpress
```

ensure that your `.env` file is readily configured. 
Start your WordPress containers via

```shell
docker-compose -f docker-compose.wordpress.yml up -d
```
disable the SSL-part in the nginx.config and get the certificates:
```shell
docker-compose -f docker-compose.proxy.yml run certbot certonly -v --webroot -w /var/www/certbot-challenges -d wordpress.mydomain.de --email your-letsencrypttest@mydomain.de --agree-tos --no-eff-email
```

enable the SSL-parts again. 
The certbot will automatically take care of this certificate, too. 


## Nextcloud

Create necessary dirs
```shell
mkdir -p ${NEXTCLOUD_DATA_DIR}/app
mkdir -p ${NEXTCLOUD_DATA_DIR}/db
mkdir -p ${NEXTCLOUD_DATA_DIR}/redis
mkdir -p ${NEXTCLOUD_DATA_DIR}/fail2ban/jail.d
mkdir -p ${NEXTCLOUD_DATA_DIR}/fail2ban/filter.d
mkdir -p ${NEXTCLOUD_DATA_DIR}/fail2ban/action.d
chmod -R 755 ${NEXTCLOUD_DATA_DIR}/
```

and ensure that `.env` is set.

Start your nextcloud again via:
```shell
docker-compose -f docker-compose.nextcloud.yml up -d
```
disable the SSL-part in the nginx.config and get the certificates:
```shell
docker-compose -f docker-compose.proxy.yml run certbot certonly -v --webroot -w /var/www/certbot-challenges -d nextcloud.mydomain.de --email your-letsencrypttest@mydomain.de --agree-tos --no-eff-email
```

Copy the `filter.d`, `action.d` and `jail.d` to the corresponding folders and name them each `nextcloud.conf`

### Useful commands
Sometimes nextcloud need a bit of help.
To import an existing database, use:
```shell
docker cp  /pat/to/nextcloud/backup.sql nextcloud_db:/tmp/
docker exec -it nextcloud_db bash -c "mariadb -u root -p"YourRootPassword" nextcloud < /tmp/backup.sql"
```

Messing with the database can be done via:
```shell
docker exec -it nextcloud_db bash -c "mariadb -u root -p'YourRootPassword' -e \"ALTER DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;\""
docker exec -it nextcloud_db bash -c "mariadb -u root -p'YourRootPassword' -e \"GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextclouduser'@'%';\""
```

And to play a bit with the nextcloud itself, use

```shell
docker exec -u www-data -it nextcloud_app php occ maintenance:mode --on
docker exec -u www-data -it nextcloud_app php occ maintenance:repair
```

unbanning in fail2ban
```shell
docker exec nextcloud_fail2ban fail2ban-client status nextcloud
docker exec nextcloud_fail2ban fail2ban-client set nextcloud unbanip 192.168.1.100
```


### Observation
When moving a nextcloud instance, do not plain copy all in `html` to the new folder but instead only copy the content of `data`, `apps` and `themes`.
Adapt your `config.php` with care and do not overwrite everything. 
Especially the instance-id has to be kept as it is unique. 

## Watchtower
Watchtower keeps all docker images up-to-date if they have the tag `latest`.

## Jumphost
I have a small computer at home, which I cannot reach directly from outside. 
Therefor I installed a jumphost.

Create directory to edit config:
```shell
mkdir -p ${CONFIG_DIR}
```

Start docker, then edit in `${CONFIG_DIR}/sshd/sshd_config` and add
```
GatewayPorts yes
PermitRootLogin no
PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM no
AllowTcpForwarding yes
PermitTunnel yes
PermitTTY no
ForceCommand echo 'This account is restricted to SSH forwarding only.'
```
These settings ensure that the jumphost can only be used as jumphost.

And, add at `${CONFIG_DIR}/.ssh/authorized_keys` your public key of the device you want to reach.

On the computer you want to reach, you can start the tunnel via: 
```shell
/usr/bin/ssh -o ServerAliveInterval=60 -o ServerAliveCountMax=3 -N -R ${JUMP_PORT}:localhost:22 jumphost@ssh.mydomain.de -p ${SSH_PORT}
```
It is reasonable to service this as systemd. Assuming, you have started this from an account named `hook`, you can then reach your computer at home via

```shell
ssh -p ${JUMP_PORT} hook@ssh.mydomain.de
``` 
