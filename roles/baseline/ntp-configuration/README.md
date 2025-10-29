# baseline/ntp-configuration

Configures Windows Time Service (W32Time) with NTP servers for time synchronization.

## Purpose

Configures W32Time service to synchronize with external NTP servers. Run early in server setup, before domain operations.

## Required Variables

```yaml
ntp_servers:
  - "time.windows.com"
  - "pool.ntp.org"

ntp_server_type: "NTP"  # Options: "NTP", "NT5DS" (domain hierarchy), "AllSync"
```

## Optional Variables

```yaml
# Sync settings (defaults shown)
ntp_sync_interval: 3600                    # Sync interval (seconds)
ntp_special_poll_interval: true            # Enable faster initial sync
ntp_max_pos_phase_correction: 54000        # Max time correction (seconds)
ntp_max_neg_phase_correction: 54000        # Max time correction (seconds)
ntp_update_interval: 100                   # Update interval (100ns units)
ntp_announce_flags: 5                      # Time server flags (0/5/10)
ntp_enable_client: true                    # Enable NTP client

# Validation settings
validate_ntp_config: true                  # Validate after configuration
ntp_validation_timeout: 30                 # Validation timeout (seconds)
ntp_validation_retries: 3                  # Retry attempts
ntp_validation_delay: 10                   # Delay between retries (seconds)
```

## Usage

```yaml
- name: Configure NTP
  hosts: domain_controllers
  roles:
    - baseline/ntp-configuration
```

**Note:** NTP servers are typically defined in `group_vars/all/02-ntp-config.yml` for centralized management.

## Behavior

1. Validates required parameters
2. Stops Windows Time service
3. Configures NTP servers and W32Time registry settings
4. Starts and enables W32Time service
5. Forces immediate time synchronization
6. Validates time sync (if enabled)
7. Tests NTP server connectivity

## Dependencies

- `ansible.windows.win_service` module
- `ansible.windows.win_powershell` module
- `ansible.windows.win_command` module

## Notes

- **NTP server type:** Use `"NTP"` for standalone servers or first DC, `"NT5DS"` for domain member DCs

```

### Corporate Environment with Internal NTP Servers

```yaml
- name: Configure NTP with corporate time servers
  hosts: all_servers
  vars:
    ntp_servers:
      - "ntp1.corp.example.com"
      - "ntp2.corp.example.com"
      - "ntp3.corp.example.com"
    ntp_sync_interval: 900  # Sync every 15 minutes
  roles:
    - common/server-baseline/ntp-config
```

## What It Does

1. **Validates Parameters**: Ensures `ntp_servers` list is provided and valid
2. **Stops W32Time Service**: Temporarily stops the service for configuration
3. **Configures NTP Servers**: Sets the NTP server list using `w32tm /config`
4. **Configures Registry Settings**: Sets W32Time parameters in Windows registry:
   - Server type (NTP, NT5DS, etc.)
   - Announce flags
   - NTP client enabled/disabled
   - Sync intervals
   - Maximum phase corrections
5. **Starts W32Time Service**: Starts and enables the service for auto-start
6. **Forces Synchronization**: Initiates immediate time sync with `w32tm /resync`
7. **Validates Configuration**:
   - Verifies service is running
   - Checks NTP configuration
   - Tests connectivity to NTP servers
   - Displays sync status

## What It Does NOT Do

- Configure firewall rules for NTP (UDP port 123)
- Modify domain time hierarchy settings (for domain-joined machines)
- Set system time zone
- Configure time source priority beyond the list order
- Handle certificate-based time authentication

## Example Group Variables

Define NTP servers in `group_vars/` for different host groups:

```yaml
# group_vars/all/ntp-config.yml
ntp_servers:
  - "time.windows.com"
  - "time.nist.gov"
  - "pool.ntp.org"

ntp_sync_interval: 3600

# group_vars/first_domain_controller/ntp.yml
ntp_server_type: "NTP"
ntp_announce_flags: 5  # Reliable time server

# group_vars/additional_domain_controllers/ntp.yml
ntp_servers:
  - "{{ first_dc_fqdn }}"
ntp_server_type: "NT5DS"  # Use domain hierarchy
```

## Validation

The role includes comprehensive validation that:
- Verifies Windows Time service is running and set to auto-start
- Checks current NTP configuration
- Displays time synchronization status
- Tests connectivity to all configured NTP servers
- Shows last sync time and stratum level

## Troubleshooting

### Time Not Synchronizing

```bash
# Check W32Time service status
Get-Service W32Time

# View current configuration
w32tm /query /configuration

# Check sync status
w32tm /query /status

# View peers
w32tm /query /peers

# Force resync
w32tm /resync /force

# Test connectivity to NTP server
w32tm /stripchart /computer:time.windows.com /samples:5
```

### Common Issues

1. **Firewall Blocking NTP**: Ensure UDP port 123 is allowed outbound
2. **Domain Conflict**: Domain members automatically use domain hierarchy
3. **Large Time Difference**: May require manual time correction first
4. **NTP Servers Unreachable**: Check network connectivity and DNS resolution
5. **Service Won't Start**: Check Windows Event Log for W32Time errors

### Windows Event Logs

Monitor these logs for time service issues:
- **Application and Services Logs** → **Microsoft** → **Windows** → **Time-Service** → **Operational**

### Manual Commands

```powershell
# View detailed status
w32tm /query /status /verbose

# Debug W32Time
w32tm /debug /enable /file:C:\temp\w32time.log /size:10485760 /entries:0-300

# Unregister and re-register service (if corrupted)
w32tm /unregister
w32tm /register

# Restart service
Restart-Service W32Time
```

## Error Handling

The role includes comprehensive error handling with contextual troubleshooting:
- Validates NTP server list is provided
- Attempts to recover W32Time service if configuration fails
- Provides detailed error messages with specific troubleshooting steps
- Supports retry logic for transient network issues

## Dependencies

- Windows target host
- Windows Time Service (W32Time) installed
- Network connectivity to NTP servers (UDP port 123)
- Administrator privileges

## Performance Notes

- Initial time synchronization may take 5-15 minutes
- Subsequent syncs occur at configured interval (default: 1 hour)
- Large time differences may require multiple sync cycles
- Stratum level indicates distance from reference clock (lower is better)

## Security Considerations

- NTP traffic is unencrypted (UDP port 123)
- Consider using internal trusted NTP servers
- For high-security environments, consider NTP authentication
- Domain controllers should use reliable external sources
- Monitor for time drift as a security indicator

## Best Practices

1. **Use Multiple NTP Sources**: Configure at least 3 NTP servers for redundancy
2. **Use Reliable Sources**: Choose well-maintained public or corporate NTP servers
3. **Match Network Topology**: Use geographically close NTP servers
4. **Monitor Time Drift**: Set up alerts for significant time differences
5. **First DC Authoritative**: Make first DC the authoritative time source
6. **Document Time Sources**: Keep track of NTP server addresses and ownership
