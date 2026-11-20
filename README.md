# Ansible Automation Platform — Workshop Examples

A curated collection of playbooks, rulebooks, and supporting files used in hands-on **Ansible Automation Platform (AAP)** workshops. Each example is designed to teach a real-world automation pattern that students can run directly against lab environments.

> **Note:** This content is for **educational purposes**. Some playbooks intentionally skip production hardening (e.g., `validate_certs: false`) to keep the focus on the core automation concept being taught.

---

## Repository Layout

```
aap-workshop-examples/
├── ansible.cfg                  # Shared Ansible configuration
├── collections/
│   └── requirements.yml         # All required Ansible collections
├── inventory/
│   └── hosts                    # Sample static inventory
├── files/                       # Templates and static files used by playbooks
│   ├── audit.rules
│   ├── exports.j2
│   ├── iis_index.html
│   ├── index.html
│   ├── issue.j2
│   ├── motd
│   ├── powershell_test.ps1
│   ├── sshd-banner.j2
│   └── zabbix_agent2.j2
├── playbooks/
│   ├── linux/                   # RHEL / Linux automation
│   ├── windows/                 # Windows Server automation
│   ├── cloud/                   # AWS EC2 provisioning
│   ├── eda/                     # Event-Driven Ansible response playbooks
│   ├── network/                 # Cisco IOS network automation
│   └── misc/                    # Cross-platform utilities
└── rulebooks/                   # EDA rulebooks (sources + rules)
```

---

## Quick Start

### 1. Install required collections

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

### 2. Configure credentials

Most playbooks require credentials passed as **extra variables** or stored as **AAP credentials**. See each playbook's documentation for the variables it expects.

### 3. Run a playbook (command line)

```bash
ansible-playbook playbooks/linux/1-rhel_simple_patching.yml -i inventory/hosts
```

### 4. Run via AAP

Import this repository as a **Project** in AAP, then create a **Job Template** pointing at the playbook you want to run.

---

## Playbook Categories

| Category | Topic | Doc |
|----------|-------|-----|
| Linux | RHEL patching, hardening, monitoring, apps | [docs/linux.md](docs/linux.md) |
| Windows | Patching, users, IIS, Dynatrace, PowerShell | [docs/windows.md](docs/windows.md) |
| Cloud | EC2 provisioning, security groups | [docs/cloud.md](docs/cloud.md) |
| EDA | Event-driven response playbooks | [docs/eda.md](docs/eda.md) |
| Network | Cisco IOS configuration | [docs/network.md](docs/network.md) |
| Misc | Cross-platform patch dashboard, ServiceNow utilities | [docs/misc.md](docs/misc.md) |

---

## Required Collections

Install all dependencies in one command:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

| Collection | Used By |
|------------|---------|
| `amazon.aws` | Cloud playbooks |
| `ansible.eda` | EDA rulebooks |
| `ansible.netcommon` | Network playbooks |
| `ansible.posix` | Linux playbooks (SELinux, firewalld, authorized_key) |
| `ansible.windows` | Windows playbooks |
| `cisco.ios` | Network playbooks |
| `containers.podman` | `6-rhel_podman_app_deploy.yml` |
| `fedora.linux_system_roles` | `7-rhel_deploy_postgresql.yml` |
| `redhat.insights` | `2-rhel_post_install.yml` |
| `servicenow.itsm` | Patching + EDA playbooks that integrate with ServiceNow |

---

## ServiceNow Integration

Several playbooks integrate with **ServiceNow ITSM**. They expect three variables:

| Variable | Description |
|----------|-------------|
| `sn_instance` | Your ServiceNow instance URL (e.g. `https://dev12345.service-now.com`) |
| `sn_username` | ServiceNow username |
| `sn_password` | ServiceNow password |

Store these as a **ServiceNow credential** in AAP and attach it to any Job Template that uses them.

---

## EDA Rulebooks

Event-Driven Ansible (EDA) rulebooks in `rulebooks/` listen on webhook sources or monitoring platforms and automatically trigger AAP Job Templates in response. See [docs/eda.md](docs/eda.md) for setup details.

---

## Contributing

Pull requests are welcome. Keep playbooks educational — prioritize clarity over brevity, and document any non-obvious design decision inline.

---

## Resources

- [Ansible Automation Platform Docs](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform)
- [Ansible Galaxy](https://galaxy.ansible.com/)
- [Event-Driven Ansible Docs](https://www.redhat.com/en/technologies/management/ansible/event-driven-ansible)
- [Ansible Workshops (upstream)](https://github.com/ansible/workshops/)
