# Triton Deployment Documentation

Each application deploys independently. Pick the guide that matches what
you are installing.

## Apps

| App | Where it runs | Guide | Scripts |
|-----|---------------|-------|---------|
| **License Server** | Central / vendor-side | [license-server.md](license-server.md) · [vps-license-server.md](vps-license-server.md) ¹ | [`scripts/deploy/license-server/`](../../scripts/deploy/license-server/) |
| **Manage Server** | On-prem / customer-side | [manage-server.md](manage-server.md) | [`scripts/deploy/manage-server/`](../../scripts/deploy/manage-server/) |

Both run as containers via Podman or Docker. Each comes with a self-contained
`compose.yaml` that bundles its own PostgreSQL — the two apps do not share
infrastructure and can run on entirely separate hosts.

¹ `vps-license-server.md` covers deploying with a **locally built image** (no
registry required). Use this when you have not yet published an image to ghcr.io.

## Which one do I need?

```
                    ┌────────────────────┐
                    │   License Server   │   ← you (vendor) run this
                    │   (issues licences)│      one instance, central
                    └─────────┬──────────┘
                              │ HTTPS validate / activate
                              │ HTTPS download worker binaries
                              ▼
                    ┌────────────────────┐
                    │   Manage Server    │   ← each customer runs this
                    │   (admin portal)   │      on their network
                    └─────────┬──────────┘
                              │ mTLS (gateway :8443)
                              ▼
                    ┌────────────────────┐
                    │   Triton agents    │   ← installed on endpoints
                    │   (scan workers)   │      via the manage portal
                    └────────────────────┘
```

- **You're a vendor / SaaS operator** running the licence + binary fleet for
  your customers → install **License Server**.
- **You're a customer / on-prem operator** running scans inside your own
  network and want a portal for your team → install **Manage Server**.
- **You're testing the full stack on one machine** → install both, or use the
  full-stack guide at [../DEPLOYMENT_GUIDE.md](../DEPLOYMENT_GUIDE.md).

## Before you start

Read [prerequisites.md](prerequisites.md) once. It covers the host OS prep,
Podman/Docker install, port + firewall layout, and TLS basics that apply to
either deployment.

## After install

Each app's guide has its own Day-2 sections:

- Backups
- Upgrades (zero-downtime where possible)
- Rotation of secrets and keypairs
- Audit log retention
- Rollback procedures

## Legacy docs

The full-stack [DEPLOYMENT_GUIDE.md](../DEPLOYMENT_GUIDE.md) and
[UBUNTU_DEPLOYMENT_GUIDE.md](../UBUNTU_DEPLOYMENT_GUIDE.md) describe the
combined-install path. They remain valid but are superseded by the per-app
guides for production deployments where licence and manage servers live on
different hosts.
