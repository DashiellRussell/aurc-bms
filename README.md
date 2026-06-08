# AURC_BMS

Battery Management System for AURC (Australian Universities Rocket Competition).
Four-channel battery board set feeding two flight computers (FC1, FC2) and two pyro boards.
KiCad 10 project set.

## Layout

```
AURC_BMS/
├── libs/                     ← shared, single source of truth (all three boards point here)
│   ├── BMS.kicad_sym         ← symbols
│   ├── BMS.pretty/           ← footprints
│   └── 3dmodels/             ← .step / .wrl
├── docs/
│   ├── interconnect.md                ← the connector contract — READ THIS before touching a board edge
│   ├── architecture.md                ← full design rationale (living doc, currently v0.3)
│   └── board1_power_capture_plan.md   ← Board 1 schematic capture plan (sheets, netlist, BOM)
├── board1_power/             ← Board 1: USB-C, PD sink, 4× SY6970 smart charger, cells, ESP+I²C mux
├── board2_monitor/           ← Board 2: 64-LED gauge + driver, 4× pull-pin power-enable (spine)
├── board3_dist/              ← Board 3: passive fan-out to FC1+Pyro1 / FC2+Pyro2
└── _archive/                 ← previous single-board attempt (v0.1), kept for reference
```

Each `boardN_*/` is an independent KiCad project (`.kicad_pro` / `.kicad_sch` / `.kicad_pcb`) with
its own `sym-lib-table` + `fp-lib-table` that resolve the **shared** library at `../libs/` under
the nickname **`BMS`**. Add a part once in `libs/`; all three boards see it.

## Working on it

- **Before changing any board-to-board connector, edit `docs/interconnect.md` first.** It is the
  only thing keeping the three separately-laid-out boards mate-compatible.
- Add/edit symbols in `libs/BMS.kicad_sym`, footprints in `libs/BMS.pretty/`, 3D models in
  `libs/3dmodels/`. Don't make per-board copies — that defeats the single-source-of-truth setup.
- Open a board by opening its `boardN_*/boardN_*.kicad_pro`.

See `docs/architecture.md` for the full design (protection stack, charging, monitoring,
power-enable/safety, BOM, open items).
