# Codex CLI Remote Control

Installs the official standalone Codex CLI for existing non-root users on
Debian/Ubuntu with systemd. Each entry (an account, optionally with a working
directory) gets its own user unit (`codex-rc.service` or
`codex-rc-<dir>.service`). The role enables lingering for the account and
creates configured working directories. An account can have several entries.
Existing working-directory permissions are preserved; new directories use mode
`0700`.
Without an entry-specific or global work directory, the service runs in the
home directory. Entries with `state: absent` are removed again. No Node.js
dependency or inbound port is needed. Configuration and credentials are
preserved by default; see [Removing](#removing) for removal effects.
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
          - agent                   # = {name: agent}, uses the global work_dir if set
          - name: agent
            work_dir: work/api      # relative to the user's home
          - name: bob
            work_dir: old
            state: absent           # optional, present (default) or absent
```

An entry uses its `work_dir`, or `codex_rc_work_dir` if its own is unset or
empty. An entry with an effective work directory gets the unit
`<codex_rc_service_name>-<dir>`,
with every run of characters other than letters, digits, `_` and `-` replaced by
`-` (e.g. `codex-rc-work-api`). The service starts in that directory and passes
it to the CLI as `codex --cd <dir> remote-control --no-daemon`, so the agent uses
it as its working root. An entry without an effective work directory
gets the plain `<codex_rc_service_name>` unit (`codex-rc`) and runs in the home
directory. An account/directory pair may only appear once, each account has at
most one entry without an effective work directory, and two directories must not map
to the same unit name (e.g. `work/api` and `work-api`). Earlier versions used
a single `codex-rc.service` per account. For an entry with a work directory,
the role stops, disables and deletes that legacy unit if its
`WorkingDirectory` matches the entry. An entry without a work directory takes
over the legacy unit name: the role replaces the file and manages the service
according to the login and `codex_rc_start` settings.

Save the example play above as `my-codex-playbook.yml`, then run it from the
repository root with `ansible-playbook -i inventory.ini my-codex-playbook.yml`. Alternatively,
edit and run `playbooks/codex-remote-control.yml`; its default configuration
contains only the plain `agent` entry. Accounts must already exist and have
access to the directories you want Codex chats to run in. Remote Control runs with those
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
   user. The CLI documents pairing for its managed daemon. Temporarily stop all
   running Remote Control services for that user before starting the daemon.
   For the two `agent` entries in the example, if `codex_rc_work_dir` is unset
   and both services are running:

   ```bash
   systemctl --user stop codex-rc codex-rc-work-api
   ~/.local/bin/codex remote-control start
   ~/.local/bin/codex remote-control pair
   # Enter the short-lived code in the mobile app if offered.
   ~/.local/bin/codex remote-control stop
   systemctl --user start codex-rc codex-rc-work-api
   ```

   Use your actual unit names and restart only services that were running before.
   Do not run the managed daemon and foreground services simultaneously.
   Never save pairing codes in inventory or logs. Check reconnection after
   switching back to the service; direct Linux pairing remains experimental.

## Settings

| Variable | Default | Purpose |
|---|---|---|
| `codex_rc_users` | required | Non-empty list of account names or dicts with `name`, `work_dir`, `state`; present accounts must exist and be non-root |
| `codex_rc_work_dir` | unset | Fallback working directory (`--cd` and the unit's `WorkingDirectory`) for entries without a nonempty `work_dir`, relative to the home; no spaces or dot components. If neither is set, the service runs in the home directory without `--cd` |
| `codex_rc_service_name` | `codex-rc` | Prefix of the user unit names |
| `codex_rc_release` | `latest` | Release passed to the official installer |
| `codex_rc_update` | `false` | Run installer again for existing installations |
| `codex_rc_start` | `true` | Enable/start with ChatGPT login; false disables/stops the unit |
| `codex_rc_remove_work_dir` | `false` | `state: absent`: also delete the entry's effective work directory and its contents; an entry without one never deletes the home directory |
| `codex_rc_uninstall` | `false` | `state: absent` for the account's last entry: also uninstall the CLI and its packages |
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

If a manually started `codex remote-control --no-daemon` fails with
`socket parent must be owned by the user or root`, a permissive shell umask
may be the cause. Try running it as the configured user with a private umask:

```bash
( umask 077; ~/.local/bin/codex remote-control --no-daemon )
```

Stop the user's running Remote Control services first to avoid running another
server alongside them; restart the services afterward. Units created by this
role already set `UMask=0077`. If a service itself reports this error, check
its effective configuration and journal, substituting its actual unit name:

```bash
systemctl --user show codex-rc-work-api -p UMask
journalctl --user -u codex-rc-work-api -n 50 --no-pager
```

## Removing

For an entry with `state: absent`, the role stops and disables the entry's unit,
deletes it (and a matching legacy `codex-rc.service`) and reloads the user
manager. The work directory and its contents are kept unless
`codex_rc_remove_work_dir: true`. Removing a work directory that contains a
present entry's work directory is rejected. Accounts that do not exist are skipped.

If no unit written by this role (recognized by the line
`# Managed by role codex_remote_control`) and no legacy `codex-rc.service` is
left for the account, `codex_rc_uninstall: true` removes the following paths,
even if they existed before the role was applied: `~/.local/bin/codex`,
`~/.local/bin/codex-code-mode-host`, `~/.codex/packages`, and
`~/.cache/codex-ansible`. It also removes the installer's
`# >>> Codex installer >>>` PATH block in shell startup files. Independently,
`codex_rc_remove_config: true` deletes the entire `~/.codex` directory, including
packages. The role disables linger if no other user service is enabled in
`~/.config/systemd/user/default.target.wants`.
