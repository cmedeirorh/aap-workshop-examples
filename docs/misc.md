# Miscellaneous Playbooks

These playbooks address cross-platform scenarios or ServiceNow lifecycle management that does not fit neatly into a single OS category.

---

## check_system_updates.yml — Cross-Platform Patch Dashboard

**File:** `playbooks/misc/check_system_updates.yml`

An enhanced version of the RHEL compliance report (`11-rhel_check_compliance.yml`) that supports **both RHEL and Windows** hosts in the same run. It:

1. **Phase 1 — Collect:** Gathers pending package updates and security advisories from all `target_servers`. RHEL hosts use `dnf check-update`, Windows hosts use `win_updates` with `state: searched`.
2. **Phase 2 — Report:** Installs Apache on the `web_server` host and publishes an HTML dashboard and per-host detail pages.

### Inventory Groups Required

| Group | Purpose |
|-------|---------|
| `target_servers` | Mixed RHEL + Windows hosts to audit |
| `web_server` | RHEL host that will serve the HTML report |

### OS Detection

The playbook uses `ansible_os_family` (gathered in Phase 1) to run the correct tasks:
- `RedHat` → uses `dnf check-update` and `dnf updateinfo list cves`
- `Windows` → uses `win_updates` and filters for Security/Critical categories

### Dashboard Features

- Per-host compliance badge: **Compliant** (green) / **Updates Pending** (yellow) / **Non-Compliant** (red)
- Clickable links to RHSA advisories (`access.redhat.com/errata/RHSA-…`)
- Clickable links to CVEs (`cve.org/CVERecord?id=CVE-…`)
- Clickable links to Windows KB articles (`support.microsoft.com/help/…`)
- Drill-down pages per host

### Accessing the Report

After the playbook completes, open `http://<web_server_ip>/` in a browser.

### Variables

None — OS detection and reporting logic is fully automatic.

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/misc/check_system_updates.yml` |
| Credentials | Machine credential (RHEL) + Windows credential |
| Inventory | Must include both `target_servers` and `web_server` groups |

---

## servicenow_schedule_crs.yml — Bulk Schedule Change Requests

**File:** `playbooks/misc/servicenow_schedule_crs.yml`

Finds all ServiceNow Change Requests in `new` state that were created by `ansible` and moves them to `scheduled`. This is used as a **batch approval step** in the patching workflow:

```
1.1 Check → (this playbook bulk-approves) → 1.2 Apply
```

It can also be triggered by an EDA rulebook or run on a schedule in AAP.

### Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `sn_instance` | *(required)* | ServiceNow instance URL |
| `sn_username` | *(required)* | ServiceNow username |
| `sn_password` | *(required)* | ServiceNow password |
| `target_assignment_group` | `Change Management` | Assignment group to set on each scheduled CR |

### Key Behaviour

- Queries ServiceNow for all `new` change requests where `sys_created_by = ansible`.
- Loops over results and moves each to `scheduled`.
- Safe to re-run — only acts on CRs that are still in `new` state.

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/misc/servicenow_schedule_crs.yml` |
| Credentials | ServiceNow credential (extra vars) |
| Schedule | Optional — set a recurring schedule in AAP to auto-approve after a review window |

---

## Full Patching Workflow (End-to-End)

Here is the recommended end-to-end patching workflow using all the related playbooks:

```
Step 1: linux/1.1 or windows/10.1
        → Checks for updates, creates ServiceNow Change Requests (state: new)

Step 2: misc/servicenow_schedule_crs.yml
        → Bulk moves Change Requests to "scheduled"
        (alternatively: manual approval in ServiceNow, or EDA from servicenow_events.yml)

Step 3: linux/1.2 or windows/10.2
        → Applies updates, closes Change Requests
```

Each step can be a node in an **AAP Workflow Job Template**, making the entire process a single click.
