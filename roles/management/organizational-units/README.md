# management/organizational-units

Creates Active Directory Organizational Unit structure following Microsoft Tiering Model.

## Purpose

Creates tiered OU structure (Tier 0/1/2) from a Jinja2 template. OUs are created with deletion protection enabled.

## Required Variables

```yaml
target_domain_name: "example.local"  # Full domain FQDN
```

## Optional Variables

```yaml
# Auto-generated from target_domain_name (no manual override needed)
domain_dc_path: "DC=example,DC=local"
```

## Usage

```yaml
- name: Create AD OU structure
  hosts: first_domain_controller
  roles:
    - management/organizational-units
```

**Note:** OU structure is defined in `templates/ous_structure.j2`.

## Behavior

1. Validates `target_domain_name` is defined
2. Auto-constructs DC path from domain name
3. Loads OU structure from template
4. Creates each OU with deletion protection
5. Supports complex domain names (e.g., "sub.domain.example.com")

## Dependencies

- `microsoft.ad.ou` module

## Customization

To modify the OU structure, edit `templates/ous_structure.j2`. See `docs/customizing-ou-structure.md` for guidance.

## Notes

- **OU structure in French:** Default template uses French naming (e.g., "Niveaux")
- **Idempotent:** Safe to run multiple times
- **First DC only:** Should be run on first domain controller after domain creation
