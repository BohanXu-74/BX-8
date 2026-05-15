# BX-8 — Homebrew 8-bit CPU from Scratch

Homebrew 8-bit TTL CPU with a custom ISA, ROM microcode, and hardware stack — built from scratch by an 8th grader.

[![Demo Video](https://img.shields.io/badge/YouTube-Demo%20Video-red?logo=youtube)](https://www.youtube.com/watch?v=CR7b72kyQng)
[![Hackaday](https://img.shields.io/badge/Hackaday.io-Project-green)](https://hackaday.io/project/204995-8-bit-cpu-from-scratch)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

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

## Overview

I started this after building a 4-bit CPU in 5th grade. I spent a lot of time studying the 8085, Z80, and 6502 and wanted to beat them in cycle efficiency, so I designed my own ISA and built the whole thing from scratch using TTL chips.

All the logic is simulated in [Digital](https://github.com/hneemann/Digital) by H. Neemann before being wired up on real hardware.

The main goals were to execute instructions in fewer cycles than the classic designs, support a full 128 KB address space, and have real subroutine support through a hardware stack.

⚠️ Still in progress! The ALU is fully built and tested on real hardware. Everything else is being integrated.

## Architecture

[![Schematic](https://github.com/BohanXu-74/from-scratch-8bit-cpu/raw/main/images/cpu8.png)](https://github.com/BohanXu-74/from-scratch-8bit-cpu/blob/main/images/cpu8.png)

Main components: ALU, register file, control unit, program counter, instruction decoder, RAM/ROM.

| Parameter | Value |
|---|---|
| Data bus | 8 bits |
| Address bus | 16 bits |
| Addressable memory | 128 KB (ROM + RAM) |
| General purpose registers | 4 (A, B, C, D) |
| Special purpose registers | 3+ (E, F, G + FG pair) |
| ALU operations | ADD, SUB, ADC, SBC, AND, OR, XOR, CPL, SHR, SHL, compare |
| Flags | Z (zero), C (carry), N (negative) |
| Stack depth | 16 entries x 16 bits wide |
| Instruction decoder | ROM-based microcode, 33-bit control word |
| Simulator | Digital by H. Neemann |

## Register File

The CPU has 7 registers:

| Register | Role |
|---|---|
| A | Accumulator — the only register that feeds directly into the ALU |
| B | General purpose / memory addressing |
| C | General purpose / memory addressing |
| D | General purpose / memory addressing |
| E | General purpose / memory addressing |
| F | Upper byte of the FG address pair |
| G | Lower byte of the FG address pair / I/O |

F and G together make the FG register pair, a 16-bit address register used for indirect jumps (`JMP [FG]`), subroutine calls (`CALL [FG]`), and memory-mapped I/O. It's basically the same idea as HL on the Z80.

All ALU results go back into A. Registers B through G are used as secondary operands and memory addresses.

## ALU

The ALU is on an FPGA and the operation is picked using a 3-bit control input (S1, S2, S3).

| Operation | What it does |
|---|---|
| ADD | Add register or immediate to A |
| ADC | Add with carry |
| SUB | Subtract from A |
| SBC | Subtract with borrow |
| AND | Bitwise AND with A |
| OR | Bitwise OR with A |
| XOR | Bitwise XOR with A |
| CPL | Bitwise NOT of A |
| SHR A | Logical shift right |
| SHL A | Logical shift left |
| CMPE | Compare — jumps to FG if equal |
| CMPD | Compare — jumps to a given address if equal |

Flags updated by the ALU: Z (zero), C (carry), N (negative).

The FPGA sits in the center of the breadboard. The chips around it handle buffering, comparison logic, the A register, and 8 LEDs on the data bus.

## Memory System

The CPU splits memory into ROM and RAM. ROM normally has its output enabled and holds the program. RAM and any peripherals get selected when `~ROM/RAM = 1`. Peripherals show up as memory addresses (memory-mapped I/O).

The main control signals for memory:

| Signal | What it does |
|---|---|
| `~ROM/RAM` | 0 = ROM, 1 = RAM or device |
| `~OE` | Read from RAM/device |
| `~WE` | Write to RAM/device |
| `~ROMoe` | Disable ROM output |
| `REGin` / `~REGout` | Intermediate latch for memory transfers |
| `TO MEM` | Picks whether the address bus gets its value from a register or the data bus |

## Stack

The hardware stack saves return addresses during subroutine calls. It's 16 entries deep and each entry is 16 bits wide, stored as two separate 8-bit halves.

| Signal | What it does |
|---|---|
| `~PUSH/POP AU` | Select push or pop for the upper byte |
| `CLKstack AU` | Clock the upper stack |
| `~PUSH/POP AD` | Select push or pop for the lower byte |
| `CLKstack AD` | Clock the lower stack |

`CALL [FG]` pushes the current PC onto the stack and jumps to the address in FG. `RET` pops it back.

## Control Unit & Microcode

Every instruction gets broken down into a sequence of micro-instructions. Each one is a 33-bit control word that directly drives the hardware signals. The microcode is stored in `src/instr-data.hex`.

| Bit | Signal | What it does |
|---|---|---|
| `0x000000001` | NOP | No operation |
| `0x000000002` | INSTRld | Load instruction into buffer |
| `0x000000004` | MINIclr | Clear the micro-step counter |
| `0x000000008` | REGadr | Load address for register selection |
| `0x000000010` | CLK | Clock the program counter |
| `0x000000020` | REG outEn | Selected register to data bus |
| `0x000000040` | REG inEn | Data bus to selected register |
| `0x000000080` | S1 | ALU select bit 1 |
| `0x000000100` | S2 | ALU select bit 2 |
| `0x000000200` | S3 | ALU select bit 3 |
| `0x000000400` | ADR BYTE1 | Load address byte 1 for JMP |
| `0x000000800` | ADR BYTE2 | Load address byte 2 for JMP |
| `0x000001000` | ~PUSH/POP AU | Push or pop, upper stack byte |
| `0x000002000` | CLKstack AU | Clock upper stack |
| `0x000004000` | ~PUSH/POP AD | Push or pop, lower stack byte |
| `0x000008000` | CLKstack AD | Clock lower stack |
| `0x000010000` | ~ROM/RAM | 0 = ROM, 1 = RAM/device |
| `0x000020000` | ~OE | Read from RAM/device |
| `0x000040000` | ~WE | Write to RAM/device |
| `0x000080000` | ~ROMoe | Disable ROM output |
| `0x000100000` | REGin | Load MOV register from bus |
| `0x000200000` | ~REGout | Output MOV register to bus |
| `0x000400000` | ADCen | Add with carry |
| `0x000800000` | SBCen | Subtract with borrow |
| `0x001000000` | Await | Advance clock for multi-cycle ops |
| `0x002000000` | ~JMP | Jump — load PC from address latch |
| `0x004000000` | G I/O | Register G on data bus |
| `0x008000000` | F I/O | Register F on data bus |
| `0x010000000` | CLKinstr | Clock instruction mini-decoder |
| `0x020000000` | FLAG OUTen | Output flag register to bus |
| `0x080000000` | O/~I | Direction control for FG / memory I/O |
| `0x100000000` | TO MEM | Address bus source: register or data bus |
| `0x200000000` | DISPLAY FG | Route FG to address bus when 0 |
| `0x400000000` | <JMP | Jump if selected register is less than A |
| `0x800000000` | =JMP | Jump if selected register equals A |
| `0x1000000000` | >JMP | Jump if selected register is greater than A |

## Instruction Set

60+ instructions and counting. Opcodes are 1 byte. Some instructions take extra immediate or address bytes.

Quick notation guide:
- `#NUM` = 8-bit immediate
- `#NUMH / #NUML` = high and low bytes of a 16-bit immediate
- `MEM[#NUM, #NUM]` = direct memory address (2 bytes)
- `[FG]` = indirect address from the FG register pair

### Control Flow

| Opcode | Instruction | Description |
|---|---|---|
| `00` | NOP | Do nothing |
| `02` | JMP #NUMH, #NUML | Jump to 16-bit address |
| `04` | JMP [FG] | Jump to address stored in FG |
| `05` | CALL [FG] | Call subroutine at FG, push PC to stack |
| `0C` | RET | Return from subroutine |

### Load Immediate

| Opcode | Instruction |
|---|---|
| `03` | MOV A, #NUM |
| `0B` | MOV B, #NUM |
| `13` | MOV C, #NUM |
| `1B` | MOV D, #NUM |
| `23` | MOV E, #NUM |
| `2B` | MOV F, #NUM |
| `33` | MOV G, #NUM |

### Load from Memory

| Opcode | Instruction |
|---|---|
| `06` | MOV A, MEM[#NUM, #NUM] |
| `09` | MOV B, MEM[#NUM, #NUM] |
| `11` | MOV C, MEM[#NUM, #NUM] |
| `19` | MOV D, MEM[#NUM, #NUM] |
| `21` | MOV E, MEM[#NUM, #NUM] |
| `29` | MOV F, MEM[#NUM, #NUM] |
| `31` | MOV G, MEM[#NUM, #NUM] |

### Move to Flags (MOVF)

Stores a register's value into the flag register.

`07` MOVF A &nbsp; `0A` MOVF B &nbsp; `12` MOVF C &nbsp; `1A` MOVF D &nbsp; `22` MOVF E &nbsp; `2A` MOVF F &nbsp; `32` MOVF G

### Register to Register

Full 7x7 move matrix. Row = destination, column = source.

| | A | B | C | D | E | F | G |
|---|---|---|---|---|---|---|---|
| A | `40` | `41` | `42` | `43` | `44` | `45` | `46` |
| B | `48` | `49` | `4A` | `4B` | `4C` | `4D` | `4E` |
| C | `50` | `51` | `52` | `53` | `54` | `55` | `56` |
| D | `58` | `59` | `5A` | `5B` | `5C` | `5D` | `5E` |
| E | `60` | `61` | `62` | `63` | `64` | `65` | `66` |
| F | `68` | `69` | `6A` | `6B` | `6C` | `6D` | `6E` |
| G | `70` | `71` | `72` | `73` | `74` | `75` | `76` |

### Shifts

| Opcode | Instruction |
|---|---|
| `08` | SHR A |
| `10` | SHL A |

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
| `90` | SUB # | `98` | SBC # |
| `91` | SUB B | `99` | SBC B |
| `92` | SUB C | `9A` | SBC C |
| `93` | SUB D | `9B` | SBC D |
| `94` | SUB E | `9C` | SBC E |
| `95` | SUB F | `9D` | SBC F |
| `96` | SUB G | `9E` | SBC G |

### Logic

| Opcode | Instruction | Opcode | Instruction |
|---|---|---|---|
| `01` | CPL (NOT A) | | |
| `A0` | AND # | `A8` | OR # |
| `A1` | AND B | `A9` | OR B |
| `A2` | AND C | `AA` | OR C |
| `A3` | AND D | `AB` | OR D |
| `A4` | AND E | `AC` | OR E |
| `A5` | AND F | `AD` | OR F |
| `A6` | AND G | `AE` | OR G |

### Compare & Conditional Jump

`CMPE` compares the operand to A and jumps to the address in FG if they're equal:

| Opcode | Instruction |
|---|---|
| `B0` | CMPE # |
| `B1–B6` | CMPE B / C / D / E / F / G |
| `B7` | CMPE MEM[#NUM, #NUM] |

`CMPD` compares the operand to A and jumps to a given address if equal:

| Opcode | Instruction |
|---|---|
| `C8` | CMPD #, #ADR, #ADR |
| `C9–CE` | CMPD B / C / D / E / F / G, #ADR, #ADR |

`C0–C7` are conditional jumps that fire when the corresponding flag is 1 (assignments TBD).

The microcode also has `<JMP`, `=JMP`, and `>JMP` signals for 3-way comparisons against A.

### Exchange

`XCHG` swaps a register with A:

| Opcode | Instruction |
|---|---|
| `B8` | XCHG # |
| `B9–BE` | XCHG B / C / D / E / F / G |

### Reserved / Not Yet Assigned

`C0–C7`, `D0–DF`, and `EB` are either reserved or not assigned yet. The ISA isn't finished — 256 instructions is a lot!

## Project Files

Each module in `src/` has a `.dig` schematic file, a `.png` screenshot, and a description file with notes and a YouTube link.

```
from-scratch-8bit-cpu/
├── src/
│   ├── CPU_4.0.dig            (main CPU schematic)
│   ├── alu/                   (ALU module)
|   │   ├── ALU_V2.dig
|   │   ├── alu.png
|   │   ├── aluin.png
|   │   ├── part-description.md
│   ├── instr-decoder/         (hardware stack)
|   │   ├── instr-data.hex
|   │   ├── instr-dec.png
|   │   ├── part-description.md
│   ├── pc/                    (hardware stack)
|   │   ├── part-description.md
|   │   ├── pc.png 
│   ├── reg/                   (hardware stack)
|   │   ├── part-description.md
|   │   ├── reg.png
│   ├── stack/         (hardware stack)
|   │   ├── CPU_4.0.dig
│   ├── instr-data.hex         (microcode ROM data)
│   └── [each module has a .dig + .png + description file]
├── docs/
│   └── instr.txt              (full instruction set + microcode reference)
├── images/
│   ├── cpu8.png
│   ├── pc2.png
│   ├── reg.png
│   └── thunbnail.jpg
├── LICENSE
└── README.md
```

## How to Run

You need [Digital](https://github.com/hneemann/Digital) by H. Neemann and a Java runtime.

1. Clone the repo:
   ```bash
   git clone https://github.com/BohanXu-74/from-scratch-8bit-cpu.git
   ```
2. Put all the `.dig` files (including any inside subfolders) into one flat folder. Digital needs them all in the same place to find components.
3. Open `CPU_4.0.dig` in Digital.
4. Right-click the ROM on the left side of the schematic, click Edit, then File > Load and select `src/instr-data.hex`.
5. Right-click the EPROM (second from the right) and write your program in hex using `docs/instr.txt` as a reference.

## Build Log

| Date | What happened |
|---|---|
| 02/06/2026 | Started the project, design phase. |
| 02/18/2026 | ALU built on breadboard. FPGA in the center does the math, surrounded by buffer chips, comparators, the A register, and 8 LEDs on the data bus. |
| 03/13/2026 | ALU not working on real hardware, started debugging. |
| 03/15/2026 | ALU fixed! A MODE pin on the FPGA was being used as a regular I/O pin which was stopping the chip from booting. Also swapped switches for direct wires to get rid of the pull-down resistor headache. |

## YouTube Series

| Video | Link |
|---|---|
| Intro | [Watch](https://www.youtube.com/watch?v=CR7b72kyQng) |
| Part 1: Program Counters | [Watch](https://www.youtube.com/watch?v=J0SWi6-3sAg) |
| Part 2: Registers | [Watch](https://www.youtube.com/watch?v=iEZ8KgztKnE) |

## Acknowledgements

Big thanks to [H. Neemann](https://github.com/hneemann) for making Digital. None of this would have been possible without it.

This is the predecessor to [APEX-16](https://hackaday.io/project/205126-16-bit-pipelined-cpu-with-fpu), a 16-bit pipelined CPU with an FPU that I'm also working on.
