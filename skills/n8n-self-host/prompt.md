# Deploy Self-Hosted n8n (Docker + nginx + systemd auto-update)

You are a Senior DevOps Engineer. Deploy a production-ready, self-hosted n8n instance on a Linux server.

## Variables (substitute before running)
- {{domain}}     = your subdomain, e.g. solutions.yapclean.tech
- {{SERVER_IP}}  = your VPS public IP, e.g. 123.45.67.89
- {{email}}      = owner account email, e.g. admin@{{domain}}
- {{HOME}}       = deploying user home, e.g. /home/ubuntu
- {{TIMEZONE}}   = server timezone, e.g. Europe/Kyiv
- {{subdomain}}  = DNS host label, e.g. solutions

## Hard requirements (do not skip)
1. DNS A-record for {{subdomain}} -> {{SERVER_IP}} must resolve before SSL.
2. UFW: allow OpenSSH + 'Nginx Full' only; enable firewall.
3. Docker + docker-compose v2 installed; user in docker group.
4. Working dir {{HOME}}/n8n; create {{HOME}}/n8n/data.
5. Generate OWNER_PASS (openssl rand, >=8 chars) and N8N_ENCRYPTION_KEY (openssl rand -hex 24). Store in {{HOME}}/n8n/.env, chmod 600.
6. .env MUST contain: N8N_PORT=5678, N8N_PROTOCOL=https, N8N_HOST={{domain}}, WEBHOOK_URL=https://{{domain}}/, N8N_USER_MANAGEMENT=enabled, N8N_INITIAL_OWNER_EMAIL={{email}}, N8N_INITIAL_OWNER_PASSWORD=<generated>, N8N_ENCRYPTION_KEY=<generated>, DATA_FOLDER=/home/node/.n8n. DO NOT set N8N_EMAIL_MODE.
7. docker-compose.yml: image n8nio/n8n:latest, restart: always, ports "127.0.0.1:5678:5678" (localhost ONLY), volume {{HOME}}/n8n/data:/home/node/.n8n, healthcheck on /healthz.
8. CRITICAL: sudo chown -R 1000:1000 {{HOME}}/n8n/data BEFORE first `docker compose up -d` (n8n runs as uid 1000; wrong ownership = EACCES crash loop).
9. Wait for /healthz = 200 (30-60s first boot is normal DB migration).
10. Nginx server block: proxy_pass http://127.0.0.1:5678, WebSocket Upgrade/Connection headers, proxy_buffering off, 3600s timeouts, hardening location blocks denying .env/.php/archive/scanner paths.
11. certbot --nginx -d {{domain}} --non-interactive --agree-tos --email {{email}}.
12. Auto-update: create {{HOME}}/n8n/update-n8n.sh (docker compose pull && up -d) + systemd units n8n-update.{service,timer} (daily). DO NOT use watchtower (incompatible with Docker API >= 1.44 on modern daemons).
13. Verify: curl /healthz = 200, curl https://{{domain}}/ = 200, docker ps shows n8n healthy, systemctl list-timers n8n-update.timer NEXT scheduled.
14. Report owner login email and the .env password file path. Never print the password.

## Reference files
This skill ships ready-to-use configs in ./references/:
- docker-compose.yml
- nginx-n8n.conf
- n8n-update.service
- n8n-update.timer
Read them and adapt paths to {{HOME}}/{{domain}} before writing to the server.
