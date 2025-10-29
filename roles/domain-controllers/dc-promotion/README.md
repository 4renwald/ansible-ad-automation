# domain-controllers/dc-promotion

Promotes Windows servers to domain controllers in an existing Active Directory domain.

## Purpose

⚠️ **JOINS EXISTING DOMAINS** - Does NOT create new domains. Use `domain/domain-creation` for new domains.

## Required Variables

```yaml
target_domain_name: "example.local"     # Full FQDN of existing domain
first_dc_ip: "192.168.1.10"            # IP address of first DC
domain_admin_username: "Administrator"  # Domain admin for promotion
domain_admin_password: "SecurePass"     # Domain admin password
dsrm_password: "SecurePass"             # DSRM password
```

## Optional Variables

```yaml
# AD paths (defaults shown)
ad_database_path: "C:\\Windows\\NTDS"
ad_sysvol_path: "C:\\Windows\\SYSVOL"
ad_log_path: "C:\\Windows\\NTDS\\logs"
```

## Usage

```yaml
- name: Promote additional domain controller
  hosts: additional_domain_controllers
  roles:
    - domain-controllers/dc-promotion
```

**Note:** Required variables are typically set in `group_vars/additional_domain_controllers/main.yml` or injected via AAP survey/credentials.

## Behavior

1. Validates all required parameters
2. Promotes server to domain controller
3. Validates promotion and replication

**Note:** Windows features and DNS forwarders must be configured separately (typically handled at playbook level).

## Dependencies

- `microsoft.ad.domain_controller` module (from `microsoft.ad` collection)
- Requires AD-Domain-Services and RSAT Windows features to be installed first
- DNS forwarders should be configured after promotion

## Notes

- **Automatic reboot:** Promotion will trigger a system reboot
- **First DC IP required:** Must point to existing first domain controller
- **Credentials:** Injected via AAP custom credential type in production
- **Standalone role:** Does not include Windows feature installation - orchestrate at playbook level
