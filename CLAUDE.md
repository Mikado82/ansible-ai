# CLAUDE.md

Collection of Ansible roles and playbooks created with AI.

## Conventions

- One role per directory under `roles/`, with an example playbook in `playbooks/`
  named after the role (e.g. `playbooks/claude-remote-control.yml`).
- `ansible.cfg` in the repo root sets `roles_path = ./roles`; run
  `ansible-playbook` from the repo root.
- Everything in English: comments, task names, messages, READMEs.
- Variables are prefixed with the role's short name (e.g. `claude_rc_`).
- Mandatory variables have no default and are enforced via `meta/argument_specs.yml`.
- Target Ansible >= 2.14; use fully qualified module names (`ansible.builtin.*`).
- `meta/main.yml`: `author: Philipp Schaber` for every role.

## Roles

- `roles/claude_remote_control` – Claude Code Remote Control as a systemd user
  service. See its own `CLAUDE.md` for design decisions and open steps.
- `roles/raspotify_homeassistant_hook` – forwards raspotify/librespot player
  events to a Home Assistant webhook. See its own `CLAUDE.md`.
