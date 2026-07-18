# Deploying a Production-Ready, Self-Hosted Ghost CMS with Docker and MySQL 8: A Secure, Zero-SaaS-Fee Publishing Platform

In the modern digital landscape, content is one of your most valuable business assets. Yet, many organizations find themselves trapped in one of two extremes: wrestling with the bloat, security vulnerabilities, and slow load times of **WordPress**, or paying exorbitant fees ($30 to over $100/month) for managed **Ghost(Pro)** SaaS hosting with arbitrary pageview limits and restricted custom integrations.

There is a better way. 

By deploying a self-hosted, containerized instance of **Ghost CMS** on a clean Linux VPS, you unlock full content ownership, pristine performance, and zero recurring SaaS platform fees. 

This technical guide walks through the exact process used to deploy a production-ready Ghost CMS with a dedicated **MySQL 8** database, secured behind an **Nginx Reverse Proxy** and protected by a hardened **UFW firewall** and **Let's Encrypt SSL**.

---

## The Architecture: Why This Stack?

Rather than running Ghost directly on the host operating system—which can lead to node version conflicts and migration headaches—we containerize the core application and the database. 

Here is our production-ready blueprint:
*   **Host OS:** Ubuntu 22.04 / 24.04 LTS (stable, lightweight, and standard).
*   **Containers (Docker):** Ghost CMS (lightweight Alpine build) isolated alongside MySQL 8.
*   **Database:** MySQL 8.0 (the official database engine recommended by Ghost for production).
*   **Reverse Proxy:** Nginx installed on the host. It intercepts incoming HTTP/HTTPS traffic and forwards it to Docker, handling SSL termination.
*   **Firewall (UFW):** Rules configured to drop all incoming packets except those bound to SSH (22) and Nginx (80/443).

---

## Step-by-Step Deployment Guide

### Step 1: DNS Configuration
Before touching the server command line, we map the subdomain to our VPS. 

In your DNS provider panel (e.g., Cloudflare), create an **A Record**:
*   **Type:** `A`
*   **Name/Host:** `solutions` (points to `<your_domain>`)
*   **Value (IPv4 Address):** `<your_server_IP>` (your VPS public IP)
*   **Proxy Status:** **DNS Only (Disabled)**. 
    > [!IMPORTANT]
    > Disabling Cloudflare’s proxy (orange cloud) during initial setup is critical. Certbot uses HTTP-01 challenges to prove domain ownership. If the proxy is enabled, Cloudflare intercepts the request, which can cause Let’s Encrypt validation to fail. You can safely re-enable the proxy after SSL generation.

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
    sudo apt install -y docker.io docker-compose-v2
    sudo systemctl enable --now docker
    ```

---

### Step 3: Isolated Docker Compose Configuration
We create a directory to house our Ghost stack config and volume directories, keeping files organized under `/opt`.

1.  **Set up directories:**
    ```bash
    sudo mkdir -p /opt/ghost
    sudo chown -R $USER:$USER /opt/ghost
    cd /opt/ghost
    ```

2.  **Create the `docker-compose.yml` file:**
    We configure MySQL 8 and Ghost to run in the same virtual network.
    
    ```yaml
    version: '3.8'

    services:
      ghost-db:
        image: mysql:8.0
        container_name: ghost-db
        restart: always
        environment:
          MYSQL_ROOT_PASSWORD: <SECURE_GENERATED_PASSWORD>
          MYSQL_DATABASE: ghost_prod
          MYSQL_USER: ghost
          MYSQL_PASSWORD: <SECURE_GENERATED_PASSWORD>
        volumes:
          - ghost-db-data:/var/lib/mysql
        command: --default-authentication-plugin=mysql_native_password

      ghost-app:
        image: ghost:5-alpine
        container_name: ghost-app
        restart: always
        depends_on:
          - ghost-db
        ports:
          # CRITICAL SECURITY RULE: Bind to localhost loopback only
          - "127.0.0.1:2368:2368"
        environment:
          url: https://<your_domain>
          database__client: mysql
          database__connection__host: ghost-db
          database__connection__user: ghost
          database__connection__password: <SECURE_GENERATED_PASSWORD>
          database__connection__database: ghost_prod
        volumes:
          - ghost-app-content:/var/lib/ghost/content

    volumes:
      ghost-db-data:
      ghost-app-content:
    ```

    > [!TIP]
    > **Architectural Security Highlight:** Notice that the ports configuration for `ghost-app` is written as `"127.0.0.1:2368:2368"`. Standard Docker guides suggest `"2368:2368"`, which binds port 2368 to all network interfaces. Doing this exposes Node.js directly to the internet, bypassing Nginx security headers and access logs. By binding strictly to `127.0.0.1`, we guarantee the container is only reachable via Nginx.

3.  **Start the containers:**
    ```bash
    docker compose up -d
    ```
    *(Ghost will automatically run the database migrations and create its internal table structure. You can monitor progress with `docker compose logs -f ghost-app`)*.

---

### Step 4: Nginx Reverse Proxy Setup
Now we install Nginx on the host. It acts as the gateway, receiving HTTPS traffic and proxying it over the loopback interface to our Docker container on port 2368.

1.  **Install Nginx:**
    ```bash
    sudo apt install -y nginx
    sudo systemctl enable --now nginx
    ```

2.  **Configure the virtual host:**
    Create a new file at `/etc/nginx/sites-available/<your_domain>`:
    ```nginx
    server {
        listen 80;
        listen [::]:80;
        server_name <your_domain>;

        access_log /var/log/nginx/<your_domain>.access.log;
        error_log /var/log/nginx/<your_domain>.error.log;

        location / {
            proxy_pass http://127.0.0.1:2368;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_buffering off;
        }
    }
    ```

3.  **Activate configuration and reload Nginx:**
    ```bash
    sudo ln -sf /etc/nginx/sites-available/<your_domain> /etc/nginx/sites-enabled/<your_domain>
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
    sudo certbot --nginx -d <your_domain> --non-interactive --agree-tos --email <your_email>
    ```
    *Certbot automatically retrieves the certificates, modifies the Nginx config to load them, and writes a global HTTP-to-HTTPS redirect rule.*

---

## Real-World Lessons & Troubleshooting

During real-world deployments, infrastructure setup rarely goes perfectly. Here are three critical troubleshooting scenarios to keep in mind:

1.  **Apt Lock Contention:** If you encounter `Could not get lock /var/lib/apt/lists/lock. It is held by process...`, it means Ubuntu's `unattended-upgrades` is updating packages in the background. Rather than killing the task (which can corrupt your package manager), wait a few minutes, or proceed with Docker setup directly if your VPS already has Docker and Nginx preinstalled.
2.  **Terminal Escaping Issues:** When executing SSH scripts containing EOF heredocs from a Windows client (PowerShell) to a Linux VPS, special characters in ports or environment variables can become corrupted. To avoid formatting issues, always write config files locally and transfer them using SCP:
    ```powershell
    scp docker-compose.yml ubuntu@<your_server_IP>:/opt/ghost/docker-compose.yml
    ```
3.  **Updating Certbot Notifications:** If you ever need to change the administrative email associated with your SSL certificates without reinstalling or wasting Let's Encrypt API limit tokens, execute:
    ```bash
    sudo certbot update_account --email <your_email> --no-eff-email
    ```

---

## Bonus: The AI Agent Prompt

If you are using an AI agent (such as Claude, Cursor, or an agentic coding assistant) to manage your server infrastructure, you can feed it this system prompt. This equips the agent with the precise rules and steps to deploy this Ghost CMS stack autonomously:

```markdown
# Role: Automated Ghost CMS Deployer

You are an expert systems administrator. Your task is to deploy a production-ready Ghost CMS instance on an Ubuntu VPS using Docker Compose, MySQL 8, Nginx, and Certbot.

## Variables Required:
- `DOMAIN` (e.g., <your_domain>)
- `EMAIL` (e.g., <your_email>)
- `SERVER_IP` (e.g., <your_server_IP>)

## Execution Protocol:

1. **DNS Validation:** Run `nslookup {{domain}} 8.8.8.8`. Ensure it resolves to `{{SERVER_IP}}` before modifying any files.
2. **Environment Audit:** SSH into the server and run `docker --version`, `docker compose version`, and `nginx -v`. Ensure prerequisites are met.
3. **Generate Secrets:** Create two separate, secure 32-character database passwords (MySQL Root and Ghost User).
4. **Deploy Stack Configuration:**
   - Create directories: `mkdir -p /opt/ghost`.
   - Write `/opt/ghost/docker-compose.yml`. Ensure the Ghost container port is bound strictly to `127.0.0.1:2368:2368`.
   - Start the stack: `cd /opt/ghost && docker compose up -d`. Verify the containers boot successfully.
5. **Nginx Reverse Proxy:**
   - Write Nginx config to `/etc/nginx/sites-available/{{domain}}`.
   - Symlink to `sites-enabled`.
   - Test configuration via `nginx -t` and reload Nginx.
6. **SSL Setup:** Execute `certbot --nginx -d {{domain}} --non-interactive --agree-tos --email {{email}}`.
7. **Verification:** Confirm the site returns HTTP `200 OK` or a `301 Redirect` by running `curl -I -k https://{{domain}}`.
```

## Need a Hand? Get a "Done-For-You" Turnkey Deployment

Setting up secure infrastructure takes time and requires ongoing maintenance to get right. If you're a business owner or agency who wants this exact Ghost CMS setup (or an n8n automation stack) but doesn't want to touch a Linux terminal—I can do it for you.

I offer a turnkey deployment service. I will build your server, secure the firewall, install the software, set up SSL, and hand over the master passwords so you retain 100% ownership.

👉 **[Let's discuss your infrastructure on Upwork](https://www.upwork.com/freelancers/~0135ef1aec6d972a2d)**
