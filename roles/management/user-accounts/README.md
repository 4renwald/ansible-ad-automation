````md
# management/user-accounts

Creates and manages Active Directory user accounts.

## Purpose

Generic role for managing AD user accounts after domain deployment. Can be used to create any type of user accounts including service accounts, admin accounts, or regular user accounts.

## Required Variables

```yaml
target_domain_name: "example.local"      # Domain FQDN
ad_user_accounts:                        # List of users to create
  - name: "user1"
    password: "SecurePassword123!"
```

## Optional Variables

### User Account Configuration

```yaml
ad_user_accounts:
  - name: "jdoe"
    upn: "jdoe@example.local"
    sam_account_name: "jdoe"             # Optional, defaults to name
    firstname: "John"
    surname: "Doe"
    description: "IT Administrator"
    password: "TempPassword123!"
    enabled: true
    password_expired: true               # Force password change on first login
    password_never_expires: false
    user_cannot_change_password: false
    email: "jdoe@example.com"
    city: "Montreal"
    company: "Example Corp"
    country: "CA"
    path: "OU=Users,DC=example,DC=local"  # Optional OU placement
    groups:
      - "Domain Users"
      - "IT Staff"
    update_password: "when_changed"       # always|on_create|when_changed
```

### Default Settings

```yaml
default_user_settings:
  password_never_expires: false
  user_cannot_change_password: false
  password_expired: false
  enabled: true
  update_password: "when_changed"
```

## Usage

### Create Regular User Accounts

```yaml
- name: Create user accounts
  hosts: first_domain_controller
  roles:
    - management/user-accounts
  vars:
    ad_user_accounts:
      - name: "jdoe"
        upn: "jdoe@domain.local"
        firstname: "John"
        surname: "Doe"
        password: "TempPass123!"
        password_expired: true
        groups:
          - "Domain Users"
```

### Create Service Accounts

```yaml
- name: Create service accounts
  hosts: first_domain_controller
  roles:
    - management/user-accounts
  vars:
    ad_user_accounts:
      - name: "svc-ansible"
        upn: "svc-ansible@domain.local"
        firstname: "Ansible"
        surname: "Service Account"
        description: "Service account for Ansible automation"
        password: "{{ ansible_service_password }}"  # From AAP credential
        enabled: true
        password_never_expires: true
        user_cannot_change_password: true
        path: "OU=ServiceAccounts,OU=Tier0,DC=domain,DC=local"
        groups:
          - "Domain Admins"
      
      - name: "svc-backup"
        upn: "svc-backup@domain.local"
        firstname: "Backup"
        surname: "Service"
        password: "{{ backup_service_password }}"  # From AAP credential
        password_never_expires: true
        groups:
          - "Backup Operators"
```

### Create Admin Accounts

```yaml
- name: Create admin accounts
  hosts: first_domain_controller
  roles:
    - management/user-accounts
  vars:
    ad_user_accounts:
      - name: "admin.jdoe"
        upn: "admin.jdoe@domain.local"
        firstname: "John"
        surname: "Doe (Admin)"
        password: "{{ admin_password }}"
        password_expired: true
        groups:
          - "Domain Admins"
          - "Enterprise Admins"
```

## Behavior

1. Validates `target_domain_name` and `ad_user_accounts` are defined
2. Creates all user accounts from `ad_user_accounts` list
3. Sets account properties and group memberships
4. Displays creation status for all accounts
5. Idempotent - safe to run multiple times

## AAP Integration

### Using AAP Credentials for Passwords

Store passwords in AAP credentials and pass them as extra variables:

**1. Create AAP Credential (Custom Type)**
```yaml
# Input Configuration
fields:
  - id: ansible_service_password
    type: string
    label: Ansible Service Account Password
    secret: true

# Injector Configuration
extra_vars:
  ansible_service_password: "{{ ansible_service_password }}"
```

**2. Create Job Template**
```yaml
Playbook: playbooks/management/create-service-account.yml
Credentials:
  - Domain Administrator
  - Service Account Passwords (custom credential)
Extra Variables:
  ad_user_accounts:
    - name: "svc-ansible"
      upn: "svc-ansible@{{ target_domain_name }}"
      password: "{{ ansible_service_password }}"
      password_never_expires: true
      groups:
        - "Domain Admins"
```

### Using Surveys for User Creation

Configure AAP survey to collect user details:

**Survey Fields:**
- `user_name`: User account name
- `user_firstname`: First name
- `user_surname`: Last name
- `user_password`: Password (encrypted)
- `user_email`: Email address

**Extra Variables:**
```yaml
ad_user_accounts:
  - name: "{{ user_name }}"
    firstname: "{{ user_firstname }}"
    surname: "{{ user_surname }}"
    upn: "{{ user_name }}@{{ target_domain_name }}"
    password: "{{ user_password }}"
    email: "{{ user_email }}"
    password_expired: true
    groups:
      - "Domain Users"
```

## Dependencies

- `microsoft.ad.user` module
- Domain controller must be operational
- Domain admin credentials required

## Notes

- **Idempotent**: Safe to run multiple times (won't recreate existing accounts)
- **Group Membership**: Uses default group mode which adds to existing groups. Use `groups: { set: [...] }` to replace all groups
- **Password Policy**: Ensure passwords meet domain complexity requirements
- **AAP Credentials**: Preferred method for password management in AAP environments
- **Validation**: Role validates that at least one user account is defined

## Example: Ansible Service Account Setup

For automated Ansible operations, create a service account:

**In inventory group_vars** (`inventory/group_vars/adlab_local_first_dc/service-accounts.yml`):
```yaml
# Ansible service account configuration
# Password should be provided via AAP credential
ansible_service_account_config:
  - name: "svc-ansible"
    upn: "svc-ansible@{{ target_domain_name }}"
    firstname: "Ansible"
    surname: "Service Account"
    description: "Service account for Ansible automation operations"
    password: "{{ ansible_service_password }}"  # Injected from AAP credential
    enabled: true
    password_never_expires: true
    user_cannot_change_password: true
    path: "OU=ServiceAccounts,OU=Tier0,{{ domain_dc_path }}"
    groups:
      - "Domain Admins"
```

**In playbook:**
```yaml
- name: Create Ansible service account
  hosts: adlab_local_first_dc
  roles:
    - management/user-accounts
  vars:
    ad_user_accounts: "{{ ansible_service_account_config }}"
```
````
