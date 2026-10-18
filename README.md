# ansible
Ansible roles and playbooks that were created using AI

## Roles

| Role | Playbook | Description |
|---|---|---|
| [`claude_remote_control`](roles/claude_remote_control/README.md) | [`playbooks/claude-remote-control.yml`](playbooks/claude-remote-control.yml) | Runs Claude Code Remote Control as a systemd user service with automatic session resume |
| [`raspotify_homeassistant_hook`](roles/raspotify_homeassistant_hook/README.md) | [`playbooks/raspotify-homeassistant-hook.yml`](playbooks/raspotify-homeassistant-hook.yml) | Sends raspotify/librespot player events incl. track metadata to a Home Assistant webhook |

## Usage

Run playbooks from the repository root so `ansible.cfg` (which sets
`roles_path = ./roles`) is picked up:

```bash
ansible-playbook -i inventory.ini playbooks/<playbook>.yml
```
