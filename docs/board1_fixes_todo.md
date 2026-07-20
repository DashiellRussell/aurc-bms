# Board 1 (main power board) — outstanding fixes

Found while building board4 (nosecone), which was copied from board1 — these bugs exist on
**board1_power** too and need fixing there. Board1 already has placed copper, so both are a
schematic **and** layout touch.

## 1. PD voltage-select straps incomplete (CFG2/CFG3 floating)

`board1_power/USB_IN.kicad_sch` — the CH224K is **not deterministically set to 12 V**. As drawn:
- CFG1 → 24 K to GND = logic 0 ✅
- **CFG2 → floating** ✗
- **CFG3 → floating** ✗

Floating CFG pins have no reliable internal default (datasheet: "add external pull-up/pull-down to
set CFG2/CFG3"), so the negotiated voltage is undefined — it may sit at 5/9/12 V unpredictably. The
4-cell board needs a real 12 V (≈30 W budget; a 9 V/2 A phone brick can't feed it), so silently
running at 9 V would input-current-limit and charge slow.

**Fix — set 12 V (CFG1=0, CFG2=0, CFG3=1):**
- CFG2 → **0 Ω to GND** (`R_0603_1608Metric`)
- CFG3 → **~10 K pull-up to `VDD_PD`** (the CH224K's own VDD rail, **not** VBUS and **not** +3V3 —
  must be ≤3.7 V; VDD_PD is alive the instant the chip powers, so no timing dependency)

*(For reference, board4/nosecone uses 9 V = all three CFG to GND — simpler, no pull-up needed.)*

## 2. Heartbeat LED has no series resistor

`board1_power` root — the heartbeat LED **D1** is wired ESP `IO2` (pin 38) → D1 → GND with **no
current-limiting resistor**. An LED straight across a 3.3 V GPIO is limited only by the pin's drive
strength (ESP32-S3 ~40 mA abs max) — stresses the GPIO and over-drives the LED.

**Fix:** insert a series resistor, `IO2 → R → D1 anode`, D1 cathode → GND.
- **1 kΩ** recommended (≈1.3 mA, plenty for a status blink, easy on the battery), `R_0603_1608Metric`.
- (This same bug was pasted into board4 and is being fixed there too.)

---

*Both fixes were identified 2026-07-21. Neither blocks the board from powering, but #1 means the
charge voltage is wrong/undefined and #2 stresses a GPIO — do both before the next board1 revision.*
