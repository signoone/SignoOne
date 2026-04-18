# Nextcloud Deployment on AWS Lightsail + Cloudflare
> Documented from a real deployment. Everything here was tested and confirmed working.

---

## Stack
- **Server:** AWS Lightsail — Ubuntu 22.04 LTS, $10/mo (2GB RAM, 2 vCPU, 60GB SSD)
- **Nextcloud:** 32.x (stable Docker image)
- **Database:** MariaDB 10.11
- **Cache/Locking:** Redis
- **Reverse Proxy + SSL:** Caddy (auto SSL via Let's Encrypt)
- **DNS:** Cloudflare (DNS only mode)
- **Domain:** cloud.yourdomain.com (example — replace with yours)

---

## Part 1: AWS Lightsail Setup

### 1.1 Create Instance
1. Go to [lightsail.aws.amazon.com](https://lightsail.aws.amazon.com)
2. Create instance → **OS Only → Ubuntu 22.04 LTS**
3. Plan: **$10/mo** (2GB RAM minimum — $5 plan will OOM)
4. Download key pair — you cannot get it again
5. Name it `nextcloud-prod` → Create

### 1.2 Attach Static IP
- Networking tab → Create static IP → Attach to instance
- Note the IP — you'll need it for Cloudflare

### 1.3 Firewall Rules
In the instance Networking tab, set these IPv4 rules:

| Protocol | Port | Source |
|----------|------|--------|
| SSH | 22 | Your public IP only |
| HTTP | 80 | Any |
| HTTPS | 443 | Any |

> Find your public IP by running `curl ifconfig.me` or Googling "what is my ip"

---

## Part 2: Cloudflare DNS

1. Cloudflare → your domain → **DNS → Records**
2. Add A record:
   - **Name:** `nc` (or whatever subdomain)
   - **IPv4:** Your Lightsail static IP
   - **Proxy:** 🔴 Grey cloud (DNS only) — NOT proxied yet
3. Verify propagation:
```bash
dig nc.yourdomain.com +short
# Must return your Lightsail IP
```

---

## Part 3: Server Setup

### 3.1 SSH In
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

## Part 4: Create Nextcloud Directory
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
      - NEXTCLOUD_TRUSTED_DOMAINS=nc.yourdomain.com
      - APACHE_DISABLE_REWRITE_IP=1
      - TRUSTED_PROXIES=172.18.0.0/16
      - OVERWRITEPROTOCOL=https
      - OVERWRITECLIURL=https://nc.yourdomain.com
      - OVERWRITEHOST=nc.yourdomain.com
    depends_on:
      - db
      - redis

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

> Replace all `CHANGE_THIS_*` and `nc.yourdomain.com` with your actual values before saving.

**Why `TRUSTED_PROXIES=172.18.0.0/16`:**
Docker assigns containers IPs in this range internally. Nextcloud needs to trust Caddy's forwarded headers (real client IP, HTTPS protocol). Without this you get login redirect loops and "Forwarded for headers" security errors.

---

## Part 6: Caddyfile

```bash
nano ~/nextcloud/Caddyfile
```

```
nc.yourdomain.com {
    reverse_proxy app:80

    header {
        Strict-Transport-Security "max-age=15552000; includeSubDomains"
        X-Content-Type-Options nosniff
        X-Frame-Options SAMEORIGIN
        X-Permitted-Cross-Domain-Policies none
        Referrer-Policy no-referrer
    }
}
```

> Note: `reverse_proxy app:80` — not `app:8080`. The `8080:80` in compose is host:container mapping. Inside Docker the app is always on port 80.

---

## Part 7: Launch

```bash
docker compose up -d
```

Verify all 4 containers are running:
```bash
docker compose ps
```

You should see: `app`, `db`, `redis`, `caddy` — all Up.

Watch first-boot logs (takes 2-3 mins):
```bash
docker compose logs -f app
```

Wait for: `Nextcloud was successfully installed` and `Apache configured -- resuming normal operations`

---

## Part 8: Enable Cloudflare Proxy (Now Safe)

1. Cloudflare → DNS → flip record to 🟠 **Orange cloud (Proxied)**
2. **SSL/TLS → Overview** → Set to **Full (Strict)** — NOT Flexible
3. **Speed → Optimization** → Disable **Rocket Loader**
4. **Speed → Optimization** → Disable **Auto Minify** (JS/CSS/HTML)

---

## Part 9: Post-Installation Fixes

### Fix Security Warnings

**Mimetype migrations:**
```bash
docker exec -u www-data nextcloud-app-1 php occ maintenance:repair --include-expensive
```

**Maintenance window (set to 1am UTC = 3am Dubai):**
```bash
docker exec -u www-data nextcloud-app-1 php occ config:system:set maintenance_window_start --type=integer --value=23
```

**Default phone region:**
```bash
docker exec -u www-data nextcloud-app-1 php occ config:system:set default_phone_region --value="AE"
```

**Disable Richdocuments (Nextcloud Office) if not using Collabora:**
```bash
docker exec -u www-data nextcloud-app-1 php occ app:disable richdocuments
```

### Set Up Cron Jobs

```bash
crontab -e
```

Add:
```
*/5 * * * * docker exec nextcloud-app-1 php -f /var/www/html/cron.php
```

Then in Nextcloud admin UI:
**Settings → Administration → Basic Settings → Background Jobs → select Cron**

---

## Part 10: Ongoing Maintenance

### Useful Commands

```bash
# Stop everything
docker compose down

# Update Nextcloud to latest stable
docker compose pull && docker compose up -d

# View app logs
docker compose logs -f app

# View Caddy logs
docker compose logs caddy

# Check disk usage
df -h

# Run occ commands
docker exec -u www-data nextcloud-app-1 php occ <command>
```

### Backup Database
```bash
docker exec nextcloud-db-1 mysqldump -u nextcloud -p nextcloud > nextcloud-backup.sql
```

---

## Known Limitations on $10/mo Plan

- **Nextcloud Office / Collabora** does not work — requires dedicated ~2GB RAM minimum. Upgrade to $20/mo (4GB RAM) if you need in-browser document editing.
- **OnlyOffice** has the same RAM constraint.
- For personal use, sync files via the Nextcloud desktop/mobile app and edit locally instead.

---

## Common Mistakes

| Mistake | Consequence |
|--------|-------------|
| Cloudflare proxy ON with SSL set to "Flexible" | Infinite redirect loop |
| `reverse_proxy app:8080` in Caddyfile | 502 error — app is on port 80 inside Docker |
| Missing `TRUSTED_PROXIES` env var | Login loop, security warnings |
| Missing `OVERWRITEPROTOCOL=https` | Nextcloud thinks it's on HTTP |
| Skipping cron job setup | Background tasks never run, silent breakage |
| Leaving SSH port 22 open to `0.0.0.0/0` | Brute force risk |
| Weak passwords on a public server | Don't do this in production |
