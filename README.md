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
Browser → Caddy (80/443, auto-HTTPS) → vitaminchecker:5000      (/vitaminchecker/)
                                     → chouchoualerte:5001        (/chouchoualerte/)
                                     → more services...
```

Caddy handles TLS termination and path routing. Each app runs in its own container, blissfully unaware of the subpath.

## Currently hosting

| Service | Path | Repo |
|---------|------|------|
| VitaminChecker | `/vitaminchecker/` | [BuildWithPaul/VitaminChecker](https://github.com/BuildWithPaul/VitaminChecker) |
| ChouChouAlerte | `/chouchoualerte/` | [BuildWithPaul/ChouChouAlerte](https://github.com/BuildWithPaul/ChouChouAlerte) |

## Quick Start

```bash
# 1. Clone all repos side by side
git clone https://github.com/BuildWithPaul/caddy-docker.git ~/caddy-docker
git clone https://github.com/BuildWithPaul/VitaminChecker.git ~/VitaminChecker
git clone https://github.com/BuildWithPaul/ChouChouAlerte.git ~/ChouChouAlerte

# 2. Configure ChouChouAlerte environment
cp ~/ChouChouAlerte/.env.example ~/ChouChouAlerte/.env
# Edit .env with your SNCF_API_TOKEN, TELEGRAM_BOT_TOKEN, TELEGRAM_CHAT_ID

# 3. Deploy
cd ~/caddy-docker
docker compose up -d --build
```

Your services are live at:
- `https://paul-sandbox.duckdns.org/vitaminchecker/`
- `https://paul-sandbox.duckdns.org/chouchoualerte/`

## Project Structure

```
caddy-docker/
├── Caddyfile              # Caddy config — domain, subpaths, reverse proxy rules
├── docker-compose.yml     # Production compose — Caddy + all app containers
├── .env.example           # Environment variable template
└── README.md
```

Expected repo layout on the server:
```
~/
├── caddy-docker/          # This repo (Caddy + compose)
├── VitaminChecker/        # Flask app
└── ChouChouAlerte/         # Flask app
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

### Adding a new service

1. Add a `handle_path` block in `Caddyfile`
2. Add the service in `docker-compose.yml`
3. Rebuild: `docker compose up -d --build`

### ChouChouAlerte specifics

ChouChouAlerte requires environment variables (`SNCF_API_TOKEN`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`). Create a `.env` file in the ChouChouAlerte repo root:

```bash
cp ../ChouChouAlerte/.env.example ../ChouChouAlerte/.env
# Edit with your tokens
```

The compose file loads `../ChouChouAlerte/.env` via `env_file`.

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
git clone https://github.com/BuildWithPaul/VitaminChecker.git ~/VitaminChecker
git clone https://github.com/BuildWithPaul/ChouChouAlerte.git ~/ChouChouAlerte

# Configure ChouChouAlerte
cp ~/ChouChouAlerte/.env.example ~/ChouChouAlerte/.env
# Edit ~/ChouChouAlerte/.env with your tokens

cd ~/caddy-docker
docker compose up -d --build
```

### Update running services

```bash
cd ~/caddy-docker && git pull
cd ~/VitaminChecker && git pull
cd ~/ChouChouAlerte && git pull

cd ~/caddy-docker && docker compose up -d --build
```

### Useful commands

```bash
# View logs
docker compose logs -f caddy              # Caddy logs
docker compose logs -f vitaminchecker     # VitaminChecker logs
docker compose logs -f chouchoualerte     # ChouChouAlerte logs

# Restart everything
docker compose restart

# Stop everything
docker compose down

# Check certificate status
docker exec caddy caddy list-modules | grep tls

# Reload Caddyfile without downtime
docker exec caddy caddy reload --config /etc/caddy/Caddyfile
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
| 502 Bad Gateway | App container not running — check `docker compose logs <service>` |
| Static assets 404 | `APPLICATION_ROOT` must match the Caddyfile subpath. For Flask, set both. |
| Redirect loop | Ensure `handle_path` strips the prefix, and `APPLICATION_ROOT` adds it back. |
| Container name conflict | `docker rm -f <name>` then `docker compose up -d --build` |

## Used by

- [VitaminChecker](https://github.com/BuildWithPaul/VitaminChecker) — Flask app at `/vitaminchecker`
- [ChouChouAlerte](https://github.com/BuildWithPaul/ChouChouAlerte) — Flask app at `/chouchoualerte`

## License

MIT