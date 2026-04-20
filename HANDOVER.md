# ESE 3700 Final Project — Handover Document
**16×4 SRAM Memory Design | 22nm HP Technology | Electric VLSI**

---

## Project Overview

The goal is a fully functional 16-row × 4-column SRAM designed in Electric using 22nm HP SPICE models. The design must operate above 500 MHz at VDD ≤ 1V and will be evaluated on a Figure of Merit (FOM = Area × Power × Delay²).

---

## Completed Work

### 1. SRAM Cell (`sram_cell;1{sch}`)
The 6T SRAM cell is fully schematic-complete and **simulation-validated**.

**Transistor sizing:**
| Transistor | Role | Width | Length |
|---|---|---|---|
| M1, M3 (nmos@3, nmos@0) | Pull-down | W=2 | L=1 |
| M2, M4 (pmos@1, pmos@0) | Pull-up | W=1 | L=1 |
| M5, M6 (nmos@4, nmos@5) | Access | W=1 | L=1 |

**Resulting ratios:**
- Cell Ratio (CR) = W_pulldown / W_access = **2** — provides read stability
- Pull Ratio (PR) = W_access / W_pullup = **1** — marginal but functional writability

**Validated operations:** Write '0', Hold '0', Write '1', Hold '1'

---

### 2. SRAM Array Cell (`sram_cell_array;1{sch}`)
Identical to `sram_cell` but with the Q and ~Q output ports removed. This version is used inside `sram_row` — the Q/~Q ports are not needed at the array level and exposing them causes unnecessary complexity.

---

### 3. SRAM Row (`sram_row;1{sch/ic}`)
Four `sram_cell_array` instances sharing one WL. Each cell has its own independent BL/~BL column pair. **Manually wired and icon manually drawn by user** for robustness.

**Icon port convention (sram_row):**
- **WL** → Left side
- **~BL0–~BL3** → Top (bitlines run vertically upward through the array stack)
- **BL0–BL3** → Right side

---

### 4. SRAM Array (`sram_array;1{sch/ic}`)
16 `sram_row` instances stacked vertically. **Fully wired in the jelib file.**

**Wiring strategy:**
- **WL0–WL15**: 16 individual horizontal wires to Off-Page connectors on the left. Each connector is placed at the exact WL port y-coordinate to avoid diagonal routing.
- **~BL0–~BL3**: Four vertical chains, each connecting all 16 rows' ~BL ports directly (row-to-row, no intermediate WirePins needed). Top connectors are offset 2 units left of the bus column so the `y` terminal aligns exactly with the bus.
- **BL0–BL3**: Each BL port exits the row icon at x=17.5/18. All four would collide on the same vertical line, so horizontal stubs route them to dedicated columns: BL0→x=22, BL1→x=24, BL2→x=26, BL3→x=28. WirePins form the vertical bus per column.

**Icon port convention (sram_array):**
- **WL0–WL15** → Left side (16 ports, stacked vertically)
- **BL0/~BL0, BL1/~BL1, BL2/~BL2, BL3/~BL3** → Right side (paired per column)

---

### 5. Supporting Cells (generated, not yet fully validated)
The following cells exist in `Final_Project.jelib` and are wired at the schematic level. They need individual unit testing before top-level integration:
- `predecoder_2to4;1{sch}` — 2-bit to 4-line predecoder
- `decoder_4to16;1{sch}` — 4-to-16 full decoder (2× predecoder + 16× AND2)
- `precharge;1{sch}` — PMOS precharge circuit
- `write_driver;1{sch}` — BL/~BL write driver
- `column_driver;1{sch}` — combines precharge + write driver + read buffer
- `nonoverlap_clk;1{sch}` — non-overlapping clock generator (phi1, phi2)
- `gate_latch;1{sch}` — gated latch for input stabilization
- `control;1{sch}` — integrates clock gen and latches
- `sram_16x4;1{sch}` — top-level cell (cleaned, no wires — needs wiring)
- `sram_test;1{sch}` — top-level testbench (instantiates sram_16x4)

---

## Validated Test Strategy

### Single Cell Test (`sram_cell;1{sch}`)

Three VPulse sources on WL, BL, ~BL. Key parameters based on the course library style:

**VDD:** `V_Generic`, DC = 1V

| Source | V1 | V2 | TD | PW | Per | TR/TF |
|---|---|---|---|---|---|---|
| WL | 0 | 1V | 0 | 10ns | 20ns | 10ps |
| BL | 0 | 1V | 10.5ns | 20ns | 40ns | 10ps |
| ~BL | 1V | 0 | 10.5ns | 20ns | 40ns | 10ps |

**Why 10.5ns (not 10ns) for BL/~BL TD:**
At exactly t=10ns, WL falls AND BL/~BL transition simultaneously. During the 10ps WL fall edge, WL is still partially on while BL=1 and ~BL=0 — this is the exact Write '1' condition, causing an inadvertent glitch write. The 0.5ns offset ensures BL/~BL only change after WL is fully at 0V.

**Why write '0' first:**
SPICE initializes all nodes at 0V. Writing '1' first fights the pull-down transistors (W=2) from the undefined initial state — Q only partially charges, then collapses when WL=0. Writing '0' first aligns with initial conditions and establishes a defined state for the subsequent Write '1'.

**Simulation:** `.tran 0.01n 40n`

---

## Pitfalls and Mistakes to Avoid

### Electric / jelib Format

1. **Duplicate instance declarations**: When editing the jelib manually or via AI tools, it is easy to accidentally insert the same instance twice (e.g., `ctrl@0` and `dec@0` appeared twice in `sram_16x4{sch}`). Electric will open the file but behave unpredictably. Always check for duplicate `I...` lines when opening a cell.

2. **Off-Page connector y-terminal offset**: In Electric, the wire connection point (`y` terminal) of an Off-Page connector is **2 units to the right** of the connector's center position (for default left-pointing connectors). When writing Awire entries manually, use `connector_x + 2` for the x-coordinate. Getting this wrong means the wire appears to connect but creates a floating net.

3. **BL lines at the same x-coordinate**: In `sram_row`, all four BL port exits from `sram_cell_array` are at the same relative x. If you try to bus them with a single vertical wire, all four columns short together. You must route each BL to a unique x-position with horizontal stubs before running the vertical bus.

4. **Zero-length wires**: Electric allows zero-length `Awire` entries (same start and end coordinate) — these are used to mark a port as connected to a wire/bus at the same physical point. They are valid but can look wrong when inspecting the jelib manually.

5. **~BL/BL bus orientation in the icon hierarchy**: The `sram_row` icon has ~BL on the **top** (because it runs vertically through the array). The `sram_array` icon has BL and ~BL on the **right** (because it connects horizontally to column drivers). Do not copy the row icon port convention to the array icon.

6. **Bbox parameter**: The `Nschematic:Bbox` node in an icon controls the visible boundary box. If it is too small, ports appear outside the box and Electric may warn about out-of-bounds ports. Always verify the bbox covers all port positions.

### SPICE Simulation

7. **SPICE initial conditions (all nodes start at 0V)**: The SRAM cell has no preferred startup state. Both inverters start at 0V — a degenerate metastable point. Always initialize the cell to a known state (write '0' first) before testing state changes. Never start by writing '1' from an uninitialized state.

8. **VPWL default values**: The default VPWL parameters `PWL = 0 0 1 1` mean "ramp from 0V to 1V over 1 second" — completely useless for nanosecond-scale simulation. Always replace PWL with proper nanosecond time-voltage pairs or switch to VPulse.

9. **Technology model path**: The testbench uses `.include /home1/e/ese3700/ptm/22nm_HP.pm` — this is a server path. Simulation must be run on the course server, not locally. Running locally without the model gives no errors but uses default SPICE models, producing completely wrong results.

10. **VPulse vs VPWL**: VPulse is simpler and less error-prone for periodic test signals. Use VPulse for all clock-like signals. Reserve VPWL for signals that need arbitrary waveforms (e.g., address bus patterns for full array testing).

### Design

11. **Cell ratio and write failure**: With CR=2 (pull-down W=2, access W=1), writing '1' into a cell that is holding '0' is a close contest. If CR were increased further (e.g., W=3 pull-down) for better read stability, write operations would become unreliable. The current CR=2 is a deliberate balance.

12. **Non-overlapping clocks matter**: The project requires operating above 500 MHz. If phi1 and phi2 overlap even briefly, the precharge and wordline-enable phases can conflict, corrupting the stored value. The `nonoverlap_clk` cell must be tested independently before integration.

13. **Precharge must complete before WL assertion**: In the read path, both BL and ~BL must be precharged to VDD before WL goes high. If WL asserts too early, the partially-precharged BL cannot produce a valid read. This timing margin is critical for the FOM delay measurement.

---

## Next Steps

1. **Wire `sram_16x4;1{sch}`** — the schematic currently has all instances and port connectors placed neatly but no wires. Connect: decoder → WL bus, sram_array BL/~BL → column drivers, control → precharge/WE/CLK signals, column driver Dout → top-level output ports.

2. **Unit test the decoder** — apply all 16 address combinations (A0–A3) and verify exactly one WL goes high per address. Use 4× VPulse sources with binary-weighted periods.

3. **Unit test the column driver** — verify precharge, write, and read phases independently.

4. **Full system simulation** — once wired, run `.tran 0.01n 200ns` with a sequence of write/read operations across multiple addresses.

5. **FOM measurement** — measure propagation delay (write-to-read cycle time), average power, and estimate area. Compute FOM = Area × Power × Delay².

---

## File Locations

| File | Description |
|---|---|
| `Final_Project.jelib` | Main Electric library — all cells |
| `ese3700.jelib` | Old homework library — reference for VPulse syntax and component reuse |
| `proj2.pdf` | Project specification (VDD, FOM, architecture requirements) |
| `lec17.pdf` | SRAM cell fundamentals, CR/PR ratios |
| `lec18.pdf` | Memory architecture, decoders, column circuitry |

---

*Document prepared April 2026. All schematic work completed in Electric using the `.jelib` format.*
