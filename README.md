# Infrastructure Repository

This repository serves as an infrastructure example for using Docker Compose to deploy services such as Jitsi, WordPress, and Nextcloud. 
It aims to provide a lightweight introduction to these tools and demonstrate the benefits of Infrastructure as Code.

## Status

> [!WARNING]
> This repository is currently in beta status.

## Overview
This repository provides a comprehensive infrastructure setup using Docker Compose to deploy various services including:
- Jitsi (Video Conferencing)
- WordPress (Content Management)
- Nextcloud (File Sharing & Collaboration)
- Monitoring Stack (Loki, Prometheus, Grafana)
- Additional utilities (Watchtower, Jumphost)

The goal is to demonstrate Infrastructure as Code practices while providing a lightweight, maintainable solution for self-hosted services.

## Prerequisites

### System Requirements
- Linux server (tested with Ubuntu 24.04)
- Internet domain with DNS control
- Docker Compose v2.33.1 or later
- Open firewall ports (see Firewall Configuration)

### Required Docker Plugins
- Loki logging plugin (required for container logging):
```bash
docker plugin install grafana/loki-docker-driver:latest --alias loki --grant-all-permissions
```

### Domain Configuration
Ensure your domain's DNS settings are properly configured to point to your server's IP address.
This usually means to add a few entries to your DNS zone (with `<domain>` being "your-domain.com" and the `<server_ip_address>` you got from your server provider):
Name: `$DOMAIN`, TTL: 86400, Type: A, Priority: 0, Data: `<server_ip_address>`
Name: `www.<domain>`, TTL: 86400, Type: A, Priority: 0, Data: `<server_ip_address>`
Name: `*.<domain>`, TTL: 86400, Type: A, Priority: 0, Data: `<server_ip_address>`

### Recommended Ubuntu Server Setup
Using the root account is generally not recommended, therefore one should create a new user (replace `<user>` with the name it is supposed to have below) with sudo rights and disable the root user login (after verifying that the new user works as intended).
- `sudo adduser <user>`
- Give the new user sudo rights: `usermod -aG sudo <user>`
- Check that logging in works with the new user and that it has sudo rights, e.g. via `sudo -v`
- Add a ssh-key for easier connection (replace `<name>`):
    - Create a ssh keypair on your local machine (in the .ssh folder): `ssh-keygen -t ed25519 -f "id_<name>"`
    - Copy the public ssh key to the server: `scp id_<name>.pub <user>@<ip_address>:id_<name>.pub`
    - Add the file to the list of authorized keys on the server (create .ssh folder in the folder of the `<user>` on the server if necessary): `cat id_<name>.pub >> .ssh/authorized_keys`
    - Create config entry locally for the server (here with "vps" as the host name):
        ```
        Host vps
        Hostname <server_ip_address>
        User <user>
        IdentityFile ~/.ssh/id_<name>
        ```
    - Access the server from the command line via: ssh vps
- After having set up a different login method (e.g. via ssh key) one can deactivate the option to log in via a password. To do so open `/etc/ssh/sshd_config` and set `PasswordAuthentication` to `no`. Restart sshd by `service sshd restart`.

> **Security Note**: It's highly recommended to disable password authentication for SSH and use only public key authentication.

## Server Setup
This section describes the general server setup necessary for then deploying the services like Jitsi, Nextcloud and so on.

### Firewall Configuration
1. Install and configure UFW:
```bash
sudo apt install ufw
sudo ufw allow 22/tcp
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable
```

2. Verify firewall status:
```bash
sudo ufw status
```

### Fail2ban Setup (Recommended)
1. Install fail2ban:
```bash
sudo apt install fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

2. Configure fail2ban (edit `/etc/fail2ban/jail.local`):
```ini
[sshd]
enabled = true 
port    = ssh
filter = sshd[mode=aggressive]
logpath = %(sshd_log)s
backend = %(sshd_backend)s
maxretry = 3 
findtime = 3600
bantime = 1200
```

3. Enable and start fail2ban:
```bash
sudo systemctl enable fail2ban 
sudo systemctl start fail2ban
```

4. Verify fail2ban status:
```bash
systemctl status fail2ban
fail2ban-client status sshd
```

### Docker Setup
1. Install Docker and Docker Compose:
```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

2. Add your user to the docker group (to run Docker without sudo):
```bash
sudo usermod -aG docker <user>
```

# Jitsi and Reverse Proxy Setup

This section describes how to set up Jitsi Meet with a reverse proxy for secure access. The setup consists of two main components:
1. Nginx reverse proxy with SSL termination
2. Jitsi Meet stack (Prosody, Jicofo, JVB, and Web interface)

## Prerequisites
- Docker and Docker Compose installed
- A domain name pointing to your server
- Ports 80 and 443 available on your server
- Environment variables, especially the paths, configured (see `env.example` files in both `proxy` and `jitsi` directories):
    - copy the example files via `cp env.example .env` and adjust the values as you need them - at least `_JITSI_DOMAIN` and the component secrets/passwords for jicofo and jvb (fill them with values generated by 'openssl rand -hex 16')
    - source them, so that the values are available in the shell via `source ./proxy/.env` and `source ./jitsi/.env`

## Directory Structure Setup

1. Create required directories for the proxy:
```bash
mkdir -p ${NGINX_DATA_DIR}/proxy 
mkdir -p ${NGINX_DATA_DIR}/config 
mkdir -p ${NGINX_DATA_DIR}/config/services-config 
mkdir -p ${NGINX_DATA_DIR}/certbot-challenges 
mkdir -p ${NGINX_DATA_DIR}/certbot-etc 
mkdir -p ${NGINX_DATA_DIR}/certbot-logs 
```

2. Copy configuration files:
```bash
cp ./proxy/nginx.conf ${NGINX_DATA_DIR}/config
cp ./proxy/ssl-params.conf ${NGINX_DATA_DIR}/config
cp ./proxy/security-headers.conf ${NGINX_DATA_DIR}/config
cp ./proxy/jitsi.conf ${NGINX_DATA_DIR}/config/services-config
```

3. Set proper permissions:
```bash
chmod -R 755 ${NGINX_DATA_DIR}
```

4. The proxy, Jitsi and Nextcloud require an external network:
```bash
docker network create proxy_network
docker network create jitsi_network
docker network create nextcloud_network
```

5. Create the folder for the certbot challenges:
```bash
sudo mkdir -p /var/www/certbot-challenges/
sudo chown www-data:www-data /var/www/certbot-challenges/
```

## Jitsi Setup
Jitsi has to be started first, since this will create the required network. Afterwards the reverse proxy can be started, as it connects to the then existing network.

1. Export the `NGINX_DATA_DIR`, so that it is available to the environment
```bash
export NGINX_DATA_DIR
```

2. Start Jitsi services in the jitsi folder:
```bash
docker compose -f docker-compose.jitsi.yml up -d
```

## Reverse Proxy Setup

1. Enable Jitsi configuration in Nginx:
    - Edit `{$NGINX_DATA_DIR}/config/nginx.conf`
    - Uncomment or add: `include /etc/nginx/services-config/jitsi.conf;`
    - Update the Nginx config for Jitsi under `${NGINX_DATA_DIR}/config/services-config` to have the correct `server_name` (both for port 80 and 443)  

2. Start the Nginx reverse proxy in the proxy folder:
```bash
docker compose -f docker-compose.proxy.yml up -d
```

## SSL Certificate Setup

1. Obtain SSL certificate (ensure all SSL/HTTPS related code is commented out):
```bash
docker compose -f docker-compose.proxy.yml run --rm certbot certonly \
  --webroot -w /var/www/certbot-challenges \
  -d ${_JITSI_DOMAIN} \
  --email <your-email@example.com> \
  --agree-tos --no-eff-email
```

2. Enable SSL in Jitsi configuration:
   - Edit `${NGINX_DATA_DIR}/config/services-config/jitsi.conf` and `${NGINX_DATA_DIR}/config/nginx.conf`
   - Uncomment all SSL (HTTPS) related code and update SSL certificate paths:
```nginx
ssl_certificate /etc/letsencrypt/live/<jitsi.example.com>/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/<jitsi.example.com>/privkey.pem;
```

3. Restart the proxy:
```bash
docker restart nginx_reverse_proxy
```

## User Management

To add users to Jitsi:
```bash
 docker exec -it jitsi_prosody prosodyctl --config /config/prosody.cfg.lua register <username> <jitsi_domain> <password>
```
which starts with a space, since prepending the command with a space (usually) keeps it out of the shell history.
If the password contains spaces, put quoutation marks around it.

## Verification

1. Check that the Jitsi and reverse proxy containers are up and running:
```bash
docker ps -a
```
Alternatively, check for example in the jitsi folder all respective containers via:
```bash
docker compose -f docker-compose.jitsi.yml ps
```

2. Test the setup:
   - Visit `https://<jitsi.example.com>`
   - Create and join a new meeting
   - Test audio/video functionality
   - Verify SSL certificate
   - Join the meeting from a second device or browser to verify that it works also for multiple people

## Troubleshooting

If your docker containers are restarting that usually means something is wrong with their config.
Errors can include:
- the SSL part is not commented out while not having a certificate yet
- a variable is used in the nginx config

1. Check logs:
```bash
docker logs -t --tail 100 jitsi_jvb
```
where -t adds timestamps and --tail 100 shows only the last 100 logs.

2. Common issues:
   - If SSL doesn't work, ensure ports 80 and 443 are open
   - If meetings don't connect, check JVB or Prosody logs
   - If authentication fails, verify Prosody configuration

## Updating Jitsi and the Jitsi configuration

In case you want to update to a newer version of Jitsi, simply run
```bash
docker compose -f docker-compose.jitsi.yml up -d
```
in the jitsi folder of the repo.
For changing the configuration, e.g. allowing guests, change the .env file, source it and then run docker compose again like for the update.
For disabling for example the tile enlargement, go to the `${NGINX_DATA_DIR}/config/services-config/web` folder, open the config file via, e.g., `sudo vim config.js`, add (under video configuration) the line `config.disableTileEnlargement = false;` and restart the jitsi web container via `docker restart jitsi-web`. 

# WordPress Setup

This section describes how to set up WordPress with MariaDB using Docker Compose. The setup includes:
- WordPress application container
- MariaDB database container
- Integration with the reverse proxy
- Resource limits and health monitoring

## Prerequisites
- Docker and Docker Compose installed
- Reverse proxy configured (see previous sections)
- Environment variables configured (see `wordpress/env.example`)

## Directory Structure Setup

1. Create required directories:
```bash
mkdir -p ${NGINX_DATA_DIR}/wordpress/html
mkdir -p ${NGINX_DATA_DIR}/wordpress/db
chmod -R 755 ${NGINX_DATA_DIR}/wordpress
```

## Configuration

1. Set up environment variables:
   - Copy `wordpress/env.example` to `wordpress/.env`
   - Update the following values:
     - `DB_NAME`: Your WordPress database name
     - `DB_USER`: Database user
     - `DB_PASSWORD`: Secure database password
     - `ROOT_PASSWORD`: Secure MariaDB root password
     - `WP_TABLE_PREFIX`: Optional table prefix (default: wp_)
     - Resource limits (optional):
       - `WORDPRESS_MEM_LIM`: Memory limit for WordPress (default: 1G)
       - `WORDPRESS_CPU_LIM`: CPU limit for WordPress (default: 0.5)
       - `WORDPRESS_DB_MEM_LIM`: Memory limit for database (default: 2G)
       - `WORDPRESS_DB_CPU_LIM`: CPU limit for database (default: 0.5)

## Deployment

1. Start WordPress services:
```bash
docker compose -f docker-compose.wordpress.yml up -d
```

2. Verify the deployment:
```bash
docker compose -f docker-compose.wordpress.yml ps
```

## SSL Configuration

1. Obtain SSL certificate:
```bash
docker compose -f docker-compose.proxy.yml run --rm certbot certonly -v \
  --webroot -w /var/www/certbot-challenges \
  -d <wordpress.example.com> \
  --email <your-email@example.com> \
  --agree-tos --no-eff-email
```

2. Enable SSL in WordPress configuration:
   - Edit `${NGINX_DATA_DIR}/config/services-config/wordpress.conf`
   - Uncomment and update SSL certificate paths:
```nginx
ssl_certificate /etc/letsencrypt/live/<wordpress.example.com>/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/<wordpress.example.com>/privkey.pem;
```

3. Restart the proxy:
```bash
docker restart nginx_reverse_proxy
```

## Initial Setup

1. Access WordPress:
   - Visit `https://wordpress.example.com`
   - Follow the WordPress installation wizard
   - Choose your language
   - Create an admin account
   - Complete the installation

2. Post-installation:
   - Install and configure essential plugins
   - Set up your theme
   - Configure permalinks

> **Note**: WordPress and MariaDB containers will be automatically updated by Watchtower when new versions are available. No manual update is required.

## Maintenance

1. Database backup:
```bash
docker exec wordpress_db mariadb -u root -p${ROOT_PASSWORD} ${DB_NAME} > backup.sql
```

2. Restore from backup:
```bash
docker exec -i wordpress_db mariadb -u root -p${ROOT_PASSWORD} ${DB_NAME} < backup.sql
```

## Troubleshooting

1. Check container logs:
```bash
docker compose -f docker-compose.wordpress.yml logs
```

2. Common issues:
   - If WordPress can't connect to the database, verify environment variables
   - If SSL doesn't work, check certificate paths and permissions
   - If updates fail, check disk space and permissions

# Nextcloud Setup

This section describes how to set up Nextcloud with MariaDB, Redis, and fail2ban using Docker Compose. The setup includes:
- Nextcloud application container
- MariaDB database container
- Redis for caching
- Cron job container for background tasks
- fail2ban for security
- Integration with the reverse proxy

## Prerequisites
- Docker and Docker Compose installed
- Reverse proxy configured:
    - Copy the nextcloud.conf from the proxy folder to `${NGINX_DATA_DIR}/config/services-config`
    - Make it executable: `chmod 755 ${NGINX_DATA_DIR}/config/services-config/nextcloud.conf`
    - Adjust it to point to the correct domain (for all occurances in both port 80 and 443)
    - Uncomment the include command for the new nextcloud configuration in the nginx.conf
    - Restart the reverse proxy to use the new configuration: `docker restart nginx_reverse_proxy`
- Environment variables configured (see `nextcloud/env.example`)
    - In the nextcloud folder copy the env.example file: `cp env.example .env`
    - Set at least:
        - `NEXTCLOUD_DATA_DIR`: location for nextcloud data
        - `DB_PASSWORD`: MariaDB database password
        - `ROOT_PASSWORD`: MariaDB root password
        - `REDIS_PASSWORD`
    - Other relevant settings include:
        - `NEXTCLOUD_VERSION`: Nextcloud version (default: latest)
        - `DB_NAME`: Database name
        - `DB_USER`: Database user
        - `PHP_UPLOAD_LIMIT`: Maximum upload size (default: 16GB)
        - `TIMEZONE`: Your timezone
        - `PUID` and `PGID`: User/Group IDs for file permissions (should be the result of `id <user>`, usually 1000:1000)
- Watchtower installed (for automatic updates), but can be also done after the nextcloud setup

## Directory Structure Setup

1. Create required directories:
```bash
mkdir -p ${LOCAL_NEXTCLOUD_DATA_DIR}/app
mkdir -p ${LOCAL_NEXTCLOUD_DATA_DIR}/db
mkdir -p ${LOCAL_NEXTCLOUD_DATA_DIR}/redis
mkdir -p ${LOCAL_NEXTCLOUD_DATA_DIR}/fail2ban/jail.d
mkdir -p ${LOCAL_NEXTCLOUD_DATA_DIR}/fail2ban/filter.d
mkdir -p ${LOCAL_NEXTCLOUD_DATA_DIR}/fail2ban/action.d
mkdir -p ${LOCAL_NEXTCLOUD_DATA_DIR}/fail2ban/db
chmod -R 755 ${LOCAL_NEXTCLOUD_DATA_DIR}
```

2. Copy fail2ban configuration files from the nextcloud directory:
```bash
cp nextcloud-jail.d.conf ${LOCAL_NEXTCLOUD_DATA_DIR}/fail2ban/jail.d/nextcloud.conf
cp nextcloud-filter.d.conf ${LOCAL_NEXTCLOUD_DATA_DIR}/fail2ban/filter.d/nextcloud.conf
cp iptables-multiport-docker.conf ${LOCAL_NEXTCLOUD_DATA_DIR}/fail2ban/action.d/
```

## Deployment

1. Start Nextcloud services from the nextcloud folder:
```bash
docker compose -f docker-compose.nextcloud.yml up -d
```

2. Verify the deployment:
```bash
docker compose -f docker-compose.nextcloud.yml ps
```

## SSL Configuration

1. Obtain SSL certificate by running the proxy folder:
```bash
docker compose -f docker-compose.proxy.yml run --rm certbot certonly -v \
  --webroot -w /var/www/certbot-challenges \
  -d <nextcloud.example.com> \
  --email <your-email@example.com> \
  --agree-tos --no-eff-email
```

2. Uncomment the SSL/HTTPS part in the Nextcloud configuration:
    - In `${NGINX_DATA_DIR}/config/services-config/nextcloud.conf`
    - Update the SSL certificate paths (if not done before):
```nginx
ssl_certificate /etc/letsencrypt/live/<nextcloud.example.com>/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/<nextcloud.example.com>/privkey.pem;
```

3. Restart the proxy:
```bash
docker restart nginx_reverse_proxy
```

## Initial Setup
1. Access Nextcloud:
    - Visit `https://<nextcloud.example.com>`
    - Create an admin account (chose a name and password for it)
        - Select the MariaDB option for the database and enter the database password
        - All other fields should be correctly prefilled

2. Nextcloud settings:
    - Log in with the admin account to complete the initial setup, e.g.:
        - Clean up the initial files
        - Add under 'Personal Settings' an email address and set the 'Locale'
        - Add the desired apps (under the user menu in the top right corner -> Apps), e.g.
            - Calendar
            - Contacts
            - Deck
            - Nextcloud Office
            - Forms
            - disable Recommendations
    - Adjust the nextcloud config:
        - Set correct logfile path, default phone region, configure the trusted proxies, maintenance window and email configuration by
            - Open the config under `${LOCAL_NEXTCLOUD_DATA_DIR}/html/config` with the correct user:
              `sudo -u www-data vim config.php`
            - Add to the config
              ```
  # User config
  'logfile' => '/var/www/html/data/nextcloud.log',
  'default_phone_region' => '<your region, e.g. DE>',
  'forwarded_for_headers' => ['HTTP_X_FORWARDED_FOR'],
  'trusted_proxies' => ['<reverse_proxy_ip_address_in_the_nextcloud_docker_network>'],
  'overwritehost' => '<nextcloud.example.com>',
  'overwriteprotocol' => 'https',
  'overwritewebroot' => '',
  'maintenance_window_start' => <start time for maintenance window in UTC, e.g. 1>,
              ```
              You can find out the IP address of the reverse proxy by looking at the nextcloud docker network via `docker network inspect nextcloud_network`. The `trusted_proxies` entry could be for example `'172.22.0.2'`
    - Set up email server via the `Administration Settings / Basic Settings` after logging in to your nextcloud instance.
    - Under 'Administration Settings' go to 'Overview' (on the left side) and check/fix the 'Security & setup warnings'

> **Note**: Nextcloud containers will be automatically updated by Watchtower when new versions are available. No manual update is required.

## Maintenance

1. Nextcloud Updates:
    > **Note**: While Watchtower handles container updates, Nextcloud apps and core updates need to be managed separately.

    ```bash
    # Update all Nextcloud apps
    docker exec -u www-data nextcloud_app php occ app:update --all --no-interaction

    # Update Nextcloud core (if not using 'latest' tag)
    docker exec -u www-data nextcloud_app php occ upgrade --no-interaction
    ```

2. Database maintenance:
    ```bash
    # Add missing indices
    docker exec nextcloud_app php occ db:add-missing-indices

    # Check database integrity
    docker exec nextcloud_app php occ integrity:check-core -n -v

    # Repair if needed
    docker exec nextcloud_app php occ maintenance:repair --include-expensive
    ```

3. File system maintenance:
    ```bash
    # Clean up file cache
    docker exec nextcloud_app php occ files:cleanup

    # Scan for new files
    docker exec nextcloud_app php occ files:scan --all
    ``` 

4. Database backup:
    ```bash
    # Enable maintenance mode
    docker exec -u www-data nextcloud_app php occ maintenance:mode --on

    # Create database backup
    docker exec nextcloud_db mariadb -u root -p${ROOT_PASSWORD} ${DB_NAME} > backup.sql

    # Disable maintenance mode
    docker exec -u www-data nextcloud_app php occ maintenance:mode --off
    ```

5. Restore from backup:
    ```bash
    docker exec -i nextcloud_db mariadb -u root -p${ROOT_PASSWORD} ${DB_NAME} < backup.sql
    ```

## Security

1. Check fail2ban status:
    ```bash
    docker exec nextcloud_fail2ban fail2ban-client status nextcloud
    ```

2. Unban an IP if needed:
    ```bash
    docker exec nextcloud_fail2ban fail2ban-client set nextcloud unbanip <ip_address>
    ```

## Troubleshooting

1. Check container logs:
    ```bash
    docker compose -f docker-compose.nextcloud.yml logs
    ```

2. Common issues:
   - If Nextcloud can't connect to the database, verify environment variables
   - If SSL doesn't work, check certificate paths and permissions
   - If background jobs fail, check cron container logs
   - If Redis connection fails, verify Redis password
   - If database access fails, ensure proper permissions:
     ```bash
     # Grant all privileges to the Nextcloud user
     docker exec nextcloud_db mariadb -u root -p${ROOT_PASSWORD} -e "GRANT ALL PRIVILEGES ON ${DB_NAME}.* TO '${DB_USER}'@'%';"
     docker exec nextcloud_db mariadb -u root -p${ROOT_PASSWORD} -e "FLUSH PRIVILEGES;"
     ```

## Migration Notes

When moving a Nextcloud instance, the following files/directories contain the important information:
    - config.php (usually under /var/www/nextlcloud/config or similar)   
    - mysql database
    - `data/`
    - `apps/`
    - `themes/`


# Watchtower Setup

Watchtower automatically updates your Docker containers when new versions are available. It's particularly useful for keeping your services up-to-date with security patches and new features.

## Features
- Automatic container updates
- Configurable update schedule
- Rolling restarts to minimize downtime
- Automatic cleanup of old images
- Logging integration with Loki

## Prerequisites
- Docker and Docker Compose installed
- Docker socket accessible
- Environment variables configured (see `watchtower/env.example`)

## Configuration

1. Set up environment variables:
   - Go into the `watchtower` directory, copy `env.example` to `.env` and source it `source .env`
   - Configure the following values by changing them in the `.env`:
     - `WATCHTOWER_VERSION`: Watchtower version (default: latest)
     - `RESTART_POLICY`: Container restart policy (default: unless-stopped)
     - `SCHEDULE`: Cron expression for update checks (default: 0 0 5 * * *)
     - (`TIMEZONE`: Your timezone (default: Europe/Berlin))
     - (`WATCHTOWER_CLEANUP`: Remove old images (default: true))
     - (`WATCHTOWER_ROLLING_RESTART`: Enable rolling restarts (default: true))

## Deployment

1. Start Watchtower:
```bash
docker compose -f docker-compose.watchtower.yml up -d
```

2. Verify the deployment:
```bash
docker compose -f docker-compose.watchtower.yml ps
```

## How It Works

1. **Update Process**:
   - Watchtower checks for new images at the scheduled time
   - If updates are found, it pulls the new images
   - Stops the old containers
   - Starts new containers with the same configuration
   - Removes old images if cleanup is enabled

2. **Container Selection**:
   - Watchtower updates all running containers by default
   - Containers with specific version tags (e.g., `stable`, `v1.0.0`) are also updated
   - To prevent updates, add the `--no-update` label to containers
   - To prevent restarts, add the `--no-restart` label to containers
   - Stopped containers are not updated

3. **Rolling Restarts**:
   - When enabled, containers are updated one at a time
   - Helps maintain service availability during updates
   - Particularly useful for multi-container applications

## Check Logs

```bash
docker compose -f docker-compose.watchtower.yml logs
```
or
```bash
docker logs watchtower
```

## Troubleshooting

1. Common issues:
   - If updates aren't happening, check the schedule
   - If containers aren't being updated, verify they use the `latest` tag
   - If cleanup isn't working, check disk space
   - If rolling restarts fail, check container dependencies

2. Manual update trigger (from the watchtower folder):
```bash
docker compose -f docker-compose.watchtower.yml run --rm -v /var/run/docker.sock:/var/run/docker.sock watchtower --run-once
```

# Jumphost Setup

A jumphost provides secure access to devices that are not directly reachable from the internet.

## Features
- Secure SSH tunneling
- Key-based authentication only

## Prerequisites
- Docker and Docker Compose
- Public domain name
- SSH key pair

## Setup

1. Configure environment:
```bash
cp jumphost/env.example jumphost/.env
# Edit .env with your settings
```

2. Create directories and config:
```bash
mkdir -p ${CONFIG_DIR}/sshd ${CONFIG_DIR}/.ssh
chmod 700 ${CONFIG_DIR}/.ssh
```

3. Create `${CONFIG_DIR}/sshd/sshd_config`:
```ini
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

4. Add your public key:
```bash
# Add your key to ${CONFIG_DIR}/.ssh/authorized_keys
chmod 600 ${CONFIG_DIR}/.ssh/authorized_keys
```

5. Start jumphost:
```bash
docker compose -f docker-compose.jumphost.yml up -d
```

## Client Setup

1. Create systemd service `/etc/systemd/system/ssh-tunnel.service`:
```ini
[Unit]
Description=SSH Tunnel to Jumphost
After=network.target

[Service]
Type=simple
User=YOUR_USER
ExecStart=/usr/bin/ssh -o ServerAliveInterval=60 -o ServerAliveCountMax=3 -N -R ${JUMP_PORT}:localhost:22 ${JUMPHOST_USER}@your-domain.com -p ${SSH_PORT}
Restart=always

[Install]
WantedBy=multi-user.target
```

2. Enable service:
```bash
sudo systemctl enable --now ssh-tunnel
```

## Usage

Access remote device:
```bash
ssh -p ${JUMP_PORT} YOUR_USER@your-domain.com
```


# Monitoring

1. Check container health:
```bash
docker compose -f docker-compose.jumphost.yml ps
```

2. View logs:
```bash
docker compose -f docker-compose.jumphost.yml logs
```

## Troubleshooting

1. Common issues:
   - If tunnel fails, check SSH service on remote device
   - If connection drops, verify ServerAlive settings
   - If access denied, check authorized_keys permissions
   - If port conflicts, verify port availability

2. Connection testing:
```bash
# Test SSH connection
ssh -p ${SSH_PORT} hook@your-domain.com

# Test tunnel port
nc -zv your-domain.com ${JUMP_PORT}
```

## Monitoring

Stack components:
- Loki: Log aggregation and storage
- Prometheus: Metrics collection and storage
- Grafana: Visualization and dashboards
- Promtail: Log shipping to Loki
- fail2ban: Security monitoring

### Setup

1. Create directories:
```bash
mkdir -p ${MONITORING_DATA_DIR}/{prometheus/{config,data},grafana,grafana_logs,loki/{data,config},promtail/config,fail2ban/{jail.d,filter.d,action.d,db}}
chmod -R 755 ${MONITORING_DATA_DIR}
chown -R 65534:65534 ${MONITORING_DATA_DIR}/prometheus/data
chown -R 10001:10001 ${MONITORING_DATA_DIR}/loki
chown -R 1000:1000 ${MONITORING_DATA_DIR}/grafana_logs
```

2. Copy configs:
```bash
cp prometheus.yml ${MONITORING_DATA_DIR}/prometheus/config/
cp loki-config.yml ${MONITORING_DATA_DIR}/loki/config/
cp promtail-config.yml ${MONITORING_DATA_DIR}/promtail/config/
cp grafana-jail.conf ${MONITORING_DATA_DIR}/fail2ban/jail.d/
cp grafana-filter.conf ${MONITORING_DATA_DIR}/fail2ban/filter.d/
cp iptables-multiport-docker.conf ${MONITORING_DATA_DIR}/fail2ban/action.d/
```

3. Start services:
```bash
docker compose -f docker-compose.monitoring.yml up -d
```

### Usage

1. Access Grafana: `http://grafana.mydomain.de`
   - Default login: admin/passwort from the env file
   - Add Prometheus data source: `http://prometheus:9090`
   - Add Loki data source: `http://monitoring_loki:3100`

3. Security monitoring:
```bash
# Check fail2ban status
docker exec logging_fail2ban fail2ban-client status grafana

# View banned IPs
docker exec logging_fail2ban fail2ban-client status
```

