# Pipelined MIPS32 Processor in Verilog

## Overview

This project implements a 32-bit pipelined MIPS-like processor using Verilog HDL. The processor supports arithmetic, logical, memory-access, conditional-branch, and halt instructions.

The design uses a **five-stage instruction pipeline** to overlap instruction execution and improve processor throughput. Two clock signals, `clk1` and `clk2`, are used to control the different pipeline stages.

A key modification in this implementation fixes a race condition caused by multiple `always` blocks driving the `TAKEN_BRANCH` register.

## Features

- 32-bit datapath and instruction-processing registers.
- Five-stage instruction pipeline.
- 32 general-purpose registers.
- 1024-word memory array.
- Arithmetic and logical operations.
- Load and store instructions.
- Conditional branching using `BEQZ` and `BNEQZ`.
- Immediate arithmetic instructions.
- Halt instruction.
- Branch-handling fix to eliminate multiple procedural drivers of `TAKEN_BRANCH`.

## Processor Architecture

The processor consists of five pipeline stages:

| Stage | Clock | Description |
|---|---|---|
| Instruction Fetch (IF) | `clk1` | Fetches instructions from memory and updates the program counter. |
| Instruction Decode (ID) | `clk2` | Decodes the instruction, reads source registers, generates the immediate value, and identifies the instruction type. |
| Execution (EX) | `clk1` | Performs arithmetic, logical, address-generation, and branch-condition operations. |
| Memory Access (MEM) | `clk2` | Reads data from memory for load instructions and writes data for store instructions. |
| Write Back (WB) | `clk1` | Writes arithmetic results and loaded data into the register file. |

The alternating clock signals separate successive pipeline stages in the implementation.

### Pipeline Registers

The following registers transfer instructions and intermediate results between stages:

- `IF_ID_IR`, `IF_ID_NPC`: Instruction-fetch/decode pipeline registers.
- `ID_EX_IR`, `ID_EX_NPC`, `ID_EX_A`, `ID_EX_B`, `ID_EX_Imm`: Instruction-decode/execution registers.
- `EX_MEM_IR`, `EX_MEM_ALUOut`, `EX_MEM_B`, `EX_MEM_cond`: Execution/memory registers.
- `MEM_WB_IR`, `MEM_WB_ALUOut`, `MEM_WB_LMD`: Memory/write-back registers.

The `ID_EX_type`, `EX_MEM_type`, and `MEM_WB_type` registers identify the instruction class at the corresponding pipeline stages.

## Instruction Set

The processor supports the following instructions.

| Instruction | Opcode | Operation |
|---|---|---|
| `ADD` | `000000` | `rd = rs + rt` |
| `SUB` | `000001` | `rd = rs - rt` |
| `AND` | `000010` | `rd = rs & rt` |
| `OR` | `000011` | `rd = rs \| rt` |
| `SLT` | `000100` | `rd = (rs < rt)` |
| `MUL` | `000101` | `rd = rs * rt` |
| `LW` | `001000` | `rt = Mem[rs + immediate]` |
| `SW` | `001001` | `Mem[rs + immediate] = rt` |
| `ADDI` | `001010` | `rt = rs + immediate` |
| `SUBI` | `001011` | `rt = rs - immediate` |
| `SLTI` | `001100` | `rt = (rs < immediate)` |
| `BNEQZ` | `001101` | Branch if the tested register is nonzero |
| `BEQZ` | `001110` | Branch if the tested register is zero |
| `HLT` | `111111` | Halt processor execution |

**Note:** The instruction formats and opcodes follow the supplied Verilog implementation. This is a MIPS-like instruction set and should not be assumed to be fully compatible with standard MIPS machine code.

## Instruction Classification

Instructions are classified into internal types during the decode stage.

| Internal Type | Encoding | Instructions |
|---|---|---|
| `RR_ALU` | `000` | `ADD`, `SUB`, `AND`, `OR`, `SLT`, `MUL` |
| `RM_ALU` | `001` | `ADDI`, `SUBI`, `SLTI` |
| `LOAD` | `010` | `LW` |
| `STORE` | `011` | `SW` |
| `BRANCH` | `100` | `BEQZ`, `BNEQZ` |
| `HALT` | `101` | `HLT` |

## Branch Handling

Conditional branches are evaluated in the execution stage.

The branch target is calculated using:

`Branch Target = ID_EX_NPC + ID_EX_Imm`

The branch condition is recorded in `EX_MEM_cond`, and the branch instruction and target information are passed to the following pipeline stage.

During instruction fetch, the processor checks whether the instruction in the execution/memory pipeline register represents a taken `BEQZ` or `BNEQZ` branch. If a branch is taken, the program counter is updated to the branch target.

### Race Condition Fix

In the original implementation, `TAKEN_BRANCH` was assigned in two separate `always @(posedge clk1)` blocks:

- The instruction-fetch stage set the flag when a branch was taken.
- The execution stage reset the flag.

Because both blocks attempted to drive the same register on the same clock edge, the behavior could depend on simulator scheduling.

In this implementation, **`TAKEN_BRANCH` is driven only by the instruction-fetch stage**. The flag is asserted when a branch is taken and cleared in that same block when the branch is not taken.

This removes the multiple procedural drivers and makes the branch-flag assignment deterministic with respect to the intended control flow.

## Memory and Register File

### Register File

The processor declares 32 registers, each 32 bits wide:

`reg [31:0] Reg [0:31];`

The instruction fields select the source and destination registers:

- `IR[25:21]`: First source register (`rs`).
- `IR[20:16]`: Second source register (`rt`).
- `IR[15:11]`: Destination register (`rd`) for register-register ALU instructions.

The decode stage explicitly substitutes zero when a source-register index is zero. However, the implementation does not globally prevent writes to `Reg[0]`, so register-zero immutability is not guaranteed.

### Memory

The memory array is declared as:

`reg [31:0] Mem [0:1023];`

It contains 1024 words of 32 bits each and is used for instruction fetches and data-memory accesses.

The supplied module does not initialize the memory contents internally. A testbench or an initialization mechanism must load the program and any required data before simulation.

## Control Signals

| Signal | Purpose |
|---|---|
| `PC` | Holds the current instruction address. |
| `TAKEN_BRANCH` | Indicates that the fetch stage is processing a taken branch. |
| `HALTED` | Indicates that the processor has halted. |
| `EX_MEM_cond` | Stores the evaluated branch condition. |
| `ID_EX_Imm` | Holds the sign-extended immediate value. |
| `EX_MEM_ALUOut` | Holds the ALU result, effective address, or branch target. |
| `MEM_WB_LMD` | Holds data read from memory for a load instruction. |

## Simulation

### Requirements

- Verilog simulator, such as Icarus Verilog or Vivado Simulator.
- A testbench that generates the two clock signals.
- A program loaded into the `Mem` array.
- Register and memory initialization as required by the test program.

### Example Testbench Clock Generation

The following is an illustrative clock-generation fragment for a testbench:

```verilog
initial begin
    clk1 = 0;
    clk2 = 0;
    forever begin
        #5 clk1 = 1;
        #5 clk1 = 0;
        #5 clk2 = 1;
        #5 clk2 = 0;
    end
end
```

This generates alternating clock phases. The exact timing should be checked against the timing assumptions of the processor and the testbench.

### Compile with Icarus Verilog

Assuming the module is saved as `pipe_MIPS32.v` and the testbench is saved as `tb_pipe_MIPS32.v`:

```bash
iverilog -g2012 -o mips_sim pipe_MIPS32.v tb_pipe_MIPS32.v
vvp mips_sim
```

These commands compile and run the simulation. A suitable testbench must be supplied separately.

For waveform inspection, add `$dumpfile` and `$dumpvars` statements to the testbench and open the generated VCD file with a waveform viewer such as GTKWave.

## Limitations and Future Improvements

The current implementation provides the basic structure of a pipelined processor. The following areas should be reviewed before treating it as a complete processor implementation:

- **Pipeline hazards:** Data hazards and control hazards are not comprehensively handled.
- **Branch handling:** The branch-flag fix removes multiple procedural drivers but does not implement a complete branch-flush or pipeline-stall mechanism.
- **Register zero:** Writes to `Reg[0]` are not globally prohibited.
- **Memory initialization:** The program and data memory must be initialized externally.
- **Halt behavior:** `HALTED` is set during write-back, but the design does not explicitly initialize it in the module.
- **Signed comparisons:** `SLT` and `SLTI` use ordinary relational expressions on unsigned-declared registers, so signed comparison semantics are not explicitly implemented.
- **Pipeline timing:** The design uses delayed nonblocking assignments (`#2`), which are simulation delays rather than a synthesizable timing-control mechanism for hardware.
- **Verification:** The processor should be tested with arithmetic, memory, branch, and halt test programs.

## Conclusion

This project demonstrates the organization of a five-stage pipelined MIPS-like processor using Verilog HDL. It includes instruction decoding, arithmetic and logical execution, memory access, conditional branching, and register write-back.

The branch-flag modification addresses a race condition caused by multiple procedural drivers of `TAKEN_BRANCH`, providing a cleaner basis for further simulation, debugging, and processor development.

