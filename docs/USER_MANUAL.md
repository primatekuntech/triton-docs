# Triton User Manual

**Version 3.0** | Post-Quantum Cryptography Compliance Scanner

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Installation](#2-installation)
3. [Quick Start](#3-quick-start)
4. [Scan Profiles](#4-scan-profiles)
5. [Scanner Modules](#5-scanner-modules)
6. [Scan Flags and Options](#6-scan-flags-and-options)
7. [Resource Limits](#7-resource-limits)
8. [Detached Scans](#8-detached-scans)
9. [Output Formats and Reports](#9-output-formats-and-reports)
10. [Incremental Scanning and Caching](#10-incremental-scanning-and-caching)
11. [Policy Compliance Engine](#11-policy-compliance-engine)
12. [Agent Mode](#12-agent-mode)
13. [Fleet and Device Scanning](#13-fleet-and-device-scanning)
14. [Report Server](#14-report-server)
15. [Manage Portal](#15-manage-portal)
16. [License Server](#16-license-server)
17. [License Tiers and Feature Gating](#17-license-tiers-and-feature-gating)
18. [PQC Algorithm Classification](#18-pqc-algorithm-classification)
19. [Utilities and Diagnostics](#19-utilities-and-diagnostics)
20. [Configuration Reference](#20-configuration-reference)
21. [Glossary](#21-glossary)

---

## 1. Introduction

Triton is an enterprise-grade cryptographic asset scanner designed for Malaysian government Post-Quantum Cryptography (PQC) compliance assessment. It discovers cryptographic assets across an organisation's entire technology stack, classifies them by PQC readiness, and produces SBOM/CBOM reports in the formats required for national compliance frameworks (NACSA-2030, CNSA 2.0).

### 1.1 What Triton Scans

Triton covers all nine CBOM-scanning categories defined in national PQC assessment guidelines:

- **Category 1** — Runtime processes and active cryptographic operations
- **Category 2** — Binaries and executable files on disk
- **Category 3** — Cryptographic libraries
- **Category 4** — Kernel modules
- **Category 5** — Certificates, keys, and key stores
- **Category 6** — Source code, scripts, and compiled bytecode
- **Category 7** — Web applications and web server configuration
- **Additional** — Network protocols, infrastructure, package managers, containers, databases, cloud-native workloads, and more

### 1.2 Product Components

| Component | Binary | Purpose |
|---|---|---|
| Triton CLI | `triton` | Standalone scanner + report generator |
| Report Server | `triton server` | REST API + embedded web UI (PostgreSQL-backed) |
| Manage Portal | `triton-manage` container | On-prem orchestration for SSH-agent and port-survey scans |
| Triton Agent | `triton-agent` | Persistent enrolled-agent daemon for Manage Portal |
| SSH Agent | `triton-sshagent` | Agentless SSH scan worker |
| Port Scanner | `triton-portscan` | Network discovery and port survey worker |
| License Server | `triton-license-server` | Centralised license management |

The Report Server, Manage Portal, and License Server are optional and can be deployed independently. The CLI works fully standalone.

---

## 2. Installation

### 2.1 Pre-built Binaries

Download the appropriate release archive from GitHub Releases. Triton ships pre-built for:

- Linux: `amd64`, `arm64`
- macOS: `amd64`, `arm64`
- Windows: `amd64`

```bash
# Linux amd64
curl -LO https://github.com/primatekuntech/triton/releases/latest/download/triton_linux_amd64.tar.gz
tar -xzf triton_linux_amd64.tar.gz
sudo install -m 755 triton /usr/local/bin/triton
```

### 2.2 Building from Source

Requires Go 1.25 or later.

```bash
git clone https://github.com/primatekuntech/triton.git
cd triton
make build          # builds bin/triton for the current platform
make build-all      # cross-compiles all platforms
```

### 2.3 Container

```bash
# Build locally
make container-build

# Start the full stack (PostgreSQL + Triton server)
make container-run

# Stop the full stack
make container-stop
```

The container image is also published to `ghcr.io/primatekuntech/triton` on each release.

### 2.4 System Requirements

| Component | Minimum | Notes |
|---|---|---|
| CPU | 2 cores | Worker count is capped by CPU count |
| RAM | 512 MB | Comprehensive profile with eBPF requires more |
| Disk | 200 MB | For binaries + working files |
| OS | Linux, macOS, Windows | eBPF, UEFI, TPM modules are Linux-only |
| Go | 1.25+ | Build-time requirement only |
| PostgreSQL | 18+ | Required for Report Server and Manage Portal |

### 2.5 Optional System Dependencies

Some scanner modules require external tools. Run `triton doctor` to check which are present.

| Tool | Module | Notes |
|---|---|---|
| `jarsigner` | `java_bytecode` | JAR/WAR/EAR signature verification |
| `osslsigncode` | `codesign` | Authenticode PE signature verification |
| `fingerprintx` | `triton-portscan` | Port survey service fingerprinting |
| `libpcap` | `tls_observer` | Live packet capture (Linux) |
| Root/CAP_BPF | `ebpf_trace` | eBPF runtime crypto tracing |
| Root/CAP_NET_RAW | `tls_observer` | Live AF_PACKET capture |

---

## 3. Quick Start

### 3.1 Your First Scan

Run a quick scan of the current directory and print a summary:

```bash
triton --profile quick --format json --output report.json /path/to/scan
```

Run a standard scan with all allowed output formats:

```bash
triton --profile standard --format all --output-dir ./reports /path/to/scan
```

Scan with PQC compliance policy evaluation:

```bash
triton --profile standard --policy nacsa-2030 --output-dir ./reports /path/to/scan
```

### 3.2 Scan a Remote Host (agentless)

```bash
triton device-scan \
  --inventory /etc/triton/devices.yaml \
  --credentials /etc/triton/credentials.yaml
```

### 3.3 View the Doctor Report

Before your first scan, check system readiness:

```bash
triton doctor
```

This checks for required tools, optional dependencies, database connectivity, and license status.

---

## 4. Scan Profiles

Scan profiles control which modules run, how deep the file system traversal goes, and how many concurrent workers execute.

| Profile | Module Count | Depth | Workers | Use Case |
|---|---|---|---|---|
| `quick` | 3 | 3 | 4 | Fast pre-flight check; certificates, keys, packages only |
| `standard` | ~30 | 10 | 8 | Typical compliance scan; covers all primary asset categories |
| `comprehensive` | 56 | unlimited | 16 | Full audit; includes bytecode analysis, eBPF, TPM, UEFI, pcap |

Worker count is capped at the host CPU count regardless of profile setting.

### 4.1 Quick Profile

Runs: `certificates`, `keys`, `packages`

Suitable for a rapid health check or CI/CD gating where scan time must be minimal.

### 4.2 Standard Profile

Runs: `certificates`, `keys`, `packages`, `libraries`, `binaries`, `scripts`, `webapp`, `configs`, `containers`, `certstore`, `database`, `deps`, `web_server`, `vpn`, `password_hash`, `deps_ecosystems`, `mail_server`, `dnssec`, `netinfra`, `messaging`, `db_atrest`, `archive`, `ftps`, `ssh_cert`, `ldif`, `python_ast`, `codesign`, `hsm`, `ldap`, `network`, `protocol`

Suitable for periodic compliance scans on production systems.

### 4.3 Comprehensive Profile

Runs all 56 modules, adding: `asn1_oid`, `java_bytecode`, `dotnet_il`, `ebpf_trace`, `tpm`, `uefi`, `tls_observer`, and all other specialised modules.

Requires Pro+ licence tier. Some modules have additional prerequisites (root, kernel 5.8+, BTF, libpcap).

### 4.4 Selecting a Profile

```bash
triton --profile quick /var/www
triton --profile standard /
triton --profile comprehensive /   # requires Pro+ licence
```

---

## 5. Scanner Modules

Triton has 56 scanner modules. Each module has a name that can be used with `--modules` to run a subset. Below is the complete list grouped by function.

### 5.1 Certificates and Keys (Category 5)

| Module | Description |
|---|---|
| `certificates` | X.509 certificates in PEM/DER/PKCS#12/JKS/JCEKS/BKS formats. Detects expired, weak-signature, short-key, SHA-1, MD5 issuers. Key-quality analysis: ROCA (CVE-2017-15361), Debian PRNG (CVE-2008-0166). |
| `keys` | Standalone private/public keys: RSA, EC, Ed25519, DSA. Includes SSH host keys (`/etc/ssh/ssh_host_*_key`). |
| `certstore` | OS certificate stores: Windows Certificate Store (certutil), macOS Keychain, Linux /etc/ssl and NSS databases. |
| `archive` | ZIP/TAR/JAR/WAR/EAR with 2-level nesting. Zip-bomb protection. Delegates cert/key parsing to existing modules. |
| `ldif` | LDIF files: extracts base64-encoded certificates from `userCertificate`, `cACertificate`, `userSMIMECertificate` attributes. RFC 2849 folded-line support. |

### 5.2 Runtime and Active Operations (Category 1)

| Module | Description |
|---|---|
| `process` | Running processes: inspects /proc (Linux) and equivalent paths for loaded cryptographic libraries, key material in memory-mapped files. |
| `ebpf_trace` | eBPF runtime crypto tracer: uprobes on libcrypto/GnuTLS/NSS and kprobes on the kernel crypto API. Observation-window scan. **Linux-only; requires root/CAP_BPF; kernel 5.8+ with BTF; comprehensive profile + Pro+ tier.** |

### 5.3 Binaries on Disk (Category 2)

| Module | Description |
|---|---|
| `binaries` | ELF, Mach-O, PE executables: symbol table analysis for imported/exported crypto symbols. |
| `asn1_oid` | ASN.1 OID byte scanner: walks ELF/Mach-O/PE read-only sections for DER-encoded OIDs. Classifies via the OID registry. **Comprehensive profile + Pro+ tier.** |

### 5.4 Cryptographic Libraries (Category 3)

| Module | Description |
|---|---|
| `libraries` | Shared and static libraries: `libssl`, `libcrypto`, OpenSSL, GnuTLS, NSS, BoringSSL, Bouncy Castle, and others. Version extraction and PQC readiness classification. |

### 5.5 Kernel Modules (Category 4)

| Module | Description |
|---|---|
| `kernel` | Linux kernel modules (`.ko`): identifies crypto algorithm registrations in `crypto_alg` structures. |

### 5.6 Source Code and Bytecode (Category 6)

| Module | Description |
|---|---|
| `scripts` | Shell, JavaScript, Ruby, PHP, Go, C/C++ source: regex-based crypto literal detection. |
| `python_ast` | Python source: full two-phase AST walk (import + call resolution). Reachability analysis (direct/transitive) via import graph BFS. Standard profile + Pro+ tier. |
| `java_bytecode` | Java `.class`/JAR/WAR/EAR: parses constant pool for JCA names, BouncyCastle, and PQC algorithm literals. **Comprehensive profile + Pro+ tier.** |
| `dotnet_il` | .NET assemblies: parses CLI metadata (TypeRef table + #US heap) for BCL types, CAPI/CNG constants, BouncyCastle.NET PQC. Supports classic and single-file bundles. **Comprehensive profile + Pro+ tier.** |

### 5.7 Web Applications (Category 7)

| Module | Description |
|---|---|
| `webapp` | Web application files: JavaScript bundles, JSP, ASP, PHP. Detects crypto algorithm names in source. |

### 5.8 Network and Protocol

| Module | Description |
|---|---|
| `network` | Network scan: probe live hosts for TLS certificates and protocol versions. |
| `protocol` | Enhanced TLS probing: cipher enumeration, TLS version range (1.0–1.3), cipher preference order, key exchange/PFS classification, chain validation. Hybrid PQC named-group classification (X25519MLKEM768, SecP256r1MLKEM768, draft Kyber hybrids). |
| `ftps` | FTPS certificate discovery: explicit AUTH TLS (port 21) + implicit FTPS (port 990). Standard profile + Pro+ tier. |
| `ssh_cert` | SSH certificate scanner: network SSH handshake to extract host key algorithm/size and OpenSSH certificate metadata. Standard profile + Pro+ tier. |
| `tls_observer` | Passive TLS pcap/wire observer: reads `.pcap`/`.pcapng` files or live AF_PACKET capture (Linux). Extracts ClientHello/ServerHello, computes JA3/JA3S/JA4/JA4S fingerprints. **Comprehensive profile + Pro+ tier; live capture requires root/CAP_NET_RAW.** |
| `mail_server` | Mail server TLS: SMTP STARTTLS (port 25/587), IMAPS (993), POP3S (995). |
| `dnssec` | DNSSEC zone signing: checks DNSKEY algorithm, key size, NSEC/NSEC3 mode. |
| `netinfra` | Network infrastructure: router/switch SSH banner analysis and config file scanning for IKE/IPsec policy. |

### 5.9 Configuration and Infrastructure

| Module | Description |
|---|---|
| `configs` | Configuration files: detects hardcoded keys, algorithm settings in `.conf`, `.ini`, `.xml`, `.yaml` files. |
| `web_server` | Web server TLS config: nginx `ssl_protocols`/`ssl_ciphers`, Apache `SSLProtocol`/`SSLCipherSuite`, HAProxy, Caddy. HSTS detection. |
| `vpn` | VPN configuration: strongSwan IPsec (IKE/ESP/AH/PFS), WireGuard, OpenVPN cipher/auth/tls-auth. |
| `database` | Database configuration: TLS connection strings, cipher settings in PostgreSQL, MySQL, Oracle, MSSQL configs. |
| `db_atrest` | Database at-rest encryption: Transparent Data Encryption (TDE) settings, key management references. |
| `password_hash` | Password hash discovery: identifies weak hash schemes (MD5, SHA-1, unsalted SHA-2) in `/etc/shadow`, config files, and database dumps. |

### 5.10 Packages and Dependencies

| Module | Description |
|---|---|
| `packages` | Package managers: APT/dpkg, RPM/yum, Homebrew, npm, pip. Extracts crypto library versions. |
| `deps` | Go module dependency crypto reachability analysis. Classifies crypto usage as `direct` (0.95), `transitive` (0.75), or `unreachable` (0.50). |
| `deps_ecosystems` | Multi-ecosystem dependency scanning: `package.json`, `requirements.txt`, `Gemfile`, `pom.xml`, `build.gradle`, `Cargo.toml`. |

### 5.11 Containers and Cloud

| Module | Description |
|---|---|
| `containers` | Docker and OCI container images: Dockerfile `FROM`, ENV, RUN instruction analysis; docker-compose TLS settings. |
| `container_signatures` | Container signing: cosign keys, Notary v1 trust stores, Sigstore root metadata, Kubernetes service account JWT header inspection, K8s encryption-config.yaml provider analysis. |
| `oci_image` | Local OCI image layer scanning: extracts and analyses image layers for crypto assets. |

### 5.12 Hardware and Firmware

| Module | Description |
|---|---|
| `tpm` | TPM 2.0 attestation: parses `/sys/class/tpm` sysfs + TCG PFP measured-boot log. Classifies firmware against CVE registry (Infineon ROCA, Intel PTT, STMicro ST33). **Linux-only; comprehensive profile + Pro+ tier.** |
| `uefi` | UEFI Secure Boot key inventory: parses `/sys/firmware/efi/efivars/` for PK/KEK/db certs + dbx revocation list. Checks dbx for missing CVE revocations (BlackLotus CVE-2023-24932, BootHole CVE-2020-10713, Baton Drop CVE-2022-21894). **Linux-only; comprehensive profile + Pro+ tier.** |
| `hsm` | Hardware Security Module: detects PKCS#11 slot/token metadata via `pkcs11-tool`. |
| `firmware` | Firmware images: extracts and analyses embedded certificates and crypto configs. |

### 5.13 Identity and Authentication

| Module | Description |
|---|---|
| `codesign` | Code signing: macOS `codesign`, cross-platform `osslsigncode` (Authenticode PE), `jarsigner` (JAR/WAR/EAR). |
| `ldap` | LDAP configuration: TLS/SASL settings in `slapd.conf`, `ldap.conf`. |
| `auth_material` | Authentication material: SSH `authorized_keys`, Kerberos keytabs, GSSAPI configs. |
| `credentials` | Credential files: `.netrc`, `.pgpass`, AWS credentials, Kubernetes secrets with embedded keys. |

### 5.14 Observability and Kubernetes

| Module | Description |
|---|---|
| `k8s_live` | Live Kubernetes cluster: API server certificates, etcd encryption config, service account signing keys, admission webhook TLS. |
| `service_mesh` | Service mesh: Istio/Envoy/Linkerd mTLS certificate inventory, cipher policy. |
| `secrets_mgr` | Secrets manager integration: Vault, AWS Secrets Manager, Azure Key Vault — key types and algorithms. |
| `supply_chain` | Supply chain: SBOM metadata (CycloneDX, SPDX) cross-referenced against known-weak algorithm lists. |
| `helm_chart` | Helm chart TLS and secret configurations. |
| `oidc_probe` | OIDC/OAuth2 provider metadata: JWKS key algorithms, signing key rotation state. |
| `xml_dsig` | XML Digital Signatures (XMLDSig/XAdES): signature algorithm and key extraction. |
| `blockchain` | Blockchain wallet files and smart contract crypto usage. |
| `kerberos_runtime` | Kerberos runtime: detects weak enctypes (DES, RC4) in `/etc/krb5.conf` and active KDC negotiation. |
| `fido2` | FIDO2/WebAuthn authenticator attestation certificates. |
| `enrollment` | Device enrollment certificates (MDM, SCEP, EST). |
| `vpn_runtime` | VPN runtime: active IPsec SA dump via `ip xfrm state`. |

### 5.15 Running Specific Modules

Use `--modules` to run only a named subset:

```bash
# Run only certificate and key modules
triton --modules certificates,keys /etc/ssl

# Run TLS protocol probing only
triton --modules protocol,ftps,ssh_cert --profile standard /

# Run Python and Java code scanners
triton --modules python_ast,java_bytecode /opt/apps
```

---

## 6. Scan Flags and Options

### 6.1 Core Flags

| Flag | Short | Default | Description |
|---|---|---|---|
| `--profile` | `-p` | `standard` | Scan profile: `quick`, `standard`, `comprehensive` |
| `--format` | `-f` | `all` | Output format: `json`, `cdx`, `html`, `xlsx`, `sarif`, `all` |
| `--output` | `-o` | `triton-report.json` | Output file (used with `--format json`) |
| `--output-dir` | `-d` | `.` | Output directory (used with `--format all`) |
| `--modules` | `-m` | (profile default) | Comma-separated list of modules to run |
| `--metrics` | | false | Show per-module scan timing table after completion |
| `--incremental` | | false | Skip unchanged files using hash cache (Pro+ only) |
| `--db` | | (env/default) | PostgreSQL connection URL for result persistence |
| `--policy` | | | Policy file path or builtin name (`nacsa-2030`, `cnsa-2.0`) |
| `--license-key` | | | Licence token (literal string) |
| `--license-file` | | | Path to file containing licence token |
| `--config` | | `~/.triton.yaml` | Config file path |
| `--kubeconfig` | | | Kubeconfig for live Kubernetes cluster scan |

### 6.2 Environment Variables

| Variable | Description |
|---|---|
| `TRITON_LICENSE_KEY` | Licence token (overridden by `--license-key` flag) |
| `TRITON_LICENSE_FILE` | Path to licence token file |
| `TRITON_EXEMPTIONS_FILE` | Path to policy exemptions YAML |
| `TRITON_KEYSTORE_PASSWORDS` | Comma-separated passwords to try on PKCS#12/JKS/JCEKS keystores |

### 6.3 Output Format Details

| Format | File Extension | Description |
|---|---|---|
| `json` | `.json` | Triton native JSON (full finding detail) |
| `cdx` | `.cdx.json` | CycloneDX CBOM JSON |
| `html` | `.html` | Interactive HTML report with policy summary |
| `xlsx` | `.xlsx` | Excel spreadsheet (Jadual 1 & 2 government format, Malay headers) |
| `sarif` | `.sarif` | SARIF for SIEM/security tool ingestion |
| `all` | (all above) | Generates all formats allowed by your licence tier |

Formats available per tier: see Section 17.

---

## 7. Resource Limits

Triton exposes five resource-limiting flags that work on all platforms without systemd, cgroups, or elevated privileges. They apply to foreground scans, detached scans, and agent-supervised scans.

| Flag | Description |
|---|---|
| `--max-memory <MB>` | Soft memory cap in megabytes. Scan pauses new module dispatches when usage exceeds this value. |
| `--max-cpu-percent <N>` | CPU throttle (0–100). Inserts sleep between module batches to stay below the target percentage. `0` disables throttling. |
| `--max-duration <duration>` | Wall-clock time limit (e.g. `30m`, `2h`). Scan is cancelled after this duration. |
| `--stop-at <HH:MM>` | Stop at a specific local time (e.g. `06:00`). Resolved against the actual clock at scan start. |
| `--nice <N>` | Process scheduling priority. Unix: -20 (high) to 20 (low). Windows: 0 (high) to 5 (idle). Default `0`. |

### 7.1 Examples

```bash
# Limit to 512 MB RAM and stop at 06:00 local time
triton --max-memory 512 --stop-at 06:00 /

# Run low-priority with CPU cap
triton --max-cpu-percent 25 --nice 10 --profile standard /

# Run for a maximum of 2 hours
triton --max-duration 2h --profile comprehensive /
```

### 7.2 Behaviour Notes

- `--max-memory` is a **soft** limit. The scan will not terminate, but will pause dispatching new modules until memory drops below the threshold.
- `--stop-at` and `--max-duration` are **hard** cancellations: the scan exits and writes a partial report.
- `--nice` on Windows maps to `ABOVE_NORMAL` (0), `NORMAL` (1), `BELOW_NORMAL` (2–3), `IDLE` (4–5).
- These flags also work in `agent.yaml` via the `resource_limits:` block (see Section 12.5).

---

## 8. Detached Scans

A detached scan forks itself into the background, prints a job ID, and survives SSH disconnect. This is useful for long comprehensive scans on remote servers.

### 8.1 Starting a Detached Scan

```bash
# Start a background scan; prints the job ID
triton --detach --profile comprehensive /

# Suppress all output except the job ID (for scripting)
triton --detach --quiet --profile standard /var/www
```

Output:
```
job-id: a1b2c3d4-e5f6-...
```

### 8.2 Checking Status

```bash
triton --status --job-id a1b2c3d4-e5f6-...
```

### 8.3 Collecting Results

```bash
# Wait for job to finish and collect reports to ./output
triton --collect --job-id a1b2c3d4-e5f6-... --output-dir ./output

# Collect without removing the job state directory
triton --collect --job-id a1b2c3d4-e5f6-... --no-remove --output-dir ./output
```

### 8.4 Cancelling a Job

```bash
# Cancel and wait for the process to exit
triton --cancel --job-id a1b2c3d4-e5f6-... --wait

# Cancel with a timeout
triton --cancel --job-id a1b2c3d4-e5f6-... --wait --cancel-timeout 30s
```

### 8.5 Listing and Cleaning Up Jobs

```bash
# List all background jobs
triton --list-jobs

# Machine-readable output
triton --list-jobs --json-output

# Remove completed/failed job state
triton --cleanup

# Remove all finished jobs (not just failed)
triton --cleanup --all
```

### 8.6 Job State Location

Job state files are stored in `~/.triton/jobs/<job-id>/`. A custom directory can be set with `--work-dir`.

---

## 9. Output Formats and Reports

### 9.1 JSON Report

The native Triton JSON report contains:

- Scan metadata (timestamp, hostname, profile, duration, module metrics)
- All findings with: asset type, algorithm, key size, PQC classification, file path, subject/issuer (for certs), quality warnings
- Policy evaluation results (if `--policy` was used)
- SBOM/CBOM component list

```bash
triton --format json --output report.json /path
```

### 9.2 CycloneDX CBOM

CycloneDX v1.6 JSON format for integration with SBOM tooling and supply chain platforms.

```bash
triton --format cdx --output report.cdx.json /path
```

### 9.3 HTML Report

An interactive single-file HTML report suitable for sharing with compliance officers. When `--policy` is used, the HTML report includes:

- Verdict banner (colour-coded PASS / WARN / FAIL)
- Violations-by-rule summary table
- Threshold violations table
- Per-system policy evaluation table
- Full finding detail tables

```bash
triton --format html --output report.html /path
triton --format html --policy nacsa-2030 --output report.html /path
```

### 9.4 Excel (XLSX)

The XLSX format uses the Malaysian government Jadual 1 & Jadual 2 layout with Malay column headers, as required for NACSA PQC assessment submissions.

```bash
triton --format xlsx --output-dir ./reports /path
```

### 9.5 SARIF

SARIF 2.1 format for direct ingestion into SIEM platforms, GitHub Advanced Security, and CI/CD security gates.

```bash
triton --format sarif --output report.sarif /path
```

### 9.6 Generating All Formats

```bash
triton --format all --output-dir ./reports /path
```

This generates all formats allowed by your licence tier. The `--output-dir` flag is used for `all`; `--output` is used for a single named format.

---

## 10. Incremental Scanning and Caching

Incremental mode (`--incremental`) skips files that have not changed since the previous scan, using a SHA-256 hash cache stored at `~/.triton/cache.db`. This can dramatically reduce scan time on large file systems with few changes.

```bash
# First scan builds the cache
triton --incremental --profile standard /opt/apps

# Subsequent scans are faster
triton --incremental --profile standard /opt/apps
```

### 10.1 Cache Management

```bash
# Show cache statistics
triton cache stats

# Clear the hash cache
triton cache clear
```

Requires Pro+ licence tier.

---

## 11. Policy Compliance Engine

The policy engine evaluates scan findings against a set of compliance rules and emits a `PASS`, `WARN`, or `FAIL` verdict.

### 11.1 Built-in Policies

| Policy Name | Description |
|---|---|
| `nacsa-2030` | Malaysian NACSA 2030 PQC readiness: RSA ≥ 2048, granular MD5/SHA-1/DES/RC4 rules, 60% readiness threshold |
| `cnsa-2.0` | US CNSA 2.0: RSA ≥ 3072, ECDSA ≥ P-384, prefers SHA-384+, 50% SAFE threshold |

```bash
# Check against NACSA-2030
triton --policy nacsa-2030 --format all /path

# Check against CNSA 2.0
triton --policy cnsa-2.0 /path
```

### 11.2 Custom Policy Files

Create a YAML policy file:

```yaml
name: my-policy
rules:
  - id: no-md5
    severity: error
    action: deny
    match:
      algorithm: MD5
    message: "MD5 is prohibited"
    risk_level: high

  - id: rsa-min-size
    severity: warning
    action: warn
    match:
      algorithm: RSA
      key_size_lt: 2048
    message: "RSA key smaller than 2048 bits"
    risk_level: medium

thresholds:
  - name: pqc_readiness
    metric: safe_percent
    min: 60
    message: "At least 60% of findings must be SAFE"
```

```bash
triton --policy ./my-policy.yaml /path
```

### 11.3 Exemptions

Suppress known-accepted findings to avoid alert fatigue:

```yaml
# exemptions.yaml
exemptions:
  - rule_id: no-md5
    path: /opt/legacy-app/config.xml
    reason: "Legacy system under approved migration plan — ticket JIRA-1234"
    expires: 2026-12-31
```

```bash
triton --policy nacsa-2030 \
       --exemptions exemptions.yaml \
       --output-dir ./reports /path
```

Exemptions can also be set via `TRITON_EXEMPTIONS_FILE` environment variable.

### 11.4 Policy CLI Commands

```bash
# List available builtin policies
triton policy list

# Check the latest scan in the database against a policy
triton policy check --policy nacsa-2030 --db postgres://...

# Check a specific scan
triton policy check --policy nacsa-2030 --scan-id <uuid>
```

### 11.5 Exit Codes

| Exit Code | Meaning |
|---|---|
| 0 | Scan succeeded; policy passed (or no policy) |
| 1 | Scan succeeded; policy verdict is FAIL |
| 2 | Scan error or invalid arguments |

---

## 12. Agent Mode

The `triton agent` command (not to be confused with `triton-agent`) runs a persistent scan loop driven by a schedule defined in `agent.yaml`. This is the primary mode for recurring compliance scans on a host that submits results to a Report Server or License Server.

> **Note:** `triton agent` and `triton-agent` are separate binaries with different roles. `triton agent` is a scan-loop daemon for the standalone Triton CLI, configured via `agent.yaml`. `triton-agent` is the Manage Portal enrolled-agent daemon, configured with `manage_url`, mTLS certs, and `host_id`. See Section 15 for the Manage Portal agent.

### 12.1 agent.yaml Reference

```yaml
# Scan settings
profile: standard
report_server: https://triton.example.com
license_key: "eyJ..."           # or use TRITON_LICENSE_KEY env var

# Scheduling — choose one
interval: 24h                    # Option A: repeat every N with ±10% jitter
schedule: "0 2 * * 0"           # Option B: cron (wins over interval when both set)
schedule_jitter: 30s             # Optional fleet staggering (default: 0 = disabled)

# Resource limits (CLI flags win when set)
resource_limits:
  max_memory_mb: 512
  max_cpu_percent: 40
  max_duration: 2h
  nice: 10

# Optional license server binding
license_server: https://license.example.com
license_id: "lic-uuid"
```

### 12.2 Running the Agent

```bash
# Run scan loop in foreground
triton agent --config /etc/triton/agent.yaml

# Validate config without running a scan
triton agent --check-config --config /etc/triton/agent.yaml

# Override interval from CLI (wins over yaml)
triton agent --interval 12h

# Override report server
triton agent --report-server https://triton-dev.example.com
```

### 12.3 Scheduling

#### Interval Mode

The agent runs every N duration (e.g. `24h`) with ±10% jitter to stagger fleet scans.

```yaml
interval: 24h
```

#### Cron Mode

The agent fires at a specific wall-clock time using a standard 5-field cron expression, evaluated in the host's local timezone.

```yaml
schedule: "0 2 * * 0"    # Every Sunday at 02:00 local time
schedule_jitter: 30s     # Stagger up to 30 seconds (fleet use)
```

Common cron expressions:

| Expression | Meaning |
|---|---|
| `0 2 * * *` | Every day at 02:00 |
| `0 2 * * 0` | Every Sunday at 02:00 |
| `0 6 1 * *` | First day of every month at 06:00 |
| `0 3 * * 1-5` | Monday–Friday at 03:00 |

**Scheduling precedence (highest wins):**
1. Server-pushed schedule override (from license server)
2. `schedule:` in `agent.yaml`
3. `interval:` in `agent.yaml`
4. `--interval` CLI flag
5. One-shot (no repeat)

Notes:
- Invalid cron expressions fail fast at startup. `--check-config` surfaces the error.
- No catch-up: if the host was off at the scheduled time, the next fire is the following scheduled time.
- Long-running scans that overrun the next fire do not queue up; the scan finishes, then the next future fire is computed.

### 12.4 Server-Pushed Schedule Override

When an agent is bound to a License Server, the server can push a `schedule` and `scheduleJitterSeconds` override via the `/validate` heartbeat. The agent adopts the override at the next iteration.

```bash
# Set override via admin API
curl -X PATCH https://license.example.com/api/v1/admin/licenses/<id> \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"schedule": "0 2 * * 0", "scheduleJitterSeconds": 60}'

# Clear override (agent reverts to yaml-derived schedule)
curl -X PATCH https://license.example.com/api/v1/admin/licenses/<id> \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"schedule": "", "scheduleJitterSeconds": 0}'
```

### 12.5 Resource Limits in agent.yaml

```yaml
resource_limits:
  max_memory_mb: 512      # MB; same semantics as --max-memory
  max_cpu_percent: 40     # 0-100; same as --max-cpu-percent
  max_duration: 2h        # duration string; same as --max-duration
  nice: 10                # 0-20 on Unix; same as --nice
```

CLI flags win over `agent.yaml` when set. This allows per-invocation overrides without modifying the config file.

### 12.6 Remote Control (pause / cancel / force-run)

When `report_server:` is set in `agent.yaml`, the agent opens a long-poll connection to the Report Server and responds to operator commands:

| Command | Effect |
|---|---|
| `pause` | Suspends scanning for a configured duration (up to 90 days) |
| `cancel` | Cancels the currently running scan |
| `force_run` | Triggers an immediate scan outside the normal schedule |

Administer via:
```bash
# Pause a specific agent
POST /api/v1/admin/agents/{machineID}/pause

# Force an immediate scan
POST /api/v1/admin/agents/{machineID}/commands

# List all agents
GET /api/v1/admin/agents
```

### 12.7 Installing the Agent as an OS Service

The `triton agent` command supports installing itself as a native OS daemon. This is the recommended deployment for production hosts.

#### Install

```bash
# Linux (systemd)
sudo triton agent install --config /etc/triton/agent.yaml

# macOS (launchd)
sudo triton agent install --config /etc/triton/agent.yaml

# Windows (Service Control Manager) — run as Administrator
triton agent install --config C:\triton\agent.yaml

# FreeBSD (rc.d)
sudo triton agent install --config /usr/local/etc/triton/agent.yaml
```

#### Uninstall

```bash
sudo triton agent uninstall      # Linux, macOS, FreeBSD
triton agent uninstall           # Windows (as Administrator)
```

After installation:

- **Linux**: Service is registered as `triton-agent` under systemd, enabled at boot, and started immediately.
- **macOS**: LaunchDaemon plist written to `/Library/LaunchDaemons/com.triton.agent.plist`, loaded with `launchctl`.
- **Windows**: Service registered in SCM as `TritonAgent`, starts automatically.
- **FreeBSD**: rc.d script written to `/etc/rc.d/triton_agent`, enabled in `/etc/rc.conf`.

Requires root (Linux/macOS/FreeBSD) or Administrator (Windows).

### 12.8 Generating agent.yaml from the License Server

An `agent.yaml` pre-populated with the licence token can be downloaded from the License Server admin UI:

1. Log in to the License Server admin UI at `https://license.example.com/ui/`
2. Navigate to the license record
3. Click **Download agent.yaml**

Or via the API:
```bash
curl -H "Authorization: Bearer $TOKEN" \
     -o agent.yaml \
     https://license.example.com/api/v1/admin/licenses/<license-id>/agent-yaml
```

---

## 13. Fleet and Device Scanning

### 13.1 Device Scan (Agentless SSH/NETCONF)

`triton device-scan` is an agentless scanner that SSHs into remote hosts, runs a scan, and returns results — without installing any persistent agent on the target.

Supported targets:
- Unix hosts: Linux, macOS, AIX (via SSH)
- Network routers: Cisco IOS-XE, Juniper Junos (via NETCONF/SSH)

```bash
triton device-scan \
  --inventory /etc/triton/devices.yaml \
  --credentials /etc/triton/credentials.yaml
```

#### Inventory File (devices.yaml)

```yaml
devices:
  - name: web01
    type: unix
    host: 192.168.1.10
    port: 22
    group: webservers

  - name: db01
    type: unix
    host: 192.168.1.20
    group: databases

  - name: core-router
    type: cisco-ios-xe
    host: 10.0.0.1
    group: network
```

#### Credentials File (credentials.yaml)

The credentials file is AES-GCM encrypted. The decryption key must be in `TRITON_SCANNER_CRED_KEY` (32 bytes = 64 hex chars).

```yaml
credentials:
  - name: ssh-admin
    type: ssh
    username: triton
    private_key_path: /etc/triton/id_ed25519
    groups: [webservers, databases]

  - name: net-admin
    type: netconf
    username: triton
    password_ref: vault:secret/triton/netconf
    groups: [network]
```

#### Device Scan Options

| Flag | Default | Description |
|---|---|---|
| `--inventory` | `/etc/triton/devices.yaml` | Path to devices inventory |
| `--credentials` | `/etc/triton/credentials.yaml` | Path to encrypted credentials |
| `--group` | | Scan only devices in this group |
| `--device` | | Scan only this named device |
| `--concurrency` | 20 | Max concurrent device scans |
| `--device-timeout` | 5m | Max time per device |
| `--dry-run` | false | Validate inventory + credentials without scanning |
| `--interval` | | Continuous mode: repeat every interval |
| `--report-server` | | Report server URL for result submission |
| `--known-hosts` | | Path to SSH known_hosts file |
| `--insecure-host-key` | false | Accept any host key (lab/test only) |

#### SSH Known Hosts

Always use `--known-hosts` in production to prevent MITM attacks:

```bash
triton device-scan \
  --inventory devices.yaml \
  --credentials credentials.yaml \
  --known-hosts /etc/triton/known_hosts
```

The `--insecure-host-key` flag is provided for lab environments only and must not be used in production.

### 13.2 Network Alias

`triton network-scan` is an alias for `triton device-scan` (deprecated, will be removed in a future release).

---

## 14. Report Server

The Triton Report Server is an optional REST API backend with an embedded web UI. It provides centralised storage of scan results, historical diff/trend analysis, team access, and analytics dashboards.

### 14.1 Starting the Server

```bash
triton server --listen :8080 --db postgres://triton:triton@localhost:5434/triton
```

With TLS:
```bash
triton server \
  --listen :8443 \
  --tls-cert /etc/triton/server.crt \
  --tls-key /etc/triton/server.key \
  --db postgres://...
```

### 14.2 Environment Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `REPORT_SERVER_JWT_SIGNING_KEY` | Yes (multi-tenant) | — | Hex-encoded Ed25519 signing key (min 32 bytes) |
| `REPORT_SERVER_DATA_ENCRYPTION_KEY` | Recommended | — | Hex AES-256-GCM key for result_json at rest |
| `REPORT_SERVER_SERVICE_KEY` | No | — | Pre-shared key for agent result submission |
| `REPORT_SERVER_TENANT_PUBKEY` | No | — | Ed25519 public key for tenant-signed licence tokens |
| `REPORT_SERVER_LOGIN_RATE_LIMIT_MAX_ATTEMPTS` | No | 5 | Max login attempts before lockout |
| `REPORT_SERVER_LOGIN_RATE_LIMIT_WINDOW` | No | 15m | Rate limit window duration |
| `REPORT_SERVER_LOGIN_RATE_LIMIT_LOCKOUT` | No | 30m | Lockout duration after threshold |
| `REPORT_SERVER_SESSION_CACHE_SIZE` | No | 10000 | JWT session cache size (0 disables) |
| `REPORT_SERVER_SESSION_CACHE_TTL` | No | 60s | JWT session cache TTL [5s–5m] |
| `REPORT_SERVER_RESEND_API_KEY` | No | — | Resend.com API key for invite emails |
| `REPORT_SERVER_RESEND_FROM_EMAIL` | No | — | Sender email for invite flow |
| `REPORT_SERVER_RESEND_FROM_NAME` | No | — | Sender display name |
| `REPORT_SERVER_INVITE_URL` | No | — | Base URL for invite login links |

### 14.3 Web UI

The embedded web UI is available at `http://localhost:8080/` and includes:

- **Dashboard** — aggregate stat cards, machines table, Chart.js charts
- **Scans** — list and detail view of all scans, per-machine history
- **Analytics** — crypto inventory, expiring certificates (90-day window), migration priority
- **Diff** — compare two scans side-by-side
- **Trend** — PQC migration trend chart over time
- **Machines** — per-machine scan history and cert expiry warnings

### 14.4 REST API Summary

| Endpoint | Description |
|---|---|
| `POST /api/v1/scans` | Submit a scan result |
| `GET /api/v1/scans` | List all scans |
| `GET /api/v1/scans/{id}` | Get scan detail |
| `GET /api/v1/diff?a=<id>&b=<id>` | Compare two scans |
| `GET /api/v1/trend` | PQC trend data |
| `GET /api/v1/history` | Scan history list |
| `GET /api/v1/inventory` | Crypto inventory analytics |
| `GET /api/v1/certificates/expiring` | Expiring certificate list |
| `GET /api/v1/priority` | Migration priority (top-20) |

Analytics endpoints require the results backfill to complete on first boot. A banner appears in the UI while backfill is in progress.

### 14.5 CLI Commands That Use the Server

```bash
# List past scans
triton history --db postgres://...

# Diff two scans
triton diff <scan-id-1> <scan-id-2> --db postgres://...

# Show trend
triton trend --db postgres://...
```

---

## 15. Manage Portal

The Manage Portal is an optional on-prem admin server for operators who need to orchestrate network-based scans without installing persistent agents on every host. It is separate from the Report Server and License Server.

### 15.1 Architecture

```
Manage Portal (:8082)
    ├── Admin web UI (operator browser)
    ├── mTLS gateway (:8443) ← triton-agent daemons connect here
    ├── Credential vault (HashiCorp Vault)
    ├── triton-sshagent workers (SSH scan jobs)
    └── triton-portscan workers (port survey jobs)
```

The Manage Portal can optionally relay scan results to a Report Server via the `TRITON_MANAGE_REPORT_SERVER` integration.

### 15.2 Installation (One-Shot Installer)

```bash
curl -LO https://github.com/primatekuntech/triton/releases/latest/download/install-manage.sh
chmod +x install-manage.sh

./install-manage.sh \
  --license-server-pubkey <64-hex-chars> \
  --manage-url https://manage.example.com:8082 \
  --out /opt/triton
```

Requirements: `podman`, `podman-compose` (or `podman compose`), `openssl`, `jq`, `curl`.

The installer:
1. Generates Ed25519 JWT signing key, bcrypt admin password, and worker shared secret
2. Creates a Vault container for credential storage
3. Writes `compose.yaml` and `.env` to `--out`
4. Starts PostgreSQL, Vault, and Manage Server containers
5. Prints the admin login URL and generated secrets

### 15.3 Environment Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `TRITON_MANAGE_DB_URL` | Yes | — | PostgreSQL connection URL |
| `TRITON_MANAGE_JWT_SIGNING_KEY` | Yes | — | Hex Ed25519 signing key (min 32 bytes) |
| `TRITON_MANAGE_LICENSE_SERVER_PUBKEY` | Yes | — | Hex Ed25519 public key from License Server |
| `TRITON_MANAGE_LISTEN` | No | `:8082` | Admin UI + API listen address |
| `TRITON_MANAGE_GATEWAY_LISTEN` | No | `:8443` | mTLS gateway listen address |
| `TRITON_MANAGE_GATEWAY_HOSTNAME` | No | `localhost` | Hostname/IP for agent mTLS (in TLS cert) |
| `TRITON_MANAGE_GATEWAY_URL` | No | (derived) | Public URL for the mTLS gateway |
| `TRITON_MANAGE_PARALLELISM` | No | `10` | Max concurrent scan jobs (1–50) |
| `TRITON_MANAGE_WORKER_KEY` | No | — | Shared secret for `triton-sshagent`/`triton-portscan` workers |
| `TRITON_MANAGE_BIN_DIR` | No | `/bins` | Directory of worker binaries served via install endpoint |
| `TRITON_MANAGE_REPORT_SERVER` | No | — | Report Server URL for scan result relay |
| `TRITON_MANAGE_REPORT_SERVICE_KEY` | No | — | Service key for Report Server submission |
| `TRITON_MANAGE_HOST_IP` | No | (auto) | Host machine LAN IP (required in containers) |
| `TRITON_MANAGE_SESSION_TTL` | No | `24h` | JWT session TTL |

### 15.4 Portal Features

**Host Management**
- Add hosts manually, via CSV import, or via network discovery (CIDR scan)
- Assign tags and groups for organised scanning
- View per-host scan history, certificate expiry warnings, and last-seen timestamps

**Scan Job Orchestration**
- **SSH scan jobs** (`triton-sshagent`): dispatched when a host needs agentless scanning; worker SSHs in, uploads a `triton` binary, runs the scan, and returns results
- **Port survey jobs** (`triton-portscan`): CIDR discovery and port fingerprinting via `fingerprintx`; results appear as host records for follow-up scanning

**Credential Vault**
- Credentials stored in Vault (never in the database)
- Assign credentials to host groups; workers retrieve them at job dispatch time

**Enrolled Agents**
- Hosts with `triton-agent` installed show as enrolled agents
- Certificate expiry warnings displayed in the agents table
- Manage Portal signs mTLS certificates for each enrolled agent

### 15.5 Installing the Enrolled Agent (triton-agent)

The enrolled agent (`triton-agent`) is a lightweight daemon that connects to the Manage Portal's mTLS gateway, heartbeats periodically, and executes scans on demand.

#### Downloading the Agent Binary

```bash
# Download from the Manage Portal install endpoint
curl -LO "https://manage.example.com/api/v1/install/<token>/agent/linux/amd64"
chmod +x triton-agent
sudo install -m 755 triton-agent /usr/local/bin/triton-agent
```

#### Agent Configuration (agent.yaml)

```yaml
manage_url: https://manage.example.com:8443
cert_path: /etc/triton-agent/agent.crt
key_path: /etc/triton-agent/agent.key
ca_path: /etc/triton-agent/ca.crt
host_id: "host-uuid-from-portal"
scan_profile: standard
```

#### Installing as OS Service

```bash
# Install with bundle (cert + key + ca + config from Portal)
sudo triton-agent install \
  --config /etc/triton-agent/agent.yaml \
  --bundle /tmp/agent-bundle.tar.gz

# Install without bundle (config already in place)
sudo triton-agent install --config /etc/triton-agent/agent.yaml

# Uninstall
sudo triton-agent uninstall
```

Supported platforms: Linux (systemd), macOS (launchd), Windows (SCM), FreeBSD (rc.d). Requires root (Linux/macOS/FreeBSD) or Administrator (Windows).

### 15.6 Worker Binaries

Workers (`triton-sshagent`, `triton-portscan`) authenticate to the Manage Portal via `TRITON_WORKER_KEY` (a shared secret, never passed as a flag to avoid process listing exposure).

```bash
# Run an SSH scan job (dispatched by Manage Portal)
export TRITON_WORKER_KEY=<shared-secret>
triton-sshagent --manage-url https://manage.example.com:8082 --job-id <uuid>

# Run a port survey job
export TRITON_WORKER_KEY=<shared-secret>
triton-portscan --manage-url https://manage.example.com:8082 --job-id <uuid>
```

Workers are normally dispatched by the Manage Portal automatically; manual invocation is for troubleshooting.

---

## 16. License Server

The License Server is a standalone binary for centralised, seat-counted license management. It is separate from the Report Server and Manage Portal.

### 16.1 Starting the License Server

```bash
triton-license-server
```

Configuration is entirely via environment variables:

| Variable | Required | Description |
|---|---|---|
| `TRITON_LICENSE_SERVER_DB_URL` | Yes | PostgreSQL connection URL |
| `TRITON_LICENSE_SERVER_SIGNING_KEY` | Yes | Hex Ed25519 key pair (64 bytes = 128 hex chars) |
| `TRITON_LICENSE_SERVER_ADMIN_EMAIL` | Yes | Bootstrap superadmin email |
| `TRITON_LICENSE_SERVER_ADMIN_PASSWORD` | Yes | Bootstrap superadmin password |
| `TRITON_LICENSE_SERVER_LISTEN` | No | Listen address (default `:8081`) |
| `TRITON_LICENSE_SERVER_REPORT_URL` | No | Report Server URL for agent.yaml generation |

### 16.2 Admin Web UI

The admin UI is at `http://localhost:8081/ui/` and provides:

- Dashboard (org count, licence count, active activations, recent audit events)
- Organisation management (create, edit, delete)
- Licence management (create per-org licences, set seat count, expiry, tier)
- Activation tracking (view active machines, revoke individual activations)
- Audit log (all admin and agent actions)
- Superadmin management (invite/list/delete platform admins)

### 16.3 Online Activation Workflow

Agents activate online by calling the License Server, which validates and increments the seat counter. Activation is atomic (serialisable transaction) — concurrent activations never exceed the seat limit.

```bash
# Activate this machine
triton license activate \
  --license-server https://license.example.com \
  --license-id lic-uuid-from-admin

# Deactivate and release seat
triton license deactivate
```

After activation:
- Token and cache metadata are saved to `~/.triton/license.key` and `~/.triton/license.meta`
- Subsequent CLI invocations use the cached token
- If the License Server is unreachable, the 7-day offline grace period applies

### 16.4 Offline Grace Period

Triton caches licence validation results for 7 days. If the License Server is unreachable:

- Tier from cache is used for up to 7 days
- After 7 days without a successful validation, the agent degrades to Free tier
- The cache is stored at `~/.triton/license.meta`

### 16.5 License Commands

```bash
# Show current licence details and tier
triton license show

# Verify a specific token
triton license verify <token>

# Activate online
triton license activate --license-server <url> --license-id <id>

# Deactivate and release seat
triton license deactivate
```

---

## 17. License Tiers and Feature Gating

Triton uses three licence tiers: **Free**, **Pro**, and **Enterprise**. Tier is enforced at scan time with graceful degradation — invalid or missing licences fall back to Free rather than blocking.

### 17.1 Feature Matrix

| Feature | Free | Pro | Enterprise |
|---|---|---|---|
| Profile: `quick` | Yes | Yes | Yes |
| Profile: `standard` | — | Yes | Yes |
| Profile: `comprehensive` | — | Yes | Yes |
| Scanner modules | 3 (certs, keys, packages) | All 56 | All 56 |
| Format: JSON | Yes | Yes | Yes |
| Format: CycloneDX, HTML, XLSX | — | Yes | Yes |
| Format: SARIF | — | — | Yes |
| Report Server mode | — | — | Yes |
| Agent mode | — | — | Yes |
| Policy: builtin (`nacsa-2030`, `cnsa-2.0`) | — | Yes | Yes |
| Policy: custom YAML | — | — | Yes |
| Metrics (`--metrics`) | — | Yes | Yes |
| Incremental scanning | — | Yes | Yes |
| DB persistence | — | Yes | Yes |
| Diff / Trend / History | — | Yes | Yes |
| Policy exemptions | — | — | Yes |

### 17.2 Machine Binding

By default, licence tokens are bound to the machine that activated them. Machine identity is computed as:

```
SHA-3-256(hostname | GOOS | GOARCH)
```

This is a deterministic 64-character hex string that does not require elevated privileges.

A token used on a different machine degrades gracefully to Free tier without error. To issue an unbound token (for shared environments), use `triton keygen --no-bind`.

### 17.3 Token Resolution Order

```
--license-key flag
  → TRITON_LICENSE_KEY env var
    → --license-file flag path
      → TRITON_LICENSE_FILE env var
        → ~/.triton/license.key (default file)
```

### 17.4 Checking Your Licence

```bash
triton license show
```

Output example:
```
Tier:             Pro
Expiry:           2027-01-01 00:00:00 UTC
Machine:          bound (this machine)
Seats:            3/10 used
License Server:   https://license.example.com
```

---

## 18. PQC Algorithm Classification

Every cryptographic asset Triton discovers is assigned one of four PQC readiness classifications:

| Classification | Meaning |
|---|---|
| `SAFE` | Quantum-safe algorithm (ML-KEM, ML-DSA, SLH-DSA, XMSS, etc.) |
| `TRANSITIONAL` | Acceptable for near-term use but requires migration planning (RSA-2048+, ECDSA P-256+, AES-128+) |
| `DEPRECATED` | Should be migrated but not yet prohibited (SHA-1 signatures, RSA-1024) |
| `UNSAFE` | Prohibited algorithms (MD5, DES, RC4, RSA < 1024, export-grade) |

### 18.1 Migration Priority Score

Each finding has a numeric migration priority (0.0–1.0) used to rank findings in the HTML report and XLSX:

- `UNSAFE` findings: 1.0
- `DEPRECATED` findings: 0.8
- `TRANSITIONAL` findings: 0.5–0.6 (lower if key quality is good)
- `SAFE` findings: 0.0
- Dependency reachability reduces score: `unreachable` dependencies score 0.50 × base

### 18.2 Key Quality Warnings

For RSA and EC keys, Triton also runs quality analysis and attaches warnings:

| Warning | CVE | Description |
|---|---|---|
| `ROCA` | CVE-2017-15361 | Infineon TPM weak RSA key generation |
| `DEBIAN_PRNG` | CVE-2008-0166 | Debian OpenSSL weak RNG key |
| `SMALL_PRIME` | — | RSA modulus has small prime factor |
| `SIZE_MISMATCH` | — | Declared key size does not match actual modulus |

### 18.3 TLS Hybrid PQC Groups

Triton classifies TLS named groups including hybrid PQC combinations:

| Group | Classification |
|---|---|
| `X25519MLKEM768` | SAFE (hybrid PQC) |
| `SecP256r1MLKEM768` | SAFE (hybrid PQC) |
| `X25519Kyber768Draft00` | TRANSITIONAL (draft Kyber hybrid) |
| `x25519`, `secp256r1`, `secp384r1` | TRANSITIONAL |
| `secp192r1`, `secp160r1` | DEPRECATED |

---

## 19. Utilities and Diagnostics

### 19.1 Doctor

Checks system readiness for scanning:

```bash
triton doctor
```

Checks include:
- Required tools: `osslsigncode`, `jarsigner`, `pkcs11-tool`, `fingerprintx`
- Optional tools: `libpcap`, eBPF kernel version, BTF availability
- Database connectivity
- Licence status and tier
- Profile availability for the current licence

### 19.2 History

Lists past scans stored in the database:

```bash
triton history
triton history --db postgres://...
triton history --limit 20
```

### 19.3 Diff

Compares two scans:

```bash
triton diff <scan-id-1> <scan-id-2>
triton diff <scan-id-1> <scan-id-2> --format json
```

Shows new findings (appeared in scan 2), removed findings (disappeared), and changed findings (algorithm or classification changed).

### 19.4 Trend

Shows PQC migration trend over recent scans:

```bash
triton trend
triton trend --limit 10 --format json
```

### 19.5 Cache

Manages the file hash cache used for incremental scanning:

```bash
# Show cache statistics
triton cache stats

# Clear all cached hashes
triton cache clear
```

### 19.6 Policy Check

Evaluates a policy against a stored scan without re-running the scan:

```bash
# Check latest scan
triton policy check --policy nacsa-2030

# Check specific scan
triton policy check --policy nacsa-2030 --scan-id <uuid>

# List available policies
triton policy list
```

---

## 20. Configuration Reference

### 20.1 Triton Config File (~/.triton.yaml)

Triton reads a YAML config file from `~/.triton.yaml` (override with `--config`). All CLI flags can be set here as defaults.

```yaml
# ~/.triton.yaml
profile: standard
format: all
output-dir: ~/triton-reports
db: postgres://triton:triton@localhost:5434/triton?sslmode=disable
license-key: "eyJ..."
```

Values set on the CLI override values from the config file.

### 20.2 agent.yaml (Standalone Agent Mode)

Full reference with all supported keys:

```yaml
# Scan settings
profile: standard                         # quick | standard | comprehensive
report_server: https://triton.example.com # URL to submit results (optional)
license_key: "eyJ..."                     # can use TRITON_LICENSE_KEY env var instead
license_server: https://license.example.com
license_id: "lic-uuid"

# Scheduling
interval: 24h                             # Option A
schedule: "0 2 * * 0"                    # Option B (wins over interval)
schedule_jitter: 30s                      # Optional fleet staggering (default: disabled)

# Resource limits
resource_limits:
  max_memory_mb: 512
  max_cpu_percent: 40
  max_duration: 2h
  nice: 10
```

### 20.3 triton-agent Config (Manage Portal Agent)

```yaml
manage_url: https://manage.example.com:8443   # Manage Portal mTLS gateway
cert_path: /etc/triton-agent/agent.crt
key_path: /etc/triton-agent/agent.key
ca_path: /etc/triton-agent/ca.crt
host_id: "host-uuid-assigned-by-portal"
scan_profile: standard
```

### 20.4 PostgreSQL Default Connection

The default database URL (when `--db` is not set) is:

```
postgres://triton:triton@localhost:5434/triton?sslmode=disable
```

Port 5434 is used to avoid conflicts with other PostgreSQL instances. The compose stack starts PostgreSQL on port 5434.

### 20.5 File and Directory Locations

| Path | Purpose |
|---|---|
| `~/.triton.yaml` | Default CLI config file |
| `~/.triton/license.key` | Default licence token file |
| `~/.triton/license.meta` | Licence cache (7-day offline grace) |
| `~/.triton/cache.db` | File hash cache for incremental scanning |
| `~/.triton/jobs/` | Detached scan job state directories |
| `/etc/triton/devices.yaml` | Default device inventory for device-scan |
| `/etc/triton/credentials.yaml` | Default credentials for device-scan |

---

## 21. Glossary

| Term | Definition |
|---|---|
| **CBOM** | Cryptography Bill of Materials — an inventory of all cryptographic algorithms, keys, and certificates used in a system. |
| **SBOM** | Software Bill of Materials — a complete inventory of software components and dependencies. |
| **PQC** | Post-Quantum Cryptography — cryptographic algorithms designed to be secure against both classical and quantum computers. |
| **NACSA** | National Cyber Security Agency (Malaysia) — the national authority for cybersecurity policy and PQC assessment frameworks. |
| **NACSA-2030** | Malaysian national PQC readiness policy targeting 2030 compliance. |
| **CNSA 2.0** | US Commercial National Security Algorithm Suite 2.0 — the NSA's updated cryptographic standards for classified systems. |
| **ML-KEM** | Module-Lattice Key Encapsulation Mechanism (FIPS 203, formerly Kyber) — a SAFE PQC key exchange algorithm. |
| **ML-DSA** | Module-Lattice Digital Signature Algorithm (FIPS 204, formerly Dilithium) — a SAFE PQC signature algorithm. |
| **SLH-DSA** | Stateless Hash-Based Digital Signature (FIPS 205, formerly SPHINCS+) — a SAFE PQC signature algorithm. |
| **SAFE** | Triton classification: algorithm is quantum-safe. |
| **TRANSITIONAL** | Triton classification: algorithm is acceptable now but requires a migration plan. |
| **DEPRECATED** | Triton classification: algorithm is no longer recommended and should be migrated. |
| **UNSAFE** | Triton classification: algorithm is cryptographically broken and must be replaced. |
| **ROCA** | Return of Coppersmith's Attack (CVE-2017-15361) — weak RSA key generation in Infineon TPM chips. |
| **JA3/JA4** | TLS fingerprinting methods that generate a hash of the ClientHello parameters, used to identify TLS client implementations. |
| **eBPF** | Extended Berkeley Packet Filter — a Linux kernel technology used by the `ebpf_trace` module to observe cryptographic operations at runtime without modifying the target process. |
| **mTLS** | Mutual TLS — both client and server authenticate with certificates, used by the Manage Portal gateway for enrolled agent communication. |
| **Ed25519** | An elliptic-curve signature scheme used for Triton's licence tokens and JWT signing keys. |
| **Machine fingerprint** | SHA-3-256 hash of `hostname|GOOS|GOARCH` used to bind licence tokens to a specific machine. |
| **seat** | A single machine activation counted against a licence's seat limit. Deactivating releases a seat. |
| **Enrolled agent** | A host with `triton-agent` installed and registered with the Manage Portal via mTLS certificate. |
| **Agentless scan** | A scan performed by SSHing into a remote host and running a scan binary, without leaving any persistent daemon on the target (`triton device-scan`, `triton-sshagent`). |
| **Detached scan** | A scan running as a background process, identified by a job ID, survives SSH disconnect. |
| **CycloneDX** | An open SBOM/CBOM standard supported by OWASP. Triton outputs CycloneDX v1.6 JSON. |
| **SARIF** | Static Analysis Results Interchange Format — a standard for security tool output, ingested by GitHub Advanced Security and many SIEM platforms. |
| **Jadual 1 & 2** | The mandatory Excel report tables in the Malaysian government PQC submission format, with Malay column headers. |
| **Vault** | HashiCorp Vault — used by the Manage Portal for credential storage. |
| **OID** | Object Identifier — a hierarchical identifier used in ASN.1/X.509 to identify cryptographic algorithms and attributes. |
| **BTF** | BPF Type Format — Linux kernel debug information required by the eBPF trace module (available in kernels 5.8+). |
