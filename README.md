# HA Touch Panel

ESPHome + LVGL firmware for **Guition JC8048W550** (ESP32-S3 N16R8, portrait **480×800** capacitive touch) as Home Assistant room control panels.

First-class HA integration (native API). Entity IDs are set per panel in YAML and compiled into that panel’s firmware.

| | |
|---|---|
| Board | Guition JC8048W550 |
| MCU | ESP32-S3, 16MB flash, 8MB PSRAM |
| Display | 800×480 RGB (`mipi_rgb`) + GT911, UI rotated to **480×800** |
| First panel | `sites/home/panels/suite` |

![Board front](docs/images/board-front.png)

## Project layout

```text
hardware/guition-jc8048w550.yaml   # shared board definition
packages/common/                     # API, Wi-Fi/Improv, fonts, sleep mode
packages/ui/room-shell.yaml          # shared LVGL UI shell (PT-BR labels)
sites/<site>/panels/<panel>/
  panel.yaml                         # entity IDs + live-state wiring
  factory.yaml                       # release / web-flash entrypoint
  dev.yaml                           # local Wi-Fi via secrets
static/                              # GitHub Pages flash site
.github/workflows/                   # CI, Release, Pages
```

- **Site** (`home`, …) = one house / HA instance  
- **Panel** (`suite`, …) = one physical display  
- Device id for releases = `<site>/<panel>` (e.g. `home/suite`)

## Suite controls

UI strings are PT-BR; entity IDs and code stay English.

| UI | Entity | Live state |
|----|--------|------------|
| Spot TV / Pendentes / Sanca / Principal (L1–L4) | `light.suite_switch_suite_l1` … `_l4` | on/off |
| RGB Suíte | `light.rgb_suite` | on/off, brightness, color presets |
| Cabeceira | `light.led_cabeceira_suite` | on/off, brightness |
| Persiana suíte | `cover.persiana_suite` | open / stop / close (RF — no position %) |
| Clima | `climate.suite_2` | temp, setpoint, HVAC `off`/`cool`/`heat`, fan `focus`/`auto`/`high` |
| Desligar tudo | *(local action)* | turns off L1–L4, RGB, cabeceira |
| Sleep Mode | `switch.home_suite_sleep_mode` | when ON: blank after idle + pause LVGL; touch wakes |
| Display Backlight | `light.home_suite_display_backlight` | screen brightness (`gamma_correct: 1.0`, floor via `display_brightness_min`) |

Info overlay (header **ℹ**): ambient / firmware / Wi-Fi / IP, plus **Modo sono** toggle and **Brilho** slider.

Edit entity IDs in [`sites/home/panels/suite/panel.yaml`](sites/home/panels/suite/panel.yaml) substitutions, then rebuild/OTA that panel.

### Sleep mode (night / idle blank)

Exposed as a config switch on the device (e.g. **Sleep Mode**), and also toggled from the info overlay. Schedule it from Home Assistant — do not hard-code times in firmware.

| Switch | Behavior |
|--------|----------|
| OFF | Backlight on at last preferred brightness |
| ON | After `sleep_idle_timeout` (default **60s**) without touch → backlight off + `lvgl.pause`. First touch resumes LVGL and restores preferred brightness **without** activating the widget under the finger. Idle blanks again while the switch stays ON. |

Example HA automations:

```yaml
# Turn sleep mode on at night
alias: Suite panel sleep mode night
trigger:
  - platform: time
    at: "22:00:00"
action:
  - action: switch.turn_on
    target:
      entity_id: switch.home_suite_sleep_mode  # confirm entity id in HA

# Turn sleep mode off in the morning
alias: Suite panel sleep mode morning
trigger:
  - platform: time
    at: "07:00:00"
action:
  - action: switch.turn_off
    target:
      entity_id: switch.home_suite_sleep_mode
```

Override idle timeout per panel with substitution `sleep_idle_timeout` in `panel.yaml` (e.g. `"90s"`).

### Display brightness

- Backlight light uses `gamma_correct: 1.0` so HA / slider brightness % tracks real PWM duty (ESPHome’s default `2.8` crushed the lower half into near-black).
- Preferred brightness is stored and restored on wake, sleep-off, and boot.
- Awake brightness is clamped to a hidden floor (`display_brightness_min`, default **0.15** in [`hardware/guition-jc8048w550.yaml`](hardware/guition-jc8048w550.yaml)); the info slider UI still shows 0–100% mapped onto that floor→100% range. Raise the substitution if the dimmest end is still unreadable after flash.
- Sleep idle blanking still turns the backlight fully off.
## Flash (end users)

1. Open the [GitHub Pages installer](https://dflourusso.github.io/ha-touch-panel/) (after the first release + Pages publish).
2. USB-C → **Install Suite panel** (Chrome/Edge).
3. Set Wi-Fi:
   - **Configure Wi-Fi** in the installer if Improv Serial is detected, or
   - SoftAP **`TouchPanel-Setup`** / password `touchpanel-setup`, or
   - BLE Improv ([improv-wifi.com](https://www.improv-wifi.com/))
4. Adopt in Home Assistant → enable **Allow the device to perform Home Assistant actions**.

**Note:** GT911 touch uses GPIO19/20, which conflict with ESP32-S3 USB-Serial-JTAG. USB Improv after boot often fails on this board; SoftAP and BLE are the reliable paths. Opening **Logs & Console** then **Back** in the installer sometimes re-triggers Improv detection.

You can also download `*.factory.bin` from [Releases](https://github.com/dflourusso/ha-touch-panel/releases).

## Local development

```bash
cp secrets.template.yaml sites/home/panels/suite/secrets.yaml
# edit SSID / password

# Docker (recommended)
docker run --rm -v "$PWD":/config -w /config ghcr.io/esphome/esphome:stable \
  run sites/home/panels/suite/dev.yaml
```

Factory (no home Wi-Fi secrets; SoftAP + Improv serial/BLE):

```bash
docker run --rm -v "$PWD":/config -w /config ghcr.io/esphome/esphome:stable \
  compile sites/home/panels/suite/factory.yaml
```

## Add another panel

1. Copy `sites/home/panels/suite` → `sites/home/panels/<name>`.
2. Change `device_name`, `friendly_name`, `panel_title`, and entity substitutions in `panel.yaml`.
3. Update `factory.yaml` OTA `source` URL to `…/firmware/home-<name>.manifest.json`.
4. Add `home/<name>` to the `device` choice list in [`.github/workflows/release.yml`](.github/workflows/release.yml).
5. Release that device (or `all`).

## Release runbook

1. GitHub → **Actions** → **Release Firmware**
2. Inputs:
   - `version` — semver, e.g. `1.0.0`
   - `build_number` — integer build counter (release tag + artifact folder become `1.0.0+42`)
   - `device` — `home/suite` or `all`
   - optional `release_notes`
3. Workflow builds factory firmware, uploads a GitHub Release tagged `version+build_number`, then **Publish Pages** refreshes the web installer from the latest release assets.

`factory.yaml` must keep unquoted `project.version: dev` — the ESPHome build workflow sed-replaces the literal `version: dev`. Quoted `"dev"` or a hardcoded semver will leave the install page stuck on the wrong version.

## CI

PRs/pushes that touch YAML compile every discovered `sites/**/factory.yaml` and `dev.yaml` (dev uses `secrets.template.yaml`).

## Notes

- GPIO19/20 are used for the GT911 I²C bus on this board (conflicts with USB-Serial-JTAG; logger defaults to UART0).
- Changing entity bindings requires a recompile/OTA of that panel (not runtime rebinding).
- If the UI is upside-down after flash, change `rotation` in `packages/ui/room-shell.yaml` from `90°` to `270°`.
