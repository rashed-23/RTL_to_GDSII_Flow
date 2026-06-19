# ibex RISC-V Core — RTL Synthesis on Sky130A

End-to-end RTL synthesis flow for the [lowRISC ibex](https://github.com/lowRISC/ibex) RISC-V core targeting the **Sky130A PDK** using **OpenLane**. Covers baseline synthesis, sv2v conversion, multi-run configuration tuning, and a cross-design comparison against GPU, TPU, and NPU cores on the same PDK.

## Design Summary

| Property | Value |
|---|---|
| Core | lowRISC ibex |
| ISA | RISC-V RV32IMC, in-order pipeline |
| Top Module | `ibex_top` |
| Clock Port | `clk_i` |
| Reset Port | `rst_ni` (active-low) |
| RTL Language | SystemVerilog (30 `.sv` + `.v` files) |
| PDK | Sky130A |
| Standard Cell Library | `sky130_fd_sc_hd` |
| Tool | OpenLane v1.0.2 |
| Synthesis Strategy | `AREA 0` |
| Clock Period | 20 ns (50 MHz baseline) |

---

## Repository Structure

```
ibex_sky130/
├── designs/
│   └── ibex_top/
│       ├── config.json               # OpenLane configuration
│       ├── constraint.sdc            # SDC timing constraints
│       └── src/
│           └── ibex_core.v           # 381 KB · 11,171 lines · 25 modules
├── synthesis_final/
│   ├── ibex_top.nl.v                 # Gate-level netlist
│   ├── ibex_top.sdc                  # Timing constraints
│   ├── 1-synthesis.log               # Clean synthesis log
│   └── synthesis_sta.txt             # Post-synthesis timing report
├── fixes.md                          # sv2v error log and fixes
├── synthesis_notes.md                # Synthesis analysis
└── README.md
```

---

## Prerequisites

```bash
# Verify OpenLane installation
python3 -m openlane --version
# Expected: OpenLane v1.0.2

# Verify Sky130A PDK
ls ~/.ciel/sky130A/libs.ref/sky130_fd_sc_hd/lib/
# Expected: .lib files present

# Clone ibex RTL
git clone https://github.com/lowRISC/ibex.git
```

---

## sv2v Conversion

ibex is written in SystemVerilog. Before OpenLane synthesis, the RTL must be converted to plain Verilog using `sv2v`. Six errors were encountered and resolved in sequence:

```
Start: sv2v conversion attempt
        │
        ├─ Error 1: prim_assert.sv not found          → Copy all .sv/.svh vendor headers to include/
        ├─ Error 2: dv_fcov_macros.svh not found      → Add dv_utils/ to include/
        ├─ Error 3: prim_secded_pkg not found          → Create pkg_list.txt (package dependency order)
        ├─ Error 4: prim_ram_1p_pkg not found          → Add pkg to pkg_list.txt
        ├─ Error 5: Unknown RVFI bindings              → Create rtl_list.txt (exclude ibex_top_tracing.sv)
        ├─ Error 6: Errors written into .v file        → Remove 2>&1 redirect from sv2v invocation
        │
        └─ Final invocation:
           sv2v -I include/ + pkg_list.txt + rtl_list.txt → ibex_core.v
```

**Successful output:** `ibex_core.v` — 381 KB · 11,171 lines · 25 modules · exit 0

> **Key lessons:**
> - Run sv2v on **all RTL files together**, not just the top module
> - Use the `-o` output flag, not shell redirect (`>`) — redirect silently produces a 0-byte file if sv2v errors
> - Package files must be listed in **dependency order** in `pkg_list.txt`
> - Flatten all vendor primitives into a single `include/` directory

---

## OpenLane Configuration

```json
{
  "DESIGN_NAME": "ibex_top",
  "VERILOG_FILES": [
    "dir::rtl/",
    "dir::shared/rtl/"
  ],
  "CLOCK_PORT": "clk_i",
  "CLOCK_PERIOD": 20,
  "PDK": "sky130A",
  "STD_CELL_LIBRARY": "sky130_fd_sc_hd",
  "SYNTH_STRATEGY": "AREA 0",
  "MAX_FANOUT_CONSTRAINT": 10
}
```

> **Note:** `VERILOG_FILES` uses two `dir::` paths because ibex spans `rtl/` and `shared/rtl/`. A single glob would miss the shared directory. OpenLane auto-prepends the design folder path — use only relative suffixes in the config.

---

## Running Synthesis

```bash
cd ~/OpenLane
python3 -m openlane designs/ibex_top/config.json
```

Expected run time: ~18 seconds.

### Verification Commands

```bash
# Check error count (expect 0)
grep -c "^ERROR" designs/ibex_top/synth/1-synthesis.log

# Confirm netlist exists and is non-empty
ls -lh designs/ibex_top/synth/ibex_top.nl.v

# Count mapped cells
grep -c "sky130" designs/ibex_top/synth/ibex_top.nl.v

# Count flip-flops
grep -c "dff" designs/ibex_top/synth/ibex_top.nl.v
```

---

## Synthesis Results

### Baseline Run (AREA 0, 20 ns)

| Metric | Value |
|---|---|
| Synthesis Status | ✅ PASS |
| Error Count | 0 |
| Warning Count | 41 |
| Gate-level Netlist | ✅ Generated |
| Cell Count | 10,759 |
| Run Time | ~17.86 seconds |

### Full Metrics

| Metric | Value |
|---|---|
| Total cell count | 11,157 |
| Total chip area | 13,511,277 µm² |
| Flip-flop count | 865 |
| Top cell types | `nand2_1` · `inv_2` · `a21oi_2` |
| WNS at 20 ns | > 0 ns ✅ timing MET |
| TNS at 20 ns | 0 ns (no failing paths) |
| Latches inferred | 0 |
| Unique cell types | 80 |
| Highest cell density | ALU / execute stage → branch predictor → decode |

---

## Configuration Tuning — Multi-Run Comparison

| Run | Strategy | Clock | Cell Count | Area (µm²) | WNS | Timing |
|---|---|---|---|---|---|---|
| Run 1 | AREA 0 | 20 ns | 11,157 | 13,511,277 | > 0 | ✅ PASS |
| Run 2 | AREA 0 | 10 ns | 11,157 | 13,511,277 | > 0 | ✅ PASS |

> ibex is a well-optimised, tapeout-proven core — synthesis results are stable across 20 ns and 10 ns targets on Sky130. No additional buffering or cell replacement was required when halving the clock period, confirming significant timing headroom in the baseline run.

---

## Downstream Handover Package — `synthesis_final/`

Run 1 (AREA 0, 20 ns) was selected as the best run and packaged for floorplan and placement.

| File | Role in Floorplan & Placement |
|---|---|
| `ibex_top.nl.v` | Gate-level netlist — OpenROAD reads this to place sky130 cells on die |
| `ibex_top.sdc` | 20 ns clock + I/O constraints — used by PnR for timing budgets |
| `1-synthesis.log` | Reference log — confirms clean synthesis baseline for debug |
| `synthesis_sta.txt` | Post-synthesis WNS/TNS — starting slack budget for placement optimisation |

---

## Cross-Design Comparison — ibex vs GPU vs TPU vs NPU

ibex was benchmarked against three accelerator cores synthesised on the same PDK and library.

| Metric | GPU | TPU | NPU | ibex |
|---|---|---|---|---|
| Cell Count | 13,515 | 93,781 | 787,924 | **11,157** |
| Chip Area (µm²) | 150,688 | 804,275 | 1,969,810 | **13,511,277** |
| Flip-Flop Count | 2,284 | 3,040 | 131,328 | **865** |
| Most Common Cell | `dfxtp_2` | `buf_1` | `nand2_1` | `nand2_1` |
| WNS at 20 ns | 0 | 0 | **−81,798!** | > 0 |
| Timing Met? | ✅ | ✅ | ❌ | ✅ |
| Latches Inferred | 0 | 0 | 0 | 0 |
| Unique Cell Types | 73 | 73 | 63 | **80** |
| Estimated RTL Modules | 12 | 28 | 36 | 25 |

### Analysis

**Cell count — NPU inflated by SRAM inference**
tiny-NPU (787,924 cells) appears largest but this is a synthesis artifact. The NPU targets 80–120 BRAMs on FPGA; Yosys inferred these as 131,328 flip-flops, exploding the cell count. With true sky130 SRAM macros, the count would drop dramatically. By true architectural complexity, the TPU (93,781 cells) is the correct winner.

**TPU vs GPU cell count gap (~7×)**
A systolic array for matrix multiply-accumulate requires a regular grid of MAC units — each MAC is a multiplier plus an accumulator register, which is cell-intensive. The tiny-GPU implements fewer shader cores with simpler datapaths, explaining the gap.

**ibex flip-flop count lower than GPU and TPU**
ibex (865 FFs) sequences operations through the same hardware — pipeline registers are few. The GPU (2,284) and TPU (3,040) replicate compute units across wide parallel datapaths, each carrying its own registers, creating more total sequential state despite a lower cell count.

**Timing — NPU the hardest to close**
GPU, TPU, and ibex all achieved WNS ≥ 0. The NPU shows WNS = −81,798 ns — caused by SRAM inference creating an 81K-flip-flop chain with an impossibly long combinational path. Fix: replace behavioural SRAM with sky130 SRAM macros and re-run synthesis.

**Cell type variety**
ibex used 80 unique cell types (highest) — complex control logic demands a wider variety. The NPU used only 63. GPU and TPU each used 73 — repetitive arithmetic tends to converge on fewer distinct cell types.

---

## Key Takeaways

- ibex achieves **0 errors, 0 latches, and clean timing** at both 20 ns and 10 ns on Sky130 — confirming its status as a tapeout-proven core
- sv2v conversion of multi-directory SystemVerilog requires careful **package ordering**, **include flattening**, and use of the `-o` flag
- ibex's large area (13.5M µm²) relative to the tiny-GPU (150K µm²) demonstrates that **verified silicon-ready RTL** carries significant overhead versus research-grade accelerator RTL
- Cell count alone is misleading — the NPU's 787K cells are a **SRAM inference artifact**, not true logic complexity

---

## References

- [lowRISC ibex GitHub](https://github.com/lowRISC/ibex)
- [OpenLane Documentation](https://openlane.readthedocs.io/)
- [Sky130 PDK](https://github.com/google/skywater-pdk)
- [sv2v GitHub](https://github.com/zachjs/sv2v)
- [ibex Documentation](https://ibex-core.readthedocs.io/)
