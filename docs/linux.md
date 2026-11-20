# Linux Playbooks

All Linux playbooks target RHEL (Red Hat Enterprise Linux) hosts. They use `become: true` to run as root unless noted otherwise.

**Default inventory group:** `all` (override with AAP inventory limits as needed)

---

## 0 — Sync SSH Keys

**File:** `playbooks/linux/0-rhel_sync_ssh_keys.yml`

Adds a GitHub user's public SSH keys to the `ec2-user` authorized_keys file. Useful for granting workshop students SSH access to lab nodes without manually distributing keys.

### Variables

| Variable | Default | Description |
|----------|---------|-------------|
| *(none)* | | Key URL is hard-coded to `https://github.com/cmedeirorh.keys` — change it to point at your own GitHub account |

### Notes

- `validate_certs: false` is intentional for lab environments where the GitHub certificate chain may not be trusted.
- Uses `ansible.posix.authorized_key`.

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/linux/0-rhel_sync_ssh_keys.yml` |
| Credentials | Machine (SSH) credential for target hosts |
| Inventory | Your RHEL hosts |

---

## 1 — Simple Patching

**File:** `playbooks/linux/1-rhel_simple_patching.yml`

Updates all installed packages on RHEL (excluding the kernel in the first pass, then updating the kernel separately). Reboots automatically if a kernel update was applied.

### Key Behaviour

1. Updates all non-kernel packages via DNF.
2. Updates the kernel package separately.
3. If the kernel was updated, the `Reboot when required` handler fires automatically.

### Variables

None. All hosts in the inventory are targeted.

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/linux/1-rhel_simple_patching.yml` |
| Credentials | Machine credential |

---

## 1.1 — Check Updates and Create ServiceNow Change Request

**File:** `playbooks/linux/1.1-rhel_check_updates_servicenow.yml`

Checks for pending DNF updates on each host and, if updates exist and no active Change Request already exists for that host, creates a **new** ServiceNow Change Request listing every pending package.

### Variables

| Variable | Description |
|----------|-------------|
| `sn_instance` | ServiceNow instance URL |
| `sn_username` | ServiceNow username |
| `sn_password` | ServiceNow password |

### Workflow Position

Run this **before** `1.2-rhel_apply_update_servicenow.yml`. The typical workflow is:

```
1.1 Check → (manual approval in ServiceNow) → 1.2 Apply
```

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/linux/1.1-rhel_check_updates_servicenow.yml` |
| Credentials | Machine credential + ServiceNow credential (extra vars) |

---

## 1.2 — Apply Updates and Close ServiceNow Change Request

**File:** `playbooks/linux/1.2-rhel_apply_update_servicenow.yml`

Applies all pending DNF updates to RHEL hosts, then moves the matching ServiceNow Change Request through the lifecycle (`implement` → `review` → `closed`). Reboots if the kernel or other reboot-requiring packages were updated.

### Variables

| Variable | Description |
|----------|-------------|
| `sn_instance` | ServiceNow instance URL |
| `sn_username` | ServiceNow username |
| `sn_password` | ServiceNow password |

### Prerequisites

A Change Request in `scheduled` state must already exist for each target host (created by `1.1`).

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/linux/1.2-rhel_apply_update_servicenow.yml` |
| Credentials | Machine credential + ServiceNow credential (extra vars) |

---

## 2 — Post-Install Configuration

**File:** `playbooks/linux/2-rhel_post_install.yml`

Runs a standard set of post-install tasks on freshly provisioned RHEL nodes:

- Installs `insights-client` and `cockpit`
- Registers the host with Red Hat Insights
- Deploys an SSH login banner (`files/sshd-banner.j2`)
- Sets `/etc/motd` and `/etc/issue`
- Enables the Cockpit web console socket
- Reboots if required

### Variables

| Variable | Description |
|----------|-------------|
| `admin_message` | Custom message shown in the SSH banner and /etc/issue |
| `public_dns_name` | Public DNS name or IP shown in the banner |

### Dependencies

Requires the `redhat.insights` collection (included in `requirements.yml`).

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/linux/2-rhel_post_install.yml` |
| Credentials | Machine credential + Red Hat network credential (for Insights) |

---

## 3 — SSH Hardening (DevSec)

**File:** `playbooks/linux/3-rhel_devsec_ssh_hardening.yml`

Applies the `devsec.hardening.ssh_hardening` role to enforce SSH best practices (disabling weak ciphers, enforcing key-only auth, etc.).

### Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `ssh_max_auth_retries` | `5` | Maximum SSH authentication attempts |

### Dependencies

Requires the `devsec.hardening` collection. Install it:

```bash
ansible-galaxy collection install devsec.hardening
```

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/linux/3-rhel_devsec_ssh_hardening.yml` |
| Credentials | Machine credential |

---

## 4 — Deploy Zabbix Monitoring Agent

**File:** `playbooks/linux/4-rhel_monitoring_deploy.yml`

Installs and configures the **Zabbix Agent 2** on RHEL hosts, opens the firewall port, and starts the service.

### Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `installer_url` | Zabbix 7.4 RHEL 9 RPM | RPM URL for the Zabbix release package |
| `zabbix_server` | `REPLACE_ME` | **Required** — IP or hostname of your Zabbix server |

> **Important:** Set `zabbix_server` to your Zabbix server address before running.

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/linux/4-rhel_monitoring_deploy.yml` |
| Credentials | Machine credential |
| Extra Variables | `zabbix_server: "192.168.1.10"` |

---

## 5 — Deploy NFS Server

**File:** `playbooks/linux/5-rhel_deploy_nfs_server.yml`

Installs `nfs-utils`, creates a shared directory, and configures NFS exports using the `files/exports.j2` template.

### Variables

| Variable | Description |
|----------|-------------|
| `nfs_folder` | **Required** — path of the directory to share (e.g. `/exports`) |

> The `nfs_folder` variable is commented out in the playbook's vars section — pass it as an extra variable when launching.

### Template: `files/exports.j2`

```
{{ nfs_folder }} 192.168.0.0/24(rw)
```

Adjust the subnet in the template to match your environment.

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/linux/5-rhel_deploy_nfs_server.yml` |
| Credentials | Machine credential |
| Extra Variables | `nfs_folder: /exports` |

---

## 6 — Deploy Container App with Podman

**File:** `playbooks/linux/6-rhel_podman_app_deploy.yml`

Installs Podman, deploys a container application, and opens the required firewall port.

### Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `app_port` | `8080` | Port the application listens on |
| `app_name` | *(required)* | Name to give the Podman container |
| `app_image` | `quay.io/cmedeiro/tasks-app:latest` | Container image to deploy |
| `public_dns_name` | *(required)* | Public DNS or IP shown in the debug output |

### Dependencies

Requires the `containers.podman` collection (included in `requirements.yml`).

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/linux/6-rhel_podman_app_deploy.yml` |
| Credentials | Machine credential |
| Extra Variables | `app_name: my-app`, `public_dns_name: node1.example.com` |

---

## 7 — Deploy PostgreSQL

**File:** `playbooks/linux/7-rhel_deploy_postgresql.yml`

Installs and configures PostgreSQL 16 using the `fedora.linux_system_roles.postgresql` role.

### Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `postgresql_version` | `16` | PostgreSQL major version |
| `postgresql_password` | `CHANGE_ME` | **Required** — set a strong password before running |

### Dependencies

Requires the `fedora.linux_system_roles` collection (included in `requirements.yml`).

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/linux/7-rhel_deploy_postgresql.yml` |
| Credentials | Machine credential |
| Extra Variables | `postgresql_password: "your-secure-password"` |

---

## 8 — Enable SELinux Enforcing (Simple)

**File:** `playbooks/linux/8-rhel_sec_selinux_enforce.yml`

A single-task playbook that sets SELinux to `enforcing` mode with the `targeted` policy. Good for a quick demonstration.

### Variables

None.

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/linux/8-rhel_sec_selinux_enforce.yml` |
| Credentials | Machine credential |

---

## 9 — Full Linux Hardening

**File:** `playbooks/linux/9-rhel_hardening.yml`

Comprehensive hardening playbook covering:

- **Firewalld**: installs, starts, blocks unnecessary services (cockpit, dhcpv6-client), ensures SSH is open
- **SELinux**: enables enforcing mode; reboots if the state changed
- **Kernel parameters**: ASLR, dmesg restriction, perf paranoia
- **Network parameters**: TCP SYN cookies, martian logging, source-route blocking, IP forwarding disabled, ICMP redirect/echo controls
- **Auditd**: installs the audit daemon and deploys `files/audit.rules`

### Variables

None. All settings are defined inline.

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/linux/9-rhel_hardening.yml` |
| Credentials | Machine credential |

---

## 10 — SELinux Configuration with Auditd

**File:** `playbooks/linux/10-rhel_selinux_config.yml`

Similar to playbook 8 but also installs and configures `auditd` with a baseline rules file (`files/audit.rules`). Reboots automatically if SELinux required a state change.

### Variables

None.

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/linux/10-rhel_selinux_config.yml` |
| Credentials | Machine credential |

---

## 11 — RHEL Compliance Report

**File:** `playbooks/linux/11-rhel_check_compliance.yml`

A two-phase playbook that:

1. **Phase 1** — Collects pending package updates and CVE/security advisories from all `target_servers` hosts.
2. **Phase 2** — Installs Apache on a `report` host and publishes an HTML dashboard and per-host detail pages.

The dashboard shows each host's compliance status (Compliant / Updates Pending / Non-Compliant) with clickable links to RHSA and CVE advisories.

### Inventory Groups Required

| Group | Purpose |
|-------|---------|
| `target_servers` | Hosts to audit for updates/CVEs |
| `report` | RHEL host that will serve the HTML report |

### Variables

None — all logic uses inventory group membership.

### Accessing the Report

After the playbook runs, open `http://<report-host-ip>/` in a browser.

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/linux/11-rhel_check_compliance.yml` |
| Credentials | Machine credential |
| Inventory | Must include both `target_servers` and `report` groups |
