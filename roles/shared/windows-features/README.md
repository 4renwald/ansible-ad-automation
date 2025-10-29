# shared/windows-features

Installs Windows Server features with automatic reboot handling.

## Purpose

Shared role that installs a configurable list of Windows features. Used by:
- `domain/domain-creation` - Installs AD-DS, DNS, and RSAT features
- `domain-controllers/dc-promotion` - Installs AD-DS and RSAT features

## Required Variables

```yaml
windows_features_to_install:
  - AD-Domain-Services
  - DNS
  - RSAT-AD-Tools
```

## Optional Variables

```yaml
# Reboot settings (defaults shown)
reboot_after_features: true        # Reboot if features require it
reboot_timeout: 600                # Reboot timeout (seconds)
reboot_shutdown_timeout: 60        # Shutdown timeout (seconds)
```

## Usage

```yaml
- name: Install Windows features
  ansible.builtin.include_role:
    name: shared/windows-features
  vars:
    windows_features_to_install:
      - AD-Domain-Services
      - DNS
      - RSAT-AD-Tools
      - RSAT-AD-PowerShell
```

## Behavior

1. Validates feature list is provided and non-empty
2. Installs each feature with sub-features and management tools
3. Checks if any feature requires reboot
4. Reboots server if needed (and `reboot_after_features` is true)
5. Displays detailed installation status

## Dependencies

- `ansible.windows.win_feature` module
- `ansible.windows.win_reboot` module

## Notes

- **Idempotent:** Safe to run multiple times (skips already installed features)
- **Automatic reboot:** Handles reboot requirements automatically
