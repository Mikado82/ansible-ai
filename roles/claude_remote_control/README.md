# Role `claude_remote_control`

Sets up Claude Code "Remote Control" as a systemd user service that runs
independently of SSH logins and survives reboots. Every entry in
`claude_rc_users` (a user and a project directory) gets its own service; a user
can have several project directories. Entries with `state: absent` are removed
again.

## What it does

For each entry in `claude_rc_users` with `state: present` (the default):

1. Installs Claude Code (native installer) if it is not present yet.
2. Detects the actual path of the `claude` binary and writes it into the
   systemd unit as an **absolute** path, so the service does not depend on PATH.
3. Checks whether the binary's directory is in the PATH of the user's login
   shell and adds it to `~/.bash_profile` or `~/.profile` if it is missing
   (disable with `claude_rc_manage_path: false`).
4. Enables linger, writes the unit `claude-rc-<project dir>`, enables it and
   starts it as soon as a login exists.

For entries with `state: absent`, see [Removing](#removing).

## Usage

```bash
ansible-playbook -i inventory.ini playbooks/claude-remote-control.yml
```

Run it from the repository root, so `ansible.cfg` points Ansible at `roles/`.

`claude_rc_users` is mandatory and has no default. Each entry is either a user
name or a dict:

```yaml
claude_rc_users:
  - deploy                    # = {name: deploy}, uses claude_rc_project_dir
  - name: alice
    project_dir: work/api     # optional, default claude_rc_project_dir
  - name: alice
    project_dir: work/web
  - name: bob
    project_dir: old-project
    state: absent             # optional, present (default) or absent
```

The role fails early if the list is missing or empty, if an entry has unknown
keys, no `name` or an invalid `state`, if a user/project directory pair appears
twice, or if a user with `state: present` does not exist on the host. Set it in
the playbook, in inventory vars, or on the command line:

```bash
ansible-playbook -i inventory.ini playbooks/claude-remote-control.yml -e '{"claude_rc_users": ["deploy", "alice"]}'
```

### Unit names

Each entry gets the unit `<claude_rc_service_name>-<project dir>`, with every
run of characters other than letters, digits, `_`, `.` and `-` replaced by `-`
(e.g. `work/api` → `claude-rc-work-api`, the default `claude-remote` →
`claude-rc-claude-remote`). Two directories of one user that map to the same
name (e.g. `work/api` and `work-api`) are rejected.

Earlier versions used a single unit `claude-rc` per user. If such a unit
exists and its `WorkingDirectory` is the project directory of an entry, the
role stops it (with SIGINT, so the new unit resumes its sessions), disables it
and deletes it.

On the first run, log in once manually as each user (`ssh` → `claude` → `/login`), trust
the project directory (see below), then run the role again. The sessions then
appear at [claude.ai/code](https://claude.ai/code) and in the Claude app. Logs:

```bash
journalctl --user -u claude-rc-claude-remote -f
```

By default only stderr reaches the journal. Without a TTY, `claude remote-control`
redraws its status screen (URL, capacity, session name) about once per second,
which would add hundreds of thousands of lines per day and push other logs out
of the journal. Set `claude_rc_log_stdout: true` to log it anyway, e.g. briefly
for debugging.

## Variables

| Variable | Default | Description |
|---|---|---|
| `claude_rc_users` | **mandatory, no default** | List of user names or dicts with `name`, `project_dir`, `state` (see above) |
| `claude_rc_project_dir` | `claude-remote` | Project directory of entries without `project_dir`, always relative to the user's home directory (a leading `~/` is accepted; absolute paths are rejected) |
| `claude_rc_service_name` | `claude-rc` | Prefix of the unit names |
| `claude_rc_install` | `true` | Install Claude Code if missing |
| `claude_rc_binary_path` | `""` | Fixed path to the binary (empty = detect automatically) |
| `claude_rc_binary_candidates` | `~/.local/bin/claude`, `/usr/local/bin/claude`, `/usr/bin/claude` | Locations searched for the binary |
| `claude_rc_manage_path` | `true` | Add the binary's directory to the login shell's PATH |
| `claude_rc_extra_args` | `""` | Additional arguments for `claude remote-control` |
| `claude_rc_start` | `true` | Start the service once logged in |
| `claude_rc_log_stdout` | `false` | Send stdout (status screen, redrawn every second) to the journal; stderr is always logged |
| `claude_rc_resume_sessions` | `true` | Automatically resume sessions after a restart |
| `claude_rc_stop_timeout` | `30` | Seconds systemd waits when stopping the service |
| `claude_rc_remove_project_dir` | `false` | `state: absent`: also delete the project directory and its contents |
| `claude_rc_remove_config` | `false` | `state: absent` for the user's last entry: also delete `~/.claude` and `~/.claude.json` (login, settings, history) |

## Removing

For an entry with `state: absent`, the role

1. stops (with SIGINT) and disables the unit of that project directory,
   deletes it and reloads the user manager (a matching legacy `claude-rc` unit
   is removed as well),
2. keeps the project directory and its contents, unless
   `claude_rc_remove_project_dir: true`.

If no unit written by this role (recognized by the line
`# Managed by role claude_remote_control`) and no legacy `claude-rc` unit is
left for the user, it also

3. removes the native installation (`~/.local/bin/claude`,
   `~/.local/share/claude`, `~/.local/state/claude`, `~/.cache/claude`);
   system-wide installations are left alone,
4. removes the PATH block from `~/.bash_profile` and `~/.profile`,
5. deletes `~/.claude` and `~/.claude.json` if `claude_rc_remove_config: true`,
6. disables linger if no other user service is enabled in
   `~/.config/systemd/user/default.target.wants` (so e.g. a Codex Remote
   Control service of the same user keeps running).

Users that do not exist on the host are skipped.

## Resuming sessions after a restart

On startup, `claude remote-control` automatically brings back **all**
sessions the previous server in the same directory was serving. To make this
work, the role does the following when `claude_rc_resume_sessions: true`:

- Sets `KillSignal=SIGINT`: systemd stops the server as if with Ctrl+C, so
  Claude Code does not archive its sessions (`KillMode=mixed` and
  `TimeoutStopSec` give it time to shut down cleanly).
- Verifies that `claude_rc_extra_args` contains neither
  `--no-create-session-in-dir` (archives sessions on stop) nor
  `--continue`/`--session-id` (server exits after a single session).

Limitations according to the docs: resuming only works if the server was
offline for **no more than about 4 hours**. After that, it starts with a new
session. A hard crash or power loss can prevent the clean shutdown.

With `claude_rc_resume_sessions: false`, the service starts with
`--no-create-session-in-dir`: sessions are archived on stop and every start
begins empty.

## Workspace trust

Without a terminal, `claude remote-control` cannot ask for workspace trust and
exits with `Workspace not trusted`. Therefore, run `claude remote-control`
interactively once in the project directory, confirm with `y` and stop it with
Ctrl+C. The role does not accept the home directory itself as the project
directory, because Claude Code never saves trust for it.

Requires Ansible ≥ 2.14 (`ansible.builtin.systemd_service`).
