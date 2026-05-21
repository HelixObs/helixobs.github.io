# Deployment

The HelixObs stack is deployed with Docker Compose. All services run on a single host, communicating over a private Docker network.

## Prerequisites

- Docker and Docker Compose v2
- Persistent storage for database, traces, and logs (see [Volumes](#volumes))
- Outbound internet access for Caddy to obtain a TLS certificate

## Quick start

```bash
git clone https://github.com/HelixObs/deploy
cd deploy

# Set production environment variables (see below)
cp .env.example .env
$EDITOR .env

docker compose up -d --build
```

## Environment variables

Create a `.env` file in the `deploy/` directory. Docker Compose reads it automatically.

### Required in production

| Variable | Example | Description |
|---|---|---|
| `HELIXOBS_DOMAIN` | `helixobs.example.org` | Public domain — Caddy obtains a TLS cert for this |
| `UI_BASE_URL` | `https://helixobs.example.org` | Base URL used in notification links to the Entity Inspector |
| `GRAFANA_URL` | `https://helixobs.example.org:3001` | Grafana URL used in notification links |

### Notification credentials (per instrument)

These are read from environment variables referenced in each instrument's YAML config:

```bash
# Example — replace with your actual env var names from the instrument YAML
MY_INST_SLACK_WEBHOOK=https://hooks.slack.com/services/...
MY_INST_GITHUB_TOKEN=ghp_...
```

### Optional

| Variable | Default | Description |
|---|---|---|
| `JWT_SECRET` | *(empty — auth disabled)* | Comma-separated signing secrets for herald JWT auth. Empty = auth disabled |
| `ANTHROPIC_API_KEY` | *(empty)* | Required for Sherlock AI troubleshooting |
| `GITHUB_TOKEN` | *(empty)* | Used by Sherlock to fetch source files from private repos |

## Volumes

Production deployments should back volumes onto persistent host paths. The compose file expects these directories to exist:

```bash
sudo mkdir -p /data/prometheus /data/loki /data/tempo
sudo chown -R 1000:1000 /data/loki  # Loki runs as UID 1000
```

The `db` service uses a named Docker volume (`pg-data`) by default. For persistence across `docker compose down`, ensure this volume is not removed (`down -v` removes it — avoid in production).

## TLS

Caddy handles TLS termination for the UI (port 443) and Grafana (port 3001). It automatically obtains and renews a Let's Encrypt certificate for `HELIXOBS_DOMAIN`.

For local development, Caddy issues a local CA certificate for `localhost` — no configuration needed.

!!! warning
    The herald (`:4317`) and OTel Collector (`:4319`) currently accept plaintext gRPC. Restrict access to these ports at the firewall level to trusted instrument hosts only.

## Rebuilding a single service

```bash
docker compose build herald && docker compose up -d herald
```

## Viewing logs

```bash
docker compose logs -f herald sherlock
```

## Resetting the database

```bash
# WARNING: destroys all entity data, traces, and logs
docker compose down -v
docker compose up -d
```

## Migrations

Database migrations run automatically on first start via `docker-entrypoint-initdb.d/`. To apply a new migration on an existing deployment:

```bash
# Migrations only run on a fresh DB volume.
# For an existing deployment, apply manually:
docker compose exec db psql -U helix -d helixobs -f /migrations/00N_description.sql
```
