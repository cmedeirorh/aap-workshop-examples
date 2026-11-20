# Cloud Playbooks

These playbooks manage **AWS infrastructure** using the `amazon.aws` collection. They run against `localhost` (the AAP controller acts as the AWS client) using an AWS credential.

**Prerequisites:**
- An AAP **Amazon Web Services** credential with `Access Key` and `Secret Key` configured.
- The target VPC, subnet, and security group must already exist in the target region.

---

## Create EC2 Instances

**File:** `playbooks/cloud/create_ec2_instances.yml`

Provisions two EC2 instances (`node1` and `node2`) in an existing VPC, attaching them to a specified subnet and security group. Instances are tagged with `Environment: Dev`, `ManagedBy: Ansible`, and the `owner` variable.

### Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `aws_region` | `us-east-2` | AWS region to deploy into |
| `vpc_name` | `aws-test-vpc` | Name tag of the existing VPC |
| `subnet_name` | `aws-test-subnet` | Name tag of the existing subnet |
| `security_group_name` | `aws-test-sg` | Name of the existing security group |
| `ssh_key_name` | `aws-test-key` | EC2 key pair name for SSH access |
| `ami_id` | `ami-0ff44cc29626b706c` | AMI ID (RHEL 9 in us-east-2 — change for other regions) |
| `instance_type` | `t3.small` | EC2 instance type |
| `owner` | *(required)* | Unique owner tag; also used as the `uuid` tag |

### Key Behaviour

- Looks up the VPC, subnet, and security group **by name tag** rather than hard-coding IDs — this keeps the playbook portable across environments.
- Uses `exact_count: 1` so re-running is idempotent (it will not create duplicates).
- Assigns a public IP to each instance.

### Example Extra Variables

```yaml
owner: student01
aws_region: us-east-2
```

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/cloud/create_ec2_instances.yml` |
| Credentials | AWS credential |
| Extra Variables | `owner: student01` |

---

## Modify EC2 Security Group

**File:** `playbooks/cloud/modify_ec2_sg.yml`

Updates the inbound rules of an existing EC2 Security Group to open ports 22, 80, 443, and 8080 from anywhere (`0.0.0.0/0`). Useful for demonstrating how Ansible can manage cloud firewall rules.

### Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `aws_region` | `us-east-2` | AWS region |
| `vpc_name` | `ansible-demo` | Name tag of the VPC containing the security group |
| `sg_name` | `ansible-demo` | Name of the security group to modify |
| `sg_description` | `AWS demo SG for the demo` | Description for the security group |

### Key Behaviour

- Purges all existing rules first (workaround for policy restrictions that prevent appending), then adds the desired rules.
- Fails fast if the named VPC cannot be found.

### Example Extra Variables

```yaml
vpc_name: my-workshop-vpc
sg_name: my-workshop-sg
```

### AAP Job Template

| Setting | Value |
|---------|-------|
| Playbook | `playbooks/cloud/modify_ec2_sg.yml` |
| Credentials | AWS credential |
| Extra Variables | Override vpc/sg names as needed |
