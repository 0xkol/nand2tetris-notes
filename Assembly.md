# Nand2Tetris - Assembly (Chapters 4, 6)

This document describes the *Hack* machine language and the *Hack* assembler.

- [Nand2Tetris - Assembly (Chapters 4, 6)](#nand2tetris---assembly-chapters-4-6)
- [*Hack* Machine Language Specification](#hack-machine-language-specification)
    - [Computer Components](#computer-components)
    - [The A-Instruction](#the-a-instruction)
    - [The C-Instruction](#the-c-instruction)
    - [Labels and Symbols](#labels-and-symbols)
    - [Input/Output Handling](#inputoutput-handling)
    - [Bit Shifting Support](#bit-shifting-support)
- [The *Hack* Assembly Process](#the-hack-assembly-process)
    - [Initialization](#initialization)
    - [First Pass](#first-pass)
    - [Second Pass](#second-pass)

# *Hack* Machine Language Specification

## Computer Components

The *Hack* computer is a von Nuemann machine, and consist of:

- CPU - executing instructions and contain the ALU.
- Memory - data memory (read/write) and instruction memory (read only).
- Input device - keyboard.
- Output device - screen.

Note:
- *Data memory* contains 24577 (24K + 1) registers, each 16-bit.

  It is addressable by 15-bit address.
- *Instruction memory* contains 32K registers and we refer to it as ROM.

## The A-Instruction

Symbolic Syntax: `@value`, where:
- `value` is non-negative integer <= 32767 (in decimal), or
- `value` is a symbol referring to such a constant.

Binary:
```
 15                                                           0
+---------------------------------------------------------------+
| 0 | v | v | v | v | v | v | v | v | v | v | v | v | v | v | v |
+---------------------------------------------------------------+
  |  \                                                         / 
  |   +--------------------------+----------------------------+
  |                              |
  v                              v
opcode                         value
```

Function:
- **Data** -  Set the value of the A register to the integer constant.
- **Addressing** - Manipulating certain data memory location, using a subsequent C-Instruction, in particular, the M register (M := RAM[A]).
- **Control Flow** - Set the stage for a jump, by first loading an address of jump destination into the A register. Note that the address of the jump is from the prespective of the ROM (instruction memory), not the RAM (the data memory).

Examples:
```
// Writing to memory - Set RAM[7421] to 0
@7421
M=0

// Writing to memory - set RAM[7421] to 5
@5
D=A
@7421
M=D

// Reading from memory - set D to RAM[7421]
@7421
D=M

// Jump to line 15 unconditionally
@15
0;JMP
```

## The C-Instruction

The purpose of the C-Instruction is to *compute* arithmetic and logic expressions (using the ALU), *store* the result in a register or memory location, and *jump* to certain program instruction based on the result of the computation.

Symbolic Syntax: `dest=comp;jump`, where:
- "dest" stands for destination, where the computed value should be stored.
- "comp" stands for computation, and specify the expression that need to be computed.
- "jump" field specify the condition on which the program shall continue execution from another instruction.

All mnemonics are listed in three tables below.

Binary:
```
 15                                                                         0
+-----------------------------------------------------------------------------+
|  1  | 1 | 1 | a | c1 | c2 | c3 | c4 | c5 | c6 | d1 | d2 | d3 | j1 | j2 | j3 |
+-----------------------------------------------------------------------------+
  |     |   |   |                             |   |          |   |          |
  v     +---+   +-----------------------------+   +----------+   +----------+
opcode  reserved         "comp" bits              "dest" bits    "jump" bits
```

"dest" field specification:

| d1 (A) | d2 (D) | d3 (M) | Mnemonic | Destination           |
|:-------|:-------|:-------|:---------|:----------------------|
| 0      | 0      | 0      | *null*   | value not stored      |
| 0      | 0      | 1      | M        | stored in RAM[A]      |
| 0      | 1      | 0      | D        | stored in D register  |
| 0      | 1      | 1      | MD       | RAM[A] and D register |
| 1      | 0      | 0      | A        | A register            |
| 1      | 0      | 1      | AM       | A register and RAM[A] |
| 1      | 1      | 0      | AD       | A, D registers        |
| 1      | 1      | 1      | AMD      | A, D and RAM[A]       |

"jump" field specification ('out' refer to the result of the "comp" field):

| j1 (out < 0) | j2 (out = 0) | j3 (out > 0) | Mnemonic | Effect           |
|:-------------|:-------------|:-------------|:---------|:-----------------|
| 0            | 0            | 0            | *null*   | no jump          |
| 0            | 0            | 1            | JGT      | jump if out > 0  |
| 0            | 1            | 0            | JEQ      | jump if out = 0  |
| 0            | 1            | 1            | JGE      | jump if out >= 0 |
| 1            | 0            | 0            | JLT      | jump if out < 0  |
| 1            | 0            | 1            | JNE      | jump if out != 0 |
| 1            | 1            | 0            | JLE      | jump if out <= 0 |
| 1            | 1            | 1            | JMP      | always jump      |

"comp" field specification:

| Mnemonic when a=0 | c1 | c2 | c3 | c4 | c5 | c6 | Mnemonic when a=1 |
|:------------------|:---|:---|:---|:---|:---|:---|:------------------|
| 0                 | 1  | 0  | 1  | 0  | 1  | 0  |                   |
| 1                 | 1  | 1  | 1  | 1  | 1  | 1  |                   |
| -1                | 1  | 1  | 1  | 0  | 1  | 0  |                   |
| D                 | 0  | 0  | 1  | 1  | 0  | 0  |                   |
| A                 | 1  | 1  | 0  | 0  | 0  | 0  | M                 |
| !D                | 0  | 0  | 1  | 1  | 0  | 1  |                   |
| !A                | 1  | 1  | 0  | 0  | 0  | 1  | !M                |
| -D                | 0  | 0  | 1  | 1  | 1  | 1  |                   |
| -A                | 1  | 1  | 0  | 0  | 1  | 1  | -M                |
| D+1               | 0  | 1  | 1  | 1  | 1  | 1  |                   |
| A+1               | 1  | 1  | 0  | 1  | 1  | 1  | M+1               |
| D-1               | 0  | 0  | 1  | 1  | 1  | 0  |                   |
| A-1               | 1  | 1  | 0  | 0  | 1  | 0  | M-1               |
| D+A               | 0  | 0  | 0  | 0  | 1  | 0  | D+M               |
| D-A               | 0  | 1  | 0  | 0  | 1  | 1  | D-M               |
| A-D               | 0  | 0  | 0  | 1  | 1  | 1  | M-D               |
| D&A               | 0  | 0  | 0  | 0  | 0  | 0  | D&M               |
| D\|A              | 0  | 1  | 0  | 1  | 0  | 1  | D\|M              |

Examples:
```
// Jump to LABEL unconditionally (computes 0, store it nowhere, and jump unconditionally)
@LABEL
0;JMP

// Jump to LABEL if D = 0
@LABEL
D;JEQ
```

## Labels and Symbols

- Symbols in *Hack* assembly program are used to refer to memory location.
- A symbol adhere to the following regex: `[a-zA-Z._:$][\w._:$]*`
  > Any sequence of letters, digits, underscore (_), dot (.), dollar sign ($) and colon (:),
  that does not begin with a digit.

### Predefined Symbols

| Predefined Symbol | RAM Address   |
|:------------------|:--------------|
| R0, R1, ..., R15  | 0, 1, ..., 15 |
| SP                | 0             |
| LCL               | 1             |
| ARG               | 2             |
| THIS              | 3             |
| THAT              | 4             |
| SCREEN            | 16384         |
| KBD               | 24576         |

### Labels

- Syntax: `(LABEL)`
- Causes the assembler to assign the label `LABEL` to the ROM location in which the next instruction is stored.
- Generates no machine code

### Variables

When the assembler encounters an A-Instruction of the form: `@symbol`, it checks against
it's symbol table for a label declaration or previous variable declaration. If no such symbol exists, the assembler assumes that `symbol` is the name of a new variable, and assign it a
new memory location, starting from RAM address 16.

## Input/Output Handling

### Screen

- Black and white screen, where "0" means white and "1" means black (think about the beginning - the screen memory map is all 0's and the screen is white).
- Screen memory map starting address is 16384, and contains 8192 registers (each 16-bit).
- Screen has 256 rows, each of 512 pixels.
- Given (x, y) coordinate on the screen, we can locate the exact register in which this pixel reside:
  - The register offset `y * 32 + x / 16` (where `/` means integer division)
  > The reason is that each row consist of 32 registers (indeed, `512 / 16 = 32`)
  - The bit number (starting from 0) is `x % 16`

### Keyboard

- Recognized all ASCII and well as other useful keys.
- 1 Register at address 24576.
- If a key is pressed, the keyboard register is populated with the scan code (which is non-zero).
- If none of the keys are pressed, the keyboard register contain 0.

## Bit Shifting Support

- Support for shift right or shift left operation is the "comp" field, through the usage of the reserved bits of the C-Instruction.
- For shifting, bit number 14 should be set to `0` and bit number 13 should be set to `1`. 
- `c1` controls the shift direction (0 - right, 1 - left) and `c2` controls the shift input from the ALU perspective (0 - y, 1 - x).
- `c3, c4, c5, c6` are ignored.

| Mnemonic when a=0 | c1 | c2 | c3 | c4 | c5 | c6 | Mnemonic when a=1 |
|:------------------|:---|:---|:---|:---|:---|:---|:------------------|
| D<<               | 1  | 1  | ?  | ?  | ?  | ?  |                   |
| D>>               | 0  | 1  | ?  | ?  | ?  | ?  |                   |
| A<<               | 1  | 0  | ?  | ?  | ?  | ?  | M<<               |
| A>>               | 0  | 0  | ?  | ?  | ?  | ?  | M>>               |

# The *Hack* Assembly Process

**Assembler** is a program that translates assembly language to machine code.

## Initialization

1. Construct empty symbol table and add the [predefined symbols](#predefined-symbols) into it.

## First Pass

2. Scan the program and maintain an instruction counter which increments only on lines containing A-Instruction or C-Instruction.
3. When encountered command of the form: `(LABEL)`,
   add `LABEL` to the symbol table along with the next instruction count.

## Second Pass

4. Initialize `n = 16`
5. Scan the entire program, focusing on A-Instruction or C-Instruction.
    - For A-Instruction of the form `@value`:
        - If `value` is a constant, translate directly.
        - If `value` is a symbol, consult with the symbol table:
            - If the symbol was found, use the symbol's stored address and translate.
            - If the symbol wasn't found, it's a new declaration of a variable.
  
              Assign the current `n` value as it's address, add the symbol to the symbol table and translate.

              Increment `n`.
    - For C-Instruction, translate directly.
6. Output the translated instruction into the output file (*.hack* extension).