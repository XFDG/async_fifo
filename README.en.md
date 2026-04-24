# Asynchronous Dual-Clock FIFO Reproduction

This repository reproduces and documents the open-source Verilog project [`dpretet/async_fifo`](https://github.com/dpretet/async_fifo). It is prepared for the ICS6204/MCE5913 Final Project, whose goal is to select an open-source RTL project, analyze it technically, and present it in a 5-8 minute talk.

## Project Purpose

An asynchronous FIFO transfers data safely between two independent clock domains. The write side runs on `wclk`, the read side runs on `rclk`, and the design combines dual-clock memory, binary/Gray-code pointers, two-flop synchronizers, and full/empty detection logic.

This project is a good final-project target because:

- It is small enough to understand but deep enough for CDC and micro-architecture analysis.
- Its module boundaries are clear, which makes block diagrams and walkthroughs practical.
- It includes simulation, lint, and synthesis scripts for future verification work.
- It matches the Async FIFO topic recommended in the course project document.

## Added Deliverables

- `guide.md`: detailed reproduction guide, task analysis, architecture notes, report outline, and GitHub collaboration workflow.
- `README.zh-CN.md`: Chinese project README.
- `README.en.md`: English project README.
- `.gitattributes`: keeps common source and documentation files on LF line endings to reduce Windows/WSL noise.

## Repository Layout

```text
.
├── rtl/                         # RTL sources
│   ├── async_fifo.v             # Basic asynchronous FIFO top level
│   ├── async_bidir_fifo.v       # Bidirectional FIFO with internal RAM
│   ├── async_bidir_ramif_fifo.v # Bidirectional FIFO with external RAM interface
│   ├── fifomem.v                # Basic FIFO memory
│   ├── fifomem_dp.v             # Dual-port memory
│   ├── rptr_empty.v             # Read pointer and empty/almost-empty logic
│   ├── wptr_full.v              # Write pointer and full/almost-full logic
│   ├── sync_r2w.v               # Read pointer synchronizer into write domain
│   ├── sync_w2r.v               # Write pointer synchronizer into read domain
│   └── sync_ptr.v               # Generic pointer synchronizer
├── sim/                         # SVUT/Icarus Verilog testbench
├── syn/                         # Yosys synthesis scripts and cell libraries
├── doc/                         # Upstream specification and test plan
├── flow.sh                      # Unified lint/sim/syn entry point
└── guide.md                     # Detailed guide for this reproduction
```

## Architecture Summary

The basic top level, `rtl/async_fifo.v`, is built from five main blocks:

- `fifomem`: stores FIFO payload data.
- `wptr_full`: maintains the write binary pointer and write Gray pointer, and generates `wfull` and `awfull`.
- `rptr_empty`: maintains the read binary pointer and read Gray pointer, and generates `rempty` and `arempty`.
- `sync_r2w`: synchronizes the read Gray pointer into the write clock domain for full detection.
- `sync_w2r`: synchronizes the write Gray pointer into the read clock domain for empty detection.

The main design idea is to use binary pointers for local memory addressing and Gray-code pointers for clock-domain crossing. Gray code reduces CDC risk because only one bit changes between adjacent pointer values.

## Main Interface

`async_fifo` parameters:

| Parameter | Description |
| --- | --- |
| `DSIZE` | Data width, default 8 bits |
| `ASIZE` | Address width; FIFO depth is `2**ASIZE` |
| `FALLTHROUGH` | `"TRUE"` enables first-word fall-through for lower read-side latency |

`async_fifo` ports:

| Port | Direction | Description |
| --- | --- | --- |
| `wclk` / `wrst_n` | input | Write clock and active-low write reset |
| `winc` / `wdata` | input | Write enable and write data |
| `wfull` / `awfull` | output | Full and almost-full flags in the write domain |
| `rclk` / `rrst_n` | input | Read clock and active-low read reset |
| `rinc` / `rdata` | input/output | Read enable and read data |
| `rempty` / `arempty` | output | Empty and almost-empty flags in the read domain |

Usage constraints:

- Assert `winc` only when `wfull == 0`.
- Assert `rinc` only when `rempty == 0`.
- Reset both sides before normal use; the upstream specification recommends resetting both domains together.

## Optional Verification Commands

The current task was performed in WSL and explicitly skipped testing, so no simulation, lint, or GUI waveform checks were run. When the toolchain is available, use:

```bash
./flow.sh lint
./flow.sh sim
./flow.sh syn
```

Required tools:

- Lint: Verilator
- Simulation: Icarus Verilog + SVUT
- Synthesis: Yosys

## Suggested Report Structure

The final technical report should cover:

1. Project overview: what an asynchronous FIFO does and where it is used.
2. Build & simulation: optional commands and pass criteria if the tools are available.
3. Module diagram: top level, pointer logic, synchronizers, and memory.
4. Interface description: write/read sides, resets, flags, and handshake rules.
5. Micro-architecture: Gray code, CDC synchronizers, and full/empty detection.
6. Code walk-through: responsibilities of the most important files.
7. Verification: optional summary of the existing SVUT tests.
8. Findings: 3-5 concrete observations.
9. References: GitHub repository, Clifford Cummings FIFO paper, and course documents.

See [`guide.md`](guide.md) for the full reproduction and collaboration guide.

## License and Attribution

This project is based on Damien Pretet's [`dpretet/async_fifo`](https://github.com/dpretet/async_fifo). Keep the original copyright and license notices. The top-level `LICENSE` is MIT, while some RTL file headers contain Apache-2.0 notices; preserve the relevant notices when reusing or modifying files.

