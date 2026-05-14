
# Homemade 8-bit CPU

[![8-bit CPU Demo](images/thunbnail.jpg)](https://www.youtube.com/watch?v=CR7b72kyQng)

Watch Demo Video above!!

A homebrew 8-bit CPU designed from scratch that was inspired by classic CPUs like the 6502 and Z80.

My files are designed using Digital (look at bottom for more info) so they are in dig files that you have to download Digital to simulate and run.

PLZ READ THE PART DESCRIPTIONS!!!! They help explain a lot of confusion

## How I store stuff

- Go in src folder
- Choose a part or module of my CPU
- there will be a png containing the schematic
- there will also be a decription file with a description and a youtube link
- By the way .dig files are Digital files that I used to design my CPU link is down at the end of this file

## Why I Built This

I’m an 8th-grade student interested in CPU design, computer architecture, and how modern processors work internally.
This project is part of my journey toward building more advanced CPUs with pipelining, virtual memory, and out-of-order execution.

## Features
- 8-bit data bus
- 16-bit address bus
- Custom ISA
- Flags: Z, C, N
- Memory-mapped I/O
- Designed and implemented from scratch

## Architecture
![Schematic](images/cpu8.png)

Main components:
- ALU
- Register file
- Control unit
- Program counter
- Instruction decoder
- RAM / ROM

## Instruction Set

The instruction set is still not fully complete because 256 instructions is a lot!
I will probably update a this repo a few times.

See [`docs/instr.txt`](docs/instr.txt)

## Acknowledgements

BIG thanks to Hneemann for designing the software logic simulator Digital to make all this possible
- [Digital](https://github.com/hneemann/Digital) by Hneemann
If you want to run and test my CPU:
- Download all my src files
- make sure all the files, even the ones in the folder are put into one folder
- open the CPU_4.0.dig file
- open the ROM on the left side by right clicking it and clicking edit
- go on the left top side and press file, load
- load the instr.txt

