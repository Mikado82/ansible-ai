# Role `sudo_rules`

Grants sudo rights per user or group, with or without password. For every
entry you decide which commands may be run (with or without fixed arguments),
as which user/group, which files may be edited via `sudoedit`, and whether a
password is required.

## What it does

1. Validates all entries (argument spec plus checks for absolute paths, single
   lines and unique names), so nothing malformed reaches the sudoers files.
2. Installs `sudo` if it is missing (`sudo_rules_install`).
3. Checks that `/etc/sudoers` includes `/etc/sudoers.d` (`@includedir` or
   `#includedir`) and fails otherwise.
4. Writes one drop-in file per entry to `/etc/sudoers.d/90-sudo-rules-<user>`
   (mode `0440`). Each file is checked with `visudo -cf` before it is put in
   place, so a broken file can never lock you out of sudo.
5. Removes the files of entries with `state: absent` and, with
   `sudo_rules_purge: true`, every other file with the prefix that is no
   longer listed.

## Usage

```bash
ansible-playbook -i inventory.ini playbooks/sudo-rules.yml
```

Run it from the repository root, so `ansible.cfg` points Ansible at `roles/`.

```yaml
sudo_rules_list:
  - user: deploy
    comment: Deployment user, may only manage nginx
    defaults:
      - env_keep += "HOME"
    rules:
      - commands:
          - /usr/bin/systemctl restart nginx   # exactly these arguments
          - /usr/bin/journalctl                # any arguments
        nopasswd: true                         # no password prompt
      - edit_files:
          - /etc/nginx/nginx.conf
      - commands: /usr/bin/apt                 # password required (default)
      - commands: /usr/bin/psql
        run_as: postgres
        nopasswd: true
  - user: "%admins"
    rules:
      - commands: ALL
        run_as: ALL
        run_as_group: ALL
  - user: olduser
    state: absent
```

results in `/etc/sudoers.d/90-sudo-rules-deploy`:

```
# Ansible managed
# sudo rights for deploy
# Deployment user, may only manage nginx
Defaults:deploy env_keep += "HOME"
deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx, \
    /usr/bin/journalctl
deploy ALL=(root) PASSWD: sudoedit /etc/nginx/nginx.conf
deploy ALL=(root) PASSWD: /usr/bin/apt
deploy ALL=(postgres) NOPASSWD: /usr/bin/psql
```

and `/etc/sudoers.d/90-sudo-rules-group-admins`:

```
%admins ALL=(ALL:ALL) PASSWD: ALL
```

## Variables

| Variable | Default | Description |
|---|---|---|
| `sudo_rules_list` | **mandatory, no default** | List of entries, see below |
| `sudo_rules_nopasswd` | `false` | Default of `nopasswd` for rules that do not set it, so a password is required unless a rule says otherwise |
| `sudo_rules_dir` | `/etc/sudoers.d` | Directory for the drop-in files |
| `sudo_rules_main_file` | `/etc/sudoers` | Main sudoers file, checked for an include of the directory |
| `sudo_rules_file_prefix` | `90-sudo-rules-` | File name prefix (letters, digits, `_`, `-`) |
| `sudo_rules_purge` | `false` | Remove files with the prefix whose entry is no longer listed |
| `sudo_rules_install` | `true` | Install the `sudo` package if missing |

### Entry (`sudo_rules_list[]`)

| Key | Default | Description |
|---|---|---|
| `user` | **mandatory** | User name, or `%<group>` for a group |
| `state` | `present` | `absent` removes the entry's file |
| `comment` | – | Comment at the top of the file (may span several lines) |
| `defaults` | `[]` | sudo options only for this user/group, written as `Defaults:<user> <option>` |
| `rules` | **mandatory if present** | List of rules, each one becomes one sudoers line |

### Rule (`rules[]`)

A rule needs `commands`, `edit_files` or both.

| Key | Default | Description |
|---|---|---|
| `commands` | – | `ALL`, one command or a list of commands. Each starts with an absolute path. Without arguments, **any** arguments are allowed; with arguments, only exactly these. `/usr/bin/foo ""` allows the command only without arguments |
| `edit_files` | `[]` | Absolute paths the user may edit with `sudoedit <file>` (`sudo -e`). The editor runs as the user, so this does not grant a root shell |
| `nopasswd` | `sudo_rules_nopasswd` | `true` writes `NOPASSWD` (no prompt), `false` writes `PASSWD` (the user's own password is required) |
| `run_as` | `root` | User the commands run as (`ALL` for any) |
| `run_as_group` | – | Group the commands may run as (`ALL` for any) |
| `hosts` | `ALL` | Host list of the line, only relevant for shared sudoers files |
| `noexec` | `false` | Add `NOEXEC` (the command cannot start further programs) |
| `setenv` | `false` | Add `SETENV` (the user may pass environment variables) |

The characters `,` `:` `=` and `\` in commands and paths are escaped
automatically, so write them as you would type them (e.g.
`/usr/bin/journalctl --since=today`). Wildcards (`*`, `?`) are passed through
unchanged.

## Restricting sudo rights

sudo rules only ever **add** rights, there is no real "deny". To limit a user
to the commands listed here, make sure no other rule grants more:

- Remove the user from the admin group (`sudo` on Debian/Ubuntu, `wheel` on
  RHEL). Otherwise the default rule `%sudo ALL=(ALL:ALL) ALL` still gives
  full access, no matter what this role writes.
- Do not rely on negations like `ALL, !/usr/bin/su`. They are easy to bypass
  (copy the binary, start a shell, `sudo -s`). Only list what is allowed.
- If several rules match the same command, the last one read decides about
  the password prompt. Files in `/etc/sudoers.d` are read in lexical order at
  the position of the `includedir` line.

Check the result with `sudo -l -U <user>`.

## Security notes

- Granting a command that can spawn a shell or write arbitrary files (editors,
  `less`, `vi`, `find`, `tar`, `cp`, `systemctl edit`, interpreters, ...) is
  effectively full root access, with or without password. Prefer `edit_files`
  for editing files and fixed arguments for everything else; `noexec: true`
  helps against shell escapes but is not a complete protection.
- Wildcards in arguments are matched very loosely by sudo
  (`/usr/bin/cat /var/log/*` also matches `/var/log/../../etc/shadow`). Avoid
  them unless you know the implications.
- `sudoedit` refuses symlinks and files in directories writable by the user.
- The `defaults` options are not interpreted by the role but checked by
  `visudo`. Which options exist depends on the sudo implementation; e.g.
  `sudo-rs` (default on newer Ubuntu releases) does not know `requiretty`.

Requires Ansible ≥ 2.14 and `sudo` (or `sudo-rs`) with `visudo`.
