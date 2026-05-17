# Triton Scanning Agents: Enrolled Agent vs SSH Agent

This guide explains the two agentless and agent-based scanning approaches in Triton, the scanner modules each approach activates, and how operating system and privilege level affect coverage. Use it to choose the right deployment model and understand what a given scan will and will not find.

---

## Table of Contents

1. [Overview](#overview)
2. [triton-agent — Enrolled Agent](#triton-agent--enrolled-agent)
3. [triton-sshagent — SSH Agent (Agentless)](#triton-sshagent--ssh-agent-agentless)
4. [Side-by-Side Comparison](#side-by-side-comparison)
5. [Privilege Level Impact](#privilege-level-impact)
6. [OS Coverage Matrix](#os-coverage-matrix)
7. [Scanner Module Reference](#scanner-module-reference)

---

## Overview

Triton offers two ways to scan a remote host from the Manage Server:

| | triton-agent | triton-sshagent |
|---|---|---|
| **Installed on target?** | Yes — long-running daemon | No — binary uploaded per scan, removed after |
| **Auth to Manage Server** | mTLS client certificate (per-host) | X-Worker-Key (shared secret) |
| **How it scans** | Runs the full scanner engine locally on the host | SSHes into the host; uploads embedded `triton` binary; runs `triton scan` natively on the remote |
| **Windows targets** | Supported | Not supported — linux/amd64 and linux/arm64 binaries only |
| **macOS targets** | Supported | Not supported — no embedded macOS binary |
| **Real-time commands** | Admin can push on-demand scans | Job-dispatch only (Dispatcher spawns it per job) |
| **Live runtime modules** | Yes (processes, eBPF, live pcap) | Yes — when SSH user has root/CAP_BPF/CAP_NET_RAW |
| **Prerequisite on target** | triton-agent binary + agent.yaml config | SSH server + valid credentials stored in Manage Server |

**When to use triton-agent:** targets that need continuous scanning, real-time command dispatch, runtime process inspection, or Windows support.

**When to use triton-sshagent:** targets where installing a persistent daemon is not permitted, or for ad-hoc agentless scans of Linux hosts (amd64 or arm64) already reachable via SSH.

---

## triton-agent — Enrolled Agent

### How it works

1. The host operator installs the `triton-agent` binary and configures `agent.yaml` with the Manage Server URL, mTLS certificate paths, and default scan profile.
2. On startup the agent contacts the Manage Server's mTLS gateway (`:8443`) and registers itself.
3. It runs a heartbeat loop (default every 30 s) and a command-poll loop in parallel.
4. When the Manage Server pushes a scan command, the agent invokes `pkg/scanner.Engine` directly on the local host — no subprocess, no SSH.
5. The completed `ScanResult` is posted to `POST /agents/scans` on the mTLS gateway and relayed asynchronously to the Report Server via the scanresults outbox.

### Key characteristics

- **Full local access** — the scanner reads every file the process user can open. As root, this is essentially the entire filesystem.
- **Runtime modules** — process enumeration, eBPF crypto tracing, live pcap TLS observation, TPM attestation, and UEFI Secure Boot key inventory all run locally and have access to the host's kernel and hardware.
- **Persistent connection** — the agent maintains a persistent authenticated channel; the Manage Server can trigger an immediate on-demand scan at any time.
- **mTLS identity** — each enrolled host gets a unique certificate signed by the Manage Server's CA. Revocation is per-certificate-serial.

### Scan flow

```
Manage Server           triton-agent (on target)
     │                        │
     │──── push command ──────▶│
     │                        │── scanner.Engine.Scan()
     │                        │     (local filesystem + kernel)
     │                        │
     │◀─── POST /agents/scans ─│
     │                        │
  scanresults outbox
     │
  Report Server
```

---

## triton-sshagent — SSH Agent (Agentless)

### How it works

1. The Manage Server's `Dispatcher` polls for queued `JobTypeSSH` scan jobs and spawns `triton-sshagent` as a subprocess for each one.
2. `triton-sshagent` calls `POST /api/v1/worker/jobs/{id}/claim` to claim the job, then `GET /api/v1/worker/hosts/{id}` to resolve the target IP and SSH port.
3. It fetches SSH credentials from `GET /api/v1/worker/credentials/{id}` (password or private key, stored in Manage Server).
4. It opens an SSH connection to the target host (optionally through a bastion ProxyJump host if `BastionHostIP` is set on the job).
5. It runs `uname -m` to detect the remote architecture (x86_64 or aarch64/arm64).
6. It uploads the matching embedded `triton` binary (linux/amd64 or linux/arm64) to `/tmp/triton-<job-uuid>` via SFTP and makes it executable.
7. It runs `triton scan --profile <profile> --format json --output -` on the remote host. The binary runs **natively** on the remote host under the SSH user's effective UID — all scanner modules available for that OS and privilege level are active.
8. It captures stdout, parses the JSON `ScanResult`, and removes the temporary binary via SFTP.
9. The result is submitted to `POST /api/v1/worker/jobs/{id}/submit`, which enqueues it in the scanresults outbox for relay to the Report Server.

### Key characteristics

- **No permanent binary on target** — the triton binary is uploaded per-scan via SFTP and removed on completion. Only an SSH server and valid credentials are required.
- **Full local scanner execution** — the scanner runs natively on the remote host. All modules supported on the target OS run, including runtime modules (process inspection, eBPF crypto tracing, live pcap) when the SSH user has the required privileges.
- **Linux targets only** — embedded binaries are linux/amd64 and linux/arm64. macOS and Windows targets are not supported.
- **Bastion support** — if the target host has `connection_type = ssh_bastion`, the SSH dial routes through a jump host (ProxyJump) transparently.
- **Coverage depends on privilege** — the SSH credentials determine which files and kernel resources are accessible. Root gives near-complete coverage including runtime modules; an unprivileged user misses root-owned files and cannot attach eBPF probes.

### Scan flow

```
Manage Server                     triton-sshagent           Target host (SSH)
     │                                  │                        │
  Dispatcher                            │                        │
  spawns subprocess ────────────────────▶│                        │
     │                                  │── claim job            │
     │                                  │── get host/credentials │
     │                                  │── SSH dial ────────────▶│
     │                                  │── uname -m             │── detects arch
     │                                  │── SFTP upload triton ──▶│── /tmp/triton-<uuid>
     │                                  │── triton scan (JSON) ───▶│── scanner.Engine.Scan()
     │                                  │◀─────────────────────── │── stdout JSON result
     │                                  │── SFTP rm /tmp/triton   │
     │◀── POST /worker/jobs/{id}/submit ─│                        │
     │                                  │                        │
  scanresults outbox
     │
  Report Server
```

---

## Side-by-Side Comparison

| Capability | triton-agent | triton-sshagent |
|---|:---:|:---:|
| No permanent binary on target | | ✓ |
| Persistent on-demand scan trigger | ✓ | |
| Filesystem scan (certs, keys, configs) | ✓ | ✓ (Linux) |
| Live process inspection | ✓ | ✓ (Linux, root SSH) |
| eBPF runtime crypto tracing | ✓ (Linux, root) | ✓ (Linux, root SSH) |
| Live TLS pcap observation | ✓ (Linux, root) | ✓ (Linux, root SSH) |
| TPM attestation | ✓ (Linux) | ✓ (Linux, root SSH) |
| UEFI Secure Boot key inventory | ✓ (Linux) | ✓ (Linux, root SSH) |
| OS certificate store (certstore) | ✓ | ✓ (Linux) |
| Package manager scan | ✓ | ✓ (Linux) |
| Windows target support | ✓ | |
| macOS target support | ✓ | |
| Linux target support | ✓ | ✓ |
| SSH bastion / jump host routing | | ✓ |
| /etc/shadow password hash scan | ✓ (root) | ✓ (root SSH) |
| Root-only key/cert directories | ✓ (root) | ✓ (root SSH) |

---

## Privilege Level Impact

### triton-agent

The privilege level is the OS user that runs the `triton-agent` daemon.

#### Linux

| Module | Non-root | Root |
|---|---|---|
| **password_hash** | Only `/etc/passwd` (hashes are `x` — unusable) | Full `/etc/shadow` + `/etc/gshadow` → hash algorithm per user |
| **key** | Own `~/.ssh/id_*` and world-readable keys | All SSH host private keys (`/etc/ssh/ssh_host_*_key`), `/root/.ssh/`, system service keys |
| **certificate** | `/etc/ssl/certs/`, world-readable PEM files | Also `/etc/ssl/private/`, service-owned cert dirs |
| **auth_material** | Own `authorized_keys`, `known_hosts` | All users' `authorized_keys` + `/root/.ssh/` |
| **certstore** | System CA bundle (world-readable) | Same — no difference |
| **kernel** | `/lib/modules/*/` (mostly world-readable) | Guaranteed access to compressed `.ko.gz`/`.ko.xz` modules |
| **ebpf_trace** | Skipped — emits skipped-finding | Full eBPF uprobes on libcrypto/gnutls/nss + kprobes on kernel crypto API |
| **tls_observer** (live) | Skipped — no `CAP_NET_RAW` | Live `AF_PACKET` capture on any interface |
| **process** | Visible processes only (own + world-accessible `/proc`) | All `/proc/<pid>/` entries including those of other users |
| **tpm** | `/sys/class/tpm` sysfs — usually world-readable | Same; measured-boot log may require root on some kernels |
| **uefi** | `/sys/firmware/efi/efivars/` — usually root-only | Full Secure Boot key inventory (PK/KEK/db/dbx) |
| **vpn_config** | World-readable configs only | Pre-shared keys in root-owned WireGuard/IPsec configs |
| **secrets_mgr** | Vault agent config if world-readable | Vault agent tokens (`0600`), AWS system credential files |
| All other filesystem modules | World-readable files | All files |

#### macOS

| Module | Non-root | Root |
|---|---|---|
| **certstore** | System keychain (world-readable) | Same |
| **key** | `~/Library/`, `~/.ssh/` | Also `/etc/ssh/ssh_host_*_key`, `/var/root/.ssh/` |
| **certificate** | `/etc/ssl/`, `/usr/share/` | Also `/private/etc/` dirs with `0600` permissions |
| **password_hash** | Not applicable — macOS uses Directory Services, no `/etc/shadow` | Not applicable |
| **process** | Visible processes | All processes via `ps aux` / `/proc` equivalent |
| **ebpf_trace** | Skipped — macOS is not Linux | Skipped — macOS is not Linux |
| **tpm**, **uefi** | Skipped — Linux-only | Skipped — Linux-only |
| All other filesystem modules | World-readable files | All files |

#### Windows

| Module | Non-admin | Admin |
|---|---|---|
| **certstore** | `CurrentUser\Root`, `CurrentUser\My` (via PowerShell) | Also `LocalMachine\Root`, `LocalMachine\CA`, `LocalMachine\My` — full machine-wide store |
| **certificate** | User-owned cert files | System-wide cert files |
| **key** | User profile keys | System service private keys |
| **binary** | All world-readable PE binaries | All binaries including system32 protected files |
| **library** | All world-readable DLLs | Same |
| **process** | Own processes + accessible handles | All processes via Windows API |
| **password_hash** | Not applicable — Windows uses SAM/NTDS | Not applicable |
| **ebpf_trace**, **tpm**, **uefi** | Skipped — Linux-only | Skipped — Linux-only |
| **kernel** | Skipped — Linux-only | Skipped — Linux-only |

---

### triton-sshagent

The privilege level is the **SSH user's effective UID on the remote host**. Because `triton-sshagent` uploads a full `triton` binary and runs `triton scan` natively on the remote, the same privilege rules apply as running the scanner locally — root unlocks runtime modules in addition to full filesystem access.

#### Linux targets (SSH)

| Module | Non-root SSH user | Root SSH user |
|---|---|---|
| **password_hash** | `/etc/passwd` only (no usable hashes) | Full `/etc/shadow` + `/etc/gshadow` |
| **key** | User-owned `~/.ssh/` keys | All SSH host private keys + `/root/.ssh/` + system service keys |
| **certificate** | World-readable certs only | Also `/etc/ssl/private/`, root-owned service cert dirs |
| **auth_material** | Own `~/.ssh/authorized_keys`, `known_hosts` | All users' auth material + `/root/.ssh/` |
| **vpn_config** | World-readable VPN configs | Pre-shared keys in root-owned WireGuard `/etc/wireguard/*.conf` |
| **secrets_mgr** | Vault config if readable | Vault tokens, AWS system-level credentials |
| **kernel** | Most of `/lib/modules/` (world-readable) | Guaranteed full kernel module tree |
| **db_atrest** | Publicly readable DB configs | Also root-owned PostgreSQL `pg_hba.conf`, MySQL configs with `0600` |
| **archive** | Archives readable by SSH user | Also root-owned archives |
| **binary**, **library** | World-readable ELF files | All binaries and shared objects |
| **script**, **webapp**, **python_ast** | World-readable source files | Also root-owned scripts and web assets |
| **process** | Visible processes (own + world-accessible `/proc`) | All `/proc/<pid>/` entries including other users' processes |
| **ebpf_trace** | Skipped — requires root/CAP_BPF | Full eBPF uprobes on libcrypto/gnutls/nss + kprobes on kernel crypto API |
| **tls_observer** (live) | Skipped — no `CAP_NET_RAW` | Live `AF_PACKET` capture on any interface |
| **tpm** | `/sys/class/tpm` sysfs (usually world-readable) | Same; measured-boot log may require root on some kernels |
| **uefi** | `/sys/firmware/efi/efivars/` — usually root-only | Full Secure Boot key inventory (PK/KEK/db/dbx) |
| All other filesystem modules | World-readable files | All files |

#### macOS targets (SSH) — Not Supported

`triton-sshagent` does not support macOS targets. The embedded binaries are linux/amd64 and linux/arm64 only. Use `triton-agent` (enrolled) for macOS hosts.

#### Windows targets (SSH) — Not Supported

`triton-sshagent` does not support Windows targets. The embedded binaries are Linux-only. Use `triton-agent` (enrolled) for Windows hosts.

---

## OS Coverage Matrix

### triton-agent

| Module | Linux | macOS | Windows | Notes |
|---|:---:|:---:|:---:|---|
| certificate | ✓ | ✓ | ✓ | PEM/DER files on any OS |
| key | ✓ | ✓ | ✓ | SSH, RSA, EC private keys |
| package | ✓ | ✓ | | dpkg/rpm (Linux), Homebrew (macOS) |
| library | ✓ | ✓ | ✓ | ELF (Linux), dylib (macOS), DLL (Windows) |
| binary | ✓ | ✓ | ✓ | ELF, Mach-O, PE |
| kernel | ✓ | | | Linux kernel modules only |
| config | ✓ | ✓ | ✓ | Config file patterns cross-platform |
| script | ✓ | ✓ | ✓ | Shell, Python, Ruby, JS source |
| webapp | ✓ | ✓ | ✓ | HTML/JS/TS source |
| process | ✓ | ✓ | ✓ | `/proc` (Linux), `ps` (macOS), Win API |
| network | ✓ | ✓ | | netstat/ss — no Windows support |
| protocol | ✓ | ✓ | ✓ | TLS handshake probing (outbound) |
| container | ✓ | ✓ | | Docker/Podman/k8s configs |
| certstore | ✓ | ✓ | ✓ | CA bundle (Linux), Keychain (macOS), Registry (Windows) |
| database | ✓ | ✓ | ✓ | DB config file patterns |
| hsm | ✓ | ✓ | ✓ | PKCS#11 / HSM device files |
| ldap | ✓ | ✓ | ✓ | LDAP TLS probe (outbound) |
| codesign | ✓ | ✓ | ✓ | `codesign` (macOS), `osslsigncode` (Linux/Win) |
| deps | ✓ | ✓ | ✓ | go.mod reachability analysis |
| web_server | ✓ | ✓ | | nginx/Apache/Caddy configs |
| vpn_config | ✓ | ✓ | | strongSwan/WireGuard/OpenVPN configs |
| container_signatures | ✓ | ✓ | | cosign, Notary, Sigstore |
| password_hash | ✓ | | | `/etc/shadow` — Linux/BSD only |
| auth_material | ✓ | ✓ | ✓ | SSH keys, authorized_keys |
| deps_ecosystems | ✓ | ✓ | ✓ | npm, pip, Maven, Cargo manifests |
| service_mesh | ✓ | ✓ | | Istio/Envoy/Linkerd configs |
| xml_dsig | ✓ | ✓ | ✓ | XML digital signatures |
| mail_server | ✓ | ✓ | | Postfix/Dovecot TLS configs |
| oci_image | ✓ | ✓ | | OCI/Docker image layer scanning |
| oidc_probe | ✓ | ✓ | ✓ | OIDC discovery endpoint probe |
| k8s_live | ✓ | ✓ | ✓ | Live Kubernetes API |
| dnssec | ✓ | ✓ | ✓ | DNSSEC key material files |
| vpn_runtime | ✓ | | | WireGuard/IPsec runtime state |
| netinfra | ✓ | ✓ | | Network device config files |
| firmware | ✓ | ✓ | | Firmware image analysis |
| messaging | ✓ | ✓ | | Kafka/RabbitMQ/NATS TLS configs |
| db_atrest | ✓ | ✓ | ✓ | Encryption-at-rest config files |
| secrets_mgr | ✓ | ✓ | ✓ | Vault/AWS/GCP credential files |
| supply_chain | ✓ | ✓ | ✓ | SBOM, SLSA, Sigstore attestations |
| kerberos_runtime | ✓ | | | Kerberos runtime on Linux |
| enrollment | ✓ | ✓ | ✓ | MDM/device enrollment certs |
| fido2 | ✓ | ✓ | ✓ | FIDO2/WebAuthn credential files |
| blockchain | ✓ | ✓ | ✓ | Wallet/signing key files |
| helm_chart | ✓ | ✓ | ✓ | Helm chart crypto values |
| asn1_oid | ✓ | ✓ | ✓ | ELF/PE/Mach-O OID byte scanner |
| java_bytecode | ✓ | ✓ | ✓ | JAR/WAR/EAR/class crypto literals |
| dotnet_il | ✓ | ✓ | ✓ | .NET CLI metadata crypto types |
| ebpf_trace | ✓ (root) | | | Linux ≥ 5.8, BTF, root/CAP_BPF |
| tpm | ✓ | | | Linux /sys/class/tpm |
| uefi | ✓ | | | Linux /sys/firmware/efi/efivars |
| archive | ✓ | ✓ | ✓ | ZIP/JAR/TAR nested extraction |
| tls_observer | ✓ (root) | | | Live pcap: Linux root/CAP_NET_RAW; pcap files: any OS |
| ftps | ✓ | ✓ | ✓ | Explicit/implicit FTPS cert extraction |
| ssh_cert | ✓ | ✓ | ✓ | SSH host key + OpenSSH cert (network probe) |
| ldif | ✓ | ✓ | ✓ | LDIF certificate extraction |
| python_ast | ✓ | ✓ | ✓ | Python AST import-graph analysis |

### triton-sshagent (Linux targets only)

`triton-sshagent` uploads a full `triton` binary (linux/amd64 or linux/arm64) and runs `triton scan` natively on the remote Linux host. All modules that work on Linux are active; coverage is identical to running `triton scan` locally as that SSH user. macOS and Windows targets are not supported.

| Module | Linux target | Notes |
|---|:---:|---|
| certificate | ✓ | PEM/DER certs in any accessible path |
| key | ✓ | Private keys in any accessible path |
| library | ✓ | `.so` shared libraries |
| binary | ✓ | ELF executables |
| kernel | ✓ | Linux kernel modules |
| config | ✓ | Application config files |
| script | ✓ | Shell, Python, Ruby, JS source |
| webapp | ✓ | HTML/JS/TS source files |
| web_server | ✓ | nginx/Apache/Caddy config paths |
| vpn_config | ✓ | strongSwan/WireGuard/OpenVPN |
| container_signatures | ✓ | cosign.pub, Notary trust stores |
| password_hash | ✓ | `/etc/shadow` — root SSH required |
| auth_material | ✓ | SSH authorized_keys, known_hosts |
| deps | ✓ | go.mod/go.sum reachability |
| deps_ecosystems | ✓ | npm/pip/Maven/Cargo manifests |
| service_mesh | ✓ | Istio/Envoy/Linkerd configs |
| xml_dsig | ✓ | XML digital signature files |
| mail_server | ✓ | Postfix/Dovecot TLS configs |
| dnssec | ✓ | DNSSEC key material files |
| netinfra | ✓ | Network device config files |
| messaging | ✓ | Kafka/RabbitMQ/NATS TLS configs |
| db_atrest | ✓ | Encryption-at-rest config files |
| secrets_mgr | ✓ | Vault/AWS/GCP credential files |
| supply_chain | ✓ | SBOM/SLSA/Sigstore attestations |
| enrollment | ✓ | MDM/device enrollment certs |
| fido2 | ✓ | FIDO2/WebAuthn credential files |
| blockchain | ✓ | Wallet/signing key files |
| helm_chart | ✓ | Helm chart crypto values |
| archive | ✓ | ZIP/JAR/TAR nested extraction |
| ldif | ✓ | LDIF certificate extraction |
| python_ast | ✓ | Python AST import-graph analysis |
| certstore | ✓ | Linux CA bundle |
| package | ✓ | dpkg/rpm installed package list |
| codesign | ✓ | `osslsigncode` Authenticode verification |
| asn1_oid | ✓ | ELF OID byte scanner (comprehensive) |
| java_bytecode | ✓ | JAR/WAR/EAR/class crypto literals (comprehensive) |
| dotnet_il | ✓ | .NET CLI metadata crypto types (comprehensive) |
| process | ✓ | `/proc` process inspection |
| ebpf_trace | ✓ (root) | eBPF uprobes on libcrypto/gnutls/nss |
| tls_observer | ✓ (root) | Live pcap; requires CAP_NET_RAW |
| tpm | ✓ | `/sys/class/tpm` sysfs |
| uefi | ✓ (root) | `/sys/firmware/efi/efivars/` |
| kerberos_runtime | ✓ | Kerberos keytab + krb5.conf |
| vpn_runtime | ✓ (root) | WireGuard/strongSwan runtime state |
| network | ✓ | `ss`/`netstat` open port enumeration |
| protocol | ✓ | Active TLS probing (outbound) |
| ftps | ✓ | Explicit/implicit FTPS cert extraction |
| ssh_cert | ✓ | SSH host key + OpenSSH cert probe |
| ldap | ✓ | LDAP StartTLS cert probe |
| database | ✓ | Live DB TLS probe (needs DB creds) |
| hsm | ✓ | PKCS#11 token enumeration |
| k8s_live | ✓ | Kubernetes API (needs kubeconfig) |
| oci_image | ✓ | OCI/Docker image layer scanning |
| container | ✓ | Dockerfile/compose/k8s YAML |
| firmware | ✓ | Firmware image analysis |
| oidc_probe | ✓ | OIDC discovery endpoint probe |

**Unsupported target OSes:** macOS and Windows — no embedded binaries for those platforms. Use `triton-agent` (enrolled) for those hosts.

---

## Scanner Module Reference

Full description of every module, with availability and privilege requirements.

### Filesystem modules (available in both triton-agent and triton-sshagent)

| Module | Description | Privilege needed | SSH available |
|---|---|---|:---:|
| **certificate** | Walks the filesystem for PEM/DER X.509 certificates. Classifies key algorithm, size, expiry, SANs, and chain validity. Flags SHA-1/MD5 signatures and near-expiry certs. | None — but root needed for `/etc/ssl/private/` | ✓ |
| **key** | Finds private key files (RSA, EC, Ed25519, OpenSSH format). Assesses key size, algorithm, and quality (ROCA CVE-2017-15361, Debian weak PRNG). | None — but root needed for `/root/.ssh/`, SSH host keys | ✓ |
| **library** | Finds shared libraries (`.so`, `.dylib`, `.dll`) linked to known crypto implementations (OpenSSL, libgcrypt, BouncyCastle, NSS). Classifies version against PQC readiness. | None | ✓ |
| **binary** | Scans ELF/Mach-O/PE executables for crypto library linkage by inspecting dynamic symbol tables and import sections. | None | ✓ |
| **kernel** | Reads Linux kernel modules (`.ko`, `.ko.gz`, `.ko.xz`) from `/lib/modules/$(uname -r)/`. Identifies crypto algorithm registrations. Linux only. | None generally; root helps for compressed modules on some distros | ✓ (Linux only) |
| **script** | Parses shell, Ruby, and JavaScript source files for crypto API calls, hard-coded keys, and weak algorithm references. | None | ✓ |
| **webapp** | Scans HTML, TypeScript, and web framework config files for client-side crypto usage (Web Crypto API, TLS pinning, JWT libraries). | None | ✓ |
| **web_server** | Parses nginx `ssl_protocols`/`ssl_ciphers`, Apache `SSLProtocol`, Caddy TLS blocks, and HAProxy TLS config. Flags insecure protocol versions and weak cipher strings. | None | ✓ |
| **vpn_config** | Reads strongSwan IKE/ESP proposals, WireGuard `PrivateKey` presence, and OpenVPN `cipher`/`tls-*` directives. Classifies post-quantum readiness. | None for most; root for WireGuard PSK (`0600`) | ✓ |
| **container_signatures** | Looks for cosign public/private key pairs, Notary v1 trust store, Sigstore root TUF metadata, and Kubernetes encryption config YAML. Warns on `identity` (plaintext) provider. | None | ✓ |
| **password_hash** | Reads `/etc/shadow` and `/etc/gshadow` to classify the hash algorithm per user (bcrypt, SHA-512, SHA-256, MD5, yescrypt). Linux only; skipped on macOS/Windows. | **Root required** for `/etc/shadow` | ✓ (Linux, root) |
| **auth_material** | Parses `~/.ssh/authorized_keys` and `~/.ssh/known_hosts` for key types and sizes across all user home directories. | None for own home; root for all users | ✓ |
| **deps_ecosystems** | Finds `package.json`, `requirements.txt`, `pom.xml`, `Cargo.toml`, `go.mod`, `Gemfile.lock`, and classifies crypto dependencies by ecosystem. | None | ✓ |
| **service_mesh** | Reads Istio, Envoy, and Linkerd mTLS configuration files (YAML/JSON). Flags mutual TLS settings and cipher suite overrides. | None | ✓ |
| **xml_dsig** | Parses XML files for `<ds:Signature>` elements. Classifies the signature algorithm (RSA, ECDSA, HMAC) and digest method (SHA-1, SHA-256, SHA-384). | None | ✓ |
| **mail_server** | Reads Postfix `smtpd_tls_*` and Dovecot `ssl_*` directives. Flags self-signed certs, weak protocols, and missing STARTTLS enforcement. | None | ✓ |
| **dnssec** | Finds BIND/NSD/Knot zone key files (`.key`, `.private`) and `named.conf` DNSSEC settings. Classifies key algorithm (RSA, ECDSA, Ed25519). | None | ✓ |
| **netinfra** | Parses network device configuration files (Cisco IOS, Juniper JunOS, Arista EOS) for SSH/TLS/IPsec algorithm settings. | None | ✓ |
| **messaging** | Reads Kafka TLS keystore/truststore config, RabbitMQ `ssl_options`, and NATS TLS directives. | None | ✓ |
| **db_atrest** | Parses PostgreSQL `pg_hba.conf`, MySQL `my.cnf`, MongoDB `net.tls`, and Redis `tls-*` for encryption-at-rest and in-transit settings. | None for world-readable; root for `0600` DB configs | ✓ |
| **secrets_mgr** | Looks for HashiCorp Vault agent config, AWS credential files, GCP service account JSON, and Azure managed identity tokens. Flags plaintext secrets. | None for user-owned; root for system service credentials | ✓ |
| **supply_chain** | Reads SPDX/CycloneDX SBOM files, SLSA provenance attestations, and Sigstore `rekor`/`fulcio` root metadata. | None | ✓ |
| **enrollment** | Parses MDM enrollment profiles (macOS `.mobileconfig`, Linux SCEP configs) for certificate and crypto algorithm settings. | None | ✓ |
| **fido2** | Finds FIDO2/WebAuthn authenticator credential files and relying party configurations. Classifies credential algorithm. | None | ✓ |
| **blockchain** | Scans for Ethereum keystore JSON files, Bitcoin wallet files, and raw PEM signing keys used by blockchain tooling. | None | ✓ |
| **helm_chart** | Reads Helm `values.yaml` and chart templates for TLS certificate references, CA bundle paths, and crypto config overrides. | None | ✓ |
| **archive** | Extracts ZIP/JAR/WAR/EAR/TAR archives (up to 2 levels deep, with zip-bomb protection) and delegates cert/key parsing to other modules. | None | ✓ |
| **ldif** | Parses LDIF files for base64-encoded `userCertificate`, `cACertificate`, and `userSMIMECertificate` attributes. Supports RFC 2849 folded lines. | None | ✓ |
| **python_ast** | Two-phase AST walk of Python source files: resolves imports then classifies `hashlib`, `cryptography`, `pycryptodome`, and PyCQC API calls. Produces reachability (direct/transitive) findings. | None | ✓ |

---

### Modules requiring elevated privilege (both triton-agent and triton-sshagent)

These modules run in both deployment modes when the user has the necessary privilege. In triton-sshagent, the SSH user's UID on the remote host is the effective privilege level.

| Module | Privilege required | Notes |
|---|---|---|
| **password_hash** | Root | `/etc/shadow`/`/etc/gshadow` are root-readable on Linux |
| **ebpf_trace** | Root or CAP_BPF + CAP_PERFMON | Kernel ≥ 5.8 with BTF; Linux only |
| **tls_observer** (live capture) | Root or CAP_NET_RAW | Pcap file mode needs no privilege |
| **uefi** | Root (most distros) | `/sys/firmware/efi/efivars/` |
| **vpn_runtime** | Root | WireGuard/strongSwan kernel interface access |

---

### Network modules

Available in both triton-agent and triton-sshagent. In triton-sshagent these run on the remote Linux host, probing network endpoints reachable from that host.

| Module | Description | Privilege needed | OS support |
|---|---|---|---|
| **network** | Enumerates open ports and listening services via `ss`/`netstat`. Maps services to crypto protocols. | None | Linux, macOS |
| **protocol** | Active TLS probing: cipher enumeration, version range, key exchange classification, chain validation, SAN extraction. Hybrid PQC named-group detection (X25519MLKEM768, etc.). | None | All |
| **ftps** | Connects to port 21 (explicit FTPS) and port 990 (implicit FTPS) to extract the server certificate chain and negotiated cipher. | None | All |
| **ssh_cert** | SSH handshake probe to extract host key algorithm/size and OpenSSH certificate metadata (CA key, validity period, serial). | None | All |
| **oidc_probe** | Fetches `/.well-known/openid-configuration` from configured endpoints; classifies `id_token_signing_alg_values_supported`. | None | All |
| **ldap** | LDAP StartTLS probe to extract server certificate and supported cipher suite. | None | All |

---

### Runtime / hardware modules

Available in both triton-agent and triton-sshagent (Linux targets). In triton-sshagent, runtime modules run natively on the remote host under the SSH user's UID.

| Module | Description | Privilege needed | OS support |
|---|---|---|---|
| **process** | Enumerates running processes and maps them to crypto library linkage via `/proc/<pid>/maps`. | None for own processes; root for all | Linux, macOS, Windows |
| **ebpf_trace** | Attaches eBPF uprobes on libcrypto, GnuTLS, NSS, and kprobes on kernel crypto API. Observes runtime algorithm use during a scan window. Requires kernel ≥ 5.8 with BTF. | **Root or CAP_BPF** | Linux only |
| **tls_observer** | Passive TLS pcap/wire observer. Offline mode: reads `.pcap`/`.pcapng` files (no privilege). Live mode: `AF_PACKET` capture on a named interface. Computes JA3/JA3S/JA4/JA4S fingerprints. | Root or CAP_NET_RAW for live capture | Linux (live); all (pcap files) |
| **tpm** | TPM 2.0 attestation via `/sys/class/tpm` sysfs and TCG measured-boot log. Classifies firmware against CVE registry. | None (sysfs) | Linux only |
| **uefi** | UEFI Secure Boot key inventory from `/sys/firmware/efi/efivars/`. Checks dbx for CVE revocation gaps. | **Root** on most distros | Linux only |
| **kerberos_runtime** | Reads Kerberos keytab files and `krb5.conf` for encryption type settings. Flags DES/RC4 enctypes. | None for world-readable keytabs; root for `/etc/krb5.keytab` | Linux only |
| **vpn_runtime** | Reads WireGuard and strongSwan runtime state (loaded keys, active sessions) from kernel interfaces. | Root | Linux only |

---

### Specialised modules

Available in both triton-agent and triton-sshagent (Linux targets). These modules call live APIs or external tools; they require that the relevant service or tool is accessible from the host running the scan.

| Module | Description | Privilege needed | OS support |
|---|---|---|---|
| **database** | Probes live database instances (PostgreSQL, MySQL, MongoDB, Redis) to enumerate TLS settings and cipher suite negotiation. | None (needs DB credentials) | All |
| **hsm** | Discovers PKCS#11 tokens and HSM devices. Enumerates key slots and mechanism support lists. | None (needs token PIN) | All |
| **k8s_live** | Queries the Kubernetes API for TLS secrets, CA bundles, configmaps with cert data, and encryption-at-rest config. | Needs kubeconfig or in-cluster RBAC | All |
| **oci_image** | Pulls OCI/Docker image manifests and scans layer tarballs for certs, keys, and crypto configs. | None (needs registry auth) | Linux, macOS |

---

## Quick Decision Guide

```
Is the target a Windows or macOS host?
  └─ Yes → triton-agent (enrolled) only
  └─ No (Linux)
       └─ Can you install a persistent daemon?
            └─ Yes → triton-agent
                 └─ Need eBPF/TPM/UEFI/process/live-pcap with root?
                      └─ Yes → triton-agent as root
                      └─ No → triton-agent as service user is sufficient
            └─ No → triton-sshagent
                 └─ Need runtime modules (eBPF, process, live-pcap)?
                      └─ Yes → SSH credentials must be root
                      └─ No → SSH credentials as service/operator user is sufficient
```
