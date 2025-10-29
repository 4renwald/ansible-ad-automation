# Ansible AD Automation

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Ansible](https://img.shields.io/badge/ansible-core%202.12%2B-red.svg)
![Platform](https://img.shields.io/badge/platform-windows-lightgrey.svg)
![Language](https://img.shields.io/badge/language-yaml-yellow.svg)

Active Directory automation using Ansible with support for Ansible Automation Platform (AAP) and AWX.

Automates Active Directory domain deployment and management including baseline server configuration, domain creation, domain controller promotion, organizational unit management, and health monitoring.

## ✨ Key Features

- Complete AD lifecycle management: Baseline configuration → Domain creation → DC promotion → OU management → Health monitoring
- Multi-domain support: Hierarchical inventory structure for managing multiple AD forests and domains
- AAP/AWX integration: Custom credential types, survey variables, and tag-based workflow orchestration
- Security: Ansible Vault integration for all sensitive data with reference pattern
- Error handling: Block/rescue patterns with detailed error context in every role
- Validation: Post-operation validation tasks verify successful configuration
- Modular design: Lifecycle-based role organization (baseline → domain → management → monitoring)
- Zero role dependencies: All orchestration happens at playbook level

## 📋 Table of Contents

- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Quick Start](#quick-start)
- [Usage](#usage)
  - [Creating a New Domain](#creating-a-new-domain)
  - [Promoting Additional Domain Controllers](#promoting-additional-domain-controllers)
  - [Managing Organizational Units](#managing-organizational-units)
  - [Health Checks](#health-checks)
- [Inventory Structure](#inventory-structure)
- [Available Playbooks](#available-playbooks)
- [Role Categories](#role-categories)
- [AAP/AWX Integration](#aapaawx-integration)
- [Development](#development)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)
- [Contact](#contact)

## 🏗️ Architecture

### Design Philosophy

Roles are organized by deployment phase rather than technology:
- **baseline/**: Initial server configuration (hostname, DNS, NTP)
- **domain/**: Domain and forest creation
- **domain-controllers/**: DC promotion and DNS management
- **management/**: OUs, users, service accounts
- **monitoring/**: Health checks and validation
- **shared/**: Reusable components (Windows features)

Roles do not include other roles. All orchestration happens in playbooks using `ansible.builtin.include_role`, enabling flexible composition and AAP/AWX workflow integration.

First DC vs. replica DC separation with multi-domain support:
```
domain_controllers/
  adlab_local/                    # Domain-level configuration
    adlab_local_first_dc/         # Creates new forest
    adlab_local_additional_dcs/   # Join existing domain
```

Variable precedence: `group_vars/all/` → `group_vars/domain_name/` → `group_vars/domain_name_dc_group/` → host vars

## 📦 Prerequisites

### Control Node (Ansible)
- **Ansible Core**: 2.12 or higher
- **Python**: 3.8 or higher
- **Operating System**: Linux (tested on Debian/Ubuntu, RHEL/CentOS)

### Managed Nodes (Windows Servers)
- **Windows Server**: 2016 or higher (2019/2022 recommended)
- **PowerShell**: 5.1 or higher
- **WinRM**: Configured for HTTPS with CredSSP authentication
- **Network**: Static IP address configured
- **Resources**: Minimum 2GB RAM, 60GB disk space

### Required Ansible Collections
- `ansible.windows` (>= 3.2.0)
- `microsoft.ad` (>= 1.9.2)
- `community.general` (>= 11.2.1)

## 🚀 Installation

### 1. Clone the Repository

```bash
git clone https://github.com/4renwald/ansible-ad-automation.git
cd ansible-ad-automation
```

### 2. Install Required Collections

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

### 3. Configure Windows Hosts for WinRM

On each Windows server, run this PowerShell script as Administrator:

```powershell
# Enable WinRM with CredSSP authentication
Enable-PSRemoting -Force
Enable-WSManCredSSP -Role Server -Force

# Configure WinRM for Ansible
winrm quickconfig -quiet
winrm set winrm/config/service/auth '@{Basic="true"}'
winrm set winrm/config/service/auth '@{CredSSP="true"}'
winrm set winrm/config/service '@{AllowUnencrypted="true"}'
```

### 4. Set Up Inventory

Copy and customize the inventory for your environment:

```bash
cp inventory/hosts.yml inventory/hosts-production.yml
# Edit inventory/hosts-production.yml with your server details
```

### 5. Configure Ansible Vault

Create vault files for sensitive data:

```bash
# Create vault password file (keep this secure!)
echo "your-vault-password" > ~/.ansible-vault-pass

# Create encrypted vault for domain credentials
ansible-vault create inventory/group_vars/adlab_local_first_dc/vault
```

Example vault content:
```yaml
---
vault_dsrm_password: "YourSecurePassword123!"
vault_domain_admin_password: "YourAdminPassword123!"
```

## ⚙️ Configuration

### Global Configuration (All Hosts)

Edit `inventory/group_vars/all/main.yml`:

```yaml
---
# NTP Configuration
ntp_servers:
  - "time.windows.com"
  - "time.nist.gov"
  - "pool.ntp.org"

ntp_sync_interval: 3600
validate_ntp_config: true
```

### Domain Configuration

Edit `inventory/group_vars/adlab_local/main.yml`:

```yaml
---
target_domain_name: adlab.local
```

### Domain Controller Configuration

Edit `inventory/group_vars/domain_controllers/main.yml`:

```yaml
---
# Network Configuration
target_domain_nic: "*"
domain_netbios_name: "{{ target_domain_name.split('.')[0] | upper }}"

# Active Directory Paths
ad_database_path: "C:\\Windows\\NTDS"
ad_sysvol_path: "C:\\Windows\\SYSVOL"
ad_log_path: "C:\\Windows\\NTDS\\logs"

# DNS Forwarders
dns_forwarders:
  - "1.1.1.1"
  - "1.1.1.3"
```

### First DC Configuration

Edit `inventory/group_vars/adlab_local_first_dc/main.yml`:

```yaml
---
# Domain/Forest Functional Levels
domain_mode: "WinThreshold"  # Windows Server 2016+
forest_mode: "WinThreshold"

# DNS Settings
install_dns: true
create_dns_delegation: false
create_new_forest: true

# DNS Configuration for First DC (self-reference)
dns_servers:
  - "127.0.0.1"
  - "1.1.1.1"
  - "1.1.1.3"
```

## 🚦 Quick Start

### Create Your First Domain

```bash
# 1. Install collections
ansible-galaxy collection install -r collections/requirements.yml

# 2. Configure inventory and vault files (see Configuration section)

# 3. Create new domain and forest
ansible-playbook -i inventory/hosts.yml playbooks/domain/create-domain.yml \
  --limit adlab_local_first_dc \
  --ask-vault-pass

# 4. Verify domain creation
ansible-playbook -i inventory/hosts.yml playbooks/validation/health-check.yml \
  --limit adlab_local_first_dc \
  --ask-vault-pass
```

Expected duration: 15-20 minutes (includes automatic reboot and service stabilization)

## 📖 Usage

### Creating a New Domain

Creates a new Active Directory forest and domain on the first domain controller:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/domain/create-domain.yml \
  --limit adlab_local_first_dc \
  --ask-vault-pass
```

What it does:
1. Installs Windows features (AD-Domain-Services, DNS, RSAT tools)
2. Creates AD domain and forest with specified functional levels
3. Configures DNS forwarders for external resolution
4. Automatically reboots and validates services

Post-creation tasks:
```bash
# Create organizational unit structure
ansible-playbook -i inventory/hosts.yml playbooks/management/create-ous.yml \
  --limit adlab_local_first_dc \
  --ask-vault-pass

# Create service accounts
ansible-playbook -i inventory/hosts.yml playbooks/management/create-service-account.yml \
  --limit adlab_local_first_dc \
  --ask-vault-pass
```

### Promoting Additional Domain Controllers

Promotes a server to replica domain controller in an existing domain:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/domain-controllers/promote-dc.yml \
  --limit adlab_local_additional_dcs \
  --extra-vars "first_dc_ip=192.168.1.46" \
  --ask-vault-pass
```

Pass `first_dc_ip` as extra var pointing to the first DC's IP address.

DNS Configuration for Replica DCs:
Edit `inventory/group_vars/adlab_local_additional_dcs/main.yml`:
```yaml
dns_servers:
  - "192.168.1.46"  # First DC IP
  - "1.1.1.1"       # External forwarder 1
  - "1.1.1.3"       # External forwarder 2
```

### Managing Organizational Units

Creates tiered OU structure (Tier 0/1/2):

```bash
ansible-playbook -i inventory/hosts.yml playbooks/management/create-ous.yml \
  --limit adlab_local_first_dc \
  --ask-vault-pass
```

Edit `roles/management/organizational-units/templates/ous_structure.j2` to define your OU hierarchy.

### Health Checks

Validates domain controller health and services:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/validation/health-check.yml \
  --limit domain_controllers \
  --ask-vault-pass
```

Validates:
- AD services (NTDS, DNS, Netlogon, W32Time)
- Domain connectivity
- DNS resolution
- NTP synchronization

### Baseline Server Configuration

Configure hostname, DNS, and NTP on servers before domain operations:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/baseline/configure-server.yml \
  --limit adlab_local_first_dc \
  --ask-vault-pass
```

## 📂 Inventory Structure

```yaml
all:
  vars:
    ansible_connection: psrp
    ansible_psrp_auth: credssp
    ansible_psrp_cert_validation: ignore
  children:
    domain_controllers:
      children:
        adlab_local:                        # Domain: adlab.local
          children:
            adlab_local_first_dc:           # First DC (creates forest)
              hosts:
                dc01.adlab.local:
                  ansible_host: 192.168.1.46
            
            adlab_local_additional_dcs:     # Replica DCs (join domain)
              hosts:
                dc02.adlab.local:
                  ansible_host: 192.168.1.48
```

### Group Variables Hierarchy

```
group_vars/
├── all/
│   └── main.yml                    # Global settings (NTP, validation)
├── domain_controllers/
│   └── main.yml                    # DC-specific settings (AD paths, DNS)
├── adlab_local/
│   └── main.yml                    # Domain-specific (target_domain_name)
├── adlab_local_first_dc/
│   ├── main.yml                    # First DC settings (forest mode, DNS)
│   └── vault                       # Encrypted secrets (DSRM password)
└── adlab_local_additional_dcs/
    ├── main.yml                    # Replica DC settings (DNS points to first DC)
    └── vault                       # Encrypted secrets
```

## 📚 Available Playbooks

### Baseline Configuration
| Playbook | Purpose | Tags |
|----------|---------|------|
| `baseline/configure-hostname.yml` | Set server hostname | `hostname` |
| `baseline/configure-server.yml` | Complete baseline (hostname, DNS, NTP) | `hostname`, `dns-client`, `ntp` |

### Domain Operations
| Playbook | Purpose | Tags |
|----------|---------|------|
| `domain/create-domain.yml` | Create new AD forest and domain | `windows-features`, `domain-creation`, `dns-forwarders` |

### Domain Controller Operations
| Playbook | Purpose | Tags |
|----------|---------|------|
| `domain-controllers/promote-dc.yml` | Promote server to replica DC | `windows-features`, `dc-promotion`, `dns-forwarders` |

### Management Operations
| Playbook | Purpose | Tags |
|----------|---------|------|
| `management/create-ous.yml` | Create OU structure | `ous` |
| `management/create-service-account.yml` | Create service accounts | `service-accounts` |

### Validation Operations
| Playbook | Purpose | Tags |
|----------|---------|------|
| `validation/health-check.yml` | Validate DC health and services | `health-check` |

### Using Tags

Run specific portions of playbooks using tags:

```bash
# Only configure DNS, skip other baseline tasks
ansible-playbook -i inventory/hosts.yml playbooks/baseline/configure-server.yml \
  --tags dns-client \
  --ask-vault-pass

# Run multiple tagged tasks
ansible-playbook -i inventory/hosts.yml playbooks/domain/create-domain.yml \
  --tags windows-features,domain-creation \
  --ask-vault-pass
```

## 🔧 Role Categories

### baseline/
- **dns-client-configuration**: Configure DNS client settings and validation
- **hostname-configuration**: Set server hostname
- **local-admin-password-reset**: Reset local administrator password
- **ntp-configuration**: Configure Windows Time Service with NTP servers

### domain/
- **domain-creation**: Create new AD forest and domain (first DC only)

### domain-controllers/
- **dc-promotion**: Promote server to replica domain controller
- **dns-forwarders**: Configure DNS forwarders on domain controllers

### management/
- **organizational-units**: Create tiered OU structure from template
- **user-accounts**: Manage AD user accounts (including service accounts)

### monitoring/
- **health-checks**: Validate DC services and domain connectivity

### shared/
- **windows-features**: Install/remove Windows features and roles

## 🔌 AAP/AWX Integration

Integration with Ansible Automation Platform (AAP) and AWX.

### Custom Credential Types

Create a custom credential type for domain administrator credentials:

**Inputs Configuration**:
```yaml
fields:
  - id: target_domain_admin_user
    type: string
    label: Domain Admin Username
  - id: target_domain_admin_pass
    type: string
    label: Domain Admin Password
    secret: true
required:
  - target_domain_admin_user
  - target_domain_admin_pass
```

**Injector Configuration**:
```yaml
extra_vars:
  target_domain_admin_user: '{{ target_domain_admin_user }}'
  target_domain_admin_pass: '{{ target_domain_admin_pass }}'
```

### Survey Variables

Use job template surveys for runtime parameters:

**Example: DC Promotion Survey**
- **Variable Name**: `first_dc_ip`
- **Answer Variable**: `first_dc_ip`
- **Required**: Yes
- **Default**: `192.168.1.46`

### Workflow Templates

Create workflows for complex operations:

1. **Deploy New Domain**:
   - Job 1: Configure baseline (hostname, DNS, NTP)
   - Job 2: Create domain and forest
   - Job 3: Create OU structure
   - Job 4: Create service accounts
   - Job 5: Validate domain health

2. **Add Domain Controller**:
   - Job 1: Configure baseline
   - Job 2: Promote to DC (with `first_dc_ip` survey)
   - Job 3: Configure DNS forwarders
   - Job 4: Validate DC health

### Tag-Based Execution

All playbooks support tag-based execution in AAP/AWX workflows.

## 💻 Development

### Code Standards

- **FQCN Required**: Always use fully qualified collection names (e.g., `ansible.windows.win_feature`)
- **Block/Rescue Pattern**: Every role task file uses block/rescue error handling
- **Validation Tasks**: Include post-operation validation in every role
- **Task Naming**: Start with action verb, capitalize first letter, no trailing periods
- **State Explicitness**: Always specify `state:` parameter

### Branch Strategy

```bash
# Always work on develop branch
git checkout develop
git pull origin develop

# Create feature branch
git checkout -b feature/your-feature-name

# Make changes and commit
git add .
git commit -m "feat: add new feature"

# Push and create PR to develop
git push origin feature/your-feature-name
```

### Linting

```bash
# Run ansible-lint
ansible-lint playbooks/

# Run yamllint
yamllint .

# Syntax check
ansible-playbook --syntax-check playbooks/domain/create-domain.yml
```

### Adding New Roles

1. Create under appropriate category: `baseline/`, `domain/`, `domain-controllers/`, `management/`, `monitoring/`, or `shared/`
2. Add comprehensive `README.md` with variables table and usage examples
3. Include `defaults/main.yml` with commented default values
4. Use block/rescue pattern in `tasks/main.yml`
5. Add validation tasks for verifying operations
6. Create playbook in `playbooks/<category>/` that orchestrates the role
7. Document any new `group_vars` requirements
8. Update `.github/copilot-instructions.md` with new patterns

### Role Structure Template

```
roles/<category>/<role-name>/
├── README.md                # Purpose, variables, dependencies, behavior
├── defaults/
│   └── main.yml            # Default values with comments
├── tasks/
│   ├── main.yml           # Entry point with block/rescue
│   ├── <operation>.yml    # Specific operation tasks
│   └── validate-<op>.yml  # Validation tasks
└── templates/             # Jinja2 templates (if needed)
    └── config.j2
```

## 🧪 Testing

### Syntax Validation

```bash
# Check playbook syntax
ansible-playbook --syntax-check playbooks/domain/create-domain.yml

# Validate inventory
ansible-inventory -i inventory/hosts.yml --list
```

### Dry Run

```bash
# Check mode (no changes made)
ansible-playbook -i inventory/hosts.yml playbooks/domain/create-domain.yml \
  --check \
  --diff \
  --ask-vault-pass
```

### Verify Inventory and Variables

```bash
# List all hosts
ansible-inventory -i inventory/hosts.yml --graph

# Show variables for specific host
ansible-inventory -i inventory/hosts.yml --host dc01.adlab.local

# Test connectivity
ansible -i inventory/hosts.yml all -m ansible.windows.win_ping --ask-vault-pass
```

## 🔍 Troubleshooting

### Common Issues

#### WinRM Connection Failures

Symptom: `Connection timeout` or `Connection refused` errors

Solution:
```bash
# Test WinRM from control node
curl -u "administrator:password" --ntlm -k https://target-server:5986/wsman

# Verify CredSSP on Windows server
Get-WSManCredSSP
Enable-WSManCredSSP -Role Server -Force
```

#### DNS Configuration Issues

Symptom: DC promotion fails with DNS errors

Solution:
- First DC: Ensure DNS servers include `127.0.0.1` (self-reference)
- Replica DC: Ensure DNS servers include first DC IP address
- Verify DNS forwarders are reachable: `nslookup google.com 1.1.1.1`

#### Service State Validation Failures

Symptom: Service validation tasks fail

Note: `ansible.windows.win_service_info` returns `state: 'started'`, NOT `'running'`

Solution: Roles already use correct `state == 'started'` comparison

#### Vault Password Issues

Symptom: `Decryption failed` errors

Solution:
```bash
# Create vault password file
echo "your-vault-password" > ~/.ansible-vault-pass
chmod 600 ~/.ansible-vault-pass

# Use in playbooks
ansible-playbook ... --vault-password-file ~/.ansible-vault-pass

# Or use prompt
ansible-playbook ... --ask-vault-pass
```

### Debug Mode

Run playbooks with verbose output:

```bash
# Basic verbosity
ansible-playbook -v ...

# More verbosity (connection details)
ansible-playbook -vv ...

# Maximum verbosity (debug)
ansible-playbook -vvv ...
```

### Getting Help

1. **Check role README**: Each role has comprehensive documentation
2. **Review copilot-instructions.md**: `.github/copilot-instructions.md` contains architectural patterns
3. **Examine validation tasks**: Look at `validate-*.yml` tasks to understand expected state
4. **Enable verbose logging**: Use `-vvv` flag to see detailed execution logs

## 📄 License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

```
MIT License

Copyright (c) 2025 4renwald

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
```

## 📧 Contact

Maintainer: 4renwald

Repository: [https://github.com/4renwald/ansible-ad-automation](https://github.com/4renwald/ansible-ad-automation)

Issues: [https://github.com/4renwald/ansible-ad-automation/issues](https://github.com/4renwald/ansible-ad-automation/issues)

---
