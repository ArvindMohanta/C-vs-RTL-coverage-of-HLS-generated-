# hls-coverage-gap

**Does C testbench coverage survive high-level synthesis?**
A case study on an 8-tap FIR filter with AMD Vitis HLS, Synopsys VCS and Cadence JasperGold.

> CS5291: C-Based VLSI Design, Project 1 (LLM-Based HLS Testbench Generation and Coverage Evaluation), IIT Guwahati.

In an HLS flow, the C testbench is reused automatically to verify the generated RTL (C/RTL co-simulation).
This project measures whether a testbench that looks complete at C level also exercises the generated
hardware, finds out why it does not, and shows what closes each kind of gap.

---

## Key results

**100% C branch coverage gave only 43% RTL toggle coverage.** Every uncovered RTL item falls into one of three kinds, each with its own fix:

| Kind of gap | Shows up as | Cause | Fix | Evidence |
|---|---|---|---|---|
| Data gaps | toggle coverage | narrow test values (all coefficients = 1) | better stimulus | toggle 43 → 72% (xsim), 50 → 89% (VCS) |
| Untested control scenarios | FSM transitions | reset during a computation, which a C call cannot express | new RTL test type: reset injection | FSM 60 → 100% |
| Dead HLS template code | line, branch, condition | stall logic generated for an interface that can never stall | exclude, backed by a formal proof | JasperGold: all items unreachable |

### C coverage (gcov)

| Testbench | Tests | Line | Branch |
|---|---|---|---|
| tb_v0 | 1 | 100% | 66.7% |
| tb_v1 | 3 | 100% | 100% |
| tb_v2 | 304 | 100% | 100% |

### RTL coverage of the `fir` module

| Metric | xsim, tb_v1 | xsim, tb_v2 | VCS, v1 | VCS, v2 | VCS, v2 + gaps | VCS, v2 + reset injection |
|---|---|---|---|---|---|---|
| Statement / line | 96.9 | 96.9 | 96.94 | 96.94 | 96.94 | 96.94 |
| Branch | 89.6 | 89.6 | 87.69 | 87.69 | 87.69 | 87.69 |
| Condition | 81.8 | 81.8 | 76.22 | 76.22 | 76.76 | 77.30 |
| FSM | n/a | n/a | 60.00 | 60.00 | 60.00 | **100.00** |
| Toggle | 43.2 | **72.4** | 50.31 | **89.22** | 89.45 | 89.60 |

xsim and VCS count toggle and condition coverage slightly differently; the pattern is the same in both.
Gap and reset columns are merged with the v2 run.

### Why line and branch coverage never moved

```verilog
assign ap_block_pp0_stage0_subdone = ~(1'b1 == 1'b1);   // always 0
```

Every uncovered branch in the generated `fir.v` depends on this pipeline-stall signal. The arrays use simple
memory ports, which can never stall, so the stall-handling code Vitis generates from its template is dead.
One more uncovered `else` (line 304) follows two complementary conditions and can never execute.

### Formal proofs (JasperGold)

| Proof | Result |
|---|---|
| 1 | both stall signals **proven** always 0; stall branches, statements 174–175 and line 304 **unreachable** |
| 2 | line-216 branch and the remaining condition rows (lines 191, 274, 282, 292, 301) **unreachable** |
| Sanity | normal behaviour (loop exit, `ap_done`, start in idle) **covered**, so the proofs are not vacuous |

With the proven-dead items excluded: **line 100%, branch 100%, FSM 100%** (with reset injection), toggle 89.6%.

---

## Design under test

```cpp
int fir(int x[8], int c[8]) {
    int acc = 0;
    for (int i = 0; i < 8; i++) acc += x[i] * c[i];
    if (acc > 1000)  acc = 1000;    // saturate high
    if (acc < -1000) acc = -1000;   // saturate low
    return acc;
}
```

Vitis HLS 2025.1, `xc7z020clg400-1`, 10 ns clock, no pragmas. Vitis built a 5-stage pipelined loop with one
shared 32×32 multiplier: 14-cycle latency, interval 15, 3 DSP, 338 FF, 286 LUT, 0 BRAM, and a 4-state FSM.

## Testbenches and test types

| Name | What it does |
|---|---|
| `tb_v0` | 1 normal test, coefficients all 1 |
| `tb_v1` | 3 directed tests: normal, saturate high, saturate low; coefficients all 1 |
| `tb_v2` | 304 tests: boundaries, walking ones on every bit of `x` and `c`, 200 fixed-seed random signed tests, independent 64-bit golden model, overflow-safe range (±16383) |
| `+GAP=n` | RTL only: 0..n random idle cycles between calls |
| `+RST=n` | RTL only: every n-th call interrupted by a reset at a random cycle, then re-run |

---

## Repository layout

```
hls-coverage-gap/
├── laptop_kit/            Windows: Vitis HLS + xsim + gcov
│   ├── fir.cpp            design (HLS top function)
│   ├── tb_v0.cpp, tb_v1.cpp, tb_v2.cpp
│   ├── run_all.bat        C coverage, synthesis, co-simulation, xsim coverage
│   ├── summarize.py       collects everything into results/summary.csv
│   └── README.txt
├── server_kit/            Linux: Synopsys VCS + Cadence JasperGold
│   ├── fir.cpp, tb_v1.cpp, tb_v2.cpp
│   ├── fir_logger.cpp     records every fir() call of a C testbench
│   ├── tb_fir.v           Verilog testbench replaying the calls (+VEC, +GAP, +RST)
│   ├── cm_hier.cfg        restricts coverage to the design instance
│   ├── run_vcs.sh         v1 and v2 with VCS coverage + URG reports
│   ├── run_extra.sh       random gaps, reset injection, both JasperGold proofs
│   ├── fir_props.sv, jg_run.tcl     JasperGold proof 1
│   ├── fir_props2.sv, jg_run2.tcl   JasperGold proof 2
│   └── README.txt
├── CODE_INDEX.md          what every file does
└── README.md
```

The HLS-generated Verilog is not committed. Generate it with `run_all.bat`, then copy
`laptop_kit/fir_prj_v2/sol1/syn/verilog/*.v` into `server_kit/rtl/`.

## Requirements

| Flow | Tools |
|---|---|
| Laptop (Windows) | AMD Vitis 2025.1 (HLS + xsim + xcrg); MSYS2 g++ with gcov; Python 3 with `gcovr` |
| Server (Linux) | Synopsys VCS + URG (tested U-2023.03); Cadence JasperGold (tested 2023.03p002); g++ |

VCS and JasperGold need commercial licences (academic licences are often available through the institute).

## How to run

### Laptop (Windows)

Edit the tool paths at the top of `run_all.bat` if your installs differ, then:

```bat
cd laptop_kit
run_all.bat
type results\summary.csv
```

`run_all.bat` puts MSYS2 first on `PATH` (the Vitis environment otherwise breaks the MSYS2 g++), checks the
tools, runs C coverage for all three testbenches, then for tb_v1 and tb_v2 runs Vitis csim, synthesis and
C/RTL co-simulation, re-runs the generated xsim simulation with `-cc_type sbct`, and builds the xcrg report.

### Server (Linux)

```bash
# load your site's Synopsys and Cadence environment first
cd server_kit
bash run_vcs.sh      # v1 and v2: vectors, VCS compile/sim with coverage, URG reports
bash run_extra.sh    # random gaps, reset injection, JasperGold proofs 1 and 2

cat results/urg_rst/hierarchy.txt          # coverage after reset injection
grep -A8 '^transitions' results/urg_rst/modinfo.txt
cat results/jg_summary.txt results/jg_summary2.txt
```

## Practical notes

- **Measure the design only.** The overall coverage summary includes the testbench Vitis generates for
  co-simulation and is not comparable with C coverage.
- **Report branch, not only line coverage.** A one-line `if` counts as covered even when its action never runs.
- **Clear old coverage databases** (`*.vdb`) before a fresh VCS run; URG merges earlier runs otherwise
  (`run_vcs.sh` does this).
- **VCS needs `-full64`** on systems with only 64-bit libraries (the 32-bit binary fails with a missing `libelf.so.1`).
- **JasperGold:** run with `< /dev/null` and delete `jgproject/` after an interrupted run.

## Limitations

- One small kernel with simple memory interfaces; designs with streaming (FIFO) interfaces can stall, so their
  stall logic would be reachable.
- Integer data types only; fixed-point (`ap_fixed`) adds rounding and saturation hardware.
- The VCS flow uses its own Verilog testbench (replaying the same transactions, timing matched to co-simulation)
  rather than the Vitis-generated one.

## Future work

- **LLM-generated testbenches:** compare manual, random, LLM basic, LLM corner-case and LLM-with-RTL-coverage-feedback
  testbenches on toggle and condition coverage.
- More MachSuite kernels and HLS pragma variants (pipeline, unroll, array partition).
- Mutation testing: does higher RTL coverage also catch more injected bugs?

## References

1. K. Xu et al., "HLSTester: Efficient testing of behavioral discrepancies with LLMs for high-level synthesis," arXiv:2504.14641, 2025.
2. "Shift-left high-level synthesis verification via knowledge-augmented LLM agent" (HLS-LeVeri), arXiv:2606.17128, 2026.
3. Z. Zhang et al., "LLM4DV: Using large language models for hardware test stimuli generation," arXiv:2310.04535, 2023.
4. S. Abi-Karam and C. Hao, "HLS-Eval: A benchmark and framework for evaluating LLMs on HLS design tasks," arXiv:2504.12268, 2025.
5. Y. Herklotz et al., "An empirical study of the reliability of high-level synthesis tools," IEEE FCCM, 2021.
6. B. Reagen et al., "MachSuite: Benchmarks for accelerator design and customized architectures," IEEE IISWC, 2014.
7. AMD UG1399 (Vitis HLS User Guide) and UG900 (Vivado Logic Simulation).

## Authors

[Name 1], [Name 2], [Name 3] — CS5291, IIT Guwahati.
