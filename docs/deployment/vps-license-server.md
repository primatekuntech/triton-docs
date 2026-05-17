# VPS Deployment — License Server (Locally Built Image)

This guide walks through deploying the Triton License Server on a fresh VPS
using a locally built container image. Use this when you do not yet have a
published image on a registry.

---

## Prerequisites

- Ubuntu 22.04 / 24.04 (or any systemd-based Linux)
- 2 GB RAM, 20 GB disk minimum
- Podman installed (`podman --version`)
- `podman-compose` or the Podman compose plugin
- `git`, `openssl`, `curl` installed
- Port **8081** open in your firewall

Install Podman on Ubuntu if not already done:

```bash
sudo apt-get update && sudo apt-get install -y podman git openssl curl
pip3 install podman-compose          # or: sudo apt-get install podman-compose
```

---

## 1 — Clone the repository

```bash
git clone https://github.com/primatekuntech/triton.git
cd triton
```

---

## 2 — Build the license server image

```bash
podman build -f Containerfile.licenseserver -t triton-license-server:local .
```

This is a multi-stage build (Node → Go → scratch). It takes 3–5 minutes on
first run; subsequent builds are faster due to layer caching.

Verify the image was created:

```bash
podman images | grep triton-license-server
```

---

## 3 — Generate the environment file and start

```bash
cd scripts/deploy/license-server
sudo bash install.sh \
    --admin-email admin@yourcompany.com \
    --no-tls
```

`install.sh` will:
- Copy `env.template` → `.env` (mode 0600)
- Generate a random PostgreSQL password
- Generate a fresh Ed25519 signing keypair
- Generate a bootstrap admin password (printed to stdout — save it)
- Create the binary storage directory (`/opt/triton/binaries` by default)
- Start the containers

> **Save the printed admin password.** It is only shown once.

### Binary directory

Worker binaries are stored as files on the host at `TRITON_LICENSE_SERVER_HOST_BIN_DIR`
(default `/opt/triton/binaries`). The install script creates it automatically.
To use a different path, edit `.env` before running `install.sh`:

```bash
sed -i 's|^TRITON_LICENSE_SERVER_HOST_BIN_DIR=.*|TRITON_LICENSE_SERVER_HOST_BIN_DIR=/data/triton/binaries|' \
    scripts/deploy/license-server/.env
```

Ensure the directory is on a volume with at least **5 GB** free — each binary
is 10–400 MB and you typically upload one per platform/arch combination.

---

## 4 — Verify the environment

```bash
cd triton
./scripts/verify-license-env.sh scripts/deploy/license-server/.env
```

All checks must pass before proceeding. The script validates the signing key
format, TLS config, binary directory, and optional integrations.

## 5 — Point compose to the local image

Open `.env` and set `TRITON_LICENSE_IMAGE` to the local tag:

```bash
sed -i 's|^TRITON_LICENSE_IMAGE=.*|TRITON_LICENSE_IMAGE=triton-license-server:local|' .env
```

Restart the license server container to pick up the change:

```bash
podman-compose down license-server
podman-compose up -d license-server
```

---

## 6 — Verify it is running

```bash
# Container status
podman ps --filter name=triton

# Health check
curl -s http://localhost:8081/api/v1/health | python3 -m json.tool

# Tail logs
podman logs -f triton-licenseserver
```

Expected health response:

```json
{"status": "ok"}
```

---

## 7 — First login and setup

Open the admin UI in your browser (replace with your VPS IP or hostname):

```
http://<VPS-IP>:8081/ui/
```

1. Log in with the admin email and the bootstrap password from step 3.
2. Go to **Account → Change password** and rotate the bootstrap password.
3. Navigate to **Organisations → New** and create your first customer org.
4. Navigate to **Licences → New** and issue a licence to that org.
5. Copy the licence token — the customer pastes it into their Manage Server
   setup wizard.

---

## 8 — Reverse proxy with Nginx (recommended)

Running behind Nginx lets you add TLS without touching the container.

Install Nginx and Certbot:

```bash
sudo apt-get install -y nginx certbot python3-certbot-nginx
```

Create `/etc/nginx/sites-available/license-server`:

```nginx
server {
    listen 80;
    server_name license.yourcompany.com;

    location / {
        proxy_pass         http://127.0.0.1:8081;
        proxy_http_version 1.1;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
        client_max_body_size 512M;   # allow binary uploads
    }
}
```

Enable and get a certificate:

```bash
sudo ln -s /etc/nginx/sites-available/license-server /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d license.yourcompany.com
```

Update `PUBLIC_URL` in `.env`:

```bash
sed -i 's|^TRITON_LICENSE_SERVER_PUBLIC_URL=.*|TRITON_LICENSE_SERVER_PUBLIC_URL=https://license.yourcompany.com|' .env
podman-compose down license-server && podman-compose up -d license-server
```

---

## 9 — Auto-start on reboot

Generate and install a systemd service so the containers come back up after
a reboot:

```bash
podman generate systemd --name triton-licenseserver --files --new
sudo mv container-triton-licenseserver.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now container-triton-licenseserver.service
```

Verify it starts cleanly:

```bash
sudo systemctl status container-triton-licenseserver.service
```

---

## Day-2 operations

### Upgrade (rebuild and restart)

```bash
cd triton
git pull
podman build -f Containerfile.licenseserver -t triton-license-server:local .

cd scripts/deploy/license-server
podman-compose down license-server
podman-compose up -d license-server
podman logs -f triton-licenseserver   # watch migration logs
```

### Backup

Three things to back up:

```bash
mkdir -p /var/backups/triton

# 1. Database — licence metadata (orgs, licences, activations, audit)
podman exec triton-license-db pg_dump -U triton triton_license \
    | gzip > /var/backups/triton/license-$(date +%F).sql.gz

# 2. .env — signing key lives here; losing it forces all customers to re-activate
cp scripts/deploy/license-server/.env \
    /var/backups/triton/license-env-$(date +%F)

# 3. Worker binaries — NOT in the DB; stored at TRITON_LICENSE_SERVER_HOST_BIN_DIR
tar -czf /var/backups/triton/license-binaries-$(date +%F).tar.gz \
    /opt/triton/binaries   # adjust to match TRITON_LICENSE_SERVER_HOST_BIN_DIR
```

Add to crontab for daily backups:

```bash
crontab -e
# Add:
0 3 * * * cd /path/to/triton && \
  podman exec triton-license-db pg_dump -U triton triton_license | gzip > /var/backups/triton/license-$(date +\%F).sql.gz && \
  tar -czf /var/backups/triton/license-binaries-$(date +\%F).tar.gz /opt/triton/binaries
```

### Restore

```bash
gunzip < /var/backups/triton/license-2026-01-15.sql.gz \
    | podman exec -i triton-license-db psql -U triton triton_license
```

### View logs

```bash
podman logs triton-licenseserver           # last run
podman logs -f triton-licenseserver        # follow
podman logs --tail 100 triton-licenseserver
```

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `REPORT_KEY` error on compose up | Not needed for standalone — leave `TRITON_LICENSE_SERVER_REPORT_KEY=` blank in `.env` |
| `go.mod requires go >= 1.26.3` during build | Pull latest code: `git pull` (golang:1.26.3 is now pinned in Containerfile) |
| Health check fails after start | Wait 15s for DB migrations; check `podman logs triton-licenseserver` |
| Port 8081 not reachable | `sudo ufw allow 8081/tcp` or check your cloud provider's security group |
| Bootstrap password lost | `grep ADMIN_PASSWORD scripts/deploy/license-server/.env` |
| Container exits immediately | Check logs; common cause is missing `TRITON_LICENSE_SERVER_SIGNING_KEY` in `.env` |
| Binary upload returns 503 | `TRITON_LICENSE_SERVER_HOST_BIN_DIR` is not set or the host directory does not exist — run `./scripts/verify-license-env.sh` |
| Binary upload fails for large files (>32 MB) | Ensure the host directory has enough disk space; check Nginx `client_max_body_size 512M` is set |

---

## Quick reference

```bash
# Status
podman ps --filter name=triton

# Health
curl -s http://localhost:8081/api/v1/health

# Logs
podman logs -f triton-licenseserver

# Stop
cd scripts/deploy/license-server && podman-compose down

# Start
cd scripts/deploy/license-server && podman-compose up -d

# Restart license server only (keep DB running)
podman restart triton-licenseserver
```
