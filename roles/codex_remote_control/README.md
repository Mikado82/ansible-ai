# Codex CLI Remote Control

Installs the official standalone Codex CLI for existing non-root users on
Debian/Ubuntu with systemd. Each account gets a `codex-rc.service` user unit,
lingering, and a working directory. No Node.js dependency or inbound port is
needed by this role. Existing Codex configuration and credentials are preserved.
Run with privilege escalation and gathered facts (Ansible >= 2.14).

## Mobile availability

The [CLI reference](https://learn.chatgpt.com/docs/developer-commands?surface=cli)
documents `remote-control --no-daemon`, `start`, `stop`, and `pair` as experimental.
This role uses foreground mode so systemd can supervise the process.
It checks that the installed CLI supports this flag before configuring the service.

The [mobile setup guide](https://learn.chatgpt.com/docs/remote-connections) still
requires setup through the desktop app on macOS/Windows and says setup from the
CLI is unavailable. Direct Linux-to-mobile pairing therefore cannot be guaranteed
by this role. Account/workspace access and the mobile UI must support it; a
workspace administrator may need to enable Remote Control. The documented
desktop-app setup is the fallback if direct pairing is unavailable.

## Usage

```ini
[codex_remote_control]
devbox ansible_host=192.0.2.10
```

```yaml
- name: Configure Codex
  hosts: codex_remote_control
  become: true
  roles:
    - role: codex_remote_control
      vars:
        codex_rc_users: [admin]
```

Run `ansible-playbook -i inventory.ini playbooks/codex-remote-control.yml` from
the repository root. Accounts must already exist and have access to the projects
you want Codex to work on. Remote Control runs with those users' permissions.

## First login and pairing

1. Apply the role. Without a ChatGPT login the unit is installed but disabled
   and stopped, avoiding an unauthenticated service at boot.
2. SSH into the target as each configured user and run:

   ```bash
   ~/.local/bin/codex login --device-auth
   ```

   Complete the browser login with the same ChatGPT account/workspace used on
   mobile. Device-code login must be available for your account; alternatively
   use `~/.local/bin/codex login` and its browser/SSH-forwarding flow. See
   [authentication](https://learn.chatgpt.com/docs/auth). API-key login alone does
   not activate this role's Remote Control service. Credentials stay on the host.
3. Apply the role again to enable and start the service.
4. If the mobile app offers manual pairing, generate a fresh code as the same
   user. The CLI documents pairing for its managed daemon, so temporarily stop
   the systemd service before using that daemon:

   ```bash
   systemctl --user stop codex-rc
   ~/.local/bin/codex remote-control start
   ~/.local/bin/codex remote-control pair
   # Enter the short-lived code in the mobile app if offered.
   ~/.local/bin/codex remote-control stop
   systemctl --user start codex-rc
   ```

   Do not run the managed daemon and the foreground service simultaneously.
   Never save pairing codes in inventory or logs. Check reconnection after
   switching back to the service; direct Linux pairing remains experimental.

## Settings

| Variable | Default | Purpose |
|---|---|---|
| `codex_rc_users` | required | Non-empty list of distinct existing non-root accounts |
| `codex_rc_project_dir` | `codex-work` | Working directory relative to each home; no spaces or dot components |
| `codex_rc_service_name` | `codex-rc` | User unit name without `.service` |
| `codex_rc_release` | `latest` | Release passed to the official installer |
| `codex_rc_update` | `false` | Run installer again for existing installations |
| `codex_rc_start` | `true` | Enable/start with ChatGPT login; false disables/stops the unit |

Installation uses `https://chatgpt.com/codex/install.sh` as the account owner,
with `CODEX_NON_INTERACTIVE=1`, installing to `~/.local/bin/codex`. The installer
manages shell PATH setup. By default the role only installs a missing binary;
changing `codex_rc_release` for an existing installation requires
`codex_rc_update: true`. For example, `-e codex_rc_update=true` updates to the
requested release and restarts the service. Return this option to false for
normal idempotent runs. Upstream CLI auto-updates, if enabled, are independent.

The role does not overwrite `config.toml`, grant project trust, or change sandbox
or approval policies. The working directory does not restrict filesystem access.

## Operations

As the configured user:

```bash
systemctl --user status codex-rc
journalctl --user -u codex-rc -f
~/.local/bin/codex login status
```

Check-mode skips service setup for a fresh installation because the binary does
not exist yet. Syntax validation does not establish account eligibility or an
end-to-end mobile connection. If login expires, log in again and restart the
service. To disable it persistently, apply with `codex_rc_start: false`.
