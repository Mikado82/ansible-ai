# CLAUDE.md

Context for Claude Code.

## What this is

An Ansible role, `sudo_rules`, that grants sudo rights per user or group via
drop-in files in `/etc/sudoers.d`, with or without password, limited to
specific commands (optionally with fixed arguments) or to files editable via
`sudoedit`. It started as `passwordless_sudo` and was renamed when the
`nopasswd` option was added.

- `playbooks/sudo-rules.yml` – example playbook (run from repo root)
- `roles/sudo_rules/` (this directory)
  - `defaults/main.yml` – defaults (all variables prefixed `sudo_rules_`)
  - `vars/main.yml` – internal: file name mapping and validation regexes
  - `meta/argument_specs.yml` – argument validation incl. nested entry/rule keys
  - `tasks/main.yml` – checks, sudo install, include check, files, purge
  - `templates/sudoers.j2` – one drop-in file per entry
- `roles/sudo_rules/README.md` – user-facing docs

## Decisions (keep these unless asked otherwise)

- **Naming:** the main variable is `sudo_rules_list` (not `sudo_rules_users`,
  which suggested that rules are defined elsewhere), and the entry key is
  `user` (a user name or `%group`, as in sudoers) instead of `name`. A bare
  `sudo_rules` was rejected because ansible-lint requires the `sudo_rules_`
  prefix for role variables (`var-naming[no-role-prefix]`).
- **One file per entry** (`<prefix><user>`), so removing an entry is a single
  file delete and a broken entry never affects other users. `%group` maps to
  `group-<group>`; characters other than `[A-Za-z0-9_-]` become `_`, because
  sudo ignores files containing `.` in an includedir. Name collisions after
  this mapping are rejected.
- **Password per rule:** `nopasswd: true` writes `NOPASSWD`, `false` writes
  an explicit `PASSWD`. sudo only inherits tags within one line (to the
  following commands), not across lines, so `PASSWD` is not strictly needed;
  it is written anyway so every line states its behaviour.
  The default `sudo_rules_nopasswd` is `false` (secure by default):
  passwordless rights must be requested explicitly per rule. All tag flags
  go through `| bool`, because extra vars (`-e x=false`) arrive as strings.
  `NOEXEC` and `SETENV` are optional tags.
- **Own template instead of `community.general.sudoers`** (checked against
  12.1.0 and the 14.0.1 docs): the module writes one line per file and has
  no run-as group, comments, escaping, input validation (a newline in a
  command can inject a valid extra line), purge or include check; `defaults`
  only exists since 13.1.0. `linux-system-roles.sudo` does not support
  Debian/Ubuntu and rewrites `/etc/sudoers` by default.
- **No membership management:** the role does not remove users from the
  `sudo`/`wheel` group; the README explains that sudo rules only add rights.
- **`edit_files` → `sudoedit <path>`** instead of granting an editor, because
  an editor run as root is a root shell.
- **Defense against sudoers injection:** argument specs reject unknown keys and
  wrong types; asserts require absolute paths, single lines and restricted
  characters in names/run_as/hosts. `, : = \` in commands and paths are
  escaped in the template. Every file goes through `visudo -cf` via
  `validate:` before it replaces the old one.
- **Include check instead of editing `/etc/sudoers`**: the role never touches
  the main file; it only fails if `@includedir`/`#includedir` for the
  directory is missing.
- **Purge is opt-in** and limited to files with the prefix, so manually
  created files and files of other roles are never removed.
- Argument spec defaults are documentation only for nested keys; tasks and
  template therefore use `| default(...)` for every optional key.

## Status / next steps

- Tested locally (Ansible 2.20, sudo-rs `visudo-rs` 0.2.13): template rendering
  incl. `visudo -cf` validation, a full role run in check mode against temp
  paths (validation, include check, absent entries, purge), and `PASSWD` rules, the negative
  cases (relative command, unknown key, missing rules, empty rule, invalid and
  colliding names, missing include, newline injection, invalid state).
- **No real run with `become` against a host yet**, and not tested with
  classic `sudo` (Debian/RHEL) `visudo`.
