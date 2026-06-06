# LED Panel Board — Component Reference

## The Display

This is a late 1990s large-format LED dot-matrix display, likely used in a public transit or information board context. It is built from a hierarchy of modular components:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     FULL DISPLAY  200 × 64 pixels                       │
│                                                                         │
│  Strip 1 ████████████████████████████████████████████  rows  0–15       │
│  Strip 2 ████████████████████████████████████████████  rows 16–31       │
│  Strip 3 ████████████████████████████████████████████  rows 32–47       │
│  Strip 4 ████████████████████████████████████████████  rows 48–63       │
└─────────────────────────────────────────────────────────────────────────┘
```

| Layer | Count | Description |
|-------|-------|-------------|
| Display | 1 | 200 columns × 64 rows — 12,800 LEDs total |
| Strips | 4 | Stacked vertically, each independently driven by the control PC |
| Boards per strip | 5 | Daisy-chained horizontally via 20-pin flat cable |
| Modules per board | 16 | Arranged as 2 rows × 8 modules |
| Module | — | 5 columns × 8 rows = 40 LEDs |

Each strip has its own dedicated flat cable connection to the control PC and is driven completely independently.

---

## Controller Board

### MM74HC244WM — Octal Buffer / Line Driver

- **Function:** 8-bit buffer and line driver with 3-state outputs
- **Package:** 20-pin Wide SOIC (WM = Wide-body SOIC)
- **Supply voltage:** 2V – 6V
- **Purpose on board:** First-stage signal buffering between control logic and the LED board. Boosts drive strength so one signal can drive many loads. The 3-state outputs allow multiple devices to share a bus.

| Pin | Name | Description |
|-----|------|-------------|
| 1 | /OE1 | Output Enable 1 (active LOW) |
| 19 | /OE2 | Output Enable 2 (active LOW) |
| 2,4,6,8,11,13,15,17 | A1–A8 | Inputs |
| 3,5,7,9,12,14,16,18 | Y1–Y8 | Outputs |
| 10 | GND | Ground |
| 20 | VCC | Supply voltage |

---

### TL74HC04D — Hex Inverter

- **Function:** 6 independent NOT gates
- **Package:** SOIC-14 (D suffix)
- **Supply voltage:** 2V – 6V
- **Purpose on board:** Signal polarity correction — flips active-HIGH signals to active-LOW (or vice versa) as required by downstream chips. Can also be used to buffer and clean up clock signals.

**Truth table (per gate):**

| Input | Output |
|-------|--------|
| LOW (0) | HIGH (1) |
| HIGH (1) | LOW (0) |

---

### 74HC02D — Quad 2-Input NOR Gate

- **Function:** 4 independent NOR gates
- **Package:** SOIC-14 (D suffix)
- **Supply voltage:** 2V – 6V
- **Purpose on board:** Combining control signals, generating active-LOW enable/reset signals, and potentially forming SR latches for signal conditioning.

**Truth table (per gate):**

| Input A | Input B | Output |
|---------|---------|--------|
| LOW (0) | LOW (0) | HIGH (1) |
| LOW (0) | HIGH (1) | LOW (0) |
| HIGH (1) | LOW (0) | LOW (0) |
| HIGH (1) | HIGH (1) | LOW (0) |

---

### NE555 (×2) — Timer IC (STMicroelectronics)

- **Manufacturer:** STMicroelectronics
- **Date code:** 731 (Week 31, ~1997)
- **Package:** 8-pin DIP or SOIC
- **Supply voltage:** 4.5V – 16V
- **Purpose on board:** Two 555 timers provide the timing backbone of the display:
  - **NE555 #1 (Astable mode):** Generates a continuous square wave clock for LED matrix scanning
  - **NE555 #2 (Monostable mode):** Generates precise strobe/latch pulses to control LED refresh timing

| Pin | Name | Description |
|-----|------|-------------|
| 1 | GND | Ground |
| 2 | TRIG | Trigger input |
| 3 | OUT | Output |
| 4 | RESET | Reset (active LOW) |
| 5 | CTRL | Control voltage |
| 6 | THR | Threshold |
| 7 | DIS | Discharge |
| 8 | VCC | Supply voltage |

---

## LED Board

### 74HC244Q (×4) — Octal Buffer / Line Driver (Automotive Grade)

- **Function:** Same as MM74HC244 — 8-bit buffer with 3-state outputs
- **Package:** Q suffix = automotive/high-reliability grade
- **Temperature range:** −40°C to +125°C
- **Purpose on board:** Final output stage driving the **8 row lines** of the LED matrix with sufficient current. Four chips provide 32 buffered drive lines total.

---

### 74HC4094 (×10) — 8-bit Shift Register with Output Latch

- **Function:** Serial-in, parallel-out 8-bit shift register with storage latch
- **Purpose on board:** Column data is shifted in serially and latched to all 40 column outputs simultaneously. The 10 chips are split into two banks of 5, one per physical module row.

```
2 banks × 5 chips × 8 outputs = 40 column lines = 8 modules × 5 columns
```

> ⚠️ **IMPORTANT:** The two banks have **separate DATA lines**. The top and bottom module rows are driven independently, allowing them to display different content. CLOCK and STROBE are shared across both banks, but each bank requires its own serial data feed.

**How it works:**
1. Serial column data is clocked in simultaneously on DATA_TOP and DATA_BOT
2. Each bank of 5 chips is independently chained — data ripples through all 40 stages per bank
3. A single strobe/latch pulse locks all 80 bits (40 top + 40 bottom) to outputs at once
4. LEDs in the selected row light up in both banks according to their respective latched data

---

### 74HC238 (×2) — 3-to-8 Line Decoder

- **Function:** Decodes a 3-bit binary input into 1 of 8 active outputs
- **Purpose on board:** Drives the **row scanning** of the LED matrix. Two chips handle one bank of 8 rows each — HC238 #1 for the top module row, HC238 #2 for the bottom module row. Both receive the same 3-bit address and scan in sync, selecting the same logical row in each bank simultaneously.
Two chips likely handle row selection and module bank/enable control respectively.

## 20-Pin Flat Cable Pinout

Connects the controller board to the LED board. Pin assignments documented here are as measured on the **LED board** connector. Standard IDC ribbon cable layout: **odd pins on one side, even pins on the other**. Pins confirmed by physical tracing are marked ✅; unknown pins are marked `?`.

```
 1   3   5   7   9  11  13  15  17  19
 2   4   6   8  10  12  14  16  18  20
```

| Odd pin | Signal | Description | Even pin | Signal | Description |
|---------|--------|-------------|----------|--------|-------------|
| 1 | ? | Unknown | 2 | ? | Unknown |
| 3 | ? | Unknown | 4 | ? | Unknown |
| 5 | DATA_TOP | Serial data, top bank (→ HC4094 bank 1) | DATA_BOT | Serial data, bottom bank (→ HC4094 bank 2) |
| 7 | STROBE | Latch pulse, shared across both banks | 8 | CLOCK | Shift clock, shared across all 10 HC4094s |
| 9 | VCC | 12V | 10 | ? | Unknown |
| 11 | ? | Unknown | 12 | ? | Unknown |
| 13 | ? | Blocked | 14 | ? | Unknown |
| 15 | VBB | 5V | 16 | VBB | 5V |
| 17 | ? | Unknown | 18 | ? | Unknown |
| 19 | GND | Ground | 20 | GND | Ground |

 4 |  |
**Candidates for unknown pins:** A0/A1/A2 (row address → HC238), OE (output enable → HC4094/HC244Q), RESET.


---

## Board Daisy-Chain Topology

The 5 LED boards are connected in series via the 20-pin flat cable. The serial output (`QS1`) of the last HC4094 chip on each board feeds into the `DATA` pin of the next board's first chip. CLOCK and STROBE are shared across all boards.

```
CONTROLLER
    │
    ├─ DATA_TOP ──► [Board 1, top bank] ──QS1──► [Board 2, top bank] ──► ... ──► [Board 5, top bank]
    ├─ DATA_BOT ──► [Board 1, bot bank] ──QS1──► [Board 2, bot bank] ──► ... ──► [Board 5, bot bank]
    ├─ CLOCK    ─────────────────────────────────────── all boards (shared)
    └─ STROBE   ─────────────────────────────────────── all boards (shared)
```

### Bit ordering

To draw on a specific board, place pixel data at the correct offset in the **200-bit stream** (per bank, per row):

```
Bit position: [0 ────────── 39][40 ─────────79][80 ────────119][120 ───────159][160 ──────199]
                  Board 5           Board 4          Board 3          Board 2         Board 1
              ◄── clocked in first                                  clocked in last ──►
```

> The **last 40 bits** clocked in appear on **Board 1** (closest to the controller). The **first 40 bits** clocked in appear on **Board 5** (furthest away).

To target a specific board, fill all other board slots with `0x00` and place your data at the correct offset.

---

## Full Signal Flow (per strip)

```
[CONTROLLER BOARD]
        |
        |  NE555 #1 → Continuous clock signal
        |  NE555 #2 → Strobe / latch pulse
        |  74HC02D  → Combines control signals (NOR logic)
        |  74HC04D  → Inverts signals to correct polarity
        |  74HC244  → First-stage output buffering
        |
        ↓  Flat cable / connector (20-pin)
        |
[LED BOARD]
        |
        |  5× 74HC4094  → Shift in 40 bits via DATA_TOP → latch top 40 column outputs
        |  5× 74HC4094  → Shift in 40 bits via DATA_BOT → latch bottom 40 column outputs
        |               (CLOCK and STROBE shared; DATA_TOP and DATA_BOT are independent)
        |
        |  74HC238 #1  → Decode 3-bit row address → activate 1 of 8 rows (top bank)
        |  74HC238 #2  → Decode 3-bit row address → activate 1 of 8 rows (bottom bank)
        |               (both receive same address, scan in sync)
        |
        |  4× 74HC244Q  → Buffer and drive row/column lines with sufficient current
        |
        ↓
[640 LEDs — 2 banks of 8×(5×8) modules, 40 columns × 16 rows]
```

### Refresh cycle (one full frame):
1. NE555 #1 generates the master clock
2. Column data for Row N is shifted simultaneously into the top bank (DATA_TOP) and bottom bank (DATA_BOT) — independently, allowing different content per bank
3. NE555 #2 fires a strobe — both HC4094 banks latch their 40 column bits to outputs
4. Both HC238s select Row N (0–7) via the same 3-bit address, activating that row in both banks
5. HC244Q drives the selected rows and active columns
6. Steps 2–5 repeat for all 8 row positions (covering all 16 physical rows across both banks) at high speed (~1000 Hz+)
7. Persistence of vision makes the full image appear static to the human eye
