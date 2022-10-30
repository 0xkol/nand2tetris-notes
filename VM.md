# Nand2Tetris - VM (Chapters 7, 8)

This document describes the *Virtual Machine Language* specification, and it's implementation on the *Hack* platform.

- [Nand2Tetris - VM (Chapters 7, 8)](#nand2tetris---vm-chapters-7-8)
- [General](#general)
    - [What is a VM?](#what-is-a-vm)
    - [The Stack](#the-stack)
    - [Our VM Characteristics](#our-vm-characteristics)
- [Arithmetic and Logical Commands](#arithmetic-and-logical-commands)
- [Memory Access Commands](#memory-access-commands)
    - [Syntax](#syntax)
    - [Memory Segments Description](#memory-segments-description)
    - [RAM Usage](#ram-usage)
    - [Register Usage](#register-usage)
    - [Memory Segments Standard Mapping](#memory-segments-standard-mapping)
- [Branching Commands](#branching-commands)
- [Function Calling Commands](#function-calling-commands)
    - [Description](#description)
    - [Function Calling Protocol Description](#function-calling-protocol-description)
    - [Function Calling Protocol Implementation](#function-calling-protocol-implementation)
- [Bootstrap](#bootstrap)

# General

## What is a VM?

- The virtual machine is an emulation of computer program.
- Its purpose is to provide a platform-independent programming environment that abstracts away details of the underlying hardware or operating system and allows a program to execute in the same way on any platform.
- Examples: JVM which runs *bytecode*, CLR which runs *IL*.

## The Stack

- We realize the stack as an array.
- SP register points at the next available location in the stack.

```
Stack =============
       +---------+
       |   v1    |
       +---------+
       |   v2    |
       +---------+
SP +-> |   ...   |
       +---------+
```

### Generic Push

We push `v3` onto the stack by performing:
```
*SP = v3
SP++
```

### Generic Pop

We pop `v2` into some register by performing:
```
SP--
D=*SP
```

## Our VM Characteristics

- The VM is stack-based, meaning all operations are performed on a stack (using push and pop).
- The VM is function-based, meaning all code is organized in functions.
- Each function has its own stand-alone code and is separately handled.
- Has 16-bit data type that can be used as:
  - Integer
  - Boolean
  - Pointer
- Has four types of commands:
  - *Arithmetic commands* perform arithmetic and logical operations on the stack.
  - *Memory access commands* transfer data between the stack and virtual memory segments.
  - *Program flow commands* facilitate conditional and unconditional branching operations.
  - *Function calling commands* call functions and return from them.

# Arithmetic and Logical Commands

- Features 9 arithmetic and logical commands.
- Represents *true* as -1 and *false* as 0.

Current Stack View:
```
Stack =============
       +---------+
       |    x    |
       +---------+
       |    y    |
       +---------+
SP +-> |   ...   |
       +---------+
```

| Command | Result                        | Description                          |
|:--------|:------------------------------|:-------------------------------------|
| `add`   | x + y                         | Binary Addition (2's complement)     |
| `sub`   | x - y                         | Binary Subtraction (2's complement)  |
| `neg`   | -y                            | Arithmetic Negation (2's complement) |
| `eq`    | *true* if x = y, else *false* | Equality                             |
| `gt`    | *true* if x > y, else *false* | Greater Than                         |
| `lt`    | *true* if x < y, else *false* | Less Than                            |
| `and`   | x & y                         | Bit-wise AND                         |
| `or`    | x \| y                        | Bit-wise OR                          |
| `not`   | !y                            | Bit-wise NOT                         |

Handling overflows:

If `eq`, `gt` or `lt` are implemented naively using subtraction, the operation may overflow or underflow.

Consider the following stack view and applying the VM command `gt`:
```
+---------+
|  32767  |
+---------+
|   -1    |
+---------+
```

The result of the subtraction may cause overflow (the result is `-32768`), thus, we may judge accidentally that `32767 < -1` because `32767 - (-1) < 0` on 16-bit two's complement binary system.

Another issue may occur in the following situation (the next VM command is `lt`):
```
+---------+
| -32768  |
+---------+
|    1    |
+---------+
```

In two's complement, we get `-32768 - 1 = 32767`, and this is called underflow.

Thus we may accidentally think that since `-32768 - 1 > 0` then `-32768 > 1` which is false.

Upon examination, we see that overflow/underflow situations can occur only when `sign(x) != sign(y)`.

# Memory Access Commands

## Syntax

- `push <segment> <index>` - Push the value of *segment[index]* onto the stack
- `pop <segment> <index>` - Pop the top stack value and store it in *segment[index]*

## Memory Segments Description

| Segment | Purpose | Comments |
|:--------|:--------|:---------|
| `argument` | Stores the function's arguments | Allocated dynamically by the VM implementation when the function is called |
| `local` | Stores the function's local variables | Allocated dynamically by the VM implementation and initialized to 0's when the function is called |
| `static` | Stores static variables shared by all functions in the same `.vm`  file | Allocated by the VM impl. for each `.vm` file |
| `constant` | Virtual segment that holds all the constants in the range 0..32767 | Emulated by the VM impl. |
| `this`, `that` | General purpose segments. Can be used to refer to different areas in the heap. | Any VM function can use these segments to manipulate selected areas on the heap |
| `pointer` | A two-entry segment that holds the base address of the `this` and `that` segments | `pointer 0` refer to `this` segment, `pointer 1` refer to `that` segment |
| `temp` | Fixed 8 entries that holds temporary variables for general use | May be used by any VM function for any purpose. Shared by all VM functions |

- `argument`, `local`, `this` and `that` segments are part of the function *state* or *frame*, and they are saved when a function calls another function.
- `constant` is a virtual segment (does not exists for real), you cannot `pop` into `constant`.
- `static` shared by all VM functions in .vm file.
-  `pointer` and `temp` are shared by all VM functions.

## RAM Usage

| RAM address   | Usage                                      |
|:--------------|:-------------------------------------------|
| 0 - 15        | See [Register Usage](#register-usage)      |
| 16 - 255      | Static variables (shared by all .vm files) |
| 256 - 2047    | Stack                                      |
| 2048 - 16383  | Heap                                       |
| 16384 - 24576 | Memory mapped I/O                          |

## Register Usage

| Virtual Register Symbol | Also known as...     | Usage                                                              |
|:------------------------|:---------------------|:-------------------------------------------------------------------|
| `R0`                    | `SP` (Stack Pointer) | Points to the next available location of the stack                 |
| `R1`                    | `LCL`                | Points to the base of the current VM function's `local` segment    |
| `R2`                    | `ARG`                | Points to the base of the current VM function's `argument` segment |
| `R3`                    | `THIS`               | Points to the base of the current `this` segment (within the heap) |
| `R4`                    | `THAT`               | Points to the base of the current `that` segment (within the heap) |
| `R5` - `R12`            |                      | Holds the contents of the `temp` segment                           |
| `R13` - `R15`           |                      | General purpose registers                                          |

## Memory Segments Standard Mapping

- `local`, `argument`, `this` and `that` are mapped directly on the RAM, and its location is maintained by
keeping its physical base address in a dedicated register (`LCL`, `ARG`, `THIS` and `THAT`).

  To access the `i`th entry for each segment, we simply access the address `<segment-register> + i`.

  To service `push` we simply read from that location and store the value onto the stack.

  To service `pop` we read the topmost value from the stack and store that value in the above location.
- `pointer` maps directly into `R3` and `R4` registers. When we access `pointer 0` we access `R3` (a.k.a. `THIS`), and when we access `pointer 1` we access `R4` (a.k.a. `THAT`).
- `temp` maps directly to `R5` to `R12`. A command that tries to access `temp 9` is invalid.
- `static` maps to RAM locations 16 - 255, where `static j` directive in VM file called `file.vm` is translated to a new symbol variable in the assembly level, called: `file.j`. In the subsequent assembly process, the assembler will allocate new RAM location in the static segment address space.
- `constant` is virtual segment. Specifying `push constant i` should result in the value `i` pushed onto the stack. This segment does not occupy RAM locations.

# Branching Commands

- `label LABEL_NAME` - labels the current location in the function's code.
  - `LABEL_NAME` is a string composed of any sequence of letters, digits, underscore (_), dot (.) and colon(:) that does begin with a digit.
  - Scope of the label is the function in which it is defined.
- `goto LABEL_NAME` - effects an unconditional goto operation, causing execution to continue from the location marked by `LABEL_NAME`.
  - The jump destination must be located in the same function.
- `if-goto LABEL_NAME` - effects a conditional goto operation. The stack's topmost value is popped, if the value is **not zero**, execution continues from the location marked by `LABEL_NAME`.
  - The jump destination must be located in the same function.

> The translated label name in the *Hack* assembly code should be `functionName$LABEL_NAME`.
> 
> For example, if the command `label LBL` reside inside a function called `foo`, the standard VM mapping requires that the generated *Hack* assembly code will be `(foo$LBL)`.

# Function Calling Commands

## Description

- A function has a symbolic name used to call it. The **function name** is a string composed of any sequence of letters, digits, underscore (_), dot (.) and colon(:) that does begin with a digit.
- The **scope** of the function is global: all functions in all files are seen by each other and may call each other using the function name.
- As a side note, we expect that a method called `bar` in a class called `Foo` in some high-level language gets translated to a VM function named `Foo.bar`.

The VM language features 3 commands related to function calling:

- `function <funcName> <nVars>` - declare a VM function named `funcName` that has `nVars` local variables.
- `call <funcName> <nArgs>` - calls a VM function named `funcName`, stating that `nArgs` arguments have already been pushed to the stack by the caller.
- `return` - return to the calling function.

## Function Calling Protocol Description

The *calling* function view:

- The caller must push as many arguments as necessary onto the stack.
- The caller invokes the function using `call` command.
- After the called function returns, the arguments previously pushed are replaced by the function's returns value (which always exist).
- After the called function returns, the segments `local`, `argument`, `this`, `that` and `pointer` are the same as before the call, and the `temp` segment is undefined.

The *called* function view:

- Before the function starts, the `argument` segment has been initialized to point to the arguments that were pushed by the caller.
- Before the function starts, the `local` segment has been initialized to point to a memory location that has `nVars` words initialized to 0.
- Before the function starts, the `static` segment has been set to the `static` segment of the VM file.
- Before the function starts, the working stack is empty.
- The `this`, `that` and `pointer` segments are undefined upon entry.
- Before returning, the called function must push a value onto the stack.


## Function Calling Protocol Implementation

### The Global Stack

```
          +--------------+
          |working stack |
          +--------------+
Args. +-->| argument 0   |<- ARG
Pushed|   +--------------+
By    |   |     ...      |
Caller|   +--------------+
      +-->|argum. nArgs-1|
          +--------------+
      +-->|Return Address|
      |   +--------------+
      |   |     LCL      |
Saved |   +--------------+
State |   |     ARG      |
      |   +--------------+
      |   |     THIS     |
      |   +--------------+
      +-->|     THAT     |
          +--------------+
      +-->|    local 0   |<- LCL
Local |   +--------------+
Vars  |   |     ...      |
      |   +--------------+
      +-->|local nVars-1 |
          +--------------+
          |              |<- SP
          +--------------+
```

### `call` Implementation

**Syntax**: `call <funcName> <nArgs>`

**Role**: calling a function `funcName` after `nArgs` arguments have been pushed onto the stack.

**Pseudo-Assembly Code**:
```
// Save the state of the calling function
push RETURN_ADDRESS_LABEL // Using the label declared below
push LCL // Save LCL of the calling function
push ARG // Save ARG of the calling function
push THIS // Save THIS of the calling function
push THAT // Save THAT of the calling function

// Repositions segments for the callee
ARG = SP - nArgs - 5 // Reposition ARG
LCL = SP             // Reposition LCL

// Transfer control
goto funcName

// Declare a label for return address
(RETURN_ADDRESS_LABEL)
```

**Notes**: Each VM function call should generate and insert into the translated code stream a unique symbol that serves as a return address, namely the memory location (for the *Hack* platform, this will be in the instruction memory - ROM) of the command following the `call` command.


### `function` Implementation

**Syntax**: `function <funcName> <nVars>`

**Role**: declaring a function `funcName` that has `nVars` local variables.

**Pseudo-Assembly Code**:
```
// Declare a label for the function
(funcName)

// Push nVars zeros onto the stack (this initializes all local vars to 0)
Repeat nVars times:
push constant 0
```

**Notes**: Each VM function `funcName` should generate a symbol `funcName` that refers to its entry point in the instruction memory of the target computer. As noted already, functions are globally unique.

### `return` Implementation

**Syntax**: `return`

**Role**: return from a function.

**Pseudo-Assembly Code**:
```
FRAME = LCL // FRAME is temp var that points to the head of the saved caller frame
RET = *(FRAME - 5) // Put the return address in a temp var

// Replace function arguments with its return value (conforms to the function calling protocol)
*ARG = pop() // Copy the return value onto 'argument 0'

// Restore caller state
SP = ARG + 1 // Restore SP of the caller
THAT = *(FRAME - 1) // Restore THAT of the caller
THIS = *(FRAME - 2) // Restore THIS of the caller
ARG = *(FRAME - 3) // Restore ARG of the caller
LCL = *(FRMAE - 4) // Restore LCL of the caller

// Jump to return address in the caller's code
goto RET
```

# Bootstrap

- The *Hack* platform begins program's execution at ROM[0].
- The computer's bootstrap code should effect the following operations:

  ```
  // Initialize the stack pointer
  SP = 256

  // Start executing Sys.init
  call Sys.init 0
  ```

- Sys.init expects to call the main function of the main program, and then enter an infinite loop.