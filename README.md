# Ansible Active Directory Join Role

This repository joins Fedora and Debian Linux servers to Windows Active Directory using:

- realmd
- sssd
- adcli
- oddjob-mkhomedir
- sudoers.d AD group permissions

## Inventory

Edit:

```text
inventories/prod.yml
```

Each host should have:

```yaml
ad_domain: corp.example.com
```

The host must also be placed into the matching domain group:

```yaml
domain_corp_example_com:
  hosts:
    my-host:
```

## Vault files

Encrypt each vault file:

```bash
ansible-vault encrypt group_vars/all/vault.yml
ansible-vault encrypt group_vars/fedora/vault.yml
ansible-vault encrypt group_vars/debian/vault.yml
ansible-vault encrypt group_vars/domain_corp_example_com/vault.yml
ansible-vault encrypt group_vars/domain_lab_example_com/vault.yml
```

Edit encrypted files:

```bash
ansible-vault edit group_vars/domain_corp_example_com/vault.yml
```

## Run syntax check

```bash
ansible-playbook playbooks/join-ad.yml --syntax-check
```

## Run playbook

```bash
ansible-playbook playbooks/join-ad.yml --ask-vault-pass
```

Or with a vault password file:

```bash
ansible-playbook playbooks/join-ad.yml --vault-password-file .vault-pass
```

## Rejoining a different domain

By default, this repo will fail if the host is already joined to a different domain.

To allow rejoining, set this in:

```text
group_vars/all/common.yml
```

```yaml
ad_allow_rejoin_to_new_domain: true
```

## Sudoers permissions

Domain-specific files define this list:

```yaml
ad_sudo_permissions:
  - name: INF_ADMIN
    ad_group: "INF_ADMIN@corp.example.com"
    sudoers_file: "10-ad-inf-admin"
    runas: "ALL"
    commands: "ALL"
    nopasswd: false
```

The role creates one file per entry under:

```text
/etc/sudoers.d/
```

Each sudoers file is validated with:

```bash
visudo -cf
```

## Rollback behavior

If the AD join workflow fails, the playbook rescue block runs `ad_rollback`.

Rollback can:

- leave the requested AD domain
- restore backed up files
- remove managed sudoers files
- restart SSSD

Controlled by:

```yaml
ad_rollback_leave_domain_on_failure: true
ad_rollback_restore_backups_on_failure: true
```

## Recommended first run

Start with one host:

```bash
ansible-playbook playbooks/join-ad.yml \
  --limit debian-web-01 \
  --ask-vault-pass
```

Then verify on the target host:

```bash
realm list
id test.linux@corp.example.com
getent group INF_READ@corp.example.com
sudo -l -U test.linux@corp.example.com
```

## Notes for production hardening

For production use, consider adding:

```text
roles/ad_preflight/
```

With checks for:

- DNS resolution for AD domain controllers
- NTP/time sync drift
- Kerberos SRV records
- firewall access to LDAP, LDAPS, Kerberos, DNS, SMB
- existing `/etc/krb5.conf` conflicts
- duplicate computer object handling
- OU existence validation
- join account permission validation
