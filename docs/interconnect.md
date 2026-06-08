# AURC BMS — Interconnect Contract

**The single source of truth for everything that crosses a board boundary.**

The BMS is now three separately-laid-out KiCad projects (`board1_power`, `board2_monitor`,
`board3_dist`). Nothing but this document guarantees they mate. Treat every pin map below as
**authoritative**: if a board's connector disagrees with this file, the board is wrong, not the
file. Change a pinout here first, in a commit of its own, before touching copper on either side.

Derived from `architecture_v0.2.md` §2–§3, §6, §8. Read those for *why*; read this for *what pin*.

---

## Channels

Four electrically independent 1S channels. One 18650 each. Independent positive rails; returns
join at a **single star point** (location TBD, §13.3), not a shared plane.

| Channel | Load | Pull-pin colour/key | Notes |
|---|---|---|---|
| CH1 | FC1 (primary flight computer) | FC key | ESP module lives on this channel |
| CH2 | FC2 (backup flight computer) | FC key | |
| CH3 | Pyro1 | PYRO key (distinct) | low-Rds protection FET, carries firing pulse |
| CH4 | Pyro2 | PYRO key (distinct) | low-Rds protection FET, carries firing pulse |

Pin-1 / channel-ordering convention used everywhere below: **CH1 → CH2 → CH3 → CH4** (FC1, FC2,
Pyro1, Pyro2), low pin number first. Keep this order identical on both halves of every connector.

---

## Board stack (mechanical, to confirm — §13.1)

```
        loads (FC1/FC2/Pyro1/Pyro2)
              ▲   ▲
          Conn A   Conn B
              │
   ┌──────────────────────┐
   │  Board 3 — Distribution│  passive fan-out
   └──────────┬─────────────┘
        J3  power 8-pin  (through pull-pin enable interface)
              │
   ┌──────────────────────┐
   │  Board 2 — Safe/Monitor│  spine: 64-LED gauge + 4× pull pins
   └───┬──────────────┬─────┘
   J2 power 8-pin   J2s signal 4-pin
       │              │
   ┌──────────────────────┐
   │  Board 1 — Power/Charge│  USB-C, 4× charger, cells, fuel gauges, ESP
   └──────────────────────┘
```

Boards 1 and 3 stack parallel; Board 2 is the spine carrying the gauge and the pull-pin interface.
Board-to-board via **stacking mezzanine connectors** — part/pitch **TBD (§13.1)**. The pin maps
below are logical; assign physical positions once the mezzanine part is chosen, preserving the
CH1→CH4 ordering and keeping each channel's V adjacent to its own GND.

---

## J1 — Board 1 ↔ Board 2  (power, 8-pin)

4× channel rails + 4× returns. **3 A per pin.** Each channel's V paired adjacent to its own GND
(independent returns — do not common the grounds here).

| Pin | Net | Pin | Net |
|---|---|---|---|
| 1 | CH1_V (FC1)   | 2 | CH1_GND |
| 3 | CH2_V (FC2)   | 4 | CH2_GND |
| 5 | CH3_V (Pyro1) | 6 | CH3_GND |
| 7 | CH4_V (Pyro2) | 8 | CH4_GND |

Each rail is the per-channel **`CHn_V`** output: a `0Ω` strap selects its source — **`SYS`** (SY6970
power-path, **default**) or **`LOAD_P`** (direct protected battery) — and a single PTC fuses it after
the selector. Live whenever a charged cell is fitted (independent of USB). See architecture §4/§5/§12.

## J1s — Board 1 ↔ Board 2  (signal, 4-pin)

I²C to the Board-2 gauge driver, plus the gauge-button **wake** line.

| Pin | Net | Notes |
|---|---|---|
| 1 | SDA | I²C, pulled up 4.7 k on **Board 1** |
| 2 | SCL | I²C, pulled up 4.7 k on **Board 1** |
| 3 | **WAKE** | Board-2 gauge button → Board-1 ESP enable (momentary; the button connects `CH1_V` → `WAKE`, which feeds the ESP-enable diode-OR) |
| 4 | SGND | signal ground reference |

**Board-2 power:** the gauge (IS31FL3731 + LED matrix) is powered **directly from `CH1_V`** via the 8-pin power connector (J1) — no 3V3 pass-through, no Board-2 regulator. The 3.3 V I²C bus is level-compatible across the cell range; the gauge is dark in flight (ESP off → no I²C frames).

**I²C topology:** ESP → TCA9548A → 4× **SY6970** chargers (each addr 0x6B, integrated V/I/charge/TS
monitoring — no separate fuel gauge), all on Board 1. The IS31FL3731 gauge driver on Board 2
(addr **0x74**, no conflict) sits **upstream of the mux** and is reached across J1s. Only SDA/SCL
cross the boundary; the mux and chargers stay on Board 1. Pull-ups (4.7 k) live on Board 1.

> **RESOLVED (§13.2).** The dedicated channel-live line was **dropped** — pin 3 now carries `WAKE`.
> Channel-live state is derived in firmware from the SY6970 I²C readings and streamed over BLE /
> shown on the gauge, so no hardware sense line is needed. J1s stays 4-pin.

---

## J2 — Board 2 ↔ Board 3  (power, 8-pin, through the pull-pin enable interface)

Same 4× V / 4× GND map as J1, **but routed through the four pull-pin power-enable switches on
Board 2.** Pin inserted = channel isolated (safe); pin removed = power flows (*default-unpinned is
live* — use captive/retained pins, §8).

| Pin | Net | Pin | Net |
|---|---|---|---|
| 1 | CH1_V_SW (FC1)   | 2 | CH1_GND |
| 3 | CH2_V_SW (FC2)   | 4 | CH2_GND |
| 5 | CH3_V_SW (Pyro1) | 6 | CH3_GND |
| 7 | CH4_V_SW (Pyro2) | 8 | CH4_GND |

`_SW` = downstream of the pull-pin switch. Grounds pass straight through (returns stay independent).
**3 A per pin.** Pull pins themselves are 4× 2-pin screw terminals on Board 2, **not** part of this
connector. Key/colour the two PYRO pins distinctly from the two FC pins.

---

## J3 — Board 3 → loads  (distribution fan-out)

Board 3 is passive. It splits the four switched channels into two load connectors, each carrying a
flight computer **and** its paired pyro board. Per-channel V and GND kept separate end-to-end.

### Conn A — FC1 + Pyro1
| Pin | Net |
|---|---|
| 1 | FC1_V    (= CH1_V_SW) |
| 2 | FC1_GND  (= CH1_GND)  |
| 3 | PYRO1_V  (= CH3_V_SW) |
| 4 | PYRO1_GND(= CH3_GND)  |

### Conn B — FC2 + Pyro2
| Pin | Net |
|---|---|
| 1 | FC2_V    (= CH2_V_SW) |
| 2 | FC2_GND  (= CH2_GND)  |
| 3 | PYRO2_V  (= CH4_V_SW) |
| 4 | PYRO2_GND(= CH4_GND)  |

The BMS delivers power only — **no arming, no pyro circuitry** on any of these three boards. Firing
FETs, opto-isolation, e-match continuity all live on the downstream pyro boards. The BMS must
deliver clean, transient-free power so applying it can't glitch a load (size input caps + soft-start
on Board 1, §8).

---

## Net-name conventions (use these exact labels on hierarchical pins / global labels)

- `CH{1..4}_V`      — protected, charge-side rail, Board 1 output → J1
- `CH{1..4}_V_SW`   — same rail after the Board-2 pull-pin switch → J2/J3
- `CH{1..4}_GND`    — that channel's independent return (star-bonded at one point only)
- `SDA` / `SCL`     — I²C bus (4.7 k pull-ups on Board 1)
- `WAKE`            — Board-2 gauge button → Board-1 ESP enable (momentary; replaced `CH_LIVE`)
- `SGND`            — signal/I²C reference

Star-point: bond the four `CH*_GND` returns at exactly **one** location (likely Board 1 by the
cells, §13.3) via per-channel ground-to-star 0 Ω links (§12). Nowhere else.

---

## Open items affecting this contract

1. **§13.1** Mezzanine connector part + pitch → fixes physical pin assignment of J1/J1s/J2.
2. ~~**§13.2** Channel-live sense granularity~~ **RESOLVED** — line dropped; pin 3 = `WAKE`, channel-live derived in firmware.
3. **§13.3** Star-point location → which board carries the ground-to-star 0 Ω links.

When any of these closes, edit this file **first**, in its own commit, then update both board edges.
