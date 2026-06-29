# Counter-Frezzing
Counter-Frezzing
# Counter-Freezing Attack Reproduction Project README
## Project Name
Counter-Freezing Attack on Cute-Lock-Str: Implementation Reproduction of Counter-Freezing Attack for Time-Based Multi-Key Sequential Logic Locking Scheme Cute-Lock

## 1. Project Overview
This project reproduces the **Counter-Freezing Attack** proposed in the paper *MLC-Cute-Lock: A Sequential Logic Locking Scheme Resisting Counter-Freezing Attack*. The full attack pipeline targets the gate-level time-variant multi-key encryption scheme Cute-Lock-Str, including three core steps: counter localization on gate-level netlist, gate-level netlist tampering to freeze counter states, sequential SAT solving for secret key extraction, and functional simulation validation of decrypted circuits.

### Core Attack Principle
Cute-Lock-Str relies on a global cyclic counter to switch multiple groups of time-dependent keys. Attackers tamper with the sequential logic of the counter to lock the counter into a single fixed steady state. This degrades the original time-domain multi-key architecture into a single-key circuit, which can be rapidly cracked by the KC2 sequential SAT solver.

### 1.1 Supported Targets
- Valid only for original Cute-Lock-Str (gate-level sequential multi-key logic locking).
- MLC-Cute-Lock three-layer collaborative defense architecture can fully neutralize this attack; the attack cannot succeed on MLC-Cute-Lock circuits.

### 1.2 Environment Dependencies
| Tool | Purpose | Version Requirement |
|------|---------|--------------------|
| ABC | Logic synthesis, netlist tampering optimization, sequential circuit modeling | ABC 1.01 or higher |
| Icarus Verilog (iverilog) | Circuit functional simulation, decryption waveform verification | v10 or higher |
| KC2 Sequential SAT Solver | Key cracking on frozen netlists | Open-source KC2 toolchain from the reference paper |
| Python3 | Netlist parsing, counter localization scripts, automated batch experiments | Python 3.8+ |
| Synopsys DC (Optional) | Hardware overhead evaluation and comparison | 2019.03+ |
| Benchmark Circuits | Test cases | ISCAS'89, ITC'99 standard sequential benchmark suites |

### 1.3 Project Directory Structure
```
CounterFreeze-Attack/
├── docs/                    # Original paper, attack schematic diagrams
│   ├── MLC-CuteLock.pdf
│   ├── counter_freeze_schematic.png
│   └── cute_mux_tree.png
├── netlist/                 # Original encrypted netlists & tampered frozen netlists
│   ├── original/            # Cute-Lock-Str encrypted benchmark netlists
│   └── frozen/              # Output netlists after counter freezing tampering
├── scripts/
│   ├── locate_counter.py    # Topology analysis script to locate counter modules (high fanin/fanout & timing loop detection)
│   ├── freeze_counter.py    # Generate frozen netlists via three tampering modes
│   ├── sat_attack_run.py    # Invoke KC2 to launch sequential SAT attacks
│   └── sim_verify.py        # Icarus Verilog simulation to compare original & decrypted circuit outputs
├── abc_scripts/             # ABC synthesis & optimization command scripts
│   ├── strash_rewrite.abc
│   └── netlist_export.abc
├── benchmark/               # Raw unencrypted ISCAS89/ITC99 benchmark circuits
├── result/                  # Experiment outputs: SAT logs, simulation waveforms, overhead statistics
│   ├── sat_log/
│   ├── wave_vcd/
│   └── overhead_report.csv
└── README.md                # This documentation
```

## 2. Three Core Implementation Stages of Counter-Freezing Attack
### Stage 1: Core Counter Localization (locate_counter.py)
#### Implementation Logic
1. Parse gate-level netlists encrypted by Cute-Lock-Str and build circuit topological graphs.
2. Traverse all D flip-flops (DFFs), filter nodes with high fan-in/fan-out and existing sequential feedback loops — these nodes are the core key scheduling counter (Counter_B).
3. Output counter bit width, state transition logic, control ports of MUX tree, and sequential feedback paths.
4. Generate localization report and mark target registers, input pins and feedback wires for subsequent tampering.

#### Output File
`result/counter_location_report.csv`: Contains circuit name, counter bit width, recommended frozen state, and register node IDs.

### Stage 2: Circuit Tampering & Counter State Freezing (freeze_counter.py)
Three hardware tampering modes defined in the original paper are provided to lock the counter to a fixed target state (e.g., 2-bit counter locked to `00`):
1. **Rewrite counter state update excitation**
   Replace all next-state input logic of the counter with target fixed logic levels, cutting off the auto-increment counting logic.
2. **Force constant logic levels at register inputs**
   Insert constant logic 0/1 at DFF input ports; multiplexers select fixed static signals and ignore dynamic clock counting signals.
3. **Cut sequential feedback loops**
   Disconnect feedback wires from counter outputs to inputs, and directly connect fixed steady-state signals to form a self-loop (e.g., `00 → 00`).

#### Post-Tampering Processing
Invoke ABC tool to execute `strash` and `rewrite` commands for structural optimization to eliminate redundant logic, then export the modified frozen benchmark netlist to `netlist/frozen/`.

### Stage 3: Architecture Degradation & Sequential SAT Key Extraction
1. After counter freezing, the cyclic multi-key MUX tree permanently activates only one group of keys, degrading the multi-key sequential architecture into a single-key encryption circuit.
2. Call KC2 sequential SAT solver to construct temporal miter on frozen netlists and extract Discriminating Input Sequences (DIS).
3. Iteratively prune invalid keys via I/O constraints until convergence and output complete valid unlocking keys.
4. Record solving runtime, iteration count, key space size, and attack result (Success/Failed).

### Stage 4: Simulation Validation (sim_verify.py)
1. Inject keys recovered by SAT solver into frozen encrypted circuits.
2. Simulate both original unencrypted circuits and decrypted frozen circuits separately.
3. Compare output VCD waveform files. If all output waveforms are identical, the attack is fully validated.

## 3. Quick Reproduction Guide
### Step 1: Environment Setup
1. Install dependent tools: Icarus Verilog, ABC, Python3.
2. Compile and deploy KC2 sequential SAT solver, add executable path to system environment variables.
3. Place Cute-Lock-Str encrypted benchmark netlists under `netlist/original/`.

### Step 2: Automatic Counter Localization
```bash
python3 scripts/locate_counter.py --circuit s298 --counter_min_bit 2
```
Parameter Description:
- `--circuit`: Specify target ISCAS/ITC benchmark circuit name
- `--counter_min_bit`: Minimum bit width threshold for counter filtering

### Step 3: Execute Counter Freezing Tampering
```bash
# Mode 1: Force constant input on registers, freeze counter to state 00
python3 scripts/freeze_counter.py --circuit s298 --tamp_mode force_const --target_state 00
# Mode 2: Cut sequential feedback loop for freezing
# python3 scripts/freeze_counter.py --circuit s298 --tamp_mode cut_feedback --target_state 00
# Mode 3: Rewrite next-state excitation logic for freezing
# python3 scripts/freeze_counter.py --circuit s298 --tamp_mode rewrite_next --target_state 00
```
Output File: `netlist/frozen/s298_frozen_00.bench`

### Step 4: Netlist Optimization via ABC
```bash
abc -f abc_scripts/strash_rewrite.abc netlist/frozen/s298_frozen_00.bench
```

### Step 5: Launch KC2 Sequential SAT Attack for Key Recovery
```bash
python3 scripts/sat_attack_run.py --frozen_net netlist/frozen/s298_frozen_00.bench --output_log result/sat_log/s298_log.txt
```
Log content includes total solving time, DIS count, iteration rounds, and final recovered secret keys.

### Step 6: Verilog Simulation to Verify Decryption Correctness
```bash
python3 scripts/sim_verify.py --original_net benchmark/s298.bench --locked_net netlist/frozen/s298_frozen_00.bench --key "00,10,00,10"
```
Output waveform file: `result/wave_vcd/s298_decrypt.vcd`, viewable via GTKWave for output comparison.

## 4. Experimental Output Metrics
All attack logs are aggregated into `result/attack_summary.csv` with the following metrics:
1. Name of benchmark circuit
2. Counter bit width, theoretical key space size, key bit width per group
3. Frozen target state (00 / 100 / 111, etc.)
4. Attack result: ✓ Success / × Failed
5. Total sequential SAT solving time (seconds)
6. SAT iteration count, number of Discriminating Input Sequences (DIS)

### Attack Success Criterion
When recovered keys are loaded, all output waveforms of encrypted circuits fully match the original unencrypted circuit without any error signals.

## 5. Limitations & Boundary Conditions
1. **Only effective for Cute-Lock-Str**
   MLC-Cute-Lock integrates an independent authentication layer Counter_A. Before authentication passes, the temporal obfuscation layer Counter_B remains held in reset state. Attackers cannot locate or tamper with functional counting logic, rendering Counter-Freezing Attack completely invalid.
2. Matches the adversary threat model in the original paper
   Attackers are assumed to have full gate-level netlist, physical chip oracle, scan chain access, and timing analysis capability. Without these preconditions, the attack cannot be executed.
3. Tampering requires editable gate-level netlist
   If counter modules are protected by metal layer shielding or hardware camouflage during manufacturing, counter localization difficulty rises sharply.
4. Circuit scale affects SAT solving latency
   Small-scale circuits (s298 / b03) finish solving within milliseconds; large-scale benchmarks such as b14 (245 DFFs) may take over 80 seconds.

## 6. Troubleshooting Common Issues
1. **Counter localization fails**
   - Verify input netlist is encrypted by Cute-Lock-Str; non-multi-key circuits contain no cyclic time-based counters.
   - Adjust `counter_min_bit` parameter to filter trivial general registers.
2. **SAT solver outputs no valid keys, attack failed**
   - Check whether frozen target state matches the original key binding states of the circuit.
   - Re-run ABC strash & rewrite commands to eliminate redundant netlist logic.
3. **Simulation waveforms mismatch between original & decrypted circuit**
   - SAT solver may return local suboptimal false keys; increase SAT unrolling depth and re-run the attack.
   - Confirm tampering operations do not damage original combinational signal paths.

## 7. Citation & Copyright
This project reproduces the attack scheme proposed in the referenced academic paper. For academic research usage, please cite the original work:
Weizheng Wang, Zhou Xie, Tieqiao Liu. MLC-Cute-Lock: A Sequential Logic Locking Scheme Resisting Counter-Freezing Attack.

Benchmark circuits: Public ISCAS'89 / ITC'99 sequential benchmark suites.
Tool Sources: Open-source ABC logic synthesis tool, open-source KC2 sequential deobfuscation solver.

# Appendix: Official Download Links for All Dependent Tools
Add this section to the end of the English README you generated before.

## 1. ABC Logic Synthesis Tool (Gate-level Netlist Processing)
- Official GitHub Repository (Berkeley ABC):
  https://github.com/berkeley-abc/abc
- Quick one-click build script:
  https://getabc.sh/
- Official Documentation:
  https://people.eecs.berkeley.edu/~alanmi/abc/abcman.pdf

## 2. KC2 Sequential SAT Deobfuscation Solver
- Original Paper Release Repository (University of Texas at Austin):
  https://github.com/yierjin/KC2
- Paper Reference (DATE 2019):
  https://ieeexplore.ieee.org/document/8714820
- Supplementary Benchmark & Tool Archive:
  https://github.com/kavehshamsi/KC2_Attack

## 3. Icarus Verilog (Verilog Simulation Engine)
- Official GitHub Source Code:
  https://github.com/steveicarus/iverilog
- Official Homepage & Installation Guide:
  https://steveicarus.github.io/iverilog/
- Windows Prebuilt Binary (bundled with GTKWave):
  http://bleyer.org/icarus/
- FTP Source Archive:
  ftp://ftp.icarus.com/pub/eda/verilog/

## 4. GTKWave (VCD Waveform Viewer)
- Official GitHub (New Main Repo):
  https://gtkwave.github.io/gtkwave/
- SourceForge Legacy Download Page:
  https://gtkwave.sourceforge.net/
- Official User Manual PDF:
  https://gtkwave.sourceforge.net/gtkwave.pdf
- macOS Homebrew Tap:
  https://github.com/randomplum/homebrew-gtkwave

## 5. ISCAS'89 & ITC'99 Standard Sequential Benchmark Circuits
### ISCAS'89
- TTU Verilog Source Mirror:
  https://pld.ttu.ee/~maksim/benchmarks/iscas89/verilog/
- USC SPORTlab Benchmark Database (.bench format):
  https://sportlab.usc.edu/~msabrishami/benchmarks.html
- NCSU Collaborative Benchmark Lab:
  http://www.cbl.ncsu.edu/

### ITC'99
- Official UT Austin ITC'99 Homepage:
  https://www.cerc.utexas.edu/itc99-benchmarks/bench.html
- Full RTL & EDIF Archive:
  https://www.cerc.utexas.edu/itc99-benchmarks/bendoc1.html

## 6. Python 3 (Script Automation)
- Official Download Page:
  https://www.python.org/downloads/
- Official Documentation:
  https://docs.python.org/3/

## 7. Synopsys Design Compiler (Optional, Hardware Overhead Evaluation)
- Official Vendor Resource Page (Commercial Tool):
  https://www.synopsys.com/tools/implementation/logic-synthesis/design-compiler.html

## 8. Reference Paper of MLC-Cute-Lock
- Original PDF (MLC-Cute-Lock Counter-Freezing Attack):
  Attached `docs/MLC-CuteLock.pdf` in project folder
- Cute-Lock Original Paper (DATE 2025):
  https://ieeexplore.ieee.org/document/10887621
