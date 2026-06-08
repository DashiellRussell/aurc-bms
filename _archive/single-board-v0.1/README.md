# Archived — single-board attempt (v0.1)

This is the original **single-board** AURC_BMS layout, archived on 2026-06-08 when the design was
split into the three-board set (`board1_power`, `board2_monitor`, `board3_dist`) per
`docs/architecture_v0.2.md`.

Kept for reference only — do **not** continue work here.

Contents:
- `AURC_BMS.kicad_pro` / `.kicad_sch` / `.kicad_pcb` — top-level single-board project
- `charge_chanel.kicad_sch`, `USB_IN.kicad_sch` — hierarchical sheets (per-channel charger, USB input)
- `sym-lib-table`, `fp-lib-table` — original project-local lib tables (referenced the
  `easyeda2kicad` library nicknames, since renamed to `BMS` and relocated to `../../libs/`)

The schematics embed their symbol definitions (`lib_symbols`), so they still open and display even
though the library was renamed. Footprint/3D-model links may not resolve against the new `libs/`
layout — expected, since this is frozen reference material.
