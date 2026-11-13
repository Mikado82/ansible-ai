# Codex CLI Remote Control

Installs the official standalone Codex CLI for existing non-root users on
Debian/Ubuntu with systemd. Each entry (an account, optionally with a working
directory) gets its own `codex-rc-<dir>.service` user unit and
lingering, plus the directory itself for entries that have one; an account can
have several entries. Entries without a work_dir run in the home directory.
Entries with `state: absent` are removed again. No Node.js dependency or inbound port is
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
        codex_rc_users:
          - agent                   # = {name: agent}, no work_dir
          - name: agent
            work_dir: work/api      # optional; Codex's default path if unset
          - name: bob
            work_dir: old
            state: absent           # optional, present (default) or absent
```

An entry with a work_dir gets the unit `<codex_rc_service_name>-<dir>`,
with every run of characters other than letters, digits, `_` and `-` replaced by
`-` (e.g. `codex-rc-work-api`). The service starts in that directory and passes
it to the CLI as `codex --cd <dir> remote-control --no-daemon`, so the agent uses
it as its working root. An entry without one
gets the plain `<codex_rc_service_name>` unit (`codex-rc`) and runs in the home
directory. An
account/directory pair may only appear once, each account has at most one
entry without a work_dir, and two directories of an account must not map
to the same unit name (e.g. `work/api` and `work-api`). Earlier versions used
a single `codex-rc.service` per account; if its `WorkingDirectory` matches an
entry, or an entry without a work_dir takes over the name, the role
stops, disables and deletes it.

Run `ansible-playbook -i inventory.ini playbooks/codex-remote-control.yml` from
the repository root. Accounts must already exist and have access to the
directories you want Codex chats to run in. Remote Control runs with those
users' permissions.

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
   systemctl --user stop codex-rc-work-api
   ~/.local/bin/codex remote-control start
   ~/.local/bin/codex remote-control pair
   # Enter the short-lived code in the mobile app if offered.
   ~/.local/bin/codex remote-control stop
   systemctl --user start codex-rc-work-api
   ```

   Do not run the managed daemon and the foreground service simultaneously.
   Never save pairing codes in inventory or logs. Check reconnection after
   switching back to the service; direct Linux pairing remains experimental.

## Settings

| Variable | Default | Purpose |
|---|---|---|
| `codex_rc_users` | required | Non-empty list of account names or dicts with `name`, `work_dir`, `state`; present accounts must exist and be non-root |
| `codex_rc_work_dir` | unset | Working directory of the agent (`--cd` and the unit's `WorkingDirectory`), used by entries without `work_dir`, relative to the home; no spaces or dot components. If unset, Codex uses its default path |
| `codex_rc_service_name` | `codex-rc` | Prefix of the user unit names |
| `codex_rc_release` | `latest` | Release passed to the official installer |
| `codex_rc_update` | `false` | Run installer again for existing installations |
| `codex_rc_start` | `true` | Enable/start with ChatGPT login; false disables/stops the unit |
| `codex_rc_remove_work_dir` | `false` | `state: absent`: also delete the entry's work directory and its contents (no directory exists for entries without one) |
| `codex_rc_remove_config` | `false` | `state: absent` for the account's last entry: also delete `~/.codex` (login, configuration, history) |

Installation uses `https://chatgpt.com/codex/install.sh` as the account owner,
with `CODEX_NON_INTERACTIVE=1`, installing to `~/.local/bin/codex`. The installer
manages shell PATH setup. By default the role only installs a missing binary;
changing `codex_rc_release` for an existing installation requires
`codex_rc_update: true`. For example, `-e codex_rc_update=true` updates to the
requested release and restarts the service. Return this option to false for
normal idempotent runs. Upstream CLI auto-updates, if enabled, are independent.

The role does not overwrite `config.toml`, grant project trust, or change sandbox
or approval policies. The work directory does not restrict filesystem access.

## Operations

As the configured user:

```bash
systemctl --user status codex-rc-work-api
journalctl --user -u codex-rc-work-api -f
~/.local/bin/codex login status
```

Check-mode skips service setup for a fresh installation because the binary does
not exist yet. Syntax validation does not establish account eligibility or an
end-to-end mobile connection. If login expires, log in again and restart the
service. To disable it persistently, apply with `codex_rc_start: false`.

## Removing

For an entry with `state: absent`, the role stops and disables the entry's unit,
deletes it (and a matching legacy `codex-rc.service`) and reloads the user
manager. The work directory and its contents are kept unless
`codex_rc_remove_work_dir: true`. Accounts that do not exist are skipped.

If no unit written by this role (recognized by the line
`# Managed by role codex_remote_control`) and no legacy `codex-rc.service` is
left for the account, it also removes what the official installer created
(`~/.local/bin/codex`, `~/.local/bin/codex-code-mode-host`, `~/.codex/packages`,
`~/.cache/codex-ansible` and the `# >>> Codex installer >>>` PATH block in the
shell startup files), deletes `~/.codex` if `codex_rc_remove_config: true`, and
disables linger if no other user service is enabled in
`~/.config/systemd/user/default.target.wants`.
