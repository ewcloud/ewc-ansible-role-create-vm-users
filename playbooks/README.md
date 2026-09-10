# Create VM local users

Applies the desired list of Linux user accounts to the hosts in the inventory.

## Usage

```bash
# Use the default users file (users.yml in the main repo folder)
ansible-playbook -i inventory/hosts.ini playbooks/create_vm_users.yml

# Use a custom users file
ansible-playbook -i inventory/hosts.ini playbooks/create_vm_users.yml -e users_file=path/to/users.yml

# If the users file is Vault-encrypted
ansible-playbook -i inventory/hosts.ini playbooks/create_vm_users.yml -e users_file=path/to/users.yml --ask-vault-pass
```

## Passing the users file

The playbook reads the account list from a YAML file with a top-level `users:` key.
Point to it with `-e users_file=<path>`; if omitted, `users.yml` beside the playbook is used.

```yaml
# users.yml
users:
  - username: alice
    fullname: Alice Example
    groups: [developers]
    sudo: true
    ssh_keys:
      - "ssh-ed25519 AAAA... alice@laptop"

  - username: carol
    state: absent
```

Vault-encrypt the file if it contains any password hashes:

```bash
ansible-vault encrypt users.yml
```

## Variables

### Per-user fields (`users` list)

| Field         | Required | Description                                                                 |
| ------------- | -------- | --------------------------------------------------------------------------- |
| `username`    | yes      | Login name of the account.                                                  |
| `fullname`    | no       | Human-readable name, stored as the account comment (GECOS).                 |
| `ssh_keys`    | no       | List of SSH **public** keys authorized for the user.                        |
| `groups`      | no       | List of supplementary groups. Groups are created automatically if missing.  |
| `sudo`        | no       | `true` grants sudo via `/etc/sudoers.d/<username>`. Defaults to `false`.     |
| `password`    | no       | Pre-computed password **hash** (crypt `$6$...`, never plaintext; Vault it). Set only on account creation. |
| `remove_home` | no       | When `state: absent`, also delete the home directory.                       |
| `state`       | no       | `present` (default) creates/updates the user; `absent` removes the account. |

### Global defaults

| Variable                     | Default     | Description                                              |
| ---------------------------- | ----------- | -------------------------------------------------------- |
| `user_default_shell`         | `/bin/bash` | Login shell for created accounts.                        |
| `user_create_home`           | `true`      | Create the user's home directory.                        |
| `user_remove_home_on_delete` | `false`     | Also remove the home directory when `state: absent`.     |
| `sudo_nopasswd`              | `true`      | `true` = passwordless sudo (NOPASSWD); `false` = prompt. |
| `user_force_password_change` | `true`      | Expire an initial `password` on creation, forcing a change at first login. |
