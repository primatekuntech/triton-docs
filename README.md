# Triton Documentation

Official documentation for Triton — enterprise PQC compliance scanner for Malaysian government NACSA/CNSA requirements.

## Contents

### User Guides
- [User Manual](docs/USER_MANUAL.md) — CLI usage, scan profiles, report formats
- [Manage Portal User Manual](docs/MANAGE_PORTAL_USER_MANUAL.md) — Web UI walkthrough
- [Managing Hosts](docs/MANAGE_SERVER_HOSTS.md) — Host inventory, credentials, tags
- [Scanning Agents](docs/SCANNING_AGENTS.md) — Agent installation, configuration, scheduling

### Deployment
- [Deployment Guide](docs/DEPLOYMENT_GUIDE.md) — Full production deployment reference
- [Ubuntu Deployment Guide](docs/UBUNTU_DEPLOYMENT_GUIDE.md) — Step-by-step Ubuntu setup
- [Prerequisites](docs/deployment/prerequisites.md)
- [Manage Server](docs/deployment/manage-server.md)
- [License Server](docs/deployment/license-server.md)
- [VPS License Server](docs/deployment/vps-license-server.md)

### Licensing
- [License Key Guide](docs/LICENSE_KEY_GUIDE.md) — Licence tiers, activation, machine binding
- [License Server Guide](docs/LICENSE_SERVER_GUIDE.md) — Self-hosted licence server setup

### Architecture & Reference
- [System Architecture](docs/SYSTEM_ARCHITECTURE.md)
- [Fleet Scan Architecture](docs/FLEET_SCAN_ARCHITECTURE.md)

### Scanner Reference
- [ASN.1 OID Scanner](docs/scanners/asn1_oid.md)
- [Hybrid PQC Detection](docs/scanners/hybrid_pqc.md)
- [Java Bytecode Scanner](docs/scanners/java_bytecode.md)

### Examples
- [Agentless Scanning](docs/examples/agentless/README.md)
- [Fleet Scan](docs/examples/fleet-scan/README.md)

---

## Quick Install

```bash
curl -fsSL https://raw.githubusercontent.com/primatekuntech/triton-install/main/manage-server/install.sh | sudo bash
```

See [primatekuntech/triton-install](https://github.com/primatekuntech/triton-install) for full installation instructions.
