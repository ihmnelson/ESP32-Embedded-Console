# ESP32 Embedded Console

Handheld game/macropad device, roughly halfway between a Stream Deck and a Game Boy. Personal learning project — KiCad hardware design, not yet fabbed.

## Status

Schematic-in-progress. Root sheet is wired, ERC is mostly clean, footprints are not yet fully assigned. Not ready for PCB layout. See `esp32-board-design.md` for the full design log — decisions, part numbers, GPIO budget, and open items, all dated and kept current. That file is the source of truth; this README is just an orientation.

## Design summary

| Area | Decision |
|---|---|
| MCU | ESP32-S3-WROOM-2-N32R16V (32MB flash, 16MB octal PSRAM, native USB, 33 GPIOs) |
| Power | USB-C only for v1 — no battery this rev. TPS62A02NDRL buck for the 3.3V rail |
| Display | Adafruit #1770, 2.8" 240×320 ILI9341, 8-bit i80 parallel, touch dropped |
| Storage | microSD, 4-bit SDMMC, Hirose DM3AT socket |
| Audio | MAX98357A I2S Class-D into an 8Ω speaker (Adafruit #3923) |
| Input | 10× Omron B3F-1000 tactile buttons on an MCP23017 I2C expander, interrupt-driven |
| Firmware target | ESP-IDF + LVGL + FATFS, aiming for retro-go compatibility |

Analog inputs (joysticks, volume pot) are deferred to a later revision — see the design doc for why.

## Repo layout

- `esp32build.kicad_*` — the main project (root sheet + hierarchy)
- `core.kicad_sch`, `power.kicad_sch`, `storage.kicad_sch`, `user_in_out.kicad_sch`, `display_and_audio.kicad_sch` — hierarchical sheets
- `display/` — reference KiCad project for the Adafruit display breakout (footprint/symbol import source)
- `datasheets/` — parts datasheets pulled in for reference (inductors so far)
- `esp32-board-design.md` — the actual design log: decisions, reasoning, GPIO map, BOM, open items
- `CLAUDE_KICAD_SETUP.md` — notes on an abandoned attempt to hook Claude Code into KiCad's IPC API directly; kept as a record, not an open task
- `logo.drawio` — logo sketch

## Tooling

KiCad 10. No firmware yet — this repo is hardware-only for now.
