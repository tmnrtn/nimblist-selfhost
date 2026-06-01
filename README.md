# Nimblist — Community Edition (self-hosting)

Run your own instance of Nimblist — collaborative shopping lists, recipes, and meal
planning — from public container images. No source build required; everything is
configured at runtime through a `.env` file.

> This is the **Community Edition**. It is branded distinctly from the hosted service
> at [nimblist.app](https://nimblist.app) and is provided under a self-hosting
> licence (personal/internal use only — see [LICENSE](./LICENSE)).

> **Prefer not to run it yourself?** The fully-managed hosted version — automatic
> updates, backups, and zero ops — is at
> **[nimblist.app](https://nimblist.app/?utm_source=selfhost&utm_medium=readme&utm_campaign=ce)**.
> Free to start; Premium adds recipe import (URL + photo) and meal planning.

## What you get

A self-contained stack of public images from `ghcr.io/tmnrtn/nimblist-*`:

| Service | Image | Purpose |
|---|---|---|
| api | `nimblist-api` | ASP.NET Core API + SignalR |
| frontend | `nimblist-frontend` | React SPA (served by nginx) |
| classification | `nimblist-classification` | item categorisation (Python) |
| recipescraper | `nimblist-recipescraper` | recipe import (Python) |
| caddy | `caddy:2-alpine` | internal HTTP reverse proxy (one origin) |
| db | `postgres:15` | database (bundled, auto-migrates) |
| redis | `redis:7-alpine` | SignalR backplane + key ring |

## Prerequisites

- **Docker** with the Compose plugin (`docker compose`).
- **Your own TLS-terminating reverse proxy** (Nginx Proxy Manager, Traefik, Caddy,
  nginx, …) with a domain and certificate. This stack serves **plain HTTP** on one
  port — you put HTTPS in front of it.

## Quickstart

```bash
# 1. Get these files (docker-compose.yml, Caddyfile, .env.example)
# 2. Create your env
cp .env.example .env

# 3. Set at least the two required values in .env:
#    POSTGRES_PASSWORD   (openssl rand -base64 24)
#    PUBLIC_BASE_URL     (e.g. https://nimblist.example.com)

# 4. Start it
docker compose up -d
```

The stack now listens on `http://<host>:8080` (override with `NIMBLIST_HTTP_PORT`).
Point your reverse proxy's vhost at that address. The bundled Postgres creates and
migrates its schema automatically on first boot.

## ⚠️ HTTPS is required

The app sets **Secure-only authentication cookies**, so login only works over HTTPS.
Your reverse proxy must:

1. **Terminate TLS** and serve the site over `https://`.
2. **Forward `X-Forwarded-Proto: https`** — without it the API thinks the request is
   insecure, won't set the auth cookie, and login silently fails. (Most proxies do
   this automatically.)
3. **Preserve the `Host` header** (used for OAuth redirects and email links).
4. **Allow WebSocket upgrades** for `/hubs/*` (SignalR live updates).

Example with a front Caddy (on another host/container):

```caddy
nimblist.example.com {
    reverse_proxy http://<host>:8080
}
```

Caddy sets `X-Forwarded-Proto`, preserves `Host`, and upgrades WebSockets by default.
In **Nginx Proxy Manager**, enable "Websockets Support" and (recommended) "Block
Common Exploits"; NPM forwards the proto/host headers automatically.

## Configuration

All configuration lives in `.env` — see [.env.example](./.env.example) for the full,
commented list. Summary:

- **Required:** `POSTGRES_PASSWORD`, `PUBLIC_BASE_URL`.
- **Recommended:** `ADMIN_EMAIL` (your admin account), `JWT_KEY` (browser extension),
  VAPID keys (web push).
- **Optional:** OAuth providers (Google/Facebook/Microsoft), Resend (email), LLM
  recipe import, image/version/port tuning.

### First admin

Set `ADMIN_EMAIL` in `.env`, then register a normal account with that exact email —
it is promoted to Admin automatically on the next start.

### Web push notifications

Generate a VAPID key pair and set `VAPID_SUBJECT` / `VAPID_PUBLIC_KEY` /
`VAPID_PRIVATE_KEY`:

```bash
npx web-push generate-vapid-keys
```

The public key is served to the app at runtime (`/api/config`) — no rebuild needed.
If you leave VAPID unset, push is simply disabled (the prompt never appears).

## Upgrading

```bash
docker compose pull
docker compose up -d
```

Database migrations apply automatically on API start. To pin a specific version
instead of tracking `latest`, set `NIMBLIST_VERSION=<x.y.z>` in `.env`.

## Backup & restore

Your data lives in the `postgres_data` Docker volume. Back it up with `pg_dump`:

```bash
# Backup
docker compose exec -T db pg_dump -U nimblist nimblistdb > nimblist-backup.sql

# Restore (into a fresh, empty database)
cat nimblist-backup.sql | docker compose exec -T db psql -U nimblist nimblistdb
```

> Adjust the user/db names if you changed `POSTGRES_USER` / `POSTGRES_DB`.
> The `dataprotection_keys` volume holds the cookie/token signing keys — if you lose
> it, existing sessions are invalidated and everyone must log in again (not data loss).

## Troubleshooting

- **502 / Bad Gateway right after `up`** — the API is still starting (it waits for
  Postgres). Check `docker compose logs -f nimblist-api`; give it ~30–60s.
- **Can't log in / login does nothing** — you're not on HTTPS, or your proxy isn't
  forwarding `X-Forwarded-Proto: https`. Auth cookies are Secure-only. See *HTTPS is
  required* above.
- **No notification prompt** — VAPID keys aren't set (push is disabled).
- **Password-reset / invite emails not arriving** — Resend isn't configured
  (`RESEND_API_KEY`). The app runs fine without it; emails are just skipped.
- **Health check** — `curl https://nimblist.example.com/healthz` should return `200`.

## Licence

Provided under the Nimblist Self-Hosting License (see [LICENSE](./LICENSE)): free for
personal and internal use; **not** for offering as a hosted/commercial service, and
not for redistribution. The application source code is proprietary and not included.
