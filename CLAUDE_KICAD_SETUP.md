# Claude Code <-> KiCad bridge setup

Notes on connecting Claude Code directly to this KiCad 10 project via the IPC API,
instead of screenshot-based review.

## Status: abandoned 2026-08-10 — going manual instead

Decided not to keep chasing the "Pending approval" block below. Workflow going forward:
Claude drafts parts/values/wiring specs from datasheets, Nelson enters them into KiCad by
hand. No netlist round-trip needed for that — screenshots/exports only if a specific review
question comes up. This file is kept as a record of what was tried, not an open task.

## What's done
1. **KiCad IPC API enabled**: Preferences -> Plugins -> Enable IPC API Server.
2. **Konnect plugin installed** via KiCad's Plugin and Content Manager (PCM), listed under
   the `mixelpixx` package (`com_github_mixelpixx_konnect`), version 0.2.2.
   - Installed binary: `C:\Users\inelson\OneDrive - Tritium Power Solutions\Documents\KiCad\10.0\3rdparty\plugins\com_github_mixelpixx_konnect\bin\konnect.exe`
   - Downloaded packages still sitting in `Downloads\`: `Konnect-main.zip` (source),
     `konnect-pcm-v0.2.2-windows.zip` (the PCM package actually installed).
3. **`.mcp.json` created** in this project root, pointing Claude Code at `konnect.exe`
   (using its WSL mount path, since this Claude Code session runs inside WSL and invokes
   the Windows binary through WSL/Windows interop).

## To actually use it
- Keep KiCad open with this project loaded (Konnect talks to a *running* KiCad instance
  via its IPC socket — it doesn't work headless).
- Restart/reload the Claude Code session from this project folder so it picks up
  `.mcp.json`. Ask Claude to check `ToolSearch` for kicad/konnect tools to confirm the
  connection came up.

## Known caveats
- **WSL interop is NOT the problem — verified 2026-08-07.** Konnect's listed OS support is
  Windows/macOS only, but launching the Windows `konnect.exe` from a WSL process works fine:
  piping a raw `initialize` request into it returns a valid MCP handshake
  (`serverInfo: konnect v0.2.2`, protocol `2025-06-18`). Rule this out before debugging further.
- **Actual blocker as of 2026-08-07: the server sits at "Pending approval".** `claude mcp list`
  reports `konnect: ... - ⏸ Pending approval`. Claude Code won't load a project-scoped
  `.mcp.json` server until the startup approval prompt is answered, and no konnect tools appear
  in `ToolSearch` until then. Reloading KiCad and the Claude Code session did not clear it, and
  the approval prompt never appeared — **unresolved, parked**. Next things to try when picked back
  up: `/mcp` in-session, and `claude mcp reset-project-choices` followed by relaunching `claude`
  from this folder to re-trigger the prompt.
- This project folder lives inside the Tritium Work OneDrive vault
  (`Tritium Work/projects/esp32build/`) even though the ESP32 board is a personal
  project, not Tritium work — worth relocating later if that commingling isn't wanted.

## Related
- [[esp32-board-design]] — personal-project EE notes/decisions for this board (module choice,
  USB-C circuit, buck converter, microSD, indicator LEDs, and a dated snapshot of the current
  schematic build state). **That note is the source of truth for design decisions**; this file is
  just the tooling/bridge setup. Its `## Next steps` carries the reminder to unblock the approval
  issue described above.
