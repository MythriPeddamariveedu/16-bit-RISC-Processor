# 16-bit Single-Cycle RISC Processor with 2-Stage IF/ID Pipeline

**Verilog HDL • RTL Design • Xilinx Vivado 2019.1**

---

## Overview

This project implements a **16-bit Single-Cycle RISC Processor** in **Verilog HDL** with a **2-stage Instruction Fetch / Instruction Decode (IF/ID) pipeline**. The design follows a modular RTL architecture and was verified using simulation, synthesis, implementation, timing analysis, resource utilization, and power analysis in **Xilinx Vivado 2019.1**.

The processor demonstrates the complete RTL design flow while maintaining clean, synthesizable, and reusable Verilog modules, and exposes dedicated debug output ports (`pc_out`, `instruction_out`, `alu_result_out`, `write_back_out`, `zero_out`, `carry_out_flag`) for FPGA-level observation.

---

## Instruction Set Architecture (ISA)

### Instruction Format (16-bit fixed width)

Confirmed directly from `instruction_decoder.v`:

```
 15    12 11     8 7      4 3      0
+--------+--------+--------+--------+
| Opcode |   Rd   |  Rs1   | Rs2/Imm|
+--------+--------+--------+--------+
  4 bits   4 bits   4 bits   4 bits
```

```verilog
assign opcode  = instruction[15:12];
assign rd      = instruction[11:8];
assign rs1     = instruction[7:4];
assign rs2_imm = instruction[3:0];
```

For **STORE**, the `Rs2/Imm` field is reused: it supplies both the immediate offset (via `sign_extend`) for effective-address calculation, and the register holding the value written to memory.

### Opcode Table

*Matches `control_unit.v` exactly.*

| Opcode | Mnemonic | Type   | Operation                    | alu_sel |
|:------:|----------|--------|-------------------------------|:-------:|
| `0000` | ADD      | R-type | `Rd = Rs1 + Rs2`               | `000`   |
| `0001` | SUB      | R-type | `Rd = Rs1 - Rs2`                | `001`   |
| `0010` | AND      | R-type | `Rd = Rs1 & Rs2`                | `010`   |
| `0011` | OR       | R-type | `Rd = Rs1 \| Rs2`               | `011`   |
| `0100` | LOAD     | I-type | `Rd = Mem[Rs1 + imm]`           | `000`   |
| `0101` | STORE    | I-type | `Mem[Rs1 + imm] = Rs2`          | `000`   |
| `0110` | XOR      | R-type | `Rd = Rs1 ^ Rs2`                | `100`   |
| `0111` | NOT      | R-type | `Rd = ~Rs1` (Rs2 ignored)       | `101`   |
| `1000` | SHL      | R-type | `Rd = Rs1 << 1`                 | `110`   |
| `1001` | SHR      | R-type | `Rd = Rs1 >> 1`                 | `111`   |

The `alu_sel` encoding directly mirrors the `SEL` opcode used in the standalone 8-bit ALU project, reused here as the Execute-stage core.

### Register File Specification

- **16 general-purpose registers** (4-bit `rd`/`rs1`/`rs2_imm` address fields → 2⁴ = 16 registers)
- **16-bit width** per register

### Data Memory Specification

- **256 × 16-bit** data memory
- Addressed using the **lower 8 bits** of the ALU result (`alu_result[7:0]`)

---

## Processor Datapath

<p align="center">
  <img src="architecture/Datapath Architecture.png" alt="Datapath Architecture" width="700">
</p>

**Figure 1.** Datapath architecture of the proposed 16-bit Single-Cycle RISC Processor with IF/ID Pipeline.

---

## RTL Architecture

<p align="center">
  <img src="architecture/rtl_schematic.png" alt="RTL Schematic" width="700">
</p>

**Figure 2.** RTL schematic generated after synthesis in Xilinx Vivado.

---

## Features

- 16-bit Single-Cycle RISC Architecture
- 2-Stage IF/ID Pipeline
- Modular RTL Design (13 independent submodules)
- Dedicated debug output ports for FPGA-level observation
- Hierarchical Verilog Implementation
- Synthesizable RTL
- Functional Verification
- Timing Analysis
- Resource Utilization Analysis
- Power Analysis

---

## Processor Organization

The processor consists of the following RTL modules:

- `program_counter` — Program Counter
- `pc_adder` — PC Increment Logic
- `instruction_memory` — Instruction Memory
- `if_id_pipeline` — IF/ID Pipeline Register
- `instruction_decoder` — Instruction Decoder
- `control_unit` — Control Unit
- `register_file` — Register File
- `sign_extend` — Sign Extension Unit
- `mux2x1` — ALU Operand Multiplexer
- `alu_top` — 16-bit ALU
- `flag_register` — Flag Register (Zero, Carry, Negative, Overflow)
- `data_memory` — Data Memory
- Write Back Multiplexer (inline `assign` in `risc_top`)

---

## Pipeline Architecture

The processor integrates a **2-stage IF/ID pipeline**.

### Stage 1 – Instruction Fetch (IF)
- Program Counter Update
- Instruction Fetch
- Instruction Memory Access
- Latch into IF/ID Pipeline Register

### Stage 2 – Instruction Decode (ID) → Execute → Memory → Write Back
- Instruction Decode
- Register File Read
- Control Signal Generation
- ALU Execution
- Data Memory Access (LOAD/STORE)
- Write Back to Register File

Because fetch is registered separately from decode/execute/memory/write-back, each instruction has a **2-cycle latency** (fetched in cycle *N*, completed in cycle *N+1*), while a new instruction can still be fetched every cycle — giving steady-state throughput of 1 instruction/cycle. No hazard detection or forwarding is implemented yet (see Future Enhancements).

---

## Control Signal Truth Table

*Matches `control_unit.v` exactly.*

| Instruction | reg_write | alu_src | mem_read | mem_write | mem_to_reg | alu_sel |
|-------------|:---------:|:-------:|:--------:|:---------:|:----------:|:-------:|
| ADD         | 1         | 0       | 0        | 0         | 0          | `000`   |
| SUB         | 1         | 0       | 0        | 0         | 0          | `001`   |
| AND         | 1         | 0       | 0        | 0         | 0          | `010`   |
| OR          | 1         | 0       | 0        | 0         | 0          | `011`   |
| LOAD        | 1         | 1       | 1        | 0         | 1          | `000`   |
| STORE       | 0         | 1       | 0        | 1         | 0          | `000`   |
| XOR         | 1         | 0       | 0        | 0         | 0          | `100`   |
| NOT         | 1         | 0       | 0        | 0         | 0          | `101`   |
| SHL         | 1         | 0       | 0        | 0         | 0          | `110`   |
| SHR         | 1         | 0       | 0        | 0         | 0          | `111`   |
| *(invalid opcode)* | 0  | 0       | 0        | 0         | 0          | `000`   |

---

## Supported Operations

### Arithmetic
- ADD, SUB

### Logical
- AND, OR, XOR, NOT

### Shift
- Logical Shift Left, Logical Shift Right

### Memory
- LOAD, STORE

### Register Operations
- Register Read, Register Write Back

---

## Execution Cycle Count

| Instruction Type | Latency (cycles) | Notes |
|-------------------|:-----------------:|-------|
| R-type (ADD/SUB/AND/OR/XOR/NOT/SHL/SHR) | 2 | 1 cycle fetch (IF), 1 cycle decode+execute+writeback (ID/EX/WB) |
| I-type (LOAD)      | 2 | Address computed and memory read complete in the ID/EX/MEM/WB cycle |
| I-type (STORE)     | 2 | Address computed and memory write complete in the ID/EX/MEM cycle |

All instruction types share the same 2-cycle latency due to the fixed IF/ID pipeline boundary; steady-state throughput remains 1 instruction/cycle once the pipeline is full.

---

## Functional Verification

### Processor Waveform

<p align="center">
  <img src="simulation_results/processor_waveform.png" alt="Processor Waveform" width="800">
</p>

**Figure 3.** Processor execution showing PC progression, instruction fetch, IF/ID pipeline operation, instruction decoding, register updates, and control signal generation.

### ALU Verification

<p align="center">
  <img src="simulation_results/alu_waveform.png" alt="ALU Waveform" width="800">
</p>

**Figure 4.** Functional verification of arithmetic, logical, and shift operations performed by the 16-bit ALU.

---

## Implementation Results

### Resource Utilization

<p align="center">
  <img src="simulation_results/Resource Utilization.png" alt="Resource Utilization" width="700">
</p>

### Timing Analysis

<p align="center">
  <img src="simulation_results/Timing_summary.png" alt="Timing Summary" width="700">
</p>

### Power Analysis

<p align="center">
  <img src="simulation_results/power_analysis.png" alt="Power Analysis" width="700">
</p>

---

## Development Flow

- RTL Design
- Functional Simulation
- RTL Verification
- Synthesis
- Design Implementation
- Timing Analysis
- Resource Utilization Analysis
- Power Analysis

---

## Repository Structure

```
16-bit-Single-Cycle-RISC-Processor-with-IF-ID-Pipeline/
│
├── RTL/                          # All Verilog source modules
├── Testbench                     # risc_top_tb testbench file
├── memory                        # Instruction/data memory init file
│
├── architecture/
│   ├── Datapath Architecture.png
│   └── rtl_schematic.png
│
├── simulation_results/
│   ├── processor_waveform.png
│   ├── alu_waveform.png
│   ├── Resource Utilization.png
│   ├── Timing_summary.png
│   └── power_analysis.png
│
└── README.md
```

---

## Tools Used

- Verilog HDL
- Xilinx Vivado 2019.1
- XSim Simulator

---

## Future Enhancements

- Hazard Detection Unit
- Data Forwarding
- Branch Prediction
- Multi-stage Pipeline
- Cache Memory
- Interrupt Handling

---

## Author

**Mythri Peddamariveedu**

Bachelor of Technology
Electronics and Communication Engineering
