# n8n + WAHA Stack

This repository separates local testing and production deployment.

## Project structure

- `test/`: local/test stack (insecure defaults)
  - `test/docker-compose.test.yml`
  - `test/.env.test.example`
- `production/`: production stack (single domain)
  - `production/docker-compose.prod.yml`
  - `production/.env.prod.example`
  - `production/deploy/nginx/default.conf.template`
- `PRODUCTION_DEPLOY.md`: step-by-step production deploy guide

## Local/Test usage

1. Create local test env file:

```bash
cp test/.env.test.example test/.env.test
```

2. Start test stack:

```bash
docker-compose --env-file test/.env.test -f test/docker-compose.test.yml up -d
```

3. Check status/logs:

```bash
docker-compose --env-file test/.env.test -f test/docker-compose.test.yml ps
docker-compose --env-file test/.env.test -f test/docker-compose.test.yml logs -f
```

4. Stop test stack:

```bash
docker-compose --env-file test/.env.test -f test/docker-compose.test.yml down
```

## Production usage

Follow [`PRODUCTION_DEPLOY.md`](./PRODUCTION_DEPLOY.md).

Production routes:

- n8n: `https://<N8N_DOMAIN>/`
- WAHA: `https://<N8N_DOMAIN>/waha/`
- WAHA API/WebSocket fallback for dashboard compatibility: `https://<N8N_DOMAIN>/api/*` and `https://<N8N_DOMAIN>/ws`

Quick start (VPS commands use `sudo docker compose`):

```bash
cp production/.env.prod.example production/.env.prod
# edit production/.env.prod with real values

sudo docker compose --env-file production/.env.prod -f production/docker-compose.prod.yml --profile init run --rm --service-ports certbot-init
sudo docker compose --env-file production/.env.prod -f production/docker-compose.prod.yml up -d
sudo docker compose --env-file production/.env.prod -f production/docker-compose.prod.yml --profile certbot up -d certbot
```

## Notes

- Do not use test defaults in production.
- Keep real secrets only in `production/.env.prod`.
- Set `WAHA_IMAGE` to match VPS architecture (`gows-arm-*` for ARM, `gows-*` for amd64).
- WAHA dashboard may connect using origin URL; keep nginx routes `/api/*` and `/ws` pointed to WAHA in production.
