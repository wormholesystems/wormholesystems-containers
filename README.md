# Wormhole Systems — Container Stack

[![Submodule Updates](https://github.com/WormholeSystems/wormholesystems-containers/actions/workflows/update.yml/badge.svg)](https://github.com/WormholeSystems/wormholesystems-containers/actions/workflows/update.yml)
[![wsctl](https://img.shields.io/github/v/release/WormholeSystems/wormholesystems-cli?label=wsctl)](https://github.com/WormholeSystems/wormholesystems-cli)
[![App](https://img.shields.io/badge/app-WormholeSystems-blue)](https://github.com/WormholeSystems/wormholesystems-legacy)
[![License](https://img.shields.io/badge/license-MIT-green)](https://github.com/WormholeSystems/wormholesystems-legacy/blob/main/LICENSE)

Production-ready Docker stack for self-hosting [Wormhole Systems](https://wormhole.systems): Traefik with automatic SSL, frankenPHP, MySQL, Redis, Reverb websockets, queue worker and scheduler.

## Requirements

- A server with Docker and Docker Compose (v2.24+), at least 4 GB RAM
- Ports 80 and 443 open, domain names pointing at the server (e.g. `wormhole.systems` and `ws.wormhole.systems`)
- An [EVE developer application](https://developers.eveonline.com/) — the setup tells you the exact callback URL and scopes to configure

## Setup

The recommended way is [`wsctl`](https://github.com/WormholeSystems/wormholesystems-cli), the interactive setup wizard:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://install.wormhole.systems | sh
```

It clones this repository, asks for your domains and EVE credentials, generates all secrets (database passwords, Reverb keys, Laravel `APP_KEY`) with guaranteed-matching configuration files, then builds and starts the stack and initializes the database. Interrupted setups resume where they stopped, and preflight checks catch missing Docker, occupied ports and the like before they cost you a build.

There is also a local test mode (no SSL, localhost) for trying things out on a workstation.

<details>
<summary><strong>Manual setup</strong> (what the wizard automates)</summary>

```bash
git clone --recurse-submodules https://github.com/WormholeSystems/wormholesystems-containers.git
cd wormholesystems-containers

# 1. Docker + Laravel configuration — fill in domains, ACME email, EVE
#    credentials, DB credentials and the Reverb secrets:
#    REVERB_APP_ID: openssl rand -hex 8, KEY/SECRET: openssl rand -hex 16
cp .env.production.example .env
nano .env

# 2. MySQL container credentials — MUST match DB_* in .env exactly
cp dockerfiles/mysql/.env.example dockerfiles/mysql/.env
nano dockerfiles/mysql/.env

# 3. Traefik network, build, start
docker network create -d bridge web
docker compose build
docker compose up -d

# 4. Initialize: EVE static data (~500MB), app key, database
docker compose exec app php artisan sde:download
docker compose exec app php artisan key:generate
# IMPORTANT: key:generate only writes the key inside the container —
# print it and copy it into APP_KEY in the host .env, or it is lost
# when the container is recreated:
docker compose exec app grep APP_KEY .env
docker compose exec app php artisan migrate --seed
docker compose exec app php artisan optimize:clear
docker compose exec app php artisan optimize
```

⚠️ **Set `CONTACT_EMAIL` to a real address** (`you@example.com | Your EVE Character`) — CCP requires contact info on third-party apps; leaving it unset risks an EVE API ban.

Optionally restrict logins to specific characters/corporations/alliances via `ALLOWED_AFFILIATION_IDS` (comma-separated EVE IDs, empty allows everyone). The value is read at container start — after changing it, restart the stack or re-cache with `docker compose exec app php artisan optimize`.

</details>

Once up, your instance is available at `https://your-domain` (log in via EVE SSO) with websockets at `wss://ws.your-domain`. SSL certificates are issued and renewed automatically by Traefik.

## Updating game data

EVE's static data export (SDE) changes with game patches. With the stack running:

```bash
wsctl update
```

or manually: `sde:download`, `migrate`, `sde:seed` via `docker compose exec app php artisan ...`.

## Services

| Service | Role |
|---|---|
| traefik | Reverse proxy, automatic Let's Encrypt SSL |
| app | frankenPHP application server |
| mysql | Database |
| redis | Cache |
| reverb | Websocket server (real-time map updates) |
| queue / scheduler | Laravel queue worker and task scheduler |
| killmail-listener | Ingests killmails from zKillboard |

## Troubleshooting

```bash
docker compose ps                  # service status
docker compose logs -f [service]   # logs (e.g. traefik for SSL issues)
docker compose restart             # restart everything
docker compose exec app php artisan tinker   # poke the app
```

## Related repositories

- [wormholesystems-cli](https://github.com/WormholeSystems/wormholesystems-cli) — `wsctl`, the setup and management tool
- [wormholesystems-legacy](https://github.com/WormholeSystems/wormholesystems-legacy) — the Laravel application

## License

MIT
