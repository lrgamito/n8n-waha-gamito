# Production Deployment (Single Domain + /waha)

This guide deploys production on a VPS with:

- n8n at `https://<N8N_DOMAIN>/`
- WAHA at `https://<N8N_DOMAIN>/waha/`

Reverse proxy behavior (important for WAHA Dashboard):

- `/waha/*` -> WAHA backend
- `/dashboard/*` -> WAHA dashboard assets
- `/api/*` and `/ws` -> WAHA backend (fallback required because dashboard may use origin URL)
- `/` -> n8n

## 1. Outside environment checks (DNS + firewall)

- Confirm DNS A record: `N8N_DOMAIN -> VPS_PUBLIC_IP`
- Ensure inbound ports are open on cloud firewall/security list:
  - `22/tcp` (SSH)
  - `80/tcp` (Let's Encrypt HTTP challenge)
  - `443/tcp` (HTTPS)

Important: if certbot reports `Timeout during connect`, your cloud firewall still blocks `80/tcp`.

## 2. Prepare production env locally

Create the real env file and set real values:

```bash
cp production/.env.prod.example production/.env.prod
```

Rules:

- Keep secrets in `production/.env.prod`
- Keep `production/.env.prod.example` sanitized

Validate compose config locally:

```bash
docker-compose --env-file production/.env.prod -f production/docker-compose.prod.yml config > /tmp/prod-compose-resolved.yml
```

## 3. Upload files from local machine to VPS

Default VPS target path:

- `/home/ubuntu/n8n-waha`

Upload with rsync:

```bash
rsync -az --delete -e "ssh -i ./keys/cloudubuntu.priv" ./ ubuntu@164.152.61.217:/home/ubuntu/n8n-waha/
```

Fallback (`scp`):

```bash
scp -i ./keys/cloudubuntu.priv -r ./ ubuntu@164.152.61.217:/home/ubuntu/n8n-waha
```

## 4. Connect and prepare VPS runtime

Connect:

```bash
ssh ubuntu@164.152.61.217 -i ./keys/cloudubuntu.priv
```

On VPS:

1. Install Docker and Docker Compose plugin (`docker compose`) if missing.
2. Go to project directory:

```bash
cd /home/ubuntu/n8n-waha
```

3. If Docker socket is restricted, run compose commands with `sudo`.
4. If this VPS image has restrictive `iptables`, allow `80/443` on host:

```bash
sudo iptables -I INPUT 1 -p tcp --dport 80 -m state --state NEW -j ACCEPT
sudo iptables -I INPUT 1 -p tcp --dport 443 -m state --state NEW -j ACCEPT
```

## 5. Choose WAHA image tag by CPU architecture

- ARM VPS (`aarch64`): use `WAHA_IMAGE=devlikeapro/waha:gows-arm-2026.4.2`
- AMD64 VPS (`x86_64`): use `WAHA_IMAGE=devlikeapro/waha:gows-2026.4.2` (or another tested amd64 tag)

## 6. Issue initial TLS certificate

Before running certbot-init:

- Ensure no process binds port `80`
- Ensure cloud firewall allows inbound `80/tcp`

Run:

```bash
sudo docker compose --env-file production/.env.prod -f production/docker-compose.prod.yml --profile init run --rm --service-ports certbot-init
```

## 7. Start production services

Start all services:

```bash
sudo docker compose --env-file production/.env.prod -f production/docker-compose.prod.yml up -d
```

Enable automatic certificate renewal:

```bash
sudo docker compose --env-file production/.env.prod -f production/docker-compose.prod.yml --profile certbot up -d certbot
```

## 8. Acceptance checks

Container status:

```bash
sudo docker compose --env-file production/.env.prod -f production/docker-compose.prod.yml ps
```

Endpoints:

- `https://<N8N_DOMAIN>/` should load n8n login
- `https://<N8N_DOMAIN>/waha/` should respond from WAHA

WAHA API check (root fallback route):

```bash
curl -H "X-Api-Key: <WAHA_API_KEY>" https://<N8N_DOMAIN>/api/server/version
```

Logs:

```bash
sudo docker compose --env-file production/.env.prod -f production/docker-compose.prod.yml logs -f nginx n8n waha
```

Functional check:

- Trigger a WAHA `message` event and confirm n8n receives it at `https://<N8N_DOMAIN>/webhook/webhook`.

## 9. Troubleshooting WAHA dashboard connection

Symptom:

- Dashboard shows `Server connection failed` for `https://<N8N_DOMAIN>`.

Checks:

1. Confirm the API key includes all characters (including trailing `=` if present).
2. Confirm this works:

```bash
curl -H "X-Api-Key: <WAHA_API_KEY>" https://<N8N_DOMAIN>/api/server/status
```

3. In dashboard server config, set:
   - URL: `https://<N8N_DOMAIN>`
   - API key: `<WAHA_API_KEY>`

If browser storage is stale, reset `localStorage.servers` from browser devtools and reload.

## 10. Update / rollback

Update:

```bash
sudo docker compose --env-file production/.env.prod -f production/docker-compose.prod.yml pull
sudo docker compose --env-file production/.env.prod -f production/docker-compose.prod.yml up -d
```

Rollback:

1. Revert image tags in `production/.env.prod`.
2. Re-run:

```bash
sudo docker compose --env-file production/.env.prod -f production/docker-compose.prod.yml up -d
```
