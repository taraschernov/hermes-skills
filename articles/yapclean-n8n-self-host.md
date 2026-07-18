# Deploying a Production-Ready, Self-Hosted n8n with Docker: A Secure, Zero-SaaS-Fee Automation Platform

In the modern digital landscape, workflow automation is one of your most valuable business杠杆s. Yet, many organizations find themselves trapped in one of two extremes: wrestling with brittle, per-seat SaaS contracts that silently inflate their OpEx, or accepting arbitrary execution limits and data-residency blind spots from third-party automation vendors.

There is a better way.

By deploying a self-hosted, containerized instance of **n8n** on a clean Linux VPS, you unlock full data ownership, unlimited workflows, and zero recurring platform fees. 

This technical guide walks through the exact process used to deploy a production-ready n8n automation engine with a hardened **Nginx Reverse Proxy**, protected by a **UFW firewall** and **Let's Encrypt SSL**, plus a native **systemd timer** for unattended daily updates.

---

## The Architecture: Why This Stack?

Rather than running n8n directly on the host operating system—which leads to node version conflicts, manual upgrades, and migration headaches—we containerize the application and let the host handle only the edge: TLS and routing.

Here is our production-ready blueprint:
*   **Host OS:** Ubuntu 22.04 / 24.04 LTS (stable, lightweight, and standard).
*   **Containers (Docker):** n8n (official `n8nio/n8n` image) isolated in its own bridge network.
*   **Reverse Proxy:** Nginx installed on the host. It intercepts incoming HTTP/HTTPS traffic and forwards it to Docker over the loopback interface, handling SSL termination and WebSocket upgrades.
*   **TLS:** Let's Encrypt (Certbot) for free, auto-renewing certificates.
*   **Auto-Update:** A native `systemd` timer (not a third-party container) that pulls the latest image and redeploys daily.
*   **Firewall (UFW):** Rules configured to drop all incoming packets except those bound to SSH (22) and Nginx (80/443).

---

## Step-by-Step Deployment Guide

### Step 1: DNS Configuration
Before touching the server command line, we map the subdomain to our VPS.

In your DNS provider panel (e.g., Cloudflare), create an **A Record**:
*   **Type:** `A`
*   **Name/Host:** `solutions` (points to `solutions.yapclean.tech`)
*   **Value (IPv4 Address):** `123.45.67.89` (your VPS public IP)
*   **Proxy Status:** **DNS Only (Disabled)**.
    > [!IMPORTANT]
    > Disabling Cloudflare's proxy (orange cloud) during initial setup is critical. Certbot uses HTTP-01 challenges to prove domain ownership. If the proxy is enabled, Cloudflare intercepts the request, which can cause Let's Encrypt validation to fail. You can safely re-enable the proxy after SSL generation.

---

### Step 2: VPS Security & Firewall Hardening
With the DNS pointing to the server, we log in via SSH and lock down the host. A clean VPS should not expose unnecessary ports to the public internet.

1.  **Update package lists:**
    ```bash
    sudo apt update
    ```
2.  **Harden the firewall (UFW):**
    We restrict access so only SSH traffic and public web traffic can reach the server.
    ```bash
    sudo ufw allow OpenSSH
    sudo ufw allow 'Nginx Full'
    ```
3.  **Enable the firewall:**
    ```bash
    sudo ufw enable
    ```
    *Confirm the command by typing `y` when prompted.*

4.  **Install Docker (if not present):**
    ```bash
    curl -fsSL https://get.docker.com | sudo sh
    sudo systemctl enable --now docker
    sudo usermod -aG docker $USER
    ```

---

### Step 3: Isolated Docker Compose Configuration
We create a directory to house our n8n stack config and volume directories, keeping files organized under `~/n8n`.

1.  **Set up directories:**
    ```bash
    sudo mkdir -p ~/n8n/data
    sudo chown -R $USER:$USER ~/n8n
    cd ~/n8n
    ```

2.  **Generate secrets:**
    ```bash
    OWNER_PASS=$(openssl rand -base64 18 | tr -dc 'A-Za-z0-9' | head -c24)
    ENC_KEY=$(openssl rand -hex 24)
    echo "PASS=$OWNER_PASS ENC=$ENC_KEY"
    ```

3.  **Create the `.env` file** (chmod 600 — do not commit this):
    ```env
    N8N_PORT=5678
    N8N_PROTOCOL=https
    N8N_HOST=solutions.yapclean.tech
    WEBHOOK_URL=https://solutions.yapclean.tech/
    N8N_SECURE_COOKIE=false
    GENERIC_TIMEZONE=Europe/Kyiv
    TZ=Europe/Kyiv
    N8N_USER_MANAGEMENT=enabled
    N8N_INITIAL_OWNER_EMAIL=admin@solutions.yapclean.tech
    N8N_INITIAL_OWNER_PASSWORD=<SECURE_GENERATED_PASSWORD>
    N8N_INITIAL_OWNER_FIRST_NAME=Admin
    N8N_INITIAL_OWNER_LAST_NAME=User
    N8N_ENCRYPTION_KEY=<SECURE_GENERATED_KEY>
    DATA_FOLDER=/home/node/.n8n
    ```
    > [!TIP]
    > **Do not set `N8N_EMAIL_MODE`.** The only valid values are `''` or `smtp`. Setting it to `enabled` crashes the container on boot with an enum error. Omit it unless you are wiring an SMTP relay.

4.  **Create the `docker-compose.yml` file:**
    ```yaml
    services:
      n8n:
        image: n8nio/n8n:latest
        container_name: n8n
        restart: always
        ports:
          # CRITICAL SECURITY RULE: Bind to localhost loopback only
          - "127.0.0.1:5678:5678"
        env_file:
          - .env
        environment:
          - N8N_PORT=5678
          - N8N_PROTOCOL=https
          - N8N_HOST=solutions.yapclean.tech
          - WEBHOOK_URL=https://solutions.yapclean.tech/
          - GENERIC_TIMEZONE=Europe/Kyiv
          - TZ=Europe/Kyiv
          - N8N_SECURE_COOKIE=false
        volumes:
          - /home/ubuntu/n8n/data:/home/node/.n8n
        healthcheck:
          test: ["CMD", "wget", "--spider", "-q", "http://127.0.0.1:5678/healthz"]
          interval: 30s
          timeout: 10s
          retries: 5
          start_period: 30s
        networks:
          - n8n-net

    networks:
      n8n-net:
        driver: bridge
    ```

    > [!TIP]
    > **Architectural Security Highlight:** Notice the port mapping is `"127.0.0.1:5678:5678"`. Standard Docker guides suggest `"5678:5678"`, which binds to all network interfaces and exposes n8n directly to the internet—bypassing Nginx security headers, rate limiting, and access logs. Binding strictly to `127.0.0.1` guarantees the container is only reachable via Nginx.

5.  **Fix volume ownership (mandatory, see Lessons Learned):**
    Inside the image, n8n runs as UID/GID `1000:1000` (user `node`). The host mount must match or the container will crash on first boot.
    ```bash
    sudo chown -R 1000:1000 ~/n8n/data
    ```

6.  **Start the containers:**
    ```bash
    docker compose up -d
    ```
    *(n8n will automatically run database migrations and create its internal SQLite structure. You can monitor progress with `docker compose logs -f n8n`. Wait for `/healthz` to return `200` before proceeding—this can take 30–60 seconds on first boot.)*

---

### Step 4: Nginx Reverse Proxy Setup
Now we install Nginx on the host. It acts as the gateway, receiving HTTPS traffic and proxying it over the loopback interface to our Docker container on port 5678 (including WebSocket upgrades).

1.  **Install Nginx:**
    ```bash
    sudo apt install -y nginx
    sudo systemctl enable --now nginx
    ```

2.  **Configure the virtual host:**
    Create a new file at `/etc/nginx/sites-available/n8n`:
    ```nginx
    server {
        listen 80;
        listen [::]:80;
        server_name solutions.yapclean.tech;

        client_max_body_size 100m;

        # Hardening: block scanner probes, archives, and secrets exposure
        location ~ /\.(?!well-known/) { return 404; }
        location ~* \.(?:zip|tar|tgz|gz|bz2|xz|zst|7z|rar|sql)(?:/|$) { return 404; }
        location ~* \.php(?:/|$) { return 404; }
        location ~* ^/(?:.*\.env(?:[./_-].*)?|.*phpinfo.*|.*service-account.*\.json|.*gcp.*\.json|.*firebase.*\.json|.*google.*credentials.*\.json|.*keyfile.*\.json|dns-query) {
            return 404;
        }

        location / {
            proxy_pass http://127.0.0.1:5678;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;
            proxy_buffering off;
            proxy_read_timeout 3600s;
            proxy_send_timeout 3600s;
        }
    }
    ```

3.  **Activate configuration and reload Nginx:**
    ```bash
    sudo ln -sf /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/n8n
    sudo nginx -t
    sudo systemctl reload nginx
    ```

---

### Step 5: SSL Generation with Certbot
Let's secure the connection. We request a free SSL/TLS certificate from Let's Encrypt using Certbot.

1.  **Install Certbot and the Nginx plugin:**
    ```bash
    sudo apt install -y certbot python3-certbot-nginx
    ```

2.  **Generate the certificate:**
    We run this non-interactively (ideal for automation or deployment scripts):
    ```bash
    sudo certbot --nginx -d solutions.yapclean.tech --non-interactive --agree-tos --email admin@solutions.yapclean.tech
    ```
    *Certbot automatically retrieves the certificates, modifies the Nginx config to load them, and writes a global HTTP-to-HTTPS redirect rule.*

---

### Step 6: Unattended Daily Updates via systemd (No Watchtower)
Rather than relying on a third-party auto-update container, we use a native `systemd` timer. This is more reliable on modern Docker daemons (see Lesson 3).

1.  **Create the update script** `/home/ubuntu/n8n/update-n8n.sh`:
    ```bash
    #!/bin/bash
    set -e
    cd /home/ubuntu/n8n
    docker compose pull n8n
    docker compose up -d n8n
    ```
    ```bash
    sudo chmod +x /home/ubuntu/n8n/update-n8n.sh
    ```

2.  **Create the service unit** `/etc/systemd/system/n8n-update.service`:
    ```ini
    [Unit]
    Description=n8n auto-update (docker-compose pull + up)
    After=docker.service
    Requires=docker.service

    [Service]
    Type=oneshot
    ExecStart=/home/ubuntu/n8n/update-n8n.sh
    ```

3.  **Create the timer unit** `/etc/systemd/system/n8n-update.timer`:
    ```ini
    [Unit]
    Description=Run n8n auto-update daily

    [Timer]
    OnCalendar=*-*-* 04:17:00
    Persistent=true
    RandomizedDelaySec=600
    Unit=n8n-update.service

    [Install]
    WantedBy=timers.target
    ```

4.  **Enable and start the timer:**
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable --now n8n-update.timer
    ```
    *Confirm scheduling with `systemctl list-timers n8n-update.timer`.*

---

## Real-World Lessons & Troubleshooting

During real-world deployments, infrastructure setup rarely goes perfectly. Here are the critical troubleshooting scenarios we hit and resolved on this exact build:

1.  **Volume Permission Crash (EACCES):** On first boot the container failed with `Error: EACCES: permission denied, open '/home/node/.n8n/config'`. The n8n image runs as UID/GID `1000:1000`, but the host mount was owned by `root`. The fix is mandatory and non-negotiable: `sudo chown -R 1000:1000 ~/n8n/data` **before** the first `docker compose up`.
2.  **Invalid Environment Variable:** Setting `N8N_EMAIL_MODE=enabled` produced `Invalid value for N8N_EMAIL_MODE - Invalid enum value. Expected '' | 'smtp'`. The container refused to start. Fix: omit the variable entirely unless configuring an SMTP relay.
3.  **Watchtower Incompatibility:** We initially tried the canonical `containrrr/watchtower` container for auto-updates. It entered an infinite restart loop with `client version 1.25 is too old. Minimum supported API version is 1.44`. The image was built against an ancient Docker API; modern daemons (29.x) reject it. Live community forks proved unreliable. Fix: drop watchtower, use the native `systemd` timer in Step 6.
4.  **First-Boot Latency is Normal:** `/healthz` returns non-200 for 30–60 seconds on first start. This is database migration, not failure. Poll with retries; do not assume the container is broken.

---

## Bonus: The AI Agent Prompt

If you are using an AI agent (such as Claude, Cursor, or an agentic coding assistant) to manage your server infrastructure, you can feed it this system prompt. This equips the agent with the precise rules and steps to deploy this n8n stack autonomously:

```markdown
# Role: Automated n8n Deployer

You are an expert systems administrator. Your task is to deploy a production-ready, self-hosted n8n instance on an Ubuntu VPS using Docker Compose, Nginx, and Certbot, with a systemd timer for daily image updates.

## Variables Required:
- `DOMAIN` (e.g., solutions.yapclean.tech)
- `EMAIL` (e.g., admin@solutions.yapclean.tech)
- `SERVER_IP` (e.g., 123.45.67.89)

## Execution Protocol:

1. **DNS Validation:** Run `nslookup {{domain}} 8.8.8.8`. Ensure it resolves to `{{SERVER_IP}}` before modifying any files.
2. **Environment Audit:** SSH into the server and run `docker --version`, `docker compose version`, and `nginx -v`. Ensure prerequisites are met.
3. **Generate Secrets:** Create a secure owner password (openssl rand) and a stable N8N_ENCRYPTION_KEY.
4. **Deploy Stack Configuration:**
   - Create directories: `mkdir -p ~/n8n/data`.
   - Write `~/n8n/.env` (chmod 600) and `~/n8n/docker-compose.yml`. Ensure the n8n container port is bound strictly to `127.0.0.1:5678:5678`.
   - CRITICAL: `sudo chown -R 1000:1000 ~/n8n/data` before first launch (n8n runs as uid 1000).
   - Do NOT set N8N_EMAIL_MODE.
   - Start the stack: `cd ~/n8n && docker compose up -d`. Verify the container boots and `/healthz` returns 200.
5. **Nginx Reverse Proxy:**
   - Write Nginx config to `/etc/nginx/sites-available/n8n` with WebSocket Upgrade/Connection headers, proxy_buffering off, 3600s timeouts, and hardening location blocks denying .env/.php/archive/scanner paths.
   - Symlink to `sites-enabled`.
   - Test configuration via `nginx -t` and reload Nginx.
6. **SSL Setup:** Execute `certbot --nginx -d {{domain}} --non-interactive --agree-tos --email {{email}}`.
7. **Auto-Update:** Create `/home/ubuntu/n8n/update-n8n.sh` (docker compose pull && up -d) and systemd units `n8n-update.{service,timer}`; enable the timer. DO NOT use watchtower (incompatible with Docker API >= 1.44).
8. **Verification:** Confirm the site returns HTTP `200 OK` via `curl -I -k https://{{domain}}` and that the systemd timer NEXT is scheduled.
```

---

## Hand This Skill to Your AI Agent

You do not need to run a single command yourself. If you already use a terminal-capable AI agent (Claude, Cursor, a coding assistant, or a self-hosted agent like Hermes), paste the block below into its context. The agent will deploy the entire stack correctly—including the security bindings and the systemd auto-update timer—without the trial-and-error we documented above.

```markdown
# Skill: Deploy Self-Hosted n8n (Docker + nginx + systemd auto-update)

You are a Senior DevOps Engineer. Deploy a production-ready, self-hosted n8n instance.

## Variables (substitute before running)
- {{domain}}     = your subdomain, e.g. solutions.yapclean.tech
- {{SERVER_IP}}  = your VPS public IP, e.g. 123.45.67.89
- {{email}}      = owner account email, e.g. admin@{{domain}}
- {{HOME}}       = deploying user home, e.g. /home/ubuntu
- {{TIMEZONE}}   = server timezone, e.g. Europe/Kyiv
- {{subdomain}}  = DNS host label, e.g. solutions

## Hard requirements (do not skip)
1. DNS A-record for {{subdomain}} → {{SERVER_IP}} must resolve before SSL.
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
```

That is the complete, field-tested procedure. Hand it to your agent and it will build the server, secure the edge, install n8n, issue SSL, and wire unattended updates—correctly, the first time.

---

## Get the Reusable Skill

Everything in this article is also packaged as a portable, open-source skill you can drop into your own agent or fork for your team. No copy-paste from the post required.

**Repository:** [github.com/taraschernov/hermes-skills](https://github.com/taraschernov/hermes-skills)

The `n8n-self-host` skill folder ships:
- `SKILL.md` — Hermes/Claude-style skill with frontmatter (auto-loads on matching requests).
- `prompt.md` — a self-contained deployment prompt for *any* chat agent (no plugin needed).
- `references/` — ready-to-adapt configs: `docker-compose.yml`, `nginx-n8n.conf`, `n8n-update.service`, `n8n-update.timer`.

### Add it to your agent

**If you use Hermes (or a Claude-style skills system):**
```bash
cp -r skills/n8n-self-host ~/.hermes/skills/devops/n8n-self-host
```
The agent loads it automatically when you ask to deploy or self-host n8n.

**If you use any other chat agent (Claude, Cursor, a coding assistant, etc.):**
Open `skills/n8n-self-host/prompt.md` from the repo and paste its contents into the chat. It is a complete, standalone instruction set — no skills plugin required.

**If you have no agent:**
Use the files in `references/` as drop-in configs and follow the steps in this article (or the `articles/` walkthrough in the same repo).

The collection is template-driven: every environment value is a `{{placeholder}}`, and no real secrets, IPs, or emails are committed. MIT licensed — reuse freely.

---

## Ready to Automate Your Infrastructure Without the SaaS Tax?

Setting up secure infrastructure takes time and requires ongoing maintenance to get right. If you're a business owner or agency who wants this exact setup but doesn't want to touch a Linux terminal—I can do it for you. I offer a turnkey deployment service. I will build your server, secure the firewall, install the software, set up SSL, and hand over the master passwords so you retain 100% ownership. 

 👉 [Let's discuss your infrastructure on Upwork](https://www.upwork.com/freelancers/~0135ef1aec6d972a2d)
