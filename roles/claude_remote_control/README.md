# Role `claude_remote_control`

Sets up Claude Code "Remote Control" as a systemd user service that runs
independently of SSH logins and survives reboots. Every user in
`claude_rc_users` gets their own installation, project directory and service.

## What it does

For each user in `claude_rc_users`:

1. Installs Claude Code (native installer) if it is not present yet.
2. Detects the actual path of the `claude` binary and writes it into the
   systemd unit as an **absolute** path, so the service does not depend on PATH.
3. Checks whether the binary's directory is in the PATH of the user's login
   shell and adds it to `~/.bash_profile` or `~/.profile` if it is missing
   (disable with `claude_rc_manage_path: false`).
4. Enables linger, writes the unit, enables it and starts it as soon as a
   login exists.

## Usage

```bash
ansible-playbook -i inventory.ini playbooks/claude-remote-control.yml
```

Run it from the repository root, so `ansible.cfg` points Ansible at `roles/`.

`claude_rc_users` is mandatory and has no default. The role fails early if it
is missing, empty, contains empty or duplicate entries (via
`meta/argument_specs.yml` and an explicit check) or if one of the users does
not exist on the host. Set it in the playbook, in inventory vars, or on the
command line:

```bash
ansible-playbook -i inventory.ini playbooks/claude-remote-control.yml -e '{"claude_rc_users": ["deploy", "alice"]}'
```

The former single-user variable `claude_rc_user` is no longer supported; the
role fails if it is still set.

On the first run, log in once manually as each user (`ssh` → `claude` → `/login`), trust
the project directory (see below), then run the role again. The sessions then
appear at [claude.ai/code](https://claude.ai/code) and in the Claude app. Logs:

```bash
journalctl --user -u claude-rc -f
```

By default only stderr reaches the journal. Without a TTY, `claude remote-control`
redraws its status screen (URL, capacity, session name) about once per second,
which would add hundreds of thousands of lines per day and push other logs out
of the journal. Set `claude_rc_log_stdout: true` to log it anyway, e.g. briefly
for debugging.

## Variables

| Variable | Default | Description |
|---|---|---|
| `claude_rc_users` | **mandatory, no default** | List of users that each run Claude Code and their own service (must exist on the host) |
| `claude_rc_project_dir` | `claude-code-rc` | Working directory, always relative to each user's home directory (a leading `~/` is accepted; absolute paths are rejected) |
| `claude_rc_service_name` | `claude-rc` | Name of the unit |
| `claude_rc_install` | `true` | Install Claude Code if missing |
| `claude_rc_binary_path` | `""` | Fixed path to the binary (empty = detect automatically) |
| `claude_rc_binary_candidates` | `~/.local/bin/claude`, `/usr/local/bin/claude`, `/usr/bin/claude` | Locations searched for the binary |
| `claude_rc_manage_path` | `true` | Add the binary's directory to the login shell's PATH |
| `claude_rc_extra_args` | `""` | Additional arguments for `claude remote-control` |
| `claude_rc_start` | `true` | Start the service once logged in |
| `claude_rc_log_stdout` | `false` | Send stdout (status screen, redrawn every second) to the journal; stderr is always logged |
| `claude_rc_resume_sessions` | `true` | Automatically resume sessions after a restart |
| `claude_rc_stop_timeout` | `30` | Seconds systemd waits when stopping the service |

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
