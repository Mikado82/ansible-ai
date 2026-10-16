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
  - `tasks/main.yml` – input checks, packages, loops over `claude_rc_users`
  - `tasks/user.yml` – per user: lookup, effective project dir, includes the rest
  - `tasks/install.yml` – packages, project dir, Claude Code native installer
  - `tasks/path.yml` – locate the binary, optionally fix login-shell PATH
  - `tasks/service.yml` – checks, linger, unit, enable/start
  - `templates/claude-rc.service.j2` – the systemd unit
- `roles/claude_remote_control/README.md` – user-facing docs

## Decisions (keep these unless asked otherwise)

- **Language:** everything in English: comments, task names, messages, README.
- **`claude_rc_users` (list) is mandatory, no default.** Enforced by
  `argument_specs.yml` (`required: true`) plus an explicit `assert` that
  rejects empty lists, empty entries and duplicates, plus a per-user check that
  the user exists (`getent`, `fail_key: false`). The old `claude_rc_user` is
  rejected with a migration hint.
- **Per-user loop:** `main.yml` runs `include_tasks: user.yml` with
  `loop_var: _claude_rc_user`; all per-user facts are overwritten each
  iteration. There is **no handler**: handlers run after the loop and would
  only see the last user's facts, so the service task restarts directly when
  the registered unit template result (`claude_rc_unit`) changed.
- **Project dir:** `claude_rc_project_dir` (default `claude-code-rc`) is
  **always relative to each user's home** (real home from getent). A leading
  `~/` is stripped; absolute paths, empty values (= home itself) and `.`/`..`
  components are rejected in `main.yml`. The per-user result is the internal
  fact `_claude_rc_project_dir`. Defaults must not reference other runtime vars,
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
