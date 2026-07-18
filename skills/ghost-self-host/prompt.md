# Deploy Self-Hosted Ghost CMS (Docker + MySQL 8 + Nginx + Certbot)

You are a Senior DevOps Engineer. Deploy a production-ready, self-hosted Ghost CMS instance on a Linux server.

## Variables (substitute before running)
- {{domain}}           = your subdomain, e.g. solutions.yapclean.tech
- {{SERVER_IP}}        = your VPS public IP, e.g. 123.45.67.89
- {{email}}            = owner email for SSL alerts, e.g. founder@{{domain}}
- {{DB_ROOT_PASSWORD}}  = random password for MySQL root
- {{DB_GHOST_PASSWORD}} = random password for MySQL user 'ghost'

## Hard requirements (do not skip)
1. DNS A-record for {{domain}} -> {{SERVER_IP}} must resolve before SSL setup.
2. UFW: allow OpenSSH + 'Nginx Full' only; enable firewall.
3. Docker + docker-compose v2 installed.
4. Working dir `/opt/ghost`. Create it with write permissions for the deploying user.
5. Generate secure random passwords (>=24 chars) for MySQL root and Ghost user.
6. Write `/opt/ghost/docker-compose.yml` using Ghost 5-alpine and MySQL 8.
7. CRITICAL: Bind the Ghost container port strictly to localhost loopback: `- "127.0.0.1:2368:2368"`. This prevents direct exposure to the public internet.
8. Start the containers (`docker compose up -d`) and wait for the database migrations to finish.
9. Nginx server block: proxy traffic to `http://127.0.0.1:2368`. Link to sites-enabled, verify syntax, and reload Nginx.
10. Certbot SSL: Run `sudo certbot --nginx -d {{domain}} --non-interactive --agree-tos --email {{email}}` to set up HTTPS and redirects.
11. Verify: Confirm the site returns HTTP `200 OK` on `https://{{domain}}`.

## Reference files
This skill ships ready-to-use configs in `./references/`:
- docker-compose.yml
- nginx.conf
Read and adapt them to {{domain}} before deploying on the server.
