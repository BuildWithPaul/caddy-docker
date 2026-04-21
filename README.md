# caddy-docker

Caddy reverse proxy in Docker — auto-HTTPS with Let's Encrypt. Production-ready config for exposing Docker services to the internet.

## Why this repo

Running Docker containers is easy. Exposing them securely with HTTPS is not. This repo provides a reusable Caddy setup that:

- **Auto-obtains** Let's Encrypt certificates (no certbot, no cron)
- **Auto-renews** certificates ~30 days before expiry (zero downtime, hot-swap)
- **Reverse-proxies** any number of Docker services on custom subpaths
- **Strips path prefixes** so your apps don't need to know they're behind a proxy

## Architecture

```
Browser → Caddy (80/443, auto-HTTPS) → service-a:5000
                                     → service-b:8080
                                     → service-c:3000
```

Caddy handles TLS termination and path routing. Each app runs in its own container, blissfully unaware of the subpath.

## Quick Start

```bash
git clone https://github.com/BuildWithPaul/caddy-docker.git
cd caddy-docker

# 1. Edit Caddyfile — set your domain and services
# 2. Edit docker-compose.yml — add your app services
# 3. Deploy
docker compose up -d
```

## Project Structure

```
caddy-docker/
├── Caddyfile              # Caddy config — domain, subpaths, reverse proxy rules
├── docker-compose.yml     # Production compose — Caddy + app containers
├── .env.example           # Environment variable template
└── README.md
```

## Configuration

### Caddyfile

The Caddyfile defines your domain and which services to proxy:

```
your-domain.duckdns.org {
    handle_path /myapp/* {
        reverse_proxy myapp:5000
    }

    redir /myapp /myapp/ permanent
}
```

Key directives:

| Directive | Purpose |
|-----------|---------|
| `handle_path /myapp/*` | Matches `/myapp/...` and **strips** `/myapp` before forwarding |
| `reverse_proxy myapp:5000` | Forwards request to the `myapp` service on port 5000 |
| `redir /myapp /myapp/` | Redirects bare path to path with trailing slash |

### Adding services

Add a `handle_path` block per service in the Caddyfile, then add the service in `docker-compose.yml`:

```yaml
services:
  myapp:
    build: ../myapp
    container_name: myapp
    restart: unless-stopped
    environment:
      - APPLICATION_ROOT=/myapp    # Flask-specific, adjust for your framework
    volumes:
      - ../myapp/uploads:/app/uploads

  caddy:
    image: caddy:2
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - myapp

volumes:
  caddy_data:
  caddy_config:
```

### Environment variables

Copy `.env.example` to `.env` and edit:

```bash
cp .env.example .env
```

| Variable | Default | Description |
|----------|---------|-------------|
| `DOMAIN` | `paul-sandbox.duckdns.org` | Your domain name |
| `APP_PATH` | `/vitaminchecker` | Subpath prefix |

## Prerequisites

1. **VPS** with a public IP (Hetzner, OVH, DigitalOcean, etc.)
2. **DNS** pointing your domain to the VPS IP
3. **Ports 80 and 443** open on the firewall

```bash
# UFW (Ubuntu)
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Or iptables
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
```

4. **Docker + Docker Compose** installed on the VPS

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# Log out and back in
```

## Deployment

### Full deploy (clone + start)

```bash
git clone https://github.com/BuildWithPaul/caddy-docker.git ~/caddy-docker
cd ~/caddy-docker

# Edit Caddyfile with your domain and services
# Edit docker-compose.yml with your app services

docker compose up -d
```

### Update running services

```bash
cd ~/caddy-docker && git pull
docker compose up -d --build
```

### Useful commands

```bash
# View logs
docker compose logs -f caddy          # Caddy logs
docker compose logs -f myapp          # App logs

# Restart everything
docker compose restart

# Stop everything
docker compose down

# Check certificate status
docker exec caddy caddy list-modules | grep tls
```

## Certificate Auto-Renewal

Caddy handles this. No manual intervention needed.

- Certificate obtained on first HTTPS request
- Renewed ~30 days before expiry
- Zero downtime (SNI + hot-swap)

## Hosting at root path

If you want your app at the root (`https://yourdomain.com/` instead of `https://yourdomain.com/myapp/`):

```caddyfile
your-domain.duckdns.org {
    reverseProxy myapp:5000
}
```

Remove the `handle_path` and `redir` blocks. Remove `APPLICATION_ROOT` from the app environment.

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Certificate fails | DNS not pointing to VPS yet — wait for propagation. Check port 80 is open. |
| 502 Bad Gateway | App container not running — check `docker compose logs myapp` |
| Static assets 404 | `APPLICATION_ROOT` must match the Caddyfile subpath. For Flask, set both. |
| Redirect loop | Ensure `handle_path` strips the prefix, and `APPLICATION_ROOT` adds it back. |

## Used by

- [VitaminChecker](https://github.com/BuildWithPaul/VitaminChecker) — Flask app at `/vitaminchecker`

## License

MIT