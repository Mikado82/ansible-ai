# opencode_remote_control

Install **OpenCode v2** for existing non-root accounts on Debian/Ubuntu
(x86_64 or aarch64) and run `opencode serve` as a systemd **user** service.
Requires Ansible >= 2.14 and privilege escalation to root.

The role uses the [official v2 installer](https://opencode.ai/v2/install),
installs into `~/.opencode/bin`, and verifies the installed major version.
The generic `https://opencode.ai/install` installer installs v1 and is not used.
No Node.js installation is needed. Shell profiles are not modified.

One server runs per account. It provides the API and web UI and can work with
multiple projects. Lingering starts the user manager at boot without an SSH
login. Failed servers restart automatically, and logs go to the user journal.

## Usage

Accounts must already exist. Run the playbook with `become: true`:

```yaml
- hosts: ai-agent
  become: true
  roles:
    - role: opencode_remote_control
      vars:
        opencode_rc_users:
          - name: agent
            work_dir: projects
```

Provide passwords in an encrypted vars file, for example `opencode-vault.yml`:

```yaml
opencode_rc_passwords:
  agent: "replace-with-a-long-random-password"
```

```bash
ansible-vault encrypt opencode-vault.yml
ansible-playbook -i inventory.ini playbooks/opencode-remote-control.yml \
  -e @opencode-vault.yml --ask-vault-pass
```

Keep the encrypted file outside version control unless you intend to commit it.
The role requires a non-empty password even for localhost listeners. It writes
the password and optional server environment to
`~/.config/opencode-ansible/opencode-rc.env` with mode `0600`; secret tasks
suppress logs and diffs. Password changes restart the service.

For multiple accounts, assign distinct ports (all servers share the host network):

```yaml
opencode_rc_users:
  - name: agent
    work_dir: projects
    port: 4096
  - name: developer
    port: 4097
opencode_rc_passwords:
  agent: "{{ vault_agent_opencode_password }}"
  developer: "{{ vault_developer_opencode_password }}"
```

A plain account name, such as `opencode_rc_users: [agent]`, uses the default
port and working directory. `work_dir` is relative to the account's home,
without spaces, absolute paths, or `.`/`..` components. An empty directory means
the home itself. Missing working directories are created; existing permissions
are preserved. The working directory is a starting location, **not a sandbox**:
OpenCode runs with that account's filesystem and command permissions.

## Access and provider credentials

The default listener is `127.0.0.1:4096`. Forward it from your computer:

```bash
ssh -N -L 4096:127.0.0.1:4096 agent@server
```

Open `http://127.0.0.1:4096` in your browser and authenticate with the configured
password (HTTP Basic username: `opencode`). Or connect a **v2** CLI on your
computer, setting `OPENCODE_PASSWORD` to the corresponding password:

```bash
opencode --server http://127.0.0.1:4096
```

For direct VPN access, set `opencode_rc_hostname` to the server's VPN IP.
For browser access through an HTTPS reverse proxy, keep the localhost listener
and proxy to it, including WebSocket connections. This role does not configure
VPNs, proxies, TLS or firewall rules. Use SSH, a VPN or HTTPS when transporting
the password over a network.

Provider credentials are separate from the server password. Log in as the
managed account and configure a provider before using its models:

```bash
~/.opencode/bin/opencode auth login
```

Alternatively supply provider environment variables through encrypted vars:

```yaml
opencode_rc_environment:
  ANTHROPIC_API_KEY: "{{ vault_anthropic_api_key }}"
```

These variables apply to all managed servers. Interactive shell environment
variables are not automatically inherited by a systemd user service.

This is a dedicated foreground server. Manage it with `systemctl --user`, and
connect clients with `--server`. `opencode service` and `opencode pair --remote`
manage OpenCode's separate built-in background service; the role does not
configure that service or OpenTunnel. Do not use them to manage this unit.
See the [OpenCode standalone server documentation](https://opencode.ai/docs/cli/web/#standalone).

## Variables

| Variable | Default | Purpose |
|---|---|---|
| `opencode_rc_users` | required | Account names or dictionaries with `name`, `work_dir`, `port`, `state`; one entry per account |
| `opencode_rc_passwords` | `{}` | Account-to-password mapping; required for present accounts |
| `opencode_rc_version` | `latest` | Current stable release on first install, or an explicit stable `2.x.y`; other majors are refused |
| `opencode_rc_update` | `false` | Reinstall/update even if v2 is already installed |
| `opencode_rc_start` | `true` | Enable/start the unit; `false` stops/disables it |
| `opencode_rc_service_name` | `opencode-rc` | User unit name; keep stable after deployment |
| `opencode_rc_hostname` | `127.0.0.1` | Listen address shared by all managed accounts |
| `opencode_rc_port` | `4096` | Default unprivileged port; per-account `port` overrides it |
| `opencode_rc_work_dir` | `''` | Default relative working directory; empty means home |
| `opencode_rc_cors` | `[]` | Extra HTTP(S) browser origins, for example `https://app.example.com` |
| `opencode_rc_environment` | `{}` | Additional single-line server environment values, including provider API keys |
| `opencode_rc_uninstall` | `false` | For absent accounts, also remove the native binary, compatibility shim and installer cache |

`latest` leaves an existing stable v2 installation in place unless updates are
requested. Changing an explicit version installs that version, including
downgrades. A v1 binary at the managed path is replaced with v2. Installations
at other paths are not changed. If the latest-release endpoint ever returns
another major version, installation fails before running the installer; pin a
`2.x.y` version in that case. No automatic upgrade policy is added by the role.

An unchanged run leaves a running service alone. The role stops services with
changes to their unit, password, environment or installation before starting
them on their configured ports. This lets existing accounts exchange ports in
one run. After an authenticated request to `/api/location`, the role records
the applied configuration in a private `.applied` file alongside the
environment file. A later run can then finish an interrupted restart.
An existing managed service without an applied record restarts on the first
run with this version of the role.
`--check` previews changes without installing OpenCode or contacting its API.

## Operations and removal

As the managed account:

```bash
systemctl --user status opencode-rc
systemctl --user restart opencode-rc
journalctl --user -u opencode-rc -f
```

To remove a service, set that account's state to `absent` and apply the role:

```yaml
opencode_rc_users:
  - name: agent
    state: absent
# Optional: also remove ~/.opencode/bin/opencode, opencode2 and installer cache.
opencode_rc_uninstall: true
```

Absent entries need no password and tolerate missing accounts. Removal stops
and disables the unit, deletes its environment file, and reloads the user
manager if running. Working directories, OpenCode credentials/configuration,
session history, and lingering are preserved. Lingering may be needed by other
user services, including the Claude and Codex roles.

If `user@<uid>.service` fails to start, check the account's password expiry with
`chage -l <user>`; PAM can reject a user manager for an expired password. The
role does not change account passwords or expiry policy.

## Local validation

From this repository root:

```bash
ansible-playbook -i localhost, playbooks/opencode-remote-control.yml --syntax-check
ansible-playbook -i localhost, -c local roles/opencode_remote_control/tests/validate.yml
```

The local tests exercise accepted and rejected settings, verify the unit with
`systemd-analyze`, and check restart planning for port swaps and interrupted
password rotations. They use temporary fixtures for the binary and service files.
