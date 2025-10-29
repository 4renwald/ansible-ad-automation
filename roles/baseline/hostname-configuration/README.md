# baseline/hostname-configuration

Configures the Windows server hostname based on inventory hostname.

## Purpose

Baseline role that sets the Windows hostname to match the inventory hostname (short name only). This is typically the first configuration step before domain operations.

## Required Variables

None - the role uses `inventory_hostname` to automatically extract the short hostname.

## Optional Variables

```yaml
# Override the automatic hostname extraction
hostname: "custom-hostname"

# Reboot settings (defaults shown)
reboot_timeout: 300         # Reboot timeout (seconds)
post_reboot_delay: 30       # Wait time after reboot (seconds)
```

## Usage

```yaml
- name: Configure server hostname
  ansible.builtin.include_role:
    name: baseline/hostname-configuration
```

### With custom hostname

```yaml
- name: Configure server hostname
  ansible.builtin.include_role:
    name: baseline/hostname-configuration
  vars:
    hostname: "dc01"
```

## Behavior

1. Validates hostname parameter:
   - Must be defined and non-empty
   - Must be 15 characters or less (Windows NetBIOS limit)

2. Displays current and target hostname for verification

3. Changes the Windows hostname using `ansible.windows.win_hostname`

4. Reboots the server if required (automatic reboot handling)

5. Displays success message after hostname change

## Dependencies

- `ansible.windows.win_hostname` module
- `ansible.windows.win_reboot` module

## Notes

- **Automatic hostname extraction**: By default, extracts the short hostname from `inventory_hostname` using `split('.')[0]`
  - Example: `dc01.adlab.local` → `dc01`
- **Automatic reboot**: Hostname changes require a reboot, which is handled automatically
- **Idempotent**: Safe to run multiple times (no change if hostname already matches)
- **Windows NetBIOS limit**: Hostnames are limited to 15 characters on Windows

## Examples

### Using inventory hostname (automatic)

For inventory host `dc01.adlab.local`, the role will automatically set hostname to `dc01`:

```yaml
- name: Configure hostname from inventory
  ansible.builtin.include_role:
    name: baseline/hostname-configuration
```

### Using explicit hostname

```yaml
- name: Configure specific hostname
  ansible.builtin.include_role:
    name: baseline/hostname-configuration
  vars:
    hostname: "fileserver01"
```
