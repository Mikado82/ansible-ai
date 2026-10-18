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
  - `templates/raspotify-dropin.conf.j2` – systemd drop-in for service events
  - `examples/homeassistant/raspotify_media_player.yaml` – generic Home
    Assistant package (webhook sensor + Template Media Player), not deployed by
    the role
- `roles/raspotify_homeassistant_hook/README.md` – user-facing docs

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
- **Service events against stale state:** librespot sends nothing when
  raspotify restarts, crashes or the Pi shuts down/boots, so the HA sensor could
  stay on `playing` forever. A systemd drop-in
  (`/etc/systemd/system/raspotify.service.d/homeassistant-hook.conf`, toggled by
  `raspotify_ha_hook_service_events`, default true) calls the same script with
  `PLAYER_EVENT=service_started` (`ExecStartPost`) and `service_stopped`
  (`ExecStopPost`, also runs after a crash). Both are prefixed with `-` so a
  failure never affects the service. Only these events use `curl --retry`
  (boot: network may not be up yet); player events are sent once to keep their
  order. A hard power loss is only cleared on the next boot.
- **No timeout-based staleness check in HA** (rejected): it would only work for
  `playing`, because a paused track sends no events for an arbitrary time.
- **Handler is fine here:** unlike `claude_remote_control` there is no per-user
  loop, so a normal "Restart raspotify" handler is used.
- **`LIBRESPOT_NAME`** is only touched when `raspotify_ha_hook_device_name` is
  set, so an existing name is kept by default.
- **Home Assistant package:** placeholders only (`my_stereo`, `switch.my_stereo_power`,
  ...). Entity IDs of the media player live once in its `variables` block and
  are reused via YAML anchors (`&power_switch` / `*power_switch`) in the
  service scripts and via the variables in the templates. Power state and
  on/off come from one switch (the user wraps power metering + IR in a
  template switch); volume from two buttons; play/pause/next/previous go to the
  Spotify integration.
- **Two sources of truth, on purpose:** the display (state, title, cover) comes
  from the webhook sensor (fast, but only knows the last event it received).
  Playback commands are guarded only by the Spotify integration's `source`
  attribute (active device from the Spotify cloud) == `spotify_device_name`
  (should match `raspotify_ha_hook_device_name`), via the shared condition
  anchor `&spotify_on_this_device`. So a missed webhook event can never pause or
  skip on another device of the account. Trade-off: the Spotify integration
  polls, so commands may do nothing for up to ~30 s after switching devices.
- **Sensor reset in HA:** the template sensor has a second trigger (state of the
  power switch to `off`, id `power_off`) that sets it to `stopped`;
  `session_disconnected`, `service_started` and `service_stopped` also map to
  `stopped`. Templates check `trigger.id == 'webhook'` before touching
  `trigger.json`. The media player block comes first in the file because its
  `variables` define the YAML anchors that the sensor's trigger reuses. The package is not deployed by the role because
  many Home Assistant installations (HA OS) are not managed via SSH/Ansible.
- **Auto power on is an automation in the package** (packages may contain
  `automation:`; only the UI editor can't edit it). Triggers: webhook sensor
  `to: playing` (immediate) and Spotify `source` `to: spotify_device_name`
  (polled fallback); condition switch is off; action `switch.turn_on`.
- **Auto pause is a second automation** (switch `from: on` `to: off`,
  conditions `*spotify_on_this_device` and Spotify state `playing`, action
  `media_pause`). It replaces the pause that used to be inside the media
  player's `turn_off` (which now only switches off), so pausing lives in one
  place and also covers switching off via the switch or the device's own remote.
  It also prevents auto power on from switching the device on again with the
  next `playing` event. Limitation: within ~30 s after switching to the Pi the
  polled `source` may not match yet, so nothing is paused. Spotify has no
  "stop", only pause. The anchor `&spotify_on_this_device` is now defined in
  `media_play`. Both automations sit last in the file so the anchors are
  defined. A trigger-based template
  sensor's `actions:` was rejected as a workaround: it is meant for fetching
  data, not for switching devices.
- **raspotify install is opt-in** (`raspotify_ha_hook_install_raspotify`),
  because it pipes a remote install script.

## Status / next steps

- Only YAML validity and rendering/running the hook script with sample
  environment variables were checked. **No real Ansible run yet**: run
  `ansible-playbook playbooks/raspotify-homeassistant-hook.yml --syntax-check`
  and `ansible-lint`, then test against a Pi with raspotify.
- The Home Assistant package was only checked for YAML validity. Not yet
  verified in a real Home Assistant: that Template Media Player resolves
  `variables` inside `global_template`; the sensor templates were rendered with
  plain Jinja and sample triggers only.
- Not verified on a real Pi: which events librespot sends when playback is
  transferred to another device (expected `stopped`, version-dependent).
