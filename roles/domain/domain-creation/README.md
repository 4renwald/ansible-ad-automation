# domain/domain-creation

Creates a new Active Directory domain and forest on the first domain controller.

## Purpose

⚠️ **CREATES NEW DOMAINS** - Establishes a new AD forest and root domain. Use `domain-controllers/dc-promotion` to add replica DCs to an existing domain.

This role creates the first domain controller in a new AD forest with configurable domain/forest functional levels, database paths, and DNS settings.

## Required Variables

```yaml
target_domain_name: "example.local"     # Full FQDN of the new domain
domain_netbios_name: "EXAMPLE"          # NetBIOS name (max 15 chars)
dsrm_password: "SecurePassword123!"     # Directory Services Restore Mode password (min 8 chars)
```

## Optional Variables

```yaml
# AD paths (defaults shown)
ad_database_path: "C:\\Windows\\NTDS"
ad_sysvol_path: "C:\\Windows\\SYSVOL"

# Functional levels (defaults shown)
domain_mode: "WinThreshold"             # Windows Server 2016+
forest_mode: "WinThreshold"             # Windows Server 2016+

# DNS settings (defaults shown)
install_dns: true                       # Install DNS server role
create_dns_delegation: false            # Create DNS delegation
```

**Valid functional levels:** `Win2008`, `Win2008R2`, `Win2012`, `Win2012R2`, `WinThreshold`

## Usage

```yaml
- name: Create new Active Directory domain
  hosts: first_domain_controller
  roles:
    - domain/domain-creation
```

**Note:** Windows features (AD-Domain-Services, DNS) must be installed separately via `shared/windows-features` role before running this role. DNS forwarders should be configured after domain creation using `domain-controllers/dns-forwarders`.

### Typical Inventory Structure

```yaml
# inventory/group_vars/adlab_local/main.yml
target_domain_name: "adlab.local"

# inventory/group_vars/domain_controllers/main.yml
domain_netbios_name: "{{ target_domain_name.split('.')[0] | upper }}"
ad_database_path: "C:\\Windows\\NTDS"
ad_sysvol_path: "C:\\Windows\\SYSVOL"

# inventory/group_vars/adlab_local_first_dc/main.yml
domain_mode: "WinThreshold"
forest_mode: "WinThreshold"
install_dns: true
create_dns_delegation: false

# Vault file or AAP credential (sensitive)
dsrm_password: "SecurePassword123!"
```

## Behavior

1. **Validates all parameters** - Ensures all 9 required/optional variables are defined and properly formatted:
   - Domain name and NetBIOS name (max 15 chars)
   - DSRM password (min 8 chars)
   - AD paths (valid Windows paths)
   - Functional levels (valid values)
   - DNS settings (boolean values)

2. **Creates AD domain and forest** - Uses `microsoft.ad.domain` module with all parameters from inventory variables

3. **Triggers automatic reboot** - Domain creation requires system restart to complete

4. **Waits for stabilization** - 5-minute pause after reboot for AD services to fully initialize

5. **Validates domain services** - Checks critical services are running:
   - NTDS (Active Directory Domain Services)
   - DNS (DNS Server)
   - Netlogon (Net Logon)
   - W32Time (Windows Time)

6. **Tests domain connectivity** - Validates domain configuration via PowerShell `Get-ADDomain`

## Dependencies

- `microsoft.ad.domain` module (from `microsoft.ad` collection)
- Requires AD-Domain-Services and DNS Windows features to be installed first (use `shared/windows-features` role)
- DNS forwarders should be configured after domain creation (use `domain-controllers/dns-forwarders` role)

## Notes

- **Automatic reboot:** Domain creation will trigger a system reboot
- **First DC only:** This role creates the forest. Use `domain-controllers/dc-promotion` for additional DCs
- **Credentials:** `dsrm_password` should be injected via AAP custom credential type or Ansible Vault in production
- **Standalone role:** Does NOT include Windows feature installation - orchestrate at playbook level
- **No hardcoded values:** All parameters come from inventory variables for flexibility across environments
