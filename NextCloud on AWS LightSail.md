# Nextcloud + Collabora Office Deployment
### AWS Lightsail + Docker + Cloudflare
> Documented from a real deployment. Everything here was tested and confirmed working.
<img width="3840" height="2704" alt="image" src="https://github.com/user-attachments/assets/b446c094-d486-481c-82bb-be1f4ec7c2af" />

 
---
 
## Stack
- **Server:** AWS Lightsail — Ubuntu 22.04 LTS, $24/mo (4GB RAM, 2 vCPU, 80GB SSD)
- **Nextcloud:** 32.x (stable Docker image)
- **Database:** MariaDB 10.11
- **Cache/Locking:** Redis
- **Document Editor:** Collabora Online (CODE)
- **Reverse Proxy + SSL:** Caddy (auto SSL)
- **DNS:** Cloudflare
---
 
## Part 1: AWS Lightsail Setup
 
### 1.1 Create Instance
1. Go to [lightsail.aws.amazon.com](https://lightsail.aws.amazon.com)
2. Create instance → **OS Only → Ubuntu 22.04 LTS**
3. Plan: **$24/mo (4GB RAM)** — required for Nextcloud + Collabora
4. Download key pair — you cannot get it again
5. Name it `nextcloud-prod` → Create
### 1.2 Attach Static IP
- Networking tab → Create static IP → Attach to instance
- Note the IP — needed for Cloudflare and config
### 1.3 Firewall Rules
 
| Protocol | Port | Source |
|----------|------|--------|
| SSH | 22 | Your public IP only |
| HTTP | 80 | Any |
| HTTPS | 443 | Any |
 
> Find your public IP: `curl ifconfig.me`
 
---
 
## Part 2: Cloudflare DNS
 
Add **two** A records — one for Nextcloud, one for Collabora:
 
| Type | Name | IPv4 | Proxy |
|------|------|------|-------|
| A | `cloud` | Your Lightsail IP | 🔴 DNS only |
| A | `office` | Your Lightsail IP | 🔴 DNS only |
 
This gives you:
- `cloud.yourdomain.com` → Nextcloud
- `office.yourdomain.com` → Collabora
Verify both are live before continuing:
```bash
dig cloud.yourdomain.com +short
dig office.yourdomain.com +short
# Both must return your Lightsail IP
```
 
---
 
## Part 3: Server Setup
 
### 3.1 SSH In
 
If you get `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED` (happens after instance upgrade):
```bash
ssh-keygen -f '/home/YOUR_USER/.ssh/known_hosts' -R 'YOUR_SERVER_IP'
```
Then SSH in normally:
```bash
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@YOUR_STATIC_IP
```
 
### 3.2 Update System
```bash
sudo apt update && sudo apt upgrade -y
```
 
### 3.3 Install Docker
```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker ubuntu
newgrp docker
```
 
### 3.4 Install Docker Compose
```bash
sudo apt install -y docker-compose-plugin
docker compose version
```
 
---
 
## Part 4: Create Project Directory
```bash
mkdir ~/nextcloud && cd ~/nextcloud
```
 
---
 
## Part 5: docker-compose.yml
 
```bash
nano ~/nextcloud/docker-compose.yml
```
 
```yaml
services:
  db:
    image: mariadb:10.11
    restart: always
    command: --transaction-isolation=READ-COMMITTED --log-bin=binlog --binlog-format=ROW
    volumes:
      - db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=CHANGE_THIS_ROOT_PASS
      - MYSQL_PASSWORD=CHANGE_THIS_DB_PASS
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
 
  redis:
    image: redis:alpine
    restart: always
 
  app:
    image: nextcloud:stable
    restart: always
    extra_hosts:
      - "office.yourdomain.com:YOUR_LIGHTSAIL_IP"
    ports:
      - 8080:80
    volumes:
      - nextcloud:/var/www/html
    environment:
      - MYSQL_PASSWORD=CHANGE_THIS_DB_PASS
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=db
      - REDIS_HOST=redis
      - NEXTCLOUD_ADMIN_USER=CHANGE_THIS_ADMIN
      - NEXTCLOUD_ADMIN_PASSWORD=CHANGE_THIS_ADMIN_PASS
      - NEXTCLOUD_TRUSTED_DOMAINS=cloud.yourdomain.com
      - APACHE_DISABLE_REWRITE_IP=1
      - TRUSTED_PROXIES=172.18.0.0/16
      - OVERWRITEPROTOCOL=https
      - OVERWRITECLIURL=https://cloud.yourdomain.com
      - OVERWRITEHOST=cloud.yourdomain.com
    depends_on:
      - db
      - redis
 
  collabora:
    image: collabora/code:latest
    restart: always
    environment:
      - aliasgroup1=https://cloud.yourdomain.com
      - DONT_GEN_SSL_CERT=YES
      - extra_params=--o:ssl.enable=false --o:ssl.termination=true
    cap_add:
      - MKNOD
      - SYS_ADMIN
    devices:
      - /dev/fuse
    expose:
      - 9980
 
  caddy:
    image: caddy:alpine
    restart: always
    ports:
      - 80:80
      - 443:443
    volumes:
      - caddy_data:/data
      - caddy_config:/config
      - ./Caddyfile:/etc/caddy/Caddyfile
    depends_on:
      - app
 
volumes:
  db:
  nextcloud:
  caddy_data:
  caddy_config:
```
 
> ⚠️ **Before saving:**
> - Replace all `CHANGE_THIS_*` values with strong passwords
> - Replace `cloud.yourdomain.com` with your actual Nextcloud domain
> - Replace `office.yourdomain.com` with your actual Collabora domain
> - Replace `YOUR_LIGHTSAIL_IP` with your actual static IP
 
---
 
## Part 6: Caddyfile
 
```bash
nano ~/nextcloud/Caddyfile
```
 
```
cloud.yourdomain.com {
    reverse_proxy app:80
 
    header {
        Strict-Transport-Security "max-age=15552000; includeSubDomains"
        X-Content-Type-Options nosniff
        X-Frame-Options SAMEORIGIN
        X-Permitted-Cross-Domain-Policies none
        Referrer-Policy no-referrer
    }
}
 
office.yourdomain.com {
    reverse_proxy collabora:9980
}
```
 
> `reverse_proxy app:80` — not `app:8080`. Inside Docker the app runs on port 80. The `8080:80` in compose is host:container mapping only.
 
> Collabora must have its own subdomain (`office.yourdomain.com`). Path-based routing (e.g. `/collabora/*`) does not work with Collabora — it rejects requests with a path prefix.
 
---
 
## Part 7: Launch
 
```bash
docker compose up -d
```
 
Verify all 5 containers are running:
```bash
docker compose ps
```
 
You should see: `app`, `db`, `redis`, `collabora`, `caddy` — all Up.
 
Watch first-boot logs (takes 2-3 mins):
```bash
docker compose logs -f app

```
 
Wait for:
```
Nextcloud was successfully installed
Apache configured -- resuming normal operations
```

> That's it.
Go to `https://cloud.yourdomain.com` — you should see Nextcloud fully running.
---
 
## Part 8: Enable Cloudflare Proxy
 
Now that SSL is working on the server:
 
1. Cloudflare → DNS → flip **both** records to 🟠 **Orange cloud (Proxied)**
2. **SSL/TLS → Overview** → Set to **Full (Strict)** — NOT Flexible
3. **Speed → Optimization** → Disable **Rocket Loader**
4. **Speed → Optimization** → Disable **Auto Minify** (JS/CSS/HTML)
---
 
## Part 9: Post-Installation Fixes
 
### Security & Config
 
**Mimetype migrations:**
```bash
docker exec -u www-data nextcloud-app-1 php occ maintenance:repair --include-expensive
```
 
**Maintenance window:**
```bash
docker exec -u www-data nextcloud-app-1 php occ config:system:set maintenance_window_start --type=integer --value=23
```
 
**Default phone region:**
```bash
docker exec -u www-data nextcloud-app-1 php occ config:system:set default_phone_region --value="US"
```
 
### Cron Jobs
 
```bash
crontab -e
```
 
Add:
```
*/5 * * * * /usr/bin/docker exec -u www-data nextcloud-app-1 php -f /var/www/html/cron.php
```
 
> ⚠️ Use full `/usr/bin/docker` path — cron doesn't load your shell PATH. Use `-u www-data` — cron.php must run as the file owner (uid 33) or it will fail silently.
 
Then in Nextcloud: **Settings → Administration → Basic Settings → Background Jobs → select Cron**
 
---
 
## Part 10: Configure Collabora in Nextcloud
 
1. Go to **Settings → Administration → Nextcloud Office**
2. Select **Use your own server**
3. Enter: `https://office.yourdomain.com`
4. Click **Save**
You should see:
```
✅ Collabora Online server is reachable.
URL used by the browser: https://office.yourdomain.com
Nextcloud URL used by Collabora: https://cloud.yourdomain.com
```
 
### Fix WOPI Allow-list Warning
 
```bash
docker exec -u www-data nextcloud-app-1 php occ config:app:set richdocuments wopi_allowlist --value="YOUR_LIGHTSAIL_IP"
```
 
---
 
## Part 11: Ongoing Maintenance
 
```bash
# Check all containers
docker compose ps
 
# View Nextcloud logs
docker compose logs -f app
 
# View Caddy logs
docker compose logs caddy
 
# View Collabora logs
docker compose logs collabora
 
# Update everything
docker compose pull && docker compose up -d
 
# Check disk usage
df -h
 
# Run occ commands
docker exec -u www-data nextcloud-app-1 php occ <command>
 
# Backup database
docker exec nextcloud-db-1 mysqldump -u nextcloud -p nextcloud > nextcloud-backup.sql
```
 
---
 
## Common Mistakes
 
| Mistake | Consequence |
|--------|-------------|
| Cloudflare SSL set to "Flexible" | Infinite redirect loop |
| `reverse_proxy app:8080` in Caddyfile | 502 error |
| Missing `TRUSTED_PROXIES` | Login loop, security warnings |
| Missing `OVERWRITEPROTOCOL=https` | Nextcloud thinks it's HTTP |
| Collabora on path route instead of subdomain | 400/404 errors, discovery fails |
| Missing `extra_hosts` for office domain in app | cURL error 6, cannot resolve host |
| Wrong IP in `extra_hosts` (using private AWS IP) | Collabora unreachable from Nextcloud |
| Skipping cron job | Background tasks silently broken |
| SSH port 22 open to `0.0.0.0/0` | Brute force risk |
