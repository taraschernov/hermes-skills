---
name: n8n-self-host
description: Deploy a self-hosted, auto-updating n8n instance on a personal Linux server via Docker Compose, behind nginx + Let's Encrypt, with a systemd timer for daily image updates. Use when the user asks to install/setup/publish n8n on their own server/VPS, expose it on a subdomain over HTTPS, or make n8n self-updating. Covers the real pitfalls (volume permissions uid 1000, invalid N8N_EMAIL_MODE, broken watchtower on modern Docker API).
---

# n8n Self-Host (Docker + nginx + systemd auto-update)

## When to use
- User wants n8n installed on their own server (not n8n.cloud).
- User wants n8n published on a subdomain with valid HTTPS.
- User wants n8n to auto-update.
- Trigger phrases: "установи n8n", "подними n8n", "n8n на сервере", "self-host n8n".

## Assumptions / inputs needed
- A Linux server (Ubuntu 22.04/24.04 tested) with Docker + docker-compose v2.
- A domain/subdomain with DNS A-record pointing to the server IP.
- nginx + certbot available (or install them).
- Working directory: `{{HOME}}/n8n` (replace `{{HOME}}` with the deploying user's home, e.g. `/home/ubuntu`).
- Domain: `{{domain}}` (e.g. solutions.yapclean.tech). Owner email: `{{email}}`.
- Timezone: `{{TIMEZONE}}` (e.g. Europe/Kyiv).

## Variables (template form — substitute before running)
```
{{domain}}     = your subdomain, e.g. solutions.yapclean.tech
{{SERVER_IP}}  = your VPS public IP, e.g. 123.45.67.89
{{email}}      = owner account email, e.g. admin@{{domain}}
{{HOME}}       = deploying user home, e.g. /home/ubuntu
{{TIMEZONE}}   = server timezone, e.g. Europe/Kyiv
{{subdomain}}  = DNS host label, e.g. solutions
```

## Step-by-step (verified working)

### 1. Prepare
```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER   # re-login after
mkdir -p ~/n8n/data && cd ~/n8n
docker --version && docker-compose --version
```

### 2. Generate secrets
```bash
OWNER_PASS=$(openssl rand -base64 18 | tr -dc 'A-Za-z0-9' | head -c24)
ENC_KEY=$(openssl rand -hex 24)
echo "PASS=$OWNER_PASS ENC=$ENC_KEY"
```

### 3. .env (chmod 600)
Do NOT set `N8N_EMAIL_MODE` at all (valid: '' or 'smtp'; 'enabled' crashes n8n).
```env
N8N_PORT=5678
N8N_PROTOCOL=https
N8N_HOST=n8n.example.com
WEBHOOK_URL=https://n8n.example.com/
N8N_SECURE_COOKIE=false
GENERIC_TIMEZONE=Europe/Kyiv
TZ=Europe/Kyiv
N8N_USER_MANAGEMENT=enabled
N8N_INITIAL_OWNER_EMAIL=you@example.com
N8N_INITIAL_OWNER_PASSWORD=СЮДА_OWNER_PASS
N8N_INITIAL_OWNER_FIRST_NAME=Admin
N8N_INITIAL_OWNER_LAST_NAME=User
N8N_ENCRYPTION_KEY=СЮДА_ENC_KEY
DATA_FOLDER=/home/node/.n8n
```

### 4. docker-compose.yml
- Bind port to `127.0.0.1:5678` only (nginx in front).
- Mount `./data:/home/node/.n8n`.
- `restart: always` + healthcheck on `/healthz`.

### 5. Fix volume ownership (CRITICAL)
Inside image n8n runs as uid:gid 1000:1000 (user `node`). If `./data` is owned by root, n8n throws:
`EACCES: permission denied, open '/home/node/.n8n/config'` and loops on restart.
```bash
sudo chown -R 1000:1000 ~/n8n/data
```

### 6. Launch + wait for healthy
```bash
docker-compose up -d
for i in $(seq 1 25); do
  code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 5 http://127.0.0.1:5678/healthz)
  [ "$code" = "200" ] && { echo HEALTHY; break; }
  sleep 3
done
docker ps   # n8n Up ... (healthy)
```

### 7. nginx + Certbot
- Base server block proxies `http://127.0.0.1:5678` with `Upgrade`/`Connection` headers for WebSocket.
- Optional hardening: block `/\.*`, `*.php`, `*.env`, archive extensions, scanner paths.
- `sudo certbot --nginx -d n8n.example.com` adds 443 + redirect.
- Verify: `curl -sS -o /dev/null -w "%{http_code}\n" https://n8n.example.com/`

### 8. Auto-update — use systemd timer, NOT watchtower
**Pitfall:** classic `containrrr/watchtower:latest` (and `ghcr.io/containrrr/watchtower:latest`) use Docker API 1.25; modern daemon requires >= 1.44 → watchtower loops on restart with:
`client version 1.25 is too old. Minimum supported API version is 1.44`.
Live forks (e.g. beatkindev/watchtower) are unreliable/nonexistent. Use systemd instead.

Create `/home/ubuntu/n8n/update-n8n.sh`:
```bash
#!/bin/bash
set -e
cd /home/ubuntu/n8n
docker-compose pull n8n
docker-compose up -d n8n
```
chmod +x it.

Create `/etc/systemd/system/n8n-update.service` (Type=oneshot, ExecStart=/home/ubuntu/n8n/update-n8n.sh, After/Requires docker.service).

Create `/etc/systemd/system/n8n-update.timer`:
```
[Timer]
OnCalendar=*-*-* 04:17:00
Persistent=true
RandomizedDelaySec=600
Unit=n8n-update.service
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now n8n-update.timer
systemctl list-timers n8n-update.timer   # confirm NEXT is set
```

## Verification checklist
- `curl http://127.0.0.1:5678/healthz` → 200
- `docker ps` → n8n Up (healthy)
- `ls ~/n8n/data` → database.sqlite present
- `curl -sS -o /dev/null -w "%{http_code}\n" https://n8n.example.com/` → 200
- `systemctl list-timers n8n-update.timer` → NEXT not empty

## Common errors → fixes
1. `EACCES ... /home/node/.n8n/config` → `sudo chown -R 1000:1000 ~/n8n/data`
2. `Invalid value for N8N_EMAIL_MODE ... Expected '' | 'smtp'` → remove the var.
3. `watchtower: client version 1.25 is too old` → use systemd timer, drop watchtower.
4. healthz silent 30-60s on first boot → normal (DB migrations running).
5. container Restarting loop → `docker logs n8n --tail 30` → usually #1 or #2.

## Notes
- n8n logs a non-critical warning about Python task runner (internal mode) when no external runner is set; only matters if using Python Code nodes. Add a separate task-runner container if needed.
- Owner credentials survive restarts/updates because N8N_ENCRYPTION_KEY is fixed.
- Save generated OWNER_PASS to a 600-permission local file; never echo it in chat unless asked.
