# Manage Server Deployment

The Manage Server is the **on-prem** Triton component. Each customer runs
one. It owns:

- The customer's user accounts and login flow (admin + network engineers)
- The host inventory and tag system
- Scan job orchestration (SSH + agent + portscan workers)
- Embedded report server (results, dashboards, NACSA Arahan 9 views)
- The agent gateway (mTLS, port 8443) for endpoint enrolment
- Stored host credentials, encrypted at rest in the local vault

It activates against a vendor-operated **License Server** to validate its
licence. Without that activation, it runs in free tier (limited features).

> Read [prerequisites.md](prerequisites.md) before starting. You also need
> the License Server's public key — get it from your vendor.

## Quick start

```bash
curl -fsSL https://raw.githubusercontent.com/primatekuntech/triton-install/main/get.sh \
    | sudo bash -s -- \
        --license-server-pubkey 2c59996cff525418d88fd3c9b51e350ee3cb615a84f7af44a69a7aca28bc8451 \
        --license-server-url    https://license.vendor.example.com \
        --gateway-hostname      manage.customer.example.com \
        --manage-host-ip        10.0.1.50
```

The script:

1. Detects Podman or Docker.
2. Generates: Postgres password, JWT signing key, worker-fleet shared
   secret, AES-GCM vault key.
3. Writes `.env` with mode `0600`.
4. Pulls the image and starts containers: `triton-manage-db`,
   `triton-manageserver`.
5. Waits for the server to respond on `:8082`.

Open `http://localhost:8082` and complete the setup wizard:

1. Create the first admin (email + password).
2. Paste the licence token issued by your vendor's License Server.
3. Done.

## Production install

### TLS via reverse proxy (recommended)

The container speaks plain HTTP on `:8082`. Terminate TLS at Caddy or
Nginx (see [prerequisites.md](prerequisites.md)). Once the proxy is up:

```bash
curl -fsSL https://raw.githubusercontent.com/primatekuntech/triton-install/main/get.sh \
    | sudo bash -s -- \
        --license-server-pubkey HEX \
        --license-server-url    https://license.vendor.example.com \
        --gateway-hostname      manage.customer.example.com \
        --manage-host-ip        10.0.1.50 \
        --no-tls
```

`--no-tls` skips the manage-server's own TLS check; the proxy handles HTTPS.

### TLS in the container

Set cert + key paths in `.env` after the install completes:

```env
TRITON_MANAGE_TLS_CERT=/etc/triton/tls/fullchain.pem
TRITON_MANAGE_TLS_KEY=/etc/triton/tls/privkey.pem
TLS_CERT_HOST_DIR=/etc/triton/tls
```

then run the upgrade one-liner to recreate the container:

```bash
curl -fsSL https://raw.githubusercontent.com/primatekuntech/triton-install/main/get.sh \
    | sudo bash -s -- --upgrade
```

Note: the **agent gateway** (port 8443) ALWAYS uses mTLS with a self-managed
CA. It does not need or use the public TLS cert above. Don't proxy 8443.

## What got created

Installer files are saved to:
- **Linux**: `/opt/triton-manage-server/`
- **macOS**: `~/.local/share/triton-manage-server/`

```
/opt/triton-manage-server/     (or ~/.local/share/triton-manage-server/ on macOS)
├── compose.yaml         # bundled stack (postgres + manage-server)
├── env.template         # reference; do not edit
├── .env                 # YOUR secrets; mode 0600; back this up
└── *.sh                 # install / upgrade / uninstall
```

Containers:

| Container | Image | Purpose |
|-----------|-------|---------|
| `triton-manage-db` | `postgres:18-alpine` | DB; volume `triton-manage-db-data` |
| `triton-manageserver` | `triton-manageserver:latest` | API + admin UI + report server |

Volumes:

- `triton-manage-db-data` — all manage-server data (users, hosts, scans, credential ciphertext).
- `triton-manage-bins` — worker binaries synced from the License Server.

## Connection to the License Server

Two values link this manage server to the vendor:

| Setting | What it is | Where to set it |
|---------|------------|-----------------|
| `TRITON_MANAGE_LICENSE_SERVER_PUBKEY` | Vendor's Ed25519 public key (64 hex chars) | `--license-server-pubkey` flag at install |
| `TRITON_LICENSE_SERVER_URL` | Vendor's License Server public URL | `--license-server-url` flag at install |
| Licence token | Per-customer JWT issued by vendor | Pasted in setup wizard or `--license-token` flag |

The manage server activates by POSTing the licence token to
`/api/v1/license/activate` on the License Server. Activation binds this
manage server's hostname + machine fingerprint to a seat under the
vendor's customer org.

After activation, the manage server validates with the License Server every
24 hours. If unreachable for 14 days (or whatever
`TRITON_LICENSE_SERVER_STALE_THRESHOLD` is set to on the licence side), the
manage server falls back to free tier until it reconnects.

## First-run admin tasks

After login at `http://localhost:8082`:

1. **Create network engineer accounts.** Admin → Users.
2. **Add hosts.** Inventory → Hosts → New (or import via CSV / discovery scan).
3. **Add credentials.** Admin → Credentials. SSH keys, passwords, ssh_bastion configs.
4. **Run a scan.** Operations → Scan Jobs → New.
5. **(Optional) Enrol agents.** Admin → Agents → Generate enrolment token,
   install on the endpoint via the install URL.

Full UI walkthrough: [../MANAGE_PORTAL_USER_MANUAL.md](../MANAGE_PORTAL_USER_MANUAL.md).

## Day-2 operations

### Backup

```bash
mkdir -p /var/backups/triton
podman exec triton-manage-db pg_dump -U triton triton_manage \
    | gzip > /var/backups/triton/manage-$(date +%F).sql.gz

# Back up .env (vault key + JWT key + worker key live here)
cp /opt/triton-manage-server/.env /var/backups/triton/manage-env-$(date +%F)
```

Daily via `cron`. Retain 14+ days.

### Restore

```bash
gunzip < /var/backups/triton/manage-2026-01-15.sql.gz \
    | podman exec -i triton-manage-db psql -U triton triton_manage
```

The `.env`'s `TRITON_VAULT_KEY` MUST match the dump's era. If they don't,
stored host credentials are unrecoverable (the data is encrypted at rest).
Re-enter credentials in the portal.

### Upgrade

```bash
# Latest from ghcr.io
curl -fsSL https://raw.githubusercontent.com/primatekuntech/triton-install/main/get.sh \
    | sudo bash -s -- --upgrade

# Pin a specific version
curl -fsSL https://raw.githubusercontent.com/primatekuntech/triton-install/main/get.sh \
    | sudo bash -s -- --upgrade --image ghcr.io/primatekuntech/triton-manage-server:1.4.2
```

Pre-upgrade `pg_dump` runs automatically. Migrations run on container
start. The DB stays up; only the manage-server container is recreated.

Logs:

```bash
podman logs -f triton-manageserver
```

### Rollback

```bash
curl -fsSL https://raw.githubusercontent.com/primatekuntech/triton-install/main/get.sh \
    | sudo bash -s -- --upgrade --image ghcr.io/primatekuntech/triton-manage-server:1.4.1
```

If a new version's migration is incompatible with the older binary,
restore the pre-upgrade dump first:

```bash
gunzip < /var/backups/triton/manage-pre-upgrade-2026-01-15-...sql.gz \
    | podman exec -i triton-manage-db psql -U triton triton_manage
curl -fsSL https://raw.githubusercontent.com/primatekuntech/triton-install/main/get.sh \
    | sudo bash -s -- --upgrade --image ghcr.io/primatekuntech/triton-manage-server:<PREVIOUS_TAG>
```

### Uninstall

```bash
# Stop + remove containers, KEEP data
curl -fsSL https://raw.githubusercontent.com/primatekuntech/triton-install/main/get.sh \
    | sudo bash -s -- --uninstall

# Also delete DB + bins + vault (DESTRUCTIVE)
curl -fsSL https://raw.githubusercontent.com/primatekuntech/triton-install/main/get.sh \
    | sudo bash -s -- --uninstall --purge-data
```

`.env` is never deleted automatically. **If the host had agents enrolled,
they keep trying to call back** — kill the agents on the endpoints first
or re-enrol them against a replacement manage server.

### Manually trigger binary sync

The Manage Server pulls worker binaries (`triton-sshagent`, `triton-portscan`)
from the License Server on startup and after each successful licence
heartbeat. To force a sync — say, immediately after the License Server
operator publishes a new release — use the admin UI:

Admin → Binaries → **Sync now**.

Or via API:

```bash
curl -X POST -H "Authorization: Bearer $JWT" \
    http://localhost:8082/api/v1/admin/binaries/sync
```

Returns `202` (started) or `409` (sync already running).

### Re-activate against a different License Server

If you're moving customers off one License Server onto another:

1. Issue a new licence token on the new License Server.
2. Edit `.env`:
   ```env
   TRITON_LICENSE_SERVER_URL=https://license.newvendor.com
   TRITON_MANAGE_LICENSE_SERVER_PUBKEY=<new vendor's pubkey>
   TRITON_LICENSE_TOKEN=<new licence token>
   ```
3. Run the upgrade one-liner to restart the container with the new env (re-activation happens on first heartbeat):
   ```bash
   curl -fsSL https://raw.githubusercontent.com/primatekuntech/triton-install/main/get.sh \
       | sudo bash -s -- --upgrade
   ```

### Rotate the JWT signing key

Every active session is invalidated; users re-login.

```bash
sed -i "s|^TRITON_MANAGE_JWT_SIGNING_KEY=.*|TRITON_MANAGE_JWT_SIGNING_KEY=$(openssl rand -hex 32)|" \
    /opt/triton-manage-server/.env
curl -fsSL https://raw.githubusercontent.com/primatekuntech/triton-install/main/get.sh \
    | sudo bash -s -- --upgrade
```

### Rotate the vault key

Stored credentials are encrypted with `TRITON_VAULT_KEY`.
Rotating the key without re-encrypting orphans the existing credentials.

There's no automatic re-encryption tool today. Recipe:

1. Export each credential via the API (it gets decrypted with the OLD key).
2. Update `TRITON_VAULT_KEY` in `/opt/triton-manage-server/.env`.
3. Run the upgrade one-liner to restart with the new key:
   ```bash
   curl -fsSL https://raw.githubusercontent.com/primatekuntech/triton-install/main/get.sh \
       | sudo bash -s -- --upgrade
   ```
4. Re-enter each credential through the UI (encrypts with NEW key).

## Sizing reference

| Hosts in inventory | Recommended | Notes |
|--------------------|-------------|-------|
| ≤500 | 2 vCPU / 4 GB / 50 GB | default `PARALLELISM=10` is fine |
| 500–2000 | 4 vCPU / 8 GB / 100 GB | bump `PARALLELISM=20` |
| 2000+ | 8 vCPU / 16 GB / 200 GB+ | `PARALLELISM=30+`; consider PG tuning |

Scan results dominate disk. A scan finds 100–500 crypto assets per host
on average; each finding is ~2 KB JSON. Plan for 200 MB of DB growth per
1000 host-scans.

## Troubleshooting

### Setup wizard appears after restart (state seems to reset)

The setup wizard checks the `manage_setup` table for `admin_created` and
`license_activated` flags. If the wizard reappears, the DB is fresh —
either the volume was deleted, or you connected to a different DB URL.

If state seems to "reset" on every restart but the DB is the same: usually
the JWT key in `.env` rotated, invalidating sessions. The setup wizard
itself only runs once.

### "licence validation failed: invalid signature"

The licence token was signed with a different Ed25519 key than the public
key in `TRITON_MANAGE_LICENSE_SERVER_PUBKEY`. Causes:

- The vendor's License Server signing key was rotated. Get a new token.
- You set the wrong public key (e.g., used the seed half not the pub half;
  `TRITON_LICENSE_SERVER_SIGNING_KEY` is `seed||pub`, you want the LAST
  64 hex chars).

The server runs in free tier in this state. Fix the keys, restart.

### Credential vault not configured

The startup log prints the vault status. If `TRITON_VAULT_KEY` is not set,
the credential API returns 503. Set `TRITON_VAULT_KEY` to a 64 hex-char
(32-byte) AES key and restart. Back it up — losing the key makes all stored
host credentials unreadable.

### Agents can't connect to the gateway

Agents resolve `TRITON_MANAGE_GATEWAY_HOSTNAME` and connect to `:8443`.
Common issues:

- The hostname doesn't resolve from the agent's network.
- A firewall blocks 8443 inbound to the manage server.
- The hostname is correct but the agent was enrolled with a stale URL —
  re-enrol after changing `TRITON_MANAGE_GATEWAY_HOSTNAME`.

```bash
# From the agent host:
openssl s_client -connect manage.customer.example.com:8443 </dev/null
```

### Workers can't claim jobs

Workers (`triton-sshagent`, `triton-portscan`) authenticate with
`TRITON_MANAGE_WORKER_KEY`. If they get 401s on `/api/v1/scan-jobs/claim`,
the worker's stored key doesn't match the manage server's. Re-deploy the
worker with the current key (it's in `.env`).

### Container restart loop

```bash
podman logs --tail 30 triton-manageserver
```

Look for: missing `TRITON_MANAGE_DB_URL`, JWT key shorter than 32 bytes
after hex-decode, license-server-pubkey not 64 hex chars, DB pool ping
timeout (postgres still warming up).

## Reference

- Env var details — `/opt/triton-manage-server/env.template` (Linux) or `~/.local/share/triton-manage-server/env.template` (macOS)
- Manage Server feature reference — [../MANAGE_PORTAL_USER_MANUAL.md](../MANAGE_PORTAL_USER_MANUAL.md)
- Hosts model — [../MANAGE_SERVER_HOSTS.md](../MANAGE_SERVER_HOSTS.md)
- Architecture — [../SYSTEM_ARCHITECTURE.md](../SYSTEM_ARCHITECTURE.md)
