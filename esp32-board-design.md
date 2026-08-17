---
type: knowledge
tags: [esp32, microcontroller, pcb-design, learning-project, personal]
updated: 2026-08-14
summary: Full design record for a handheld ESP32-S3 games/macropad board (USB-only v1, battery deferred) — module choice, USB-C power, Adafruit 2.8in ILI9341 parallel display (touch dropped), microSD, MAX98357A audio, MCP23017/button comms bus, and the KiCad build state.
---

# ESP32 Board Design

> ## ▶ RESUME HERE (2026-08-14)
>
> **Notes in scope:** this note only.
>
> **Konnect bridge is back.** [[CLAUDE_KICAD_SETUP]]'s "Pending approval" MCP block is no longer
> reproducing — Konnect's `sch_*`/`sch_hierarchy` tools connected and worked directly against the
> real `.kicad_sch` files this session (KiCad itself doesn't need to be running for schematic
> edits). Claude did the remaining wiring directly instead of staying read-only. Still true: KiCad's
> own IPC (needed for live PCB edits) wasn't reachable — KiCad wasn't open. `kicad-cli.exe` also
> isn't on Konnect's PATH yet, so `run_erc`/`run_drc` fail with "Failed to spawn kicad-cli.exe" —
> **fix that PATH before the next fab-readiness check** (likely needs `...\KiCad\10.0\bin` added
> to the environment Konnect's process sees).
>
> **Stopped at:** the display bus is now fully wired end-to-end and verified by direct file/coordinate
> tracing (not just visual): all 12 `core.kicad_sch`↔`display_and_audio.kicad_sch` signals plus
> `user_in_out.kicad_sch`↔`display_and_audio.kicad_sch`'s `display_RST` land pin-to-pin with no
> orphans (`validate_sheet_pins` reports 0 issues, `find_shorted_nets` reports 0). Turned out most of
> this was already done by hand before this session — only the `display_DAT0`–`DAT7` label-shape bug
> (`output`→`input`) and two leftover `display_INT` sheet pins (orphaned since touch was dropped)
> still needed fixing, both done now. The two cosmetic ERC items previously flagged
> (`I2C_SDA`/`SCL` shape, `storage.kicad_sch`'s duplicate `3V3+` shape) are also already resolved —
> both read correctly now. `J1`'s `VCONN` question is moot: the USB-C symbol in use
> (`GCT_USB4085`-style 14-pin reduced footprint) has no `VCONN` pin to begin with.
>
> **#1770 vs #1743 pinout — resolved.** Both product pages link to the *same* Adafruit Learn guide
> (`adafruit-2-dot-8-color-tft-touchscreen-breakout-v2`), and that guide's 8-bit-mode pin list (GND,
> Vin, CS, C/D, WR, RD, RST, Backlite, D0–D7) matches the already-wired `display_and_audio.kicad_sch`
> bus exactly. Good enough to treat the 12-pin bus plan as confirmed; the physical header pin
> *order* on the real #1770 board is still worth a glance against the board in hand before soldering.
>
> **Next:**
> 1. Fix the `kicad-cli.exe` PATH issue, then run a real ERC/DRC pass to confirm clean (the manual
>    connectivity checks this session found nothing, but that's not a substitute for the real tool).
> 2. Open KiCad, eyeball the new root-page wiring once visually to make sure the auto-routed L-bends
>    aren't crossing anything awkwardly.
> 3. Footprint assignment pass is still the top blocker before layout — see
>    [[#2026-08-14 — Footprint-assignment spec, and pre-layout checklist]].
>
> **Don't re-derive:** the GPIO/expander pin assignment (12 direct + 1 expander pin, all verified by
> coordinate-tracing the real `.kicad_sch` files, not guessed) and the smaller-display candidate
> search (#2478 out of stock, cheap SPI-only boards ruled out, bare-panel FPC option ruled out as
> too much extra work) — both fully written up in the sections linked below.
>
> <!-- HANDOFF-COMPLETE -->

**Personal learning project, not Tritium work.** Design record for a handheld device "halfway between a Stream Deck and a Game Boy", built around the `ESP32-S3-WROOM-2-N32R16V` module. Covers what to connect, why each part was chosen, and the traps hit along the way.

> **Portability note (2026-08-07).** This note and its KiCad project (`projects/esp32build/`) currently sit inside the Tritium Work OneDrive vault for convenience, but the content is entirely personal. It's written to be **self-contained** so both can be lifted out to a personal machine without losing context — the only outbound link is [[CLAUDE_KICAD_SETUP]], which lives in the project folder and travels with it. See [[#Related]].

## Contents
- [[#Design summary (start here)]]
- [[#Chip family variants]]
- [[#Module vs. bare chip]]
- [[#WROOM vs. WROVER modules]]
- [[#Minimum circuit (WROOM module)]]
- [[#Powering from USB-C]]
- [[#microSD storage (4-bit SDMMC)]]
- [[#Display choice: 3.5-inch 320×480, 8-bit i80 parallel]]
- [[#2026-08-10 — Panel decision superseded: Adafruit 3.5" TFT Capacitive Touch Breakout (HX8357D)]]
- [[#2026-08-07 — Original panel pick, kept for reference: LCDWIKI MRB3511]]
- [[#GPIO budget (current)]]
- [[#2026-08-10 — Expander decision: MCP23017, interrupt-driven]]
- [[#Analog inputs, and the "I need more pins" pattern]]
- [[#Joystick part: Adafruit #5628 (Joy-Con style) — chosen, deferred to a later revision]]
- [[#Battery: 18650 with power-path charging — DECIDED 2026-08-07]]
- [[#2026-08-10 — Battery deferred to a later revision]]
- [[#Audio (I2S → Class-D, MAX98357A)]]
- [[#Firmware stack (no custom OS needed)]]
- [[#Indicator LEDs]]
- [[#Reference sources]]
- [[#2026-08-07 — Chip variant mismatch gotcha (classic ESP32 vs S3)]]
- [[#2026-08-07 — KiCad USB-C symbol gotcha: Plug vs. Receptacle]]
- [[#2026-08-07 — Schematic build state (first pass)]]
- [[#2026-08-11 — Schematic build state (measured from the files)]]
- [[#Next steps]]
- [[#2026-08-14 — Root sheet wiring spec (drafted, not yet wired)]]
- [[#2026-08-14 — Root sheet wired; PWR_FLAGs added (verified from the files)]]
- [[#2026-08-14 — display_and_audio re-verified: far more complete than this table shows]]
- [[#2026-08-14 — Footprint-assignment spec, and pre-layout checklist]]
- [[#2026-08-14 — L1 switched: XGL3520 → XAL4020 (no footprint import needed)]]
- [[#2026-08-14 — Display bus never reached the MCU/expander: found and specified]]
- [[#2026-08-14 — Panel decision superseded again: Adafruit 3.2" TFT Touchscreen Breakout (ILI9341), touch dropped]]
- [[#Related]]

## Design summary (start here)

**Refreshed 2026-08-10 — read this section fresh, don't trust memory of earlier sessions.** A huge amount changed in the 2026-08-10 session specifically; the table below is the actual current state, not the 08-07 baseline. **Decisions below are unchanged as of 2026-08-11; only the build-state/open-items counts were re-measured that day.**

### Decision register

| Area | Decision | Deciding reason |
|---|---|---|
| **MCU** | ESP32-S3-WROOM-2-N32R16V module | 32 MB flash + 16 MB octal PSRAM; pre-certified module avoids RF design; native USB. Exposes **33 GPIOs**. Considered and rejected an ESP32-P4 add-on for camera/second-screen — see [[#2026-08-10 — P4 / second-screen / camera / foldable — considered, not pursued]] |
| **USB** | USB-C **receptacle** (14P simplified symbol), 5.1 kΩ on CC1 **and** CC2 as separate nets | A cable can insert either way up; only one CC is live per orientation |
| **Port ESD** | USBLC6-2SC6 on D+/D−/VBUS, 33 Ω series on data | User-facing connector |
| **Power in / battery** | **USB-only for v1** — no battery this rev | Simplicity for the first build. See [[#2026-08-10 — Battery deferred to a later revision]]; the BQ24074/18650 and NiMH/2S2P research is kept for when it's revisited. A JST-PH battery connector is the forward-looking pick whenever that happens — [[#Battery connector — forward-looking note, not yet built]] |
| **3.3 V rail** | **TPS62A02NDRL buck** (verified wiring in [[#Chosen buck: TPS62A02NDRL (2A, SOT-563-6) — verified wiring]]) | USB-only means a straight 5V→3.3V buck is enough — no buck-boost needed |
| **Display** | ~~3.5″ 320×480, Adafruit HX8357D #5846~~ **switched 2026-08-14 to 2.8" 240×320, Adafruit #1770 (ILI9341), touch dropped** — still **8-bit i80 parallel** | Sized down after #5846/#1743 felt too big in hand; #1770 keeps the ILI9341 family and the same 12-pin bus plan. See [[#2026-08-14 — Panel decision superseded again: Adafruit 3.2" TFT Touchscreen Breakout (ILI9341), touch dropped]]. One display, flat non-folding shell (like the original Game Boy) — a dual-screen foldable DS-style concept was seriously explored and dropped, see the P4/foldable section linked above |
| **Storage** | microSD, **4-bit SDMMC**, Hirose DM3AT socket | 4× the bandwidth of 1-bit for 3 extra pins. Explicitly kept at 4-bit even under pin pressure — see [[#2026-08-10 — Reclaiming pins without touching SD or display speed]] |
| **Audio** | **MAX98357A** I2S Class-D, **8 Ω** speaker, **Adafruit #3923 ($1.95)** | The S3 has **no DAC at all**; 8 Ω roughly halves current vs 4 Ω. Gain set to **6dB** (direct VDD tie), SD_MODE resistor corrected to **634kΩ to 3.3V** (not the old 1MΩ-to-VDD approximation) — see [[#Chosen part: MAX98357A]]. Volume is a **firmware menu setting via buttons**, not a hardware pot — the analog volume path was dropped along with the ADC, see Analog row below |
| **Buttons** | **10 buttons** on an **MCP23017-E/ML (QFN)** I2C expander, interrupt-driven | 2-3 pins vs a matrix's 6, no ghosting, no diodes. MCP23017 picked over PCF8575 for true push-pull GPIO (needed for LCD_RST/status LED) and interrupt masking — see [[#2026-08-10 — Expander decision: MCP23017, interrupt-driven]]. Status LED and `display_RST` ride the expander's spare I/O at zero direct-GPIO cost; a touch IRQ was also planned there but touch is dropped as of the 2026-08-14 panel swap — see [[#2026-08-14 — Panel decision superseded again: Adafruit 3.2" TFT Touchscreen Breakout (ILI9341), touch dropped]] |
| **Analog (joysticks, volume)** | **Deferred entirely for this rev** — no ADS7828 on the board | Volume moved to a button/menu setting; joysticks confirmed non-essential (retro-go's reference hardware is D-pad + buttons only). Real, verified research kept for a later rev — see [[#Analog inputs, and the "I need more pins" pattern]] and [[#Joystick part: Adafruit #5628 (Joy-Con style) — chosen, deferred to a later revision]] |
| **Wi-Fi** | Yes, module PCB antenna | Forces ADC2 unusable (moot now analog is deferred); antenna keepout still non-negotiable |
| **Camera** | **Not this rev.** If added later, a self-contained SPI camera module (e.g. Arducam Mini, ~$25-36) is the realistic path, not a direct 12-GPIO parallel interface | See [[#2026-08-10 — Reclaiming pins without touching SD or display speed]] and the P4 section |
| **Firmware** | ESP-IDF (FreeRTOS) + LVGL + FATFS, targeting **retro-go** compatibility | **No custom OS** — this is an MCU. [retro-go](https://github.com/ducalex/retro-go) checked directly against source multiple times this session (input keymaps, volume, P4/multi-display support) — see the P4 section for what it can't do |

### Budgets

- **GPIO: 25 of 28 used, 3 spare** (pool grew from 27→28 after reclaiming IO45; see [[#2026-08-10 — Reclaiming pins without touching SD or display speed]]). A second display would cost +1 (a second CS); a direct camera would need ~12 more and doesn't fit even with every reclaim spent — see [[#2026-08-10 — Projection: joystick + second display + camera]].
- **Expander: 12 of 16 used once fully built** — 10 buttons, status LED, `display_RST`. **4 spare** (touch IRQ dropped 2026-08-14 with the panel swap — see [[#2026-08-14 — Panel decision superseded again: Adafruit 3.2" TFT Touchscreen Breakout (ILI9341), touch dropped]]), more if joysticks stay deferred.
- **I2C bus: ~5% utilised** at 100 Hz polling on 400 kHz. Not a constraint.
- **Runtime**: not applicable this rev — USB-only, no battery. Revisit once [[#2026-08-10 — Battery deferred to a later revision]] is un-deferred.
- **Physical**: not yet settled — the ~85×110mm Game-Boy-DMG-sized estimate assumed the now-deferred 18650. Revisit once battery and enclosure are real.

### Running BOM

**Caution: BOM reference designators (R#, C#, U#) are project-global and get reassigned as things are actually placed in KiCad — don't trust old ref numbers below without checking the real schematic files.** This table is a parts-by-block summary, not a wiring source of truth.

| Block | Parts |
|---|---|
| MCU | ESP32-S3-WROOM-2-N32R16V; decoupling per [[#Minimum circuit (WROOM module)]]; EN RC network + reset button; BOOT button doubles as settings button |
| USB | USB-C receptacle; CC1/CC2 5.1kΩ pulldowns (separate nets); USBLC6-2SC6 ESD; 33Ω series on D+/D-; polyfuse |
| Power (v1) | TPS62A02NDRL buck only — no charger, no battery, no buck-boost this rev |
| Display | ~~Adafruit #5846~~ **Adafruit #1770 ($29.95)** — pin-header breakout, `J4`'s connector/footprint choice not yet finalized for this part's form factor; bus/pin plan in [[#2026-08-14 — Display bus never reached the MCU/expander: found and specified]], panel history in [[#2026-08-14 — Panel decision superseded again: Adafruit 3.2" TFT Touchscreen Breakout (ILI9341), touch dropped]] |
| Storage | Hirose DM3AT socket; 5× 10kΩ pull-ups (CMD/DAT0-3); 22Ω series on CLK; VDD decoupling |
| Audio | MAX98357A (TQFN-16); 10µF+0.1µF at VDD; 634kΩ SD_MODE resistor to 3.3V; GAIN tied direct to VDD (6dB, no resistor); Adafruit #3923 speaker, JST-PicoBlade connector |
| Comms/Input | MCP23017-E/ML; I2C pull-ups (4.7kΩ ×2 to 3V3+); 10 tactile switches — **Omron B3F-1000** (100gf, 4.3mm, through-hole, decided for durability under repeated presses) |
| Indicators | 3V3-present LED, VBUS-present LED (both on Power sheet), status LED (moved to expander) |

### Biggest open items

**Verified against the actual KiCad files 2026-08-11 — see [[#2026-08-11 — Schematic build state (measured from the files)]] for the measurements behind these.**

1. **Root page has no wires** — all five sheets have had Import Sheet Pin run now (26 pins total), but nothing on the root page joins them. This is the top blocker: until it's wired, the hierarchy is five islands.
2. **`display_and_audio` is started but incomplete** — connector, amp and speaker header are placed; the MAX98357A's support parts and the full 20-pin display table still need drawing in.
3. **No `PWR_FLAG` anywhere in the project** — ERC cannot pass on VBUS/3V3+ until they're added.
4. **Footprints: 8 of 55 assigned** — every passive, switch, LED and J1 is still bare.
5. **Buttons/joystick physical layout** — switch part and count decided (10× Omron B3F-1000), but the actual controller-shaped arrangement (D-pad left, face buttons right, etc.) hasn't been designed yet.
6. Minor cosmetic cleanup still outstanding: `I2C_SDA`/`I2C_SCL` global label direction on Core (currently `input`, should be `bidirectional`), and storage's duplicate `3V3+` label direction mismatch.

Gating order is **1 → 3 → 4** (a clean ERC plus full footprint coverage is what unlocks PCB work at all); 2 can proceed in parallel.

Full history and reasoning for all of the above lives in [[#Next steps]] and the dated subsections throughout this note.

## Chip family variants

Espressif's "ESP32" name covers several distinct chips — different pinouts, cores, and peripherals, not just speed grades:

| Series | Core / arch | Wireless | Notes |
|---|---|---|---|
| ESP32 (classic) | Dual-core Xtensa LX6/LX7 | Wi-Fi + Bluetooth Classic + BLE | Original chip, most mature ecosystem, no longer getting new features |
| ESP32-S2 | Single-core Xtensa | Wi-Fi only | Native USB OTG |
| ESP32-S3 | Dual-core Xtensa LX7 | Wi-Fi + BLE | Native USB OTG, AI/vector instructions, more GPIOs — common upgrade path from classic, and what this project uses |
| ESP32-C2/C3 | Single-core RISC-V | Wi-Fi + BLE | Cost/power-optimized; C3 is the popular low-cost classic-ESP32 alternative |
| ESP32-C6 | Single-core RISC-V | Wi-Fi 6 + BLE + Zigbee/Thread (802.15.4) | Mesh protocols for Matter/smart-home |
| ESP32-C5 | Single-core RISC-V | Dual-band (2.4+5GHz) Wi-Fi 6 + BLE | Newest C-series |
| ESP32-H2 | Single-core RISC-V | BLE + 802.15.4, no Wi-Fi | Ultra-low-power Thread/Zigbee radio companion chip |
| ESP32-P4 | Dual-core RISC-V | No radio | HMI/vision co-processor, pairs with an S3/C6 for connectivity |

Suffixes on a chip name (e.g. `ESP32-S3R8`, `ESP32-S3FN8`) aren't separate versions — they're packaged SKUs of the same die encoding flash size, PSRAM size/type, and temp grade. Module names (WROOM/WROVER/MINI/PICO) are a further layer on top: ESP32-S3-WROOM-1 is a *module* built around the ESP32-S3 *die*.

### ESP32-S3 SKUs (per Espressif's official datasheet v2.2)

Nomenclature: `ESP32-S3` `[F/N][temp][flash MB]` `R` `[H][PSRAM MB]` `[V]` — F/N = flash present + temp grade (F=high temp, N=normal), R = has in-package PSRAM (H after it = high-temp grade), trailing V = 1.8V external SPI flash/PSRAM instead of default 3.3V.

| Part Number | In-Pkg Flash | In-Pkg PSRAM | Ambient Temp | VDD_SPI | Chip Rev |
|---|---|---|---|---|---|
| ESP32-S3 | — | — | −40~105°C | 3.3V/1.8V | v0.1/v0.2 |
| ESP32-S3FN8 | 8MB Quad SPI | — | −40~85°C | 3.3V | v0.1/v0.2 |
| ESP32-S3RH2 | — | 2MB Quad SPI | −40~105°C | 3.3V | v0.2 |
| ESP32-S3R8 | — | 8MB Octal SPI | −40~65°C | 3.3V | v0.1/v0.2 |
| ESP32-S3R16V | — | 16MB Octal SPI | −40~65°C | 1.8V | v0.2 |
| ESP32-S3FH4R2 | 4MB Quad SPI | 2MB Quad SPI | −40~85°C | 3.3V | v0.1/v0.2 |
| ESP32-S3R8V | — | 8MB Octal SPI | −40~65°C | 1.8V | **EOL** |
| ESP32-S3R2 | — | 2MB Quad SPI | −40~85°C | 3.3V | **EOL** → upgraded to S3RH2 |

Bare `ESP32-S3` (no suffix) has no in-package flash/PSRAM — external SPI flash would need to be wired up separately, so a suffixed SKU (or a WROOM module, which bakes one of these in — see [[#WROOM vs. WROVER modules]]) is the practical choice. Note: same part number can mean chip revision v0.1 *or* v0.2 on the market — check the ESP32-S3 Series SoC Errata doc if revision-specific behavior ever matters.

**Guideline docs and pinouts are per-chip, not shared across the family** — see the CAP1/CAP2 gotcha below, discovered from mixing up classic ESP32 and S3 docs.

## Module vs. bare chip

Don't design around the bare ESP32 die for a first project — it needs a matched RF antenna network and precise crystal layout. Almost everyone, including Espressif's own dev boards, builds around a pre-certified module like the [ESP32-WROOM-32](https://www.espressif.com/en/products/modules) or ESP32-S3-WROOM-1, which already integrates the crystal, flash, and antenna matching network. A custom board just wires up the module's external pins — this is the realistic entry point for a learning project.

### Which schematic actually applies to you

Chip-level guidelines (the bare-chip Hardware Design Guidelines/schematic checklist, and CAP1/CAP2-style pins) and a module's own datasheet cover overlapping ground, but only one applies once a module is chosen. Every module datasheet actually separates this explicitly into two figures — use that split as the rule:

- **"Module Schematics"** section (e.g. WROOM-1/1U datasheet §8) — captioned "reference design of the module": crystal, RF antenna matching network, internal flash/PSRAM SPI wiring. This is Espressif's internal build reference for the module itself — **ignore entirely**, it's sealed inside the part you're buying.
- **"Peripheral Schematics"** section (e.g. §9) — captioned "typical application circuit of the module connected with peripheral components": power decoupling, EN pin RC delay + reset button, boot-strap header, USB D+/D−, JTAG/UART headers. **This is the one to actually build from**, along with the datasheet's own Pin Definitions table and PCB Layout Recommendations (land pattern, antenna keepout).

Net: once using a module, the bare-chip guidelines become background reading only — design from the module datasheet's "Peripheral Schematics" + "PCB Layout Recommendations" sections, not the chip-level checklist.

## WROOM vs. WROVER modules

- **WROOM** — base module line: chip + flash, PCB-trace antenna by default.
- **WROVER** — adds external **PSRAM** inside the module; used historically for classic ESP32 and ESP32-S2 (e.g. `ESP32-WROVER-E` = ESP32 + flash + 8MB PSRAM).
- **`-U`/`-I` suffix** (orthogonal to WROOM/WROVER) — swaps the PCB-trace antenna for a U.FL/IPEX external-antenna connector, e.g. `ESP32-WROOM-32U`.
- **No `ESP32-S3-WROVER` exists** — confirmed against Espressif's module listing. For S3, PSRAM is just a config picked via the WROOM-1 ordering-code suffix, not a separate module family:

| Module | Ordering codes | Flash | PSRAM |
|---|---|---|---|
| ESP32-S3-WROOM-1 | N4 / N8 / N16 | 4/8/16MB | none |
| | N4R2 / N8R2 / N16R2 | 4/8/16MB | 2MB |
| | N4R8 / N8R8 / N16R8 | 4/8/16MB | 8MB |
| | N16R16V | 16MB | 16MB |
| ESP32-S3-WROOM-1U | same pattern, IPEX antenna | — | — |
| ESP32-S3-WROOM-2 | N32R16V (only active SKU) | 32MB Octal | 16MB Octal |
| ESP32-S3-MINI-1 | N8 / N4R2 | 4–8MB | 0–2MB |

`MINI-1` = smaller footprint, fewer exposed GPIOs — useful for space-constrained boards. Don't search for "ESP32-S3-WROVER"; pick a WROOM-1/WROOM-2/MINI-1 ordering code matching the flash/PSRAM needed instead. (`ESP32-S3-WROOM-2-N16R8V` and `-N32R8V` are EOL — `N32R16V` is the only current WROOM-2 SKU, per Espressif's [WROOM-2 datasheet](https://documentation.espressif.com/esp32-s3-wroom-2_datasheet_en.pdf).)

### Project decision: switched to ESP32-S3-WROOM-2-N32R16V

Same 18.0×25.5×3.1mm footprint as WROOM-1, so no change to keepout/board-size planning. **But WROOM-2 only exposes 33 GPIOs vs WROOM-1's 36** — it uses Octal SPI for both flash and PSRAM (vs WROOM-1's typical Quad-flash/Octal-PSRAM), consuming 2 extra internal lines. Same package outline, different pin function map — pull WROOM-2's own Pin Definitions table before finalizing the schematic, don't reuse a WROOM-1 pin assignment. On-board PCB antenna only (no `-U` option exists for WROOM-2). Native USB OTG, USB Serial/JTAG, and SD/MMC host controller are all still present, so earlier guidance on those is unaffected.

**Footprint is identical across flash/PSRAM SKUs within a module line** — confirmed via the [ESP32-S3-WROOM-1/1U datasheet](https://documentation.espressif.com/esp32-s3-wroom-1_wroom-1u_datasheet_en.pdf): one shared Pin Definitions table, Pin Layout diagram, and PCB Land Pattern covers every WROOM-1 SKU from N4 through N16R16VA (all 18.0×25.5×3.1mm). Schematic/PCB footprint doesn't change based on which flash/PSRAM capacity is picked — only which line (WROOM-1 vs -1U vs -2 vs MINI-1) matters for footprint.

**Flash/PSRAM is fixed at purchase, no field upgrade** — it's soldered in-package inside the module. If more capacity is ever needed, that means redesigning around a bigger SKU/line, not adding a chip later. For expandable storage instead (game assets, logs, etc.), the ESP32-S3 has a built-in **SD/MMC host controller** (2 slots) — add a microSD card slot to the board design. No equivalent expansion path exists for RAM/PSRAM on this chip.

## Minimum circuit (WROOM module)

1. **3.3V power rail** — clean 3.3V supply (up to ~500mA peak during Wi-Fi TX bursts), with decoupling caps at the module's supply pins per its footprint.
2. **EN pin (chip enable/reset)** — pull up to 3.3V through an RC network (R=10kΩ, C=0.1µF) so EN comes up *after* the 3.3V rail stabilizes. Add a push-button to GND for manual reset.
3. **Boot-mode strapping pins** — GPIO0 needs a pull-up (pulled to GND at boot to enter the UART bootloader for flashing); a few other GPIOs (e.g. GPIO2, MTDO) have strapping requirements too — full list in the schematic checklist below.
4. **USB-to-UART bridge for programming** — needed on classic ESP32 (no native USB): a USB-serial bridge chip (CP2102 or CH340 are common choices) wired to UART0 (TX/RX), plus the standard two-transistor auto-reset circuit driven by the bridge's DTR/RTS lines so `esptool`/Arduino IDE can auto-enter bootloader mode without a manual button press. **ESP32-S3/S2 have a native USB peripheral** (GPIO19/20 = D-/D+) and can enumerate directly as a USB device — the external bridge chip may not be needed at all on this project; check the S3 schematic checklist for the direct-USB wiring (series resistors, ESD protection) instead.
5. **Antenna keepout** — if using the PCB-trace-antenna module variant, keep copper/ground plane clear under and around the antenna section per the module's footprint drawing.

The module absorbs almost all the hard RF/analog design — a first board is mostly power supply + reset + boot-strap + USB-serial.

### WROOM-2-N32R16V pin-specific circuit

Verified against the module's own Pin Definitions table and Peripheral Schematics guidance (41-pin module; numbers below are WROOM-2-specific, don't reuse WROOM-1 pin numbers):

1. **Power** — Pin 2 (3V3): 10–22µF bulk + 0.1µF ceramic decoupling. GND: pins 1, 40, and EPAD (pin 41, exposed pad) — soldering EPAD to ground plane optional but recommended for RF/thermal performance.
2. **EN (pin 3)** — 10kΩ to 3.3V + 1µF to GND (Espressif's stated recommended values, not the generic 0.1µF above), plus a pushbutton to GND for manual reset. Never leave floating — explicit datasheet warning.
3. **Boot-strap pins** — GPIO0 (pin 27): default weak pull-up, add a "BOOT" pushbutton to GND for forcing download mode. GPIO46 and GPIO3: leave floating, their internal default pulls are already correct for normal operation. GPIO45 (VDD_SPI voltage strap): irrelevant for WROOM-2 — since it always ships with PSRAM, VDD_SPI voltage is fixed at the factory via eFuse regardless of this pin's level (explicit datasheet note); leave floating.
4. **USB (native, no bridge chip)** — IO19 (pin 13) = USB_D-, IO20 (pin 14) = USB_D+, wired straight to the USB connector. Bootloader entry over USB is automatic via `esptool`/Arduino IDE; the BOOT/EN buttons above are the manual fallback.
5. **Antenna keepout** — respect the "Keepout Zone" marked at the top of the module in its pin-layout diagram, no copper/ground on any layer underneath.

## Powering from USB-C

Since D+/D- are already wired for native USB, the same connector can carry both power and programming.

1. **Connector** — a basic USB 2.0-only USB-C receptacle is enough (ESP32-S3 is USB 2.0 full-speed only, no SuperSpeed pairs needed): VBUS, GND, D+, D-, CC1, CC2, shield ground.
2. **CC1/CC2 pull-downs (easy to forget, breaks power if missing)** — 5.1kΩ resistor from each CC pin to GND. This tells a USB-C host/charger "I'm a standard sink device, provide 5V" — without both resistors, many compliant chargers/hosts supply no power at all. No PD negotiation chip needed for plain 5V.
3. **VBUS (5V) → 3.3V regulation** — never feed VBUS directly into the module's 3V3 pin (needs 3.0–3.6V). Keep **both** VCC_5V and VCC_3V3 as separate nets/rails — confirmed from Espressif's own [ESP32-S3-DevKitC-1 schematic](https://dl.espressif.com/dl/schematics/SCH_ESP32-S3-DevKitC-1_V1.1_20220413.pdf): both nets are kept and exposed on its header, since some peripherals (audio amps, backlight LEDs) want 5V for more output power while logic (module, display, SD card) wants 3.3V.
   - Espressif's own reference LDO is an **SGM2212-3.3XKC3G/TR** (small SOT-23-5-style), with 10µF input cap + 10µF output cap — fine for powering just the module, but the DevKitC-1 doesn't drive extra peripherals, so don't copy it blind if adding a display/speaker/etc.
   - For headroom with extra peripherals, **AMS1117-3.3** (1A, cheap, common TO-252/SOT-223) is a solid, ubiquitous step-up choice; note it's linear so it dissipates `(5V−3.3V)×current` as heat (~1.7W at 1A) — if sustained load exceeds ~600–700mA, a small buck converter runs cooler/more efficiently instead.
   - **For this project's actual peripheral load** (per-key RGB LEDs + display + speaker, on top of the module's Wi-Fi peak): likely exceeds AMS1117 territory — a dozen backlit keys alone can add several hundred mA at full brightness. Prefer a small **synchronous buck converter (2–3A class, e.g. AP632xx/TPS62xxx family)** over a linear LDO — verify the specific part's input-voltage floor against USB VBUS (~5V, can sag under load) before finalizing. If realistic peak draw creeps past ~900mA, that brushes the default-USB power ceiling from the USB-C section above — see `## Next steps`.

### Chosen buck: TPS62A02NDRL (2A, SOT-563-6) — verified wiring

> **Back in the design as of 2026-08-10.** Briefly superseded 2026-08-07 when the board went battery-powered; battery support is now deferred to a later revision (see [[#2026-08-10 — Battery deferred to a later revision]]), so this is the live 3.3V rail again for v1.

Confirmed against TI's real datasheet (schematic pin numbers VIN=3, EN=4, GND=1, SW=2, FB=5, OUT=6 checked correct against Table 5-1). This "N" variant has an **OUT sense pin (6) instead of Power-Good** — it's a high-impedance sense input, not a power pin; it ties to the same VOUT net as the inductor/output-cap, it doesn't source current itself.

Topology (TI Figure 8-3, "TPS62A02N Typical Application Circuit"): VIN ← C1 (4.7µF) to GND; EN tied straight to VIN (always-on); SW → L1 (1.0µH) → VOUT net; VOUT net → C2 (22µF) to GND; OUT pin ties to the VOUT net (sense only); VOUT net → R1 → FB pin → R2 → GND (optional C3 feedforward cap in parallel with R1).

**3.3V divider** — TI's equation `R1 = R2 × (VOUT/VFB − 1)`, VFB = 0.6V (verified from electrical characteristics), R2 ≤ 100kΩ for noise immunity: **R1 = 450kΩ, R2 = 100kΩ** (check: 0.6V × (1 + 450k/100k) = 3.3V).

**Parts** (TI's own qualified components, sized for 2A): C1 4.7µF X7R 0805 (Murata GRM21BR71A475KA73L), L1 1.0µH **2A-rated** — ~~Coilcraft XGL3520-102MEC~~ **switched 2026-08-14 to Coilcraft XAL4020-102MEC**, see [[#2026-08-14 — L1 switched: XGL3520 → XAL4020 (no footprint import needed)]] — the 1A-rated Murata part in TI's table is undersized for this load), C2 22µF X7R 0805 (Murata GRM21BZ71A226KE15L, TI's "standard, recommended" choice for VOUT ≥ 1.8V), R1/R2 1% 0603, C3 optional 120pF 0603.
   - Wiring: VIN ← VCC_5V, VOUT → VCC_3V3, 10µF caps at both VIN and VOUT (check the specific IC's datasheet for output-cap ESR requirements), GND → ground plane, EN (if broken out) tied to VIN/5V unless software control over the 3.3V rail is wanted. Reference design's dual-diode VBUS OR-ing is only needed if you have two USB inputs to combine — with a single USB-C connector, VBUS feeds VCC_5V directly.
4. **D+/D- series resistors** — 22–33Ω on the traces between connector and IO19/IO20 (per Espressif's S3 schematic checklist).
5. **ESD protection** — a USB-specific TVS diode array on VBUS and D+/D-, since the connector is user-facing. TVS clamps electrostatic spikes (several kV from static-charged contact) to a safe voltage and shunts the current to ground before it reaches the module's traces.
   - **Chosen part: USBLC6-2SC6** (SOT-23-6). Pinout confirmed straight from KiCad's own symbol library (`Power_Protection.lib`): Pin 1/6 = I/O1 (same net, wire to D-), Pin 3/4 = I/O2 (same net, wire to D+), Pin 5 = VBUS, Pin 2 = GND. Each channel clamps its line to GND.
   - **Placement**: between the connector and the series resistors — `Connector (VBUS/D+/D-) → TVS (clamp to GND) → 22–33Ω series resistor → module/buck`. Confirmed against ST's own USBLC6-2 application note (Figure 13, "USB 2.0 port application diagram"): the Rs resistor sits between the transceiver and the D+/D- net, with the TVS clamping that same net closer to the connector. (That figure also shows USB speed-detection pull-up switching (SW1/SW2/Rpu) and a hub example — neither applies here; ESP32-S3's USB PHY handles speed detection internally.)
   - **PTC fuse** (optional, for the RGB-key overcurrent scenario): VBUS pin → TVS (clamp) → PTC fuse → downstream (buck VIN) — TVS placed right at the pin for the fastest ESD response, fuse after it for sustained-overcurrent protection.
6. **Connector's other pins** — **GND**: tie straight to the board's ground plane (receptacle has multiple GND pins for a solid return path). **SHIELD** (metal shell, not a signal pin): tie to the same GND net — spec requires "all GND wires and shielding... connected together," and it gives ESD/EMI hitting the shell somewhere safe to go. **VCONN**: powers ID chips inside e-marked/active cables; only a power *source* would ever supply it, and only to support such cables — leave it **unconnected (NC)** on a sink-only board like this one.
7. **Power budget ceiling** — without USB-PD/BC1.2 negotiation (extra complexity, likely unnecessary here), expect ~500mA–900mA total from a standard port/charger — budget the module's Wi-Fi peak plus all game peripherals against that.

## microSD storage (4-bit SDMMC)

The expansion path flagged in [[#WROOM vs. WROVER modules]] — in-package flash/PSRAM is fixed at purchase, so a card slot is the only way to add storage for game assets, logs, etc.

### Interface choice

The ESP32-S3's SDMMC host ([SD/MMC controller](https://en.wikipedia.org/wiki/SD_card#Transfer_modes), Espressif's `sdmmc` peripheral) is routed through the **GPIO matrix** — the S3's internal crossbar that maps peripheral signals onto arbitrary pads. Per the [SDMMC Host Driver docs](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-reference/peripherals/sdmmc_host.html): *"The slots are connected to ESP32-S3 GPIOs using the GPIO matrix. This means that any GPIO may be used for each of the SD card signals."* Two slots, each supporting 1-, 4- and 8-line. **This is an S3-specific freedom** — classic ESP32 has slot 1 hard-wired to fixed pads, so don't carry that constraint across (same trap as [[#2026-08-07 — Chip variant mismatch gotcha (classic ESP32 vs S3)]]).

| Mode | Pins | Speed | Verdict |
|---|---|---|---|
| **4-bit SDMMC** | 6 | 40 MHz × 4 lines (High Speed) | **Chosen** — asset/audio streaming wants the bandwidth |
| 1-bit SDMMC | 3 | 40 MHz × 1 line | Fallback if the GPIO budget tightens. Espressif's own ESP32-S3-EYE runs 1-bit — CMD=IO38, CLK=IO39, D0=IO40, no card detect ([esp-bsp `esp32_s3_eye.h`](https://github.com/espressif/esp-bsp/blob/master/bsp/esp32_s3_eye/include/bsp/esp32_s3_eye.h)) |
| SPI | 4 | 1 line | Simplest library support, slowest. No reason to take it here |

Host supports Default Speed (20 MHz), High Speed (40 MHz), and High Speed DDR (4-line eMMC only).

### How the pins actually get assigned (no hardware step)

**The mapping is declared entirely in firmware, at runtime — there is no strapping pin, eFuse, jumper or bootloader setting involved.** The GPIO matrix does the routing internally; you just name the pads:

```c
// ESP-IDF
sdmmc_slot_config_t slot = SDMMC_SLOT_CONFIG_DEFAULT();
slot.clk = GPIO_NUM_39;  slot.cmd = GPIO_NUM_38;
slot.d0  = GPIO_NUM_40;  slot.d1  = GPIO_NUM_41;
slot.d2  = GPIO_NUM_42;  slot.d3  = GPIO_NUM_21;
slot.cd  = GPIO_NUM_47;   // optional
slot.width = 4;
```
```cpp
// Arduino-ESP32 — setPins(clk, cmd, d0[, d1, d2, d3]) before begin()
SD_MMC.setPins(39, 38, 40, 41, 42, 21);
SD_MMC.begin();
```

Two consequences that should drive the design:

- **The commitment is in copper, not silicon.** Freedom is total at schematic time and zero after fab — the traces fix the mapping and the firmware must simply agree. So **choose the pins for routing convenience**: place J2 on the board edge first, see which module pins fall naturally opposite it, then assign. There is no electrical penalty for an "odd-looking" pin set, which is the opposite of how a fixed-pinout MCU behaves. Treat the table below as a known-legal placeholder, not a decision.
- **`setPins()` is mandatory on this module, not optional.** Espressif's [SD_MMC README](https://github.com/espressif/arduino-esp32/blob/master/libraries/SD_MMC/README.md) warns that the library's *default* SD pins are GPIO 33–37, which on the **ESP32-S3-WROOM-2 collide with the module's internal octal flash/PSRAM** and cause a **boot loop**. Consistent with the module only exposing 33 GPIOs (see [[#Project decision: switched to ESP32-S3-WROOM-2-N32R16V]]) — 33–37 aren't brought out at all.

Residual constraints on the choice: not IO0/IO3/IO45/IO46 (strapping), not IO19/IO20 (USB), not 33–37 (don't exist here). All exposed ESP32-S3 GPIOs are bidirectional — there's no input-only block like classic ESP32's GPIO34–39 to avoid. Clock rates are only integer fractions of 40 MHz. If 40 MHz High Speed misbehaves on the real board, the tuning knob is the host's **input delay phase** (`sdmmc_host_set_input_delay`), which shifts when the host samples the card's response; Espressif does *not* document a GPIO-matrix-vs-IO_MUX frequency penalty for S3 SDMMC either way, so don't assume one exists.

### Proposed pin map

Pins already committed: IO0 (BOOT), IO19/IO20 (USB), IO3/IO45/IO46 (strapping, left NC). Putting SD entirely in the **high-numbered block** keeps IO1–IO18 free for peripherals — which matters because **ADC1 is IO1–IO10 and ADC2 (IO11–IO20) is unusable while Wi-Fi is on**, so any future analog (battery sense, volume pot) must live in IO1–IO10.

| Signal | GPIO | WROOM-2 pin |
|---|---|---|
| CLK | IO39 (MTCK) | 32 |
| CMD | IO38 | 31 |
| D0 | IO40 (MTDO) | 33 |
| D1 | IO41 (MTDI) | 34 |
| D2 | IO42 (MTMS) | 35 |
| D3 | IO21 | 23 |
| CD (optional) | IO47 | 24 |

**Accepted tradeoff**: IO39–42 are the MTCK/MTDO/MTDI/MTMS **JTAG** pins, so this forecloses attaching an external JTAG probe. Acceptable because the S3 has a built-in **USB Serial/JTAG** peripheral reachable over the existing D+/D− pair — and Espressif does exactly this on the S3-EYE. Revisit only if hardware-debug-over-probe ever becomes necessary.

Leaves IO1–IO18 and IO48 unallocated for display / audio / buttons.

### Whole-board GPIO budget — SUPERSEDED (SPI-display case)

> **Superseded.** The display went 8-bit parallel and the per-key RGB LEDs were dropped — the live numbers are in [[#GPIO budget (current)]]. This section is retained only as the SPI comparison case, and its FSPI IO_MUX note is still valid if SPI is ever revisited.

Sanity check that the SD map doesn't starve the rest of the design. **Committed**: IO0 (BOOT), IO3/45/46 (strapping, NC), IO19/20 (USB), IO21 + IO38–42 + IO47 (SD), IO48 (status LED). **Free: IO1–IO18, 18 pins.**

| Peripheral | Pins | Suggested |
|---|---|---|
| Display, **SPI** panel (SCLK, MOSI, CS, DC, RST, BLK — no MISO) | 6 | IO12/11/10 + three of IO13–18 |
| RGB key chain (single WS2812-style data line) | 1 | any spare |
| I2S audio (BCLK, LRCLK, DIN) | 3 | any spare |
| Key matrix (4×3) *or* I2C expander | 7 / 2 | remainder |

17 of 18 with a discrete key matrix; 12 with an I2C expander. Fits.

**Put the display on IO9–IO14 if possible** — those are the ESP32-S3's **FSPI IO_MUX** pins (CS0=10, MOSI=11, SCLK=12, MISO=13, WP=14, HD=9). Unlike SDMMC, SPI *does* have an IO_MUX-vs-GPIO-matrix distinction, though only above 40 MHz — per the [SPI master docs](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-reference/peripherals/spi_master.html): *"When an SPI Host is set to 40 MHz or lower frequencies, routing SPI pins via the GPIO matrix will behave the same compared to routing them via IOMUX."* Note this eats IO9/IO10 out of ADC1, leaving IO1–IO8 for analog.


### Socket pinout, and why only some pins get pull-ups

Easy to conflate two different pin sets — only one of them is assignable:

| | Assignable? | Why |
|---|---|---|
| **Module side** — which GPIO carries CLK, CMD, DAT0–3 | **Yes**, freely, in firmware | Peripheral signals routed through the GPIO matrix |
| **Socket side** — J2's 11 pads | **No — fixed by the SD standard** | VDD/VSS/SHIELD/DET aren't peripheral signals at all; they never touch the SDMMC block |

**Pull-ups attach to nets, not to GPIOs.** A 10k on CMD sits between socket pin 3 and whichever module pin was chosen — the resistor follows the *net*, so it's indifferent to the GPIO at the far end. The two concerns are orthogonal.

| Pin | Name | Direction | Function | Needs |
|---|---|---|---|---|
| 1 | DAT2 | bidir | Data bit 2 (4-bit mode); SDIO *read-wait* line | 10k → 3V3 |
| 2 | DAT3/CD | bidir | Data bit 3 (4-bit mode); **mode strap**, see below; /CS in SPI mode | 10k → 3V3 (mandatory) |
| 3 | CMD | bidir | Command/response channel — host commands and card replies share this wire | 10k → 3V3 |
| 4 | VDD | power in | 3.3V to the card | C7 10µF + C8 100nF at the pin |
| 5 | CLK | host → card | Host-driven clock, ≤40 MHz | **No pull-up.** Optional 22Ω series (R13) |
| 6 | VSS | ground | Card ground return | → ground plane |
| 7 | DAT0 | bidir | Data bit 0 — used in *every* mode (1-bit, 4-bit, SPI-MISO). Also the **busy** flag: card holds it low while writing | 10k → 3V3 |
| 8 | DAT1 | bidir | Data bit 1 (4-bit mode); SDIO *interrupt* line | 10k → 3V3 |
| 9 | DET_B | switch | One side of the mechanical detect switch | → IO47 + 10k → 3V3 (R14) |
| 10 | DET_A | switch | Other side of the same switch | → GND |
| SH | SHIELD | mechanical | Metal shell | → GND |

**Why those five and not CLK.** CMD and DAT0–3 are **bidirectional and go high-impedance between transactions** — both ends tri-state at points during card identification, before the host even knows what's attached. A floating CMOS input is undefined and will oscillate on coupled noise, so the card reads garbage. The pull-up pins the idle state to logic 1, which is also the SD bus's own idle/stop symbol. **CLK is never bidirectional and never tri-stated** while the bus runs — the host drives it push-pull, always — so a pull-up there would only burn current and slow the rising edge, the opposite of what 40 MHz wants.

**DAT3 is a mode strap on top of that.** At the first clock after power-up the card samples DAT3//CS to pick its protocol: **high → SD mode, low → SPI mode**. Miss the resistor and the card silently comes up as an SPI device while the SDMMC driver talks SD at it — presents as a dead card, not as a wiring error.

**Card detect** (pins 9/10) is just a mechanical microswitch, electrically unrelated to the SD bus. The DM3AT's is **normally-open**: slot empty → open → pull-up wins → GPIO reads **HIGH**; card seated → closed to GND → GPIO reads **LOW**. Debounce in firmware, insertion chatters. Worth confirming the polarity against the switch timing chart in the [DM3 series datasheet](https://www.hirose.com/product/p/CL0609-0031-0-00) — not fatal either way, since firmware can invert.

### Connection list (draw from this)

Module pin numbers verified against the symbol actually in the schematic (`RF_Module:ESP32-S3-WROOM-2`, 41 pins).

| Net | Socket J2 | → | Module U1 | GPIO | Pull-up |
|---|---|---|---|---|---|
| `SD_CLK` | pin 5 (CLK) | via R13 22Ω | **pin 32** | IO39 | none |
| `SD_CMD` | pin 3 (CMD) | direct | **pin 31** | IO38 | R8 10k → 3V3 |
| `SD_D0` | pin 7 (DAT0) | direct | **pin 33** | IO40 | R9 10k → 3V3 |
| `SD_D1` | pin 8 (DAT1) | direct | **pin 34** | IO41 | R10 10k → 3V3 |
| `SD_D2` | pin 1 (DAT2) | direct | **pin 35** | IO42 | R11 10k → 3V3 |
| `SD_D3` | pin 2 (DAT3/CD) | direct | **pin 23** | IO21 | R12 10k → 3V3 |
| `SD_CD` | pin 9 (DET_B) | direct | **pin 24** | IO47 | R14 10k → 3V3 |

Non-signal pins: J2.4 (VDD) → `3V3+` with **C7 10µF + C8 100nF** to GND at the pin; J2.6 (VSS), J2.10 (DET_A) and J2.SH (SHIELD) all → `GND`.

```
                        3V3+ ─────┬────┬────┬────┬────┬────┬─────┐
                                  │    │    │    │    │    │     │
                                 R8   R9   R10  R11  R12  R14   C7 ═╦═ C8
                                 10k  10k  10k  10k  10k  10k    10µ ║ 100n
                                  │    │    │    │    │    │     │  ║
   ESP32-S3-WROOM-2 (U1)          │    │    │    │    │    │    GND GND
   ┌──────────────┐               │    │    │    │    │    │
   │  pin 32 IO39 ├──[R13 22Ω]────┼────┼────┼────┼────┼────┼──── J2.5   CLK
   │  pin 31 IO38 ├───────────────┴────┼────┼────┼────┼────┼──── J2.3   CMD
   │  pin 33 IO40 ├────────────────────┴────┼────┼────┼────┼──── J2.7   DAT0
   │  pin 34 IO41 ├─────────────────────────┴────┼────┼────┼──── J2.8   DAT1
   │  pin 35 IO42 ├──────────────────────────────┴────┼────┼──── J2.1   DAT2
   │  pin 23 IO21 ├───────────────────────────────────┴────┼──── J2.2   DAT3
   │  pin 24 IO47 ├────────────────────────────────────────┴──── J2.9   DET_B
   │              │
   │  pin  2 3V3  ├──────────────────── 3V3+ ─────────────────── J2.4   VDD
   │  pin  1 GND  ├──────────────────── GND ──┬───────────────── J2.6   VSS
   │  pin 40 GND  ├───────────────────────────┼───────────────── J2.10  DET_A
   │  pin 41 EPAD ├───────────────────────────┴───────────────── J2.SH  SHIELD
   └──────────────┘
```

R13 splits CLK into two nets (`U1.32 → R13.1`, `R13.2 → J2.5`); every other signal is a straight two-pin net with a pull-up hanging off it. Whole block = 1 socket + 7 resistors + 2 caps.

**Indicator LED nets** (see [[#Indicator LEDs]] for the reasoning):

| Net | From | Through | To |
|---|---|---|---|
| Power-good | `3V3+` | R15 1kΩ → D1 anode | D1 cathode → `GND` |
| VBUS present | `VCC_5V` | R16 2.2kΩ → D2 anode | D2 cathode → `GND` |
| User/status | U1 **pin 25** (IO48) | R17 1kΩ → D3 anode | D3 cathode → `GND` |

D3 is active-high — drive IO48 high to light it; ~1.3 mA at 1kΩ off 3.3V, well inside the GPIO's source capability.

### Parts and circuit

- **J2 — socket**: symbol `Connector:Micro_SD_Card_Det_Hirose_DM3AT`, footprint `Connector_Card:microSD_HC_Hirose_DM3AT-SF-PEJM5`. Both ship in the stock KiCad 10 libraries (verified against the local install, no custom library needed). The [Hirose DM3AT-SF-PEJM5](https://www.hirose.com/product/p/CL0609-0031-0-00) is **push-push** (spring eject) with a card-detect switch and reverse-insertion protection — the right feel for a handheld. Symbol pinout: `1 DAT2 · 2 DAT3/CD · 3 CMD · 4 VDD · 5 CLK · 6 VSS · 7 DAT0 · 8 DAT1 · 9 DET_B · 10 DET_A · SH SHIELD`.
- **R8–R12 — 5× 10kΩ to 3V3** on CMD, DAT0, DAT1, DAT2, DAT3. Espressif's [SD pull-up requirements](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-reference/peripherals/sd_pullup_requirements.html) is explicit: *"the CMD and DATA (DAT0 - DAT3) lines of the SD bus must be pulled up by 10 kΩ resistors,"* and the driver docs add *"the internal pullups are insufficient however, please make sure external pullups are connected."* **No pull-up on CLK.**
  - DAT1 and DAT2 need pull-ups even though they look idle in 1-bit mode — DAT1 is the SDIO interrupt line and DAT2 is read-wait.
  - The **DAT3 pull-up is what keeps the card out of SPI mode** at power-on. Omitting it is a classic silent failure. It stays on the board even if the design later drops to 1-bit mode — Espressif's SD_MMC README is explicit: *"even if card's D3 line is not connected to the ESP chip, it still has to be pulled up."*
  - Espressif warns the *"pullup and pulldown requirements of SD and strapping may conflict"* — none of the chosen pins are strapping pins, so this is clear.
- **R13 — 22Ω series on CLK**, at the module end. Damps ringing at 40 MHz; can be DNP'd.
- **C7 10µF + C8 100nF** at socket VDD (pin 4), as close as layout allows. Cards are bursty (~100–200 mA typical, higher spikes at init).
- **Grounds**: VSS (6) and SHIELD (SH) both to the ground plane.
- **Card detect**: DET_A (10) → GND, DET_B (9) → IO47 with R14 10kΩ pull-up to 3V3. Optional — firmware can poll for a card instead, which is what the S3-EYE does.
- **U4 — ESD array (optional, recommended)** on CMD/CLK/D0–D3. Same argument as the USB port in [[#Powering from USB-C]]: the slot is finger-accessible. A low-capacitance 6-channel array (TPD6E05U06 class), or two `Power_Protection:PESD3V3L4UF` parts already in the stock libraries.

### Layout notes (for when the PCB starts)

- CLK is the fast net — short, over solid ground, no adjacency to the RGB-key data chain.
- Hold CMD/D0–D3 within ~10 mm of CLK's length for 40 MHz High Speed.
- Keep the pull-up pack (R8–R12, R14) together **near the socket**, short stubs off each net — not scattered across the board.
- C7/C8 hard against J2 pin 4. R13 at the **module** end of CLK, since its job is damping the driver's edge.

### Power budget impact

An SD card's ~100–200 mA sits on top of the module's Wi-Fi peak and the LED keys. The TPS62A02 (2 A) has ample room — the binding constraint remains the ~900 mA default-USB ceiling from [[#Powering from USB-C]], so the card belongs in that tally.

## Display choice: 3.5-inch 320×480, 8-bit i80 parallel

Target was "about a Game Boy screen, ideally 2″ × 3″". The standard **3.5″ 320×480 (HVGA)** panel class lands almost exactly on it: **active area 73.44 × 48.96 mm = 2.89″ × 1.93″**, 3:2 aspect (the same proportions as a Game Boy Advance).

### Why parallel, not SPI

320×480 = 153,600 pixels per frame, and at that count the interface decides whether the thing is playable:

| Interface | Full-frame rate | Pins | Notes |
|---|---|---|---|
| SPI, **ILI9488** | **~5–15 fps** | 6 | The killer: the [ILI9488 cannot do RGB565 over SPI](https://github.com/Bodmer/TFT_eSPI/discussions/2153) — it's **RGB666, 3 bytes/pixel**, ~460 KB per full frame. [TFT_eSPI](https://github.com/Bodmer/TFT_eSPI) also declines to DMA 18-bit displays, since converting 16→18-bit costs more CPU than DMA saves |
| SPI, **ST7796S** | ~10–25 fps | 6 | Does proper RGB565 (2 bytes/pixel). Better, still not good |
| **8-bit i80 parallel** | **~42 fps** | ~13 | Community benchmarks on ESP32-S3 at 40 MHz report ~42 fps for *both* controllers in 8-bit mode — the controller stops mattering once off SPI |

**Chosen: 8-bit i80 parallel.** The ESP32-S3's **LCD_CAM** peripheral drives it with DMA, and the N32R16V's 16 MB octal PSRAM holds a framebuffer trivially (320×480 RGB565 = 300 KB). 42 fps is a *floor* — retro-style rendering redraws dirty rectangles, not whole frames.

Treat the 42 fps as a reported community benchmark, not a datasheet guarantee. The SPI-vs-parallel order-of-magnitude gap is well documented; the exact number isn't.

### Why not 16-bit i80 — it buys literally nothing

Two independent reasons, and the second is decisive:

1. **It doesn't fit.** 20 pins (D0–D15, WR, DC, RST, BL) against a 27-pin pool leaves nothing for SD + audio + buttons — over by five.
2. **It's no faster on this chip.** Per the [esp_lcd i80 docs](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-reference/peripherals/lcd/i80_lcd.html): *"When bus_width is 8, the PCLK frequency is recommended to be less than 80 MHz. When bus_width is 16, the PCLK frequency is recommended to be less than 40 MHz."* 8-bit × 80 MHz = **80 MB/s**; 16-bit × 40 MHz = **80 MB/s**. Identical bandwidth, for 8 more pins.

**8-bit isn't a compromise here, it's the correct answer.** At 80 MB/s a 320×480 RGB565 frame is 3.8 ms of bus time, so the bus is nowhere near the limit — the ~42 fps community figure was measured at 40 MHz and there is real headroom above it.

The refresh-rate levers that actually pay here are firmware-side:
- **Dirty-rectangle redraw.** 42 fps is the *full-screen* figure. Retro-style games change a small fraction of the screen per frame.
- **Render below native and scale on blit.** The panel is 320×480; a Game Boy is 160×144 and a GBA 240×160. Drawing into a small PSRAM buffer and scaling up cuts the per-frame pixel push several-fold and arguably looks *more* authentic.

### What this newly adds to the board

- **Backlight driver — not yet in the design at all.** A 3.5″ panel's backlight is several white LEDs, and the topology is panel-specific: **series** strings need 12–25 V and therefore a **boost LED driver**; **parallel** strings (~3.2 V each) need current ballast or a constant-current sink off the 5 V rail. Either way it's roughly **100–180 mA at 5 V**, landing directly on the ~900 mA ceiling from [[#Powering from USB-C]]. Budget one PWM GPIO for brightness (the BL pin above).
- **FPC connector.** Bare panels terminate in a 0.5 mm-pitch flex tail, typically 40-pin — a footprint *and* a mechanical mounting problem, not just a schematic symbol.
- **Ordering gotcha**: these panels select SPI vs parallel via **IM0/IM1/IM2 strap pins** on the flex tail, and plenty of cheap 3.5″ modules ship hard-strapped for SPI. Confirm the panel exposes the IM pins, or is already configured for 8080 8-bit, **before** committing the footprint.
- **Board size**: 73 × 49 mm of active area puts a floor of roughly 85 × 60 mm on the PCB before the key field — expect ~85 × 110 mm overall, i.e. genuinely Game Boy sized. Sanity-check against the intended enclosure.

### 2026-08-10 — Panel decision superseded: Adafruit 3.5" TFT Capacitive Touch Breakout (HX8357D)

**Final decision — replaces the LCDWIKI MRB3511 below.** Same 3.5″ 320×480 class, but chosen specifically for documentation quality after the MRB3511's mechanical-drawing gap became a real blocker for standoff placement. **$39.95** (product #5846) — same price whether resistive or capacitive touch, Adafruit doesn't sell a bare no-touch variant of this board at all.

- **Controller: HX8357D** (not ILI9488/ST7796 like the MRB3511) + **FT5336** capacitive touch (I2C, address `0x38`).
- **Real mounting hole data, pulled directly from Adafruit's own Eagle CAD source file** (`Adafruit 3.5 inch 480x320 Capacitive Touch Display.brd`, on GitHub — not measured, not estimated): **4 holes, 3.0mm diameter, plated**, at (±30.48mm, −40.64mm) and (±30.48mm, +50.8mm) relative to board center — 60.96mm horizontal spacing, 91.44mm vertical. Lands exactly on a 1.2"/1.6"/2.0" grid. This is the mechanical certainty the MRB3511 never had.
- **8-bit mode is a single solder jumper** (IM2, top-center of the board) — close it for 8-bit, leave open (default) for SPI. Much simpler than the MRB3511's dual-resistor R8/R16 rework.
- **Onboard microSD slot exists but isn't the storage path** — it's SPI-mode only (slower than the 4-bit SDMMC already designed via the DM3AT socket on the Storage sheet). Leave it unused, don't wire it.
- Backlight and charge-pump caps are handled on the board, same win the MRB3511 offered.
- **JLCPCB reality check still applies unchanged** — the module itself never goes through assembly regardless of which board; only the mating header does.

#### Connector: JP1, `Conn_01x20` in KiCad — full pin-by-pin confirmed from the real Eagle file

**Not a branded part number to search for in KiCad** — this is a standard single-row, 20-position, 2.54mm-pitch pin header (Eagle package `1X20_ROUND`, breadboard-compatible). Symbol: search `Conn_01x20`. Footprint: stock `Connector_PinHeader_2.54mm` library, no custom footprint needed (unlike the joystick's 2mm-pitch part).

The board actually has **two** of these headers — JP1 (8-bit + touch side) and JP2 (SPI side, unused). Confirmed which is which and the exact pad-to-net mapping directly from the `.brd` file's `<signal>` definitions, not a photo read:

| Pin | Eagle net | Wire to |
|---|---|---|
| 1 | GND | GND |
| 2 | +5V | `5V+` |
| 3 | LCD_CS_5V (CS) | Direct MCU GPIO — timing-critical |
| 4 | LCD_RS_5V (C/D) | Direct MCU GPIO |
| 5 | LCD_WR_5V (WR) | Direct MCU GPIO |
| 6 | LCD_RD_5V (RD) | Tie **HIGH** → `3V3+` |
| 7 | LCD_RST_5V (RST) | **MCP23017 expander spare pin** — per the pin-reclaim plan, not a direct GPIO |
| 8 | LCD_LITE (backlight PWM) | Direct MCU GPIO |
| 9 | CTP_IRQ | **MCP23017 spare pin** (decided 2026-08-10) — rides the existing `IO_EXP_INT` line back to the MCU, same zero-direct-GPIO pattern as the status LED/LCD_RST. Not timing-critical, so the expander's own interrupt-aggregation is fine for it. Touch itself still isn't a planned input method, but wiring the IRQ costs nothing and keeps the option open without hardware rework later |
| 10 | CTP_SCL_5V | Shared `I2C_SCL` bus |
| 11 | CTP_SDA_5V | Shared `I2C_SDA` bus |
| 12 | GND (2nd) | GND — don't skip this one, it's placed next to the data bus for return-path integrity |
| 13–20 | LCD_DATA0_5V – LCD_DATA7_5V (D0–D7) | Direct MCU GPIOs ×8, D0=LSB/D7=MSB |

`_5V` suffixes just mean those pins are level-shifted to tolerate up to 5V (onboard 74LVX245 buffers) — 3.3V GPIOs are well within range, nothing extra needed.

**Direct MCU GPIO total: 12** (CS, C/D, WR, D0-7, Backlite) — identical to the MRB3511's budgeted cost, so this swap doesn't change [[#GPIO budget (current)]] at all.

### 2026-08-07 — Original panel pick, kept for reference: LCDWIKI MRB3511

**Superseded above.** [LCDWIKI MRB3511](https://www.lcdwiki.com/res/MRB3511/3.5inch_8&16BIT_Module_MRB3511_User_Manual_EN.pdf) — 3.5″ 320×480, ILI9488 driver, active area 48.96×73.44mm (matches the target almost exactly). Confirmed via the real user manual + schematic (not just the product-page blurb).

- **8-bit mode needs a rework, doesn't ship that way**: default is 16-bit (R16=0Ω populated, R8 open). Per the schematic and manual: **solder R8 with 0Ω, remove R16** → selects 8-bit, DB0–DB7 only. Exactly the "ships hard-strapped" gotcha flagged above — confirmed real on this specific part, with a documented fix.
- **This is a "module with PCB backplane," not a bare panel + FPC** — a real, deliberate tradeoff, not the original plan:
  - **Backlight driver — resolved.** The backplane has its own transistor backlight switch (Q1/S8050 + R3/R4 per the schematic) — just drive its `BL_CTR` pin high (or PWM) from a spare GPIO. The "not yet in the design at all" item above is now moot for this panel.
  - **Charge-pump caps — resolved.** Handled on the backplane; not something to design ourselves.
  - **Connector is a 34-pin header, not a 0.5mm FPC tail.** Easier to source/prototype with, but bulkier than the original FPC plan — board-space tradeoff, not free.
  - Bundled **GT911 capacitive touch** (own I2C pins: SDA/SCL/INT/RST) — unused unless we want it, safe to leave unconnected.
  - Power: accepts 3.3V or 5V (onboard LDO), but the manual itself recommends 3.3V — 5V generates more heat and "affects module life" per their own wording.
- **JLCPCB reality check**: the module itself is never something an assembly house places — not this one, not any display module, regardless of connector type. It's a hobbyist breakout board, not an LCSC-catalog part, and glass panels never go through reflow/pick-and-place anyway; they're always hand-plugged in after assembly. What actually goes on the PCBA order is just the **mating header** (a bog-standard 2.54mm THT part) — everything else is a normal SMT+THT assembly job, then the display module presses into that header by hand once the board is back. Zero soldering skill needed for the display step itself.
- **Why it got dropped**: no mechanical/outline drawing was ever found for the module PCB itself, despite checking the user manual, the vendor's "structural engineering drawing" download (turned out to be for the *touch sensor glass*, not the backplane), and the schematic (electrical only). That gap was the deciding reason to switch to the Adafruit board above, which has real Eagle-file-verified mounting hole coordinates.

### Module-side pin map

Free pool after SD is IO1–IO18 plus IO47/IO48 = 20 pins. Same caveat as the SD map — firmware-declared, so treat it as a known-legal placeholder and finalise after the FPC connector is placed.

| Signal | GPIO |
|---|---|
| LCD D0–D7 | **IO1–IO8** (contiguous — much easier to route as a bus) |
| LCD WR (PCLK) | IO9 — write strobe, data latched on the rising edge |
| LCD DC (RS) | IO10 — data/command select |
| LCD CS | IO11 |
| LCD RST | IO12 — active-low hardware reset |
| LCD BL | IO13 — PWM to the backlight *driver*, not to the panel |
| I2S BCLK / LRC / DIN | IO14, IO15, IO16 |
| I2C SDA / SCL | IO17, IO18 |
| Status LED | IO47 |
| **Spare** | IO48, plus IO45 / IO43 / IO44 in reserve |

**Tradeoff**: IO1–IO10 is ADC1, so this consumes every analog-capable pin. Fine for the current design (no battery, no pots), but it forecloses an analog volume wheel or battery gauge without reshuffling.

### What the panel needs beyond the data bus

Underestimated part of a bare-panel design. **The exact FPC pinout is panel-specific** — pin counts and orderings aren't standardised across 3.5″ panels — so J3's symbol has to come from the chosen panel's datasheet. Regardless of which panel, expect all of:

1. **IM0/IM1/IM2 straps** — resistors to 3V3 or GND on the flex tail selecting 8080-8bit. Values from the controller datasheet's interface-select table.
2. **Multiple supply rails** — typically VDD (logic), VDDI (I/O) and VCI (analog); often all 3.3 V, but not always. Each needs its own decoupling.
3. **Driver-IC charge-pump capacitors** — ILI9488/ST7796 generate VGH/VGL/VCOM/DDVDH internally with an on-chip boost, needing roughly **6–10 external caps** on dedicated flex pins. The datasheet's reference application circuit specifies them. **The most commonly missed part of a bare-panel design — the panel simply won't light without them.**
4. **Backlight driver** — separate LEDA/LEDK flex pins, unrelated to the logic rails. See above.
5. **FPC connector** — 0.5 mm pitch, likely 40-pin, and the **contact side (top vs bottom) must match the flex** or the panel physically won't mate.

**Order of operations**: pick the panel → get its datasheet → draw J3 from the flex pinout with IM straps, decoupling and charge-pump caps → wire the bus to the map above → design the backlight driver from its LED spec. Step one unblocks the other four.

### 2026-08-10 — P4 / second-screen / camera / foldable — considered, not pursued

A long detour: whether to redesign around a foldable, dual-screen (DS-style) device, possibly with a camera and an ESP32-P4 or Raspberry-Pi-class second chip to support it. **Decided against, for this rev** — staying on the single ESP32-S3, one display, flat non-folding shell (like the original Game Boy). Kept here since the research is real and could matter for a genuine v2:

- **ESP32-P4**: dual-core RISC-V up to 400MHz, no wireless radio at all (pairs with an S3/C6 for that), but has genuine **MIPI-CSI with an integrated ISP** and **MIPI-DSI** — real camera/display hardware the S3 lacks. ~$5-6 bulk chip cost, ~24mA typical active / ~112mA doing real camera+vision work — not prohibitively expensive or power-hungry on its own.
- **retro-go breaks in a two-chip or multi-display setup.** Checked its actual source, not assumed: `rg_input.h`'s keymaps and the whole `rg_display` pipeline assume one chip does everything — reads buttons, renders, and drives the display via its own local LCD_CAM peripheral. No concept of "render here, display over there on a different chip," and no P4 support mentioned anywhere in the repo. Adding a P4 for display/camera would mean writing a custom display backend yourself, not lightly modifying something that already supports it.
- **A second display is cheap in pins** (+1 GPIO for a second CS, sharing the existing 8-bit bus — see [[#2026-08-10 — Projection: joystick + second display + camera]]) but the panel chosen for the *first* screen (MRB3511) uses a **rigid pin header, not a flex cable** — wrong connector for a hinge-crossing lid screen regardless of pin budget. A second, different-sized panel for the body was never picked (open, if this comes back).
- **A hinge-crossing cable is a real, standard solution** (same idea as laptop screens) but needs a genuinely dynamic-flex-rated FFC, not just any ribbon, and the hinge mechanism needs physical clearance designed in for the cable's width.
- **Camera, direct on the S3, doesn't fit**: a real parallel camera interface needs ~12 GPIOs, and even every pin-reclaim trick found only frees ~5-7. The only path that fits is a **self-contained SPI camera module** (e.g. Arducam Mini, ~$25-36) with its own onboard capture+compression — not a bare sensor, and not a DIY coprocessor either.
- **Actual Nintendo DS game emulation (not just a DS-*shaped* device) is not feasible on any ESP32 variant, full stop** — not a pins-or-power problem, a raw compute one. DraStic needs ARMv7a+NEON and 256MB+ RAM; melonDS needs a 64-bit CPU outright. Neither the S3 (32-bit Xtensa) nor the P4 (32-bit RISC-V, no NEON-equivalent) clears that bar. Real DS emulation lives on application processors (phones, Raspberry Pi, Rockchip/Allwinner-based retro handhelds) — a genuinely different chip category, not a faster microcontroller.
- **If a Pi-class rebuild is ever seriously considered**: a bare Raspberry Pi Zero 2 W ($15) already clears both DraStic's and melonDS's requirements outright. Going the "custom carrier board" route doesn't require the exact Broadcom chip (not sold standalone — that's why Compute Modules exist) — Rockchip-based CM4-pin-compatible modules (Orange Pi CM4 ~$23-47, Radxa CM3 ~$25-90) are real, cheaper alternatives that pre-integrate RAM (no DDR routing needed on the carrier) and can reuse the large ecosystem of published CM4 carrier board designs. Total parts cost for a Pi-class dual-screen build was ballparked at $105-350+ depending on module/display choices — notably more than the sub-$100 ESP32 build, and Linux/RetroArch-class images already bundle DraStic/melonDS, a huge software-effort difference from retro-go.

## GPIO budget (current)

Supersedes [[#Whole-board GPIO budget — SUPERSEDED (SPI-display case)]], kept only for comparison. **Pool: 28** — 33 exposed GPIOs minus BOOT (IO0), USB (IO19/20), and two strapping pins (IO3/46) — **IO45 reclaimed 2026-08-10** (see [[#2026-08-10 — Reclaiming pins without touching SD or display speed]]), no longer subtracted.

**2026-08-10 — reconciled against the actual schematic files, not just this table.** Two corrections: SD card detect (`sd_detb`) is actually wired as a real hierarchical MCU signal, not dropped as the table previously said — the "dropped" note was stale. And the status LED has actually been moved onto the MCP23017's spare I/O (per [[#2026-08-10 — Reclaiming pins without touching SD or display speed]]), so it no longer costs a direct GPIO.

| Block | Pins | Notes |
|---|---|---|
| Display, 8-bit i80 | **12** | D0–D7, WR, DC, **CS**, BL. RST moved to the expander (planned — display sheet not yet drawn). **RD tied high** permanently — the panel is never read back |
| microSD, 4-bit | **7** | IO39–42 + IO21, **+ card detect (`sd_detb`)** — actually wired, not dropped |
| I2S audio | **3** | BCLK, LRCLK, DIN — see [[#Audio (I2S → Class-D, MAX98357A)]] |
| I2C bus (SDA, SCL) | **2** | Shared bus — MCP23017, ADS7828 both ride these for free |
| Expander interrupt | **1** | `IO_EXP_INT` |
| Status LED | **0** | Moved to the expander |
| **Total** | **25 / 28** | **3 spare** |

**CS costs a pin — corrected 2026-08-07.** An earlier draft assumed CS could be strapped low as the only device on the bus. The i80 bus config takes `cs_gpio_num`, and unlike `reset_gpio_num` it is not documented as accepting `-1`, so budget the GPIO. Reclaim it only if `-1` turns out to be accepted.

### 2026-08-10 — Projection: joystick + second display + camera

| Addition | Real MCU-GPIO cost | Why |
|---|---|---|
| Joystick (back in scope) | **0** | Axes ride the ADS7828 (already on the I2C bus), clicks ride the MCP23017 spares — same zero-GPIO pattern as everything else on the bus |
| Second display | **+1** | Just a second CS — RST can go on the expander for both screens (not timing-critical), but CS has to be a real, fast direct GPIO; the i80 driver toggles it in sync with the parallel bus, too fast for an I2C-expander pin to keep up with |
| Camera | **+~12** | 8-bit data + PCLK + VSYNC + HREF + XCLK — no way to offload this to the expander, it's a real-time parallel video feed, not a slow status signal |

**Joystick + second display fit easily**: 3 spare → 2 spare, comfortably positive, no further reclaiming needed.

**Camera does not fit**, even spending every reclaim identified so far (including the IO43/44 UART0 reserve, +2 more): 3 + 2 (UART0) − 1 (2nd display) = 4 spare, still **8 short** of the ~12 a direct camera interface needs. The only path that actually fits the budget is the coprocessor-with-compression approach from [[#2026-08-10 — Reclaiming pins without touching SD or display speed]] — relaying compressed stills/low-fps video over a ~4-pin SPI link to the main chip, not a direct 12-pin camera interface on the main MCU.

### Where to find more pins

In rough order of how cheap they are:

1. **Hang it on I2C instead.** The general pattern, and the one that scales — see [[#Analog inputs, and the "I need more pins" pattern]]. Every I2C device costs zero GPIOs.
2. **The I2C expander is a pin bank, not just a button reader.** An MCP23017 has 16 I/O and buttons need 9 — **7 spare, costing zero GPIOs.** Move anything slow onto it: status LED, LCD RST (a one-shot; just bring I2C up first), SD card detect, charger status outputs, power-button sense, joystick click switches. Moving the status LED and LCD RST alone takes the budget from 2 spare to 4. **Nothing timing-critical** — no I2S, no bus signals.
3. **Reshuffle to protect ADC1.** ADC1 is IO1–IO10 and the map above spends all of it on the LCD bus. Move the data bus up to **IO9–IO16**, control signals to IO17/IO18/IO47/IO48, I2S and I2C into IO3–IO8, and **IO1/IO2 stay free as analog inputs**. Costs nothing but redrawing.
4. **IO45 is genuinely free on WROOM-2** — its VDD_SPI strap function is irrelevant because PSRAM voltage is fixed by eFuse on this module (see [[#WROOM-2-N32R16V pin-specific circuit]]).
5. **IO43/IO44 (UART0)** if you give up a serial-console fallback.
6. **Drop SD to 1-bit** for 3 more — last resort, hurts asset load times.

### 2026-08-10 — Reclaiming pins without touching SD or display speed

Prompted by wanting headroom for a possible second display (DS-style dual-screen) without sacrificing SD/display performance. **Decided: execute items 2 and 4 now, hold item 5 in reserve.**

- **Move the status LED (D3/R17) and LCD_RST onto the MCP23017's spare I/O** — both already flagged above as candidates, now actually being executed rather than just noted. **+2 real GPIOs**, zero added parts (the expander pins are already there, unused).
- **IO45** — already free, claim it. **+1 GPIO.**
- **IO43/IO44 (UART0)** — held in reserve, not yet spent. Available if needed (native USB Serial/JTAG is already the primary console/programming path, so UART0 is a fallback, not the main route).
- **Explicitly not doing**: dropping SD to 1-bit. Ruled out — SD asset-load speed matters more than the pins it would free.

**Why this matters now**: a second display doesn't need a second full 8-bit bus. The ESP32-S3's LCD_CAM peripheral supports multiple panels sharing one i80 bus via separate CS lines — real incremental cost is **+1 GPIO for a second CS**, +1 more only if independent RST per screen is wanted. The reclaimed pins above (+3 firm, +2 more in reserve) comfortably cover that.

**Camera — a much bigger ask, and a real bandwidth limit on the "just use a coprocessor" idea.** A typical camera module (OV2640/OV7670-class) needs ~12 GPIOs (8-bit parallel data + PCLK + VSYNC + HREF + XCLK) — comparable to the display's own cost, not a small add-on. The "push it onto a coprocessor MCU" pattern already used for buttons/ADC in [[#Analog inputs, and the "I need more pins" pattern]] **does not translate directly to a camera**: that pattern works because a button/ADC read is a few hundred bits at ~100Hz — trivial for a slow I2C/UART link. A camera's native parallel interface exists specifically because real-time video needs far more bandwidth than I2C or UART can carry; relaying raw pixel data over a slow link would bottleneck frame rate to a crawl regardless of how fast the camera's own interface is.
- **What does work**: a coprocessor (e.g. an ESP32-CAM-class board) captures and *compresses* the image on its own end (e.g. to JPEG) and sends only the compressed result over a faster-but-still-thin link like SPI (~4 pins: MOSI/MISO/SCLK/CS) — well-trodden territory, plenty of existing example firmware for exactly this. This is realistic for **still photos or low-frame-rate video**, not a smooth real-time viewfinder — the bottleneck is the inter-chip link, not the camera itself, and compression is what makes that link's low bandwidth tolerable.
- Not yet confirmed: whether the LCD_CAM peripheral's camera-input and display-output paths can run **simultaneously** on this chip, or are mutually exclusive — worth checking before assuming both a display and a direct (non-coprocessor) camera can coexist.

### Battery connector — forward-looking note, not yet built

Not part of this rev (battery is still deferred, see [[#2026-08-10 — Battery deferred to a later revision]]), but decided for whenever it happens: use a **JST-PH 2-pin** connector rather than soldering a specific cell directly — it's the de facto standard on LiPo pouch cells sold with a connector, so swapping to a different capacity/thickness pouch later is just unplugging one and plugging in another. Caveat: the connector is chemistry-agnostic, the charging circuit behind it isn't — swapping to a genuinely different chemistry (e.g. NiMH) later would still need the charger circuitry redone, not just the connector.

### Buttons: expander, not a matrix

Target is **9 buttons** — 8 game buttons plus settings. A 3×3 matrix does give 9 buttons on 6 pins (the arithmetic is right), but it's the wrong choice here:

| Approach | Pins | Left over | Catch |
|---|---|---|---|
| **I2C expander** (MCP23017, decided) | **3** (SDA/SCL + INT) | 3 | One ~$1 chip. No ghosting, no scan loop |
| Shift register (74HC165) | 3 | 3 | ~$0.30, two chained for >8 inputs |
| 3×3 matrix | 6 | 0 | **Needs 9 diodes** — consumes the entire remaining budget |
| Direct GPIO | 9 | −3 | Doesn't fit |

**The diodes are the real argument.** A game controller sees 3–4 simultaneous presses constantly (a D-pad diagonal is already two, plus A and B). An undiodedd matrix generates *phantom* keypresses whenever three keys form a rectangle. Fixing that needs a 1N4148W (SOD-323) per key — 9 extra parts and 9 extra nets, to save nothing. The expander reads all inputs in parallel so simultaneous presses are free, costs 4 fewer pins, and leaves SDA/SCL available for a future sensor or touch controller.

Two free wins:
- **IO0 already has a button** (SW2, the BOOT button). IO0 is a strapping pin only *at boot* — at runtime it's an ordinary input, so the "settings" button can just be that existing switch. No new part, no new pin.
- **"Power button" is soft-only.** The board is USB-powered with no battery, so unplugging *is* the power switch. A power GPIO would only toggle a firmware sleep/wake state — don't design a latching power circuit for it.

### 2026-08-10 — Expander decision: MCP23017, interrupt-driven

**MCP23017 over PCF8575**, for two reasons beyond raw I/O count (both have 16):

- **True GPIO vs. quasi-bidirectional.** The PCF8575's pins are quasi-bidirectional — no direction register at all; every pin has a permanent weak (~100µA) pull-up, "output low" means actively sinking, and "output high" only ever means releasing the pin to that weak pull-up, never real push-pull sourcing. Fine for reading a button (which just wants a pull-up and a switch to ground) but the wrong fit for **driving LCD RST or the status LED** — both already planned to live on this expander's spare I/O in [[#Where to find more pins]] — since those need an actively-driven edge, not a weak-pullup release. The MCP23017 has real per-pin direction control and push-pull outputs, so it covers both jobs cleanly.
- **Interrupt granularity.** Confirmed: **interrupt-driven, not polled.** MCP23017 gives two interrupt pins (INTA/INTB) with per-pin masking, but they can be configured in **MIRROR mode** (`IOCON.MIRROR`) so both ports' interrupts appear on one physical pin — that's the plan, so this costs **1 GPIO**, not 2. Firmware reads `INTF`/`INTCAP` to see which pin fired. PCF8575's single INT line fires on *any* pin change with no masking, which would've meant every future spare-I/O use (status LED, LCD RST) also triggering interrupts whether wanted or not.

**Budget impact**: the interrupt line moves [[#GPIO budget (current)]] from 25/27 (2 spare) to **26/27 (1 spare)**.

**Package: MCP23017-E/ML (QFN, 6×6mm, 28-pin).** Checked LCSC/JLCPCB directly: all four package options (ML/QFN, SO/SOIC, SS/SSOP, SP/SPDIP) are stocked, and **all four are "Extended" parts** for JLCPCB assembly — no cost-tier difference between them, so PCBA handles any of them the same way. ML is the smallest by a wide margin (~36mm² body vs ~53mm² SSOP, ~134mm² SOIC, ~270mm² SPDIP) but has no leads — if a hand-touch-up is ever needed it takes solder paste + hot air/reflow, not an iron. Accepted tradeoff: PCBA assembles it either way, and the fallback plan is learning reflow rather than picking a hand-solderable package.

**Layout note for later**: the QFN has an **exposed thermal pad** (center pad, ties to VSS per the datasheet's Table 2-1) — the KiCad footprint needs that pad connected to the ground plane, not left floating.

## Analog inputs, and the "I need more pins" pattern

Prompted by wanting 2 joysticks + a volume slider on a board with 2 spare GPIOs. The general lesson matters more than the specific parts.

### The pattern: budget a bus, not GPIOs

Past a certain peripheral count, real products stop wiring peripherals to the MCU. **I2C supports 127 addresses and every device on it costs zero pins.** What gets budgeted instead is **bus bandwidth and latency**. This board already has the bus — the expander, the fuel gauge and the ADC below all ride it for free.

### Applying it

Original requirement: 2 joysticks (4 analog axes + 2 click switches) + 1 volume slider (1 analog) = **5 analog, 2 digital**.

**2026-08-10 — SUPERSEDED, analog dropped entirely for this rev.** Joysticks were deferred first (below), then volume followed: volume will be a **button-based setting in the firmware menu** instead of a physical pot, so the ADS7828 has no remaining job on this board and comes out too. Everything below (ADS7828 part pick + wiring, the joystick part, the ratiometric/RC-filter design notes) is kept as a real, verified path back in if either comes back in a later rev — none of it needs re-deriving, just re-adding to the schematic.

- **Joysticks**: confirmed retro-go doesn't need them — its reference hardware (ODROID-GO and most supported boards) is D-pad + buttons only, and the 9-button MCP23017 setup is already a complete, fully-functional controller with zero dependency on analog input.
- **Volume**: checked retro-go's own repo — it has a real settings menu with audio-related options (`Audio Out: Ext DAC` is a real menu item), though the exact phrasing around volume ("MENU, the volume knob") suggests its reference hardware may use a physical knob rather than pure button-adjustment, so this isn't a fully-confirmed out-of-the-box feature. Not a real blocker either way: adding a button-adjustable volume level to a menu-driven firmware is a small, ordinary piece of code (increment/decrement a variable from a menu option) — nowhere near the complexity of anything else considered tonight (see [[#2026-08-10 — P4 / second-screen / camera / foldable — considered, not pursued]]).

| Need | Where it goes | GPIO cost |
|---|---|---|
| 5 analog channels (volume + 2 joysticks), **none wired this rev** | **I2C ADC** — deferred entirely, no ADS7828 on the board this rev | **0** |
| 2 joystick clicks, **not wired this rev** | MCP23017 spare I/O | **0** |

**Zero GPIOs spent either way.** With analog deferred entirely, the expander keeps all 7 free I/O beyond the status LED and LCD RST (5 spare on the expander itself) — none committed to joystick clicks since neither the ADC nor the joystick part are on the board this rev.

| ADC part | Res | Ch | Note |
|---|---|---|---|
| **[ADS7828](https://www.ti.com/lit/ds/symlink/ads7828.pdf)** | 12-bit | 8 | **Decided 2026-08-10**, over the 8-bit ADS7830, for smoother joystick centring/deadzone feel. Verified against the real datasheet: TSSOP-16 (hand-solderable, unlike the MCP23017's QFN), A0/A1 address pins are a simple GND/VDD tie (4 possible addresses), I2C up to 3.4MHz (High-Speed) though Fast mode (400kHz) already gives 8kHz throughput, miles beyond the ~100Hz polling this needs. **Only 1 of 8 channels used this rev** (volume) with joysticks deferred — kept the 12-bit part anyway since it costs nothing extra and leaves the other 7 channels ready for joysticks later |
| [ADS7830](https://www.ti.com/product/ADS7830) | 8-bit | 8 | Cheaper. 256 steps is coarse for a thumbstick, fine for a volume slider — passed over for the 12-bit's smoother feel |
| ADS1115 | 16-bit | 4 | Highest resolution, but 4 channels isn't enough alone |

### ADS7828 wiring (2026-08-10)

TSSOP-16, pinout confirmed against the real datasheet:

| Pin | Signal | Wire to |
|---|---|---|
| 16 | +VDD | 3.3V + decoupling cap |
| 9 | GND | GND |
| 14 | SCL | Existing I2C bus (shared with MCP23017 — don't double the pull-ups) |
| 15 | SDA | Same bus |
| 12, 13 | A0, A1 | Both → GND. Only ADC on the bus, so the exact combo doesn't matter — pull the real resulting 7-bit address from the datasheet's address table for firmware, don't assume one |
| 11 | COM | GND — puts all 8 channels in single-ended mode, each read relative to ground |
| 1–8 | CH0–CH7 | **1 used this rev** (volume wiper) — joysticks deferred, see [[#Joystick part: Adafruit #5628 (Joy-Con style) — chosen, deferred to a later revision]]. 7 spare, 4 reserved for the 2 joysticks if they come back. 100nF filter cap to GND per [[#Analog design notes]] |
| 10 | REF_IN/REF_OUT | **Enable the internal 2.5V reference in firmware (an I2C config bit, not a hardware strap) and use this pin's 2.5V *output* to power the joystick/volume pot tops — not 3.3V.** This is what makes [[#Analog design notes|the ratiometric principle]] concrete: pot supply = ADC reference supply, so the pot's full mechanical travel maps to the full 0–4095 digital range (powering from 3.3V instead would saturate the top ~0.8V of travel and waste range), and rail noise cancels out of the reading. Bypass cap here too — check the datasheet's applications section for the exact value before finalising. |

### Joystick part: Adafruit #5628 (Joy-Con style) — chosen, deferred to a later revision

**Part choice made, but not built this rev** — see the deferral note above [[#Applying it|in "Applying it"]]. Kept here so the decision doesn't need re-deriving if joysticks come back.

2× 10kΩ pots (matches the pot-value guidance above exactly) + a built-in center click switch. Chosen over the classic self-contained PSP-style module (Adafruit #3102, ~$20) because #5628 is sold as **bare potentiometers meant to be soldered directly onto a custom PCB**, not a separate module with its own board — meaning it can be **board-mounted like the buttons**, fixing its position now rather than waiting on an enclosure, per [[#2026-08-10 — Expander decision: MCP23017, interrupt-driven|the board-mount-vs-connector split from earlier]]. Also cheaper (~$4.50) and saves sourcing a separate click switch. **Watch for**: 2mm pin pitch, not the standard 2.54mm — needs a custom KiCad footprint, won't drop into a standard header/breadboard spacing. No official Adafruit KiCad library exists (unlike SparkFun) — symbol is trivial (stock pot + switch symbols), footprint needs real pad coordinates once the part is in hand or from Adafruit's own dimensions if published.

**retro-go compatibility checked directly against source, not assumed**: `components/retro-go/rg_input.h` has a first-class `rg_keymap_adc_t` type — a built-in ADC-channel-to-button mapping with min/max thresholds — alongside `rg_keymap_i2c_t` for expander-based buttons (what the MCP23017 buttons will use). Analog joysticks don't block retro-go and don't need custom glue code; thresholding an axis into a virtual D-pad press is an existing, supported keymap type in the framework itself.

**Cheap alternative**: a **74HC4051** 8:1 analog mux — 3 select lines (put them on expander spares) + 1 analog output = **1 GPIO**, ~$0.30 vs **$7.11** (ADS7828, DigiKey single-unit price — corrected 2026-08-10, the earlier "~$4" guess was low). Feeds the internal ADC, so it inherits the problems below.

**Scale-up answer** (not needed here, but it's where the road leads): a coprocessor MCU — RP2040, STM32C0 — doing all input scanning and ADC, reporting over I2C/UART. Standard practice in custom keyboards and handhelds.

### Why not the ESP32-S3's own ADC

- **Hard blocker: ADC2 (IO11–IO20) is unusable while Wi-Fi is on**, so only ADC1 (IO1–IO10) counts — and the LCD data bus sits on it. Even after the reshuffle in [[#Where to find more pins]] that frees two channels, not five.
- **Linearity**: Espressif's own measurements show the S3 well-behaved below ~2750 mV but nonlinear above it. ESP-IDF's [curve-fitting calibration](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/adc_calibration.html) compresses full-scale error to roughly −30…0 mV using factory-tuned per-chip coefficients — usable, but noise-sensitive, and no external reference can be supplied.

Irrelevant for a volume slider; mildly annoying for joystick centring and deadzones. The external ADC removes the question.

### Bus bandwidth check

Bandwidth is the resource now, so it's worth the arithmetic: ADS7828 across 5 channels ≈ 150 bits including addressing and ACKs, MCP23017 read ≈ 40 bits → **≈ 190 bits per input poll ≈ 0.5 ms at 400 kHz**. At 100 Hz polling that's **~5% bus utilisation**, with Fast-mode Plus (1 MHz) in reserve. Latency well inside one frame at 60 fps.

### Analog design notes

- **Ratiometric**: power the pots from the same rail feeding the ADC's reference, so supply variation cancels out of the reading — concretely, the ADS7828's own REF_OUT pin, see [[#ADS7828 wiring (2026-08-10)]].
- **RC filter at each wiper**: 100 nF to GND. A 10 kΩ pot's ≤2.5 kΩ source impedance puts the corner near 640 Hz — far above thumbstick bandwidth, well below switching noise.
- **Use 10 kΩ pots, not 100 kΩ** — SAR ADCs need low source impedance to charge the sampling cap.
- Route analog away from the LCD data bus and especially the Class-D speaker output (see [[#Layout notes]]).

## Battery: 18650 with power-path charging — DECIDED 2026-08-07

**Decided**: battery-powered, single **18650** cell (chosen for cost and replaceability), Wi-Fi in use, and the I2C bus accepted as the way to add peripherals. This section was written while the decision was open; the alternatives are retained because the reasoning still applies.

**Impact on the existing schematic**: only the power section, and only downstream of the connector. **U2 (TPS62A02) and its L1 / C1 / C2 / R5 / R6 / C3 network come out**, replaced by a BQ24074 charger plus a buck-boost. J1, the CC resistors, F1, U3 and the whole USB front end are untouched — and the CC resistors gain a second job (see [[#USB-C current detection — now free]]).

### Power tree

| Stage | Part | Notes |
|---|---|---|
| USB in → charger | **[BQ24074](https://www.ti.com/lit/ds/symlink/bq24074.pdf)** | 1.5 A linear charger with power path; tolerates input to 28 V. `ISET` resistor sets charge current; **`ILIM` (1.1 k–8 kΩ) sets the input current limit** — this is how the USB budget gets respected |
| Charger → system | SYS node, ~3.0–4.5 V | TI's DPPM *"simultaneously and independently powers the system and charges the battery"* — the behaviour that makes the device usable while plugged in |
| SYS → 3.3 V logic | **Buck-boost** (TPS63020 / TPS63802 class) | Replaces the TPS62A02 |
| SYS → audio | MAX98357A direct | Its 2.7–5.5 V range spans the whole cell range |
| SYS → backlight | Boost LED driver | Accepts 3.0–4.2 V in |

BQ24074 is a 3×3 mm QFN — reflow-only, but the WROOM-2 module and SOT-563 buck already committed this board to reflow.

### 18650 practicalities

- **Protected cells don't fit holders sized for unprotected ones.** A flat-top 18650 is 65 mm; a protected cell adds a BMS at the negative end and runs 2–5 mm longer, frequently past 69–70 mm. Documented case: a Panasonic NCR18650B protected cell at 69.4 mm won't fit a Keystone 1049 rated to 68.9 mm. **Decide protected-vs-unprotected first, then pick the holder to match, and check the holder's max-length spec against the specific cell.**
- **Chosen approach: unprotected flat-top cell + protection IC on the board** (DW01A + dual N-FET, or a small BMS). Guaranteed 65 mm fit, thresholds under your control, cheaper cells, and — the deciding reason — protection still works when a user swaps in *any* cell, which a protected-cell-only design can't guarantee.
- **Protection is not optional on Li-ion.** Either route is acceptable; neither is not.
- **Counterfeits are an engineering problem here, not just a value one** — see the brownout interaction below. 18650 is the most faked cell format in circulation; anything advertising over ~3600 mAh is fabricated. Buy Samsung / LG / Sony / Molicel from a real distributor.

### Wi-Fi + battery: sag and brownout

Wi-Fi TX draws ~350–500 mA in ~1–2 ms bursts, and that meets the cell's internal resistance:

- **Good 18650** (~50–100 mΩ) → 25–50 mV sag. Invisible.
- **Tired or counterfeit cell** (300–500 mΩ) → 150–250 mV sag. From a 3.2 V resting cell that's ~2.95 V — below the buck-boost's floor, i.e. a reset mid-game.

Mitigation: **≥100 µF bulk at the buck-boost input**, plus a genuine cell.

### Form factor and runtime

18650 is 18 mm diameter, ~20–21 mm with a holder; plus PCB, panel and enclosure walls the device lands around **28–32 mm thick** — essentially a Game Boy DMG (~32 mm), so on-target for the concept rather than a compromise. (A LiPo pouch such as a 103450 would be 10 mm thick, but loses on cost and replaceability.)

On a good 3000 mAh cell (~11 Wh) against ~2.0 W of load plus converter losses: **~4.5–5 h** with Wi-Fi active and audio playing, **~8 h** with Wi-Fi off and quiet. Worth dropping Wi-Fi to modem-sleep when idle — it's a meaningful fraction of the draw.

### USB-C current detection — now free

There are now **two** loads on USB: running the device and charging the cell. At the default 500 mA the system takes most of it and charging crawls.

Drawing 1.5 A or 3.0 A legally requires detecting what the source offers by **measuring the CC line voltage** — the source's Rp forms a divider against the existing 5.1 kΩ Rd from [[#Powering from USB-C]]. That needs an ADC channel, and the **ADS7828 has three spare** ([[#Analog inputs, and the "I need more pins" pattern]]). Switch a second `ILIM` resistor in with a small MOSFET driven from an expander pin.

**Both ends cost zero GPIOs**, which resolves the long-standing "research USB-C current advertisement" open item. Check the exact vRd thresholds against the USB-C spec's Rp table before implementing — the concept is settled, the numbers aren't yet verified here.

### Pin cost: near zero

| Function | GPIO cost |
|---|---|
| Charger IC | 0 — status outputs go on the I2C expander |
| **Fuel gauge on I2C** (MAX17048 class) | **0** — shares the existing SDA/SCL |
| Buck-boost | 0, or 1 for an enable line |
| Power switch | 0 with a slide switch |

The I2C fuel gauge is the elegant answer to battery sensing: real state-of-charge percentage rather than a raw divider reading, and **no ADC pin at all** — which sidesteps the ADC1 problem in [[#Where to find more pins]] entirely.

### The real cost: the power tree gets reworked

A Li-ion cell swings **3.0–4.2 V**, straddling 3.3 V — **a buck physically cannot regulate 3.3 V from a 3.4 V input.** And on battery there is no 5 V rail at all, which is where audio and backlight currently sit.

| Block | USB-only (current) | With battery |
|---|---|---|
| Input | VBUS direct | USB → **charger with power path** → SYS node |
| 3.3 V logic | TPS62A02 buck | **Buck-boost** (TPS63020 / TPS63802 class) |
| Audio amp | off `VCC_5V` | Straight off the cell — MAX98357A takes 2.7–5.5 V, just less output power |
| Backlight | off `VCC_5V` | Boost LED driver accepts 3.0–4.2 V input fine |

- **Insist on power-path charging** (BQ24074 class, **not** MCP73831/TP4056). Power path lets the device run from USB while the battery charges on a separate path. Without it, load current during charging corrupts charge-termination detection — you get a handheld that can't be used while plugged in and a charger that never reliably finishes.
- **Safety**: use a cell with integrated protection, or add a protection IC. Non-negotiable on Li-ion.
- **A hard power switch now makes sense** — unlike the USB-only case in [[#GPIO budget (current)]] where unplugging *is* the power switch. A slide switch is simplest; a soft latch (P-FET + hold GPIO) is nicer but costs a pin and a circuit.
- **Runtime**: at ~400–500 mA average (backlight dominates), a 2000 mAh pouch cell gives roughly 4 hours.

**Decision rule** (applied — battery chosen): if it's even 50/50, design for the battery now. Retrofitting a power path and buck-boost later means building the power block twice.

## 2026-08-10 — Battery deferred to a later revision

**Decision: v1 is USB-only.** No battery this rev, for build simplicity — the [[#Chosen buck: TPS62A02NDRL (2A, SOT-563-6) — verified wiring|TPS62A02 buck]] above is back in as the live 3.3V rail. The 18650/BQ24074 work above and the NiMH exploration below are kept for whenever battery support gets revisited; both reached a real conclusion worth not re-deriving next time:

- **18650 + BQ24074 path** (above): fully specced, blocked on nothing electrical — just deferred.
- **NiMH alternative was explored as a simpler-sounding option, then given up on.** Worth knowing why before reopening it:
  - Runtime at ~2W active load: 2×AA ≈ 1.9–2.5h, 4×AA ≈ 3.8–5h (matches the 18650 target), 2×AAA ≈ 0.8–1h. For comparison, the original Game Boy's 15–30h on 4×AA came from its ~0.3–0.5W load (no backlight, no Wi-Fi, no amp), not from anything about AA cells — cell count/chemistry was never going to close that gap at this board's power budget.
  - **4×AA in series can't be onboard-charged from bare 5V USB.** NiMH fast-charge voltage peaks ~1.5–1.6V/cell (that peak-then-dip is what −dV/dt termination detects); 4 in series peaks ~6.2–6.4V, above the 5V supply — no charger topology can charge above its own input. Fix without giving up capacity: wire as **2S2P** (two 2-cell series strings in parallel) — same 4 cells, same capacity, pack voltage stays ~2.4–3.2V.
  - Candidate IC: **LTC4060** (standalone linear NiMH/NiCd charger, external PNP pass transistor, `I_CHARGE = 930 × 1.5V / R_PROG`, real datasheet confirmed via a yumpu mirror after analog.com's own PDF host repeatedly timed out this session — worth a direct-source re-check if this gets revisited).
  - **Why it stalled**: fast charging (~1h, matching a household AA charger) needs multi-amp current, which a linear charger off 5V dissipates as `(V_IN − V_BAT) × I_CHARGE` in the external pass transistor — several watts, impractical to cool in a handheld shell. 1A gives ~4–5h at a comfortable ~2W dissipation; pushing toward "fast" charging on this architecture just isn't realistic without a switching charger and a boosted input voltage (USB-PD/QC), which is real scope beyond a resistor change.

## Audio (I2S → Class-D, MAX98357A)

Already costed in [[#GPIO budget (current)]] at 3 pins; this section is the circuit.

### The ESP32-S3 has no DAC

Classic ESP32 has two 8-bit DAC channels (GPIO25/26) and the ESP32-S2 has two (GPIO17/18) — **the S3 removed the peripheral entirely.** ESP-IDF's DAC driver isn't built for the S3 target and Arduino's `dacWrite()` doesn't work on it. Analog audio out is therefore off the table; it has to be **I2S into an external chip**. Another instance of the family-variant trap in [[#2026-08-07 — Chip variant mismatch gotcha (classic ESP32 vs S3)]] — don't port an audio design across ESP32 variants unexamined.

### Chosen part: MAX98357A

I2S digital in, Class-D speaker out, mono, 2.7–5.5 V, up to 3.2 W into 4 Ω — DAC and amplifier in one part. **No MCLK required**: it clocks entirely off BCLK/LRCLK, so it really is 3 GPIOs.

**2026-08-10 — wiring verified against the real datasheet, two corrections to the numbers below.** Package: **TQFN-16** (3×3mm, real numbered pins) chosen over the WLP-9 ball package — far more standard, much easier to get a reliable KiCad footprint for, consistent with how packages have been picked elsewhere on this board.

| Pin (TQFN-16) | Connect to |
|---|---|
| 7, 8 (VDD) | **`5V+`**, not 3V3 — more output swing, keeps audio transients off the logic rail. + 10µF + 0.1µF bypass caps, straight from the datasheet's own functional diagram |
| 3, 11, 15 (GND) + exposed pad | Ground plane — EP isn't internally connected but the datasheet explicitly wants it tied for thermal dissipation |
| 1, 16, 14 (DIN, BCLK, LRCLK) | `I2S_DIN`/`I2S_BCLK`/`I2S_LRCLK` hierarchical labels — 3 spare pins on the MCU side |
| 9, 10 (OUTP/OUTN) | Speaker directly (bridge-tied, no coupling caps) |
| 2 (GAIN_SLOT) | **Decided: 6dB — direct tie to VDD, zero extra parts.** Real table from the datasheet: floating=9dB (default), tie to VDD=6dB, 100kΩ to VDD=3dB, tie to GND=12dB, 100kΩ to GND=15dB. 6dB picked over 3dB for the simpler wiring, still well below the 9dB default for the power reasons below |
| 4 (SD_MODE) | **Corrected: 634kΩ to 3.3V**, not the earlier "1MΩ to VDD" approximation — see below |

**SD_MODE correction.** The datasheet gives an exact formula, not just an approximate voltage divider: `R_LARGE(kΩ) = 222.2 × V_DDIO − 100`, where V_DDIO is whichever rail you pull up to. Pulling to 3.3V (consistent with every other pull-up on this board being on the 3.3V rail, not 5V) gives **R_LARGE = 634kΩ** (a real E96 value) for mono (Left/2 + Right/2) mode — the earlier "1MΩ to VDD ≈ 0.45V" note was an approximation assuming a pull-up to 5V instead (which would actually want ≈1.01MΩ, not exactly 1MΩ). SD_MODE is an analog mode pin, not a simple enable — internal 100kΩ pulldown, so **left alone the amp sits in shutdown** (the classic "my MAX98357A is silent" bug). Thresholds: `<0.16V` shutdown · `0.16–0.77V` mono average · `0.77–1.4V` right only · `>1.4V` left only.

**Never ground either output.** The outputs are **bridge-tied differential** — the speaker floats across OUT+/OUT−. This also rules out hanging a headphone jack on them, since a jack shares a ground and would short half the bridge. Headphones later would need a separate single-ended path (e.g. PCM5102A line-out plus its own amp), not a tap off this one.

**Parts**: amp (ref number TBD at placement — R18/R19 are already taken by the I2C pull-ups, so don't reuse those numbers from the old placeholder BOM), 10µF + 0.1µF bypass at VDD, 634kΩ (SD_MODE), no resistor needed for GAIN (direct VDD tie), J4 speaker connector.

### Speaker: Adafruit #3923 — DECIDED 2026-08-10

Mini Oval Speaker, 8Ω, 1W, 30×20×5mm, 3g. **Explicitly validated by Adafruit's own product description as compatible with the MAX98357A** — not a generic guess. Comes with a **Molex PicoBlade 1.25mm 2-pin connector** on 30mm wire leads already attached — no bare-wire soldering, matches the J4 connector already planned. No polarity concern on the 2 leads (bridge-tied differential output, swapping them just reverses phase — inaudible with one mono speaker). $1.95.

**Mounting — two things to get right, not yet confirmed on the physical part:**
- **Fixing hardware unknown.** Adafruit's listing covers the electrical connector but not screw holes or a frame. Small frameless speakers like this are commonly stuck down with adhesive foam tape/gasket — which conveniently also seals the back volume (next point) — but this hasn't been verified against the actual part yet.
- **Sealed back volume still required** (per [[#Layout notes]] below) — a mechanical/enclosure task, not resolved by picking the speaker.

**3D-printed enclosure compatibility — works, but not automatically.** FDM printing has real porosity risk: layer-line gaps can leak air (and sound) through the cavity walls, undermining the seal. Mitigations: enough perimeters/wall thickness in the cavity area (2-3+ perimeters, higher infill locally), and — likely the bigger leak risk than the bulk material — a proper gasket/seal at the speaker-to-cavity mounting interface, not just the printed walls themselves. **Resin (SLA/MSLA) printing sidesteps the porosity question entirely** — it cures as a continuous solid, no layer-adhesion gaps. Many DIY handheld builds do use FDM speaker cavities successfully, so this isn't a blocker — just needs the wall thickness and mounting-interface seal handled deliberately, not assumed.

**Schematic annotation, not just wiring**: worth adding a plain text note on the page near J4 (not an electrical symbol — KiCad's Place → Text tool) flagging the two things that don't show up in the wiring itself: needs a sealed back-volume cavity in the enclosure, and route OUT+/OUT− away from the antenna keepout and analog signal lines during layout. The schematic is what gets looked at during layout — better to have the reminder there than only in this note.

### Power — the real constraint

The largest new load on the board. **3.2 W into 4 Ω at full tilt is ~700 mA from 5 V**, which would exceed the ~900 mA USB ceiling on its own, before backlight and Wi-Fi. Mitigations, most effective first:

- **Use an 8 Ω speaker, not 4 Ω** — roughly halves current for the same voltage swing. A handheld isn't chasing 3 W.
- **Gain set low: 6dB**, decided — see [[#Speaker: Adafruit #3923 — DECIDED 2026-08-10]] — rather than the 9 dB default.
- Budget **~150–250 mA** for typical game audio and treat the peak as higher.

It sits on `5V+` directly, so it does **not** load the TPS62A02 — but it fully loads the USB port. Folded into the power-budget item in [[#Next steps]].

### Layout notes

- The Class-D output is a **~300 kHz PWM square wave** into an inductive load. Route OUT+/OUT− as a short tight pair, away from the display data bus, the SD clock, and especially the **module's antenna keepout** — this is a Wi-Fi board. The part is "filterless," but that assumes short speaker leads; on flying wires, add ferrite beads or a small LC filter.
- A bare speaker on a PCB sounds terrible — it needs a small **sealed back volume** in the enclosure, a mechanical problem to solve alongside the panel mounting.

## Firmware stack (no custom OS needed)

Recorded because it affects hardware choices, not as a software plan.

**No OS to write.** The ESP32-S3 is a microcontroller, not an applications processor — **ESP-IDF is built on FreeRTOS**, so tasks, queues, timers and semaphores exist from the first line. The "custom OS" of the retro-handheld world belongs to the *other* device class: Anbernic-style units on Linux-capable Rockchip/Allwinner SoCs. Different silicon, not applicable here.

| Layer | Use | Note |
|---|---|---|
| Base | **ESP-IDF** | FreeRTOS + the drivers already being designed against: `esp_lcd`, `sdmmc`, `i2s`, `i2c` |
| UI / menu | **[LVGL](https://lvgl.io/)** | Official `esp_lcd` port. Renders **dirty rectangles** — the refresh-rate strategy from [[#Why not 16-bit i80 — it buys literally nothing]], already implemented |
| Storage | **FATFS** | Built in: `esp_vfs_fat_sdmmc_mount()` mounts the SD card in a few lines |
| Audio | **I2S driver**, or **ESP-ADF** | ADF only if MP3/WAV decoding is wanted over raw samples |

A launcher with a menu is an *application feature*, not an OS — LVGL provides it.

**Prior art worth designing toward**: [retro-go](https://github.com/ducalex/retro-go) is an existing launcher-plus-emulator firmware for ESP32 handhelds, descended from the ODROID-GO, covering NES, Game Boy/Color, SNES, Master System, Game Gear, PC Engine, Lynx and DOOM, with ESP32-S3 builds in the wild. **Hardware implication: keep the design conventional** — i80 display, SDMMC card, I2S audio, GPIO/expander buttons is exactly the shape those devices take, and this design already matches.

**Honest scoping note**: the firmware is the larger half of this project. The board is a few weekends; a polished launcher and an enjoyable game is much longer. Doesn't change any hardware decision — just says where the effort sits.

## Indicator LEDs

Worth the board space specifically **because there's no USB-UART bridge chip to fall back on** — with native USB, a failed enumeration gives no diagnostic at all, so the LEDs are the only pre-firmware signal.

| Ref | Net | Resistor | Purpose |
|---|---|---|---|
| D1 | 3V3 → GND | R15, 1kΩ (~1.3 mA) | **Buck is regulating.** Dark on bring-up ⇒ power fault, not firmware — that split is the whole value |
| D2 | VCC_5V → GND | R16, 2.2kΩ | **USB is supplying VBUS.** Without it, a dead buck and a dead cable look identical |
| D3 | spare GPIO (IO48) → GND | R17, 1kΩ | **Chip is running and toggling pins.** Proves this *before* the WS2812 chain works, since addressable RGB needs correct bit-bang timing to show anything |

- **Use red / green / yellow, not blue or white** — a blue LED's ~3.0 V forward drop leaves almost no headroom on a 3.3 V rail.
- **Skip TX/RX activity LEDs** — those exist to debug USB-UART bridge chips, and this board has none (see [[#Minimum circuit (WROOM module)]], item 4).

## Reference sources

- **Espressif Hardware Design Guidelines** (canonical schematic checklist, per chip variant): [ESP32 guidelines](https://docs.espressif.com/projects/esp-hardware-design-guidelines/en/latest/esp32/) — has a literal "Schematic Checklist" page plus full reference schematic/power diagrams. **ESP32-S3 variant (the one this project uses)**: [S3 guidelines](https://docs.espressif.com/projects/esp-hardware-design-guidelines/en/latest/esp32s3/) ([PDF](https://docs.espressif.com/projects/esp-hardware-design-guidelines/en/latest/esp32s3/esp-hardware-design-guidelines-en-master-esp32s3.pdf)).
- **ESP32-S3 Series Datasheet v2.2** (chip variants/SKU table, pinout, electrical specs): [datasheet PDF](https://www.espressif.com/documentation/esp32-s3_datasheet_en.pdf).
- **Official dev-kit schematics** (copy these directly as a starting point): [espressif/esp-dev-kits on GitHub](https://github.com/espressif/esp-dev-kits) — the actual schematics/PCB/Gerbers Espressif ships for its own dev boards, e.g. the [ESP32-S3-DevKitC-1 schematic PDF](https://dl.espressif.com/dl/schematics/SCH_ESP32-S3-DevKitC-1_V1.1_20220413.pdf).

- **ESP-IDF peripheral docs** (firmware-side, but they carry the hardware rules): [SDMMC Host Driver — ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-reference/peripherals/sdmmc_host.html), [SD Pull-up Requirements](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-reference/peripherals/sd_pullup_requirements.html), and [SPI Master](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-reference/peripherals/spi_master.html) for the IO_MUX pin table.
- **MAX98357A** application detail (SD_MODE thresholds, gain resistors, BTL output): [Adafruit's MAX98357 breakout guide](https://learn.adafruit.com/adafruit-max98357-i2s-class-d-mono-amp/pinouts) is clearer than the datasheet on the mode-pin behaviour.
- **Espressif BSP pin maps** (what Espressif actually wires on its own boards — useful precedent for pin choices): [espressif/esp-bsp](https://github.com/espressif/esp-bsp), e.g. `bsp/esp32_s3_eye/include/bsp/esp32_s3_eye.h`.

Fastest path: pull up one DevKitC schematic from the GitHub repo side-by-side while drawing a first board — copying a proven layout, then trimming what isn't needed, beats designing from the datasheet alone.

## 2026-08-07 — Chip variant mismatch gotcha (classic ESP32 vs S3)

Picked ESP32-S3 (resolves the earlier "classic vs S3" open question). Hit a specific trap worth flagging: the classic-ESP32 and ESP32-S3 hardware design guideline PDFs/schematic-checklist pages are **not interchangeable** — different silicon, different pinout, different pin count, even though both are "ESP32."

- Concretely: classic ESP32 has **CAP1/CAP2** pins (47/48) — an RC network (10nF cap) that drops the internal core voltage 1.1V→~0.7V during deep-sleep for power savings. The **ESP32-S3 has no such pins** — different internal power-management scheme entirely. Their absence on an S3 symbol is correct, not a missing-pin bug.
- The KiCad symbol pulled in (labelled "ESP32-S3", pins like VDDA/LNA_IN/CHIP_PU/XTAL_P/N/VDD3P3_RTC) is the **bare chip die**, not the WROOM module — confirmed pin-for-pin against Espressif's own [ESP32-S3 schematic checklist](https://docs.espressif.com/projects/esp-hardware-design-guidelines/en/latest/esp32s3/schematic-checklist.html). This contradicted the module-first advice above. **Resolved** — the schematic now uses `RF_Module:ESP32-S3-WROOM-2` (symbol *and* footprint), the 41-pin module part, so the bare-die symbol is out of the design.
- Lesson: always fetch the guideline doc/checklist for the **exact chip variant** in the schematic (`/esp32s3/` path for S3, not `/esp32/`) — don't assume pin numbers or names carry across variants.

## 2026-08-07 — KiCad USB-C symbol gotcha: Plug vs. Receptacle

Schematic review caught the connector symbol used was `USB_C_Plug_USB2.0`, not a receptacle.

- **Plug vs. Receptacle**: a plug is the male connector molded onto a cable's end; a receptacle is the female socket mounted on a board edge. Using a plug footprint on the board means a standard phone-charger cable (which ends in its own plug) physically cannot mate with it — need `USB_C_Receptacle_USB2.0` instead for a normal charge cable to work.
- **CC1/CC2 must each get their own 5.1kΩ pull-down**: a plug only exposes a single CC pin (fixed orientation), but a receptacle exposes CC1 *and* CC2 separately, because a USB-C cable can be inserted either way up and only one CC line ends up active depending on orientation. Pulling down only one (as the plug symbol implied) means the board only powers up for one of the two possible cable orientations. Confirms the "each CC pin" wording already in the [[#Powering from USB-C]] section above — a common hobbyist mistake is only wiring one.
- **Why CC1/CC2 can't be paralleled like D+/D- can**: a cable's plug internally shorts its own D+ pins (A6+B6) and D- pins (A7+B7) together at each end — only one physical wire pair actually runs the cable's length — so tying those pairs together on the receptacle side is correct and required (what the simplified symbol does automatically). **CC is the opposite**: only *one* of the two CC positions is wired end-to-end through a cable at all (the other becomes VCONN or is unconnected), which is what lets a device detect orientation in the first place. Tying CC1+CC2 together on the board would turn two 5.1kΩ resistors into a 2.55kΩ parallel combination as seen by the host — outside spec tolerance, can break power detection. Keep CC1 and CC2 as two fully separate nets.
- **Which receptacle symbol**: use the *simplified* `USB_C_Receptacle_USB2.0` KiCad symbol, not the full 24-pin one — it pre-merges the duplicated VBUS/GND/D+/D- pads (which are just the same net twice for cable-flip support) into single schematic pins, while still keeping CC1/CC2 separate since those need individual handling. The underlying footprint still has all 24 physical pads; this is purely schematic-clarity, nothing lost. The full 24-pin symbol only matters for USB 3.x SuperSpeed or DisplayPort/Alt-Mode designs — not applicable to the ESP32-S3 (USB 2.0 full-speed only).

## 2026-08-07 — Schematic build state (first pass)

KiCad project lives at `projects/esp32build/` (`esp32build.kicad_sch`). Reconstructed from the last save and the exported `net.net` netlist after VS Code closed unexpectedly — **no work was lost**, the last explicit save is newer than the last autosave. KiCad keeps rolling snapshots in `projects/esp32build/.history/` (a real git repo, browsable via *File → Local History*).

**Complete and wired** — the entire USB-C power/data front end, 21 components:

| Block | Parts |
|---|---|
| USB-C input | J1 `USB_C_Receptacle_USB2.0_14P`; R1/R4 5.1k on CC1 and CC2 as **separate** nets (per [[#2026-08-07 — KiCad USB-C symbol gotcha: Plug vs. Receptacle]]) |
| ESD + data | U3 USBLC6-2SC6 wired as a pass-through on D+/D− (pins 1/6 and 3/4 are internally common); R2/R3 33Ω series → U1 pins 13/14 |
| 5V→3.3V buck | F1 polyfuse → U2 TPS62A02NDRL: C1 4µ7, EN tied to VIN, L1 1µH, C2 22µF, R5 450k / R6 100k divider, C3 120p feedforward, OUT (pin 6) sensing the 3V3 net |
| Module | U1 ESP32-S3-WROOM-2: C5 10µ + C6 0.1µ on 3V3; R7 10k + C4 1µ + SW1 on EN; SW2 on IO0; IO3/IO45/IO46 explicitly no-connect |

**Not started**: every GPIO except IO0 is unconnected (no display, buttons, speaker, or SD yet); `esp32build.kicad_pcb` is an empty stub, so no layout.

**Known defects to fix** (both now in [[#Next steps]]):
- **No power flags.** Only GND symbols exist (15 of them) — VBUS and 3V3+ have no `PWR_FLAG`/power symbol, so ERC will report "power input pin not driven" on U1 pin 2 and U2 pin 3.
- **Footprints assigned on only 3 of 21 parts** (U1, U2, U3).

**Housekeeping**: two stale lock files (`~esp32build.kicad_pro.lck`, `~esp32build.kicad_sch.lck`) survive from the crash — delete them if KiCad claims the project is already open.

## 2026-08-11 — Schematic build state (measured from the files)

Read out of the `.kicad_sch` files directly rather than from memory of the last session, because the 08-07 section above and the pre-sync `## Next steps` list had both drifted. **All five sheets are now started** — the "display sheet is empty" and "only `power` has sheet pins" claims were stale. Counts exclude power/GND symbols.

| Sheet | Symbols | Parts present | State |
|---|---|---|---|
| `power` | 24 | J1, R1/R4 (CC), U3 USBLC6, R2/R3 33Ω, F1, U2 TPS62A02 + C1/C2/C3/L1/R5/R6, D1/D2 + R9/R16 | **Complete** |
| `core` | 15 | U1, C5/C6, R7+C4+SW1 (EN), SW2 (BOOT), R18/R19 (I2C pull-ups) | **Complete** |
| `storage` | 15 | J2 DM3AT, R8/R10–R14 pull-ups, R15, R13 series, C7/C8 | **Complete** |
| `user_in_out` | 20 | U4 MCP23017, SW3–SW12 (all 10 buttons), D3 + R17 status LED, C9 | **Complete** |
| `display_and_audio` | 8 | J3 `Conn_01x20` (display breakout), U6 MAX98357A, J4 `Conn_01x02` (speaker), R20 | ~~Partial~~ — **stale, see [[#2026-08-14 — display_and_audio re-verified: far more complete than this table shows]]** |

**Hierarchy**: sheet pins have been imported on **all five** sheets — `core` 13, `storage` 7, `display_and_audio` 3, `power` 2, `user_input` 1 (26 total, covering `USB_Data±`, `sd_clk`/`sd_cmd`/`sd_dat0-3`/`sd_detb`, `IO_EXP_INT`, `I2S_BCLK`/`I2S_LRCLK`/`I2S_DIN`). **But the root sheet contains zero wire segments**, so none of those pins are actually joined — the pins exist as placed graphics only. Everything inter-sheet is currently carried by global labels, not by the hierarchy.

**Still outstanding, confirmed by measurement:**

- **`PWR_FLAG` count across the whole project: 0.** The 08-07 defect was never fixed — ERC will still fail "power input pin not driven" on U1 pin 2 and U2 pin 3.
- **Footprints assigned on 8 of 55 symbols.** Everything missing is passives, switches, LEDs and J1 — i.e. the ICs/connectors were done and nothing else.
- **`esp32build.kicad_pcb` is still a 79-byte stub** — layout has not begun.
- ~~`display_and_audio` gaps specifically: the MAX98357A's 10µF + 0.1µF VDD decoupling, the 634kΩ SD_MODE resistor to 3.3V, and the GAIN-to-VDD tie are not placed (only R20 exists so far); the 20-pin display table is not wired.~~ **Stale as of 2026-08-14** — this whole bullet undersold it; only the VDD decoupling caps are actually missing. See [[#2026-08-14 — display_and_audio re-verified: far more complete than this table shows]].

**Ref-designator drift confirmed** — the numbering has been reassigned since the 08-07 table, e.g. R9/R15/R16 no longer mean what that section says, and the MAX98357A is U6 while the MCP23017 is U4. Treat every ref number in the older sections of this note as historical, per the caution in [[#Running BOM]].

**Housekeeping**: the two `.lck` files are present and freshly stamped, i.e. KiCad genuinely had the project open — these are not the 08-07 crash leftovers.

## Next steps

**Refreshed 2026-08-11 against the real files** (the 2026-08-10 list had gone stale on two items — see [[#2026-08-11 — Schematic build state (measured from the files)]]). Ordered by what blocks what: 1 → 3 → 4 gates PCB work; the rest can run in parallel.

- ~~Wire the root page~~ — **done 2026-08-14**, verified against the files: [[#2026-08-14 — Root sheet wired; PWR_FLAGs added (verified from the files)]].
- ~~Finish `display_and_audio`~~ — **done 2026-08-14**, verified: [[#2026-08-14 — display_and_audio re-verified: far more complete than this table shows]]. `R20`'s missing footprint rolls into the footprint pass below.
- **Buttons/joystick physical layout** — switch part and count are decided (10× Omron B3F-1000, board-mounted, all 10 already placed as SW3–SW12), but the actual controller-shaped arrangement (D-pad, face buttons, spacing) hasn't been designed.
- **Standoff/mounting placement** — no longer blocked. Real coordinates for the display are in [[#2026-08-10 — Panel decision superseded: Adafruit 3.5" TFT Capacitive Touch Breakout (HX8357D)]] (pulled from Adafruit's own Eagle file). The joystick part's footprint still needs real dimensions once it's back in scope.
- ~~ERC cleanup — PWR_FLAG~~ — **done 2026-08-14**, and it turned out to need 3 flags not 2 (`VBUS`, `5V+`, `3V3+`) — see [[#2026-08-14 — Root sheet wired; PWR_FLAGs added (verified from the files)]]. Still worth running a real ERC pass in KiCad to confirm nothing else is undriven.
- **Footprint assignment — spec drafted 2026-08-14**, see [[#2026-08-14 — Footprint-assignment spec, and pre-layout checklist]]: batch-generic parts, real-part matches (`J2`, `L1`), and two picked-not-defaulted parts (`J1`, `F1`) all specified. Applying it in the GUI is still Nelson's step — needs finishing before layout can start (`esp32build.kicad_pcb` is still a 79-byte stub). **Still the top blocker.**
- **Display bus never wired past its own sheet — found 2026-08-14**, see [[#2026-08-14 — Display bus never reached the MCU/expander: found and specified]]. 12 signals now needed (touch's `display_INT` was dropped — see panel-swap update below), new labels on `core.kicad_sch`/`user_in_out.kicad_sch`, new sheet pins, and root-page wiring, plus a label-shape fix (`DAT0`–`DAT7` are `output`, should be `input`) on `display_and_audio.kicad_sch` before wiring the other end. Also blocks a clean ERC pass.
- **Panel picked: Adafruit #1770 (2.8" ILI9341)** — see [[#2026-08-14 — Panel decision superseded again: Adafruit 3.2" TFT Touchscreen Breakout (ILI9341), touch dropped]]. Still open: verify #1770's pinout matches #1743's 12-pin bus 1:1 before wiring (not yet diffed), re-pull real mounting/cutout coordinates for this panel, decide `J4`'s connector/footprint for the pin-header form factor, and decide whether to use the panel's onboard microSD socket instead of `J2` or leave it unpopulated.
- **Pre-layout/placement/routing checklist drafted 2026-08-14** — see [[#2026-08-14 — Footprint-assignment spec, and pre-layout checklist]] for placement order, antenna keepout, and which nets to hand-route before trusting the autorouter.
- **Minor cosmetic fixes**: `I2C_SDA`/`I2C_SCL` global label shape on `core.kicad_sch` is `input`, should be `bidirectional`; `storage.kicad_sch` has a duplicate `3V3+` global label with mismatched shapes (`bidirectional` vs `input`).
- Confirm the DM3AT card-detect switch polarity against the DM3 datasheet's switch timing chart (assumed normally-open) — tidiness item, firmware can invert either way.
- Decide whether the optional SD ESD array goes on the first spin.
- Run the drafted schematic against the full [ESP32-S3 Schematic Checklist](https://docs.espressif.com/projects/esp-hardware-design-guidelines/en/latest/esp32s3/schematic-checklist.html) — not just this note's summary.
- **Battery items — deferred, v1 is USB-only** ([[#2026-08-10 — Battery deferred to a later revision]]). Kept for whenever a battery rev happens: rebuild the power section (BQ24074 or the NiMH/2S2P + LTC4060 alternative), pick a cell/holder, add a JST-PH battery connector ([[#Battery connector — forward-looking note, not yet built]]), and recompute the power/runtime budget for whichever topology gets picked.
- **Camera, second display, and a possible ESP32-P4** — all considered and explicitly not pursued this rev; real research kept in [[#2026-08-10 — P4 / second-screen / camera / foldable — considered, not pursued]] if reconsidered later.
- `logo.drawio` in the project folder is a half-finished logo sketch (circle + six strokes, no text) — presumably silkscreen art. Finish or drop it.
- ~~Unblock the Konnect KiCad↔Claude bridge~~ — **abandoned 2026-08-10.** Working manually instead: Claude drafts parts/values/wiring specs from datasheets, Nelson enters them in KiCad. Detail in [[CLAUDE_KICAD_SETUP]].

## 2026-08-14 — Root sheet wiring spec (drafted, not yet wired)

Pin positions read directly from `esp32build.kicad_sch` (Konnect stays off — confirmed again, working manually per [[CLAUDE_KICAD_SETUP]]). All 26 sheet pins pair up 1:1 across the five sheets with no orphans, and every pair's direction is electrically sane — no output-output conflicts, every input has exactly one driver.

**13 connections needed**, grouped by bus:
- **USB**: `core.USB_Data+` ↔ `power.USB_Data+`; `core.USB_Data-` ↔ `power.USB_Data-`
- **SDMMC**: `core.sd_clk`/`sd_cmd`/`sd_dat0`/`sd_dat1`/`sd_dat2`/`sd_dat3` ↔ matching `storage.*` pins (core drives clk; cmd/dat are bidirectional both ends)
- **Card detect**: `core.sd_detb` (input) ↔ `storage.sd_detb` (output)
- **IO expander interrupt**: `core.IO_EXP_INT` (input) ↔ `user_input.IO_EXP_INT` (output)
- **I2S audio**: `core.I2S_BCLK`/`I2S_DIN`/`I2S_LRCLK` (outputs) ↔ matching `display_and_audio.*` inputs

**Two placement snags to sort before/while wiring:**
- `storage.sd_detb`'s sheet pin sits on the sheet symbol's **right edge** (x=162.56) while its six sibling SD pins are all on the **left edge** (x=119.888) — a direct wire from `core` would have to route around or over the storage sheet block. Drag it to the left edge to match its siblings first.
- `core.IO_EXP_INT` (left edge of `core`, facing away from everything) and `user_input.IO_EXP_INT` (bottom-right of the page, facing right) have no clean line of sight — `core` sits top-left, `user_input` sits at the bottom of the page. Either drag both pins to face each other and wire with bends, or skip the direct wire and drop a **global label** at each pin instead (this project already uses global labels for `I2C_SDA`/`SCL` and `3V3+`, so it's a consistent pattern, not a new one).

Not decided: whether to global-label everything given how scattered the sheet layout is, or wire the adjacent buses directly and label-only the awkward diagonal (`IO_EXP_INT`). Nelson's call in the KiCad GUI.

## 2026-08-14 — Root sheet wired; PWR_FLAGs added (verified from the files)

**Root sheet wiring done and verified.** Traced all 37 wire segments in `esp32build.kicad_sch` against the 13 required nets — every segment lands exactly on a pin coordinate, every wire is used exactly once, no strays, no shorts against unrelated pins. Both placement snags from the drafted spec got fixed in the process: `storage.sd_detb` was moved to the sheet's left edge to match its siblings, and `user_input` was repositioned right next to `core` so `IO_EXP_INT` connects with a plain direct wire (no global label needed after all). All 13 connections confirmed correct.

**PWR_FLAG — 3 needed, not 2.** The 08-11 audit only called for VBUS and 3V3+, but the real fix needs a third: `power.kicad_sch` has `F1` (polyfuse) between `VBUS` and a separate `5V+` net — the fuse splits them into two distinct nets exactly the way `L1` splits the regulator's `SW` pin from `3V3+`, so `5V+` has no `power_out`-typed pin on it either. All three flags are now placed and confirmed at the correct nets (`VBUS`, `5V+`, `3V3+`).

General ERC/PWR_FLAG rule confirmed against the actual symbols (not assumed): a net needs a flag only if it has a power-input pin but no pin on that *same* net is typed `power_out`. Connector pins (`J1`'s `VBUS`) are always `passive`, never `power_out`. A switching regulator's own `power_out` pin (`U2`'s `SW`) sits on the switch node, not on the filtered output rail — so the output rail needs its own flag too, unlike an LDO whose `VOUT` pin sits directly on the output net.

## 2026-08-14 — display_and_audio re-verified: far more complete than this table shows

Traced every pin in `display_and_audio.kicad_sch` by coordinate rather than trusting the 08-11 table above, which turned out to be stale on this sheet specifically (it likely reflected an earlier save). Nearly everything is done.

**Ref-designator swap, note for future reads**: `J3` and `J4` have swapped meaning since the 08-11 table — **`J3` is now the 2-pin speaker connector** (`Conn_01x02`, Molex PicoBlade footprint already assigned) and **`J4` is now the 20-pin display header** (`Conn_01x20`, `PinHeader_1x20_P2.54mm` footprint already assigned). Same drift pattern as the R9/R15/R16 caution in [[#Running BOM]] — trust the live file over any ref number quoted in an older section, including this one once it too goes stale.

**`J4` (20-pin display header) — all 20 pins wired correctly**, matching [[#Connector: JP1, `Conn_01x20` in KiCad — full pin-by-pin confirmed from the real Eagle file]] exactly: both GNDs (pins 1, 12), `+5V`→`5V+` (pin 2), `RD` tied high to `3V3+` (pin 6), the 8-bit data bus D0–D7 (pins 13–20) and CS/C-D/WR/RST/LITE (pins 3,4,5,7,8) as hierarchical labels, touch I2C SCL/SDA (pins 10,11) on the shared bus, and `CTP_IRQ`→`display_INT` (pin 9).

**`U6` (MAX98357A) — everything wired except VDD decoupling**: `DIN`/`BCLK`/`LRCLK` → I2S hierarchical labels ✓; `GND` (pins 3/11/15) and the exposed thermal pad (pin 17) both grounded via power symbols sitting directly on the pin coordinates ✓; `VDD` (pins 7/8) → `5V+` ✓; `GAIN_SLOT` tied to `5V+` (the decided 6dB VDD-tie) ✓; `SD_MODE` → `R20` (634k) → `3V3+` ✓; `OUTP`/`OUTN` → `J3` speaker pins 1/2 ✓. **Only the 10µF + 0.1µF VDD bypass caps are actually missing** — no `Device:C` symbol exists anywhere in this sheet yet.

**Two small loose ends, not yet resolved:**
- `R20` has no footprint assigned (blank field) — needs one whenever the footprint-assignment pass happens, despite everything electrically around it being done.
- An unattached `no_connect` flag sits at (149.86, 58.42) that doesn't align with any traced pin on `U6` or `J4` — worth a look in the GUI; may be a harmless leftover.

**Correction to [[#Layout notes]]'s speaker note**: that section says to add a text annotation "near J4" about the sealed back-volume cavity and antenna keepout — given the ref swap above, that annotation belongs **near `J3`** (the actual speaker connector) now, not `J4`.

**Update, same day**: both VDD bypass caps added and verified — `C10` (10µF) and `C11` (0.1µF), both on the `5V+`/VDD rail with grounded bottom pins, landing exactly on U6's VDD pin coordinate. The stray `no_connect` flag is gone. **`display_and_audio` is now fully wired** — the only thing left on this sheet is `R20`'s missing footprint, folded into the project-wide footprint pass below.

## 2026-08-14 — Footprint-assignment spec, and pre-layout checklist

**Batch footprint plan**, split by how settled each part is:

- **Already researched, just apply**: `J2` → `Connector_Card:microSD_HC_Hirose_DM3AT-SF-PEJM5`. `L1` → **Coilcraft XAL4020-102MEC**, footprint `L_Coilcraft_XAL4020-102` — see [[#2026-08-14 — L1 switched: XGL3520 → XAL4020 (no footprint import needed)]].
- **Common/generic, safe to batch**: all resistors → `Resistor_SMD:R_0603_1608Metric`; all capacitors → `Capacitor_SMD:C_0603_1608Metric`; `D1`–`D3` → `LED_SMD:LED_0603_1608Metric`; `SW3`–`SW12` (Omron B3F-1000 ×10) → `Button_Switch_THT:SW_PUSH_6mm` (verify pad spacing against the real B3F-1000 drawing before applying to all 10); `SW1`/`SW2` (EN/BOOT) → `Button_Switch_SMD:SW_SPST_B3U-1000P` (same switch Espressif's own DevKit boards use for EN/BOOT).
- **Picked, not just defaulted**: `J1` (USB-C) → `Connector_USB:USB_C_Receptacle_GCT_USB4085`, a common JLCPCB/hobby-board choice. `F1` (polyfuse) → **Bel Fuse 0ZCJ0110AF2E**, 1.1A hold / 2.2A trip, 0603 (`Fuse:Fuse_0603_1608Metric`) — sized with margin over the board's ~900mA USB power budget rather than blanket-defaulted to 0603 like the other passives, since a fuse's hold current is a real electrical constraint, not just a size preference. **Value field: `1.1A`**, matching the note's convention of using the defining spec as Value (like `1.0µH` for `L1`).
- Exact footprint library names above are best-known standard KiCad names, not verified against this machine's actual install (couldn't locate the real footprint library path from this session) — confirm spelling in the Assign Footprints browser.
- **How KiCad "knows" something is a fuse vs. a resistor**: the **symbol** (`Device:Polyfuse`, already correctly placed for `F1`), not the footprint — footprints are just copper land patterns and carry no part-type meaning. A 0603 fuse and 0603 resistor footprint are pad-for-pad identical.

**Pre-layout / pre-autorouting checklist**, compiled from every keepout/routing note already scattered through this doc:

1. **Placement order**: module first (respecting antenna keepout, non-negotiable — see [[#WROOM-2-N32R16V pin-specific circuit]] item 5) → `J1` at the board edge → decoupling caps snapped to their IC pins → mechanical anchors (display header at the real Adafruit mounting coordinates, SD socket at the edge, speaker near the enclosure's sealed cavity, USB-C at the edge) → buttons SW3–SW12 (still-open ergonomic layout, per [[#Next steps]]) → remaining passives last.
2. **Hand-route, don't trust the autorouter blind**: USB `D+`/`D−` (matched differential pair), the Class-D speaker `OUT+`/`OUT−` (short, tight, away from the antenna keepout, display bus, and SD clock — see [[#Layout notes]]), and the SD 4-bit bus at up to 40MHz.
3. **Draw a keepout/rule area over the antenna zone before autorouting** — otherwise the autorouter has no reason to avoid it.
4. **Ground-plane ties to double check**: module EPAD, `U6` pin 17 (exposed pad), MCP23017 exposed thermal pad, SD socket `VSS`/`SHIELD`, USB-C shield — all flagged individually elsewhere in this note as needing the ground plane, easy to miss in a placement pass.
5. **After routing**: run DRC (separate from ERC), visually re-confirm nothing crosses the antenna keepout, check the 3D viewer for mechanical fit before ordering.

**Further research**: the WROOM-2 datasheet's own "PCB Layout Recommendations" section has the authoritative antenna keepout dimensions — worth having open during module placement rather than working from the summary above alone.

## 2026-08-14 — L1 switched: XGL3520 → XAL4020 (no footprint import needed)

The Coilcraft XGL3520-102MEC doesn't have a stock KiCad footprint, and importing a custom one wasn't worth the trouble. Checked the real installed library rather than guessing at an alternative.

**KiCad 10's actual footprint library location on this machine** — for future reference, since this wasn't obvious: `/mnt/c/Users/inelson/AppData/Local/Programs/KiCad/10.0/share/kicad/footprints/`, **not** `Program Files` (not found there) and **not** the OneDrive-synced `Documents/KiCad/10.0/footprints` (that's just the empty user-config override dir). `Inductor_SMD.pretty` (692 footprints) lives there, and it has no Coilcraft **XGL** entries — but it does carry the whole Coilcraft **XAL** family by name.

**Switched to Coilcraft XAL4020-102MEC** (datasheet saved to `projects/esp32build/datasheets/Coilcraft_XAL4020_XAL4030_XAL4040_806-1_Rev2026-02-25.pdf`):

| | XGL3520-102MEC (old) | XAL4020-102MEC (new) |
|---|---|---|
| L | 1.0µH | 1.0µH |
| DCR | 14.8mΩ max | 14.6mΩ max — essentially unchanged, efficiency shouldn't move |
| Isat | 5.4A | **8.7A** — more margin over the 2A load |
| Package | 3.5×3.5×2.0mm | 4.3×4.3×2.1mm — **0.8mm bigger per side**, confirm board room near U2 |
| Stock | — | 10,973 @ LCSC, $3.58/pc, no lifecycle flag |

**Footprint: `L_Coilcraft_XAL4020-102`, already in `Inductor_SMD.pretty` — no import needed.** Verified by opening the `.kicad_mod` directly and checking pad geometry (0.98mm × 3.4mm pads at ±1.185mm offset) against the datasheet's documented land pattern — a real match, not just a name coincidence.

**Bigger-margin alternative if board space isn't tight**: Coilcraft XAL7020-102MEC (18A Isat, 8×8mm, footprint `L_Coilcraft_XAL7020-102` also already present) — datasheet saved to `projects/esp32build/datasheets/Coilcraft_XAL7020_871-1_Rev2026-02-25.pdf`. Not needed unless the extra thermal margin is wanted; XAL4020 already clears the 2A requirement comfortably.

## 2026-08-14 — Display bus never reached the MCU/expander: found and specified

Traced `esp32build.kicad_sch`, `core.kicad_sch`, and `user_in_out.kicad_sch` by coordinate after Nelson noticed unconnected pins on the display module. Root cause: `display_and_audio`'s 14 display-bus hierarchical labels (`display_DAT0`–`DAT7`, `CS`, `CD`, `WR`, `RST`, `LITE`, `INT`) are wired correctly to `J4` locally, but the `display_and_audio` sheet symbol on the root page only exposes 3 sheet pins (`I2S_BCLK`/`DIN`/`LRCLK`) — no sheet pin exists for any display signal, and `core.kicad_sch`/`user_in_out.kicad_sch` have zero matching labels. The MCU/expander side of this bus was simply never wired.

**Free-pin trace (coordinate-verified, not guessed):**
- `U1` (WROOM-2) free GPIOs: `IO1, IO2, IO4–IO10, IO17, IO18, IO48` — exactly 12, matching the 12 direct signals needed. `IO3` is unavailable (strapping-reserved) so the run isn't contiguous.
- `U4` (MCP23017) free pins: `GPB3–GPB7` — 5 spare (11/16 already used by 10 buttons + 1 status LED).

**Assignment:**

| Signal | Pin | Direction (this sheet → display) |
|---|---|---|
| `display_DAT0`–`DAT7` | `U1` `IO1, IO2, IO4, IO5, IO6, IO7, IO8, IO9` | output |
| `display_WR` | `U1` `IO10` | output |
| `display_CS` | `U1` `IO17` | output |
| `display_CD` | `U1` `IO18` | output |
| `display_LITE` | `U1` `IO48` | output |
| `display_RST` | `U4` `GPB3` | output |
| ~~`display_INT`~~ | ~~`U4` `GPB4`~~ | **dropped 2026-08-14 — touch removed, see [[#2026-08-14 — Panel decision superseded again: Adafruit 3.2" TFT Touchscreen Breakout (ILI9341), touch dropped]]** |

`GPB5`–`GPB7` stay spare. Not yet confirmed: whether the S3's LCD_CAM/i80 peripheral has the same GPIO-matrix routing freedom already confirmed for SDMMC — check Espressif's LCD_CAM docs before treating this as final, same caution already on record for SPI above 40MHz.

**Label-shape bug found in `display_and_audio.kicad_sch`:** `display_DAT0`–`DAT7` are set to shape `output`, but every other control line (`WR`/`CS`/`CD`/`RST`/`LITE`) is correctly `input` and `display_INT` is correctly `output`. Since the MCU writes pixel data into the display (no `RD` signal — `J4` pin 6 is tied straight to `3V3+`, confirmed above), the 8 data-line labels should be `input` like their siblings, not `output`. Left as-is, wiring the new MCU-side labels as `output` (correct for GPIOs that drive the bus) would put two `output`-shaped ends on the same net with ERC — a driver conflict. **Fix the 8 `DAT` labels on the display sheet to `input` before wiring the other end.**

**Wiring mechanics** (same process as the root-sheet pass): add the 12 hierarchical labels above in `core.kicad_sch`, and 1 (`display_RST` only — `display_INT` dropped, see below) in `user_in_out.kicad_sch`, then on the root page Import Sheet Pin on `core`↔`display_and_audio` (12 signals) and `user_in_out`↔`display_and_audio` (1 signal), and wire the new pairs on the root page.

**Other ERC items already on record, expected to also show up in a real ERC pass:** the two "Minor cosmetic fixes" in [[#Next steps]] (`I2C_SDA`/`SCL` label shape on `core.kicad_sch`; duplicate `3V3+` shape mismatch on `storage.kicad_sch`). `J1`'s `VCONN` pin (meant to be left unconnected per [[#Powering from USB-C]]) — not confirmed whether it already has an explicit no-connect flag; check in the GUI if ERC flags it. `R20`'s missing footprint is **not** an ERC item — that only affects footprint/BOM checks.

## 2026-08-14 — Panel decision superseded again: Adafruit 3.2" TFT Touchscreen Breakout (ILI9341), touch dropped

Nelson is swapping away from the HX8357D 3.5" panel to [Adafruit #1743](https://www.adafruit.com/product/1743), checked against Adafruit's own product page and pinout guide rather than assumed. Real deltas:

| | HX8357D 3.5" (superseded) | ILI9341 3.2" (current) |
|---|---|---|
| Resolution | 320×480 | 240×320 |
| Interface | 8-bit parallel | 8-bit parallel (also supports SPI, not used here) |
| Touch | FT6206 capacitive, I2C + IRQ | Resistive (`Y+`/`X+`/`Y-`/`X-`), no I2C — **dropped, see below** |
| Onboard SD | none | has its own microSD socket — redundant with `J2`; not yet decided whether to use it instead or leave unpopulated |
| Connector | 20-pin FPC (drove `J4`'s footprint choice) | pin-header breakout — `J4`'s footprint choice needs revisiting once the part's in hand |

**Touch dropped entirely** (Nelson's call — not essential for this device). This changes the [[#2026-08-14 — Display bus never reached the MCU/expander: found and specified]] plan:
- Delete `display_INT` — no touch IRQ signal needed at all, and the panel's resistive `Y+`/`X+`/`Y-`/`X-` pins are left unconnected.
- The 8-bit parallel bus assignment is unaffected — `D0`–`D7`, `WR`, `CS`, `CD`, `RST`, `LITE` still map to `IO1, IO2, IO4-IO10, IO17, IO18, IO48` (ESP32) and `GPB3` (expander), exactly as already specified. `RD` still ties high (unused).
- Expander now has **4 spare pins** (`GPB4`–`GPB7`), up from 5 minus the 1 `RST` uses — `GPB4` was earmarked for `display_INT` and is no longer needed.

**Not yet re-pulled**: real mounting/cutout coordinates for this panel (the logged ones are from the old HX8357D's Eagle file and no longer apply) and `J4`'s connector/footprint choice for the new pin-header form factor.

**Update, same day — sized down again to Adafruit #1770 (2.8").** #1743 felt too big once in hand. Dispatched a real stock/spec check (not assumed) across Adafruit, DigiKey, and generic-module vendors for a smaller 8-bit-parallel alternative, since most cheap SPI-labeled ILI9341 boards on Amazon/AliExpress only break out the SPI pins even though the chip itself supports parallel — a real trap, ruled those out explicitly.

**Picked: [Adafruit #1770](https://www.adafruit.com/product/1770)** — 2.8" ILI9341, 240×320, $29.95, 54 in stock at check time. Same controller as #1743 and documented on the *same* Adafruit Learn guide ("2.8 and 3.2 Color TFT Touchscreen Breakout v2"), which strongly suggests — but doesn't yet confirm — the same breakout PCB/pin layout as #1743 just with a smaller panel, meaning the 12-pin bus wiring above likely carries across unchanged. **Not yet pin-for-pin verified against #1743** — do that diff before assuming zero rewiring.

Also considered and ruled out: **Adafruit #2478** (2.4", same ILI9341 family, otherwise the natural next-size-down) — genuinely out of stock everywhere, DigiKey quoting an 18-week lead time, not buyable now. **BuyDisplay ER-TFT028A2-4** — same controller, ~$7, but a bare panel on a 50-pin FPC/ZIF connector, not a breakout; would need a custom carrier PCB and ZIF socket, real extra engineering effort not justified here.

## Related
- [[CLAUDE_KICAD_SETUP]] — tooling-side notes for this same board: enabling KiCad's IPC API, installing the Konnect plugin, and the still-unresolved "Pending approval" blocker. This note owns the *design* decisions; that one owns the *bridge* setup. It lives in `projects/esp32build/`, so it moves with the KiCad project.
- Otherwise standalone by design — no links into the Tritium-work notes elsewhere in this vault, so this note and the project folder can be relocated together without breaking anything.

### If moving this off the work machine

Three things travel as a unit:

1. `projects/esp32build/` — the whole folder. Since the 2026-08-11 vault restructure this holds
   *both* the KiCad project (including `.history/`, a real git repo of every save) **and** this
   file, alongside `CLAUDE_KICAD_SETUP.md`. Moving the folder now moves everything.
2. Nothing else — there are no other inbound or outbound vault links

Loose ends to tidy on arrival: remove the entries from the work vault's `INDEX.md`, and re-point the `.mcp.json` in the project folder, which currently holds a Windows path to `konnect.exe` under the work profile.
