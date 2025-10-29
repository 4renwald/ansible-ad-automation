# baseline/local-admin-password-reset

Resets the local Administrator account password on Windows servers.

## Purpose

Resets the local Administrator account password to a known value and ensures the account is enabled. This is typically run as part of server baseline configuration before domain operations, and is **opt-in only** (tagged with `never` - must be explicitly requested).

## Required Variables

```yaml
target_domain_admin_pass: "SecurePassword123!"  # Local Administrator password
```

## Usage

```yaml
- name: Reset local Administrator password
  hosts: servers
  roles:
    - baseline/local-admin-password-reset
```

## Behavior

1. Validates that `target_domain_admin_pass` is defined and non-empty
2. Resets local Administrator account password
3. Ensures Administrator account is enabled
4. Sets password to never expire

## Tags

- `local-admin-password` - Local Administrator password reset tasks
- `never` - Must be explicitly requested with `--tags local-admin-password`

Use in AAP: `--tags local-admin-password`

## Dependencies

- `ansible.windows.win_user` module
- Administrator privileges on target host

## Notes

- **Opt-in only:** Role is tagged with `never` and must be explicitly requested
- **Pre-domain operation:** Typically run before domain creation or DC promotion
- **Credentials:** `target_domain_admin_pass` should be injected via AAP custom credential type or Ansible Vault in production
- **Idempotent:** Safe to run multiple times
