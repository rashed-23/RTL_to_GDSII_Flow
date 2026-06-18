# CVA6 RISC-V Processor — Multi-Frequency Synthesis on ASAP7 7nm

> **PD Tasks Team — Week 4** | Prepared by: Md. Rashedul Islam

RTL-to-synthesis flow for the [CVA6 (ariane)](https://github.com/openhwgroup/cva6) RISC-V processor targeting the **ASAP7 7nm predictive PDK** using **OpenROAD Flow Scripts (ORFS)**. Three clock frequencies are exercised in parallel: **500 MHz**, **600 MHz**, and **700 MHz**.

> ⚠️ **Status:** Synthesis executed at all three frequencies. Downstream output files (synthesised netlist, timing/area reports) were **not generated** due to Yosys warnings. Root cause analysis and remediation steps are documented below.

---

## Design Summary

| Property | Value |
|---|---|
| Core | CVA6 RISC-V (ariane) |
| ISA | RV64GC, 6-stage out-of-order pipeline |
| Top Module | `ariane` |
| Clock Port | `clk_i` |
| PDK | ASAP7 7nm Predictive — RVT standard cells |
| Tool Flow | OpenROAD Flow Scripts (ORFS) — Yosys + OpenROAD |
| RTL Source | Verilog (pre-shipped with ORFS under `designs/src/ariane/`) |
| Target Frequencies | 500 MHz · 600 MHz · 700 MHz |
| Die Area | 600 × 600 µm (≈ 6–8× larger than ibex) |

---

## ibex vs CVA6 — Complexity at a Glance

| | ibex | CVA6 |
|---|---|---|
| Pipeline | 2-stage, in-order | 6-stage, out-of-order |
| Die Area | 300 × 300 µm | 600 × 600 µm |
| Standard Cells (est.) | 20,583 | >> 20,583 |
| Flip-Flops (est.) | 1,937 | 10,000 – 20,000+ |
| Complexity | Low | Very High |
| OOO Structures | None | ROB, issue queue, scoreboard, MMU |

---

## Repository Structure

```
.
├── designs/
│   └── asap7/
│       └── cva6_freq/
│           ├── config_500.mk          # ORFS config — 500 MHz
│           ├── config_600.mk          # ORFS config — 600 MHz
│           ├── config_700.mk          # ORFS config — 700 MHz
│           ├── constraint_500mhz.sdc  # SDC — 2000 ps clock, 200 ps I/O delay
│           ├── constraint_600mhz.sdc  # SDC — 1667 ps clock, 167 ps I/O delay
│           └── constraint_700mhz.sdc  # SDC — 1429 ps clock, 143 ps I/O delay
├── src/
│   └── cva6/                          # Verilog RTL files (from ORFS ariane source)
└── README.md
```

---

## Prerequisites

### System Requirements
- Ubuntu 22.04 LTS (or compatible)
- Docker Engine (`sudo apt-get install -y docker.io`)
- At least 32 GB RAM recommended (CVA6 is significantly larger than ibex)

### Docker Image
```bash
sudo systemctl start docker
sudo usermod -aG docker $USER
newgrp docker

docker pull openroad/orfs
```

> **Why Docker?** OpenROAD requires SWIG ≥ 4.3, but Ubuntu 22.04 ships SWIG 4.0.2. The ORFS Docker image bundles all dependencies pre-built.

---

## Clone & Setup

```bash
cd ~
git clone https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts
cd OpenROAD-flow-scripts

# Create directories for CVA6 config files and source
mkdir -p ~/orfs/flow/designs/src/cva6
mkdir -p ~/orfs/flow/designs/asap7/cva6_freq
```

> **Note:** A pre-converted Verilog version of CVA6 (ariane) already exists inside the ORFS repository under `orfs/flow/designs/src/ariane/`. No sv2v conversion is required for this flow. Copy or symlink these files into `designs/src/cva6/`.

---

## SDC Constraint Files

ASAP7 uses **picoseconds (ps)** as its time unit. I/O delays are set to **10% of the respective clock period** to provide realistic boundary conditions for timing closure analysis.

| Frequency | Clock Period | Input Delay | Output Delay |
|---|---|---|---|
| 500 MHz | 2000 ps | 200 ps | 200 ps |
| 600 MHz | 1667 ps | 167 ps | 167 ps |
| 700 MHz | 1429 ps | 143 ps | 143 ps |

```bash
# 500 MHz
cat > ~/orfs/flow/designs/asap7/cva6_freq/constraint_500mhz.sdc << 'EOF'
create_clock -name clk_i -period 2000 [get_ports clk_i]
set_input_delay  -clock clk_i 200 [all_inputs]
set_output_delay -clock clk_i 200 [all_outputs]
EOF

# 600 MHz
cat > ~/orfs/flow/designs/asap7/cva6_freq/constraint_600mhz.sdc << 'EOF'
create_clock -name clk_i -period 1667 [get_ports clk_i]
set_input_delay  -clock clk_i 167 [all_inputs]
set_output_delay -clock clk_i 167 [all_outputs]
EOF

# 700 MHz
cat > ~/orfs/flow/designs/asap7/cva6_freq/constraint_700mhz.sdc << 'EOF'
create_clock -name clk_i -period 1429 [get_ports clk_i]
set_input_delay  -clock clk_i 143 [all_inputs]
set_output_delay -clock clk_i 143 [all_outputs]
EOF
```

---

## ORFS Configuration Files (.mk)

CVA6 requires a larger die area than ibex (600 × 600 µm vs 300 × 300 µm) due to its significantly greater logic complexity.

```makefile
# config_500.mk
export PLATFORM               = asap7
export DESIGN_NICKNAME        = cva6_500
export DESIGN_NAME            = ariane
export VERILOG_FILES          = $(sort $(wildcard $(DESIGN_HOME)/src/cva6/*.v))
export SDC_FILE               = ./designs/asap7/cva6_freq/constraint_500mhz.sdc
export ABC_CLOCK_PERIOD_IN_PS = 2000
export DIE_AREA               = 0 0 600 600
export CORE_AREA              = 2 2 598 598
export MAX_FANOUT             = 8
export TNS_END_PERCENT        = 100
```

Repeat for `config_600.mk` (`DESIGN_NICKNAME=cva6_600`, `ABC_CLOCK_PERIOD_IN_PS=1667`) and `config_700.mk` (`DESIGN_NICKNAME=cva6_700`, `ABC_CLOCK_PERIOD_IN_PS=1429`).

---

## Running the Flow

### Step 1 — Launch the Docker Container

```bash
cd ~/orfs
docker run -it -v $(pwd)/flow:/OpenROAD-flow-scripts/flow openroad/orfs
```

### Step 2 — Run Synthesis

Inside the Docker container:

```bash
cd /OpenROAD-flow-scripts/flow

make DESIGN_CONFIG=./designs/asap7/cva6_freq/config_500.mk synth
make DESIGN_CONFIG=./designs/asap7/cva6_freq/config_600.mk synth
make DESIGN_CONFIG=./designs/asap7/cva6_freq/config_700.mk synth
```

### Step 3 — Run Floorplan

```bash
make DESIGN_CONFIG=./designs/asap7/cva6_freq/config_500.mk floorplan
make DESIGN_CONFIG=./designs/asap7/cva6_freq/config_600.mk floorplan
make DESIGN_CONFIG=./designs/asap7/cva6_freq/config_700.mk floorplan
```

### Step 4 — Extract Metrics (once output files are generated)

```bash
for FREQ in 500 600 700; do
  DESIGN="cva6_${FREQ}"
  echo "=== CVA6 — ${FREQ} MHz ==="

  echo -n "  Cells:      "
  grep 'floorplan__design__instance__count":' \
    ./logs/asap7/${DESIGN}/base/2_1_floorplan.json | awk -F': ' '{print $2}' | tr -d ','

  echo -n "  Area (um2): "
  grep 'floorplan__design__instance__area":' \
    ./logs/asap7/${DESIGN}/base/2_1_floorplan.json | awk -F': ' '{print $2}' | tr -d ','

  echo -n "  WNS (ps):   "
  grep 'floorplan__timing__setup__ws' \
    ./logs/asap7/${DESIGN}/base/2_1_floorplan.json | awk -F': ' '{print $2}' | tr -d ','

  echo -n "  Fmax (Hz):  "
  grep 'floorplan__timing__fmax":' \
    ./logs/asap7/${DESIGN}/base/2_1_floorplan.json | awk -F': ' '{print $2}' | tr -d ','

  echo -n "  Flip-flops: "
  grep -c 'DFF' ./results/asap7/${DESIGN}/base/1_2_yosys.v

  echo -n "  Top-5 cells: "
  grep -oP '\w+_ASAP7_75t_R' ./results/asap7/${DESIGN}/base/1_2_yosys.v \
    | sort | uniq -c | sort -rn | head -5
done
```

---

## Synthesis Run Status

| Frequency | Synthesis Executed | Output Files Generated |
|---|---|---|
| 500 MHz | ✅ Yes | ❌ Not generated |
| 600 MHz | ✅ Yes | ❌ Not generated |
| 700 MHz | ✅ Yes | ❌ Not generated |

Metrics extraction was **not possible** because the required output files were absent:

| Metric | Source File | Status |
|---|---|---|
| Total Cell Count | `2_1_floorplan.json` | Not generated |
| Chip Area (µm²) | `2_1_floorplan.json` | Not generated |
| Worst Negative Slack (WNS) | `2_1_floorplan.json` | Not generated |
| Maximum Frequency (Fmax) | `2_1_floorplan.json` | Not generated |
| Flip-Flop Count | `1_2_yosys.v` | Not generated |

---

## Warning Analysis & Root Cause

> **Critical note:** Yosys warnings are non-fatal by default — synthesis continues but may silently produce an incomplete or incorrect netlist. The absence of output files indicates that a hard error or post-synthesis check failure followed the warning stream.

### Warning Category 1 — Continuous Assignment

```
Warning: reg 'reg_port_o' is assigned in a continuous assignment at
  /designs/src/ariane/ariane.sv2v.v:165929.10-165929.31

Warning: reg 'tlb_update_o' is assigned in a continuous assignment at
  /designs/src/ariane/ariane.sv2v.v:166002.10-166006.48
  ...
```

**Root cause:** After sv2v conversion, signals that were originally procedural register outputs in SystemVerilog are being picked up by Yosys as unexpected combinational feedback into register nets.

**Fix:** Restructure the affected assignments into clocked `always` blocks, or post-process the sv2v output to separate register outputs from combinational logic before passing to Yosys.

---

### Warning Category 2 — Missing Module / Hierarchy Error

```
ERROR: Module '\gmblk1_1__ram' is not part of the design.
make: *** [Makefile:271: results/asap7/cva6_700/base/1_1_yosys_canonicalize.rtlil] Error 1
```

**Root cause:** CVA6 contains SRAM-like structures (register file, ROB, issue queue, scoreboards) that reference memory primitive modules. The `VERILOG_FILES` glob (`*.v`) does not include these memory model files, leaving the hierarchy incomplete.

**Fix:** Explicitly enumerate all dependent modules in `VERILOG_FILES`, including memory primitives. Check CVA6's memory subsystem includes and add them to the `.mk` config:

```makefile
export VERILOG_FILES = \
  $(sort $(wildcard $(DESIGN_HOME)/src/cva6/*.v)) \
  $(sort $(wildcard $(DESIGN_HOME)/src/cva6/mem_primitives/*.v))
```

---

## Remediation Checklist

- [ ] Identify all `continuous assignment` warnings in Yosys log and locate the affected signals in the sv2v output
- [ ] Restructure or patch those assignments so they appear inside clocked `always` blocks
- [ ] Enumerate memory primitive files and add to `VERILOG_FILES` in all three `.mk` configs
- [ ] Re-run synthesis and confirm `1_2_yosys.v` and `2_1_floorplan.json` are generated
- [ ] Extract and record metrics using the extraction script above

---

## Key Output Files (once synthesis completes cleanly)

| File | Contents |
|---|---|
| `logs/asap7/cva6_{FREQ}/base/1_1_yosys.log` | Yosys synthesis log — cell statistics |
| `results/asap7/cva6_{FREQ}/base/1_2_yosys.v` | Synthesised gate-level netlist |
| `logs/asap7/cva6_{FREQ}/base/1_synth.log` | OpenROAD post-synthesis area report |
| `logs/asap7/cva6_{FREQ}/base/2_1_floorplan.json` | Timing, power, and area metrics (JSON) |

---

## Architectural Notes

### Why CVA6 Is Harder to Synthesise Than ibex

CVA6 is a 6-stage out-of-order processor with significantly more complex hardware than the 2-stage in-order ibex:

- **Branch prediction** — predictor tables and history buffers
- **Issue queue** — dynamic scheduling with comparators, wake-up logic, dependency tracking
- **Reorder Buffer (ROB)** — large storage and complex control logic
- **Scoreboards** — register dependency tracking
- **MMU / TLBs** — address translation structures
- **Register renaming** — physical register file management

These structures produce long combinational paths, large fanout networks, and increased wire delays. Even with six pipeline stages, CVA6's critical path can be longer than ibex's at the same frequency target.

### Flip-Flop Estimate

| Source | Estimated Count |
|---|---|
| Integer register file | ~2,048 |
| Pipeline stage registers | ~1,500 |
| Reorder Buffer (ROB) | ~3,000 – 5,000 |
| Issue queue / scheduler | ~1,500 – 3,000 |
| CSR state | ~500 |
| TLBs and MMU | ~1,000 |
| Branch predictor state | ~500 |
| **Total estimate** | **~10,000 – 20,000+** |

Compare with ibex: **1,937 flip-flops**. CVA6 requires roughly 5–10× more sequential storage.

---

## References

- [CVA6 GitHub (openhwgroup)](https://github.com/openhwgroup/cva6)
- [CVA6 Requirements Specification](https://cva6.readthedocs.io/en/latest/02_cva6_requirements/cva6_requirements_specification.html)
- [OpenROAD Flow Scripts](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts)
- [ASAP7 PDK](http://asap.asu.edu/asap/)
