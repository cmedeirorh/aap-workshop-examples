# Network Playbooks

These playbooks configure **Cisco IOS** switches using the `cisco.ios` collection over SSH. They are typically triggered automatically via EDA rulebooks when monitoring alerts fire.

**Prerequisites:**
- An AAP **Network credential** (SSH username/password or key) for the Cisco devices.
- Cisco IOS devices must be reachable from the AAP execution environment over SSH.
- The `cisco.ios` and `ansible.netcommon` collections must be installed.

---

## cisco_sw_config.yml — Full Switch Configuration

**File:** `playbooks/network/cisco_sw_config.yml`

Applies a comprehensive baseline configuration to a Cisco IOS switch:

- Sets the hostname and domain name
- Configures a MOTD banner
- Creates a local admin user with privilege level 15
- Configures SNMPv2c (community string, location, contact)
- Enforces SSHv2 on VTY lines
- Enables specified GigabitEthernet interfaces
- Saves running config to startup config

### Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `scope` | *(from EDA)* | Comma-separated `key:value` pairs containing `snmp_device` to identify the target switch |
| `new_admin_user` | `netadmin` | Username for the local admin account |
| `new_admin_pass` | *(empty)* | Password for the admin account — **set this before running** |
| `domain_name` | `example.com` | DNS domain name |
| `snmp_community_string` | `watch-ro` | SNMPv2c read-only community string |
| `snmp_location` | `lab1` | SNMP system location |
| `snmp_contact` | `admin@example.com` | SNMP system contact |
| `motd` | `Add your text here` | MOTD banner text |
| `interfaces_to_enable` | GE1–GE4 | List of interfaces to enable with descriptions |

### How It Identifies the Target Switch

When triggered from EDA, the `scope` variable is a comma-separated string like:

```
snmp_device:sw1.example.com,other_key:value
```

The playbook's first play (on `localhost`) parses this string into a dictionary and extracts `snmp_device` as the target host for the second play.

When running manually, set the `scope` variable or directly target the switch in your inventory.

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/network/cisco_sw_config.yml` |
| Credentials | Network credential (Cisco IOS) |
| Extra Variables | `scope: "snmp_device:sw1.example.com"` |

---

## cisco_config_change_alert.yml — Unauthorized Config Change Incident

**File:** `playbooks/network/cisco_config_change_alert.yml`

An automated incident response playbook. When EDA detects a config change alert (e.g., from Datadog SNMP monitoring), this playbook:

1. Parses the EDA `scope` variable to identify the switch.
2. Runs `show archive log config all` on the switch to gather the config history.
3. Creates a **ServiceNow Incident** with the last 20 log lines as evidence.

### Variables

| Variable | Description |
|----------|-------------|
| `scope` | Comma-separated `key:value` pairs containing `snmp_device` |
| `sn_instance` | ServiceNow instance URL |
| `sn_username` | ServiceNow username |
| `sn_password` | ServiceNow password |

### Typical Trigger

This playbook is launched automatically by the `datadog_events.yml` rulebook when it detects a `config_change` alert tag from Datadog.

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/network/cisco_config_change_alert.yml` |
| Credentials | Network credential + ServiceNow credential (extra vars) |
| Extra Variables | `scope: "snmp_device:sw1.example.com"` |

---

## Notes for Students

- **Idempotency:** `cisco.ios` modules are generally idempotent — re-running will not make duplicate changes.
- **`save_when: modified`**: The `ios_config` task at the end of `cisco_sw_config.yml` only saves the running config if changes were made, avoiding unnecessary writes to flash.
- **EDA integration:** Both playbooks are designed to be EDA-triggered. The `scope` parsing pattern (parse on localhost, then target the extracted host) is a common EDA → network automation design pattern worth understanding.
