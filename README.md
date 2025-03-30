# Infrastructure


Steps:
- Ensure all files are present
22/tcp
80/tcp
443/tcp
10000/udp 
8000/tcp
8443/tcp
3478/udp
5349/udp


Create docker network
docker network create jitsi_network

zuerst in der nginx.conf den Teil disablen, der die Zertifikate braucht. Die kommen im zweiten Schritt

~/docker-compose-linux-x86_64 -f docker-compose.jitsi.yml up -d

Zertifikate anlegen

~/docker-compose-linux-x86_64 -f docker-compose.proxy.yml run --rm certbot certonly --webroot -w /var/www/certbot-challenges -d jitsi.test.mydomain.de --email your-letsencrypttest@mydomain.de --agree-tos --no-eff-email


register user:
docker exec -it jitsi_prosody prosodyctl --config /config/prosody.cfg.lua register youruser auth.jitsi.test.mydomain.de yourpassword

docker exec -it jitsi_prosody prosodyctl --config /config/prosody.cfg.lua register focus auth.jitsi.test.mydomain.de YOUR_SECURE_FOCUS_PASSWORD

for user: docker exec -it jitsi_prosody prosodyctl --config /config/prosody.cfg.lua register marc jitsi.test.mydomain.de SomeUltraSecret
