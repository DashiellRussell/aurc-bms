# AURC Nosecone BMS (board4) — Architecture & Design

**Version:** 0.1
**Scope:** Two-channel, single-board battery board feeding two FC/logic loads in the nosecone.
**Parent design:** `architecture.md` v0.5 (the 4-channel board1–3 set). This board inherits every
decision from that document unless a section below explicitly overrides it. Read that for *why*;
read this for *what changed*.
**Status:** Pre-schematic. Open items in §9.

---

## 1. What this board is

The nosecone carries its own avionics, powered by **two independent 1S 18650 channels**, both
feeding **flight-computer / logic loads** (no pyro). Unlike the main bay's 3-board stack, nosecone
volume dictates a **single PCB**: charger, cells, protection, pull-pin power-enable, monitoring,
and load distribution all on `board4_nosecone`. Rectangular sled mount (dimensions TBD, §9.1).

**Deltas from the 4-channel set, at a glance:**

| | board1–3 set | board4 nosecone |
|---|---|---|
| Channels | 4 (FC1, FC2, Pyro1, Pyro2) | **2 (NFC1, NFC2 — both FC/logic)** |
| Boards | 3-board mezzanine stack | **single board** |
| Pyro path | low-Rds FETs, firing-pulse PTC sizing | **none — no firing pulses anywhere** |
| I²C to chargers | TCA9548A mux (4× SY6970 @ 0x6B) | **no mux — one SY6970 per ESP32-S3 I²C controller** |
| Gauge | Board-2 64-LED IS31FL3731 matrix | **dropped — SK6812 RGB + /STAT LEDs only** |
| WAKE button | on Board 2, via J1s | **on-board tactile switch** |
| PD request | 12 V (30 W budget) | **9 V default (≈15 W budget), 12 V strap option** |

---

## 2. Channel model

- **Two electrically independent 1S channels**: `NCH1_V` → NFC1, `NCH2_V` → NFC2. Independent
  positive rails; returns star-bonded at **one point on this board, next to the cells** (per-channel
  ground-to-star `0Ω` links, R 0603 — same convention as §12 of the parent doc).
- No series stack → no balancing. No pyro → **no low-Rds exception, no firing-pulse arithmetic** —
  the DW01A + SI4920DY stack and PTC are sized purely for FC continuous draw + inrush (§9.2).
- Net names: `NCH{1,2}_V`, `NCH{1,2}_V_SW` (downstream of pull pin), `NCH{1,2}_GND`. The `N` prefix
  keeps nosecone nets grep-distinct from the main set's `CH{1..4}_*`.

## 3. Per-cell protection stack (per channel, cell outward)

Identical philosophy and parts to parent §4, minus the pyro clauses:

1. **Reverse-polarity P-FET at the cell terminal** — DMG2305UX (or AO3401A as drawn in `BMS` lib),
   SOT-23. Keyed holder remains the primary defense; compensate Rds drop via the I²C VREG trim.
2. **DW01A + SI4920DY** in the cell-negative path (`Battery_Management:DW01A`, `BMS:SI4920DY`) —
   over-charge/over-discharge/OC. No lower-Rds substitution needed.
3. **SY6970** power-path charger — same silicon rules as parent §5 (VSYSMIN 3.5 V, BATFET default-on,
   **no ship mode in flight**, `/QON` strapped high). Cap set per parent: bootstrap **47 nF (C 0603)**,
   PMID **6.8 µF (C 0805)**, BAT ≥10 µF (C 0805), SYS ≥10 µF (C 0805), REGN 4.7 µF (C 0603),
   inductor SRP5030CA-2R2M 2.2 µH.
4. **`SYS` / `LOAD_P` source-select 0Ω strap (R 0603)** then **resettable PTC** on each `NCHn_V`
   output. PTC hold/trip sized to the actual nosecone FC draw — **open item §9.2** (no 2 A pulse to
   pass this time, so it can trip much lower than the main board's).

NTC on each SY6970 TS pin (10 k NTC 0603 + bias per datasheet) for the 0–45 °C charge window.

## 4. Charging & USB-C

Parent §5 applies wholesale (single dual-purpose USB-C: PD sink + ESP native USB, USBLC6-2SC6 ESD,
CH224K strap-select). Changes:

- **Budget:** 2 cells × 1.5 A ≈ 12.6 W at the cells ≈ 15 W input → **request 9 V** (a 9 V/2 A phone
  brick suffices); keep the 12 V strap position for sharing the main board's brick.
- Charge current still I²C-programmed (~1.0–1.5 A/cell); ILIM resistor per datasheet.

## 5. I²C topology — mux deleted

Both SY6970s are fixed at 0x6B, but the **ESP32-S3 has two hardware I²C controllers** — so instead
of a TCA9548A:

- **I2C0:** SDA0/SCL0 → SY6970 #1 (NCH1)
- **I2C1:** SDA1/SCL1 → SY6970 #2 (NCH2)
- 4.7 k pull-ups (R 0603) on **each** bus, to the ESP 3V3 rail.

Nothing else shares the buses (no gauge driver on this board). Two parts and a RST line deleted
relative to board1.

## 6. ESP, monitoring & indicators

**ESP32-S3-WROOM-1** on NCH1, same power-gating as parent §7 (as-built board1 scheme):
`NCH1_V → AP2112K-3.3`, EN driven by the MMBD4448HCQW diode-OR of {USB divider, always-on 100 k
strap (R 0603), WAKE, self-hold GPIO}, 100 k pull-down (R 0603), 2N7002 RBF off-jumper shunt.
BLE stream stays **read-only, forever**.

- **WAKE is now an on-board tactile switch** (`NCH1_V` → WAKE through the button), since there is
  no Board 2. Same debounce/limiting as the board1 J1s arrangement.
- **Hardwired LEDs (always fitted, ESP-independent):** per-channel SY6970 /STAT LED (2×, LED 0603 +
  R 0603), USB-present LED, fault LED — the dumb fallback, as parent §9a.
- **Rich status LEDs — DECIDED: (a) 2× SK6812** (ease of use; everything stays ESP-controlled over
  one GPIO). Options considered for the record:
  - **(a) 2× SK6812 RGB**, daisy-chained on one GPIO (parent §9b scheme). Fewest parts, colour
    encodes state, no charge-*level* bar. VDD from `NCH1_V` (same storage-drain caveat — pull cells
    for storage or gate the rail).
  - **(b) IS31FL3731 charlieplex driver** (I²C 0x74, shares a bus with one charger, no address
    clash) + a small LED bar — e.g. 8 LEDs/cell charge bar + 2 diagnostics, ~20 LEDs of the 144 the
    matrix supports, no per-LED resistors, hardware breathe engine. This is Board 2's gauge scaled
    down and pulled on-board; symbol already in the libs.
  - **(c) Raw ESP-GPIO charlieplexing** — no driver IC, but N(N−1) LEDs costs N GPIOs *and*
    firmware scan timing, and the LEDs die the instant the ESP sleeps. Worst of both; avoid.
  - **Recommendation: (a), plus (b) only if a visible charge-level bar on the sled is genuinely
    wanted** — in the nosecone the board is buried at integration, and BLE already gives exact
    numbers on a phone, which weakens the case for a bar this time.
- SoC-by-voltage over BLE via the two chargers' internal ADCs, exactly as the main board.

## 7. Power-enable & outputs

- **2× pull-pin power-enable on a single 4-position screw terminal block** (1×4, one pin pair per
  channel: positions 1–2 = NCH1 loop, 3–4 = NCH2 loop), both **FC-keyed** — no pyro keying needed
  on this board. Pin inserted = isolated; removed = live (captive pins, parent §8). One block
  instead of the main set's four separate 2-pin terminals; silkscreen the two loops clearly since
  the block itself gives no keying.
- Because the pull-pin switch and distribution are on-board, `NCHn_V → strap/PTC → pull-pin →
  NCHn_V_SW → output connector` is one board's routing — no interconnect contract needed beyond:

**J1 — nosecone loads (4-pin, 3 A/pin):**

| Pin | Net |
|---|---|
| 1 | NFC1_V (= NCH1_V_SW) |
| 2 | NFC1_GND (= NCH1_GND) |
| 3 | NFC2_V (= NCH2_V_SW) |
| 4 | NFC2_GND (= NCH2_GND) |

Per-channel V and GND kept separate end-to-end; grounds meet only at the on-board star point.

## 8. BOM deltas vs parent §11

All quantities ×2 instead of ×4 (SY6970QCC, SRP5030CA-2R2M, DW01A+SI4920DY, reverse P-FET, PTC,
NTC, SK6812). Passives R/C **0603** throughout unless a section above says otherwise.

| Change | Part |
|---|---|
| **Deleted** | TCA9548A mux; IS31FL3731 + 64-LED matrix; mezzanine connectors; gauge-button wiring across J1s |
| **Added** | on-board tactile WAKE switch (6×6 mm THT or SMD tact); 1×4 screw terminal block (pull-pin loops, replaces 4× 2-pin terminals) |
| Unchanged | CH224K, ESP32-S3-WROOM-1, AP2112K-3.3, USBLC6-2SC6, MMBD4448HCQW, 2N7002 |

## 9. Open items

1. **Sled dimensions + mounting-hole pattern** — outline TBD; design proceeds electrically first.
2. **Nosecone FC current draw** → PTC hold/trip rating and DW01A OC sanity check.
3. **What exactly are the two loads?** (Two redundant FCs? FC + tracker?) — affects nothing
   electrical so far, but name the connector pins properly once known.
4. **18650 holder orientation on the sled** — axial G-loading on holders; confirm retention.
5. Confirm nosecone is RF-transparent for BLE (fibreglass tip likely fine; carbon is not).
6. **Rich status-LED scheme** — SK6812 vs on-board IS31FL3731 charlieplex bar (§6). Recommendation
   is SK6812-only; decide before capture since it changes the I²C bus loading and LED area.

## 10. Revision history

| Version | Notes |
|---|---|
| 0.1 | Initial: 2-channel FC/logic-only derivative of architecture.md v0.5 on a single sled-mounted board. Mux dropped (dual ESP I²C buses), gauge dropped (SK6812 + /STAT only), WAKE button moved on-board, PD request 9 V default. |
