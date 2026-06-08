# AURC Avionics BMS — Architecture & Design

**Version:** 0.3 (parts re-confirmed against existing capture; Board 1 plan in `board1_power_capture_plan.md`)
**Scope:** Four-channel battery board set feeding two flight computers (FC1 primary, FC2 backup) and two pyrotechnic boards, for AURC.
**Status:** Board 1 schematic capture in progress. Remaining open items in §13.

> **v0.3 change (2026-06-08):** charger/monitoring topology switched from *MP2617 + MAX17048* to the
> **SY6970** smart charger already drawn in the existing capture. The SY6970 (BQ25896-class) does
> power-path charging **and** integrated I²C V/I/charge monitoring in one IC, so the separate
> MAX17048 fuel gauge is dropped. The TCA9548A mux stays (4× SY6970 share addr 0x6B). ESP confirmed
> as **ESP32-S3-WROOM-1**. Tradeoff: ADC-grade monitoring instead of true coulomb-counting SoC —
> accepted for the lower part count. See §5, §6, §11.

---

## 1. Design philosophy

- **Four electrically independent 1S channels.** One 18650 per channel: FC1, FC2, Pyro1, Pyro2. Independent positive rails; returns joined at a single star point. No series stack → **no cell balancing**.
- **The monitoring layer is non-flight-critical.** The ESP, LED gauge, and BLE telemetry sit entirely outside the flight power path. Fallback is always "fit a charged cell and fly." Nothing in the flight or deployment path may depend on the ESP — hard rule.
- **No on-board arming, no pyro circuitry.** This board does not arm or fire anything. The pull pins are per-channel **power-enable / battery isolation** only. All pyro firing (e-match, firing FET, opto, continuity) lives on the pyro boards downstream; the BMS only delivers power to their inputs.
- **Remove-before-flight power enable.** Pull pin inserted = channel isolated (safe to handle). Pull pin removed = channel power flows.
- **Three boards.** Board 1 (power/charge) and Board 2 (monitor) are active; Board 3 (distribution) is passive.

---

## 2. System architecture

| Board | Role | Key contents |
|---|---|---|
| 1 — Power & charge | active | USB-C (charge + ESP programming), PD sink, 4× SY6970 smart chargers (charge + integrated I²C monitoring), 4× 18650 holders, per-cell protection, I2C mux, ESP module, indicator LEDs |
| 2 — Safe & monitor | active | 64-LED gauge + IS31FL3731 driver, gauge button; pull-pin power-enable interface at the 2→3 edge |
| 3 — Distribution | passive | fan-out to FC1+Pyro1 (Conn A) and FC2+Pyro2 (Conn B) |

**Mechanical (to confirm):** Boards 1 and 3 stack parallel; Board 2 is the spine/interconnect carrying the gauge and the pull-pin interface. Board-to-board via **stacking mezzanine connectors**.

**Inter-board connectors**

- **Board 1 ↔ 2:** 8-pin power (4× V + 4× GND, 3 A/pin) + 4-pin signal (I2C to gauge driver, plus channel-live sense + GND).
- **Board 2 ↔ 3:** 8-pin power (4× V + 4× GND, 3 A/pin) routed through the pull-pin power-enable interface.
- **Board 3 → loads:** Conn A (FC1 V/GND + Pyro1 V/GND), Conn B (FC2 V/GND + Pyro2 V/GND). Per-channel V and GND kept separate.

---

## 3. Channel & independence model

- Four independent positive rails; a fault on one channel cannot pull down another's supply.
- Grounds meet at a **single star point** (location TBD, likely Board 1 by the cells), not a shared plane — independent returns, OZONE PYRO_GND philosophy.
- One USB-C port references the four chargers to a common input ground during charging; acceptable because flight returns are independent and star-bonded at one point. No galvanic isolation.

---

## 4. Per-cell protection stack

All four channels, in series order from the cell outward:

1. **Reverse-polarity FET** — low-Rds P-FET (DMG2305UX / SI2333-class), not a Schottky.
2. **Fuse / PTC** — sustained-short backstop. Resettable PTC preferred (passes the brief 2 A firing pulse, trips on a sustained short, self-resets).
3. **Protection IC** — DW01A-class + dual N-FET (FS8205-class): over-charge, over-discharge (~2.5 V cutoff, matches the P30B), over-current.
4. **Power-path charger (SY6970)** — switching power-path, serves the system rail before topping the cell — see §5.

**Pyro-channel exception:** the protection FET carries the firing pulse, so set its over-current trip **above** worst-case firing inrush by using a lower-Rds dual FET. The fuse/PTC is the catastrophic-short backstop. The pyro cell barely cycles (a few mAh/shot), so over-discharge risk is low.

**Charge temperature cutoff:** NTC on each charger's TS pin to inhibit charging outside ~0–45 °C.

**Over-charge IC stays** even though the charger terminates at 4.2 V — it's the backstop against a charger that fails shorted.

---

## 5. Charging & USB-C

**Single USB-C port, dual purpose:** PD charging (VBUS + CC) and ESP programming (D+/D- to the S3 module's native USB). Separate pins, no conflict.

- **PD sink** requests 9–12 V (strap-selectable), falls back to 5 V on a non-PD host, so a PC port is never overvolted.
- **4× SY6970 power-path chargers** (Silergy, BQ25896-class), one per cell. ~3.9–14 V input covers both a 5 V PC port and a 9–12 V brick. Switching buck power-path (system/load served first), I²C-programmable input current limit + charge current, TS temperature pin, /STAT + /INT indicators, integrated 16-bit ADC. No external charge-current sense resistor.
- **Switching, not linear** — each charger bucks 12 V → 4.2 V efficiently; no front-end converter needed. Each needs its own inductor (SRP5030CA-2R2M) + input/output caps.
- **Charge current is set over I²C** (CICHG register), not a resistor strap — so it can be tuned in firmware per source. Target ~1.0–1.5 A/cell: 1.0 A fits a 9 V/2 A phone charger (~3 h); 1.5 A (≈0.5 C) wants a 12 V/3 A brick (~2 h). Input-current limit (ILIM pin resistor + I²C DPM) throttles on a weak source.
- **Monitoring comes free:** the same SY6970 reports VBUS/VBAT/VSYS/ICHG/IBUS/TS over I²C — see §6. This is why no separate fuel gauge is fitted.

**ESP programming:** D+/D- straight to the ESP32-S3 module (native USB-Serial-JTAG, no UART bridge). USBLC6-class ESD on D+/D-, BOOT/EN pads. The flight computers are **not** programmed here (their own SWD); the USB bypass only *powers* them for bench debug.

**Power budget:** four cells at 1.5 A ≈ 25 W at the cells ≈ 28 W input, plus ~2 W bypass ≈ 30 W → request 12 V (36 W) for margin.

---

## 6. Sensing & monitoring

- **Monitoring is built into the SY6970 chargers** — each integrates a 16-bit ADC reporting VBAT, VSYS, VBUS, ICHG, IBUS and TS over I²C. No separate fuel gauge or sense resistor is added to the firing path. Tradeoff vs. a dedicated ModelGauge (MAX17048): this is **ADC/voltage-based** state, not true coulomb-counting SoC — accepted for the lower part count; firmware estimates SoC from voltage + charge state + the TS-derived temperature.
- The SY6970 has a **fixed I²C address (0x6B)**, so the four sit behind a **TCA9548A 8-channel I²C mux** (one channel each). Total ESP pins for all monitoring: 2 (SDA/SCL).
- I²C bus topology: ESP → TCA9548A (4× SY6970 chargers, Board 1) and, **upstream of the mux**, across the 4-pin connector → IS31FL3731 gauge driver (Board 2, addr 0x74, no conflict). Pull-ups (4.7 k) on Board 1. The mux RST line (`MUX_RST`) is driven by the ESP.
- Per-cell temperature comes from each SY6970's TS NTC, read back over the same I²C ADC.

---

## 7. ESP, gauge & telemetry

**ESP32-S3-WROOM-1 module** on cell 1 (FC1 channel), behind its own load switch (TPS22917-class) + fuse so a fault can't drag down FC1.
- Reads four SY6970 chargers (V/I/charge/temperature) over the muxed I²C bus.
- Drives the gauge over I2C.
- Streams charge + channel-live state over BLE — **read-only, forever**; never accepts a command.
- Supply regulated to 3.3 V from SYS/charge rail — **never raw VBUS**.

**ESP power gating:**
- USB present → ESP on (charge rail), gauge shows charging.
- USB absent + gauge button → momentary soft-latch wake: read, paint gauge N seconds, power down.
- USB absent + launch-day jumper removed → sustained-on, BLE streaming.
- A hardware **load switch** is the real gate (firmware can't strand it on); add a watchdog + load-switch timeout.

**Gauge:** 64 LEDs via **IS31FL3731 (eTQFP-28)**, per-LED PWM, hardware breathe/animation engine. Per channel: a charge-level bar plus **2 diagnostic LEDs** (over-temp warning, fault/protection-tripped). Charlieplex matrix — wire the four channels as contiguous blocks. No per-LED resistors. Powered from the charge rail (dead in flight). AD strap = 0x74, SDB to enable/GPIO.

---

## 8. Power-enable (pull pins) & safety

- **Remove-before-flight power enable.** Pin inserted = channel isolated; removed = power flows. *Default-unpinned is live* — use captive/retained pins.
- **Four independent pull pins**, one per channel (4× 2-pin screw terminal), enabling sequenced power-up: bring up FC channels first for checks, pyro channels last.
- **Key/colour the pyro pins** distinctly from the FC pins.
- **Firing-FET-held-off** is a downstream pyro-board requirement, out of BMS scope — but the BMS must deliver clean, transient-free power so applying power can't glitch a load.
- **Hot-plug inrush:** size input caps + soft-start.

---

## 9. Indicator LEDs

Two tiers, so the board is never silent regardless of ESP firmware state.

**9a. Hardwired (ESP-independent) — the dumb fallback**
- Per-channel charge status (4×) from each SY6970 /STAT pin (on = charging, off = done, blink = fault).
- USB / input-present.
- Fault (e.g., protection or charger fault).

These give basic board state even with the ESP off or firmware dead.

**9b. Per-cell RGB status (ESP-driven) — the rich indicator**
- **One addressable RGB LED per cell (SK6812 / WS2812B), 4× total, daisy-chained on a single ESP
  GPIO** (DIN→DOUT through all four). The ESP already reads each SY6970's V/I/charge/fault/temp over
  I²C, so it has everything needed to paint exact per-cell state — visible on Board 1 alone, **no
  Board 2 required**. Because USB-present ⇒ ESP-on, these are live whenever anything is charging.
- Colour scheme (mirrors the Board-2 gauge diagnostics): **green breathing = charging · solid blue =
  charged · red = fault/protection · amber = over-temp · off = idle/no cell.**
- **Power LED VDD from the SYS/charge rail, not the 3V3 LDO** (4× WS2812 ≈ 240 mA at full white;
  the AP2112K can't source that). Only the data line touches the ESP — level-shift it (use SK6812
  for 3.3 V tolerance, or a 74AHCT125 / SN74LVC2T45). Dead in flight, like all monitoring.

---

## 10. Operational states

| State | USB | FC pins | Pyro pins | ESP / BLE | Gauge | Loads |
|---|---|---|---|---|---|---|
| Storage (days out) | out | in (isolated) | in (isolated) | off — jumper in | off | off |
| Charging | in | in | in | on, charge rail | levels | off |
| Bench / FC debug | in | out | in | on / momentary | readout | FCs live, pyro isolated |
| Pad power-up | out | out | out (last) | on, BLE — jumper out | off | live |
| Flight / recovery | out | out | out | sleeps on link loss | off | live |

Invariants: chargers inherently dead without USB; per-cell protection always live (µA); gauge never auto-powers off the cells; ESP rail hard-cut by a load switch.

---

## 11. Candidate BOM (verify current LCSC stock + package)

| Function | Candidate | Notes |
|---|---|---|
| Charger + monitor ×4 | **SY6970QCC** | BQ25896-class, ~3.9–14 V in, switching power-path, I²C 0x6B, integrated ADC (V/I/charge/TS), /STAT + /INT. Replaces the MP2617 **and** the fuel gauge. In `libs/BMS.kicad_sym`. |
| Charger inductor ×4 | SRP5030CA-2R2M | 2.2 µH, for the SY6970 buck. In `libs/BMS.kicad_sym`. |
| Protection ×4 | DW01A + SI4920DY dual N-FET | `Battery_Management:DW01A` + `libs/BMS:SI4920DY`. Use a lower-Rds FET on the pyro channels. |
| Reverse-polarity ×4 | AO3401A P-FET (existing) — consider DMG2305UX / SI2333 for lower Rds | `Transistor_FET:AO3401A` |
| Fuse/PTC ×4 | resettable PTC | rated above firing pulse, below sustained short |
| ~~Fuel gauge~~ | **dropped** | monitoring folded into the SY6970 (see §6) |
| I2C mux | TCA9548A | `Interface_Expansion:TCA9548APWR`; hosts the four 0x6B SY6970s |
| PD sink | CH224K (or STUSB4500) | resistor-strap voltage select. In `libs/BMS.kicad_sym`. |
| Gauge driver (Board 2) | IS31FL3731 | eTQFP-28, I²C 0x74, 64 LEDs |
| ESP | ESP32-S3-WROOM-1 | `RF_Module:ESP32-S3-WROOM-1` |
| ESP load switch | TPS22917-class | `Power_Switch` — hard-gates the ESP rail |
| Per-cell RGB status ×4 | SK6812 / WS2812B | addressable, daisy-chained on 1 ESP GPIO; VDD from SYS rail (§9b) |
| LED data level shift | SK6812 (3.3 V tolerant) or 74AHCT125 / SN74LVC2T45 | only if WS2812B used |
| 3V3 LDO | AP2112K-3.3 | `Regulator_Linear:AP2112K-3.3` |
| ESD | USBLC6-2SC6 | `Power_Protection:USBLC6-2SC6`, on D+/D- |

---

## 12. Configuration straps (0Ω / populate)

ESP source select (charge-rail vs cell-1) + always-on capability strap · PD voltage request (9/12/15 V) · ~~per-cell charge-current set~~ *(now I²C-programmed on the SY6970 — only the ILIM input-current resistor remains)* · D+/D- series 0Ω · PD-sink bypass (force 5 V) · per-channel ground-to-star 0Ω · per-channel enable source (pin vs always-on for bench).

---

## 13. Open items

1. **Mechanical stack confirmation** — Boards 1 & 3 parallel, Board 2 spine, mezzanine connectors? Confirm mezzanine part/pitch.
2. **Channel-live sense granularity** — combined (1 line, keeps the 4-pin connector) vs per-channel (4 lines, larger connector or local read on Board 2). Needed if the gauge/BLE should show each channel.
3. **Star-point location** — fix it in copper.
4. **Pyro OC trip numbers** — verify firing inrush vs the chosen FS8205 Rds and the PTC rating against the specific e-match.
5. **Project name.**

---

## 14. Revision history

| Version | Notes |
|---|---|
| 0.1 | Initial architecture: 3-board set, 4 independent 1S channels, RBF pull pins, power-path PD charging, dual-purpose USB-C, per-cell protection, non-flight-critical monitoring layer. |
| 0.2 | Parts selected (MP2617 charger, MAX17048 + TCA9548A mux, CH224K PD sink, DW01A+FS8205 protection, IS31FL3731 gauge, ESP32-S3-WROOM-1, P30B 18650 cell). Corrected terminology: no on-board arming — pull pins are power-enable/isolation. Added fuel-gauge sensing, hardwired indicator LEDs, mechanical stack note (Boards 1+3 stacked, Board 2 spine), candidate BOM. |
| 0.3 | **Charger/monitoring topology revised to match existing capture:** SY6970 smart charger (charge + integrated I²C monitoring) replaces MP2617 + MAX17048; separate fuel gauge dropped; TCA9548A mux retained (4× shared 0x6B). ESP confirmed ESP32-S3-WROOM-1. BOM rows mapped to concrete KiCad/`BMS` symbol libraries. Added ESP load-switch (TPS22917) + 3V3 LDO (AP2112K) to BOM. Charge current moved from resistor strap to I²C register. Board 1 capture plan written (`board1_power_capture_plan.md`). Added **per-cell addressable RGB status LEDs** (§9b) on Board 1 — ESP-driven, standalone of Board 2. |
