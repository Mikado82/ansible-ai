# CLAUDE.md

Context for Claude Code. This project was started in a claude.ai chat; this file
carries over the decisions made there so work can continue here.

## What this is

An Ansible role, `raspotify_homeassistant_hook`, that hooks into raspotify
(librespot as a Spotify Connect receiver on a Raspberry Pi) and sends every
player event to a Home Assistant webhook. The original use case is an old
stereo connected to a Pi, exposed in Home Assistant as a media player built
with the HACS integration "Template Media Player".

- `playbooks/raspotify-homeassistant-hook.yml` – example playbook (run from repo root)
- `roles/raspotify_homeassistant_hook/` (this directory)
  - `defaults/main.yml` – defaults (all variables prefixed `raspotify_ha_hook_`)
  - `meta/argument_specs.yml` – role argument validation (types, required vars)
  - `tasks/main.yml` – checks, optional raspotify install, packages, script, config
  - `handlers/main.yml` – restart raspotify
  - `templates/raspotify-ha-hook.sh.j2` – the hook script
- `roles/raspotify_homeassistant_hook/README.md` – user-facing docs, incl. the
  Home Assistant template sensor

## Decisions (keep these unless asked otherwise)

- **Language:** everything in English: comments, task names, messages, README.
- **`raspotify_ha_hook_url` and `raspotify_ha_hook_webhook_id` are mandatory,
  no default.** Enforced by `argument_specs.yml` (`required: true`) plus an
  `assert` checking the URL scheme and the webhook ID characters.
- **Hook via `LIBRESPOT_ONEVENT`:** raspotify has no Home Assistant integration
  and librespot has no local control API, so the role only *reports* events.
  Control (play/pause/next) stays with the Home Assistant Spotify integration.
- **JSON built with `jq`**, not string concatenation, so titles with quotes,
  umlauts etc. stay valid. Multiple artists are joined with ", ", only the
  first cover URL is sent. `curl` has a timeout and the script always exits 0,
  so an unreachable Home Assistant never affects playback.
- **Payload:** `track_changed` carries `title`, `artist`, `album`, `cover`,
  `duration_ms`; every other event only `event`. The Home Assistant sensor keeps
  the last metadata via `this.attributes`.
- **Handler is fine here:** unlike `claude_remote_control` there is no per-user
  loop, so a normal "Restart raspotify" handler is used.
- **`LIBRESPOT_NAME`** is only touched when `raspotify_ha_hook_device_name` is
  set, so an existing name is kept by default.
- **raspotify install is opt-in** (`raspotify_ha_hook_install_raspotify`),
  because it pipes a remote install script.

## Status / next steps

- Only YAML validity and rendering/running the hook script with sample
  environment variables were checked. **No real Ansible run yet**: run
  `ansible-playbook playbooks/raspotify-homeassistant-hook.yml --syntax-check`
  and `ansible-lint`, then test against a Pi with raspotify.
