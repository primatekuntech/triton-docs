# License Server Deployment

The License Server is the **central** Triton component. It owns:

- The signing keypair that mints all licence tokens
- Customer organisations and their assigned licences
- Per-customer activation records (one per Manage Server instance)
- Worker binaries (sshagent, portscan) served via signed install URLs
- A platform-admin web UI for issuing and revoking licences

You run **one** License Server. Each customer's Manage Server activates
against it.

> Read [prerequisites.md](prerequisites.md) before starting.

## Quick start

```bash
git clone --depth 1 https://github.com/primatekuntech/triton.git
cd triton/scripts/deploy/license-server
sudo bash install.sh \
    --admin-email admin@yourco.com \
    --public-url https://license.yourco.com
```

That's it for an HTTP test. The script:

1. Detects Podman or Docker.
2. Generates a Postgres password, an Ed25519 signing keypair, and a
   bootstrap admin password.
3. Writes `.env` with mode `0600` in the same directory.
4. Pulls the image and starts two containers: `triton-license-db`,
   `triton-licenseserver`.
5. Waits for the health endpoint and prints the admin password.

The admin UI is at `http://localhost:8081/ui/`. **Rotate the bootstrap
password immediately** via Account → Change password.

## Production install with TLS

Recommended path: terminate TLS at a reverse proxy (Caddy or Nginx),
proxy to the container's `:8081`. See [prerequisites.md](prerequisites.md)
for sample configs.

```bash
sudo bash install.sh \
    --admin-email admin@yourco.com \
    --public-url https://license.yourco.com \
    --no-tls   # the container speaks HTTP; TLS is at the proxy
```

For container-native TLS (no proxy), edit `.env` after install:

```env
TRITON_LICENSE_SERVER_TLS_CERT=/etc/triton/tls/fullchain.pem
TRITON_LICENSE_SERVER_TLS_KEY=/etc/triton/tls/privkey.pem
TLS_CERT_HOST_DIR=/etc/triton/tls    # mounted into the container
```

then `bash upgrade.sh` to recreate the container.

## What got created

```
scripts/deploy/license-server/
├── compose.yaml         # bundled stack (postgres + license-server)
├── env.template         # reference; do not edit
├── .env                 # YOUR secrets; mode 0600; back this up
└── *.sh                 # install / upgrade / uninstall
```

Containers:

| Container | Image | Purpose |
|-----------|-------|---------|
| `triton-license-db` | `postgres:18-alpine` | DB; volume `triton-license-db-data` |
| `triton-licenseserver` | `triton-licenseserver:latest` | API + admin UI |

Volumes / mounts:

| Name / Path | Purpose |
|-------------|---------|
| `triton-license-db-data` | All licence metadata (orgs, licences, activations, audit). Back this up. |
| `TRITON_LICENSE_SERVER_HOST_BIN_DIR` (default `/opt/triton/binaries`) | Worker binaries stored as files on the host. Survives container rebuilds. Back this up separately. |

## Worker binaries

Worker binaries (`triton-agent`, `triton-sshagent`, `triton-portscan`) are
stored as **files on the host**, not in the database. This avoids loading
multi-hundred-MB blobs into PostgreSQL and allows uploads of any size up to
the 512 MB per-request limit.

The host directory is bind-mounted at `/data/binaries` inside the container.
Set `TRITON_LICENSE_SERVER_HOST_BIN_DIR` in `.env` to choose the host path
(default: `/opt/triton/binaries`). `install.sh` creates it automatically.

Upload binaries after first login via **Binaries** in the admin UI.
Uploaded files are immediately available via the install URL
(`/api/v1/install/{token}/binaries/{name}/{os}/{arch}`) used by agents.

**Backup note:** the binary directory is not included in `pg_dump`. Add it
to your backup routine — see [Day-2 operations → Backup](#backup) below.

## Verify before starting

Run the env check to confirm `.env` is ready before bringing up the stack:

```bash
cd triton
make verify-env-license
# or directly:
./scripts/verify-license-env.sh /path/to/.env
```

The script checks required vars, signing key format, TLS consistency, binary
directory existence and writability, and optional integrations. Fix any ✗
before proceeding.

## First-run admin tasks

After login (`http://localhost:8081/ui/` or your public URL):

1. **Rotate the bootstrap password.**
2. **Upload worker binaries.** Navigate to **Binaries** and upload each
   binary (`triton-agent`, `triton-sshagent`, `triton-portscan`) for the
   platforms you support. Binaries are stored at
   `TRITON_LICENSE_SERVER_HOST_BIN_DIR` on the host.
3. **Create your first organisation.** Navigate to Organisations → New.
   Fill in name, contact name, contact email. The contact gets the
   30-day expiry warning emails.
4. **Issue a licence to that organisation.** Licences → New. Choose
   tier (free / pro / enterprise), expiry, seat count.
5. **Send the licence token to the customer.** They paste it into their
   Manage Server's setup wizard.
6. **(Optional)** Invite additional platform admins. Superadmins → Invite.
   Requires `RESEND_API_KEY` set in `.env` for email delivery; otherwise
   the temp password prints in the audit log.

For full UI walkthrough see [../LICENSE_SERVER_GUIDE.md](../LICENSE_SERVER_GUIDE.md).

## Day-2 operations

### Backup

Three things to back up:

```bash
mkdir -p /var/backups/triton

# 1. Database — all licence metadata
podman exec triton-license-db pg_dump -U triton triton_license \
    | gzip > /var/backups/triton/license-$(date +%F).sql.gz

# 2. .env — signing key lives here; losing it forces all customers to re-activate
cp scripts/deploy/license-server/.env /var/backups/triton/license-env-$(date +%F)

# 3. Worker binaries — not in the DB; stored at TRITON_LICENSE_SERVER_HOST_BIN_DIR
tar -czf /var/backups/triton/license-binaries-$(date +%F).tar.gz \
    /opt/triton/binaries   # adjust path to match your TRITON_LICENSE_SERVER_HOST_BIN_DIR
```

Schedule daily via `cron` or systemd timer. Keep at least 14 days of dumps.

### Restore

```bash
gunzip < /var/backups/triton/license-2026-01-15.sql.gz \
    | podman exec -i triton-license-db psql -U triton triton_license
```

The signing key in `.env` MUST match the dump's era. If you lose the key,
you cannot reissue licences that match existing customer activations —
every customer has to re-activate.

### Upgrade

```bash
cd scripts/deploy/license-server
sudo bash upgrade.sh                          # latest from ghcr.io
sudo bash upgrade.sh --image ghcr.io/.../triton-licenseserver:1.4.2
```

The script takes a pre-upgrade `pg_dump` to `/var/backups/triton/`,
pulls the new image, and recreates the `license-server` container only
(DB stays up).

Migrations run automatically on container start. Logs:

```bash
podman logs -f triton-licenseserver
```

### Rollback

```bash
sudo bash upgrade.sh --image ghcr.io/.../triton-licenseserver:1.4.1
```

Schema migrations don't auto-rollback. If a new version added a column
the old version doesn't know about, the old version usually still runs
(it just ignores the new column). If a new version dropped a column the
old version expected, restore the pre-upgrade dump:

```bash
gunzip < /var/backups/triton/license-pre-upgrade-2026-01-15-...sql.gz \
    | podman exec -i triton-license-db psql -U triton triton_license
sudo bash upgrade.sh --image <PREVIOUS_TAG>
```

### Uninstall

```bash
sudo bash uninstall.sh                # stop + remove containers, KEEP data
sudo bash uninstall.sh --purge-data   # also delete the DB volume (DESTRUCTIVE)
```

`.env` is never deleted automatically.

### Rotate signing key

This is a serious operation — every customer has to re-activate.

1. Stop accepting new activations: announce maintenance window.
2. `bash uninstall.sh` (keeps DB volume).
3. Generate a new keypair (rerun `install.sh` after deleting `.env`, or
   manually update `TRITON_LICENSE_SERVER_SIGNING_KEY` in `.env`).
4. `bash install.sh` again — same DB, new key.
5. Push new licence tokens to every customer.

## Sizing reference

- Each licence is ~2 KB in the DB.
- Each activation record is ~500 B.
- Audit log retention: capped at 10,000 entries by default; configure via
  the admin UI Settings page.
- Worker binary storage: ~10–400 MB per binary per platform/arch combination
  (varies by binary). Allocate at least 5 GB for `TRITON_LICENSE_SERVER_HOST_BIN_DIR`.
- A single-vCPU 2 GB box handles 1000+ customers comfortably as long as
  PostgreSQL has the RAM.

## Troubleshooting

### "license server has no users and no bootstrap password set"

The DB has zero users and `TRITON_LICENSE_SERVER_ADMIN_PASSWORD` is blank.
Set the env var, restart:

```bash
sed -i 's|^TRITON_LICENSE_SERVER_ADMIN_PASSWORD=.*|TRITON_LICENSE_SERVER_ADMIN_PASSWORD=somenewpass|' .env
sudo bash upgrade.sh   # recreates the container, seeds the user
```

### Container boots but health check fails

```bash
podman logs triton-licenseserver | tail -30
```

Common causes: DB still starting (the script waits, but a cold-cache
postgres on slow disk can exceed 60s), stale Postgres password (delete
the volume and reinstall in dev only), missing `TRITON_LICENSE_SERVER_SIGNING_KEY`.

### "TLS is not configured and TRITON_LICENSE_SERVER_ALLOW_INSECURE is not set"

The server refuses to start without TLS in production mode. Fix one of:

- Configure TLS (cert paths in `.env`)
- Set `TRITON_LICENSE_SERVER_ALLOW_INSECURE=1` (dev only)
- Run behind a TLS-terminating proxy and set ALLOW_INSECURE=1

### Can't reach `/api/v1/license/health`

```bash
podman ps                                     # is the container running?
podman port triton-licenseserver              # is :8081 published?
ss -tlnp | grep 8081                          # is the port listening?
curl -v http://localhost:8081/api/v1/license/health
```

### Bootstrap password got lost

```bash
sudo grep ADMIN_PASSWORD scripts/deploy/license-server/.env
```

If the password has already been changed via the UI, the env var is
stale (only used on first boot). Reset via DB:

```bash
podman exec -it triton-license-db psql -U triton triton_license \
    -c "UPDATE users SET must_change_password = TRUE WHERE email = 'admin@yourco.com'"
```

Then have an existing platform admin reset it via the UI.

## Reference

- Env var details — [env.template](../../scripts/deploy/license-server/env.template)
- License Server feature reference — [../LICENSE_SERVER_GUIDE.md](../LICENSE_SERVER_GUIDE.md)
- Architecture — [../SYSTEM_ARCHITECTURE.md](../SYSTEM_ARCHITECTURE.md)
- API surface — `pkg/licenseserver/`
