````markdown
# baseline/dns-client-configuration

Configures DNS client settings on network adapters for Windows servers.

## Purpose

Configures DNS server addresses on network interface cards. This is run early in server setup, before domain operations, to ensure proper DNS resolution for Active Directory.

## Required Variables

```yaml
dns_servers:
  - "192.168.1.10"    # Primary DNS
  - "192.168.1.11"    # Secondary DNS

target_domain_nic: "*"  # Network adapter name (use "*" for all adapters)
```

## Optional Variables

```yaml
validate_dns_config: true           # Validate DNS after configuration
dns_query_timeout: 5                # DNS query timeout (seconds)
dns_validation_retries: 3           # Retry attempts for validation
dns_validation_delay: 5             # Delay between retries (seconds)
```

## Usage

### First Domain Controller

```yaml
- name: Configure DNS client for first DC
  hosts: first_domain_controller
  roles:
    - baseline/dns-client-configuration
```

**Note:** First DC uses `dns_servers: ["127.0.0.1", ...]` for self-reference (set in group_vars).

### Additional Domain Controller

```yaml
- name: Configure DNS client for additional DC
  hosts: additional_domain_controllers
  roles:
    - baseline/dns-client-configuration
```

**Note:** Additional DCs use `dns_servers: ["<first_dc_ip>", ...]` (set in group_vars).

## Behavior

1. Validates `dns_servers` and `target_domain_nic` are defined
2. Displays DNS configuration to be applied
3. Configures DNS servers on specified network adapter(s)
4. Optionally validates DNS configuration (if `validate_dns_config: true`)
5. Displays configuration status

## Tags

- `dns-client` - DNS client configuration tasks
- `dns` - DNS-related operations

Use in AAP: `--tags dns-client` or `--tags dns`

## Behavior

1. Validates required parameters
2. Configures DNS client settings on network adapter(s)
3. Validates DNS configuration was applied (if enabled)
4. Tests DNS resolution to verify connectivity

## Dependencies

- `ansible.windows.win_dns_client` module

## Notes

- **DNS client configuration only:** Does NOT configure DNS server-side settings (forwarders, zones)


```yaml
# group_vars/first_domain_controller/main.yml
dns_servers:
  - "127.0.0.1"
  - "{{ dns_forwarders[0] }}"
  - "{{ dns_forwarders[1] }}"

# group_vars/additional_domain_controllers/main.yml
dns_servers:
  - "{{ first_dc_ip }}"
  - "{{ dns_forwarders[0] }}"
  - "{{ dns_forwarders[1] }}"

# group_vars/domain_controllers/main.yml
dns_forwarders:
  - "10.64.28.20"
  - "10.68.28.20"
  - "10.30.0.2"
```

## Error Handling

The role includes comprehensive error handling with contextual troubleshooting:

- Validates network adapter exists
- Checks DNS server address validity
- Provides specific guidance for common failures
- Supports retry logic for transient network issues

## Dependencies

- Windows target host
- `ansible.windows.win_dns_client` module
- Network connectivity to specified DNS servers
