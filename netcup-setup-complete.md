
# 🧩 Netcup VPS Setup: Discourse, Mattermost, Vaultwarden

## ✅ Overview

| App         | Domain                   | Port (Internal) | Reverse Proxy | Notes                            |
|-------------|--------------------------|------------------|----------------|----------------------------------|
| Discourse   | forum.suryakanya.com     | 8080             | Internal Nginx | Docker-managed, full-stack       |
| Mattermost  | mm.suryakanya.com        | 8065             | Host Nginx     | Self-managed Docker              |
| Vaultwarden | pam.suryakanya.com       | 8081             | Host Nginx     | Self-managed Docker              |

---

## 🧰 Step 1: Update & Install Essentials

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl git ufw -y

# Install Host Nginx
sudo apt install nginx certbot python3-certbot-nginx -y

# Docker installation (Ubuntu 24.04 safe method)
sudo apt remove docker docker.io containerd containerd.io runc -y
sudo apt install ca-certificates curl gnupg lsb-release -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

Enable firewall:

```bash
sudo ufw allow OpenSSH
sudo ufw allow http
sudo ufw allow https
sudo ufw enable
```

---

## 🐳 Step 2: Discourse Setup (forum.suryakanya.com)

```bash
# Clean previous install (if any)
cd /var
rm -rf /var/discourse

# Clone and install
git clone https://github.com/discourse/discourse_docker.git /var/discourse
cd /var/discourse
./discourse-setup
```

After setup, modify ports in `containers/app.yml`:

```yaml
expose:
  - "8080:80"
```

Then rebuild:

```bash
./launcher stop app
./launcher rebuild app
```

---

## 🌐 Step 3: Enable Host Nginx

```bash
sudo systemctl restart nginx
```

---

## 📦 Step 4: Mattermost & Vaultwarden Docker Setup

```bash
mkdir -p ~/apps && cd ~/apps
```

Create `docker-compose.yml`:

```yaml
version: "3.7"

services:
  mattermost:
    image: mattermost/mattermost-team-edition
    container_name: mattermost
    restart: always
    ports:
      - "8065:8065"
    volumes:
      - mattermost_data:/mattermost

  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    ports:
      - "8081:80"
    volumes:
      - vaultwarden_data:/data

volumes:
  mattermost_data:
  vaultwarden_data:
```

Start containers:

```bash
docker compose up -d
```

---

## 🌐 Step 5: Nginx Reverse Proxy

### Mattermost (mm.suryakanya.com)

```nginx
server {
    listen 80;
    server_name mm.suryakanya.com;

    location / {
        proxy_pass http://localhost:8065;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Vaultwarden (pam.suryakanya.com)

```nginx
server {
    listen 80;
    server_name pam.suryakanya.com;

    location / {
        proxy_pass http://localhost:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable and reload:

```bash
sudo ln -s /etc/nginx/sites-available/mm.suryakanya.com /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/pam.suryakanya.com /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## 🔐 Step 6: SSL with Certbot

```bash
sudo certbot --nginx -d mm.suryakanya.com -d pam.suryakanya.com
```

---

## 🔁 Step 7: Redirect Fixes

### Mattermost:
System Console → Environment → Web Server:
- Set **Site URL** = `https://mm.suryakanya.com`

### Vaultwarden:
Set `WEB_VAULTWARDEN_DOMAIN=https://pam.suryakanya.com`

---

## ✅ Setup Complete
