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

# Chapter 30: The Glulx Architecture

**[Glulx]** This chapter describes the architecture of the Glulx virtual
machine as targeted by the compiler. Glulx is the secondary compilation
target, designed to overcome the size and capability limitations of the
Z-machine by providing a 32-bit address space, unlimited object counts,
and a clean separation between the virtual machine and its I/O layer.
The compiler produces Glulx game files (typically with a `.ulx`
extension) when invoked with the `-G` switch.

All details in this chapter have been verified against the Inform 6.44
compiler source code and the Glulx specification (version 3.1.3). Where
the compiler's behaviour differs from or extends the specification,
those differences are noted.

## 30.1 Design Rationale and Differences from the Z-Machine

The Z-machine, originally designed by Infocom in the early 1980s,
imposes a number of constraints that become limiting for large or
complex works of interactive fiction. Its 16-bit address space caps
story files at 512 KB (version 8), it supports at most 65,535 objects
(255 in version 3), the dictionary is limited to 6 or 9 characters per
word, and its I/O is tightly coupled to the virtual machine itself.

Glulx was created to remove these constraints. Its core design
principles are:

- **32-bit address space.** All addresses, values, and object
  references are 32-bit quantities, providing a practical address space
  of up to 256 MB.

- **No built-in I/O.** All input and output is handled through the Glk
  library, an external I/O API. This separation keeps the virtual
  machine simple and allows interpreters to provide rich multimedia
  capabilities without changing the VM specification.

- **No built-in object structure.** Unlike the Z-machine, which defines
  a fixed object table format in the VM specification, Glulx provides
  only a flat memory model. The Inform compiler synthesises its own
  object table layout in memory (see §30.5).

- **Big-endian byte order.** All multi-byte values in memory are stored
  most-significant byte first.

- **Flat memory model.** Memory is a single contiguous array of bytes
  divided into ROM, RAM, and extended memory segments. There are no
  packed addresses, no banked memory, and no distinction between
  "high" and "low" memory.

### 30.1.1 Selecting Glulx Mode

The compiler targets the Z-machine by default. To compile for Glulx,
pass the `-G` switch on the command line or include `!% -G` in the
source file header. In the compiler source, this sets the global
variable `glulx_mode` to `TRUE`. When Glulx mode
is active, `WORDSIZE` is set to 4 and `MAXINTWORD` to `0x7FFFFFFF`,
reflecting the 32-bit word size.

### 30.1.2 Version Numbering and Auto-Selection

Glulx version numbers use a three-part scheme X.Y.Z, encoded as a
32-bit integer: `(X << 16) | (Y << 8) | Z`. For example, version 3.1.2
is encoded as `0x00030102`. The compiler's `select_glulx_version()`
function parses the version string provided
by the `-v` switch in Glulx mode.

If no version is explicitly requested, the compiler automatically
selects the minimum Glulx version required by the features used in the
source code. This is determined by a set of feature flags:

| Feature Flag | Minimum Version | Trigger |
|---|---|---|
| `uses_unicode_features` | 3.0.0 | `streamunichar` opcode |
| `uses_memheap_features` | 3.1.0 | `mzero`, `mcopy`, `malloc`, `mfree` opcodes |
| `uses_acceleration_features` | 3.1.1 | `accelfunc`, `accelparam` opcodes |
| `uses_float_features` | 3.1.2 | Any single-precision floating-point opcode |
| `uses_extundo_features` | 3.1.3 | `hasundo`, `discardundo` opcodes |
| `uses_double_features` | 3.1.3 | Any double-precision floating-point opcode |

These flags are set automatically during assembly when the compiler
encounters an opcode belonging to a feature-gated group (see
`assembleg_instruction()`). The final
output version is the higher of the requested
version and the version required by the features used.

### 30.1.3 Key Differences Summary

| Feature | Z-Machine | Glulx |
|---|---|---|
| Word size | 16 bits | 32 bits |
| Address space | Up to 512 KB (v8) | Up to ~256 MB |
| Packed addresses | Yes (scale factors 2, 4, or 8) | No — direct byte addresses |
| Maximum objects | 255 (v3) or 65,535 (v4+) | No fixed limit |
| Maximum attributes | 32 (v3) or 48 (v4+) | Configurable (up to 312) |
| Maximum properties | 31 (v3) or 63 (v4+) | No fixed limit |
| I/O | Built into VM | External (Glk API) |
| Byte order | Big-endian | Big-endian |
| Object table | Defined by VM spec | Compiler-synthesised |
| Story file extension | `.z3`–`.z8` | `.ulx` |
| Floating point | Not supported | IEEE-754 single and double |

## 30.2 Memory Layout and GPAGESIZE

Glulx memory is a single contiguous array of bytes, numbered from
address zero upward. It is divided into three segments whose boundaries
are specified in the file header:

```
    Address (hex)       Segment
  +-----------------+  00000000
  |     Header      |
  |  (36 bytes)     |
  +- - - - - - - - -+  00000024
  |                 |
  |    ROM          |
  |  (read-only)    |
  |                 |
  +-----------------+  RAMSTART
  |                 |
  |    RAM          |
  |  (read/write)   |
  |                 |
  +-----------------+  EXTSTART
  |                 |
  |  Extended       |
  |  Memory         |
  |  (zeroed on     |
  |   load)         |
  |                 |
  +-----------------+  ENDMEM
```

### 30.2.1 Segment Descriptions

**ROM (0x00000000 to RAMSTART):** The read-only segment contains the
36-byte Glulx header, the Inform-specific static ROM block (24 bytes),
the code area, and constant data. The interpreter must not allow the
program to write to this region. ROM is always at least 256 bytes long.

**RAM (RAMSTART to EXTSTART):** The read/write segment contains global
variables, object tables, property data, arrays, and other mutable
game state. Global variables are stored at the very beginning of RAM,
each occupying 4 bytes.

**Extended Memory (EXTSTART to ENDMEM):** This region is not stored in
the game file. It is initialised to all zeros when the game is loaded
and is used for heap allocation (`malloc`/`mfree`). The size of this
region is determined by the `MEMORY_MAP_EXTENSION` setting.

### 30.2.2 Page Alignment

All segment boundaries — RAMSTART, EXTSTART, and ENDMEM — must be
multiples of `GPAGESIZE`, which is defined as 256 bytes. The stack size specified in the header must also be a
multiple of 256. This alignment requirement simplifies interpreter
implementation by ensuring that memory protection boundaries fall on
clean page boundaries.

## 30.3 The 32-Bit Address Space

In the Z-machine, addresses are 16-bit quantities. Routines and
strings use "packed addresses" that must be multiplied by a scale
factor (2, 4, or 8 depending on the version) to obtain the actual byte
address. This packed addressing scheme allows the Z-machine to address
more than 64 KB of memory but imposes alignment requirements on code
and string placement.

Glulx eliminates packed addresses entirely. All addresses are 32-bit
byte addresses. A reference to a routine, string, or data location is
simply the byte offset from the start of memory. This has several
consequences:

- **No scale factor.** The compiler sets `scale_factor` to 0 in Glulx
  mode — it is never used.

- **No alignment restrictions on code.** Routines can begin at any byte
  address. There is no requirement for routines to be aligned to even
  addresses or any particular boundary.

- **Larger story files.** Story files can be as large as the 32-bit
  address space allows (up to 4 GB in theory, though practical limits
  are much lower).

- **Simpler address arithmetic.** All pointer arithmetic operates
  directly on byte addresses without scaling.

The compiler sets `WORDSIZE` to 4 in Glulx mode, reflecting the fact
that each machine word occupies 4 bytes. The maximum signed integer
value is `0x7FFFFFFF` (`MAXINTWORD`), and the maximum number of local
variables per routine is 119 (including `sp`).

## 30.4 The Header (36 Bytes)

Every Glulx game file begins with a 36-byte header containing nine
32-bit fields. The header size is defined by `GLULX_HEADER_SIZE` (36).
The compiler writes the header during the output phase.

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0x00 | 4 | Magic number | The ASCII string `Glul` (bytes `47 6C 75 6C`). |
| 0x04 | 4 | Version | Glulx version in `(major << 16) \| (minor << 8) \| patch` format. |
| 0x08 | 4 | RAMSTART | Byte address of the first writable memory location. |
| 0x0C | 4 | EXTSTART | Byte address where extended memory begins; equals the game file size. |
| 0x10 | 4 | ENDMEM | Total memory size needed at runtime (EXTSTART + `MEMORY_MAP_EXTENSION`). |
| 0x14 | 4 | Stack size | Maximum stack size in bytes (must be a multiple of 256). |
| 0x18 | 4 | Start function | Address of the first function to call at startup. |
| 0x1C | 4 | Decoding table | Address of the string-decoding (Huffman) table, or 0 if none. |
| 0x20 | 4 | Checksum | 32-bit checksum of the entire file, computed with this field set to 0. |

The **start function** field points to the beginning of the code area
(`Write_Code_At`), and the **decoding table** points to the string
compression table (`Write_Strings_At`). Both are set during the output
phase during output generation.

The **checksum** is computed as the sum of all bytes in the initial
memory image (ROM + RAM), treating each group of 4 bytes as a big-endian
32-bit integer. The checksum field itself is set to zero during
computation and filled in afterward.

### 30.4.1 The Inform Static ROM Block

Immediately after the 36-byte Glulx header, the compiler writes a
24-byte Inform-specific block (`GLULX_STATIC_ROM_SIZE`). This block is
part of ROM and is written immediately after the header:

| Offset | Size | Content |
|--------|------|---------|
| 0x24 | 4 | The ASCII string `Info` (Inform identifier). |
| 0x28 | 2 | Two zero bytes. |
| 0x2A | 2 | Two more zero bytes (padding). |
| 0x2C | 4 | Inform compiler version as ASCII digits (e.g., `0636` for 6.36). |
| 0x30 | 4 | Glulx back-end version (same format as compiler version). |
| 0x34 | 2 | Game release number. |
| 0x36 | 6 | Game serial number (6 ASCII characters, typically a date `YYMMDD`). |

### 30.4.2 Version Compatibility

An interpreter with version X.Y.Z can run game files with version
numbers from X.0.0 up to X.Y.* (any patch level). As a special case,
version 3.x interpreters also accept game files compiled for version
2.0.0, ensuring backward compatibility with games compiled before the
Unicode extensions were introduced.

## 30.5 Object Table Layout

Unlike the Z-machine, which defines a fixed object table format as
part of the VM specification, Glulx provides no built-in object
structure. The Inform compiler synthesises its own object table in the
RAM segment. Understanding this layout is important for advanced
debugging and for writing assembly code that manipulates objects
directly.

### 30.5.1 The Object Record

Each Glulx object is represented by the `objecttg` structure.
In the compiled output, each object
occupies a contiguous block of memory with the following layout:

| Offset (words) | Field | Size | Description |
|---|---|---|---|
| 0 | Attributes | `NUM_ATTR_BYTES` bytes | Bit array of attributes. |
| `NUM_ATTR_BYTES/4` | Type/chain | 4 bytes | Internal type byte and chain pointer. |
| 1 + `NUM_ATTR_BYTES/4` | Short name | 4 bytes | Address of the object's short name string. |
| 2 + `NUM_ATTR_BYTES/4` | Property table | 4 bytes | Address of the object's property table. |
| 3 + `NUM_ATTR_BYTES/4` | Parent | 4 bytes | Object number of the parent, or 0. |
| 4 + `NUM_ATTR_BYTES/4` | Sibling | 4 bytes | Object number of the next sibling, or 0. |
| 5 + `NUM_ATTR_BYTES/4` | Child | 4 bytes | Object number of the first child, or 0. |

The word offsets of these fields are defined by the `GOBJFIELD_*` macros:

```c
#define GOBJFIELD_CHAIN()    (1+((NUM_ATTR_BYTES)/4))
#define GOBJFIELD_NAME()     (2+((NUM_ATTR_BYTES)/4))
#define GOBJFIELD_PROPTAB()  (3+((NUM_ATTR_BYTES)/4))
#define GOBJFIELD_PARENT()   (4+((NUM_ATTR_BYTES)/4))
#define GOBJFIELD_SIBLING()  (5+((NUM_ATTR_BYTES)/4))
#define GOBJFIELD_CHILD()    (6+((NUM_ATTR_BYTES)/4))
```

The total size of each object in bytes is `OBJECT_BYTE_LENGTH`, which
the compiler calculates based on `NUM_ATTR_BYTES` and the number of
word-sized fields.

### 30.5.2 Attributes

Attributes in Glulx are stored as a byte array at the start of each
object record, rather than as a fixed-size bit field. The number of
attribute bytes is controlled by the `NUM_ATTR_BYTES` compiler setting,
with a maximum of 39 (`MAX_NUM_ATTR_BYTES`). Each
byte holds 8 attribute flags, so the maximum number of attributes is
312 (39 × 8). By contrast, the Z-machine supports at most 48 attributes
(6 bytes in versions 4+).

The default value of `NUM_ATTR_BYTES` is 7 (providing 56 attributes),
which is sufficient for the standard library. Games that need more
attributes can increase this setting via the compiler's `$` dollar
options.

### 30.5.3 Properties

Each object has a property table whose address is stored in the
object record. The property table contains a sequence of property
entries, each described by the `propg` structure:

```c
typedef struct propg {
    int num;           /* Property number */
    int continuation;  /* Continuation flag */
    int flags;         /* Property flags */
    int32 datastart;   /* Offset into property data area */
    int32 datalen;     /* Length of property data in bytes */
} propg;
```

Unlike the Z-machine, where property numbers are limited to 31 (v3) or
63 (v4+), Glulx property numbers have no fixed upper limit. The
compiler's individual property numbering begins at `INDIV_PROP_START`
(typically 256 in Glulx mode).

### 30.5.4 Comparison with Z-Machine Objects

| Feature | Z-Machine (v4+) | Glulx |
|---|---|---|
| Maximum objects | 65,535 | No fixed limit |
| Attribute bytes | 6 (48 attributes) | Configurable, up to 39 (312 attributes) |
| Tree pointers | 16-bit (parent, sibling, child) | 32-bit |
| Property numbers | 1–63 | No fixed limit |
| Property table | Fixed format, in-line data | Flexible, separate data area |
| Object record size | 14 bytes (fixed) | Variable (depends on `NUM_ATTR_BYTES`) |

## 30.6 Stack Architecture

The Glulx stack is maintained separately from main memory by the
interpreter. It is not addressable as part of the memory map — the
program cannot read or write arbitrary stack locations using memory
access opcodes.

### 30.6.1 Stack Basics

The stack pointer starts at zero and grows upward, measured in bytes.
The maximum stack size is specified in the header (offset 0x14) and must
be a multiple of 256. All values pushed onto the stack are naturally
aligned: 32-bit values are placed at addresses that are multiples of 4,
and 16-bit values at even addresses.

### 30.6.2 Call Frames

When a function is called, a new **call frame** is pushed onto the
stack. The call frame has the following structure:

```
  FramePtr + 0:             Frame Length (4 bytes)
  FramePtr + 4:             Locals Position (4 bytes)
  FramePtr + 8:             Format of Locals
                              LocalType/LocalCount pairs (2 bytes each)
                              Terminated by 0/0
                              Padded to reach 4-byte alignment
  FramePtr + LocalsPos:     Local variables
                              (1, 2, or 4 bytes each, naturally aligned)
  FramePtr + FrameLen:      Base of pushed values (grows upward)
```

The **Frame Length** records the total size of the frame header and
local variables. Above this point, the function can push and pop
temporary 32-bit values for intermediate computations. A function
cannot pop values below its own `FramePtr + FrameLen` boundary.

### 30.6.3 Call Stubs

When a function call, catch, or string-printing operation needs to
save a return context, a **call stub** is pushed onto the stack before
the new call frame:

```
  DestType   (4 bytes)   — how to store the return value
  DestAddr   (4 bytes)   — where to store it
  PC         (4 bytes)   — return address
  FramePtr   (4 bytes)   — caller's frame pointer
```

The `DestType` field indicates what to do with the value returned by
the called function:

| DestType | Meaning |
|---|---|
| 0 | Discard the return value. |
| 1 | Store in main memory at `DestAddr`. |
| 2 | Store in a local variable at frame offset `DestAddr`. |
| 3 | Push onto the stack. |
| 10 | Resume printing a compressed (E1 Huffman) string. |
| 11 | Resume function execution after embedded string printing. |
| 12 | Resume printing a decimal integer. |
| 13 | Resume printing a C-style (E0) string. |
| 14 | Resume printing a Unicode (E2) string. |

DestType values 10–14 are used internally by the interpreter during
string-printing operations and are not generated directly by compiled
Inform code.

### 30.6.4 Stack Manipulation Opcodes

Five opcodes provide direct stack manipulation:

| Opcode | Code | Description |
|---|---|---|
| `stkcount` | 0x50 | Returns the number of values above the current frame. |
| `stkpeek` | 0x51 | Reads the Nth value from the top (0 = topmost). |
| `stkswap` | 0x52 | Swaps the top two values. |
| `stkroll` | 0x53 | Rotates the top N values by M positions. |
| `stkcopy` | 0x54 | Duplicates the top N values. |

These opcodes only operate on values above `FramePtr + FrameLen` — they
cannot access local variables or the frame header. See Chapter 31 for
the complete opcode reference.

## 30.7 Glulx Instruction Set Overview

Glulx instructions use a variable-length encoding with three parts:
the opcode number, operand addressing mode bytes, and operand data. This
section provides an overview; Chapter 31 documents every opcode in
detail.

### 30.7.1 Instruction Format

```
  [Opcode: 1–4 bytes] [Mode bytes: ⌈N/2⌉ bytes] [Operand data: 0–4 bytes each]
```

The **opcode number** is encoded using 1, 2, or 4 bytes, detected by
the top two bits of the first byte:

| First Byte Pattern | Opcode Size | Range |
|---|---|---|
| `00xxxxxx` | 1 byte | 0x00–0x7F |
| `10xxxxxx` | 2 bytes | 0x0000–0x3FFF |
| `11xxxxxx` | 4 bytes | 0x00000000–0x0FFFFFFF |

The **mode bytes** encode the addressing mode for each operand, with
two operand modes packed per byte (lower nibble first). If an
instruction has an odd number of operands, the upper nibble of the
last mode byte is unused.

Each operand then follows, consuming 0 to 4 bytes depending on its
addressing mode.

### 30.7.2 Opcode Categories

The compiler's opcode table defines
over 130 opcodes organised into the following categories:

| Category | Opcode Range | Count | Description |
|---|---|---|---|
| Integer arithmetic | 0x10–0x1E | 13 | Addition, subtraction, bitwise ops, shifts |
| Branching | 0x20–0x2D, 0x104 | 14 | Conditional and unconditional jumps |
| Function calls | 0x30–0x34, 0x160–0x163 | 9 | Call, return, catch/throw, tail call |
| Data movement | 0x40–0x45 | 5 | Copy, sign-extend |
| Array access | 0x48–0x4F | 8 | Load/store at computed addresses |
| Stack operations | 0x50–0x54 | 5 | Count, peek, swap, roll, copy |
| Output | 0x70–0x73 | 4 | Stream character, number, string, Unicode char |
| Miscellaneous | 0x0100–0x0111 | 7 | Gestalt, memory size, random, debug |
| Game state | 0x0120–0x0129 | 10 | Save, restore, undo, quit, verify, protect |
| Glk interface | 0x0130 | 1 | Call Glk API function |
| String tables | 0x0140–0x0141 | 2 | Get/set string decoding table |
| I/O system | 0x0148–0x0149 | 2 | Get/set I/O system |
| Search | 0x0150–0x0152 | 3 | Linear, binary, and linked list search |
| Memory/heap | 0x170–0x179 | 4 | Zero, copy, malloc, free |
| Acceleration | 0x180–0x181 | 2 | Register accelerated functions |
| Floating point | 0x190–0x1C9 | 26 | Single-precision IEEE-754 operations |
| Double precision | 0x200–0x239 | 32 | Double-precision IEEE-754 operations |

### 30.7.3 Feature-Gated Opcode Groups

Several opcode groups are gated behind feature flags. When the compiler
encounters an opcode from one of these groups, it automatically sets
the corresponding feature flag and adjusts the minimum Glulx version
requirement. The flags are defined as bit constants:

| Flag Constant | Value | Feature | Minimum Version |
|---|---|---|---|
| `GOP_Unicode` | 1 | Unicode output | 3.0.0 |
| `GOP_MemHeap` | 2 | Memory and heap operations | 3.1.0 |
| `GOP_Acceleration` | 4 | Function acceleration | 3.1.1 |
| `GOP_Float` | 8 | Single-precision floating point | 3.1.2 |
| `GOP_ExtUndo` | 16 | Extended undo (has/discard) | 3.1.3 |
| `GOP_Double` | 32 | Double-precision floating point | 3.1.3 |

## 30.8 The Glk I/O Layer

Glulx does not include any built-in I/O facilities. Instead, all input
and output is handled through the **Glk** library, an external API
designed for portable interactive fiction I/O. This is a fundamental
architectural difference from the Z-machine, which includes dozens of
I/O-related opcodes for text display, cursor control, sound, and
graphics.

### 30.8.1 I/O System Modes

The Glulx VM supports multiple I/O system modes, selected at runtime
via the `setiosys` opcode. The available modes are:

| Mode | Value | Description |
|---|---|---|
| Null | 0 | All output is silently discarded. |
| Filter | 1 | Each output character is passed to a program-specified function. |
| Glk | 2 | Output uses the Glk API (the standard mode). |
| FyreVM | 0x20 | An alternative I/O system used by the FyreVM interpreter. |

When the VM starts, the null I/O system is active. The program must
explicitly select the Glk system (typically with `@setiosys 2 0;`)
before any output will be visible. The standard library performs this
switch during initialisation.

### 30.8.2 The `@glk` Opcode

The `@glk` opcode (0x0130) is the primary interface between Glulx
programs and the Glk library. It takes three operands:

```
@glk selector argcount result;
```

- **selector**: The Glk function identifier (e.g., `$0080` for
  `glk_put_char`).
- **argcount**: The number of arguments, which have been pushed onto
  the stack before the opcode executes.
- **result**: The destination for the Glk function's return value.

Arguments must be pushed onto the stack in reverse order (last argument
pushed first) so that the first argument is on top of the stack when
the opcode executes.

The Glk function selectors are defined in the external header file
`infglk.h`, which is not part of the compiler itself but is distributed
as part of the standard library toolkit. Programs include this file
to obtain named constants for all Glk functions.

### 30.8.3 The Glk__Wrap Veneer Routine

The compiler generates a veneer routine called `Glk__Wrap` that provides
a high-level calling interface for Glk functions. When Inform source
code calls `glk()` as a system function, the compiler emits a call to
`Glk__Wrap`.

The `Glk__Wrap` routine is a `_vararg_count` function that:

1. Pops the Glk function selector from the stack.
2. Decrements the argument count (since the selector is not a Glk
   argument).
3. Calls `@glk` with the selector, remaining argument count, and a
   result variable.
4. Returns the result.

```inform6
! The Glk__Wrap veneer routine (simplified):
[ Glk__Wrap _vararg_count callid retval;
    @copy sp callid;
    _vararg_count = _vararg_count - 1;
    @glk callid _vararg_count retval;
    return retval;
];
```

### 30.8.4 Output Opcodes

Four opcodes produce output through the current I/O system:

| Opcode | Code | Description |
|---|---|---|
| `streamchar` | 0x70 | Output a single 8-bit character. |
| `streamnum` | 0x71 | Output a signed decimal integer. |
| `streamstr` | 0x72 | Output a string object (decoded from memory). |
| `streamunichar` | 0x73 | Output a single 32-bit Unicode character. |

The `streamstr` opcode handles three string formats, identified by the
first byte of the string object:

- **E0**: A C-style null-terminated string of 8-bit characters.
- **E1**: A Huffman-compressed string (decoded using the table at
  header offset 0x1C).
- **E2**: A null-terminated string of 32-bit Unicode characters
  (with 3 bytes of padding after the type byte).

### 30.8.5 String Decoding Table

Compressed (E1) strings are decoded using a Huffman tree whose
structure is stored in memory at the address given in the header
(offset 0x1C). The decoding table format is:

| Offset | Size | Field |
|---|---|---|
| 0x00 | 4 | Table length in bytes. |
| 0x04 | 4 | Number of nodes. |
| 0x08 | 4 | Address of the root node. |
| 0x0C+ | variable | Node data. |

Each node is identified by its first byte:

| Type | Structure | Meaning |
|---|---|---|
| 0x00 | Left pointer (4) + right pointer (4) | Branch node. |
| 0x01 | (none) | String terminator. |
| 0x02 | Character (1 byte) | Single Latin-1 character. |
| 0x03 | Characters + null terminator | C-style string. |
| 0x04 | Character (4 bytes) | Single Unicode character. |
| 0x05 | Characters (4 bytes each) + null (4 bytes) | Unicode string. |
| 0x08 | Address (4 bytes) | Indirect reference to string or function. |
| 0x09 | Address (4 bytes) | Double indirect (address points to pointer). |

Decoding reads one bit at a time from the compressed data. A 0 bit
follows the left branch; a 1 bit follows the right branch. When a leaf
node is reached, its action is performed (print character, call
function, etc.) and decoding resumes from the root.

### 30.8.6 Gestalt Queries

A program can test whether the interpreter supports a particular I/O
system using the `gestalt` opcode with selector 4 (`IOSystem`):

```inform6
! Check if Glk I/O is supported:
@gestalt 4 2 result;  ! selector=IOSystem, value=Glk
if (result) {
    @setiosys 2 0;    ! Enable Glk I/O
}
```

## 30.9 Floating-Point Support

Glulx provides native support for IEEE-754 floating-point arithmetic,
a feature entirely absent from the Z-machine. Floating-point values
are stored in the same 32-bit word format as integers — the
interpretation depends on which opcodes are used to manipulate them.

### 30.9.1 Single-Precision Format (IEEE-754)

Single-precision floats use the standard IEEE-754 32-bit format, stored
in big-endian byte order:

```
  Bit 31:        Sign (S) — 0 = positive, 1 = negative
  Bits 30–23:    Exponent (E) — 8 bits, biased by 127
  Bits 22–0:     Mantissa (M) — 23 bits (implicit leading 1)
```

Some useful reference values:

| Value | Hex Encoding |
|---|---|
| 0.0 | `$00000000` |
| 1.0 | `$3F800000` |
| −1.0 | `$BF800000` |
| 2.0 | `$40000000` |
| 0.5 | `$3F000000` |
| +Infinity | `$7F800000` |
| −Infinity | `$FF800000` |
| NaN (typical) | `$7FC00000` |

### 30.9.2 Float Literals in Source Code

The Inform 6 compiler supports floating-point literals when compiling
for Glulx. The lexer's `construct_float()` function converts a decimal floating-point number into its IEEE-754 binary
representation. Float literals use the syntax:

```inform6
! Float literal examples (Glulx only):
Global pi = $+3.14159;
Global neg = $-1.0;
Global sci = $+6.022e23;
```

The `$+` or `$-` prefix distinguishes float literals from integer
hex constants. Attempting to use float literals when compiling for the
Z-machine produces an error.

### 30.9.3 Single-Precision Opcodes

Twenty-six opcodes provide single-precision floating-point operations.
These are gated behind the `GOP_Float` feature flag and require Glulx version 3.1.2 or later. The opcodes fall into
several groups:

**Conversion:** `numtof` (integer to float), `ftonumz` (float to
integer, truncating toward zero), `ftonumn` (float to integer, rounding
to nearest).

**Rounding:** `ceil` (round up), `floor` (round down). These return
float values representing integers.

**Arithmetic:** `fadd`, `fsub`, `fmul`, `fdiv` (the four basic
operations), `fmod` (modulo, producing both remainder and quotient).

**Transcendental:** `sqrt`, `exp`, `log` (natural), `pow` (power),
`sin`, `cos`, `tan`, `asin`, `acos`, `atan`, `atan2`.

**Comparison:** `jfeq` (branch if approximately equal, with tolerance),
`jfne`, `jflt`, `jfle`, `jfgt`, `jfge`.

**Special value tests:** `jisnan` (branch if NaN), `jisinf` (branch if
infinite).

NaN values are "sticky" — nearly any arithmetic operation involving NaN
produces NaN as the result. The comparison opcodes `jfeq` and `jfne`
handle NaN specially: NaN is not equal to anything, including itself.

### 30.9.4 Double-Precision Support

Glulx 3.1.3 adds support for 64-bit double-precision floating-point
values. Since Glulx is a 32-bit machine, double values are represented
as pairs of 32-bit words (high word and low word). The high word
contains the sign bit, the 11-bit exponent, and the upper 20 bits of
the mantissa; the low word contains the remaining 32 bits of the
mantissa.

Thirty-two double-precision opcodes mirror the
single-precision set: `numtod`, `dtonumz`, `dtonumn`, `ftod`, `dtof`,
`dceil`, `dfloor`, `dadd`, `dsub`, `dmul`, `ddiv`, `dmodr`, `dmodq`,
`dsqrt`, `dexp`, `dlog`, `dpow`, `dsin`, `dcos`, `dtan`, `dasin`,
`dacos`, `datan`, `datan2`, `jdeq`, `jdne`, `jdlt`, `jdle`, `jdgt`,
`jdge`, `jdisnan`, `jdisinf`.

These opcodes are gated behind the `GOP_Double` feature flag
and require Glulx version 3.1.3 or later. Because
each double occupies two 32-bit words, many double-precision opcodes
have more operands than their single-precision counterparts (e.g.,
`dadd` takes 6 operands: two pairs of inputs and one pair of outputs).

## 30.10 Acceleration and Performance

Glulx interpreters typically execute bytecode by interpreting each
instruction one at a time. For performance-critical routines that are
called very frequently — particularly the veneer routines that
implement the Inform object system — this interpretation overhead can
become significant. The acceleration mechanism allows an interpreter
to replace specific bytecode functions with native implementations.

### 30.10.1 The Acceleration Opcodes

Two opcodes control acceleration, gated behind the `GOP_Acceleration`
feature flag requiring Glulx version 3.1.1:

| Opcode | Code | Operands | Description |
|---|---|---|---|
| `accelfunc` | 0x180 | L1, L2 | Request that function number L1 be provided at address L2. |
| `accelparam` | 0x181 | L1, L2 | Set acceleration parameter L1 to value L2. |

The `accelfunc` opcode tells the interpreter: "The function at address
L2 corresponds to standard function number L1. If you have a native
implementation of function L1, use it instead of interpreting the
bytecode at L2."

### 30.10.2 Acceleration Parameters

Before registering accelerated functions, the program must provide
several parameters that the native implementations need:

| Parameter | Index | Description |
|---|---|---|
| `classes_table` | 0 | Address of the class table. |
| `indiv_prop_start` | 1 | First individual property number. |
| `class_metaclass` | 2 | Address of the Class metaclass object. |
| `object_metaclass` | 3 | Address of the Object metaclass object. |
| `routine_metaclass` | 4 | Address of the Routine metaclass object. |
| `string_metaclass` | 5 | Address of the String metaclass object. |
| `self` | 6 | Address of the `self` global variable. |
| `num_attr_bytes` | 7 | Number of attribute bytes per object. |
| `cpv_start` | 8 | Start of common property values. |

### 30.10.3 Standard Accelerated Functions

The following function numbers are defined as standard accelerated
functions:

| Number | Function | Purpose |
|---|---|---|
| 1 | `Z__Region` | Determine the type region of a value. |
| 8 | `CP__Tab` | Property table lookup. |
| 9 | `RA__Pr` | Read array property. |
| 10 | `RL__Pr` | Read property length. |
| 11 | `OC__Cl` | Object-class membership test. |
| 12 | `RV__Pr` | Read property value. |
| 13 | `OP__Pr` | Test whether an object provides a property. |

Functions 2–7 are deprecated and should not be used when
`NUM_ATTR_BYTES` differs from the default value, as they assume a
fixed attribute size.

### 30.10.4 Restrictions and Behaviour

Accelerated functions have several restrictions:

- They **cannot make VM function calls** — all logic must be
  implemented natively within the interpreter.
- They **cannot use streaming output** (`streamchar`, `streamnum`,
  `streamstr`).
- **Errors are fatal** — unlike their bytecode equivalents, which may
  call `RT__Err`, accelerated functions cannot recover from errors.
- Acceleration state is **not saved** as part of the game state. After
  a restore operation, the program must re-register all accelerated
  functions.

A program should query the interpreter's capabilities before attempting
to use acceleration:

```inform6
! Check if acceleration is supported:
@gestalt 9 0 result;    ! Acceleration gestalt selector
if (result) {
    ! Check if specific function 1 (Z__Region) is available:
    @gestalt 10 1 result;  ! AccelFunc gestalt selector
    if (result) {
        @accelparam 0 classes_table;
        @accelparam 1 INDIV_PROP_START;
        ! ... set remaining parameters ...
        @accelfunc 1 Z__Region;
    }
}
```

Acceleration applies to function calls made via `call`, `tailcall`,
`callf`, `callfi`, `callfii`, `callfiii`, filter I/O function calls,
and indirect calls from string decoding nodes.
