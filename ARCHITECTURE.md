# ESE 3700 Final Project — Architecture & Handover
**16×4 SRAM | 22nm HP | Due April 20, 11:59pm**

---

## Project Summary

Design a fully functional **16-word × 4-bit SRAM** in Electric VLSI using 22nm HP SPICE models.
Must operate above **500 MHz** at **VDD ≤ 1V**.
Graded on **FOM = 60 × BitcellArea × Power × Delay²** (lower is better).

Inputs: `CLK`, `WE`, `A[3:0]`, `Din[3:0]` — Outputs: `Dout[3:0]`

---

## Work Split

| Partner | Responsibility |
|---|---|
| Partner | `decoder_4to16` — unit test all 16 address combinations, verify exactly one WL goes high |
| You | Column circuitry — fix `write_driver`, validate `precharge`, complete `column_driver`, wire top-level `sram_16x4` |

---

## Completed ✅

- `sram_cell` — 6T cell, simulation-validated (write 0, hold 0, write 1, hold 1)
- `sram_cell_array` — array variant (no Q/~Q ports)
- `sram_row` — 4 cells sharing one WL, icon manually drawn
- `sram_array` — 16 rows fully wired (WL, BL, ~BL buses)
- All sub-cells exist in schematic: `predecoder_2to4`, `decoder_4to16`, `precharge`, `write_driver`, `column_driver`, `nonoverlap_clk`, `gate_latch`, `control`

## Still Needed ⚠️

- [ ] Fix `write_driver` — all 4 NMOS gates currently wired to WE only (see bug below)
- [ ] Fix `control` — add INV on phi1 before phi_pre output (PRE is active-low, phi1 is active-high)
- [ ] Complete `column_driver` — connect `inv@1|y` to Dout or remove it
- [ ] Unit test `precharge`, `write_driver`, `column_driver`
- [ ] Wire `sram_16x4` top-level (instances placed, no wires)
- [ ] Full system simulation + FOM measurement

---

## Known Bugs

**`write_driver` bug:** All 4 NMOS gates connect to WE only. Both BL and ~BL get pulled to GND whenever WE=1, regardless of Din. Fix: bottom NMOS of BL stack → gate = `~Din`; bottom NMOS of ~BL stack → gate = `Din`.

**`control` polarity bug:** `phi_pre = phi1` is passed directly to precharge PMOS gates. PMOS needs PRE=0 to turn ON, but phi1=1 during precharge phase → precharge is OFF when it should be ON. Fix: add one inverter on phi1 before the phi_pre output in `control`.

---

## Full Circuit Architecture

```
╔══════════════════════════════════════════════════════════════════╗
║                        sram_16x4  (top level)                    ║
║                                                                   ║
║  CLK ──► [control] ──► phi_pre ──► col×4 (precharge timing)      ║
║  WE ───►            ──► WE_out ──► col×4 (write enable)          ║
║  Din[3:0] ──►       ──► Din_out ─► col×4 (write data)            ║
║                     ──► phi_wl ──► (gates WL assertion)          ║
║                                                                   ║
║  A[3:0] ──► [decoder_4to16] ──► WL0–WL15 ──► [sram_array]       ║
║                                                                   ║
║             [sram_array] BL0–3 / ~BL0–3 ──► [col_driver × 4]    ║
║                                                  │                ║
║                                               Dout[3:0] ──► OUT  ║
╚══════════════════════════════════════════════════════════════════╝
```
*Control latches WE and Din on phi1 so inputs cannot change mid-cycle and corrupt an operation.*

---

```
╔══════════════════════════════════════════╗
║              control block               ║
║                                          ║
║  CLK ──► [nonoverlap_clk]                ║
║               │                          ║
║            phi1 ──► [INV] ──► phi_pre   ║  phi1 high = precharge phase
║            phi2 ──────────► phi_wl      ║  phi2 high = access phase
║                                          ║
║  WE  ──► [gate_latch, clocked by phi1]  ║
║               └──────────────► WE_out   ║  WE is frozen during phi2
║                                          ║
║  Din ──► [gate_latch, clocked by phi1]  ║
║               └──────────────► Din_out  ║  Din is frozen during phi2
╚══════════════════════════════════════════╝
```
*The latches are critical — they prevent glitches on WE/Din from accidentally writing the wrong cell mid-cycle.*

---

```
╔═════════════════════════════════════════════════════╗
║                   One clock cycle                    ║
║                                                      ║
║  phi1 ▔▔▔▔▔╗          ╔▔▔▔▔▔                        ║
║             ╚══════════╝                             ║
║  phi2       ╔══════════╗                             ║
║  ═══════════╝          ╚════════════════             ║
║                                                      ║
║  [──── PRECHARGE ────][──────── ACCESS ─────────]   ║
║  BL,~BL → VDD          WL[n] asserts                 ║
║  latch Din/WE          write driver / cell discharge  ║
╚═════════════════════════════════════════════════════╝
```
*phi1 and phi2 never overlap — that is the whole point of the nonoverlap_clk cell.*

---

```
╔══════════════════════════════════════════════╗
║           decoder_4to16                       ║
║                                              ║
║  A[3:0] ──► [predecoder_2to4] ─► P0–P3      ║  A[3:2] → 4 partial terms
║             [predecoder_2to4] ─► P4–P7      ║  A[1:0] → 4 partial terms
║                                              ║
║  Each WL = AND(one from top, one from bottom)║
║  WL5  = P1 AND P5   (address 0101)           ║
║  WL15 = P3 AND P7   (address 1111)           ║
║                                              ║
║  16 AND gates → WL0–WL15 (only one HIGH)     ║
╚══════════════════════════════════════════════╝
```
*Predecoding saves area — 2 small decoders feed 16 AND gates instead of 16 large 4-input AND gates.*

---

```
╔══════════════════════════════════════════════════════╗
║                  column_driver  (×4, one per bit)    ║
║                                                      ║
║   PRE ─────► [precharge]  ──► BL ──┐                ║
║                            ──► ~BL─┼─► to sram_array║
║                                    │                 ║
║   Din ─────► [write_driver]──► BL ─┘                ║
║   WE ──────►              ──► ~BL                   ║
║                                                      ║
║   BL ──► [INV] ──► Dout   (read buffer)             ║
╚══════════════════════════════════════════════════════╝
```
*Precharge and write_driver both connect to the same BL/~BL wires. The precharge holds them high; the write driver selectively pulls one low.*

---

```
╔══════════════════════════════════════════════════════╗
║                    precharge                         ║
║                                                      ║
║         VDD      VDD                                 ║
║          │        │                                  ║
║   PRE──[P]      [P]──PRE    ← pull BL, ~BL to VDD   ║
║          │        │                                  ║
║         BL ─[P]─ ~BL        ← equalize BL = ~BL     ║
║                  │                                   ║
║                 PRE                                  ║
╚══════════════════════════════════════════════════════╝
```
*Active-low. PRE=0 turns all 3 PMOS ON → BL and ~BL charge to VDD and equalize.*

---

```
╔══════════════════════════════════════════════════════╗
║                   write_driver                       ║
║                                                      ║
║         BL                      ~BL                 ║
║          │                        │                  ║
║       [nmos g=WE]            [nmos g=WE]            ║
║          │                        │                  ║
║       [nmos g=~Din]          [nmos g=Din]           ║
║          │                        │                  ║
║         GND                      GND                ║
║                                                      ║
║  Din=0,WE=1: left fires  → BL=0,  ~BL held by prech ║
║  Din=1,WE=1: right fires → ~BL=0, BL held by prech  ║
╚══════════════════════════════════════════════════════╝
```
*Open-drain. Only pulls a line LOW — the precharge PMOS holds the other line HIGH.*

---

```
╔══════════════════════════════════════════════════════╗
║                    sram_array                        ║
║                                                      ║
║  WL0  ──────────────────────────── row 0            ║
║  WL1  ──────────────────────────── row 1            ║
║  ...                                                 ║
║  WL15 ──────────────────────────── row 15           ║
║                                                      ║
║         │       │       │       │                   ║
║        BL0    BL1     BL2    BL3   (vertical)       ║
║        ~BL0  ~BL1    ~BL2   ~BL3  (vertical)       ║
║         │       │       │       │                   ║
║        col0   col1    col2   col3  (column_driver)  ║
╚══════════════════════════════════════════════════════╝
```
*All 16 rows share the same 4 BL/~BL pairs. Only the selected row has its WL high.*

---

```
╔══════════════════════════════════════════════════════╗
║               sram_row  (one of 16)                  ║
║                                                      ║
║  WL ───┬────────┬────────┬────────┐                 ║
║         │        │        │        │                 ║
║       [cell]  [cell]   [cell]  [cell]               ║
║       bit0    bit1     bit2    bit3                  ║
║         │        │        │        │                 ║
║       BL0/    BL1/     BL2/    BL3/                 ║
║       ~BL0   ~BL1     ~BL2   ~BL3                   ║
╚══════════════════════════════════════════════════════╝
```
*4 cells share one WL wire. Each cell connects to its own independent BL/~BL column.*

---

```
╔══════════════════════════════════════════════════════╗
║              6T SRAM cell                            ║
║                                                      ║
║         VDD              VDD                        ║
║          │                │                          ║
║        [P2]             [P4]    ← pull-up  (W=1)    ║
║     ┌────┤                ├────┐                     ║
║     │    Q ──────────── ~Q    │                     ║
║     └────┤                ├────┘                     ║
║        [N1]             [N3]    ← pull-down (W=2)   ║
║          │                │                          ║
║         GND              GND                        ║
║                                                      ║
║  WL─[N5]─BL          ~BL─[N6]─WL                   ║
║   (access W=1)          (access W=1)                ║
╚══════════════════════════════════════════════════════╝
```
*Cross-coupled inverters store the bit. N5/N6 are the access gates — WL high connects cell to bitlines.*
*CR=2 (pull-down W=2 vs access W=1) prevents Q from flipping during a read.*

---

## Transistor Sizing (sram_cell)

| Transistor | Role | W | L |
|---|---|---|---|
| N1, N3 | Pull-down | 2 | 1 |
| P2, P4 | Pull-up | 1 | 1 |
| N5, N6 | Access | 1 | 1 |

- **Cell Ratio CR = W_pulldown / W_access = 2** → read stability
- **Pull Ratio PR = W_access / W_pullup = 1** → marginal but functional writability
- **BitcellArea** (for FOM) = sum of widths = 2+1+2+1+1+1 = **8**

---

## FOM

```
FOM = 60 × BitcellArea × Power × Delay²
```

- **Delay** = worst case of write access time and read access time
- **Power** = average current × VDD during 4× full-array write-0 then 4× write-1 cycles
- **BitcellArea** = 8 (sum of transistor widths in one repeated cell)

---

*Last updated April 2026.*
