---
name: ghost-self-host
description: Deploy a self-hosted Ghost CMS instance alongside MySQL 8 on a Linux server via Docker Compose, behind Nginx + Let's Encrypt SSL. Use when the user wants to install/setup/publish Ghost CMS on their own VPS, protect it behind Nginx reverse proxy, and secure it over HTTPS. Covers real pitfalls like loopback port isolation (127.0.0.1:2368) and unattended-upgrades package manager lock.
---

# Ghost CMS Self-Host (Docker + MySQL 8 + Nginx + Let's Encrypt)

## When to use
- User wants Ghost CMS installed on their own server instead of paying for Ghost(Pro).
- User wants Ghost CMS exposed on a domain/subdomain with a valid HTTPS SSL certificate.
- Trigger phrases: "установи ghost", "подними ghost", "ghost cms на сервере", "self-host ghost".

## Assumptions / inputs needed
- A Linux server (Ubuntu 22.04/24.04 tested) with Docker + docker-compose v2.
- A domain/subdomain with DNS A-record pointing to the server IP.
- Nginx + certbot available on the host (or install them).
- Domain: `{{domain}}` (e.g. solutions.yapclean.tech). Owner email: `{{email}}`.
- Server IP: `{{SERVER_IP}}`.
- Database root password: `{{DB_ROOT_PASSWORD}}` (automatically generated or user-provided).
- Database user password: `{{DB_GHOST_PASSWORD}}` (automatically generated or user-provided).

## Variables (template form — substitute before running)
```
{{domain}}           = your subdomain, e.g. solutions.yapclean.tech
{{SERVER_IP}}        = your VPS public IP, e.g. 123.45.67.89
{{email}}            = owner email for SSL alerts, e.g. founder@{{domain}}
{{DB_ROOT_PASSWORD}}  = secure random password for MySQL root
{{DB_GHOST_PASSWORD}} = secure random password for MySQL user 'ghost'
```

## Step-by-step (verified working)

### 1. DNS Pre-validation
Verify that the domain resolves to the server IP before proceeding:
```bash
nslookup {{domain}} 8.8.8.8
# The output must show the public IP of your server
```

### 2. Basic Server Security (UFW)
Harden the server by allowing only SSH, HTTP, and HTTPS:
```bash
sudo apt update
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable  # Confirm with 'y'
```

### 3. Generate Database Secrets
Generate cryptographically secure passwords for MySQL:
```bash
DB_ROOT_PASSWORD=$(openssl rand -base64 18 | tr -dc 'A-Za-z0-9' | head -c24)
DB_GHOST_PASSWORD=$(openssl rand -base64 18 | tr -dc 'A-Za-z0-9' | head -c24)
echo "Root DB Pass: $DB_ROOT_PASSWORD"
echo "Ghost DB Pass: $DB_GHOST_PASSWORD"
```

### 4. Create Workspace and docker-compose.yml
```bash
sudo mkdir -p /opt/ghost
sudo chown -R $USER:$USER /opt/ghost
cd /opt/ghost
```
Write the `docker-compose.yml` config using MySQL 8 and Ghost (port bound to `127.0.0.1:2368` for security).

### 5. Launch Stack
```bash
docker compose up -d
docker compose ps
docker compose logs -f ghost-app
```
Wait for the logs to print `Ghost booted in ...` (database table migrations can take 15-30s on first startup).

### 6. Nginx Reverse Proxy Setup
Create Nginx configuration block `/etc/nginx/sites-available/{{domain}}` (proxying traffic to `http://127.0.0.1:2368`).
```bash
sudo ln -sf /etc/nginx/sites-available/{{domain}} /etc/nginx/sites-enabled/{{domain}}
sudo nginx -t
sudo systemctl reload nginx
```

### 7. Certbot SSL Configuration
Generate Let's Encrypt SSL certificate:
```bash
sudo certbot --nginx -d {{domain}} --non-interactive --agree-tos --email {{email}}
```

### 8. Final Verification
Verify the HTTPS endpoint responds:
```bash
curl -I -k https://{{domain}}
# Expected response is 200 OK or 301 Redirect to login
```

---

## Common errors → fixes
1. **Apt Lock Contention:** If `apt update` fails because of a lock held by unattended-upgrades, do not forcefully kill it. Wait a few minutes or verify if Docker/Nginx/Certbot are already pre-installed.
2. **Heredoc Escaping:** Executing Linux heredocs from Windows PowerShell can corrupt special characters in config files. Write files locally and use SCP to upload them instead.
3. **Cloudflare Validation Error:** If Certbot fails to validate, make sure the domain has Cloudflare Proxy set to **DNS Only** (grey cloud).
