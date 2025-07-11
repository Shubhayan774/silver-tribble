
# 🧩 VPS and Web Service Setup Guide
**Kordynamika Solutions LLP**  
**Author:** Shubhayan Mitra  
**Email:** shubhayan774@gmail.com

---

## 🎯 Objective
Set up a self-hosted corporate stack with:

- Discourse on `forum.suryakanya.com` (Docker, internal NGINX)
- Mattermost on `mm.suryakanya.com` and Vaultwarden on `pam.suryakanya.com` (Docker, host NGINX reverse proxy)
- Static websites on `kordynamika.com` and `suryakanya.com` (served via NGINX and Docker)

**VPS Info:** Netcup VPS (Ubuntu 24.04 LTS)  
**IP Address:** `5.45.98.158`

---

## 🔧 Base Server Setup

```bash
apt update && apt upgrade -y
apt install curl git ufw nginx certbot python3-certbot-nginx docker.io docker-compose -y

ufw allow OpenSSH
ufw allow 80,443/tcp
ufw enable

systemctl enable docker
```

---

## 🚀 Discourse Setup (`forum.suryakanya.com`)

### Install & Configure Discourse

```bash
git clone https://github.com/discourse/discourse_docker.git /var/discourse
cd /var/discourse
./discourse-setup
```

In `containers/app.yml`, set:

```yaml
DISCOURSE_HOSTNAME: forum.suryakanya.com
DISABLE_HTTPS: true

expose:
  - "8080:80"
```

### NGINX Reverse Proxy

```nginx
server {
    listen 80;
    server_name forum.suryakanya.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name forum.suryakanya.com;

    ssl_certificate /etc/letsencrypt/live/forum.suryakanya.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/forum.suryakanya.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 🛠 Mattermost & Vaultwarden Setup

### `~/apps/docker-compose.yml`

```yaml
version: '3.8'

services:
  mattermost:
    image: mattermost/mattermost-team-edition
    container_name: mattermost
    ports:
      - "8065:8065"
    environment:
      - MM_SQLSETTINGS_DRIVERNAME=postgres
      - MM_SQLSETTINGS_DATASOURCE=postgres://mmuser:mmuser_password@mattermost_db/mattermost?sslmode=disable
    depends_on:
      - mattermost_db

  mattermost_db:
    image: postgres:13
    environment:
      - POSTGRES_USER=mmuser
      - POSTGRES_PASSWORD=mmuser_password
      - POSTGRES_DB=mattermost

  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    ports:
      - "8081:80"
```

```bash
cd ~/apps
docker compose up -d
```

### NGINX Configuration

**`mm.suryakanya.com`**
```nginx
server {
    listen 80;
    server_name mm.suryakanya.com;
    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl;
    server_name mm.suryakanya.com;

    ssl_certificate /etc/letsencrypt/live/mm.suryakanya.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mm.suryakanya.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://localhost:8065;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**`pam.suryakanya.com`**
```nginx
server {
    listen 80;
    server_name pam.suryakanya.com;
    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl;
    server_name pam.suryakanya.com;

    ssl_certificate /etc/letsencrypt/live/pam.suryakanya.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/pam.suryakanya.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://localhost:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 🌍 Static Sites Setup

### Docker Compose Files

**`~/apps/kordynamika.yml`**
```yaml
version: '3.8'
services:
  kordynamika:
    image: nginx:alpine
    container_name: kordynamika
    ports:
      - "8082:80"
    volumes:
      - ./kordynamika_site:/usr/share/nginx/html:ro
    restart: unless-stopped
```

**`~/apps/suryakanya.yml`**
```yaml
version: '3.8'
services:
  suryakanya:
    image: nginx:alpine
    container_name: suryakanya
    ports:
      - "8083:80"
    volumes:
      - ./suryakanya_site:/usr/share/nginx/html:ro
    restart: unless-stopped
```

### NGINX for `kordynamika.com`

```nginx
server {
    listen 80;
    server_name kordynamika.com www.kordynamika.com;
    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl;
    server_name kordynamika.com www.kordynamika.com;

    ssl_certificate /etc/letsencrypt/live/kordynamika.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/kordynamika.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://localhost:8082;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 🔐 SSL Certificate Issuance

```bash
sudo certbot --nginx -d kordynamika.com -d www.kordynamika.com
sudo certbot --nginx -d mm.suryakanya.com
sudo certbot --nginx -d pam.suryakanya.com
sudo certbot --nginx -d forum.suryakanya.com
```

---

## ✅ Final Checks

```bash
# Validate nginx
sudo nginx -t && sudo systemctl reload nginx

# Check Docker containers
docker ps

# Check port status
curl -I http://localhost:8082
```

---

## ✅ Successfully Deployed

| Service     | Domain                   | Status     |
|-------------|---------------------------|------------|
| Discourse   | forum.suryakanya.com      | ✅ Working |
| Mattermost  | mm.suryakanya.com         | ✅ Working |
| Vaultwarden | pam.suryakanya.com        | ✅ Working |
| Static Site | kordynamika.com           | ✅ Working |
| Static Site | suryakanya.com            | 🔜 Pending |
