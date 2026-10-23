# CLAUDE.md

Context for Claude Code. This project was started in a claude.ai chat; this file
carries over the decisions made there so work can continue here.

## What this is

An Ansible role, `claude_remote_control`, that runs Claude Code Remote Control
(`claude remote-control`, server mode) as a **systemd user service** on a
server, so it keeps running after SSH logout and across reboots.

- `playbooks/claude-remote-control.yml` – example playbook (run from repo root; `ansible.cfg` sets `roles_path`)
- `roles/claude_remote_control/` (this directory)
  - `defaults/main.yml` – defaults (all variables prefixed `claude_rc_`)
  - `meta/argument_specs.yml` – role argument validation (types, required vars)
  - `tasks/main.yml` – input checks, normalizes `claude_rc_users` into
    `_claude_rc_entries`, packages, loops over present then absent entries
  - `tasks/user.yml` – per present entry: lookup, effective project dir, includes the rest
  - `tasks/install.yml` – packages, project dir, Claude Code native installer
  - `tasks/path.yml` – locate the binary, optionally fix login-shell PATH
  - `tasks/service.yml` – checks, linger, unit, enable/start
  - `tasks/legacy.yml` – removes an old single `claude-rc` unit serving the entry's dir
  - `tasks/absent.yml` – per absent entry: stop/remove unit, optionally the dir
  - `tasks/uninstall.yml` – user's last unit gone: binary, PATH block, config, linger
  - `templates/claude-rc.service.j2` – the systemd unit
- `roles/claude_remote_control/README.md` – user-facing docs

## Decisions (keep these unless asked otherwise)

- **Language:** everything in English: comments, task names, messages, README.
- **`claude_rc_users` (list) is mandatory, no default.** Entries are user
  names (= `{name: ...}`) or dicts with `name`, `project_dir` (default
  `claude_rc_project_dir`) and `state` (`present`/`absent`). Enforced by
  `argument_specs.yml` (`required: true`, `elements: raw`) plus asserts on keys,
  `state`, path rules and uniqueness of (user, dir) and (user, unit). Present
  entries fail if the user doesn't exist; absent entries skip missing users.
- **Per-entry loop:** `main.yml` normalizes into `_claude_rc_entries`
  (`name`, `rel`, `state`, `service`) and runs `user.yml` for present entries,
  **then** `absent.yml` for absent ones (`loop_var: _claude_rc_entry`), so
  moving a dir in one run keeps the binary. All per-entry facts are overwritten
  each iteration. There is **no handler**: handlers run after the loop and would
  only see the last user's facts, so the service task restarts directly when
  the registered unit template result (`claude_rc_unit`) changed.
- **Project dir:** per entry, **always relative to the user's home** (real
  home from getent). A leading `~/` is stripped; absolute paths, empty values
  (= home itself) and `.`/`..` components are rejected in `main.yml`. The
  absolute result is the internal fact `_claude_rc_project_dir`.
- **Unit names:** `<claude_rc_service_name>-<rel dir slug>` (no per-entry
  override). A legacy `claude-rc.service` whose `WorkingDirectory` matches an
  entry is stopped, disabled and deleted (`legacy.yml`).
- **`state: absent`:** removes the unit, keeps the dir (unless
  `claude_rc_remove_project_dir`). The user's agent counts as unused when no
  `*.service` with the marker line `# Managed by role claude_remote_control`
  (from the template) and no legacy unit is left; then `uninstall.yml` removes
  the home-dir native install and the PATH block, optionally `~/.claude*`
  (`claude_rc_remove_config`), and disables linger only if
  `default.target.wants` is empty (other roles' services, e.g. Codex, survive). Defaults must not reference other runtime vars,
  because argument validation renders defaults before tasks run.
- **Binary path:** detected from `claude_rc_binary_candidates` (or forced via
  `claude_rc_binary_path`) and written **absolute** into `ExecStart`, so the
  service never depends on PATH. Separately, `claude_rc_manage_path` adds the
  binary dir to `~/.bash_profile` or `~/.profile` for interactive SSH logins.
- **Session resume (`claude_rc_resume_sessions`, default true):** plain
  `claude remote-control` brings back all sessions the previous server in the
  same directory served, if it was stopped like Ctrl+C. So the unit uses
  `KillSignal=SIGINT`, `KillMode=mixed`, `TimeoutStopSec`. An assert rejects
  `--no-create-session-in-dir`, `--continue`/`-c` and `--session-id` in
  `claude_rc_extra_args`. When false, the unit adds `--no-create-session-in-dir`
  (sessions archived on stop).
- **stdout is discarded by default (`claude_rc_log_stdout: false`):** without a
  TTY the server redraws its status screen about once per second, flooding the
  journal. The unit sets `StandardOutput=null`; stderr still goes to the journal.
  There is no quiet flag for `claude remote-control`.
- **Login and trust can't be automated:** the role only enables (doesn't start)
  the service until `~/.claude/.credentials.json` exists, and prints how to log in
  and trust the project dir once interactively. The home directory is rejected as
  project dir because Claude Code never saves trust for it.
- Requires Ansible >= 2.14 (`ansible.builtin.systemd_service`).

## Known limits (from Claude Code docs)

- Resume only works if the server was offline for up to about 4 hours.
- A hard crash or power loss skips the clean SIGINT shutdown.
- Without a TTY, `claude remote-control` exits with "Workspace not trusted" if
  the directory isn't trusted yet.
- Docs: https://code.claude.com/docs/en/remote-control.md

## Status / next steps

- Only YAML validity and unit-template rendering were checked. **No real
  Ansible run yet**: run `ansible-playbook playbooks/claude-remote-control.yml --syntax-check` and
  `ansible-lint`, then test against a VM.
