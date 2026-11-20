# Event-Driven Ansible — Playbooks and Rulebooks

Event-Driven Ansible (EDA) connects monitoring systems and event sources directly to Ansible automation. This section covers:

- **EDA response playbooks** (`playbooks/eda/`) — triggered automatically by rulebook conditions
- **Rulebooks** (`rulebooks/`) — define the sources, conditions, and actions

---

## How EDA Works in This Workshop

```
Monitoring tool (Zabbix / Datadog / CrowdStrike / ServiceNow)
        │
        ▼ sends a webhook or event
  EDA Controller (rulebook running)
        │
        ▼ condition matches → action fires
  AAP Controller
        │
        ▼ launches Job Template
  Ansible playbook runs on target hosts
```

---

## EDA Response Playbooks

### change_file.yml

**File:** `playbooks/eda/change_file.yml`

Appends a comment line to a configuration file on the target host. Demonstrates a "config drift remediation" response — when EDA detects a file was changed outside Ansible, this playbook can be triggered to flag the drift.

#### Variables

| Variable | Description |
|----------|-------------|
| `file_path` | Full path to the file to modify (e.g. `/etc/ssh/sshd_config`) |

#### Example Usage

Triggered by a Zabbix or file-watch event. The rulebook passes `file_path` as an extra variable.

---

### network_switch_config.yml

**File:** `playbooks/eda/network_switch_config.yml`

Parses a comma-separated `tags` string from an EDA event payload to identify the target switch hostname and interface index, then enables (un-shuts) the specified GigabitEthernet interface on a Cisco IOS switch.

#### Variables (passed by EDA rulebook)

| Variable | Description |
|----------|-------------|
| `tags` | Comma-separated key:value pairs from the event (e.g. `host:group:device:sw1,interface_index:2`) |

#### What It Does

1. Parses the `tags` string into a dictionary.
2. Extracts the target host and interface index.
3. Runs `cisco.ios.ios_interfaces` to enable the interface.

---

### network_switch_reboot.yml

**File:** `playbooks/eda/network_switch_reboot.yml`

Recovers an unresponsive network device (virtual machine) by checking its KVM/libvirt state and issuing a `virsh start` if the VM is shut off.

#### Variables (passed by EDA rulebook)

| Variable | Description |
|----------|-------------|
| `tags` | Comma-separated key:value pairs including `device_hostname` and `agent_host` |

#### What It Does

1. Parses the `tags` string.
2. Identifies the VM name and hypervisor from the event tags.
3. Checks the VM state with `virsh domstate`.
4. Starts the VM if it is in `shut off` state.

---

### servicenow_create_incident.yml

**File:** `playbooks/eda/servicenow_create_incident.yml`

Creates a ServiceNow **Incident** when an EDA event fires. Collects the last 20 log lines from the Cisco switch that triggered the event and includes them in the incident description.

#### Variables

| Variable | Description |
|----------|-------------|
| `tags` | Comma-separated key:value pairs including `source_host` |
| `sn_instance` | ServiceNow instance URL |
| `sn_username` | ServiceNow username |
| `sn_password` | ServiceNow password |

---

## Rulebooks

Rulebooks are deployed in AAP's **EDA Controller**. Each rulebook defines one or more event sources and a set of rules.

### debug.yml

**File:** `rulebooks/debug.yml`

A simple catch-all rulebook for testing. Listens on a webhook (port 35002) and prints every event payload to the EDA log.

**Use:** Point any webhook at this to inspect the payload structure before writing a real rule.

```yaml
source:  ansible.eda.webhook (port 35002)
trigger: any event with a payload
action:  debug (print payload)
```

---

### datadog_events.yml

**File:** `rulebooks/datadog_events.yml`

Listens for Datadog webhook alerts and routes them to AAP Job Templates:

| Condition | Triggered Job Template |
|-----------|----------------------|
| App is DOWN alert | `RHEL Deploy APP With Podman` |
| SNMP device down | `EDA - Restart Switch` |
| SNMP interface down | `EDA - Network Config` |
| Config change log alert | `EDA - Report Incident to ServiceNow` |

**Setup:** Configure a Datadog Webhook integration pointing at `http://<eda-controller>:35001`.

---

### zabbix_events.yml

**File:** `rulebooks/zabbix_events.yml`

Listens for Zabbix alerts and routes them based on `event_tags.service`:

| Condition | Triggered Job Template |
|-----------|----------------------|
| `selinux_security` tag | `EDA - Linux SELinux Config` |
| `ssh_security` tag | `EDA - Linux SSH Hardening` |
| `url_checker` tag | `EDA - Linux Deploy APP With Podman` |
| Any other Zabbix trigger | `EDA - Report Incident to ServiceNow` |

**Setup:** Configure Zabbix to send webhook notifications to `http://<eda-controller>:35001` with the event tags populated.

---

### servicenow_events.yml

**File:** `rulebooks/servicenow_events.yml`

Listens for ServiceNow webhook events. When a Change Request transitions to `change_request_scheduled`, it launches the RHEL update playbook targeting the host named in the Change Request title.

| Condition | Triggered Job Template |
|-----------|----------------------|
| `event_type == "change_request_scheduled"` | `RHEL ServiceNow - Apply Updates` |

**Setup:** Configure a ServiceNow Business Rule or Flow to POST to `http://<eda-controller>:35001` when a Change Request is scheduled.

---

### crowdstrike_events.yml

**File:** `rulebooks/crowdstrike_events.yml`

Streams detection events from the CrowdStrike Falcon Event Stream and prints them. Demonstrates the `crowdstrike.falcon.eventstream` source plugin.

#### Variables

| Variable | Description |
|----------|-------------|
| `FALCON_CLIENT_ID` | CrowdStrike Falcon API client ID |
| `FALCON_CLIENT_SECRET` | CrowdStrike Falcon API client secret |

**Note:** Store these as EDA Controller credentials, not plain text.

---

### test.yml

**File:** `rulebooks/test.yml`

A minimal test rulebook. Any webhook event triggers the `RHEL Sync SSH Keys` Job Template. Use for verifying that the EDA → AAP connection works before building real rules.

---

## EDA Setup Checklist

1. Install the `ansible.eda` collection on the EDA Controller.
2. Create an **AAP Token** in AAP and configure it in the EDA Controller settings.
3. Add each rulebook as an **EDA Rulebook Activation**.
4. Configure monitoring tools to send webhooks to the EDA Controller's listening port.
5. Create matching **Job Templates** in AAP with the exact names referenced in the rulebooks.
