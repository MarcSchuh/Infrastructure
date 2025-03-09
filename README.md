# Infrastructure


Steps:
- Ensure all files are present
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

docker network create jitsi_network

~/docker-compose-linux-x86_64 -f docker-compose.jitsi.yml up -d

erst den Teil in der nginx.conf disablen, der die Zertifikate braucht

~/docker-compose-linux-x86_64 -f docker-compose.proxy.yml run --rm certbot certonly --webroot -w /var/www/certbot-challenges -d jitsi.test.marcschuh.de --email your-letsencrypttest@marc-schuh.de --agree-tos --no-eff-email


register user:
docker exec -it jitsi_prosody prosodyctl --config /config/prosody.cfg.lua register youruser auth.jitsi.test.marcschuh.de yourpassword

sudo ufw allow 8000/tcp
sudo ufw allow 8443/tcp

docker exec -it jitsi_prosody prosodyctl --config /config/prosody.cfg.lua register focus auth.jitsi.test.marcschuh.de YOUR_SECURE_FOCUS_PASSWORD

port 10000/udp
and 
      - "5222:5222"
      - "5347:5347"
      - "5280:5280"


for user: docker exec -it jitsi_prosody prosodyctl --config /config/prosody.cfg.lua register marc jitsi.test.marcschuh.de EnpnLUWbmmXV27sVQ 
