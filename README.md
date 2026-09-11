# ewc-ansible-role-create-vm-users

An Ansible role to declaratively manage Linux user accounts on target hosts. It creates,
updates, and removes users defined in a simple list, and can also:

- Set each user's full name (GECOS/comment)
- Install one or more SSH public keys (SSH-key access is the primary method)
- Optionally set an initial password (Vault-encrypted), expired to force a change at first login
- Assign one or more supplementary groups (auto-created if missing)
- Optionally grant `sudo` access via a validated `/etc/sudoers.d` drop-in
- Remove accounts when they are marked `absent`

The role uses only `ansible.builtin` modules, so **no external collections need to be
installed**.

## Role layout

```
ewc-ansible-role-create-vm-users/
├── defaults/main.yml     # tunable defaults (shell, home, sudo mode)
├── tasks/main.yml        # the logic
└── templates/sudoers.j2  # sudoers.d drop-in template
```

## Installation

This repository contains only the role. Install it as a dependency of an Ansible project
(a repo that holds your inventory and playbooks) via a `requirements.yml` file:

```yaml
# requirements.yml
roles:
  - name: ewc-ansible-role-create-vm-users
    src: https://github.com/ewcloud/ewc-ansible-role-create-vm-users.git
    scm: git
    version: 0.0.1   # or pin to another released tag
```

Then install it:

```bash
ansible-galaxy install -r requirements.yml
```

Alternatively, add it as a Git submodule or vendor it under your project's `roles/` directory.

## Usage

Reference the role from a playbook in your project and provide the `users` list (for example
in `group_vars/all.yml` or `host_vars`):

```yaml
# create_vm_users.yml (in your playbook repo)
- name: Manage VM users
  hosts: all
  become: true
  roles:
    - role: ewc-ansible-role-create-vm-users
```

## Variables

### `users` list

The `users:` list is the source of truth for which accounts exist on the targeted hosts.
Each entry supports these fields:

| Field         | Required | Description                                                                 |
| ------------- | -------- | --------------------------------------------------------------------------- |
| `username`    | yes      | Login name of the account.                                                  |
| `uid`         | no       | Numeric UID for the account. Fails if already used by another account.      |
| `fullname`    | no       | Human-readable name, stored as the account comment (GECOS).                 |
| `ssh_keys`    | no       | List of SSH **public** keys authorized for the user.                        |
| `groups`      | no       | List of supplementary groups. Groups are created automatically if missing.  |
| `sudo`        | no       | `true` grants sudo via `/etc/sudoers.d/<username>`. Defaults to `false`.    |
| `password`    | no       | Pre-computed password **hash** (crypt `$6$...`, not plaintext; **store with Ansible Vault**). Set only on account creation. Expired on first login by default. |
| `remove_home` | no       | When `state: absent`, also delete the home directory. Overrides `user_remove_home_on_delete`. |
| `state`       | no       | `present` (default) creates/updates the user; `absent` removes the account. |

Example:

```yaml
users:
  # A developer with sudo and one SSH key
  - username: alice
    fullname: Alice Example
    groups:
      - developers
    sudo: true
    ssh_keys:
      - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...replace-me alice@laptop"

  # A user in two groups, no sudo, multiple keys
  - username: bob
    fullname: Bob Example
    groups:
      - developers
      - deploy
    ssh_keys:
      - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...key-one bob@laptop"
      - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...key-two bob@desktop"

  # Remove a previously-created account
  - username: carol
    state: absent
    # remove_home: true   # optional: also delete the home directory
```

Important:
- Use the **public** key (the contents of `id_ed25519.pub`), never a private key.
- `authorized_keys` is managed when one or more `ssh_keys` are listed — the role writes exactly
  those keys and overwrites any added manually on the host. **Note:** if you remove all keys
  (empty `ssh_keys` or drop the field), existing keys are **not** revoked — the current
  `authorized_keys` file is left as-is. To revoke access, delete the keys on the host or set the
  account to `state: absent`.
- Removing a user from the list does **not** delete the account. To delete an account, set
  `state: absent` (keep the entry). Once removed from the hosts you may delete the entry.
- By default a removed account keeps its home directory. Set `remove_home: true` on the entry
  (or `user_remove_home_on_delete: true` globally) to also delete it.
- `groups` is applied **exactly** (not appended): each run makes a user a member of only the
  listed supplementary groups; any supplementary group not listed is removed from that user.

### Group GIDs (optional)

Set fixed GIDs for the groups this role creates via an optional top-level `group_gids` map
(`name: gid`). Creation fails if a GID is already used by a different group.

```yaml
group_gids:
  developers: 3000
  deploy: 3001
```

### Passwords (optional)

Access is primarily SSH-key based, but you can set an **initial** password per user via the
`password` field. Behavior:

- The value must be a **pre-computed crypt hash** (e.g. `$6$...` for sha512), never plaintext.
  It is written to the account with `update_password: on_create` — set **only when the account
  is first created** and never overriding a password the user later changes themselves.
- With `user_force_password_change: true` (default), the password is expired on creation
  (`chage -d 0`), so the user must choose a new password at first login.
- The account-management task uses `no_log: true`, so the hash is never printed.

Generate a hash locally (no extra Python libraries required):

```bash
openssl passwd -6 'ChangeMe123!'      # or: mkpasswd -m sha-512
```

**Store the hash with Ansible Vault** — don't commit it in cleartext. You can generate and
encrypt in one step:

```bash
ansible-vault encrypt_string "$(openssl passwd -6 'ChangeMe123!')" --name password
```

Paste the resulting `!vault` block under the user, then run the playbook with the vault
password available:

```bash
ansible-playbook create_vm_users.yml --ask-vault-pass
```

### Defaults

You can tune behavior by overriding the variables in [`defaults/main.yml`](defaults/main.yml):

| Variable                     | Default     | Description                                              |
| ---------------------------- | ----------- | -------------------------------------------------------- |
| `users`                      | `[]`        | List of accounts to manage (see above).                 |
| `user_default_shell`         | `/bin/bash` | Login shell for created accounts.                        |
| `user_create_home`           | `true`      | Create the user's home directory.                        |
| `user_remove_home_on_delete` | `false`     | Also remove the home directory when `state: absent`.     |
| `sudo_nopasswd`              | `true`      | `true` = passwordless sudo (NOPASSWD); `false` = prompt. |
| `user_force_password_change` | `true`      | Expire an initial `password` on creation, forcing a change at first login. |

> **Sudo note:** `sudo_nopasswd` defaults to `true`, so users with `sudo: true` get passwordless
> (`NOPASSWD:ALL`) root. This is convenient for key-based automation; if your policy requires a
> password prompt for sudo, set `sudo_nopasswd: false` (each sudo user must then have a password).

## Requirements

- Ansible installed on the control machine (`pipx install ansible` or `brew install ansible`).
- SSH access from the control machine to each host as the connection user, with sudo/root there.
- No Ansible collections required — only `ansible.builtin` modules are used.

## Verify

On a target host:

```bash
id alice                       # user, uid, groups
getent group developers        # group membership
sudo ls -l /home/alice/.ssh/authorized_keys
sudo -l -U alice               # effective sudo rights
```

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md).
