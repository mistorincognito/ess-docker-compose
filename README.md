# Matrix Server - Docker Compose Setup

A self-hosted Matrix server stack with modern OIDC authentication, web client, optional video calling, and optional messaging bridges.

## What's Included

**Core (always on)**
- [Synapse](https://github.com/element-hq/synapse) — Matrix homeserver
- [Matrix Authentication Service (MAS)](https://github.com/element-hq/matrix-authentication-service) — OIDC-based authentication
- [Element Web](https://github.com/element-hq/element-web) — Web client
- [Element Admin](https://github.com/element-hq/element-admin) — Admin dashboard
- [PostgreSQL 16](https://www.postgresql.org/) — Database
- [Caddy](https://caddyserver.com/) — Reverse proxy with automatic HTTPS

**Optional: Element Call** (`--profile element-call`)
- [LiveKit](https://livekit.io/) — WebRTC SFU media server (self-hosted, media stays on your server)
- [lk-jwt-service](https://github.com/element-hq/lk-jwt-service) — LiveKit token issuer
- [Element Call](https://github.com/element-hq/element-call) — Self-hosted video/voice calling frontend

**Optional: Messaging Bridges** (via `setup-bridges.sh`)
- [mautrix-whatsapp](https://github.com/mautrix/whatsapp) — WhatsApp bridge
- [mautrix-signal](https://github.com/mautrix/signal) — Signal bridge
- [mautrix-telegram](https://github.com/mautrix/telegram) — Telegram bridge (requires API credentials)

**Optional: Upstream OIDC** (`--profile authelia`)
- [Authelia](https://www.authelia.com/) — SSO / identity provider

## Quick Start

```bash
./deploy.sh
```

The script handles everything interactively: generates secrets, writes configs, starts Docker services. It asks whether to enable Element Call and optionally Authelia.

Bridges are set up separately after the core stack is running:

```bash
./setup-bridges.sh
```

## Architecture

```
Browser
  |
Caddy (HTTPS, Let's Encrypt)
  |
  +-- matrix.example.com  -->  Synapse :8008
  |     /.well-known       -->  (inline, served by Caddy)
  |     /login, /logout    -->  MAS :8080
  +-- auth.example.com    -->  MAS :8080
  +-- element.example.com -->  Element Web :80
  +-- admin.example.com   -->  Element Admin :8080
  +-- call.example.com    -->  Element Call :8080   (optional)
  +-- rtc.example.com     -->  lk-jwt-service :8080 (optional)
                               LiveKit :7880         (optional)
```

All services communicate over an internal Docker network. The database is not exposed.

## Deployment Options

**Local testing** — self-signed certificates, all services on localhost:
```bash
docker compose -f docker-compose.yml -f compose-variants/docker-compose.local.yml up -d
```

**Single server** — production with Caddy on the same machine:
```bash
./deploy.sh  # choose "single server" mode
```

**Multi-server** — Matrix backend on one machine, Caddy on another:
```bash
./deploy.sh  # choose "multi server" mode
```
The script generates a `caddy/Caddyfile.production` for the Caddy machine.

## Element Call

When enabled, all three components are self-hosted. Media streams never leave your server (they route through your LiveKit SFU). The Element Call frontend is served from your own `call.` subdomain instead of `call.element.io`.

Required open ports for LiveKit:
- TCP 7881 (WebRTC signaling)
- UDP 50100–50200 (media streams)

## Bridges

`setup-bridges.sh` configures WhatsApp and Signal automatically. Telegram requires API credentials from [my.telegram.org](https://my.telegram.org) — add them to `.env` before running:

```
TELEGRAM_API_ID=your_id
TELEGRAM_API_HASH=your_hash
```

Bridges use double puppet support (users in Matrix rooms appear as themselves, not the bridge bot) and have encryption disabled for compatibility with MAS.

## Requirements

- Docker and Docker Compose v2
- A domain with DNS control
- Ports 80 and 443 accessible from the internet
- For Element Call: ports 7881/TCP and 50100–50200/UDP open

## Common Operations

```bash
# Status
docker compose ps

# Logs
docker compose logs -f [service]

# Restart a service
docker compose restart synapse

# Update all images
docker compose pull && docker compose up -d

# Bridge logs
docker compose logs mautrix-whatsapp
```

## Data Directories

```
postgres/data/    database
synapse/data/     media store, signing keys
mas/data/         MAS sessions
.env              all secrets and domain config
```

Backup:
```bash
tar -czf matrix-backup-$(date +%Y%m%d).tar.gz postgres/data synapse/data mas/data .env
```

## Documentation

- [SETUP.md](SETUP.md) — manual setup reference
- [PRODUCTION_DEPLOYMENT.md](PRODUCTION_DEPLOYMENT.md) — production checklist
- [BRIDGE_SETUP_GUIDE.md](BRIDGE_SETUP_GUIDE.md) — bridge configuration details

## License

- Synapse: Apache 2.0
- Matrix Authentication Service: Apache 2.0
- Element Web / Element Admin / Element Call: Apache 2.0
- PostgreSQL: PostgreSQL License
- Caddy: Apache 2.0
- LiveKit: Apache 2.0
