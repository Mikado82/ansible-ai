# CLAUDE.md

Context for Claude Code. This project was started in a claude.ai chat; this file
carries over the decisions made there so work can continue here.

## What this is

An Ansible role, `lva_aiy_voicehat`, that makes the button and LED of the
Google AIY Voice HAT (Voice Kit v1) on a Raspberry Pi usable with Linux Voice
Assistant (LVA) through LVA's peripheral WebSocket API (port 6055).

- `playbooks/lva-aiy-voicehat.yml` – example playbook (run from repo root)
- `roles/lva_aiy_voicehat/` (this directory)
  - `defaults/main.yml` – defaults (all variables prefixed `lva_aiy_`)
  - `meta/argument_specs.yml` – role argument validation (types, choices)
  - `tasks/main.yml` – value checks, then `present.yml` or `absent.yml`
  - `tasks/present.yml` – legacy cleanup, packages, user, script, unit, start
  - `tasks/absent.yml` – stop/remove unit, script, state dir, optionally user
  - `tasks/legacy.yml` – removes the manual setup from the original chat
  - `handlers/main.yml` – restart the controller
  - `templates/lva-aiy-voicehat.py.j2` – the controller (Python, asyncio)
  - `templates/lva-aiy-voicehat.service.j2` – systemd system unit
- `roles/lva_aiy_voicehat/README.md` – user-facing docs

## Decisions (keep these unless asked otherwise)

- **Language:** everything in English, including the controller script.
- **No mandatory variables:** defaults match the AIY Voice HAT v1 (button
  GPIO 23, LED GPIO 25) and LVA on localhost.
- **Peripheral API instead of patching LVA:** the controller is a separate
  process that reacts to LVA events and sends commands, as intended by LVA.
- **Button:** context-sensitive, like LVA's minimal example: `start_listening`,
  `stop_pipeline` while active, `stop_timer_ringing` while ringing. While
  muted it does nothing (the user's Home Assistant automation also never
  starts a conversation while muted; `start_listening` would be a no-op anyway).
  Respects `button_controls_disabled`.
- **LED:** on while listening (`wake_word_detected`, `listening`), clear pulse
  while `thinking`, faint pulse while idle and not muted, off while muted or
  without HA connection. During `tts_speaking` it shows the idle pulse (not
  requested otherwise).
- **Gamma correction** (2.2) for listening and thinking, but **not** for the
  idle pulse: with gamma, 10-12 % became ~1 % duty and the transistor-driven
  LED stayed dark. `lva_aiy_idle_max` is therefore the raw duty cycle. The
  user chose every 10 s up to 10 %.
- **Runs as a system service** under a dedicated system user in group `gpio`
  (Raspberry Pi OS grants `/dev/gpiochip*` to it), not as root.
- **`StateDirectory` + `WorkingDirectory`:** lgpio creates notification pipes
  in the current directory, which must be writable for the service user.
- **Handler is fine here:** no per-user loop.
- **LVA in Docker (the user's setup, container `linux-voice-assistant`):** the
  controller stays on the host (GPIO access) and connects via
  `ws://localhost:6055`; the example playbook sets
  `lva_aiy_lva_unit: docker.service` for ordering. No dependency on the
  container itself (no `docker` module, no container name variable): the
  controller reconnects anyway. A non-fatal `wait_for` check
  (`lva_aiy_check_api`) only warns if the API is unreachable, since LVA may
  legitimately be down during the run.
- **Legacy cleanup:** the original chat installed the script manually as
  `/opt/aiy-lva/aiy_lva_peripheral.py` with `aiy-lva-peripheral.service`.
  Both are removed (configurable), so two processes never fight over the GPIOs.

## Status / next steps

- Checked: YAML validity, defaults matching `argument_specs.yml`, rendering
  both templates with plain Jinja2 (with the `trim_blocks` setting Ansible
  uses) and compiling the rendered script with Python. **Not yet run:**
  `ansible-playbook playbooks/lva-aiy-voicehat.yml --syntax-check`,
  `ansible-lint`, and a real run against a Pi.
- The controller logic itself was tested by the user on the Pi before it was
  turned into a role (manual setup).
- Not verified: `python3-lgpio` availability on older Raspberry Pi OS releases
  (Bullseye). Remove it from `lva_aiy_packages` there; gpiozero then falls
  back to RPi.GPIO.
