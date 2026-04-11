<!--
  Copyright (C) 2026 Software Freedom Conservancy, Inc.

  This file is part of The Inform 6 Programmer's Guide.

  This document is free documentation: you can redistribute it and/or modify
  it under the terms of the GNU General Public License as published by the
  Free Software Foundation, either version 3 of the License, or (at your
  option) any later version.

  This document is distributed in the hope that it will be useful, but
  WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY
  or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for
  more details.

  You should have received a copy of the GNU General Public License along
  with this document. If not, see <https://www.gnu.org/licenses/>.
-->

# Chapter 29: The Glulx Architecture

Glulx is a 32-bit virtual machine designed by Andrew Plotkin as a successor to
the Z-machine. It was created to remove the size and capability limitations
inherent in the Z-machine's 16-bit architecture, allowing for much larger
games, more objects, more properties, more attributes, and richer I/O through
the Glk library. The Inform 6 compiler can target Glulx directly, producing
`.ulx` game files (or Blorb-wrapped `.gblorb` files) from the same source
language used for Z-machine targets. This chapter describes the Glulx
architecture as it relates to programming in Inform 6, based on version 3.1.3
of the Glulx specification.

## §29.1 Design Goals and Relationship to Z-Machine

### §29.1.1 Overcoming 16-Bit Limitations

The Z-machine was designed in the early 1980s with 16-bit word sizes and
tight memory constraints appropriate to that era. Over time, Inform programs
grew beyond the comfortable limits of even Version 8 of the Z-machine. The
most significant limitations include:

| Resource              | Z-Machine Limit       | Glulx Limit               |
|-----------------------|-----------------------|----------------------------|
| Memory size           | 512 KB (V8)           | ~4 GB (2^32 bytes)         |
| Object count          | 65,535                | ~2^32                      |
| Property count        | 63 (individual: ~200) | Effectively unlimited      |
| Attribute count       | 48                    | Effectively unlimited      |
| Dictionary entries    | 65,535                | Effectively unlimited      |
| Routine address space | 8 MB (packed)         | Full 32-bit address space  |
| Global variables      | 240                   | Effectively unlimited      |
| String length         | 768 chars (V4+)       | No practical limit         |

Glulx removes these constraints by using a 32-bit word size throughout. All
addresses, object numbers, property numbers, and attribute numbers are 32-bit
values. The memory space is a flat, byte-addressable array limited only by the
32-bit address range.

### §29.1.2 Backward-Compatible Design Philosophy

Glulx was deliberately designed so that the same Inform 6 source language
compiles to both targets. The compiler handles the differences internally,
generating appropriate bytecode for whichever target is selected. From the
programmer's perspective, almost all Inform 6 source code works identically
on both platforms.

The key differences are:

- I/O is handled through the Glk library rather than Z-machine opcodes
- Assembly language instructions differ entirely
- Some low-level memory operations work differently
- Floating-point arithmetic is available only on Glulx

### §29.1.3 Targeting Glulx

To compile for Glulx, pass the `-G` switch to the Inform 6 compiler:

```inform6
! Compile with: inform6 -G mygame.inf
```

The compiler produces a `.ulx` file by default. To produce a Blorb-wrapped
`.gblorb` file (which can include images, sounds, and other resources), use
the `+blorb` option or a separate Blorb packaging tool.

Within source code, the target platform can be detected using conditional
compilation:

```inform6
#Ifdef TARGET_GLULX;
  ! Glulx-specific code
  print "Running on Glulx.^";
#Ifnot;
  ! Z-machine-specific code
  print "Running on the Z-machine.^";
#Endif;
```

The compiler defines `TARGET_GLULX` when the `-G` switch is active, and
`TARGET_ZCODE` otherwise. Library files use these constants extensively to
provide platform-appropriate implementations.

### §29.1.4 Output File Formats

| Format   | Extension | Description                                     |
|----------|-----------|-------------------------------------------------|
| Raw Glulx | `.ulx`   | Bare game file containing only Glulx bytecode   |
| Blorb    | `.gblorb` | Packaged file that can include multimedia resources |

Blorb (Binary resource Logical Organization for Bun Handling) is a container
format that wraps a game file together with optional images, sounds, and
metadata. Most modern interpreters support Blorb natively.

## §29.2 Memory Map

### §29.2.1 Linear Memory Layout

Glulx uses a single, flat, byte-addressable memory space. All addresses are
32-bit unsigned integers, giving a theoretical maximum of 4 GB of addressable
memory. Memory is divided into three regions whose boundaries are fixed at
compile time:

```
    Address (hex)     Segment
  +--------------+  00000000
  |    Header    |
  |  - - - - -  |  00000024
  |              |
  |     ROM      |
  |  (read-only) |
  |              |
  +--------------+  RAMSTART
  |              |
  |     RAM      |
  | (read-write) |
  |              |
  |  - - - - -  |  EXTSTART
  |              |
  |  (zeroed)    |
  |              |
  +--------------+  ENDMEM
```

The three boundary addresses — `RAMSTART`, `EXTSTART`, and `ENDMEM` — are
stored in the file header and must each be aligned on 256-byte boundaries.

### §29.2.2 ROM (Read-Only Memory)

ROM begins at address 0 and extends up to `RAMSTART`. It contains:

- The 36-byte file header
- Executable code (function bytecode)
- Constant data (strings, tables, grammar)
- The string decoding table

Any attempt to write to a ROM address is illegal and will cause the
interpreter to halt with an error. Placing code and constant data in ROM
allows interpreters to optimize execution, for example by memory-mapping the
game file directly.

### §29.2.3 RAM (Read-Write Memory)

RAM begins at `RAMSTART` and extends to `ENDMEM`. It contains:

- Global variables
- Object tables
- Property data
- Arrays and other mutable data structures

The sub-region from `RAMSTART` to `EXTSTART` is initialized from the game
file. The sub-region from `EXTSTART` to `ENDMEM` (called extended memory) is
initialized to zero when the game file is loaded. This mechanism keeps game
files smaller: large arrays that start as all-zeros need not be stored in the
file.

### §29.2.4 The Stack

The stack is maintained separately from main memory. It is not addressable by
normal memory-access instructions. The stack grows upward from address zero,
and its maximum size is specified in the file header. The stack holds:

- Call frames for active function calls
- Local variables
- Intermediate computation values
- Call stubs for save/restore and function return

All values pushed onto the stack are 32 bits wide. The stack pointer counts
in bytes, so pushing one value advances the pointer by four.

### §29.2.5 Dynamic Memory Extension

Unlike the Z-machine, Glulx allows programs to extend memory at runtime:

```inform6
! Extend memory to accommodate new data
@getmemsize sp;  ! Push current memory size onto stack
@setmemsize new_size result;  ! Request new memory size
```

The `@setmemsize` opcode requests that the interpreter expand (or contract)
the memory space. A return value of zero indicates success; non-zero indicates
failure. The new memory size must be a multiple of 256.

### §29.2.6 The Heap

The heap is an area of dynamically allocated memory managed by the `@malloc`
and `@mfree` opcodes. When the first `@malloc` call is made, the interpreter
extends memory beyond `ENDMEM` to create a heap region. Subsequent
allocations and frees are managed within this region. The heap grows
automatically as needed.

## §29.3 The Header

### §29.3.1 Header Layout

The first 36 bytes of every Glulx file form the header. All multi-byte values
are stored in big-endian (most significant byte first) order.

| Offset  | Size    | Field                 | Description                                        |
|---------|---------|-----------------------|----------------------------------------------------|
| `$00`   | 4 bytes | Magic Number          | `0x476C756C` (ASCII "Glul")                        |
| `$04`   | 4 bytes | Glulx Version         | Version number, packed as major.minor.sub           |
| `$08`   | 4 bytes | RAMSTART              | Start address of writable memory (RAM)              |
| `$0C`   | 4 bytes | EXTSTART              | Start of extended (zeroed) memory region            |
| `$10`   | 4 bytes | ENDMEM                | End of initially mapped memory                      |
| `$14`   | 4 bytes | Stack Size            | Maximum stack size in bytes                         |
| `$18`   | 4 bytes | Start Function        | Address of the entry-point function                 |
| `$1C`   | 4 bytes | String Decoding Table | Address of the Huffman decoding table               |
| `$20`   | 4 bytes | Checksum              | Checksum of the entire file                         |

### §29.3.2 Magic Number and Version

The magic number `0x476C756C` is the ASCII encoding of "Glul" and identifies
the file as a Glulx game. If the magic number does not match, the interpreter
refuses to load the file.

The version field packs the major, minor, and sub-minor version numbers into
a 32-bit value. For example, version 3.1.3 is encoded as `0x00030103`. The
interpreter uses this to determine which features and opcodes the game file
expects.

### §29.3.3 Boundary Addresses

- **RAMSTART** marks the boundary between ROM and RAM. Everything below this
  address is read-only. This value must be at least `0x100` (256) to
  accommodate the header and must be aligned to a 256-byte boundary.

- **EXTSTART** marks the boundary between file-initialized RAM and
  zero-initialized extended memory. The game file contains data only up to
  this address; memory from `EXTSTART` to `ENDMEM` is filled with zeros at
  load time.

- **ENDMEM** marks the end of initially mapped memory. This is the total
  memory the game expects to have available at startup. It too must be aligned
  to a 256-byte boundary.

### §29.3.4 Stack Size

The stack size field specifies the maximum extent of the stack in bytes. This
must be a multiple of 256. The interpreter allocates this much space for the
stack at startup. If the stack overflows during execution, the interpreter
halts with an error.

### §29.3.5 Start Function and Entry Point

The start function address points to the first function to execute when the
game begins. This is not the address of the first instruction; it is the
address of a function header (see §29.5). The interpreter calls this function
with zero arguments to begin execution.

### §29.3.6 Checksum

The checksum is computed over the entire game file. When the `@verify` opcode
is executed, the interpreter recalculates the checksum and compares it to the
stored value. This detects file corruption.

## §29.4 Data Types

### §29.4.1 Integer Types

All values in Glulx are fundamentally 32-bit unsigned integers. The VM does
not have distinct typed registers or typed memory locations. The same 32-bit
value can be treated as:

- An unsigned integer (0 to 4,294,967,295)
- A signed integer (−2,147,483,648 to 2,147,483,647) using two's complement
- A memory address
- A function reference
- An object reference
- A floating-point number (IEEE 754 single-precision)

The interpretation depends entirely on which opcodes are used to manipulate
the value. Arithmetic overflow and underflow wrap around silently with
truncation to 32 bits.

Memory can also be accessed in 8-bit and 16-bit units using appropriate load
and store opcodes (`@aloadb`/`@astoreb` for bytes, `@aloads`/`@astores` for
16-bit values).

### §29.4.2 Floating-Point Support

Glulx version 3.1.2 introduced IEEE 754 single-precision (32-bit)
floating-point support through dedicated opcodes. Floating-point values
occupy the same 32-bit slots as integers; the opcodes determine which
interpretation applies.

#### Conversion Opcodes

| Opcode      | Description                                       |
|-------------|---------------------------------------------------|
| `@numtof`   | Convert integer to float                          |
| `@ftonumz`  | Convert float to integer, rounding toward zero    |
| `@ftonumn`  | Convert float to integer, rounding to nearest     |

```inform6
! Convert an integer to float, multiply by 2.5, convert back
@numtof count sp;           ! Push float(count) onto stack
@fmul sp $40200000 sp;      ! Multiply by 2.5 (IEEE 754 encoding)
@ftonumz sp result;         ! Convert back to integer
```

#### Arithmetic Opcodes

| Opcode   | Description             |
|----------|-------------------------|
| `@fadd`  | Floating-point addition  |
| `@fsub`  | Floating-point subtraction |
| `@fmul`  | Floating-point multiplication |
| `@fdiv`  | Floating-point division  |
| `@fmod`  | Floating-point modulo    |

#### Rounding and Math Functions

| Opcode   | Description              |
|----------|--------------------------|
| `@ceil`  | Round toward +infinity    |
| `@floor` | Round toward −infinity    |
| `@sqrt`  | Square root              |
| `@exp`   | Exponential (e^x)        |
| `@log`   | Natural logarithm        |
| `@pow`   | Power (x^y)              |

#### Trigonometric Functions

| Opcode   | Description              |
|----------|--------------------------|
| `@sin`   | Sine                     |
| `@cos`   | Cosine                   |
| `@tan`   | Tangent                  |
| `@asin`  | Arcsine                  |
| `@acos`  | Arccosine                |
| `@atan`  | Arctangent               |
| `@atan2` | Two-argument arctangent  |

#### Comparison and Branch Opcodes

| Opcode    | Description                              |
|-----------|------------------------------------------|
| `@jfeq`   | Branch if floats are equal (with epsilon) |
| `@jfne`   | Branch if floats are not equal            |
| `@jflt`   | Branch if float less than                 |
| `@jfle`   | Branch if float less or equal             |
| `@jfgt`   | Branch if float greater than              |
| `@jfge`   | Branch if float greater or equal          |
| `@jisnan` | Branch if float is NaN                    |
| `@jisinf` | Branch if float is infinity               |

```inform6
! Check if a floating-point value is valid before using it
@jisnan value ?is_invalid;
@jisinf value ?is_invalid;
! Value is a finite number; proceed with computation
@fsqrt value result;
jump done;
.is_invalid;
  result = 0;
.done;
```

## §29.5 The Instruction Set

### §29.5.1 Instruction Encoding

Glulx instructions are variable-length. Each instruction consists of:

1. An opcode number (1 to 4 bytes)
2. Zero or more operands, each preceded by an addressing mode

The opcode number determines how many operands the instruction expects. The
addressing modes specify where each operand comes from (or where a result is
stored).

Opcode encoding uses a compact scheme:

| Opcode Range          | Encoding      | Bytes |
|-----------------------|---------------|-------|
| `0x00` – `0x7F`      | Single byte   | 1     |
| `0x80` – `0x3FFF`    | Two bytes     | 2     |
| `0x4000` – `0x1FFFFFFF` | Four bytes | 4     |

The addressing mode bits are packed two per byte, preceding the operand data.
Each addressing mode is a 4-bit value:

| Mode | Name              | Description                                    |
|------|-------------------|------------------------------------------------|
| 0    | ConstZero         | The value zero (no operand data)               |
| 1    | Const8            | 8-bit constant (1 byte of operand data)        |
| 2    | Const16           | 16-bit constant (2 bytes of operand data)      |
| 3    | Const32           | 32-bit constant (4 bytes of operand data)      |
| 5    | Addr8             | Memory address in 8 bits                       |
| 6    | Addr16            | Memory address in 16 bits                      |
| 7    | Addr32            | Memory address in 32 bits                      |
| 8    | Stack             | Push to or pop from the stack                  |
| 9    | Local8            | Local variable, 8-bit frame offset             |
| 10   | Local16           | Local variable, 16-bit frame offset            |
| 11   | Local32           | Local variable, 32-bit frame offset            |
| 13   | RAM8              | RAM address in 8 bits (offset from RAMSTART)   |
| 14   | RAM16             | RAM address in 16 bits                         |
| 15   | RAM32             | RAM address in 32 bits                         |

### §29.5.2 Arithmetic and Bitwise Operations

These opcodes perform integer arithmetic. All operands and results are 32-bit
values. Overflow wraps silently.

| Opcode     | Operands        | Description                       |
|------------|-----------------|-----------------------------------|
| `@add`     | L1 L2 → S1      | S1 = L1 + L2                      |
| `@sub`     | L1 L2 → S1      | S1 = L1 − L2                      |
| `@mul`     | L1 L2 → S1      | S1 = L1 × L2                      |
| `@div`     | L1 L2 → S1      | S1 = L1 ÷ L2 (signed, truncated)  |
| `@mod`     | L1 L2 → S1      | S1 = L1 mod L2 (signed)           |
| `@neg`     | L1 → S1         | S1 = −L1 (two's complement)       |
| `@bitand`  | L1 L2 → S1      | S1 = L1 AND L2                    |
| `@bitor`   | L1 L2 → S1      | S1 = L1 OR L2                     |
| `@bitxor`  | L1 L2 → S1      | S1 = L1 XOR L2                    |
| `@bitnot`  | L1 → S1         | S1 = NOT L1                       |
| `@shiftl`  | L1 L2 → S1      | S1 = L1 shifted left by L2 bits   |
| `@sshiftr` | L1 L2 → S1      | Signed (arithmetic) right shift    |
| `@ushiftr` | L1 L2 → S1      | Unsigned (logical) right shift     |

```inform6
! Compute a simple hash of two values
@mul a 31 sp;        ! sp = a * 31
@add sp b result;    ! result = (a * 31) + b

! Extract bits 8-15 from a value
@ushiftr value 8 sp;     ! Shift right by 8
@bitand sp $FF result;   ! Mask to lowest 8 bits
```

### §29.5.3 Comparison and Branch Operations

Branch instructions test a condition and jump to a target address if the
condition is met. The branch target is specified as a relative offset in the
instruction operands.

| Opcode   | Operands     | Description                              |
|----------|-------------|------------------------------------------|
| `@jz`    | L1 → label  | Branch if L1 == 0                        |
| `@jnz`   | L1 → label  | Branch if L1 != 0                        |
| `@jeq`   | L1 L2 → label | Branch if L1 == L2                     |
| `@jne`   | L1 L2 → label | Branch if L1 != L2                     |
| `@jlt`   | L1 L2 → label | Branch if L1 < L2 (signed)             |
| `@jgt`   | L1 L2 → label | Branch if L1 > L2 (signed)             |
| `@jle`   | L1 L2 → label | Branch if L1 <= L2 (signed)            |
| `@jge`   | L1 L2 → label | Branch if L1 >= L2 (signed)            |
| `@jltu`  | L1 L2 → label | Branch if L1 < L2 (unsigned)           |
| `@jgtu`  | L1 L2 → label | Branch if L1 > L2 (unsigned)           |
| `@jleu`  | L1 L2 → label | Branch if L1 <= L2 (unsigned)          |
| `@jgeu`  | L1 L2 → label | Branch if L1 >= L2 (unsigned)          |
| `@jump`  | L1           | Unconditional relative jump              |
| `@jumpabs` | L1         | Unconditional absolute jump              |

```inform6
! Branch based on a comparison
@jlt score 100 ?below_threshold;
  print "Score is 100 or above.^";
  jump done;
.below_threshold;
  print "Score is below 100.^";
.done;
```

Note the distinction between signed comparisons (`@jlt`, `@jgt`, etc.) and
unsigned comparisons (`@jltu`, `@jgtu`, etc.). Signed comparisons treat
operands as two's complement signed integers; unsigned comparisons treat
them as non-negative values.

### §29.5.4 Function Call and Return

Glulx provides several opcodes for calling functions and returning values.
Functions are identified by their memory address.

| Opcode        | Operands             | Description                             |
|---------------|----------------------|-----------------------------------------|
| `@call`       | L1 L2 → S1          | Call function at L1 with L2 args from stack, store result in S1 |
| `@callf`      | L1 → S1             | Call function at L1 with 0 args         |
| `@callfi`     | L1 L2 → S1          | Call function at L1 with 1 arg (L2)     |
| `@callfii`    | L1 L2 L3 → S1       | Call function at L1 with 2 args         |
| `@callfiii`   | L1 L2 L3 L4 → S1    | Call function at L1 with 3 args         |
| `@tailcall`   | L1 L2               | Tail-call: replaces current frame       |
| `@return`     | L1                   | Return L1 from the current function     |

The `@callf`, `@callfi`, `@callfii`, and `@callfiii` opcodes are
optimizations that pass arguments directly in the instruction rather than
requiring them to be pushed onto the stack first. For `@call`, arguments must
be pushed onto the stack in forward order (first argument pushed first) before
the call.

```inform6
! Call a function with two arguments using @callfii
@callfii MyFunc 42 100 result;

! Call a function with arguments via the stack
@copy arg3 sp;
@copy arg2 sp;
@copy arg1 sp;
@call MyFunc 3 result;
```

The `@tailcall` opcode performs a tail-call optimization: it replaces the
current call frame with the new function's frame, so the call does not consume
additional stack space. This is useful for deeply recursive algorithms.

### §29.5.5 Memory Access Operations

These opcodes read from and write to main memory. They support 8-bit, 16-bit,
and 32-bit access with array-style indexing.

| Opcode       | Operands          | Description                                |
|--------------|-------------------|--------------------------------------------|
| `@aload`     | L1 L2 → S1       | Load 32-bit value from address L1 + 4*L2   |
| `@aloads`    | L1 L2 → S1       | Load 16-bit value from address L1 + 2*L2   |
| `@aloadb`    | L1 L2 → S1       | Load 8-bit value from address L1 + L2      |
| `@aloadbit`  | L1 L2 → S1       | Load single bit from address L1, bit L2    |
| `@astore`    | L1 L2 L3         | Store 32-bit L3 at address L1 + 4*L2       |
| `@astores`   | L1 L2 L3         | Store 16-bit L3 at address L1 + 2*L2       |
| `@astoreb`   | L1 L2 L3         | Store 8-bit L3 at address L1 + L2          |
| `@astorebit` | L1 L2 L3         | Store bit L3 at address L1, bit L2         |

```inform6
! Read the third 32-bit element from an array
@aload my_array 2 value;    ! value = my_array-->2

! Write a byte to a byte array
@astoreb my_buffer index $41;  ! my_buffer->index = 'A'
```

### §29.5.6 Memory Block Operations

| Opcode    | Operands        | Description                              |
|-----------|-----------------|------------------------------------------|
| `@mcopy`  | L1 L2 L3        | Copy L1 bytes from address L2 to L3      |
| `@mzero`  | L1 L2           | Zero L1 bytes starting at address L2     |
| `@malloc` | L1 → S1         | Allocate L1 bytes, return address in S1  |
| `@mfree`  | L1              | Free memory previously allocated at L1   |

These are covered in more detail in §29.8.

### §29.5.7 Stack Manipulation

| Opcode     | Operands     | Description                              |
|------------|-------------|------------------------------------------|
| `@stkcount`| → S1        | Push the number of values on the stack   |
| `@stkpeek` | L1 → S1     | Read value L1 positions below stack top  |
| `@stkswap` |             | Swap the top two stack values            |
| `@stkroll` | L1 L2       | Roll top L1 values by L2 positions       |
| `@stkcopy` | L1          | Duplicate the top L1 stack values        |

```inform6
! Swap the top two stack values
@copy 10 sp;
@copy 20 sp;
@stkswap;      ! Stack now has 10 on top, 20 below
```

### §29.5.8 String Output

| Opcode          | Operands | Description                              |
|-----------------|---------|------------------------------------------|
| `@streamchar`   | L1      | Output L1 as a Latin-1 character          |
| `@streamunichar`| L1      | Output L1 as a Unicode code point         |
| `@streamnum`    | L1      | Output L1 as a signed decimal integer     |
| `@streamstr`    | L1      | Output the string at address L1           |

These opcodes use the currently selected I/O system to produce output. In
normal usage, the Glk I/O system is selected, so output goes to the
appropriate Glk window.

```inform6
! Output a character, a number, and a string
@streamchar 'H';
@streamchar 'i';
@streamnum 42;
@streamstr mystring;
```

### §29.5.9 Game State Operations

| Opcode          | Operands    | Description                             |
|-----------------|-------------|-----------------------------------------|
| `@save`         | L1 → S1    | Save game state to stream L1            |
| `@restore`      | L1 → S1    | Restore game state from stream L1       |
| `@saveundo`     | → S1       | Save undo state                         |
| `@restoreundo`  | → S1       | Restore undo state                      |
| `@hasundo`      | → S1       | Test if undo state is available         |
| `@discardundo`  | → S1       | Discard saved undo state                |
| `@protect`      | L1 L2      | Protect L2 bytes at address L1 from restore |
| `@restart`      |            | Restart the game from the beginning     |
| `@quit`         |            | Terminate execution                     |
| `@verify`       | → S1       | Verify game file checksum               |

The `@protect` opcode marks a region of memory that should not be overwritten
during `@restore` or `@restoreundo`. This is useful for preserving
configuration settings or transcript state across restores.

```inform6
! Save undo state and check the result
@saveundo result;
@jnz result ?undo_failed;
  ! Undo state saved successfully (or we just restored)
  jump continue;
.undo_failed;
  ! Save failed
.continue;
```

### §29.5.10 System Operations

| Opcode        | Operands      | Description                            |
|---------------|---------------|----------------------------------------|
| `@nop`        |               | No operation                           |
| `@gestalt`    | L1 L2 → S1   | Query interpreter capabilities         |
| `@debugtrap`  | L1            | Trigger a debugger breakpoint          |
| `@random`     | L1 → S1      | Generate random number                 |
| `@setrandom`  | L1            | Seed the random number generator       |
| `@getmemsize` | → S1          | Get current memory size                |
| `@setmemsize` | L1 → S1      | Set memory size; 0 on success          |
| `@glk`        | L1 L2 → S1   | Call Glk function (see §29.6)          |

The `@gestalt` opcode queries the interpreter for its capabilities. The first
operand is the selector (which capability to query) and the second is an
additional parameter. Selectors include:

| Selector | Name           | Description                                 |
|----------|----------------|---------------------------------------------|
| 0        | GlulxVersion   | Returns the Glulx version the interpreter supports |
| 1        | TerpVersion    | Returns the interpreter's own version       |
| 2        | ResizeMem      | Whether `@setmemsize` is supported          |
| 3        | Undo           | Whether `@saveundo`/`@restoreundo` work     |
| 4        | IOSystem       | Whether a given I/O system is available     |
| 5        | Unicode        | Whether Unicode opcodes are supported       |
| 6        | MemCopy        | Whether `@mcopy`/`@mzero` are supported     |
| 7        | MAlloc         | Whether `@malloc`/`@mfree` are supported    |
| 8        | MAllocHeap     | Returns the current heap start address      |
| 9        | Acceleration   | Whether acceleration opcodes are supported  |
| 10       | AccelFunc      | Whether a specific acceleration slot is supported |
| 11       | Float          | Whether floating-point opcodes are supported |

```inform6
! Check if the interpreter supports floating-point
@gestalt 11 0 sp;   ! selector 11 = Float
@jz sp ?no_float;
  ! Floating point is available
  jump continue;
.no_float;
  print "This interpreter does not support floating-point.^";
.continue;
```

The `@random` opcode generates random numbers. If L1 is positive, it returns
a value in the range 0 to L1−1. If L1 is negative, it returns a value in the
range L1+1 to 0. If L1 is zero, the interpreter generates a truly random
(or as random as possible) seed.

## §29.6 The Glk I/O System

### §29.6.1 Overview

Glulx deliberately delegates all input and output to an external library
called Glk (Generic Literary Kit). The Glulx VM itself contains no built-in
display, input, or file I/O facilities. Instead, a single opcode, `@glk`,
provides access to the complete Glk API.

This design separates the computational engine from the presentation layer,
allowing the same game file to run on text terminals, graphical windowing
systems, mobile devices, and web browsers — each with its own Glk
implementation.

### §29.6.2 The @glk Opcode

The `@glk` opcode has the following form:

```
@glk function_id argc -> result;
```

- **function_id**: The Glk function selector, a numeric constant identifying
  which Glk function to call. These constants are defined in `infglk.h`.
- **argc**: The number of arguments to pass. These must be pushed onto the
  stack in **reverse** order before the `@glk` call.
- **result**: The return value from the Glk function.

```inform6
! Example: Print a character 'A' to the current Glk stream
! glk_put_char (ch) — function selector = $0080
@glk $0080 1 0;   ! But first push 'A' onto the stack:
```

The correct calling convention pushes arguments in reverse order:

```inform6
! glk_put_char('A')
@copy $41 sp;          ! Push character 'A' (0x41) onto stack
@glk $0080 1 0;       ! Call glk_put_char with 1 argument, discard result
```

For functions with multiple arguments:

```inform6
! glk_window_move_cursor(win, xpos, ypos) — selector $00A9
! Arguments are pushed in REVERSE order: ypos, xpos, win
@copy ypos sp;
@copy xpos sp;
@copy win sp;
@glk $00A9 3 0;       ! 3 arguments, discard result
```

### §29.6.3 The infglk.h Header

The `infglk.h` file provides named constants for all Glk function selectors,
making assembly code more readable:

```inform6
Include "infglk.h";

! Now we can use named constants instead of hex values
@copy $41 sp;
@glk GLK_PUT_CHAR 1 0;
```

### §29.6.4 Key Glk Subsystems

#### Windows

Glk manages output through a tree of windows. The main window types are:

| Window Type   | Description                                          |
|---------------|------------------------------------------------------|
| Text Buffer   | Scrolling text output (main game text)               |
| Text Grid     | Fixed-position character grid (status lines)         |
| Graphics      | Pixel-addressable drawing surface                    |
| Pair          | Internal node that splits space between two children |

The window tree starts with a single root window. Additional windows are
created by splitting an existing window, which creates a pair window as the
parent.

```inform6
! Open the main text buffer window
@copy 0 sp;         ! rock value
@copy 0 sp;         ! method (0 = no split, for root)
@copy 0 sp;         ! size (ignored for root)
@copy 3 sp;         ! wintype_TextBuffer = 3
@copy 0 sp;         ! split from (0 = create root)
@glk GLK_WINDOW_OPEN 5 mainwin;
```

#### Streams

Every window has an associated output stream. Glk also supports:

- **Memory streams**: Write output to a memory buffer
- **File streams**: Read from or write to external files
- **Unicode streams**: Handle Unicode text

#### Events

Glk is event-driven. The program calls `glk_select()` to wait for the next
event. Event types include:

| Event Type       | Description                                  |
|------------------|----------------------------------------------|
| `evtype_CharInput` | A single character was typed                |
| `evtype_LineInput`  | A line of text was entered                 |
| `evtype_Timer`     | A timer interval elapsed                    |
| `evtype_Arrange`   | Window layout changed (e.g., resize)        |
| `evtype_MouseInput` | A mouse click occurred in a window         |
| `evtype_Hyperlink`  | A hyperlink was clicked                    |
| `evtype_SoundNotify`| A sound finished playing                   |

```inform6
! Request character input from a window
@copy mainwin sp;
@glk GLK_REQUEST_CHAR_EVENT 1 0;

! Wait for any event
! glk_select writes to an event structure (4 words)
@copy event_buf sp;
@glk GLK_SELECT 1 0;

! Read the event type from the structure
@aload event_buf 0 evtype;
@jeq evtype 2 ?got_char;    ! evtype_CharInput = 2
```

#### Sound

Glk provides sound channels for audio playback. Sound resources are typically
stored in a Blorb file alongside the game.

#### File I/O

Glk supports file operations through file references (filerefs) and file
streams:

```inform6
! Create a fileref for a data file
@copy $0102 sp;     ! fileusage_Data | fileusage_BinaryMode
@copy 0 sp;         ! rock
@glk GLK_FILEREF_CREATE_BY_PROMPT 2 fref;
@jz fref ?file_failed;

! Open the file for writing
@copy 0 sp;         ! rock
@copy $0001 sp;     ! filemode_Write
@copy fref sp;
@glk GLK_STREAM_OPEN_FILE 3 str;
```

#### Hyperlinks and Style Hints

Glk supports hyperlinks in text buffer windows, and style hints for
controlling text appearance (font, color, weight, etc.). These are
optional capabilities that interpreters may or may not support; programs
should test for them using `glk_gestalt()`.

### §29.6.5 Glk and the Inform 6 Library

In practice, Inform 6 programmers rarely call `@glk` directly. The standard
library wraps Glk operations in high-level routines, and the `print` statement
compiles to appropriate Glk output calls automatically. Direct `@glk` usage
is primarily needed for:

- Advanced windowing (status lines, split panes)
- Graphics and sound
- Timer-based events
- Custom file I/O
- Features not wrapped by the standard library

## §29.7 String Encoding and Decoding

### §29.7.1 String Types

Glulx supports three representations for strings, distinguished by a type
byte at the start of each string:

| Type Byte | Name               | Description                                |
|-----------|--------------------|--------------------------------------------|
| `0xE0`    | C-style string     | Uncompressed Latin-1, null-terminated      |
| `0xE1`    | Compressed string  | Huffman-encoded, using decoding table      |
| `0xE2`    | Unicode string     | Uncompressed 32-bit Unicode code points    |

### §29.7.2 C-Style Strings (0xE0)

The simplest string format: a type byte of `0xE0` followed by a sequence of
Latin-1 (ISO 8859-1) character bytes, terminated by a zero byte (`0x00`).

```
E0 48 65 6C 6C 6F 00
     H  e  l  l  o  \0
```

### §29.7.3 Compressed Strings (0xE1)

Compressed strings use Huffman encoding to reduce storage. The type byte
`0xE1` is followed by a stream of bits that encode characters using a
Huffman tree. The tree is stored in the string decoding table, whose address
is given in the file header at offset `$1C`.

The string decoding table is a tree structure with three node types:

| Node Type | Value | Description                                     |
|-----------|-------|-------------------------------------------------|
| Branch    | 0x00  | Non-leaf node; has left (bit=0) and right (bit=1) children |
| Terminator| 0x01  | End-of-string marker                            |
| Character | 0x02  | Leaf node containing a single character value   |

Additional node types support Unicode characters, indirect references, and
function calls within strings:

| Node Type        | Value | Description                               |
|------------------|-------|-------------------------------------------|
| C-string ref     | 0x03  | Reference to an uncompressed string       |
| Unicode char     | 0x04  | 32-bit Unicode character                  |
| C-string inline  | 0x05  | Inline uncompressed string                |
| Unicode str ref  | 0x08  | Reference to a Unicode string             |
| Unicode inline   | 0x09  | Inline Unicode string                     |
| Indirect ref     | 0x0A  | Indirect reference to another string      |
| Double indirect  | 0x0B  | Doubly-indirect reference                 |

The compiler builds the Huffman tree automatically by analyzing the frequency
of characters in all strings in the program. More common characters receive
shorter bit codes, reducing the overall size of string data.

### §29.7.4 Unicode Strings (0xE2)

Unicode strings begin with type byte `0xE2`, followed by a sequence of 32-bit
(4-byte) Unicode code points in big-endian order, terminated by a 32-bit zero
value.

```
E2 00000048 00000065 0000006C 0000006C 0000006F 00000000
      H        e        l        l        o       \0
```

This format supports the full Unicode range but uses four bytes per character,
making it much larger than compressed or Latin-1 strings.

### §29.7.5 The @streamstr Opcode

The `@streamstr` opcode takes a string address as its argument and outputs the
string using the current I/O system. It automatically detects the string type
from the type byte and decodes accordingly:

```inform6
@streamstr my_compressed_string;   ! Decodes Huffman-encoded string
@streamstr my_unicode_string;      ! Outputs Unicode characters
```

## §29.8 Dynamic Memory Allocation

### §29.8.1 The @malloc Opcode

The `@malloc` opcode allocates a block of memory from the heap:

```inform6
@malloc size addr;
```

- **size**: The number of bytes to allocate. Must be greater than zero.
- **addr**: Receives the address of the allocated block, or zero on failure.

The allocated memory is located above `ENDMEM`, in the heap region. The
interpreter manages the heap internally; the program should not make
assumptions about the layout of allocated blocks.

```inform6
! Allocate a buffer of 256 bytes
@malloc 256 buffer;
@jz buffer ?alloc_failed;

! Use the buffer
@astoreb buffer 0 $48;    ! buffer->0 = 'H'
@astoreb buffer 1 $69;    ! buffer->1 = 'i'

! Free the buffer when done
@mfree buffer;
```

### §29.8.2 The @mfree Opcode

The `@mfree` opcode frees a previously allocated block of memory:

```inform6
@mfree addr;
```

The address must be one previously returned by `@malloc`. Passing any other
address (including zero) causes undefined behavior. After freeing, the memory
may be reused by subsequent `@malloc` calls.

### §29.8.3 Checking Heap Support

Not all interpreters support dynamic allocation. Use `@gestalt` to check:

```inform6
@gestalt 7 0 sp;    ! Selector 7 = MAlloc
@jz sp ?no_heap;
  ! Heap is available
  jump continue;
.no_heap;
  print "Dynamic allocation not supported.^";
.continue;
```

### §29.8.4 Memory Size Management

Two opcodes control the overall memory size:

```inform6
@getmemsize sp;          ! Push current memory extent onto stack
@setmemsize new_size result;  ! Request resize; result = 0 on success
```

The `@setmemsize` opcode can expand or contract memory. The new size must be
at least as large as `ENDMEM` (the original end of memory) and must be a
multiple of 256. Contracting memory below any allocated heap block is not
permitted.

### §29.8.5 Use Cases for Dynamic Allocation

Dynamic allocation is useful for:

- Dynamically-sized data structures (lists, trees, hash tables)
- Temporary buffers of varying sizes
- Data structures whose size depends on runtime input
- Implementing advanced data structures not supported by Inform's built-in
  arrays

```inform6
! Allocate a linked list node (8 bytes: 4 for value, 4 for next pointer)
[ AllocNode value;
    @malloc 8 sp;
    @jz sp ?alloc_fail;
    @astore sp 0 value;     ! Store value at offset 0
    @astore sp 1 0;         ! Store null next pointer at offset 4
    @return sp;
  .alloc_fail;
    return 0;
];

! Free a linked list by walking it
[ FreeList node next;
    while (node) {
        @aload node 1 next;   ! next = node-->1
        @mfree node;
        node = next;
    }
];
```

## §29.9 Search Opcodes

### §29.9.1 Overview

Glulx provides three hardware-level search opcodes that operate on structured
data in memory. These opcodes are typically much faster than equivalent
hand-coded loops because interpreters can implement them as optimized native
code.

### §29.9.2 Linear Search

```
@linearsearch key keysize start structsize numstructs keyoffset options -> result
```

Performs a linear scan through an array of structures.

| Parameter    | Description                                             |
|--------------|---------------------------------------------------------|
| `key`        | The value to search for (or address if KeyIndirect)     |
| `keysize`    | Size of the key in bytes (1, 2, or 4)                   |
| `start`      | Address of the first structure in the array             |
| `structsize` | Size of each structure in bytes                         |
| `numstructs` | Number of structures to search (or -1 for unbounded)    |
| `keyoffset`  | Byte offset of the key field within each structure      |
| `options`    | Bit flags controlling search behavior                   |

The opcode compares the key against the key field of each structure in
sequence. If a match is found, the result depends on the options flags.

### §29.9.3 Binary Search

```
@binarysearch key keysize start structsize numstructs keyoffset options -> result
```

Performs a binary search on an array of structures that is already sorted by
key value in ascending order. The parameters are identical to
`@linearsearch`. Binary search is O(log n) compared to linear search's O(n).

### §29.9.4 Linked Search

```
@linkedsearch key keysize start keyoffset nextoffset options -> result
```

Searches a linked list of structures.

| Parameter    | Description                                             |
|--------------|---------------------------------------------------------|
| `key`        | The value to search for                                 |
| `keysize`    | Size of the key in bytes (1, 2, or 4)                   |
| `start`      | Address of the first structure in the list              |
| `keyoffset`  | Byte offset of the key field within each structure      |
| `nextoffset` | Byte offset of the next-pointer field within each structure |
| `options`    | Bit flags controlling search behavior                   |

The search follows next-pointers through the list until it finds a match or
reaches a null pointer (zero).

### §29.9.5 Search Options

The `options` parameter is a bitmask of the following flags:

| Flag              | Value | Description                                    |
|-------------------|-------|------------------------------------------------|
| `KeyIndirect`     | 0x01  | The `key` parameter is an address pointing to the actual key data, rather than the key value itself |
| `ZeroKeyTerminates` | 0x02 | Stop searching when a structure with an all-zero key field is encountered (for `@linearsearch` and `@binarysearch` only) |
| `ReturnIndex`     | 0x04  | Return the index of the matching structure rather than its address |

When `ReturnIndex` is not set, the result is the address of the matching
structure, or zero if no match was found. When `ReturnIndex` is set, the
result is the zero-based index, or -1 (0xFFFFFFFF) if no match was found.

```inform6
! Search a sorted array of 100 four-byte entries for value 42
@binarysearch 42 4 my_array 4 100 0 0 result;
@jz result ?not_found;

! Search with ReturnIndex to get the position
@binarysearch 42 4 my_array 4 100 0 4 index;
! index is now the zero-based position, or -1 if not found

! Search a linked list for a node with key value 7
@linkedsearch 7 4 list_head 0 4 0 result;
```

### §29.9.6 Performance Considerations

Interpreters are expected to implement these search opcodes in native code,
making them significantly faster than equivalent Inform 6 loops. Whenever
possible, structure searches in performance-critical code should use these
opcodes rather than hand-coded loops. The Inform 6 compiler and library use
them internally for dictionary lookups, property searches, and other
frequently-executed operations.

## §29.10 The Acceleration System

### §29.10.1 Overview

The acceleration system allows interpreters to provide optimized native-code
implementations of specific, frequently-called library routines. The game
file registers functions for acceleration, and the interpreter substitutes
its own optimized version when those functions are called.

This is a performance optimization only. If the interpreter does not support
acceleration, the original bytecode functions execute normally with no
change in behavior.

### §29.10.2 The @accelfunc Opcode

```
@accelfunc slot function
```

- **slot**: A numeric identifier specifying which library routine this
  function implements.
- **function**: The address of the Inform function that the interpreter may
  replace with an optimized version.

When the interpreter supports the specified slot, subsequent calls to the
given function address will execute the interpreter's native implementation
instead of interpreting the bytecode.

### §29.10.3 The @accelparam Opcode

```
@accelparam slot value
```

- **slot**: A parameter slot number (0–8 or higher).
- **value**: The value to store in that parameter slot.

The accelerated functions need access to certain global addresses and
constants to operate correctly. The `@accelparam` opcode provides these
values to the interpreter.

The parameter slots are:

| Slot | Purpose                                    |
|------|--------------------------------------------|
| 0    | Address of the class table                 |
| 1    | Number of `self` in the identifier table   |
| 2    | Address of `NUM_ATTR_BYTES` constant       |
| 3    | Address of `classes_table`                 |
| 4    | Address of `indiv_prop_start`              |
| 5    | Address of `class_metaclass`               |
| 6    | Address of `object_metaclass`              |
| 7    | Address of `routine_metaclass`             |
| 8    | Address of `string_metaclass`              |

### §29.10.4 Acceleration Slots

Each slot number corresponds to a specific library routine:

| Slot | Function       | Description                                      |
|------|----------------|--------------------------------------------------|
| 1    | `Z__Region`    | Determine the type of a value (object, routine, string, or nothing) |
| 2    | `CP__Tab`      | Look up a property in the common property table  |
| 3    | `RA__Pr`       | Get the address of a property's data              |
| 4    | `RL__Pr`       | Get the length of a property's data               |
| 5    | `OC__Cl`       | Test whether an object is a member of a class     |
| 6    | `RV__Pr`       | Read the value of a property                      |
| 7    | `OP__Pr`       | Test whether an object provides a property        |
| 8    | `CP__Tab` (v2) | Variant of CP__Tab for newer library versions     |

These routines are among the most frequently called in a typical Inform game.
They are invoked on every property access, every class membership test, and
every metaclass query. Accelerating them can yield significant performance
improvements, particularly for large games.

### §29.10.5 Compiler-Generated Acceleration

The Inform 6 compiler automatically generates the necessary `@accelfunc` and
`@accelparam` calls when targeting Glulx. The programmer does not normally
need to write acceleration code manually. The generated code is placed in the
game's initialization sequence and executes before the main game loop begins.

```inform6
! The compiler generates code equivalent to:
! (This is shown for illustration; it is not written by the programmer)
@accelparam 0 classes_table;
@accelparam 1 INDIV_PROP_START;
@accelparam 2 class_metaclass;
@accelparam 3 object_metaclass;
@accelparam 4 routine_metaclass;
@accelparam 5 string_metaclass;
@accelparam 6 self;
@accelparam 7 NUM_ATTR_BYTES;
@accelparam 8 cpv__start;
@accelfunc 1 Z__Region;
@accelfunc 2 CP__Tab;
@accelfunc 3 RA__Pr;
@accelfunc 4 RL__Pr;
@accelfunc 5 OC__Cl;
@accelfunc 6 RV__Pr;
@accelfunc 7 OP__Pr;
```

### §29.10.6 Checking Acceleration Support

Use `@gestalt` to test whether the interpreter supports acceleration:

```inform6
! Check if acceleration is supported at all
@gestalt 9 0 sp;       ! Selector 9 = Acceleration
@jz sp ?no_accel;

! Check if a specific slot is supported
@gestalt 10 1 sp;      ! Selector 10 = AccelFunc, slot 1 = Z__Region
@jz sp ?no_z_region;

! Acceleration is available for Z__Region
@accelfunc 1 Z__Region;
.no_z_region;
.no_accel;
```

If the interpreter does not support a given acceleration slot, the
`@accelfunc` call is silently ignored, and the original bytecode function
continues to execute normally. This ensures backward compatibility with
older interpreters.
