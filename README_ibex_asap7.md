# ibex RISC-V Core — Multi-Frequency Synthesis on ASAP7 7nm

> **PD Tasks Team — Week 4** | Prepared by: Md. Rashedul Islam

RTL-to-synthesis flow for the [lowRISC ibex](https://github.com/lowRISC/ibex) RISC-V core targeting the **ASAP7 7nm predictive PDK** using **OpenROAD Flow Scripts (ORFS)**. Three clock frequencies are exercised in parallel: **500 MHz**, **600 MHz**, and **700 MHz**.

---

## Design Summary

| Property | Value |
|---|---|
| Core | lowRISC ibex |
| ISA | RISC-V RV32IMC, 2-stage in-order pipeline |
| Top Module | `ibex_core` |
| Clock Port | `clk_i` |
| PDK | ASAP7 7nm Predictive — RVT standard cells |
| Tool Flow | OpenROAD Flow Scripts (ORFS) — Yosys + OpenROAD |
| RTL Source | Verilog (pre-converted via sv2v from lowRISC ibex SystemVerilog) |
| Target Frequencies | 500 MHz · 600 MHz · 700 MHz |

---

## Repository Structure

```
.
├── designs/
│   └── asap7/
│       └── ibex_freq/
│           ├── config_500.mk          # ORFS config — 500 MHz
│           ├── config_600.mk          # ORFS config — 600 MHz
│           ├── config_700.mk          # ORFS config — 700 MHz
│           ├── constraint_500mhz.sdc  # SDC — 2000 ps clock period
│           ├── constraint_600mhz.sdc  # SDC — 1667 ps clock period
│           └── constraint_700mhz.sdc  # SDC — 1429 ps clock period
├── src/
│   └── ibex/                          # Verilog RTL files
└── README.md
```

---

## Prerequisites

### System Requirements
- Ubuntu 22.04 LTS (or compatible)
- Docker Engine (`sudo apt-get install -y docker.io`)
- At least 16 GB RAM recommended for full ORFS runs

### Docker Image
```bash
sudo systemctl start docker
sudo usermod -aG docker $USER
newgrp docker

docker pull openroad/orfs
```

> **Why Docker?** OpenROAD requires SWIG ≥ 4.3, but Ubuntu 22.04 ships SWIG 4.0.2. The ORFS Docker image bundles all dependencies pre-built, avoiding a multi-hour local compilation.

> **Why ORFS and not OpenLane 2?** OpenLane 2 (`python3 -m openlane`) supports only the `sky130` and `gf180mcu` PDKs via volare. ASAP7 is not available through volare. ORFS ships ASAP7 as a first-class platform and is the correct tool for 7nm predictive flows.

---

## Clone & Setup

```bash
# Clone the repository with pre-converted ibex Verilog
cd ~
git clone https://github.com/inderjit303/OpenROAD-7nm-PD-ibex
cd OpenROAD-7nm-PD-ibex

# Verilog files are already present under:
# orfs/flow/designs/src/ibex/
# No sv2v conversion needed — files are pre-converted.
```

---

## SDC Constraint Files

ASAP7 uses **picoseconds (ps)** as its time unit (declared as `time_unit : "1ps"` in Liberty files). All clock periods and reported WNS/TNS values are in ps.

| Frequency | Period (ps) | SDC Command |
|---|---|---|
| 500 MHz | 2000 ps | `create_clock -name clk_i -period 2000 [get_ports clk_i]` |
| 600 MHz | 1667 ps | `create_clock -name clk_i -period 1667 [get_ports clk_i]` |
| 700 MHz | 1429 ps | `create_clock -name clk_i -period 1429 [get_ports clk_i]` |

To regenerate the SDC files manually:
```bash
mkdir -p ~/orfs/flow/designs/asap7/ibex_freq

cat > ~/orfs/flow/designs/asap7/ibex_freq/constraint_500mhz.sdc << 'EOF'
create_clock -name clk_i -period 2000 [get_ports clk_i]
EOF

cat > ~/orfs/flow/designs/asap7/ibex_freq/constraint_600mhz.sdc << 'EOF'
create_clock -name clk_i -period 1667 [get_ports clk_i]
EOF

cat > ~/orfs/flow/designs/asap7/ibex_freq/constraint_700mhz.sdc << 'EOF'
create_clock -name clk_i -period 1429 [get_ports clk_i]
EOF
```

---

## ORFS Configuration Files (.mk)

ORFS uses Makefile (`.mk`) format — not JSON as in OpenLane 2. The `ABC_CLOCK_PERIOD_IN_PS` variable sets the synthesis optimisation target per run.

```makefile
# config_500.mk
export PLATFORM             = asap7
export DESIGN_NICKNAME      = ibex_500
export DESIGN_NAME          = ibex_core
export VERILOG_FILES        = $(sort $(wildcard $(DESIGN_HOME)/src/ibex/*.v))
export SDC_FILE             = ./designs/asap7/ibex_freq/constraint_500mhz.sdc
export ABC_CLOCK_PERIOD_IN_PS = 2000
export DIE_AREA             = 0 0 300 300
export CORE_AREA            = 2 2 298 298
export MAX_FANOUT           = 8
export TNS_END_PERCENT      = 100
```

Repeat for `config_600.mk` (`DESIGN_NICKNAME=ibex_600`, `ABC_CLOCK_PERIOD_IN_PS=1667`) and `config_700.mk` (`DESIGN_NICKNAME=ibex_700`, `ABC_CLOCK_PERIOD_IN_PS=1429`).

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

make DESIGN_CONFIG=./designs/asap7/ibex_freq/config_500.mk synth
make DESIGN_CONFIG=./designs/asap7/ibex_freq/config_600.mk synth
make DESIGN_CONFIG=./designs/asap7/ibex_freq/config_700.mk synth
```

### Step 3 — Run Floorplan (generates timing reports)

```bash
make DESIGN_CONFIG=./designs/asap7/ibex_freq/config_500.mk floorplan
make DESIGN_CONFIG=./designs/asap7/ibex_freq/config_600.mk floorplan
make DESIGN_CONFIG=./designs/asap7/ibex_freq/config_700.mk floorplan
```

### Step 4 — Extract Metrics

```bash
for FREQ in 500 600 700; do
  DESIGN="ibex_${FREQ}"
  echo "=== ibex — ${FREQ} MHz ==="

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
done
```

---

## Synthesis Results

| Metric | 500 MHz | 600 MHz | 700 MHz |
|---|---|---|---|
| Standard Cell Count | 20,583 | 20,583 | 20,583 |
| Chip Area (µm²) | 2,398.51 | 2,398.51 | 2,398.51 |
| Flip-Flop Count | 1,937 | 1,937 | 1,937 |
| Setup WNS (ps) | **+902.82** | **+569.82** | **+331.82** |
| Setup TNS | CLEAN (0) | CLEAN (0) | CLEAN (0) |
| Timing Met? | ✅ YES | ✅ YES | ✅ YES |
| Estimated Fmax | 911 MHz | 911 MHz | 911 MHz |
| Most Common Cell | `INVx1_ASAP7_75t_R` (2,021) | same | same |
| DFF Cell Type | `DFFASRHQNx1_ASAP7_75t_R` | same | same |

> **Why are cell count and area identical across all three frequencies?**
> ORFS runs Yosys synthesis once and produces a single technology-mapped netlist. The floorplan stage then evaluates timing against each SDC constraint independently. At 500–700 MHz, ibex is well within ASAP7 capabilities — ABC reaches the same optimised solution regardless of target. Meaningful differences appear only at frequencies approaching Fmax (~911 MHz), where the tool inserts additional buffers to meet tighter constraints.

### Timing Slack Margin

| Frequency | WNS | Slack Margin |
|---|---|---|
| 500 MHz (2000 ps period) | +902.82 ps | **45%** above constraint |
| 600 MHz (1667 ps period) | +569.82 ps | **34%** above constraint |
| 700 MHz (1429 ps period) | +331.82 ps | **23%** above constraint |

### Flip-Flop Breakdown (1,937 total)

| Source | Count (approx) |
|---|---|
| Register file (32 × 32-bit) | ~1,024 |
| Pipeline stage registers (IF/ID) | ~350 |
| CSR (Control and Status Register) state | ~280 |
| Fetch buffer and prefetch control | ~180 |
| Load/store unit handshake registers | ~103 |

---

## Key Output Files

| File | Contents |
|---|---|
| `logs/asap7/ibex_{FREQ}/base/1_2_yosys.log` | Yosys synthesis log — cell statistics |
| `results/asap7/ibex_{FREQ}/base/1_2_yosys.v` | Synthesised gate-level netlist |
| `logs/asap7/ibex_{FREQ}/base/1_synth.log` | OpenROAD post-synthesis area report |
| `logs/asap7/ibex_{FREQ}/base/2_1_floorplan.json` | Timing, power, and area metrics (JSON) |

---

## Challenges & Fixes

| Challenge | Fix Applied |
|---|---|
| ASAP7 not supported by OpenLane 2 | Switched to ORFS via Docker |
| `sv2v` shell redirect (`>`) produced a silent 0-byte output | Used the `-o` output flag; verified sv2v was installed from GitHub release binary |
| Docker-created files owned by root (read-only on host) | Used `sudo rm -rf` on the host for cleanup before each fresh Docker session |
| Initial sv2v output placed in wrong directory | Copied pre-converted ibex Verilog from `designs/src/chameleon/ibex/` to `designs/src/ibex/` as working baseline |

---

## Notes on ASAP7 Time Units

ASAP7 Liberty files declare `time_unit : "1ps"`. All SDC periods, WNS, and TNS values are in **picoseconds**. To convert to nanoseconds, divide by 1000.

Example: WNS = +902.82 ps = +0.903 ns at 500 MHz.

---

## References

- [lowRISC ibex GitHub](https://github.com/lowRISC/ibex)
- [OpenROAD Flow Scripts](https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts)
- [ASAP7 PDK](http://asap.asu.edu/asap/)
- [ibex Documentation](https://ibex-core.readthedocs.io/)
