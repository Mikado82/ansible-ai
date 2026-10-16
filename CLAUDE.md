# CLAUDE.md

Collection of Ansible roles and playbooks created with AI.

## Conventions

- One role per directory under `roles/`, with an example playbook in the repo root
  named after the role (e.g. `claude-remote-control.yml`).
- Everything in English: comments, task names, messages, READMEs.
- Variables are prefixed with the role's short name (e.g. `claude_rc_`).
- Mandatory variables have no default and are enforced via `meta/argument_specs.yml`.
- Target Ansible >= 2.14; use fully qualified module names (`ansible.builtin.*`).

## Roles

- `roles/claude_remote_control` – Claude Code Remote Control as a systemd user
  service. See its own `CLAUDE.md` for design decisions and open steps.
