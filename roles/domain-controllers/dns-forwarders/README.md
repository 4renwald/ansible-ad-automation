# domain-controllers/dns-forwarders

Configures DNS forwarders on Active Directory domain controllers for external DNS resolution.

## Purpose

Configures DNS forwarder settings on domain controllers. Used by:
- First DC creation workflow (after `domain/domain-creation`)
- Additional DC promotion workflow (after `domain-controllers/dc-promotion`)

## Required Variables

```yaml
dns_forwarders:
  - "1.1.1.1"      # First DNS forwarder
  - "1.1.1.3"      # Second DNS forwarder
```

**Note:** Typically defined in `group_vars/domain_controllers/main.yml` for centralized management across all DCs.

## Usage

```yaml
- name: Configure DNS forwarders
  ansible.builtin.include_role:
    name: domain-controllers/dns-forwarders
```

## Behavior

1. Validates `dns_forwarders` list is defined and non-empty
2. Clears existing DNS forwarders
3. Adds configured DNS forwarders
4. Displays configuration results

## Dependencies

- `ansible.windows.win_powershell` module
- DNS Server PowerShell module (installed with AD-Domain-Services)
