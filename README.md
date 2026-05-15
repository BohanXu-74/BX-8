# 8-Bit CPU from Scratch

> A homebrew 8-bit CPU designed from the ground up by an 8th grader, inspired by classic processors like the 6502, Z80, and 8085.

[![Demo Video](https://img.shields.io/badge/YouTube-Demo%20Video-red?logo=youtube)](https://www.youtube.com/watch?v=CR7b72kyQng)
[![Hackaday](https://img.shields.io/badge/Hackaday.io-Project-green)](https://hackaday.io/project/204995-8-bit-cpu-from-scratch)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Register File](#register-file)
- [ALU](#alu)
- [Memory System](#memory-system)
- [Stack](#stack)
- [Control Unit & Microcode](#control-unit--microcode)
- [Instruction Set](#instruction-set)
- [Project Files](#project-files)
- [How to Run](#how-to-run)
- [Build Log](#build-log)
- [YouTube Series](#youtube-series)
- [Acknowledgements](#acknowledgements)

---

## Overview

I started this project to understand how real CPUs work at the logic level. After building a 4-bit CPU in 5th grade, I studied the 8085, Z80, and 6502 and wanted to design something faster — a CPU that executes instructions in fewer cycles than those classic designs.

The CPU is designed entirely from scratch using TTL logic chips, with a ROM-based instruction decoder (microcode). All logic is simulated in [Digital](https://github.com/hneemann/Digital) by H. Neemann.

**Key design goals:**
- Fewer cycles per instruction than the Z80/8085/6502
- Custom ISA tailored to the hardware
- Full 128 KB address space (RAM + ROM)
- Real subroutine support with a hardware stack

> ⚠️ This project is still in progress. The ALU is complete and tested on real hardware. Other modules are being integrated.

---

## Architecture

```
        ┌─────────────────────────────────────────┐
        │               Data Bus (8-bit)           │
        └────┬────────┬────────┬────────┬──────────┘
             │        │        │        │
         ┌───▼──┐ ┌───▼──┐ ┌──▼───┐ ┌──▼────┐
         │  ALU │ │Regs  │ │Stack │ │Mem I/O│
         │(FPGA)│ │A-G,M │ │16x16 │ │ROM/RAM│
         └───┬──┘ └───┬──┘ └──────┘ └───────┘
             │        │
        ┌────▼────────▼────────────────────────────┐
        │           Control Unit (ROM microcode)    │
        │         33-bit micro-instruction word     │
        └──────────────────────────────────────────┘
                         │
                  ┌──────▼──────┐
                  │Program Ctr  │
                  │  (16-bit)   │
                  └─────────────┘
```

**Key specs:**

| Parameter | Value |
|---|---|
| Data bus width | 8 bits |
| Address bus width | 16 bits |
| Addressable memory | 128 KB (ROM + RAM) |
| General purpose registers | 4 (A, B, C, D) |
| Special purpose registers | 4+ (E, F, G, M/memory pointer + FG pair) |
| ALU operations | 8+ (ADD, SUB, ADC, SBC, AND, OR, XOR, CPL, SHR, SHL, compare) |
| Flags | Z (zero), C (carry), N (negative) |
| Stack depth | 16 entries × 16 bits wide |
| Instruction decoder | ROM-based microcode (33-bit control word) |
| Simulator | Digital by H. Neemann |

---

## Register File

The CPU has 8 addressable registers:

| Register | Role |
|---|---|
| **A** | Accumulator — primary ALU operand and result |
| **B** | General purpose / memory addressing |
| **C** | General purpose / memory addressing |
| **D** | General purpose / memory addressing |
| **E** | General purpose / memory addressing |
| **F** | Memory address (upper byte of FG pair) |
| **G** | Memory address (lower byte of FG pair) / I/O |
| **M** | Memory pointer — 2-byte address for indirect memory access |

**FG register pair:** F and G together form a 16-bit address register used for indirect jumps (`JMP [FG]`), calls (`CALL [FG]`), and memory-mapped I/O display. This is the CPU's pointer register equivalent to HL in the Z80 or the indirect register in the 6502.

**Register A** is the only register that feeds directly into the ALU as an operand. All arithmetic and logic results are stored back to A. Registers B–G are used as secondary operands and for holding memory addresses.

---

## ALU

The ALU is implemented on an FPGA and supports the following operations, selected via a 3-bit control input (S1, S2, S3):

| Operation | Description |
|---|---|
| ADD | Add register/immediate to A |
| ADC | Add with carry |
| SUB | Subtract from A |
| SBC | Subtract with borrow (carry) |
| AND | Bitwise AND with A |
| OR | Bitwise OR with A |
| XOR | Bitwise XOR with A |
| CPL | Complement (bitwise NOT) of A |
| SHR A | Logical shift right A |
| SHL A | Logical shift left A |
| CMPE | Compare equal — jumps to FG if equal |
| CMPD | Compare equal — jumps to specified address if equal |

**Flags set by ALU:** Z (zero), C (carry), N (negative)

The FPGA-based ALU sits in the center of the breadboard. Surrounding chips handle buffering, comparison logic, the A register, and 8 data bus LEDs.

---

## Memory System

The CPU uses a **split ROM/RAM** memory model:

- **ROM** — stores the program (normally output-enabled, disabled by `~ROMoe`)
- **RAM / peripheral devices** — selected when `~ROM/RAM = 1`
- **Memory-mapped I/O** — peripherals appear as addresses on the bus

Memory access control signals:
- `~OE` — output enable (read from RAM/device)
- `~WE` — write enable (write to RAM/device)
- `~ROMoe` — disable ROM output (normally low/enabled)
- `~ROM/RAM` — selects ROM (0) or RAM/other (1)

A dedicated **MOV register** (`REGin` / `~REGout`) is used as an intermediate latch during memory transfer operations.

The `TO MEM` latch controls whether the address bus receives its value from the register file or from the data bus, enabling both direct and register-indirect addressing.

---

## Stack

The hardware stack stores return addresses for subroutine calls:

- **Depth:** 16 entries
- **Width:** 16 bits per entry (split into upper and lower 8-bit halves)
- **Operations:** PUSH and POP, each with separate upper/lower byte clock signals

Control signals:
- `~PUSH/POP AU` — select push or pop for upper byte
- `CLKstack AU` — clock data in/out of upper stack
- `~PUSH/POP AD` — select push or pop for lower byte
- `CLKstack AD` — clock data in/out of lower stack

The `CALL [FG]` instruction pushes the current PC to the stack and jumps to the address in the FG register pair. `RET` pops the return address and restores the PC.

---

## Control Unit & Microcode

Instructions are decoded by a **ROM-based microcode unit**. Each instruction is broken into a sequence of **micro-instructions**, each represented as a 33-bit control word.

The micro-instruction bit assignments are:

| Bit(s) | Signal | Description |
|---|---|---|
| `0x000000001` | NOP | No operation |
| `0x000000002` | INSTRld | Load instruction into buffer |
| `0x000000004` | MINIclr | Clear the micro-step counter |
| `0x000000008` | REGadr | Load address for register selection |
| `0x000000010` | CLK | Clock the program counter |
| `0x000000020` | REG outEn | Selected register → data bus |
| `0x000000040` | REG inEn | Data bus → selected register |
| `0x000000080` | S1 | ALU operation select bit 1 |
| `0x000000100` | S2 | ALU operation select bit 2 |
| `0x000000200` | S3 | ALU operation select bit 3 |
| `0x000000400` | ADR BYTE1 | Load address byte 1 (for JMP) |
| `0x000000800` | ADR BYTE2 | Load address byte 2 (for JMP) |
| `0x000001000` | ~PUSH/POP AU | Select push or pop, upper stack byte |
| `0x000002000` | CLKstack AU | Clock upper stack |
| `0x000004000` | ~PUSH/POP AD | Select push or pop, lower stack byte |
| `0x000008000` | CLKstack AD | Clock lower stack |
| `0x000010000` | ~ROM/RAM | 0 = ROM, 1 = RAM/device |
| `0x000020000` | ~OE | Output enable (read from RAM/device) |
| `0x000040000` | ~WE | Write enable (write to RAM/device) |
| `0x000080000` | ~ROMoe | Disable ROM output |
| `0x000100000` | REGin | Load MOV register from bus |
| `0x000200000` | ~REGout | Output MOV register to bus |
| `0x000400000` | ADCen | Add with carry enable |
| `0x000800000` | SBCen | Subtract with borrow enable |
| `0x001000000` | Await | Advance clock for multi-cycle ops |
| `0x002000000` | ~JMP | Jump (load PC from address latch) |
| `0x004000000` | G I/O | Register G ↔ data bus |
| `0x008000000` | F I/O | Register F ↔ data bus |
| `0x010000000` | CLKinstr | Clock instruction mini-decoder |
| `0x020000000` | FLAG OUTen | Output flag register to bus |
| `0x080000000` | O/~I | Control direction of FG / memory I/O |
| `0x100000000` | TO MEM | Address bus source: register or data bus |
| `0x200000000` | DISPLAY FG | Route FG to address bus (when 0) |
| `0x400000000` | <JMP | Jump if selected register < A |
| `0x800000000` | =JMP | Jump if selected register == A |
| `0x1000000000` | >JMP | Jump if selected register > A |

Microcode is stored in `instr-data.hex` and loaded into the ROM at runtime.

---

## Instruction Set

The CPU supports 60+ instructions. Opcodes are 1 byte. Some instructions take additional immediate bytes (`#NUM`) or address bytes (`#ADR`).

**Notation:**
- `#NUM` — 8-bit immediate value
- `#NUMH / #NUML` — high/low byte of a 16-bit immediate
- `MEM[#NUM, #NUM]` — direct memory address (2 bytes)
- `[FG]` — indirect address from FG register pair
- `M` — memory pointer (indirect, 2-byte address)

### Control Flow

| Opcode | Instruction | Description |
|---|---|---|
| `00` | NOP | No operation |
| `02` | JMP #NUMH, #NUML | Absolute jump to 16-bit address |
| `04` | JMP [FG] | Indirect jump to address in FG |
| `05` | CALL [FG] | Call subroutine at FG; push PC to stack |
| `0C` | RET | Return from subroutine; pop PC from stack |

### Data Movement — Immediate Load

| Opcode | Instruction |
|---|---|
| `03` | MOV A, #NUM |
| `0B` | MOV B, #NUM |
| `13` | MOV C, #NUM |
| `1B` | MOV D, #NUM |
| `23` | MOV E, #NUM |
| `2B` | MOV F, #NUM |
| `33` | MOV G, #NUM |
| `3B` | MOV M, #NUM |

### Data Movement — Load from Memory

| Opcode | Instruction |
|---|---|
| `06` | MOV A, MEM[#NUM, #NUM] |
| `09` | MOV B, MEM[#NUM, #NUM] |
| `11` | MOV C, MEM[#NUM, #NUM] |
| `19` | MOV D, MEM[#NUM, #NUM] |
| `21` | MOV E, MEM[#NUM, #NUM] |
| `29` | MOV F, MEM[#NUM, #NUM] |
| `31` | MOV G, MEM[#NUM, #NUM] |

### Data Movement — Move to Flags (MOVF)

Stores the register value into the flag register.

`07` MOVF A, `0A` MOVF B, `12` MOVF C, `1A` MOVF D, `22` MOVF E, `2A` MOVF F, `32` MOVF G

### Data Movement — Register to Register (MOV Rn, Rm)

Full 8×8 register move matrix at opcodes `0x40–0x7F`:

| | A | B | C | D | E | F | G | M |
|---|---|---|---|---|---|---|---|---|
| **→ A** | `40` | `41` | `42` | `43` | `44` | `45` | `46` | `47` |
| **→ B** | `48` | `49` | `4A` | `4B` | `4C` | `4D` | `4E` | `4F` |
| **→ C** | `50` | `51` | `52` | `53` | `54` | `55` | `56` | `57` |
| **→ D** | `58` | `59` | `5A` | `5B` | `5C` | `5D` | `5E` | `5F` |
| **→ E** | `60` | `61` | `62` | `63` | `64` | `65` | `66` | `67` |
| **→ F** | `68`* | `69` | `6A` | `6B` | `6C` | `6D` | `6E` | `6F` |
| **→ G** | `70`* | `71` | `72` | `73` | `74` | `75` | `76` | `77` |
| **→ M** | `78` | `79` | `7A` | `7B` | `7C` | `7D` | `7E` | `7F` |

> *Note: `68` and `70` appear to be duplicates in the current ISA — subject to revision.*

### Shifts

| Opcode | Instruction |
|---|---|
| `08` | SHR A — logical shift right |
| `10` | SHL A — logical shift left |

### Arithmetic

| Opcode | Instruction | Opcode | Instruction |
|---|---|---|---|
| `80` | ADD # | `88` | ADC # |
| `81` | ADD B | `89` | ADC B |
| `82` | ADD C | `8A` | ADC C |
| `83` | ADD D | `8B` | ADC D |
| `84` | ADD E | `8C` | ADC E |
| `85` | ADD F | `8D` | ADC F |
| `86` | ADD G | `8E` | ADC G |
| `87` | ADD M | `8F` | ADC M |
| `90` | SUB # | `98` | SBC # |
| `91` | SUB B | `99` | SBC B |
| `92` | SUB C | `9A` | SBC C |
| `93` | SUB D | `9B` | SBC D |
| `94` | SUB E | `9C` | SBC E |
| `95` | SUB F | `9D` | SBC F |
| `96` | SUB G | `9E` | SBC G |
| `97` | SUB M | `9F` | SBC M |

### Logic

| Opcode | Instruction | Opcode | Instruction |
|---|---|---|---|
| `01` | CPL | — | Complement (NOT) A |
| `A0` | AND # | `A8` | OR # |
| `A1` | AND B | `A9` | OR B |
| `A2` | AND C | `AA` | OR C |
| `A3` | AND D | `AB` | OR D |
| `A4` | AND E | `AC` | OR E |
| `A5` | AND F | `AD` | OR F |
| `A6` | AND G | `AE` | OR G |
| `A7` | AND M | `AF` | OR M |

### Compare & Conditional Jump

`CMPE` — compares operand to A; if equal, jumps to address in FG:

| Opcode | Instruction |
|---|---|
| `B0` | CMPE # |
| `B1–B6` | CMPE B / C / D / E / F / G |
| `B7` | CMPE MEM[#NUM, #NUM] |

`CMPD` — compares operand to A; if equal, jumps to specified `#ADR, #ADR`:

| Opcode | Instruction |
|---|---|
| `C8` | CMPD #, #ADR, #ADR |
| `C9–CE` | CMPD B / C / D / E / F / G, #ADR, #ADR |

Relative conditional jumps at `C0–C7` — jump if the corresponding flag is 1 (assignments TBD).

`<JMP`, `=JMP`, `>JMP` — 3-way comparison jumps (less than / equal / greater than A) available as micro-instruction signals.

### Exchange

`XCHG` — swaps operand with A:

| Opcode | Instruction |
|---|---|
| `B8` | XCHG # |
| `B9–BF` | XCHG B / C / D / E / F / G / M |

> Instructions marked `!` are defined but implementation may still be in progress.

### Unimplemented / Reserved

Opcodes in ranges `C0–C7`, `D0–DF`, and `EB` are reserved or not yet assigned.

---

## Project Files

```
from-scratch-8bit-cpu/
├── src/
│   ├── CPU_4.0.dig          # Main CPU — top-level Digital schematic
│   ├── ALU_V2.dig           # ALU module
│   ├── Stack_FOR_MIT.dig    # Hardware stack module
│   └── [other modules]      # Each module has a .dig + .png + description file
├── docs/
│   └── instr.txt            # Full instruction set + microcode control signals
├── images/
│   ├── cpu8.png             # Top-level CPU schematic screenshot
│   └── thunbnail.jpg        # Demo thumbnail
├── instr-data.hex           # Microcode ROM data (hex format)
├── LICENSE
└── README.md
```

---

## How to Run

### Requirements

- [Digital](https://github.com/hneemann/Digital) by H. Neemann (free, Java-based logic simulator)
- Java runtime

### Steps

1. Clone this repo:
   ```bash
   git clone https://github.com/BohanXu-74/from-scratch-8bit-cpu.git
   ```
2. Put **all** `.dig` files (including ones inside subfolders) into a **single flat folder** — Digital needs them all in one place to resolve component references.
3. Open `CPU_4.0.dig` in Digital.
4. Load the microcode ROM:
   - Right-click the ROM on the left side of the schematic → **Edit**
   - Go to **File → Load**
   - Select `instr-data.hex`
5. Load the program into the EPROM:
   - Right-click the EPROM (second from right) → write hex instructions using `instr.txt` as your reference

---

## Build Log

| Date | Update |
|---|---|
| 02/06/2026 | Project created. Design phase. |
| 02/18/2026 | ALU physically built on breadboard. FPGA handles ADD/SUB/complement/AND/OR/XOR/shifts. Surrounding chips: buffers, comparators, registers, A register, 8 LED data bus indicators. |
| 03/13/2026 | ALU not working on real hardware — began debugging. |
| 03/15/2026 | **ALU fixed!** Root cause: a MODE pin on the FPGA was being used as a normal I/O pin, preventing boot. Also replaced switches with direct wires to eliminate pull-down resistor issues. |

---

## YouTube Series

| Video | Link |
|---|---|
| Intro | [Watch](https://www.youtube.com/watch?v=CR7b72kyQng) |
| Part 1: Program Counters | [Watch](https://www.youtube.com/watch?v=J0SWi6-3sAg) |
| Part 2: Registers | [Watch](https://www.youtube.com/watch?v=iEZ8KgztKnE) |

---

## Acknowledgements

Big thanks to **[H. Neemann](https://github.com/hneemann)** for creating Digital — the free, open-source logic simulator that made this entire project possible.

---

*This is part of an ongoing series of CPU projects. The next step is a 16-bit pipelined CPU with FPU, virtual memory, and out-of-order execution.*
