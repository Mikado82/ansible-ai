# Role `lva_aiy_voicehat`

Makes the button and the LED of the Google AIY Voice HAT (Voice Kit v1) usable
with [Linux Voice Assistant](https://github.com/OHF-Voice/linux-voice-assistant)
(LVA). A small Python controller talks to LVA's
[peripheral WebSocket API](https://github.com/OHF-Voice/linux-voice-assistant/blob/main/docs/peripheral_api.md)
and runs as a systemd service.

## Behaviour

| Button press while ... | Command sent to LVA |
|---|---|
| idle, not muted | `start_listening` (LVA plays its start-listening sound) |
| listening, thinking or speaking | `stop_pipeline` |
| a timer is ringing | `stop_timer_ringing` |
| muted, button controls disabled in HA, or no connection | nothing |

| LVA state | LED |
|---|---|
| listening (wake word or button) | on |
| thinking | clear pulse |
| idle, not muted | faint pulse at a fixed interval (default every 10 s, 10 % duty cycle) |
| muted, or no connection to Home Assistant | off |

The LED follows LVA, so it also reacts to conversations started by wake word or
from Home Assistant (`assist_satellite.start_conversation`).

## What it does

1. Stops and removes a previous manual setup (`lva_aiy_legacy_units`,
   `lva_aiy_legacy_paths`), so two controllers never compete for the GPIOs.
2. Installs `python3-gpiozero`, `python3-lgpio` and `python3-websockets`.
3. Creates the system user `lva-aiy` and adds it to the `gpio` group.
4. Installs the controller to `/opt/lva-aiy-voicehat/` and a systemd unit
   `lva-aiy-voicehat.service`, then enables and starts it.

With `lva_aiy_state: absent` it stops and removes all of this again (the user
only with `lva_aiy_remove_user: true`).

## Usage

```bash
ansible-playbook -i inventory.ini playbooks/lva-aiy-voicehat.yml
```

Run it from the repository root, so `ansible.cfg` points Ansible at `roles/`.

There are no mandatory variables; the defaults match the AIY Voice HAT v1 and
LVA on the same host.

## Variables

| Variable | Default | Description |
|---|---|---|
| `lva_aiy_state` | `present` | `present` or `absent` |
| `lva_aiy_lva_uri` | `ws://localhost:6055` | LVA peripheral WebSocket API |
| `lva_aiy_lva_unit` | `""` | Optional systemd unit the controller starts after: the LVA unit, or `docker.service` if LVA runs in Docker |
| `lva_aiy_check_api` | `true` | After deploying, check that the peripheral API is reachable and warn if not |
| `lva_aiy_gpio_button` | `23` | GPIO (BCM) of the button |
| `lva_aiy_gpio_led` | `25` | GPIO (BCM) of the LED |
| `lva_aiy_listening_brightness` | `1.0` | LED while listening (gamma corrected) |
| `lva_aiy_thinking_period` | `1.0` | Seconds per pulse while thinking |
| `lva_aiy_thinking_min` / `_max` | `0.05` / `1.0` | Brightness range while thinking (gamma corrected) |
| `lva_aiy_idle_pulse` | `true` | Faint pulse while idle and not muted |
| `lva_aiy_idle_interval` | `10.0` | Seconds between two idle pulses |
| `lva_aiy_idle_duration` | `2.0` | Seconds per idle pulse |
| `lva_aiy_idle_max` | `0.10` | Raw PWM duty cycle at the peak of the idle pulse (no gamma correction) |
| `lva_aiy_debug` | `false` | Log every LVA event |
| `lva_aiy_user` | `lva-aiy` | System user of the service |
| `lva_aiy_gpio_group` | `gpio` | Group with access to `/dev/gpiochip*` |
| `lva_aiy_install_dir` | `/opt/lva-aiy-voicehat` | Location of the controller |
| `lva_aiy_service_name` | `lva-aiy-voicehat` | systemd unit name |
| `lva_aiy_packages` | see defaults | Required distribution packages |
| `lva_aiy_legacy_units` | `[aiy-lva-peripheral]` | Units of a manual setup to remove |
| `lva_aiy_legacy_paths` | `[/opt/aiy-lva]` | Paths of a manual setup to remove |
| `lva_aiy_remove_user` | `false` | With `absent`, also remove the user |

### Finding a visible idle brightness

The idle pulse uses the raw duty cycle, because the LED of the AIY button is
driven through a transistor and stays dark at very low values. To find a good
value, stop the service and try a few:

```bash
sudo systemctl stop lva-aiy-voicehat
python3 -c "
from gpiozero import PWMLED; from time import sleep
led = PWMLED(25)
for v in (0.02, 0.05, 0.1, 0.15, 0.2, 0.3):
    print(v); led.value = v; sleep(2)
"
```

## Requirements

- Raspberry Pi OS (Debian based) with the AIY Voice HAT v1. The Voice Bonnet
  (Voice Kit v2) has an RGB LED on I2C and is not supported.
- An LVA version that provides the peripheral WebSocket API (port 6055).

### LVA in Docker

The controller runs on the host, not in the container, because it needs the
GPIOs. It reaches LVA via `ws://localhost:6055`, which works if the container
uses host networking (`network_mode: host`, as in LVA's own compose file) or
publishes the port (`ports: ["6055:6055"]`). Set
`lva_aiy_lva_unit: docker.service` so the controller starts after Docker. The
role warns at the end of the run if the API is not reachable.
- Ansible ≥ 2.14.

Logs: `journalctl -u lva-aiy-voicehat -f`
