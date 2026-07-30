# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Pre-configured [ZMK](https://zmk.dev) firmware for the wireless Charybdis split trackball keyboard. Not application code — it is a ZMK *config repo*: keymaps, devicetree shields/overlays, Kconfig fragments, and a build matrix. There is no unit test suite; "does it build" is the correctness check. Targets stable ZMK (v0.4.1 line) on nice!nano v2 and Xiao BLE boards.

## Build

Firmware is built two ways; both consume the same `build.yaml` matrix.

**Local (Docker) — the fast dev loop:**
```bash
docker-compose -f local-build/docker-compose.yml run --rm builder
```
Output `.uf2` files land in `firmwares/<format>/<keymap>/`. `settings_reset.uf2` goes to `firmwares/` root.

- `ENABLE_USB_LOGGING=true docker-compose ... run --rm builder` — build with USB CDC logging (off by default; hurts battery). Conflicts with ZMK Studio, so it disables the studio snippet.
- `SKIP_WEST_UPDATE=true ...` — build against whatever is checked out in the local module folders instead of pulling configured revisions. Use only when testing local module edits.
- Interactive shell: `docker-compose -f local-build/docker-compose.yml run --rm --entrypoint bash builder`, then `bash ./local-build/build_setup.sh`.
- Save full output: `docker-compose ... run --rm builder > logs.txt 2>&1`.

**CI (GitHub Actions):** `.github/workflows/build.yml` builds the same matrix and publishes artifacts. Fork + push to trigger.

There is no lint step. A build either succeeds or fails.

## How the build actually works (`local-build/build_setup.sh`)

This ~450-line bash script is the heart of local builds. It:
1. `west init -l config` + `west update` to pull ZMK and the two extra modules from `config/west.yml`.
2. Parses `build.yaml` `include:` entries with `yq`/`jq` (both auto-downloaded to `local-build/`).
3. For each board × shield × keymap combo, creates a **fresh temp sandbox** (`mktemp -d`), copies the repo in, and copies `config/` into `zmk/app/config/` so `#include` paths resolve identically to CI.
4. Runs `west build --pristine` passing `-DZMK_EXTRA_MODULES=<boards>;<zmk-pmw3610-driver>;<prospector-zmk-module>` so custom shields and the trackball/display drivers are discovered.
5. Copies the resulting `zmk.uf2` into `firmwares/` under a name derived from `entry_format` (see the `case "$entry_format"` block for the format→dir mapping).

Key mechanics to know when editing the script or build config:
- **Custom shields** live in `boards/shields/` and are picked up via `ZMK_EXTRA_MODULES` — no manual copy.
- **Keymap selection by filename:** the script copies `config/keymaps/<keymap>.keymap` to `<shield_target>.keymap` so ZMK's cmake finds it by exact-name match. This is why keymaps are shared across shields.
- **Stacked shields** (space-separated, e.g. `"charybdis_dongle prospector_adapter"`) pass the full string to `-DSHIELD`; only the first token is used to discover overlay targets.
- **USB-central studio snippet** (`-S studio-rpc-usb-uart`) is applied only to `charybdis_right_bt` (BT central) and `charybdis_dongle` (dongle central), and only when USB logging is off.

## Layout of the config

- `build.yaml` — the build matrix. Edit here to change which formats/keymaps/boards build. Build families: `bt` (Bluetooth split), `dongle_standard_nano` (no-screen dongle), `dongle_prospector_*` (screen dongle, nano or xiao, with/without APDS9960 sensor). Comment entries in/out to include/exclude.
- `config/keymaps/*.keymap` — one file per layout (`qwerty`, `colemak_dh`, `canary`, `focal`, `graphite`). Each `#include`s the shared feature files below. **Keymaps are layout-only; behaviors/combos/macros are shared** — edit the shared files, not each keymap, for cross-layout changes.
- `config/keymap_features/` — shared devicetree: `behaviors.dtsi` (home-row mods, hold-taps, tap-dances), `combos.dtsi`, `macros.dtsi`, `prospector_brightness.dtsi`.
- `config/trackball/charybdis_pointer.dtsi` — all pointer/scroll input-processor config (speed, accel, scroll). The single place to tune trackball feel.
- `config/charybdis/` — shared `.conf` Kconfig fragments; `config/charybdis-layouts.dtsi` — physical layout.
- `config/dongle_prospector/`, `dongle_prospector_layouts/`, `dongle_prospector_themes/` — Prospector display screen configs, layout `.conf`s, and color-theme `.overlay`s, combined via `extra_conf_files`/`extra_dtc_overlay_files` in `build.yaml`.
- `boards/shields/` — custom shield definitions (`charybdis_left/right_bt`, `_dongle`, `charybdis_dongle`, `charybdis_common`, `charybdis_trackball`) with `.overlay`, `.conf`, and `Kconfig.*` files.
- `config/west.yml` — module manifest. Pulls ZMK from `zmkfirmware`, plus `zmk-pmw3610-driver` (trackball sensor) and `prospector-zmk-module` (display) from the `280Zo` fork.

### Prospector-shared keymap guard
Keymaps build for both Prospector and non-Prospector firmware from one file. Prospector-only behaviors (e.g. `pbl` display brightness) are stubbed with `/omit-if-no-ref/` behind `#ifndef CONFIG_SHIELD_PROSPECTOR_ADAPTER` so non-Prospector builds still compile. When adding a Prospector-only behavior, add a matching stub or the non-Prospector build breaks.

### APDS9960 sensor builds
Prospector sensor builds intentionally disable Zephyr's stock APDS9960 driver and enable the Prospector module's replacement (`config/dongle_prospector/dongle_prospector_sensor.conf`):
```conf
CONFIG_APDS9960=n
CONFIG_PROSPECTOR_APDS9960=y
```

## Keymap drawings (`keymap-drawer/`, `scripts/`)

Rendered keymap SVGs/PNGs in the README are generated by `.github/workflows/draw_keymaps.yml` using [keymap-drawer](https://github.com/caksoylar/keymap-drawer) plus `scripts/render_stacked_composite.js` (Playwright composite). Run the pipeline locally with:
```bash
act -W .github/workflows/draw_keymaps.yml --bind --reuse
```
Editing rules (colors, sizing, legends, composite layout) are documented in `keymap-drawer/README.md` — consult it before touching render config. This affects docs images only, not firmware.
