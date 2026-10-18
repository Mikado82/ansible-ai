# Role `raspotify_homeassistant_hook`

Forwards [raspotify](https://github.com/dtcooper/raspotify) / librespot player
events to a Home Assistant webhook. On `track_changed` the payload also contains
title, artist, album, cover URL and duration, so Home Assistant knows exactly
what this device is playing, independent of other Spotify Connect devices on
the same account.

## What it does

1. Checks that raspotify is installed (or installs it with
   `raspotify_ha_hook_install_raspotify: true`).
2. Installs `jq` and `curl`.
3. Installs a hook script that turns every librespot event into a JSON POST to
   `<raspotify_ha_hook_url>/api/webhook/<raspotify_ha_hook_webhook_id>`.
4. Sets `LIBRESPOT_ONEVENT` (and optionally `LIBRESPOT_NAME`) in the raspotify
   config.
5. Adds a systemd drop-in to `raspotify.service` that also sends
   `service_started` / `service_stopped` when raspotify starts or stops (boot,
   restart, crash, shutdown). librespot sends no `stopped` event in these cases,
   so without it Home Assistant could keep showing a stale `playing` state.
   Disable with `raspotify_ha_hook_service_events: false`.
6. Restarts raspotify when something changed.

## Usage

```bash
ansible-playbook -i inventory.ini playbooks/raspotify-homeassistant-hook.yml
```

Run it from the repository root, so `ansible.cfg` points Ansible at `roles/`.

`raspotify_ha_hook_url` and `raspotify_ha_hook_webhook_id` are mandatory and
have no default. The role fails early if they are missing (via
`meta/argument_specs.yml`) or malformed.

## Variables

| Variable | Default | Description |
|---|---|---|
| `raspotify_ha_hook_url` | **mandatory, no default** | Home Assistant base URL as reachable from the Pi, e.g. `http://homeassistant.local:8123` |
| `raspotify_ha_hook_webhook_id` | **mandatory, no default** | Webhook ID of the Home Assistant sensor (letters, digits, `_`, `-`) |
| `raspotify_ha_hook_device_name` | `""` | Name in the Spotify app (`LIBRESPOT_NAME`); empty keeps the current setting. Recommended, the Home Assistant example uses it to identify the active device |
| `raspotify_ha_hook_service_events` | `true` | Send `service_started`/`service_stopped` on raspotify start/stop (systemd drop-in) |
| `raspotify_ha_hook_timeout` | `5` | HTTP timeout in seconds |
| `raspotify_ha_hook_conf_path` | `/etc/raspotify/conf` | raspotify config file |
| `raspotify_ha_hook_script_path` | `/usr/local/bin/raspotify-ha-hook.sh` | Location of the hook script |
| `raspotify_ha_hook_install_raspotify` | `false` | Install raspotify if it is missing |
| `raspotify_ha_hook_install_url` | `https://dtcooper.github.io/raspotify/install.sh` | raspotify install script |

## Payload

```json
{"event": "track_changed", "title": "...", "artist": "Artist A, Artist B",
 "album": "...", "cover": "https://...", "duration_ms": 215000}
```

All other events only contain the event name, e.g. `{"event": "playing"}`.
Relevant events: `playing`, `paused`, `stopped`, `track_changed`,
`session_disconnected`, and from the systemd drop-in `service_started` and
`service_stopped`.

Player events are sent once without retry, to keep their order. The service
events are retried a few times, because they may fire at boot before the
network is up.

## Home Assistant side

[`examples/homeassistant/raspotify_media_player.yaml`](examples/homeassistant/raspotify_media_player.yaml)
is a ready-to-use [package](https://www.home-assistant.io/docs/configuration/packages/)
with two parts:

- a **trigger-based template sensor** that receives the webhook, stores the
  playback state and keeps the last track metadata between events. It is reset
  to `stopped` on the service events and when the power switch turns off.
- a **media player** built with the HACS integration
  [Template Media Player](https://github.com/EuleMitKeule/template-media-player)
  that combines the sensor with a power switch, volume buttons (e.g. IR via
  ESPHome) and the Spotify integration for play/pause/next/previous.

The display (state, title, cover) comes from the webhook sensor: fast, but it
only knows the last event it received. Playback control is guarded by the
Spotify integration instead: commands only act if its `source` attribute (the
active device, reported by the Spotify cloud) equals the device name. A missed
webhook event can therefore never pause or skip on another device. The Spotify
integration polls, so `source` may lag up to about 30 seconds after switching
devices; during that time the controls do nothing.

Setup:

1. Install *Template Media Player* via HACS and set up the Spotify integration.
2. Enable packages in `configuration.yaml` (once):
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
3. Copy the example to `/config/packages/` and replace the placeholders listed
   at the top of the file. The webhook ID must match
   `raspotify_ha_hook_webhook_id`, `spotify_device_name` must match
   `raspotify_ha_hook_device_name`. Entity IDs of the media player are set once
   in its `variables` block and reused via YAML anchors.
4. Check the configuration and restart Home Assistant.

## Limitations

- librespot only *reports* events; it has no local control API. Playback
  control from Home Assistant has to go through the Spotify integration.
- The playback position is not reported.
- Spotify Connect via librespot requires Spotify Premium.

Requires Ansible ≥ 2.14 (`ansible.builtin.systemd_service`) and a Debian-based
system (Raspberry Pi OS).
