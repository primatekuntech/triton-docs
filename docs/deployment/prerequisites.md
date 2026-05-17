# Deployment Prerequisites

Common host requirements for both License Server and Manage Server. Read once,
then jump to the app-specific guide.

## Supported hosts

- **Ubuntu Server 22.04 LTS or 24.04 LTS** — primary, tested
- **Debian 12+** — works, same packages
- **RHEL 9 / Rocky 9 / AlmaLinux 9** — works, swap `apt` for `dnf`
- **Container hosts** running Podman 4+ or Docker 24+ on any Linux

The deploy scripts auto-detect Debian/Ubuntu (apt) and RHEL-family (dnf).
Other distros work but you install the prereqs by hand.

## Sizing

| Workload | CPU | RAM | Disk | Notes |
|----------|-----|-----|------|-------|
| License Server (≤100 customers) | 2 vCPU | 2 GB | 20 GB | PostgreSQL is the floor |
| License Server (100–1000 customers) | 4 vCPU | 4 GB | 50 GB | |
| Manage Server (≤500 hosts) | 2 vCPU | 4 GB | 50 GB | scan results dominate disk |
| Manage Server (500–5000 hosts) | 4 vCPU | 8 GB | 200 GB | bump `TRITON_MANAGE_PARALLELISM` |

Disk is mostly PostgreSQL (`/var/lib/containers/storage/volumes/`). Plan
backup capacity at 2–3× the working set.

## Required ports

### License Server host

| Port | Protocol | Direction | Purpose |
|------|----------|-----------|---------|
| 443 (or 8081) | HTTPS | inbound | Admin UI + client API |
| 5432 | PostgreSQL | localhost only | DB (containerised) |

### Manage Server host

| Port | Protocol | Direction | Purpose |
|------|----------|-----------|---------|
| 443 (or 8082) | HTTPS | inbound | Admin UI |
| 8443 | mTLS | inbound | Agent gateway |
| 5432 | PostgreSQL | localhost only | DB (containerised) |
| 8200 | Vault | localhost only | Vault sidecar (optional) |

Outbound from manage server to license server: 443 (or whatever port the
license server publishes).

**Never** expose 5432 or 8200 to the public internet.

## Install Podman + tools (Ubuntu / Debian)

```bash
sudo apt update
sudo apt install -y podman podman-compose openssl jq curl ca-certificates
```

Verify:

```bash
podman --version          # ≥ 4.0
podman-compose --version  # ≥ 1.0
```

Or use the bundled script:

```bash
sudo bash scripts/deploy/common/prereq-ubuntu.sh
```

## Install Docker (alternative)

If you prefer Docker:

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo apt install -y docker-compose-plugin openssl jq curl
```

All commands in the per-app guides work identically — replace `podman` with
`docker` and `podman-compose` with `docker compose`.

## TLS

Both servers should run behind HTTPS in production. Three options:

1. **Reverse proxy with Let's Encrypt** (recommended) — Nginx or Caddy
   terminates TLS, proxies to the container's plain HTTP port.
2. **Container-native TLS** — pass cert + key paths via env, the binary
   serves HTTPS directly. Useful when there's no proxy layer.
3. **No TLS (dev only)** — set `TRITON_LICENSE_SERVER_ALLOW_INSECURE=1`
   to bypass the production TLS check. Never use in production.

Caddy example (manage server, replace hostname):

```caddy
manage.example.com {
    reverse_proxy localhost:8082
}
```

Nginx example with Let's Encrypt:

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
sudo certbot --nginx -d manage.example.com
# certbot edits /etc/nginx/sites-available/default to add SSL
```

Then point the upstream at `localhost:8082` (manage) or `localhost:8081`
(license).

## Firewall

UFW recipe for a public-facing manage server:

```bash
sudo ufw allow 22/tcp                    # SSH
sudo ufw allow 80/tcp                    # HTTP (Let's Encrypt redirect)
sudo ufw allow 443/tcp                   # HTTPS
sudo ufw allow 8443/tcp                  # agent gateway mTLS
sudo ufw --force enable
```

For a license server, drop 8443 (no agents connect to it directly).

## DNS

Stable hostnames matter. Two patterns:

- **Vendor (license server only)**: `license.example.com` →
  vendor-controlled DNS, public.
- **Customer (manage server)**: `manage.customer.example.com` → customer's
  DNS, may be internal-only.

Agents use the manage server's mTLS gateway hostname embedded in their
enrolment bundle. If it changes after agents are deployed, you re-enrol.

## Backup baseline

Both apps store all state in PostgreSQL. Daily `pg_dump` is enough:

```bash
# License Server
podman exec triton-license-db pg_dump -U triton triton_license \
  | gzip > /var/backups/triton/license-$(date +%F).sql.gz

# Manage Server
podman exec triton-manage-db pg_dump -U triton triton_manage \
  | gzip > /var/backups/triton/manage-$(date +%F).sql.gz
```

The license server also needs the Ed25519 signing key file backed up.
Lose it and every customer has to re-activate against a new key.

The manage server also needs the PostgreSQL vault key (or the HashiCorp
Vault root token if you use the Vault sidecar). Lose it and stored host
credentials become unreadable.

## What's next

- Installing **License Server**? Read [license-server.md](license-server.md).
- Installing **Manage Server**? Read [manage-server.md](manage-server.md).
