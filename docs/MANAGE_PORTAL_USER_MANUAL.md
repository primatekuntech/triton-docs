# Triton Manage Portal — User Manual

**Version:** 1.0  
**Audience:** IT operators and security administrators  
**Product:** Triton Manage Portal (`triton-manage`)

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [First-Run Setup Wizard](#2-first-run-setup-wizard)
3. [Dashboard](#3-dashboard)
4. [Hosts](#4-hosts)
5. [Tags](#5-tags)
6. [Credentials](#6-credentials)
7. [Network Discovery](#7-network-discovery)
8. [Scan Jobs](#8-scan-jobs)
9. [Scan Batches](#9-scan-batches)
10. [Scan Schedules](#10-scan-schedules)
11. [Enrolled Agents](#11-enrolled-agents)
12. [Scan Results and Push Queue](#12-scan-results-and-push-queue)
13. [Users](#13-users)
14. [License](#14-license)
15. [Settings](#15-settings)
16. [Navigation Reference](#16-navigation-reference)
17. [Common Workflows](#17-common-workflows)
18. [Troubleshooting](#18-troubleshooting)

---

## 1. Introduction

The Triton Manage Portal is a standalone web application that gives your security team a single pane of glass to orchestrate Post-Quantum Cryptography (PQC) compliance scans across a fleet of Linux servers. It is **separate** from the Triton Report Server (where findings are stored and analysed) and the Triton License Server (which issues seat licences). The Manage Portal sits between your host inventory and the scanning infrastructure — it dispatches jobs, tracks results, and forwards completed scan payloads to the Report Server.

### What the Manage Portal does

- Maintains a searchable inventory of target hosts with flexible tagging
- Stores SSH credentials securely in HashiCorp Vault (never in the database)
- Dispatches SSH scan jobs (`triton-sshagent`) that SSH into Linux hosts, upload the Triton binary, and run a scan remotely
- Dispatches port survey jobs (`triton-portscan`) that discover open ports and identify running services
- Manages enrolled `triton-agent` daemons that scan their own host continuously via mTLS
- Discovers new hosts by sweeping CIDR ranges for live SSH endpoints
- Schedules recurring scans using cron expressions
- Forwards all completed results to the Report Server through an encrypted outbox

### What the Manage Portal does not do

- It does not store or display scan findings — those live in the Report Server
- It does not issue licence keys — those are managed in the License Server
- It does not scan Windows hosts via SSH (WinRM support is reserved for a future release)

### Prerequisites before using the portal

- The Manage Portal stack has been deployed (`scripts/install-manage.sh`) by your infrastructure team
- You have the portal URL (typically `https://<hostname>:8082`)
- You have a valid Triton licence issued by the License Server
- For SSH-based scanning: HashiCorp Vault is reachable from the Manage Server
- For result forwarding: the Report Server is reachable from the Manage Server

---

## 2. First-Run Setup Wizard

When you open the Manage Portal for the very first time, it is in **setup mode** — no admin account exists yet and the portal does not accept normal logins. The setup wizard guides you through two mandatory steps. Once both steps are complete, the wizard disappears permanently.

### Step 1 — Create the admin account

**What you will see:** A form asking for email address, display name, and password.

| Field | Required | Notes |
|-------|----------|-------|
| Email address | Yes | This becomes the primary admin login. Use a monitored mailbox. |
| Display name | Yes | Shown in the portal header and audit logs. |
| Password | Yes | Minimum 12 characters; must include at least one digit. Store this securely. |

Click **Create account** to continue. If the portal shows a conflict error, another operator may have completed this step concurrently — try logging in with the credentials they created.

> **Important:** There is no default password and no back door. If you lose this password before creating additional users, your infrastructure team must reset the database to re-enter setup mode.

### Step 2 — Activate the licence

**What you will see:** A form asking for the License Server URL and a Licence ID.

| Field | Required | Notes |
|-------|----------|-------|
| License Server URL | Yes | Full URL including scheme, e.g. `https://license.example.com:8081` |
| Licence ID | Yes | The UUID provided by your Triton licence administrator |

Click **Activate**. The portal contacts the License Server, validates the licence, and stores the tenant context. A successful activation shows your tier (Free / Pro / Enterprise) and seat allocation.

**If activation fails:**
- Verify the License Server URL is reachable from the server running the Manage Portal (not just from your browser).
- Verify the Licence ID is correct and has not been revoked.
- Check that the Manage Server's clock is within 5 minutes of the License Server's clock (JWT validation is time-sensitive).

### After setup

Once both steps are complete, the portal loads the main dashboard and the setup wizard is gone. Log in with the admin email and password you created in Step 1.

---

## 3. Dashboard

The Dashboard is the home screen shown after login. It provides an at-a-glance summary of the portal's current state.

### Overview cards

Four summary cards appear at the top of the page:

| Card | What it shows |
|------|---------------|
| **Hosts** | Total number of hosts in the inventory |
| **Active Jobs** | Count of scan jobs currently in `running` or `queued` state |
| **Enrolled Agents** | Count of `triton-agent` daemons with `active` status |
| **Credentials** | Total number of stored credential records |

### Recent scan jobs

A table showing the last 20 scan jobs across all hosts, ordered by creation time (newest first). Columns include host name, job type, scan profile, status, and when the job was created or last updated. Click any row to go to the job detail page.

### Push queue status

A small status panel at the bottom of the dashboard shows the health of the result-forwarding pipeline to the Report Server:

| Indicator | Meaning |
|-----------|---------|
| **Queue depth** | Number of completed scan results waiting to be forwarded |
| **Last pushed** | Timestamp of the most recent successful push to the Report Server |
| **Last error** | Most recent push error message, if any |
| **Consecutive failures** | Number of back-to-back push failures since the last success |

A green indicator means the queue is draining normally. An amber indicator means the queue has depth but no recent error. A red indicator means pushes are failing — see [Section 12](#12-scan-results-and-push-queue) for troubleshooting steps.

---

## 4. Hosts

The Hosts page is the central inventory of all target machines the Manage Portal can scan. Every scan job requires at least one host.

### Host data fields

| Field | Required | Description |
|-------|----------|-------------|
| Hostname | Yes | FQDN or short name (e.g. `prod-db-01.example.com`). Does not need to be DNS-resolvable — the IP Address is used for connectivity. |
| IP Address | Yes | IPv4 or IPv6 address. Must be unique across all hosts in this portal instance. |
| OS | No | Operating system hint: Windows, Linux, macOS, Unix, or Other. Affects which job types are offered for this host in the scan form. |
| SSH Port | No | The TCP port the SSH daemon listens on. Defaults to 22. Valid range: 1–65535. |
| Connection type | Derived | How the portal reaches this host. See below. |
| Credentials | No | Reference to a stored credential used for SSH access. Required before SSH or port-survey jobs can be dispatched. |
| Bastion host | No | A second host to jump through (ProxyJump) before reaching this host. Only applicable when the connection type is `ssh_bastion`. |
| Tags | No | One or more coloured labels used for filtering and batch targeting. Hosts without tags show a warning indicator. |
| Last seen | Informational | When a scan last completed on this host, or when an enrolled agent last checked in. |

### Connection types

The portal derives a host's connection type automatically — you do not set it directly.

| Type | When it applies |
|------|----------------|
| `ssh` | The host has a credential attached. The portal will SSH in directly using that credential. |
| `ssh_bastion` | The host has both a credential and a bastion host set. The portal will ProxyJump through the bastion. |
| `agent` | A `triton-agent` daemon is enrolled for this host. The portal sends scan commands via the agent's mTLS channel. SSH credentials are not required. |
| `winrm` | Reserved for future Windows Remote Management support. Cannot be selected at this time. |

When a `triton-agent` enrols for a host, the connection type automatically changes to `agent`. If the agent is later revoked, the type reverts to `ssh` (if credentials are present) or becomes unconfigured.

### Adding a host manually

1. Click **+ Add Host** in the top-right corner of the Hosts page.
2. Fill in Hostname and IP Address (required).
3. Optionally set OS, SSH Port, and attach a credential and tags.
4. If the host is only reachable through a jump host, enable **Use bastion** and select the bastion host from the dropdown. The bastion host must already exist in the inventory.
5. Click **Save**.

### Editing a host

Click the host's row to open the detail view, then click **Edit**. All fields except ID can be changed. Changing the IP address fails if the new IP is already in use by another host.

### Deleting a host

Click **Edit** on the host detail page, then **Delete host**. Deletion is blocked when the host has jobs in `queued` or `running` state — cancel those jobs first.

### Bulk CSV import

For large inventories, use the CSV import to add many hosts at once.

1. Click **Import** on the Hosts page and select **Import CSV**.
2. Download the template if needed — it contains the correct column headers.
3. Prepare your CSV file with the following columns:

| Column | Required | Notes |
|--------|----------|-------|
| `hostname` | Yes | Any non-empty string |
| `ip_address` | Yes | Must be unique |
| `os` | No | One of: `Windows`, `Linux`, `macOS`, `Unix`, `Other` (case-insensitive) |
| `ssh_port` | No | Integer 1–65535; blank defaults to 22 |
| `tags` | No | Semicolon-separated tag names, e.g. `production;database`. Unknown tag names are created automatically. |
| `credentials` | No | Name of an existing credential to attach. If the name does not match any credential, the row imports but the host will have no credential. |

4. Upload the file and review the import preview. Rows with validation errors are highlighted in red — hover over the row for the error message.
5. Click **Confirm import**. Rows with duplicate IP addresses are skipped with a warning.

### Filtering and searching

Use the search box at the top of the hosts table to filter by hostname or IP address. Use the **Tag filter** dropdown to show only hosts with a specific tag. Both filters can be combined.

---

## 5. Tags

Tags are coloured labels you assign to hosts. They serve two purposes:

1. **Filtering** — quickly narrow the hosts list to a specific category (e.g. show only `production` hosts)
2. **Batch targeting** — when creating a Scan Batch, you select which tags to include, and the portal automatically selects all matching hosts

Hosts without any tags show a warning indicator (⚠) because they cannot be targeted by tag-based batches.

### Built-in tags

The portal comes with 13 pre-created tags. You can rename or recolour them, but you cannot delete them while any host is attached.

| Tag name | Default colour | Intended use |
|----------|---------------|--------------|
| production | Red | Live production systems |
| staging | Orange | Pre-production environments |
| development | Green | Developer workstations and lab machines |
| web | Blue | Web servers and reverse proxies |
| database | Purple | Database servers |
| windows | Cyan | Windows hosts (reserved for WinRM) |
| linux | Dark green | Linux hosts |
| unix | Gray | Non-Linux Unix systems (FreeBSD, Solaris, etc.) |
| server | Dark blue | Rack-mounted servers |
| laptop | Teal | Laptop endpoints |
| workstation | Violet | Desktop workstations |
| critical | Dark red | Business-critical systems requiring priority scanning |
| pqc-scope | Amber | Systems in scope for PQC compliance assessment |

### Creating a custom tag

1. Go to **Tags** in the sidebar.
2. Click **+ New tag**.
3. Enter a name (must be unique) and pick a colour from the colour picker.
4. Click **Save**.

### Renaming or recolouring a tag

Click the tag row to open its edit form. Change the name or colour and click **Save**. The change takes effect immediately on all hosts that carry this tag.

### Deleting a tag

Click **Delete** on the tag edit form. Deletion is blocked if any host currently has this tag assigned — remove the tag from all hosts first.

---

## 6. Credentials

Credentials store the authentication details the portal uses to SSH into hosts. The portal never stores raw secrets in its PostgreSQL database. All secret material is written to **HashiCorp Vault KV v2**, and the portal stores only the Vault path as a reference.

### Credential types

| Type | When to use |
|------|------------|
| `ssh-key` | SSH public-key authentication. You provide a username and a PEM-encoded private key. An optional passphrase is supported for encrypted keys. |
| `ssh-password` | SSH password authentication. You provide a username and password. Use only when key-based auth is not possible — password auth is less secure. |

A third type, `winrm-password`, is reserved for future Windows Remote Management support and cannot be created at this time.

### Credential data fields

| Field | Required | Notes |
|-------|----------|-------|
| Name | Yes | A human-readable label (e.g. `prod-ssh-key`, `bastion-jumpuser`). Must be unique. |
| Type | Yes | `ssh-key` or `ssh-password` |
| Username | Yes | The OS username on the target host |
| Private key | Yes (ssh-key only) | PEM-encoded RSA, ECDSA, or Ed25519 private key. Paste the full key including `-----BEGIN ... KEY-----` headers. |
| Passphrase | No (ssh-key only) | Only if the private key is passphrase-protected |
| Password | Yes (ssh-password only) | The SSH password |

> **Security note:** The portal transmits credentials to Vault over TLS at save time. Once saved, the raw secret is never returned to the browser or API. The credential record you see in the portal shows only the name, type, creation date, and how many hosts reference it.

### Adding a credential

1. Go to **Credentials** in the sidebar.
2. Click **+ Add credential**.
3. Select the type (`ssh-key` or `ssh-password`).
4. Fill in the fields and click **Save**.

If Vault is unreachable at save time, the portal returns an error and no record is created. Check Vault connectivity before retrying.

### Deleting a credential

Click the credential row, then **Delete**. Deletion is blocked when the credential is attached to one or more hosts — the **In use** count on the credential row shows how many hosts reference it. Remove the credential from those hosts first.

### Rotating a credential

There is no in-place rotation UI. To rotate:

1. Create a new credential with the updated key or password.
2. Edit each host that used the old credential and switch to the new one.
3. Delete the old credential once no hosts reference it.

---

## 7. Network Discovery

Network Discovery sweeps a CIDR range to find hosts that have SSH ports open, allowing you to build your inventory without manually adding every host.

### How it works

1. You submit a CIDR range and optional SSH port list.
2. The portal dispatches a discovery job to `triton-portscan`.
3. The worker connects to each IP in the range and checks whether the specified SSH ports are open (TCP connect).
4. Each reachable IP becomes a **candidate** in the results.
5. You review the candidates and selectively import them into the Hosts inventory.

### Running a discovery scan

1. Go to **Discovery** in the sidebar.
2. Click **+ New discovery**.
3. Fill in the fields:

| Field | Required | Notes |
|-------|----------|-------|
| CIDR range | Yes | Standard CIDR notation, e.g. `10.0.1.0/24`. A `/24` sweeps 254 addresses; limit to `/22` or smaller for interactive use. |
| SSH ports | No | Comma-separated list of TCP ports to check, e.g. `22,2222`. Defaults to port 22. |

4. Click **Start discovery**. The job appears in the discovery jobs table.

### Monitoring discovery progress

The discovery job table shows real-time progress:

| Column | Meaning |
|--------|---------|
| CIDR | The range being swept |
| Status | `queued`, `running`, `completed`, or `failed` |
| Progress | `scanned / total` IPs checked |
| Found | Number of IPs with open SSH ports discovered so far |
| Started | When the sweep began |
| Finished | When the sweep ended (blank if still running) |

Large CIDR ranges (e.g. `/20`, `/16`) can take several minutes. The progress counter updates live.

### Reviewing candidates

When the discovery job completes, click **View results** to see the candidate list. Each candidate row shows:

| Column | Meaning |
|--------|---------|
| IP address | The discovered IP |
| SSH port | The port that responded |
| Detected hostname | Reverse DNS lookup result, if available. May be blank. |
| Already in inventory | An amber warning if this IP already exists as a host in the portal. |

Candidates whose IP is already in the inventory are still shown so you can verify they are up-to-date, but importing them again will create a duplicate — the import step will warn you before proceeding.

### Importing candidates

1. Tick the checkboxes next to candidates you want to import.
2. Click **Import selected**.
3. For each selected candidate, enter or confirm the **hostname** (the detected hostname is pre-filled if available, but it may be an IP address — replace it with a meaningful name).
4. Optionally set OS, SSH port override, and tags for each host in the import list.
5. Click **Confirm import**. Successfully imported hosts appear in the Hosts inventory.

If an import fails for a specific IP (e.g. due to a duplicate IP), that row is shown with an error while others succeed.

---

## 8. Scan Jobs

Scan Jobs are individual units of scanning work dispatched against a single host. The portal supports three job types.

### Job types

| Type | Worker | What it does |
|------|--------|-------------|
| `ssh` | `triton-sshagent` | SSHes into the host, uploads the Triton binary, runs `triton scan`, collects results, removes the binary, and submits the result to the portal. |
| `port_survey` | `triton-portscan` | TCP-connects to each port in the survey range and uses `fingerprintx` to identify running services. Does not require SSH credentials. |
| `filesystem` | `triton-agent` (enrolled) | Sends a scan command to the host's enrolled `triton-agent`, which runs the scan engine locally. No SSH involved. |

> **Note:** The `filesystem` job type is only available for hosts whose connection type is `agent`. For hosts without an enrolled agent, use `ssh`.

### Scan profiles

All job types accept a **profile** that controls scan depth and the set of modules run:

| Profile | Modules | Depth | Workers | Typical duration |
|---------|---------|-------|---------|-----------------|
| `quick` | Certificates, keys, packages | 3 | 4 | Seconds to 1–2 minutes |
| `standard` | All quick modules + libraries, binaries, scripts, web apps, configs, containers, databases, and more (28+ modules) | 10 | 8 | 2–10 minutes |
| `comprehensive` | All 56 modules including Java bytecode, .NET IL, eBPF tracing, TPM attestation, UEFI Secure Boot inventory, passive TLS pcap, and Python AST | Unlimited | 16 | 10–60+ minutes |

> For most compliance sweeps, **standard** is the recommended starting profile. Use **comprehensive** only on critical systems or when you need the broadest possible coverage and have confirmed the target host can sustain the load.

### Job status lifecycle

Jobs move through the following states:

```
queued → running → completed
                 → failed
                 → cancelled
```

| Status | Meaning |
|--------|---------|
| `queued` | Job is waiting for a worker to pick it up |
| `running` | A worker has claimed the job and the scan is in progress |
| `completed` | Scan finished successfully; result forwarded to Report Server |
| `failed` | Scan encountered an error; see the progress text for details |
| `cancelled` | An operator or the system cancelled the job |

### Submitting an individual SSH scan job

1. Go to **Scan Jobs** in the sidebar.
2. Click **+ New job**.
3. Select **Job type**: `ssh`.
4. Select the **Target host** from the dropdown.
5. Select the **Scan profile** (`quick`, `standard`, or `comprehensive`).
6. The **Credentials** field pre-fills with the credential attached to the host. Override it here if needed.
7. Optionally expand **Advanced options** to set resource limits:

| Resource limit | Description | Example |
|----------------|-------------|---------|
| Max CPU % | Limit the scan process to this percentage of one CPU core. 0 means unlimited. | `50` |
| Max memory (MB) | Kill the scan if it exceeds this RSS. 0 means unlimited. | `1024` |
| Max duration (seconds) | Abort the scan after this many seconds. 0 means unlimited. | `3600` |

8. Click **Submit job**.

### Submitting a port survey job

1. Click **+ New job**, select **Job type**: `port_survey`.
2. Select the target host.
3. Set the **Scheduled at** time — leave blank for immediate dispatch or pick a future date/time.
4. Optionally override the **Port range** (defaults to the host's configured SSH port for SSH probing, plus common service ports).
5. Click **Submit job**.

### Viewing job details

Click any job row to open the detail view. It shows:

- Host name and IP
- Job type and profile
- Current status with the last **progress text** from the worker (e.g. "Scanning /etc... found 12 certificates")
- Timestamps (queued at, started at, finished at)
- Resource limits (if set)
- Bastion details (if applicable)

### Cancelling a running job

Open the job detail view and click **Cancel job**. Cancellation is cooperative — the worker receives a cancellation signal and stops at the next safe checkpoint. The job moves to `cancelled` within a few seconds to a few minutes depending on the current scan phase.

### Troubleshooting failed jobs

When a job shows `failed`, open the job detail view and read the **progress text** — the worker writes its last log message there before failing. Common causes:

| Error pattern | Likely cause and fix |
|---------------|---------------------|
| `dial tcp: connection refused` | The host is unreachable on the configured SSH port. Verify the host is up, the IP is correct, and port access is allowed by the firewall. |
| `ssh: handshake failed: ssh: unable to authenticate` | The credential is wrong — wrong username, key does not match, or passphrase is incorrect. Check the credential and re-test manually with your SSH client. |
| `vault: secret not found` | The credential's Vault entry was deleted after the credential record was created. Re-create the credential. |
| `scan exceeded max duration` | The resource limit was hit. Increase `Max duration` or narrow the scan profile. |
| `no binary for target arch` | The target host is not `linux/amd64` or `linux/arm64`. SSH scanning only supports these architectures. Use an enrolled `triton-agent` for other platforms. |

---

## 9. Scan Batches

A Scan Batch lets you dispatch scan jobs to many hosts simultaneously without submitting them one at a time. Batches are the primary way to run compliance sweeps across your fleet.

### Creating a batch

1. Go to **Scan Jobs** in the sidebar and click the **Batches** tab.
2. Click **+ New batch**.
3. Configure the batch:

| Field | Required | Description |
|-------|----------|-------------|
| Job types | Yes | One or more of `ssh`, `port_survey`, `filesystem`. The portal creates one job per (host, job type) combination. |
| Hosts | Yes | Select hosts individually, or use the **Select by tag** picker to include all hosts bearing one or more tags. |
| Profile | Yes | `quick`, `standard`, or `comprehensive`. Applied to all jobs in the batch. |
| Max CPU % | No | Resource limit applied to every job in the batch. |
| Max memory (MB) | No | Resource limit applied to every job in the batch. |
| Max duration (s) | No | Resource limit applied to every job in the batch. |

4. Click **Create batch**.

### Batch creation response

After submitting, the portal shows a summary:

- **Batch ID** — reference for the batch
- **Jobs created** — number of jobs successfully queued
- **Jobs skipped** — hosts that were excluded with a reason

The most common skip reason is `no_credential`: the host has no credential attached and the batch includes an `ssh` or `port_survey` job type that requires one. Attach a credential to those hosts and re-submit.

### Batch status

The batch status reflects the aggregate state of all its jobs:

| Batch status | Meaning |
|-------------|---------|
| `queued` | All jobs are still queued |
| `running` | At least one job is running |
| `completed` | All jobs reached a terminal state (completed, failed, or cancelled) |
| `failed` | All terminal-state jobs failed |
| `cancelled` | An operator cancelled the batch |

Click the batch row to see per-host job statuses within the batch.

### Cancelling a batch

Open the batch detail view and click **Cancel batch**. This signals cancellation on all jobs that are still `queued` or `running`.

---

## 10. Scan Schedules

Scan Schedules run Scan Batches automatically on a recurring cron schedule. Use schedules to ensure your fleet is scanned regularly without manual intervention.

### Schedule data fields

| Field | Required | Description |
|-------|----------|-------------|
| Name | Yes | A descriptive label, e.g. `Weekly production sweep` |
| Job types | Yes | Same as batch: `ssh`, `port_survey`, and/or `filesystem` |
| Host selection | Yes | Individual hosts or tag-based selection |
| Profile | Yes | `quick`, `standard`, or `comprehensive` |
| Cron expression | Yes | Standard 5-field cron, e.g. `0 2 * * 1` (Mondays at 02:00 local time). See examples below. |
| Enabled | Yes | Schedules are created enabled by default. Toggle off to pause without deleting. |
| Max CPU % | No | Resource limit forwarded to every job the schedule creates |
| Max memory (MB) | No | Resource limit forwarded to every job the schedule creates |
| Max duration (s) | No | Resource limit forwarded to every job the schedule creates |

### Cron expression examples

| Expression | Meaning |
|------------|---------|
| `0 2 * * *` | Every day at 02:00 |
| `0 2 * * 1` | Every Monday at 02:00 |
| `0 2 * * 1,4` | Every Monday and Thursday at 02:00 |
| `0 */6 * * *` | Every 6 hours |
| `0 1 1 * *` | First day of every month at 01:00 |
| `30 23 * * 5` | Every Friday at 23:30 |

The schedule uses the **local time of the Manage Server**. Confirm the server timezone with your infrastructure team if scan times are unexpected.

### Creating a schedule

1. Go to **Schedules** in the sidebar.
2. Click **+ New schedule**.
3. Fill in the name, select job types and hosts/tags, choose a profile, and enter the cron expression.
4. Click **Save**. The portal calculates and displays the **Next run** time immediately.

### Managing schedules

The schedules list shows:

| Column | Meaning |
|--------|---------|
| Name | Schedule name |
| Cron | The cron expression |
| Profile | Scan profile |
| Enabled | Green = active, grey = paused |
| Last run | When the schedule last triggered a batch |
| Next run | When the schedule will next trigger |

**To pause a schedule:** click the schedule row, then toggle **Enabled** to off and click **Save**. The schedule remains in the list and can be re-enabled at any time. When disabled, `Next run` shows `—`.

**To change the timing:** click the schedule row, update the cron expression, and click **Save**. The new `Next run` is calculated immediately.

**To delete a schedule:** click the schedule row, then **Delete**. This does not cancel any batches that are currently running — only future triggers are stopped.

### What happens when a schedule fires

At the scheduled time, the portal creates a Scan Batch using the schedule's configuration. The batch appears in the Batches tab with a reference back to the schedule that triggered it. If the previous batch triggered by this schedule is still running, the new batch is created anyway — both will run concurrently.

---

## 11. Enrolled Agents

`triton-agent` is a lightweight daemon that installs on a target host and continuously makes itself available for on-demand scans. Enrolled agents are the preferred scanning method for hosts that need real-time scan dispatch, support Windows targets, or require runtime modules (eBPF, live pcap, TPM attestation).

### How agent enrolment works

Each enrolled agent gets a unique mTLS client certificate signed by the Manage Portal's internal Certificate Authority. The agent uses this certificate to authenticate to the portal's mTLS gateway (port 8443). When the portal revokes an agent, its certificate serial number is blacklisted and future connections from that agent are refused.

### Agent lifecycle

| Status | Meaning |
|--------|---------|
| `pending` | Enrolment bundle has been generated and downloaded, but the agent has not yet connected to the portal. |
| `active` | The agent has connected at least once and is regularly checking in. |
| `revoked` | An administrator revoked the agent. The certificate serial is blacklisted; the agent cannot reconnect. |

### Viewing enrolled agents

Go to **Agents** in the sidebar. The agents list shows:

| Column | Meaning |
|--------|---------|
| Name | The agent's display name set at enrolment |
| Status | `pending`, `active`, or `revoked` |
| Certificate expires | The mTLS certificate expiry date. An amber warning appears when expiry is within 30 days. |
| Last seen | When the agent last checked in with the portal |
| Pending command | If a scan command has been pushed but not yet picked up, it shows here |

### Enrolling a new agent

Enrolment generates a bundle that the host operator uses to install and configure the `triton-agent` daemon.

1. Go to **Agents**, click **+ Enrol agent**.
2. Enter a **Name** for this agent (e.g. the hostname or a descriptive label).
3. Click **Generate bundle**. The portal creates a signed mTLS certificate and produces a download package containing:
   - `agent.yaml` — pre-configured with the portal's gateway URL and mTLS certificate paths
   - `client.crt` and `client.key` — the agent's mTLS identity
   - `ca.crt` — the portal's CA certificate for the agent to verify the gateway
   - `install.sh` — a shell script that installs `triton-agent` as a system service
4. Click **Download bundle** and deliver the bundle to the host operator responsible for that machine.
5. The agent appears in the list with status `pending` until the host operator runs the install script and the agent connects.

> **Security note:** The bundle contains private key material. Deliver it over an encrypted channel (e.g. SFTP, encrypted email, or a secrets manager). Do not post it in chat or email in plain text.

**To install on the target host** (performed by the host operator, not in the portal):

```
tar -xzf triton-agent-bundle.tar.gz
sudo bash install.sh
```

The install script places the binary, config, and certificates in the appropriate system directories and registers `triton-agent` as a systemd service (Linux), launchd daemon (macOS), or Windows Service (Windows). Once running, the agent connects to the portal and its status changes from `pending` to `active` within a few seconds.

### Pushing an on-demand scan to an agent

1. Go to **Agents** and click the active agent row.
2. Click **Push scan command**.
3. Select the **Scan profile** (`quick`, `standard`, or `comprehensive`).
4. Click **Push**. The portal stores the command; the agent picks it up on its next poll (typically within 30 seconds) and runs the scan.

> On-demand scan commands are one-shot: once the agent picks up the command, it is cleared. The scan result is forwarded through the normal push queue to the Report Server.

### Revoking an agent

Revocation permanently blocks an enrolled agent from connecting to the portal.

1. Go to **Agents** and click the agent row.
2. Click **Revoke agent**.
3. Confirm the action. The agent's certificate serial number is added to the blacklist immediately.

The agent will receive a TLS error on its next poll attempt and will stop being able to connect. To re-enable a revoked host, enrol a new agent (a new certificate is issued).

> Revocation does not uninstall the agent from the host. The host operator should stop and remove the `triton-agent` service separately if access is no longer needed.

### Certificate expiry warnings

When an agent's mTLS certificate expires within 30 days, its row is highlighted in amber. You must re-enrol the agent before the certificate expires, or it will lose connectivity to the portal.

**To renew an agent certificate:**

1. Revoke the existing agent.
2. Enrol a new agent with the same name.
3. Deliver the new bundle to the host operator and re-run the install script.

---

## 12. Scan Results and Push Queue

When a scan job completes (via SSH, port survey, or enrolled agent), the result is written to an outbox queue in the portal's database. A background goroutine drains the queue by forwarding results to the Report Server over mTLS. This asynchronous design means scan results are never lost even if the Report Server is temporarily unreachable — they accumulate in the queue and are delivered once connectivity is restored.

### Push queue status

The push queue status is visible on the Dashboard (summary panel) and in full detail at **Settings > Push queue**.

| Field | Meaning |
|-------|---------|
| Queue depth | Number of results waiting to be forwarded |
| Oldest row age | How long the oldest queued result has been waiting (in seconds) |
| Last pushed at | Timestamp of the most recent successful push |
| Last push error | The most recent error message from a failed push attempt |
| Consecutive failures | How many back-to-back push attempts have failed |

### Interpreting queue health

| State | What it means |
|-------|--------------|
| Queue depth = 0, no errors | Healthy — results are being forwarded promptly |
| Queue depth > 0, last pushed recently | Normal — results are accumulating faster than they drain, or a burst of jobs just completed. Monitor over the next few minutes. |
| Consecutive failures > 0 | The portal cannot reach the Report Server. Check connectivity. |
| Consecutive failures > 5, oldest row age is hours | The push pipeline has been broken for a while. Immediate attention required. |

### Troubleshooting push failures

**Step 1 — Check the error message.** The `last_push_error` field usually contains the HTTP status code or network error returned by the Report Server. Common values:

| Error | Cause and action |
|-------|-----------------|
| `connection refused` / `no route to host` | The Manage Portal cannot reach the Report Server's URL. Verify network connectivity and firewall rules between the two servers. |
| `x509: certificate signed by unknown authority` | The Report Server's TLS certificate is not trusted by the Manage Portal. Ensure the Report Server's CA certificate is in the Manage Portal's trust bundle. |
| `403 Forbidden` | The mTLS client certificate the portal is using to authenticate to the Report Server has expired or been revoked. Re-run the Report Server enrolment to issue a new mTLS bundle. |
| `401 Unauthorized` | The tenant ID in the push credentials does not match what the Report Server expects. Re-activate the portal's licence to refresh the tenant context. |

**Step 2 — Verify Report Server reachability** from the Manage Server host (not from your browser).

**Step 3 — Re-activate the mTLS bundle.** If the push credentials (mTLS certificate) have expired, go to **Settings > Report Server connection** and re-run the connection setup.

**Step 4 — Queue depth does not drain after fixing connectivity.** The drain goroutine retries automatically with exponential backoff. After resolving the connectivity issue, allow 1–5 minutes for the queue to begin draining. If it does not, restart the Manage Portal container.

---

## 13. Users

The Users section manages who can log into the Manage Portal. These are Manage Portal users only — they are separate from License Server users and Report Server users.

### Roles

| Role | Access level |
|------|-------------|
| `admin` | Full access to all portal features, including user management, agent revocation, credential management, and licence settings. |
| `network_engineer` | Can view hosts, run scans, view jobs, and monitor the push queue. Cannot manage users, credentials, agents, or licence settings. |

### Listing users

Go to **Settings > Users**. The list shows each user's email, display name, role, and when their account was created.

### Creating a user

1. Click **+ Add user**.
2. Enter the user's **Email address**, **Display name**, and select their **Role**.
3. Click **Create**. The portal generates a temporary password and displays it once in the confirmation dialog.
4. Send the temporary password to the new user via a secure channel.

> The new user will be required to change their password on first login. The temporary password is shown only once — if you close the dialog without copying it, you must delete and re-create the user.

If the portal shows a **"Licence seat cap exceeded"** error, your current licence does not allow more users. Contact your licence administrator to increase the seat allocation.

### Deleting a user

Click the user row and select **Delete**. You cannot delete your own account or the last remaining admin user — these actions are blocked to prevent accidental lockout.

### Password resets

There is no self-service password reset in the current version. To reset a user's password:

1. Delete the user account.
2. Re-create the user with the same email address.
3. Deliver the new temporary password to the user.

Audit log entries from the deleted account remain linked to the email address for compliance purposes.

---

## 14. License

The License page shows the current activation state of the Manage Portal and allows you to activate or deactivate.

### Viewing licence details

Go to **Settings > Licence**. The page shows:

| Field | Meaning |
|-------|---------|
| Tier | `Free`, `Pro`, or `Enterprise` |
| Seat cap | Maximum number of portal users allowed (`-1` = unlimited) |
| Host cap | Maximum number of hosts in the inventory |
| Agent cap | Maximum number of enrolled agents |
| Monthly scan cap | Maximum scan jobs per calendar month |
| Instance name | The display name for this instance shown in the Report Server |
| License Server URL | The License Server this instance is activated against |
| Last validated | When the portal last successfully validated its token against the License Server |
| Pending deactivation | If shown, a deactivation has been requested and will complete on the next validation cycle |

### Activating a licence

If the portal shows a **"Licence inactive"** banner:

1. Go to **Settings > Licence**.
2. Click **Activate**.
3. Enter the License Server URL and the Licence ID.
4. Click **Activate**. The portal contacts the License Server and stores the signed token.

### Deactivating a licence

Deactivation releases the seat on the License Server, making it available for another Manage Portal instance.

1. Go to **Settings > Licence**.
2. Click **Deactivate**.
3. Confirm. The portal marks the token for deactivation and contacts the License Server on the next validation cycle.

> After deactivation, the portal enters a 7-day offline grace period during which it continues to function with the cached tier. After 7 days without a valid activation, the portal degrades to the Free tier.

### Licence limits in practice

| Limit | What is blocked when exceeded |
|-------|-------------------------------|
| Seat cap | New users cannot be created |
| Host cap | New hosts cannot be added |
| Agent cap | New agent enrolment bundles cannot be generated |
| Monthly scan cap | New scan jobs and batches cannot be submitted |

When a limit is hit, the relevant creation form shows an error. The current cap and usage values are always visible on the Licence page.

---

## 15. Settings

The Settings section covers server-level configuration for the Manage Portal instance. Changes here affect all users of this portal.

### Server settings

**Instance name** — the display name for this Manage Portal instance as it appears in the Report Server and audit logs. Change it here and click **Save**.

### Push queue

Shows the full push queue status (same data as the Dashboard panel, with more detail). See [Section 12](#12-scan-results-and-push-queue) for interpretation.

### Report Server connection

Displays the Report Server URL this portal is forwarding results to, and the mTLS certificate status. If the mTLS certificate is expired or missing, re-run the connection setup:

1. Click **Reconfigure connection**.
2. Enter the Report Server URL.
3. Follow the on-screen prompts to generate and register a new mTLS bundle.

### Users

See [Section 13](#13-users).

### Licence

See [Section 14](#14-license).

---

## 16. Navigation Reference

The sidebar is the primary navigation. Items appear in the following order:

| Sidebar item | Section in this manual |
|-------------|------------------------|
| Dashboard | Section 3 |
| Hosts | Section 4 |
| Tags | Section 5 |
| Credentials | Section 6 |
| Discovery | Section 7 |
| Scan Jobs | Section 8 (individual jobs and batches) |
| Schedules | Section 10 |
| Agents | Section 11 |
| Settings | Section 15 (contains Users, Licence, and Server settings) |

The portal uses hash-based routing (e.g. `/#/hosts`, `/#/credentials`). Browser history navigation (back/forward) works as expected.

---

## 17. Troubleshooting

This section collects common problems that do not fit neatly into a single feature section.

### Portal shows "Service Unavailable" or 503 on all pages after login

The licence is inactive. Go to **Settings > Licence** and activate with a valid Licence ID. If the portal never completed the setup wizard, see [Section 2](#2-first-run-setup-wizard).

### Portal is stuck in setup mode (wizard reappears after login)

Setup mode is active when either step of the wizard has not been completed. This can happen if:

- The admin account was created but licence activation failed — complete activation.
- The database was reset — run the setup wizard from the beginning.

### I forgot the admin password and cannot log in

There is no self-service password reset. Your infrastructure team must connect to the PostgreSQL database and truncate the `manage_users` table, which resets the portal to setup mode. All host, credential, and job data is preserved. Re-run the setup wizard to create a new admin account.

### Scan jobs stay in "queued" indefinitely

Queued jobs are picked up by workers (`triton-sshagent` or `triton-portscan`). If no worker is running:

1. Check the **Workers** health indicator in the portal (if visible on the dashboard).
2. Verify that the `triton-sshagent` and `triton-portscan` containers/services are running and can reach the Manage Server on port 8080.
3. Check worker logs for connection errors or missing `TRITON_WORKER_KEY` environment variables.

### SSH scan jobs fail immediately with "no binary for target arch"

The SSH scanning worker (`triton-sshagent`) only ships embedded binaries for `linux/amd64` and `linux/arm64`. Hosts running other architectures (32-bit ARM, MIPS, etc.) cannot be scanned via SSH. Use an enrolled `triton-agent` instead, which runs natively on the target.

### Discovery sweep finds no hosts even though SSH servers are running

- Confirm the CIDR range is correct and the Manage Server can route traffic to those addresses.
- Check that firewall rules allow TCP traffic from the Manage Server to the target range on the SSH port.
- Try port 22 explicitly if you left the SSH ports field blank.
- Very large CIDR ranges (e.g. `/16`) may time out before completing — use smaller ranges.

### Credentials form shows "Vault unavailable"

The Manage Server cannot reach HashiCorp Vault. The credential cannot be saved until Vault is reachable. Check:

1. The Vault address configured in the Manage Server's environment variables.
2. Network connectivity and TLS certificate trust between the Manage Server and Vault.
3. The Vault token or AppRole credentials have not expired.

### Enrolled agent certificate expires in less than 30 days

Certificates are issued with a fixed validity period (typically 1 year). There is no automatic renewal. To renew:

1. Revoke the existing agent in the Agents page.
2. Enrol a new agent under the same name.
3. Deliver the new bundle to the host operator and re-run `install.sh` to replace the certificates.

The agent status will briefly return to `pending` until the new agent binary connects to the portal.

### Scan results appear in the portal as "completed" but are not visible in the Report Server

Results are forwarded asynchronously. Check:

1. The **Push queue** status — if the queue has depth and consecutive failures, results are being held back. See [Section 12](#12-scan-results-and-push-queue) for remediation steps.
2. After fixing connectivity, allow 1–5 minutes for the queue to drain.
3. In the Report Server, confirm you are looking at the correct tenant / organisation — the Manage Portal's tenant ID must match the Report Server's expectation.

### Batch shows "X jobs skipped — reason: no_credential"

Some hosts in the batch target set have no credential attached. To fix:

1. Go to **Hosts** and filter for the hosts in question.
2. Edit each host and attach the appropriate credential.
3. Re-submit the batch (or create a new batch targeting only those hosts).

---

## 17. Common Workflows

This section walks through the most frequent day-to-day tasks in the order an operator would perform them.

---

### Workflow A — Onboard a new server and run a first scan

**Scenario:** A new Linux server has been provisioned and you need to run a PQC compliance scan on it.

**Prerequisites:** The server is reachable via SSH from the Manage Server host. You have the SSH private key (or password) for a user account on the server.

**Steps:**

1. **Add a credential** (Section 6)
   - Go to **Credentials > + Add credential**.
   - Select type `ssh-key`, enter the username (e.g. `svc-triton`), paste the private key PEM, and give the credential a name (e.g. `prod-key`).
   - Click **Save**. If Vault is unreachable, resolve that before proceeding.

2. **Add the host** (Section 4)
   - Go to **Hosts > + Add Host**.
   - Enter hostname and IP address.
   - Set OS to `Linux`, SSH port to `22` (or the actual port).
   - Select the credential you just created from the **Credentials** dropdown.
   - Assign at least one tag — for a production server, select `production` and `linux`.
   - Click **Save**.

3. **Submit a scan job** (Section 8)
   - Go to **Scan Jobs > + New job**.
   - Select **Job type**: `ssh`.
   - Select the host you just added.
   - Select **Profile**: `standard` (recommended for a first scan).
   - Leave resource limits at default (unlimited).
   - Click **Submit job**.

4. **Monitor the job**
   - The job appears in the jobs table with status `queued`, then `running` within a few seconds once a worker claims it.
   - Click the job row to watch the **progress text** update in real time.
   - A standard profile scan on a typical Linux server takes 2–10 minutes.

5. **Verify the result was forwarded**
   - Once the job reaches `completed`, go to the **Dashboard** and check the push queue depth.
   - After a few seconds the result should appear in your Report Server under this host's hostname.

---

### Workflow B — Discover and import hosts from a subnet

**Scenario:** Your security team has been handed a list of IP ranges and asked to onboard all reachable SSH servers.

**Steps:**

1. **Run a discovery sweep** (Section 7)
   - Go to **Discovery > + New discovery**.
   - Enter the CIDR range, e.g. `192.168.10.0/24`.
   - Leave SSH ports at default (22) unless you know your environment uses non-standard ports.
   - Click **Start discovery**. For a `/24` this completes in under a minute.

2. **Review candidates**
   - When the job shows `completed`, click **View results**.
   - Review the candidate list. Hosts marked **Already in inventory** are ones you have previously added — skip them to avoid duplicates.
   - Tick the checkboxes for all new hosts you want to import.

3. **Import candidates**
   - Click **Import selected**.
   - For each candidate, confirm or enter a meaningful hostname (the auto-detected reverse-DNS name may be an IP — replace it).
   - Set OS and tags for the batch (you can edit individual hosts later).
   - Click **Confirm import**.

4. **Attach credentials**
   - If all imported hosts share the same SSH key, go to **Hosts**, filter by the tag you just assigned, and edit each host to attach the shared credential.
   - For large imports, consider using the CSV export/re-import flow: export the hosts list, add a `credentials` column in your spreadsheet tool, and re-import.

5. **Run a batch scan** (Section 9)
   - Go to **Scan Jobs > Batches > + New batch**.
   - Select job type `ssh`, profile `standard`.
   - Use **Select by tag** to target the tag you applied during import.
   - Click **Create batch**. Review the skipped-jobs list — any hosts missing credentials will appear here.

---

### Workflow C — Set up a recurring weekly compliance scan

**Scenario:** You want every production server scanned every Monday at 02:00 so results are fresh for the Monday morning compliance meeting.

**Prerequisites:** All production hosts are tagged `production` and have credentials attached.

**Steps:**

1. Go to **Schedules > + New schedule**.
2. Enter a name: `Weekly production scan`.
3. Select **Job types**: `ssh`.
4. Under **Host selection**, choose **Select by tag** and pick `production`.
5. Select **Profile**: `standard`.
6. Enter the cron expression: `0 2 * * 1` (Monday at 02:00 local time).
7. Optionally set a **Max duration** limit (e.g. `7200` seconds / 2 hours) to prevent runaway scans.
8. Click **Save**.

The **Next run** column will immediately show the next Monday at 02:00. When the schedule fires, it creates a Scan Batch targeting all current `production`-tagged hosts — so newly added production hosts are automatically included in future runs without editing the schedule.

**To verify the schedule ran:** on Tuesday morning, go to **Scan Jobs > Batches** and look for a batch created at approximately 02:00 with the schedule's name in the reference column.

---

### Workflow D — Enrol an agent on a critical server and push an on-demand scan

**Scenario:** A critical database server needs continuous scanning availability. You want to be able to trigger a scan from the portal at any time without waiting for SSH job dispatch.

**Steps:**

1. **Enrol the agent** (Section 11)
   - Go to **Agents > + Enrol agent**.
   - Enter a name: `prod-db-01`.
   - Click **Generate bundle**, then **Download bundle**.

2. **Deliver the bundle to the host operator**
   - Send the bundle archive via a secure channel (SFTP or an encrypted file share).
   - The host operator runs: `tar -xzf triton-agent-bundle.tar.gz && sudo bash install.sh`
   - The `triton-agent` service starts automatically and connects to the portal's mTLS gateway.

3. **Confirm enrolment**
   - Return to **Agents**. The `prod-db-01` entry should change from `pending` to `active` within 30 seconds of the agent starting.
   - The host's connection type in **Hosts** automatically changes to `agent`.

4. **Push an on-demand scan**
   - Click the `prod-db-01` agent row.
   - Click **Push scan command**.
   - Select profile `comprehensive` (since this is a critical host and you want full coverage).
   - Click **Push**.
   - The agent picks up the command within 30 seconds and begins scanning. The scan result is forwarded to the Report Server when complete.

5. **Set up a schedule for this host** (optional)
   - Tag the host as `critical` and `pqc-scope`.
   - Create a schedule targeting `pqc-scope` with job type `filesystem` and profile `comprehensive` on a nightly cron.
   - Because the connection type is `agent`, the portal sends filesystem-type jobs which are dispatched via the agent channel — no SSH required.

---

### Workflow E — Rotate an SSH credential across a fleet

**Scenario:** Your SSH key has been rotated for security reasons. You need to update all 40 production hosts to use the new key.

**Steps:**

1. **Add the new credential** (Section 6)
   - Go to **Credentials > + Add credential**.
   - Create a new `ssh-key` credential with the new key. Name it clearly (e.g. `prod-key-2026-05`).
   - Click **Save**.

2. **Update hosts in bulk** (the fastest path uses CSV)
   - Go to **Hosts** and filter by tag `production`.
   - Click **Export CSV** to download the current host list.
   - Open the CSV in a spreadsheet. Change the `credentials` column for all rows to the new credential name (`prod-key-2026-05`).
   - Save the CSV. Go back to **Hosts > Import > Import CSV** and upload the updated file.
   - The portal updates existing hosts (matching on IP address) rather than creating duplicates.

3. **Verify one host**
   - Pick a representative host and submit a quick `ssh` scan job.
   - If the job completes successfully, the new credential is working.

4. **Delete the old credential** (once all hosts have been updated and verified)
   - Go to **Credentials** and find the old credential.
   - Confirm its **In use** count is 0.
   - Click **Delete**.

---

### Workflow F — Investigate a failed scan batch

**Scenario:** You ran a batch scan over the weekend and returned Monday to find 8 of 40 jobs failed.

**Steps:**

1. Go to **Scan Jobs > Batches** and click the batch row.
2. The per-host job list shows which hosts failed.
3. Click each failed job row and read the **progress text** for the error detail.
4. Group failures by error type:
   - **All failed with `connection refused`** — network issue: check firewall, confirm hosts are online.
   - **Mixed failures with `authentication failed`** — key/password mismatch on specific hosts; re-check credentials.
   - **Failed with `scan exceeded max duration`** — resource limit hit; increase the duration limit or use `standard` instead of `comprehensive`.
   - **Failed with `no binary for target arch`** — those hosts are not `linux/amd64` or `linux/arm64`; use enrolled agents.
5. Fix the underlying cause for each group.
6. Re-submit a new batch targeting only the failed hosts:
   - Go to **Batches > + New batch**.
   - Manually select only the affected hosts (do not use a tag filter, as the tag would re-include the already-successful hosts).
   - Use the same profile as the original batch.
   - Click **Create batch**.

---

## 18. Troubleshooting

This section collects common problems that do not fit neatly into a single feature section.

### Portal shows "Service Unavailable" or 503 on all pages after login

The licence is inactive. Go to **Settings > Licence** and activate with a valid Licence ID. If the portal never completed the setup wizard, see [Section 2](#2-first-run-setup-wizard).

### Portal is stuck in setup mode (wizard reappears after login)

Setup mode is active when either step of the wizard has not been completed. This can happen if:

- The admin account was created but licence activation failed — complete activation.
- The database was reset — run the setup wizard from the beginning.

### I forgot the admin password and cannot log in

There is no self-service password reset. Your infrastructure team must connect to the PostgreSQL database and truncate the `manage_users` table, which resets the portal to setup mode. All host, credential, and job data is preserved. Re-run the setup wizard to create a new admin account.

### Scan jobs stay in "queued" indefinitely

Queued jobs are picked up by workers (`triton-sshagent` or `triton-portscan`). If no worker is running:

1. Check whether the workers are running by looking for `triton-sshagent` and `triton-portscan` containers or system services on the worker host.
2. Verify that workers can reach the Manage Server on port 8080 (HTTP API) — test with `curl http://<manage-host>:8080/health` from the worker host.
3. Check worker logs for errors about missing `TRITON_WORKER_KEY` environment variables or TLS certificate errors.
4. Verify the worker key in the worker's environment matches the one configured in the Manage Server.

### SSH scan jobs fail immediately with "no binary for target arch"

The SSH scanning worker (`triton-sshagent`) only ships embedded binaries for `linux/amd64` and `linux/arm64`. Hosts running other architectures (32-bit ARM, MIPS, s390x, etc.) cannot be scanned via SSH. Use an enrolled `triton-agent` instead, which runs natively on the target.

### Discovery sweep finds no hosts even though SSH servers are running

- Confirm the CIDR range is correct and the Manage Server can route TCP traffic to those addresses.
- Check that firewall rules allow outbound TCP from the Manage Server to the target range on the SSH port (22 by default).
- Try specifying the SSH port explicitly (e.g. `2222`) if your environment uses a non-standard port.
- For very large CIDR ranges (e.g. `/16`), the discovery sweep may take many minutes. Check the progress counter before concluding it has found nothing.

### Credentials form shows "Vault unavailable"

The Manage Server cannot reach HashiCorp Vault. The credential cannot be saved until Vault is reachable. Check:

1. The Vault address configured in the Manage Server's environment (`VAULT_ADDR`).
2. Network connectivity and TLS certificate trust between the Manage Server and Vault.
3. The Vault token or AppRole credentials used by the Manage Server have not expired or been revoked.
4. Vault is unsealed — a sealed Vault returns errors that look like unavailability.

### Enrolled agent certificate expires in less than 30 days

Certificates are issued with a fixed validity period (typically 1 year). There is no automatic renewal. To renew:

1. Revoke the existing agent on the **Agents** page.
2. Enrol a new agent under the same name.
3. Deliver the new bundle to the host operator and re-run `install.sh` on the target host to replace the certificates.
4. Restart the `triton-agent` service on the target host.

The agent status will briefly return to `pending` until the new agent binary connects to the portal.

### Scan results appear in the portal as "completed" but are not visible in the Report Server

Results are forwarded asynchronously. Check:

1. The **Push queue** status on the Dashboard or **Settings > Push queue** — if the queue has depth and consecutive failures, results are being held back. See [Section 12](#12-scan-results-and-push-queue) for remediation steps.
2. After fixing connectivity, allow 1–5 minutes for the queue to drain.
3. In the Report Server, confirm you are looking at the correct tenant / organisation — the Manage Portal's tenant ID must match the Report Server's expectation.
4. Verify the Report Server's mTLS client certificate used by the Manage Portal has not expired (visible at **Settings > Report Server connection**).

### Batch shows "X jobs skipped — reason: no_credential"

Some hosts in the batch target set have no credential attached. To fix:

1. Go to **Hosts** and filter for the hosts in question (filter by the tag you used in the batch, then look for the ⚠ credential warning).
2. Edit each host and attach the appropriate credential.
3. Re-submit the batch (or create a new batch targeting only those hosts).

### A schedule's "Next run" shows the wrong time

The cron expression is evaluated in the **local timezone of the Manage Server host**, not your browser's timezone. If the times appear offset, confirm the server timezone with your infrastructure team and adjust the cron expression accordingly.

For example, if the server is in UTC and you want 02:00 Kuala Lumpur time (UTC+8), the cron expression should be `0 18 * * *` (18:00 UTC = 02:00 MYT next day).

### Port survey jobs never complete and stay in "running"

Port survey jobs should complete within a few minutes for a single host. If a job runs for more than 30 minutes:

1. Open the job detail view and read the **progress text** — it may reveal a specific stuck operation.
2. Click **Cancel job** to terminate the stuck job.
3. Re-submit with a shorter `max_duration_s` limit (e.g. `1800` for 30 minutes) so future stuck jobs self-terminate.
4. If the issue recurs on the same host, check whether the host is filtering TCP RST packets (some firewalls silently drop connection attempts, causing long timeouts on each unreachable port).

### User cannot log in after admin changes their role

Role changes take effect on the next login. If the user is currently logged in, they will see their old role until they log out and log back in. Session cache TTL is up to 60 seconds — the updated role will be reflected at most 60 seconds after the change without requiring a logout.

---

*For deployment instructions, environment variable reference, and infrastructure configuration, see `docs/DEPLOYMENT_GUIDE.md`. For scanner module coverage and scanning agent comparison, see `docs/SCANNING_AGENTS.md`.*
