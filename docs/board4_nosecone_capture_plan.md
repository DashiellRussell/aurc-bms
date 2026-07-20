# Board 4 — Nosecone BMS — Schematic Capture Plan

Capture plan for `board4_nosecone/board4_nosecone.kicad_sch`. Authority order:
`board4_nosecone_architecture.md` (this board's deltas) → `architecture.md` v0.5 (inherited design)
→ this file (capture detail). Where this plan and board1's plan disagree, follow this one —
board1's plan (`board1_power_capture_plan.md`) predates the as-built v0.5 ESP gating.

**Locked decisions (2026-07-20):** single board · 2× FC/logic channels (no pyro) · 2× SY6970, one
per ESP32-S3 I²C controller (**no TCA9548A**) · **2× SK6812** ESP-driven status RGBs + hardwired
/STAT-fallback LEDs (**no IS31FL3731**) · on-board WAKE tact switch · 1×4 screw terminal for both
pull-pin loops · PD request 9 V default · rectangular sled (dims TBD).

> **Head start (already done):** `charge_chanel.kicad_sch` and `USB_IN.kicad_sch` are **copied from
> board1 as-built** into `board4_nosecone/` — both proven sheets, lib refs identical (`BMS:` via the
> project lib tables). The ESP/monitor section is the only fresh drawing, and it's *simpler* than
> board1's (mux deleted).

---

## 1. Sheet hierarchy

```
board4_nosecone.kicad_sch  (ROOT — ESP section drawn here, board1-style)
├── USB_IN.kicad_sch          ×1   USB-C + CH224K + ESD    ← copied, edit strap only
└── charge_chanel.kicad_sch   ×2   SY6970 + protection + 18650 + SK6812   ← copied, use as-is
ROOT also holds: ESP32-S3 + AP2112K + diode-OR gate, WAKE button, J1 out, 1×4 pull-pin block,
star ground, hardwired LEDs.
```

Instantiate **one** `charge_chanel` sheet file twice (NCH1, NCH2). Differences are wired at root
via sheet pins, exactly like board1.

## 2. Global nets (N-prefixed per architecture §2)

| Net | Meaning |
|---|---|
| `VBUS` | PD rail (**9 V** default) |
| `NCH1_V`, `NCH2_V` | per-channel SYS rail (charger out, post-strap/PTC) |
| `NCH1_V_SW`, `NCH2_V_SW` | downstream of pull-pin loop → J1 |
| `NCH1_GND`, `NCH2_GND` | independent returns → single on-board star by the cells |
| `SDA0/SCL0`, `SDA1/SCL1` | ESP I2C0 → SY6970 #1, I2C1 → SY6970 #2 (4.7 k pull-ups to `+3V3` on **each** bus, R 0603) |
| `USB_DP/USB_DM` | native USB to the S3 |
| `+3V3` | AP2112K output |
| `WAKE` | tact switch: `NCH1_V` → button → diode-OR input |
| `STAT1/STAT2` | /STAT → hardwired fallback LEDs |
| `RGB_DATA` → DIN/DOUT chain | 2× SK6812, `LED_VDD` = `NCH1_V` |

## 3. Per-sheet work

**`USB_IN` (copied):** only change — **CH224K strapped to 9 V**. Per datasheet, 9 V = CFG1 = CFG2 =
CFG3 all LOW (GND). As copied from board1: CFG1 already has a 24 K to GND (= logic 0, good), but
**CFG2 and CFG3 are floating** — add a **0 Ω (`R_0603_1608Metric`) from each of CFG2 and CFG3 to
GND**. No 12 V DNP position kept: CFG2/CFG3 cap at 3.7 V so a clean 12 V strap (CFG3 = 1) needs a
≤3.7 V pull-up rail — treat 12 V as a rework if ever needed. Everything else (USBLC6, bulk cap,
input LED) stays.

**`charge_chanel` ×2 (copied):** use as-is. The copied sheet already carries SY6970 + DW01A +
SI4920DY + AO3401A + SRP5030 + PTC + NTC + /STAT + SK6812 with the right cap set (47 nF bootstrap,
6.8 µF PMID, etc.). Per-instance wiring at root: instance 1 gets `SDA0/SCL0`, instance 2 gets
`SDA1/SCL1`. **PTC value may change** vs board1 (no firing pulse — size to FC draw, open item).

**ROOT / ESP section (fresh, ~board1's root minus the mux):**
- `NCH1_V → AP2112K-3.3 → +3V3`. EN = diode-OR (`MMBD4448HCQW`) of {VBUS 100k/33k divider,
  always-on 100 k strap from `NCH1_V`, `WAKE`, ESP self-hold GPIO}, 100 k pull-down, `2N7002`
  RBF-jumper shunt on EN (jumper in = ESP off). All R 0603. Copy this block from board1's root —
  it is the as-built v0.5 circuit.
- ESP32-S3-WROOM-1: `USB_D±` ← `USB_DP/DM`; **two GPIO pairs → `SDA0/SCL0` and `SDA1/SCL1`**
  (any GPIOs — the S3's two I²C controllers route via the GPIO matrix); one GPIO → `RGB_DATA`
  (330 Ω series, R 0603); one GPIO → self-hold into the diode-OR; EN RC + BOOT pad per board1.
- **WAKE tact switch** (6×6 mm): `NCH1_V` → switch → `WAKE` net (through the same limiting as
  board1's J1s arrangement).
- **Firmware note:** advertise as a distinct BLE name (`AURC-BMS-NOSE` vs the main board's
  `AURC-BMS-MAIN`) so the Web Bluetooth page can tell the two boards apart. No RF coexistence
  concern — BLE AFH handles two radios 50 cm apart trivially.

**ROOT / power-out section:**
- Each `NCHn_V` → **1×4 screw terminal** (pull-pin loop: pins 1–2 = NCH1, 3–4 = NCH2) → `NCHn_V_SW`.
- **J1 — loads, 4-pin, 3 A/pin:** 1 `NFC1_V`(=NCH1_V_SW) · 2 `NFC1_GND` · 3 `NFC2_V`(=NCH2_V_SW) ·
  4 `NFC2_GND`.
- **Star ground by the cells:** `NCH1_GND` + `NCH2_GND` + USB GND + signal ground join at exactly
  one node via per-channel 0Ω links (R 0603). No shared plane across channels.

## 4. Config straps (0Ω / DNP, all R 0603)

PD voltage 9 V (default) / 12 V · PD bypass (force 5 V) · D+/D− series 0Ω ×2 · ESP always-on strap ·
per-channel `SYS` vs `LOAD_P` source select ×2 · per-channel ground-to-star ×2 · SY6970 ILIM ×2.

## 5. ERC / verification checklist

- [ ] Both `NCHn_V_SW` reach J1; both `NCHn_GND` reach the star then J1; exactly **one** star node.
- [ ] I2C0 → charger 1 only, I2C1 → charger 2 only; 4.7 k pull-ups once per bus, to `+3V3`.
- [ ] ESP rail from `NCH1_V` via AP2112K — never raw VBUS; diode-OR + 2N7002 shunt per v0.5.
- [ ] Each SY6970: TS NTC, ILIM, REGN/PMID/BAT/SYS caps, inductor; `/QON` strapped high.
- [ ] RGB chain: `RGB_DATA` → NCH1 DIN, NCH1 DOUT → NCH2 DIN; `LED_VDD` = `NCH1_V`; /STAT LEDs kept.
- [ ] Pull-pin block: loop in the **positive** path, `_SW` naming downstream, silkscreen labels.
- [ ] CFG strap = 9 V default; ERC clean, PWR_FLAGs placed; footprints assigned (R/C 0603).

## 6. Capture order

1. Open the copied `USB_IN`, retarget the CFG strap to 9 V.
2. ROOT: instantiate `USB_IN` + one `charge_chanel`, wire `VBUS` + `NCH1_V`, ERC.
3. ROOT: draw the ESP block (copy board1 root's as-built diode-OR/AP2112K section, delete the
   TCA9548A, wire the two I²C pairs) + WAKE switch.
4. Add the second `charge_chanel` instance, pull-pin block, J1, star ground, LEDs, straps.
5. Full ERC vs §5 → annotate → footprints → layout (outline waits on sled dims).
