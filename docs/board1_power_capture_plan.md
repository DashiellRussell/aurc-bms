# Board 1 — Power & Charge — Schematic Capture Plan

Capture plan for `board1_power/board1_power.kicad_sch`. Draw it in KiCad from this; it's the
engineering, not the drawing. Authority order: `interconnect.md` (board edges) → `architecture.md`
(design intent) → this file (Board 1 detail).

**Locked decisions (2026-06-08):** SY6970 smart charger (charge + integrated I²C monitoring, no
separate fuel gauge) · ESP32-S3-WROOM-1 · TCA9548A I²C mux · 4 independent 1S channels (FC1, FC2,
Pyro1, Pyro2) · single star ground on Board 1.

> **Build status (2026-06-08):** the drawn capture was **migrated** from the archived single-board
> design into `board1_power/` — `USB_IN.kicad_sch` and the 4× `charge_chanel.kicad_sch` instances
> are live (lib refs fixed to `BMS:`), validated by `kicad-cli sch erc` (loads clean, 169 WIP
> violations — unconnected pins, no PWR_FLAGs, "Unspecified" pin-type warnings from the imported
> symbols). **Remaining build (do in KiCad GUI, in order):** ① swap ESP32-C3 → ESP32-S3-WROOM-1 ·
> ② add TCA9548A mux + wire each channel's I²C to its mux port (§5 table) · ③ add the 4× RGB chain +
> TPS22917 load switch (§5, §9b) · ④ add J1 / J1s connectors (§6, `interconnect.md`) · ⑤ star
> ground (§7) · ⑥ finish root wiring, add PWR_FLAGs, place the 2nd unit of the dual FET, clear ERC.

> **Pin numbers:** connections below are given **by pin function/name**, because the safe move is
> to confirm physical pin numbers against each part's symbol + datasheet as you place it. For the
> charge channel and USB input you already have a *drawn reference* in
> `_archive/single-board-v0.1/` (`charge_chanel.kicad_sch`, `USB_IN.kicad_sch`) — open it alongside.

---

## 0. What lives on Board 1

USB-C input + PD sink · 4× SY6970 power-path chargers (+ per-cell protection + 18650) · ESP32-S3 +
TCA9548A mux + 3V3 LDO behind a load switch · hardwired indicator LEDs · the two board-edge
connectors **J1** (8-pin power out) and **J1s** (4-pin signal). Star ground is here.

Nothing flight-critical depends on the ESP. The four SYS rails leave via J1 regardless of ESP state.

---

## 1. Sheet hierarchy

```
board1_power.kicad_sch  (ROOT)
├── usb_in.kicad_sch              ×1   USB-C + CH224K PD sink + ESD + input caps
├── charge_channel.kicad_sch      ×4   one per cell — SY6970 + protection + 18650   ← reusable design
└── esp_monitor.kicad_sch         ×1   ESP32-S3 + TCA9548A mux + AP2112K 3V3 + TPS22917 load switch
ROOT also holds: J1, J1s, star-ground node, fault LED, the global rails.
```

Use **one** `charge_channel` sheet file, instantiated 4×. Per-instance differences (which load,
which mux channel, pin colour) are wired at the root via sheet pins — the sheet itself is generic.

**Sheet pins (the contract between each sub-sheet and root):**

| Sheet | In | Out / bidir |
|---|---|---|
| `usb_in` | — | `VBUS` (PD rail, ~12 V), `USB_DP`, `USB_DM`, `GND` |
| `charge_channel` | `VBUS`, `SDA_n`, `SCL_n`, `RGB_DIN`, `LED_VDD`, `GND` | `CHn_V` (= SYS rail), `STATn`, `RGB_DOUT` |
| `esp_monitor` | `VBUS_PRESENT`/`SYS_FC1` (ESP rail source), `SDA`,`SCL` (main bus), `USB_DP`,`USB_DM`, `GND` | `SDA0..3`,`SCL0..3` (mux downstream → the 4 channels), `MUX_RST`, `RGB_DATA` (to first cell's `RGB_DIN`) |

**Per-cell RGB chain:** the ESP's `RGB_DATA` → instance 1's `RGB_DIN`; each instance's `RGB_DOUT` →
the next instance's `RGB_DIN` (daisy chain, one data line for all four). `LED_VDD` is the **SYS/charge
rail**, *not* `+3V3`. See §9b of `architecture.md` for colour semantics.

> `SDA_n`/`SCL_n` on each `charge_channel` instance connect to a **different** mux downstream pair
> (`SDA0/SCL0` → channel 1, … `SDA3/SCL3` → channel 4). That's how four identical 0x6B chargers
> coexist. Keep the channel→mux-port mapping fixed and documented (table in §6).

---

## 2. Global nets / naming (match `interconnect.md`)

| Net | Meaning |
|---|---|
| `VBUS` | PD-negotiated input rail (~12 V) feeding all four chargers + the ESP rail source |
| `CH1_V`..`CH4_V` | each channel's SYS rail (charger output) → J1. **= FC1, FC2, Pyro1, Pyro2** |
| `CH1_GND`..`CH4_GND` | each channel's independent return → star point (§7) → J1 |
| `SDA`,`SCL` | ESP main I²C bus (4.7 k pull-ups on Board 1) |
| `SDA0..3`,`SCL0..3` | mux downstream pairs → the four SY6970s |
| `MUX_RST` | ESP → TCA9548A reset (active low) |
| `USB_DP`,`USB_DM` | USB D+/D- → ESP native USB |
| `+3V3` | ESP/logic rail (LDO output, downstream of load switch) |
| `CH_LIVE` | combined channel-live sense → J1s (see open item §13.2) |
| `STAT1..4` | per-channel /STAT (to hardwired fallback LEDs) |
| `RGB_DATA` → `RGB_DIN/DOUT` chain | per-cell addressable RGB status, daisy-chained (§9b) |
| `LED_VDD` | RGB LED supply = **SYS/charge rail** (not +3V3) |

---

## 3. Sheet `usb_in` — USB-C + PD sink  (reuse archived `USB_IN.kicad_sch` ~as-is)

This already matches v0.3. Components: `Connector:USB_C_Receptacle_USB2.0_16P`, `BMS:CH224K`,
`Power_Protection:USBLC6-2SC6`, input bulk cap (`Device:C_Polarized`), decoupling, status LED.

Connections (by function — confirm pins):

- **USB-C** VBUS pins (A4/A9/B4/B9) → `VBUS_RAW`; GND pins (A1/A12/B1/B12) + shield → `GND`.
- **CC1/CC2** → CH224K CC1/CC2. **D+/D-** → through USBLC6 → `USB_DP`/`USB_DM` (series 0Ω strap each).
- **CH224K**: VBUS in ← `VBUS_RAW`; VOUT/sense → `VBUS`; **CFG/PG resistor strap sets the requested
  voltage** (9/12/15 V — populate per §8); decouple VDD. PG/status → input-present LED (green).
- **USBLC6** across D+/D- and VBUS for ESD; one device covers both data lines + VBUS clamp.
- Input bulk cap on `VBUS` (hot-plug inrush, §8) + soft-start consideration at the board level.

Sheet-pin out: `VBUS`, `USB_DP`, `USB_DM`, `GND`.

> Note: CH224K negotiates *input* voltage. Per-channel charge regulation is the SY6970's job.

---

## 4. Sheet `charge_channel` (×4) — SY6970 + protection + cell

**You already drew this** (archived `charge_chanel.kicad_sch`: SY6970QCC + DW01A + SI4920DY +
AO3401A + SRP5030 + Polyfuse + NTC + STAT LED). Re-draw it into the new project (capture-plan-only),
or open the archive as your template. Target net list per instance:

**Components:** `BMS:SY6970QCC` (U), `Battery_Management:DW01A` (U), `BMS:SI4920DY` dual N-FET (Q),
`Transistor_FET:AO3401A` P-FET (Q, reverse-polarity), `BMS:SRP5030CA-2R2M` 2.2µH (L),
`Device:Polyfuse` PTC (F), `Device:Thermistor_NTC` (RT), `Device:LED` (D, /STAT hardwired fallback),
**SK6812/WS2812B addressable RGB (D, per-cell status — §9b)**, 18650 holder
(`Connector`/`Device:Battery`), input/output caps, ILIM + TS divider + I²C-section resistors.

**Power path (cell → SYS):**
1. **18650** `CELL_P` / `CELL_N`.
2. **Protection (DW01A + SI4920DY)** in the cell-negative line: the two series N-FETs sit between
   `CELL_N` and the channel's protected negative (`B_MINUS`). DW01A: VDD←`CELL_P` (via 100Ω+0.1µF
   RC), GND←`B_MINUS`, CS/Vm← the FET-junction node, OC/OD gates → SI4920DY gates. *(Canonical DW01A
   1S circuit — mirror the archived sheet.)* Over-charge ~4.3 V, over-discharge ~2.4 V, OC trip set
   by FET Rds. **Pyro channels: use the lower-Rds FET option** so the firing pulse doesn't trip OC.
3. **Reverse-polarity P-FET (AO3401A)** in the protected positive line: source→`CELL_P`, drain→
   `VBAT_PROT`, gate→`B_MINUS` via 100k. (Consider DMG2305UX/SI2333 for lower Rds — §11.)
4. **PTC fuse** in series (positive line) — sized above the 2 A firing pulse, below sustained short.
5. **SY6970**: `BAT` ↔ `VBAT_PROT`; `SW` → inductor `L` → ; `SYS` → **`CHn_V`** (sheet-pin out, the
   J1 rail); `VBUS` ← global `VBUS`; `PMID` cap to GND; `REGN` cap; `ILIM` resistor to GND (input
   current limit); `TS` ← NTC divider (charge temp cutoff 0–45 °C); `/STAT` → STAT LED **and**
   sheet-pin `STATn`; `/CE` tie enabled (or ESP GPIO later); `SDA`/`SCL` ← sheet-pins `SDA_n`/`SCL_n`
   (this channel's mux pair); `OTG`/boost unused — tie per datasheet; D+/D- on SY6970 unused (PD is
   handled upstream) — terminate per datasheet.
6. **Decoupling:** VBUS in-cap, SYS out-cap, BAT cap, REGN/PMID caps per SY6970 datasheet.

**Ground:** this channel's return is `CHn_GND` — kept independent, joined only at the star (§7).

**RGB status LED (§9b):** one SK6812/WS2812B per cell. `VDD` ← `LED_VDD` (SYS rail) + 0.1µF;
`DIN` ← sheet-pin `RGB_DIN`; `DOUT` → sheet-pin `RGB_DOUT`; `GND` ← `CHn_GND`. The dumb `/STAT` LED
stays as the firmware-independent fallback.

Sheet-pins: in `VBUS`,`SDA_n`,`SCL_n`,`RGB_DIN`,`LED_VDD`,`GND` · out `CHn_V`,`STATn`,`RGB_DOUT`.

---

## 5. Sheet `esp_monitor` — ESP32-S3 + mux + 3V3 + load switch

**This is the main *new* capture work** (archived top sheet was ESP32-C3 + AP2112K; upgrade to S3
and add the mux + load switch).

**Components:** `RF_Module:ESP32-S3-WROOM-1` (U), `Interface_Expansion:TCA9548APWR` (U),
`Regulator_Linear:AP2112K-3.3` (U), `Power_Switch:TPS22917...` load switch (U), I²C pull-ups,
decoupling, BOOT/EN parts, optional gauge button input.

**ESP rail gating (the hard gate, §7 of architecture):**
- ESP-rail source = charge/SYS rail (strap-select SYS_FC1 vs charge rail — §8), **never raw VBUS**.
- That source → **TPS22917 load switch** (ON pin driven by: USB-present OR gauge-button latch OR
  launch-day jumper — see arch §7 states) → **AP2112K-3.3** → `+3V3` → ESP `3V3` + decoupling.
- TPS22917 is the real gate firmware can't defeat; add a watchdog/timeout downstream.

**ESP32-S3:**
- `3V3` ← `+3V3`; multiple `GND`; `EN` pull-up + RC; `IO0`(BOOT) pull-up + button/pad.
- **Native USB:** `USB_D+`/`USB_D-` ← `USB_DP`/`USB_DM` (programming over USB-Serial-JTAG; no UART
  bridge). USBLC6 ESD is on the usb_in sheet.
- **I²C main bus:** two GPIO → `SDA`/`SCL` (4.7 k pull-ups to +3V3 *here*).
- `MUX_RST` → one GPIO. Optional `/INT` aggregation from chargers → one GPIO.
- **`RGB_DATA` → one GPIO → first cell's `RGB_DIN`** (per-cell RGB chain, §9b). If WS2812B, put a
  level shifter (74AHCT125 / SN74LVC2T45) on this line; SK6812 usually tolerates 3.3 V direct. Add a
  ~330 Ω series resistor at the source and a bulk cap on `LED_VDD`.
- Optional: gauge-button input GPIO; STAT sense GPIOs (telemetry) — non-critical.

**TCA9548A mux:**
- `VCC`←`+3V3`, decouple; `A0/A1/A2` → address straps (pick 0x70); `/RESET`←`MUX_RST`.
- **Upstream** `SDA`/`SCL` ← ESP main bus `SDA`/`SCL`.
- **Downstream** SD0/SC0..SD3/SC3 → `SDA0/SCL0`..`SDA3/SCL3` (sheet-pins to the 4 channels).
- The Board-2 IS31FL3731 (0x74) sits on the **main** bus (upstream of mux) and exits via J1s — it is
  *not* on a mux channel (unique address, no conflict).

**Mux-channel ↔ cell map (fix this and don't change it):**

| Mux ch | Pair | Channel | Load |
|---|---|---|---|
| 0 | SDA0/SCL0 | CH1 | FC1 |
| 1 | SDA1/SCL1 | CH2 | FC2 |
| 2 | SDA2/SCL2 | CH3 | Pyro1 |
| 3 | SDA3/SCL3 | CH4 | Pyro2 |
| 4–7 | — | spare | — |

Sheet-pins: in `ESP_RAIL_SRC`,`SDA`,`SCL`,`USB_DP`,`USB_DM`,`GND` · out `SDA0..3`,`SCL0..3`,`MUX_RST`,`RGB_DATA`.

---

## 6. ROOT sheet — instances, connectors, star ground

- Place the 6 sheet instances (1× usb_in, 4× charge_channel, 1× esp_monitor).
- Wire each `charge_channel` instance's `SDA_n`/`SCL_n` to its mux pair per the table above.
- Bus `VBUS` from usb_in → all four charge_channels + esp_monitor rail source.
- **J1 — power, 8-pin** (`Connector_Generic:Conn_02x04` or chosen mezzanine):
  `CH1_V`/`CH1_GND` … `CH4_V`/`CH4_GND`, pin order = interconnect.md §J1, 3 A/pin.
- **J1s — signal, 4-pin:** `SDA`, `SCL`, `CH_LIVE`, `SGND`.
- **`CH_LIVE`** (combined): derive from a diode-OR of the four SYS rails through a divider to a
  logic-level sense (or revisit §13.2 if per-channel is needed). Mark as provisional.
- **Fault LED** (board-level) + input-present LED (on usb_in).

---

## 7. Star ground (architecture §3)

The four `CHn_GND` returns are **independent** up to a single star node on Board 1 (by the cells).
Join them there with per-channel `0Ω` links (§8 strap) — and reference USB `GND` and signal `SGND`
to that same node. **Nowhere else.** Do not pour one continuous ground plane across all four
channels; keep four return regions meeting at the star. This is the OZONE PYRO_GND philosophy.

---

## 8. Config straps (architecture §12 — place as 0Ω / DNP pads)

- **PD voltage request** (CH224K CFG): 9 / 12 / 15 V — default 12 V.
- **PD-sink bypass** (force 5 V) for a dumb host.
- **D+/D- series 0Ω** (×2) — isolate ESP USB if needed.
- **ESP rail source select** (charge-rail vs cell-1) + **always-on capability** strap.
- **SY6970 ILIM resistor** per channel (input-current limit; charge current itself is I²C-set).
- **Per-channel ground-to-star 0Ω** (×4) — the star links.
- **Per-channel enable source** (`/CE` from pull vs always-on for bench).

---

## 9. Library / symbol map

| Need | Symbol | Source | Status |
|---|---|---|---|
| Charger+monitor | `SY6970QCC` | `libs/BMS.kicad_sym` | ✓ have |
| Inductor 2.2µH | `SRP5030CA-2R2M` | `libs/BMS.kicad_sym` | ✓ have |
| PD sink | `CH224K` | `libs/BMS.kicad_sym` | ✓ have |
| Protection FET | `SI4920DY` | `libs/BMS.kicad_sym` | ✓ have |
| Protection IC | `DW01A` | `Battery_Management` | ✓ stock |
| Rev-polarity P-FET | `AO3401A` (or DMG2305UX) | `Transistor_FET` | ✓ stock (AO3401A) |
| ESP | `ESP32-S3-WROOM-1` | `RF_Module` | ✓ stock |
| I²C mux | `TCA9548APWR` | `Interface_Expansion` | ✓ stock |
| 3V3 LDO | `AP2112K-3.3` | `Regulator_Linear` | ✓ stock |
| Load switch | `TPS22917...` | `Power_Switch` | ✓ stock (pick variant) |
| ESD | `USBLC6-2SC6` | `Power_Protection` | ✓ stock |
| Per-cell RGB ×4 | `SK6812` / `SK6812MINI-E` / `WS2812B` | `LED` | ✓ stock (confirm footprint variant) |
| LED level shifter (opt) | `74AHCT1G125` | `74xGxx` | ✓ stock; only if WS2812B |
| USB-C | `USB_C_Receptacle_USB2.0_16P` | `Connector` | ✓ stock |
| PTC / NTC / R / C / LED | `Device:*` | `Device` | ✓ stock |
| 18650 holder | `Device:Battery` / `Connector` | `Device` | ✓ stock |

**Footprints to confirm before layout:** SY6970 package (QFN — already in `BMS.pretty`?), TCA9548APWR
(TSSOP-24), ESP32-S3-WROOM-1 (RF_Module.pretty), TPS22917 (SOT-23-6), DW01A (SOT-23-6), SI4920DY
(SOIC-8 — in `BMS.pretty`), AO3401A (SOT-23). No new *symbols* need creating — good.

---

## 10. ERC / verification checklist

- [ ] All four `CHn_V` reach J1; all four `CHn_GND` reach the star then J1.
- [ ] Exactly **one** star node; no second ground tie between channels.
- [ ] I²C: 4.7 k pull-ups present once (on +3V3 main bus); each SY6970 on its own mux channel;
      gauge driver (0x74) on main bus, not a mux channel; `MUX_RST` driven.
- [ ] ESP rail comes through the load switch + LDO, **never raw VBUS**; load switch ON logic correct.
- [ ] Each SY6970 has TS NTC, ILIM resistor, REGN/PMID/BAT/SYS caps, inductor on SW.
- [ ] Pyro-channel protection FET is the low-Rds option (won't trip on firing pulse).
- [ ] RGB chain: `RGB_DATA`→cell1 `DIN`, each `DOUT`→next `DIN`; `LED_VDD` = SYS rail (not +3V3);
      series R + level shift present if WS2812B; hardwired `/STAT` LED still present per cell.
- [ ] No power-flag/unconnected ERC errors; every hierarchical sheet pin matches its label.
- [ ] J1 / J1s pinout byte-for-byte matches `interconnect.md`.
- [ ] PD CFG strap set to a sane default (12 V); bypass-to-5V strap present.

---

## 11. Suggested capture order

1. `usb_in` (smallest, reuse archived) → get `VBUS` clean.
2. `charge_channel` ×1 (reuse archived; review against §4) → ERC one channel in isolation.
3. ROOT: instantiate that one channel + usb_in, verify `VBUS` + one `CHn_V`.
4. `esp_monitor` (the new work: S3 + mux + load switch).
5. ROOT: wire mux↔channel I²C, add the other 3 channel instances, J1, J1s, star ground, LEDs.
6. Full ERC pass against §10. Then annotate + assign footprints → ready for layout.

---

## 12. Open items that touch Board 1

- **§13.2** `CH_LIVE` combined vs per-channel → J1s pin count + the root sense circuit.
- **§13.3** Star-point exact location in copper (this plan assumes Board 1 by the cells).
- **§13.4** Pyro OC trip: verify firing inrush vs chosen SI4920DY Rds + PTC rating vs the e-match.
- Confirm SY6970 footprint + LCSC stock; confirm TPS22917 variant (latching vs non-latching ON).
