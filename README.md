# Infrastructure Repository

This repository serves as an infrastructure example for using Docker Compose to deploy services such as Jitsi, WordPress, and Nextcloud. 
It aims to provide a lightweight introduction to these tools and demonstrate the benefits of Infrastructure as Code.

## Status
⚠️ **Beta Status** ⚠️

This repository is currently in beta status. Please note the following important considerations:
- The nginx-example configuration requires special attention for SSL setup
- SSL configuration depends on certificates that can only be created after enabling HTTP
- Recommended approach: Create certificates → Enable SSL

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

## Installation

### Docker Setup
1. Install Docker and Docker Compose:
```bash
sudo apt install docker docker-compose
```

2. Add your user to the docker group (to run Docker without sudo):
```bash
sudo usermod -aG docker $USER
```

### Firewall Configuration
1. Install and configure UFW:
```bash
sudo apt install ufw
sudo ufw allow 22/tcp
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

> **Security Note**: It's highly recommended to disable password authentication for SSH and use only public key authentication.

# Jitsi and Reverse Proxy Setup

This section describes how to set up Jitsi Meet with a reverse proxy for secure access. The setup consists of two main components:
1. Nginx reverse proxy with SSL termination
2. Jitsi Meet stack (Prosody, Jicofo, JVB, and Web interface)

## Prerequisites
- Docker and Docker Compose installed
- A domain name pointing to your server
- Ports 80 and 443 available on your server
- Environment variables configured (see `env.example` files in both `proxy` and `jitsi` directories)

## Directory Structure Setup

1. Create required directories:
```bash
mkdir -p ${DATA_DIR}/proxy 
mkdir -p ${DATA_DIR}/config 
mkdir -p ${DATA_DIR}/config/services-config 
mkdir -p ${DATA_DIR}/certbot-challenges 
mkdir -p ${DATA_DIR}/certbot-etc 
mkdir -p ${DATA_DIR}/certbot-logs 
```

2. Copy configuration files:
```bash
cp ./nginx.conf ${DATA_DIR}/config
cp ./ssl-params.conf ${DATA_DIR}/config
cp ./security-headers.conf ${DATA_DIR}/config
cp ./jitsi.conf ${DATA_DIR}/config/services-config
```

3. Set proper permissions:
```bash
chmod -R 755 ${DATA_DIR}
```

## Setup Order

The services should be started in the following order:
1. Start Jitsi services first (this will create the required network)
2. Then start the reverse proxy (which will connect to the existing network)

## Reverse Proxy Setup

1. Start the Nginx reverse proxy:
```bash
docker-compose -f docker-compose.proxy.yml up -d
```

2. Enable Jitsi configuration in Nginx:
   - Edit `${DATA_DIR}/config/nginx.conf`
   - Uncomment or add: `include /etc/nginx/services-config/jitsi.conf;`

3. Restart the proxy to apply changes:
```bash
docker restart nginx_reverse_proxy
```

## Jitsi Setup

1. Configure environment variables:
   - Copy `jitsi/env.example` to `jitsi/.env`
   - Update the values, especially:
     - `PUBLIC_URL`: Your domain name
     - `ENABLE_GUESTS`: Set to 1 to allow guest access
     - `JVB_AUTH_USER` and `JVB_AUTH_PASSWORD`: For JVB authentication

2. Start Jitsi services:
```bash
docker-compose -f docker-compose.jitsi.yml up -d
```

## SSL Certificate Setup

1. Obtain SSL certificate:
```bash
docker-compose -f docker-compose.proxy.yml run --rm certbot certonly \
  --webroot -w /var/www/certbot-challenges \
  -d jitsi.example.com \
  --email your-email@example.com \
  --agree-tos --no-eff-email
```

2. Enable SSL in Jitsi configuration:
   - Edit `${DATA_DIR}/config/services-config/jitsi.conf`
   - Uncomment and update SSL certificate paths:
```nginx
ssl_certificate /etc/letsencrypt/live/jitsi.example.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/jitsi.example.com/privkey.pem;
```

3. Restart the proxy:
```bash
docker restart nginx_reverse_proxy
```

## User Management

To add users to Jitsi:
```bash
docker exec -it jitsi_prosody prosodyctl --config /config/prosody.cfg.lua register username jitsi.example.com password
```

## Verification

1. Check service status:
```bash
docker-compose -f docker-compose.jitsi.yml ps
docker-compose -f docker-compose.proxy.yml ps
```

2. Test the setup:
   - Visit `https://jitsi.example.com`
   - Create a new meeting
   - Test audio/video functionality
   - Verify SSL certificate

## Troubleshooting

1. Check logs:
```bash
docker-compose -f docker-compose.jitsi.yml logs
docker-compose -f docker-compose.proxy.yml logs
```

2. Common issues:
   - If SSL doesn't work, ensure ports 80 and 443 are open
   - If meetings don't connect, check JVB logs
   - If authentication fails, verify Prosody configuration

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
mkdir -p ${DATA_DIR}/wordpress/html
mkdir -p ${DATA_DIR}/wordpress/db
chmod -R 755 ${DATA_DIR}/wordpress
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
docker-compose -f docker-compose.wordpress.yml up -d
```

2. Verify the deployment:
```bash
docker-compose -f docker-compose.wordpress.yml ps
```

## SSL Configuration

1. Obtain SSL certificate:
```bash
docker-compose -f docker-compose.proxy.yml run certbot certonly -v \
  --webroot -w /var/www/certbot-challenges \
  -d wordpress.example.com \
  --email your-email@example.com \
  --agree-tos --no-eff-email
```

2. Enable SSL in WordPress configuration:
   - Edit `${DATA_DIR}/config/services-config/wordpress.conf`
   - Uncomment and update SSL certificate paths:
```nginx
ssl_certificate /etc/letsencrypt/live/wordpress.example.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/wordpress.example.com/privkey.pem;
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
docker-compose -f docker-compose.wordpress.yml logs
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
- Reverse proxy configured (see previous sections)
- Environment variables configured (see `nextcloud/env.example`)
- Watchtower installed (for automatic updates)

## Directory Structure Setup

1. Create required directories:
```bash
mkdir -p ${NEXTCLOUD_DATA_DIR}/app
mkdir -p ${NEXTCLOUD_DATA_DIR}/db
mkdir -p ${NEXTCLOUD_DATA_DIR}/redis
mkdir -p ${NEXTCLOUD_DATA_DIR}/fail2ban/jail.d
mkdir -p ${NEXTCLOUD_DATA_DIR}/fail2ban/filter.d
mkdir -p ${NEXTCLOUD_DATA_DIR}/fail2ban/action.d
mkdir -p ${NEXTCLOUD_DATA_DIR}/fail2ban/db
chmod -R 755 ${NEXTCLOUD_DATA_DIR}
```

2. Copy fail2ban configuration files:
```bash
cp nextcloud-jail.conf ${NEXTCLOUD_DATA_DIR}/fail2ban/jail.d/nextcloud.conf
cp nextcloud-filter.conf ${NEXTCLOUD_DATA_DIR}/fail2ban/filter.d/nextcloud.conf
cp iptables-multiport-docker.conf ${NEXTCLOUD_DATA_DIR}/fail2ban/action.d/
```

## Configuration

1. Set up environment variables:
   - Copy `nextcloud/env.example` to `nextcloud/.env`
   - Update the following values:
     - `NEXTCLOUD_VERSION`: Nextcloud version (default: latest)
     - `DB_NAME`: Database name
     - `DB_USER`: Database user
     - `DB_PASSWORD`: Secure database password
     - `ROOT_PASSWORD`: Secure MariaDB root password
     - `REDIS_PASSWORD`: Secure Redis password
     - `PHP_UPLOAD_LIMIT`: Maximum upload size (default: 10G)
     - `TIMEZONE`: Your timezone
     - `PUID` and `PGID`: User/Group IDs for file permissions

## Deployment

1. Start Nextcloud services:
```bash
docker-compose -f docker-compose.nextcloud.yml up -d
```

2. Verify the deployment:
```bash
docker-compose -f docker-compose.nextcloud.yml ps
```

## SSL Configuration

1. Obtain SSL certificate:
```bash
docker-compose -f docker-compose.proxy.yml run certbot certonly -v \
  --webroot -w /var/www/certbot-challenges \
  -d nextcloud.example.com \
  --email your-email@example.com \
  --agree-tos --no-eff-email
```

2. Enable SSL in Nextcloud configuration:
   - Edit `${DATA_DIR}/config/services-config/nextcloud.conf`
   - Uncomment and update SSL certificate paths:
```nginx
ssl_certificate /etc/letsencrypt/live/nextcloud.example.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/nextcloud.example.com/privkey.pem;
```

3. Restart the proxy:
```bash
docker restart nginx_reverse_proxy
```

## Initial Setup

1. Access Nextcloud:
   - Visit `https://nextcloud.example.com`
   - Create an admin account
   - Complete the initial setup

2. Post-installation:
   - Configure trusted domains
   - Set up email server
   - Configure background jobs
   - Install recommended apps

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
docker exec nextcloud_app php occ maintenance:repair
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
docker exec nextcloud_fail2ban fail2ban-client set nextcloud unbanip IP_ADDRESS
```

## Troubleshooting

1. Check container logs:
```bash
docker-compose -f docker-compose.nextcloud.yml logs
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

When moving a Nextcloud instance:
1. Only copy the following directories:
   - `data/`
   - `apps/`
   - `themes/`
2. Preserve the `config.php` file
3. Keep the original `instance-id` as it is unique
4. Update domain and database settings in `config.php`

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
   - Copy `watchtower/env.example` to `watchtower/.env`
   - Configure the following values:
     - `WATCHTOWER_VERSION`: Watchtower version (default: latest)
     - `RESTART_POLICY`: Container restart policy (default: unless-stopped)
     - `SCHEDULE`: Cron expression for update checks (default: 0 0 5 * * *)
     - `TIMEZONE`: Your timezone (default: Europe/Berlin)
     - `WATCHTOWER_CLEANUP`: Remove old images (default: true)
     - `WATCHTOWER_ROLLING_RESTART`: Enable rolling restarts (default: true)

## Deployment

1. Start Watchtower:
```bash
docker-compose -f docker-compose.watchtower.yml up -d
```

2. Verify the deployment:
```bash
docker-compose -f docker-compose.watchtower.yml ps
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

## Updating

1. Check Watchtower logs:
```bash
docker-compose -f docker-compose.watchtower.yml logs
```

2. View update history:
```bash
docker logs watchtower
```

## Troubleshooting

1. Common issues:
   - If updates aren't happening, check the schedule
   - If containers aren't being updated, verify they use the `latest` tag
   - If cleanup isn't working, check disk space
   - If rolling restarts fail, check container dependencies

2. Manual update trigger:
```bash
docker exec watchtower watchtower --run-once
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
docker-compose -f docker-compose.jumphost.yml up -d
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
docker-compose -f docker-compose.jumphost.yml ps
```

2. View logs:
```bash
docker-compose -f docker-compose.jumphost.yml logs
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
docker-compose -f docker-compose.monitoring.yml up -d
```

### Usage

1. Access Grafana: `http://grafana.mydomain.de`
   - Default login: admin/passwort from the env file
   - Add Prometheus data source: `http://prometheus:9090`
   - Add Loki data source: `http://loki:3100`

3. Security monitoring:
```bash
# Check fail2ban status
docker exec logging_fail2ban fail2ban-client status grafana

# View banned IPs
docker exec logging_fail2ban fail2ban-client status
```

