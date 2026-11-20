# Windows Playbooks

All Windows playbooks target the `windows` inventory group. They use the `ansible.windows` collection for WinRM-based communication.

**Prerequisites:**
- WinRM must be configured and reachable on target Windows hosts.
- An AAP **Windows credential** (WinRM username/password) must be attached to every Job Template.

---

## 10.1 — Check Windows Updates and Create ServiceNow Change Request

**File:** `playbooks/windows/10.1-windows_patching_change_request.yml`

Scans Windows hosts for available updates and creates a **new** ServiceNow Change Request for each host that needs patching — but only if a Change Request for that host does not already exist.

### Variables

| Variable | Description |
|----------|-------------|
| `sn_instance` | ServiceNow instance URL |
| `sn_username` | ServiceNow username |
| `sn_password` | ServiceNow password |

### Workflow Position

Run this **before** `10.2-windows_patching.yml`. The expected workflow is:

```
10.1 Scan → (review / approve in ServiceNow) → 10.2 Patch
```

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/windows/10.1-windows_patching_change_request.yml` |
| Credentials | Windows credential + ServiceNow credential (extra vars) |
| Inventory | `windows` group |

---

## 10.2 — Apply Windows Updates

**File:** `playbooks/windows/10.2-windows_patching.yml`

Applies all pending Windows Updates across all categories (Security, Critical, Service Packs, etc.), reboots automatically if required, then closes the matching ServiceNow Change Request.

### Variables

| Variable | Description |
|----------|-------------|
| `sn_instance` | ServiceNow instance URL |
| `sn_username` | ServiceNow username |
| `sn_password` | ServiceNow password |

### Key Behaviour

- Retries the update task up to **3 times** to handle transient failures.
- Reboots with an 800-second timeout to accommodate slow Windows restarts.
- Moves the Change Request through `scheduled` → `implement` → `review` → `closed` automatically.

### Prerequisites

A matching ServiceNow Change Request in `new` state must exist (created by `10.1`).

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/windows/10.2-windows_patching.yml` |
| Credentials | Windows credential + ServiceNow credential (extra vars) |
| Inventory | `windows` group |

---

## 11 — Create a Windows User

**File:** `playbooks/windows/11-windows_create_user.yml`

Creates a local Windows user account with the specified name, password, full name, description, and group membership.

### Variables

| Variable | Description |
|----------|-------------|
| `user_name` | Username for the new account |
| `user_password` | Password for the new account |
| `user_fullname` | Display name |
| `user_description` | Account description |
| `user_groups` | List of local groups to add the user to (e.g. `Administrators`) |

### Example Extra Variables

```yaml
user_name: jdoe
user_password: "P@ssw0rd!"
user_fullname: "Jane Doe"
user_description: "Workshop student account"
user_groups:
  - Users
```

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/windows/11-windows_create_user.yml` |
| Credentials | Windows credential |
| Extra Variables | See variables table above |

---

## 12 — Run a PowerShell Script

**File:** `playbooks/windows/12-windows_run_powershell.yml`

Executes an arbitrary PowerShell script on Windows hosts and prints the output. This is a quick way to demonstrate AAP's ability to run PowerShell without needing a Windows workstation.

### Variables

| Variable | Description |
|----------|-------------|
| `ps_script` | PowerShell code to execute (multi-line supported via YAML block scalar) |

### Example Extra Variables

```yaml
ps_script: |
  Get-ComputerInfo | Select-Object CsName, OsName, WindowsVersion
  Write-Output "Script executed by Ansible!"
```

### Sample Script

A sample PowerShell script is available at `files/powershell_test.ps1`.

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/windows/12-windows_run_powershell.yml` |
| Credentials | Windows credential |
| Extra Variables | `ps_script: "Get-Date"` |

---

## 13 — Deploy IIS Web Server

**File:** `playbooks/windows/13-windows_deploy_web_iis.yml`

Installs the IIS Web-Server Windows feature, starts the `W3Svc` service, deploys a custom `index.html`, and reboots if the feature install requires it.

### Variables

| Variable | Description |
|----------|-------------|
| `iis_test_message` | HTML content to write to `C:\Inetpub\wwwroot\index.html` |

### Example Extra Variables

```yaml
iis_test_message: "<h1>Hello from Ansible!</h1><p>IIS deployed by AAP.</p>"
```

### Verifying the Result

After the playbook completes, the debug task prints the URL (`http://<ansible_host>`). Open it in a browser to confirm IIS is serving your page.

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/windows/13-windows_deploy_web_iis.yml` |
| Credentials | Windows credential |
| Extra Variables | `iis_test_message: "<h1>Hello!</h1>"` |

---

## 14 — Deploy Dynatrace OneAgent

**File:** `playbooks/windows/14-windows_deploy_dynatrace_agent.yml`

Downloads and installs the Dynatrace OneAgent on Windows hosts using a PaaS token from your Dynatrace environment.

### Variables

| Variable | Description |
|----------|-------------|
| `oneagent_download_path` | Local path on the Windows host for the installer (e.g. `C:\Temp\Dynatrace`) |
| `oneagent_installer_path` | Full path to the downloaded `.exe` installer |
| `oneagent_installer_url` | Dynatrace download URL for the Windows installer |
| `oneagent_paas_token` | Dynatrace PaaS token for authenticated download |
| `oneagent_install_args` | List of installer arguments (e.g. `--set-host-group`, `--set-app-log-content-access`) |

### Example Extra Variables

```yaml
oneagent_download_path: "C:\\Temp\\Dynatrace"
oneagent_installer_path: "C:\\Temp\\Dynatrace\\oneagent.exe"
oneagent_installer_url: "https://<your-env>.live.dynatrace.com/api/v1/deployment/installer/agent/windows/default/latest"
oneagent_paas_token: "dt0c01.XXXXXXXX"
oneagent_install_args:
  - "--set-host-group=workshop"
  - "--set-app-log-content-access=true"
```

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/windows/14-windows_deploy_dynatrace_agent.yml` |
| Credentials | Windows credential |
| Extra Variables | See variables table above |
