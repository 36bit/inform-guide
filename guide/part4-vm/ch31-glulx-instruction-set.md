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

# Chapter 31: Glulx Instruction Set (Assembly)

**[Glulx]** This chapter documents the Glulx instruction set as supported
by the compiler. It covers the binary encoding of instructions, the
complete set of opcodes organised by category, function call conventions,
search opcodes, and the assembly language syntax for embedding Glulx
instructions directly in source code.

## 31.1 Encoding Format

Every Glulx instruction consists of three parts: an opcode number,
operand addressing mode bytes, and operand data. Unlike the Z-machine,
which uses several distinct encoding schemes (short form, long form,
variable form, extended), Glulx uses a single uniform encoding for all
instructions.

### 31.1.1 Opcode Number Encoding

The opcode number is encoded in 1, 2, or 4 bytes. The encoding is
determined by the top two bits of the first byte:

| First Byte Bits | Size | Encoding | Range |
|---|---|---|---|
| `00xxxxxx` | 1 byte | Value = byte | 0x00–0x7F |
| `10xxxxxx` | 2 bytes | Value = `(byte1 & 0x3F) << 8 \| byte2` | 0x0000–0x3FFF |
| `11xxxxxx` | 4 bytes | Value = `(byte1 & 0x0F) << 24 \| byte2 << 16 \| byte3 << 8 \| byte4` | 0x00000000–0x0FFFFFFF |

The compiler implements this encoding in `assembleg_instruction()`:

- Opcodes 0x00–0x7F are written as a single byte.
- Opcodes 0x80–0x3FFF are written as two bytes: `0x80 | (code >> 8)`
  followed by `code & 0xFF`.
- Opcodes 0x4000 and above are written as four bytes:
  `0xC0 | (code >> 24)` followed by the remaining three bytes.

In practice, most commonly used opcodes (arithmetic, branching, data
movement) fall in the single-byte range. Extended opcodes such as
`gestalt` (0x0100) and `glk` (0x0130) use the two-byte encoding.
Floating-point and double-precision opcodes (0x190+) also use two-byte
encoding.

### 31.1.2 Feature Flag Auto-Detection

When the compiler assembles an instruction from a feature-gated opcode
group, it automatically sets the corresponding feature flag. This ensures the output file's Glulx version number
is high enough to support all opcodes used. The six feature groups and
their flag constants are:

| Flag | Constant | Value | Opcodes |
|---|---|---|---|
| `uses_unicode_features` | `GOP_Unicode` | 1 | `streamunichar` |
| `uses_memheap_features` | `GOP_MemHeap` | 2 | `mzero`, `mcopy`, `malloc`, `mfree` |
| `uses_acceleration_features` | `GOP_Acceleration` | 4 | `accelfunc`, `accelparam` |
| `uses_float_features` | `GOP_Float` | 8 | All single-precision float opcodes |
| `uses_extundo_features` | `GOP_ExtUndo` | 16 | `hasundo`, `discardundo` |
| `uses_double_features` | `GOP_Double` | 32 | All double-precision float opcodes |

## 31.2 Operand Addressing Modes

After the opcode, each instruction has one byte of addressing mode data
for every two operands (rounded up). Each mode byte encodes two
operand modes: the lower nibble (bits 0–3) for the first operand of
the pair, and the upper nibble (bits 4–7) for the second. If an
instruction has an odd number of operands, the upper nibble of the
last mode byte is zero (unused).

The compiler writes placeholder zero bytes for the mode bytes, then
fills them in as each operand is assembled.

### 31.2.1 Addressing Mode Table

Sixteen addressing modes are defined. Modes 4 and 12 are reserved and
unused.

| Mode | Hex | Data Bytes | Load Semantics | Store Semantics |
|---|---|---|---|---|
| 0 | `0` | 0 | Constant zero | Discard result |
| 1 | `1` | 1 | 8-bit signed constant (sign-extended to 32 bits) | *Illegal* |
| 2 | `2` | 2 | 16-bit signed constant (sign-extended to 32 bits) | *Illegal* |
| 3 | `3` | 4 | 32-bit constant | *Illegal* |
| 5 | `5` | 1 | Contents of memory address 0x00–0xFF | Store to memory address 0x00–0xFF |
| 6 | `6` | 2 | Contents of memory address 0x0000–0xFFFF | Store to memory address 0x0000–0xFFFF |
| 7 | `7` | 4 | Contents of any memory address | Store to any memory address |
| 8 | `8` | 0 | Pop value from stack | Push value onto stack |
| 9 | `9` | 1 | Local variable (1-byte frame offset) | Store to local (1-byte offset) |
| 10 | `A` | 2 | Local variable (2-byte frame offset) | Store to local (2-byte offset) |
| 11 | `B` | 4 | Local variable (4-byte frame offset) | Store to local (4-byte offset) |
| 13 | `D` | 1 | RAM-relative address (RAMSTART + byte offset) | Store to RAM-relative (byte offset) |
| 14 | `E` | 2 | RAM-relative address (RAMSTART + short offset) | Store to RAM-relative (short offset) |
| 15 | `F` | 4 | RAM-relative address (RAMSTART + word offset) | Store to RAM-relative (word offset) |

**Constant modes (0–3):** When used as a load operand, these provide
immediate values. Modes 1 and 2 sign-extend their data to 32 bits.
When used as a store operand, mode 0 means "discard the result";
modes 1–3 are illegal (the compiler reports an error if an instruction
attempts to store to a constant).

**Memory modes (5–7):** These read from or write to absolute memory
addresses. The data bytes give the address directly.

**Stack mode (8):** When loading, this pops the top value from the
stack. When storing, this pushes the value onto the stack.

**Local variable modes (9–11):** These access local variables in the
current call frame. The data bytes give a byte offset from the frame's
locals area. Local variables are always 32-bit aligned.

**RAM-relative modes (13–15):** These are like memory modes 5–7, but
the data bytes give an offset from RAMSTART rather than an absolute
address. This is useful for accessing global variables and other RAM
data using shorter addressing.

### 31.2.2 Compiler Encoding of Variables

The compiler uses several strategies to encode variable references
efficiently:

**Stack pointer (sp):** Variable 0 in the compiler represents the
stack pointer. It is always encoded as mode 8 (stack pop/push) with
no data bytes.

**Local variables:** Variables 1 through `no_locals` are encoded as
local frame offsets. Each local occupies 4 bytes, so variable *n* has
frame offset `(n-1) × 4`. If the offset fits in one byte (variables 1
through 64), mode 9 is used; otherwise mode 10.

**Global variables:** Global variable *n* is stored in RAM at offset
`(n - MAX_LOCAL_VARIABLES) × 4` from RAMSTART. The compiler encodes
this as a RAM-relative address using mode 13 (1-byte offset, up to
offset 255), mode 14 (2-byte offset, up to 65535), or mode 15
(4-byte offset) as needed.

### 31.2.3 Operand Flags in the Opcode Table

Each opcode in the compiler's table has a
`flags` field indicating special operand handling:

| Flag | Value | Meaning |
|---|---|---|
| `St` | 1 | The last operand (or second-to-last if `Br` is also set) is a store destination. |
| `Br` | 2 | The last operand is a branch offset. |
| `Rf` | 4 | Execution never continues past this instruction (return, jump, quit). |
| `St2` | 8 | The second-to-last operand is also a store destination. |

## 31.3 Full Opcode Listing with Semantics

This section provides the complete listing of all Glulx opcodes as
defined in the compiler's opcode table. For
each opcode, the following information is given:

- **Code**: The opcode number in hexadecimal.
- **Name**: The assembly mnemonic.
- **Operands**: The operand signature, using L for load operands,
  S for store operands, and Br for branch offsets.
- **Semantics**: A description of the operation.

### 31.3.1 Integer Arithmetic (0x10–0x1E)

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x10 | `add` | L1 L2 S1 | S1 = L1 + L2 (32-bit, wrapping on overflow). |
| 0x11 | `sub` | L1 L2 S1 | S1 = L1 − L2. |
| 0x12 | `mul` | L1 L2 S1 | S1 = L1 × L2 (lower 32 bits of result). |
| 0x13 | `div` | L1 L2 S1 | S1 = L1 ÷ L2 (signed, truncated toward zero). Division by zero is a fatal error. |
| 0x14 | `mod` | L1 L2 S1 | S1 = L1 mod L2 (signed remainder; sign matches dividend). Division by zero is fatal. |
| 0x15 | `neg` | L1 S1 | S1 = −L1 (two's complement negation). |
| 0x18 | `bitand` | L1 L2 S1 | S1 = L1 AND L2 (bitwise). |
| 0x19 | `bitor` | L1 L2 S1 | S1 = L1 OR L2 (bitwise). |
| 0x1A | `bitxor` | L1 L2 S1 | S1 = L1 XOR L2 (bitwise). |
| 0x1B | `bitnot` | L1 S1 | S1 = NOT L1 (bitwise complement). |
| 0x1C | `shiftl` | L1 L2 S1 | S1 = L1 << L2. If L2 ≥ 32 or L2 < 0, result is 0. |
| 0x1D | `sshiftr` | L1 L2 S1 | S1 = L1 >> L2 (arithmetic right shift; sign bit is replicated). If L2 ≥ 32 or L2 < 0, result is 0 or −1 depending on sign. |
| 0x1E | `ushiftr` | L1 L2 S1 | S1 = L1 >>> L2 (logical right shift; zero-filled). If L2 ≥ 32 or L2 < 0, result is 0. |

All integer values are treated as unsigned unless explicitly noted.
Signed operations (such as `div`, `mod`, and comparisons) use two's
complement representation.

### 31.3.2 Branching and Comparison (0x20–0x2D, 0x104)

Branch instructions test a condition and, if true, transfer control to
a target location. The branch offset is computed as:

```
destination = address_after_instruction + offset − 2
```

Two special offset values have predefined meanings:

- **Offset 0**: Return 0 from the current function.
- **Offset 1**: Return 1 from the current function.

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x20 | `jump` | Br1 | Unconditional branch to Br1. |
| 0x22 | `jz` | L1 Br1 | Branch if L1 == 0. |
| 0x23 | `jnz` | L1 Br1 | Branch if L1 ≠ 0. |
| 0x24 | `jeq` | L1 L2 Br1 | Branch if L1 == L2. |
| 0x25 | `jne` | L1 L2 Br1 | Branch if L1 ≠ L2. |
| 0x26 | `jlt` | L1 L2 Br1 | Branch if L1 < L2 (signed). |
| 0x27 | `jge` | L1 L2 Br1 | Branch if L1 ≥ L2 (signed). |
| 0x28 | `jgt` | L1 L2 Br1 | Branch if L1 > L2 (signed). |
| 0x29 | `jle` | L1 L2 Br1 | Branch if L1 ≤ L2 (signed). |
| 0x2A | `jltu` | L1 L2 Br1 | Branch if L1 < L2 (unsigned). |
| 0x2B | `jgeu` | L1 L2 Br1 | Branch if L1 ≥ L2 (unsigned). |
| 0x2C | `jgtu` | L1 L2 Br1 | Branch if L1 > L2 (unsigned). |
| 0x2D | `jleu` | L1 L2 Br1 | Branch if L1 ≤ L2 (unsigned). |
| 0x104 | `jumpabs` | L1 | Jump to absolute address L1 (not an offset). |

The `jump` opcode has both the `Br` and `Rf` flags set: it is a branch
instruction (the operand is a branch offset), and execution never
continues to the next instruction. The `jumpabs` opcode has only `Rf`
— its operand is an absolute address, not a relative offset, and the
special values 0 and 1 are not treated as return-0/return-1.

Note that the compiler can only generate branches to constant offsets
(labels), even though the Glulx VM supports loading branch offsets from
variables or the stack.

### 31.3.3 Function Calls and Returns (0x30–0x34, 0x160–0x163)

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x30 | `call` | L1 L2 S1 | Call function at address L1 with L2 arguments (taken from the stack). Store return value in S1. |
| 0x31 | `return` | L1 | Return L1 from the current function. |
| 0x32 | `catch` | S1 Br1 | Generate a catch token, store it in S1, and branch to Br1. |
| 0x33 | `throw` | L1 L2 | Throw value L1 using catch token L2. Unwinds the stack to the catch point and stores L1 as the return value. |
| 0x34 | `tailcall` | L1 L2 | Tail-call function at L1 with L2 arguments. The current frame is discarded and replaced by the new call. |
| 0x160 | `callf` | L1 S1 | Call function at L1 with 0 arguments. Store return value in S1. |
| 0x161 | `callfi` | L1 L2 S1 | Call function at L1 with 1 argument (L2). Store return value in S1. |
| 0x162 | `callfii` | L1 L2 L3 S1 | Call function at L1 with 2 arguments (L2, L3). Store return value in S1. |
| 0x163 | `callfiii` | L1 L2 L3 L4 S1 | Call function at L1 with 3 arguments (L2, L3, L4). Store return value in S1. |

The `call` opcode is the general-purpose calling mechanism: arguments
are pushed onto the stack (last argument first, so the first argument
is on top), and L2 specifies how many to consume. The `callf` family
provides more efficient encoding for calls with 0–3 inline arguments,
avoiding the need to push arguments onto the stack.

The `tailcall` opcode optimises recursive and tail-position calls by
reusing the current stack frame. It has the `Rf` flag set because
execution never continues past it — the called function's return value
is delivered directly to the caller of the current function.

### 31.3.4 Data Movement (0x40–0x45)

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x40 | `copy` | L1 S1 | Copy L1 to S1 (32-bit value). |
| 0x41 | `copys` | L1 S1 | Copy L1 to S1 as a 16-bit value. When loading, zero-extends to 32 bits; when storing, truncates to 16 bits. |
| 0x42 | `copyb` | L1 S1 | Copy L1 to S1 as an 8-bit value. When loading, zero-extends; when storing, truncates. |
| 0x44 | `sexs` | L1 S1 | Sign-extend a 16-bit value to 32 bits. |
| 0x45 | `sexb` | L1 S1 | Sign-extend an 8-bit value to 32 bits. |

The `copy` opcode is used to move values between any two locations
(memory, locals, stack, or constants). It is the most frequently used
instruction for variable assignment.

Note that `copys` and `copyb` have special addressing rules: when used
with memory modes (5–7 and 13–15), they access 16-bit or 8-bit
quantities respectively, rather than 32-bit values. This is an
exception to the normal rule that all memory accesses are 32-bit.

### 31.3.5 Array Access (0x48–0x4F)

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x48 | `aload` | L1 L2 S1 | S1 = 32-bit value at address L1 + 4×L2. |
| 0x49 | `aloads` | L1 L2 S1 | S1 = 16-bit value at address L1 + 2×L2 (zero-extended). |
| 0x4A | `aloadb` | L1 L2 S1 | S1 = 8-bit value at address L1 + L2 (zero-extended). |
| 0x4B | `aloadbit` | L1 L2 S1 | S1 = single bit: bit (L2 mod 8) of byte at address L1 + (L2 / 8). Result is 0 or 1. |
| 0x4C | `astore` | L1 L2 L3 | Store 32-bit value L3 at address L1 + 4×L2. |
| 0x4D | `astores` | L1 L2 L3 | Store 16-bit value L3 at address L1 + 2×L2. |
| 0x4E | `astoreb` | L1 L2 L3 | Store 8-bit value L3 at address L1 + L2. |
| 0x4F | `astorebit` | L1 L2 L3 | Set or clear bit (L2 mod 8) of byte at address L1 + (L2 / 8). If L3 is 0, clear; if non-zero, set. |

These opcodes implement Inform's `-->` (word array) and `->` (byte
array) access patterns. The `aload`/`astore` pair corresponds to `-->`,
while `aloadb`/`astoreb` corresponds to `->`. The `aloads`/`astores`
pair accesses 16-bit (short) values, useful for Unicode string tables
and other 16-bit data. The bit-level access opcodes (`aloadbit`,
`astorebit`) can access individual bits in memory.

Note that the store opcodes (`astore`, `astores`, `astoreb`,
`astorebit`) do not have the `St` flag — they take three load operands
rather than two loads and a store, because the destination is computed
from the first two operands rather than being a simple variable or
stack location.

### 31.3.6 Stack Operations (0x50–0x54)

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x50 | `stkcount` | S1 | Store the number of 32-bit values on the stack above the current call frame into S1. |
| 0x51 | `stkpeek` | L1 S1 | Read the L1-th value from the top of the stack (0 = topmost) without removing it. |
| 0x52 | `stkswap` | (none) | Swap the top two values on the stack. Equivalent to `stkroll 2 1`. |
| 0x53 | `stkroll` | L1 L2 | Rotate the top L1 values on the stack by L2 positions. Positive L2 rolls upward (top value moves down); negative L2 rolls downward. |
| 0x54 | `stkcopy` | L1 | Duplicate the top L1 values on the stack. |

These opcodes only affect values pushed above the current call frame
boundary (`FramePtr + FrameLen`). They cannot access local variables
or frame metadata.

### 31.3.7 Output and I/O System (0x70–0x73, 0x0140–0x0149)

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x70 | `streamchar` | L1 | Output L1 as a single 8-bit (Latin-1) character via the current I/O system. |
| 0x71 | `streamnum` | L1 | Output L1 as a signed decimal integer via the current I/O system. |
| 0x72 | `streamstr` | L1 | Output the string at address L1. Decodes E0, E1, or E2 format strings. |
| 0x73 | `streamunichar` | L1 | Output L1 as a single 32-bit Unicode character. Requires Glulx 3.0+. |
| 0x140 | `getstringtbl` | S1 | Store the current string decoding table address into S1. |
| 0x141 | `setstringtbl` | L1 | Set the string decoding table to address L1. Set to 0 to disable E1 decoding. |
| 0x148 | `getiosys` | S1 S2 | Store the current I/O system mode in S1 and its "rock" value in S2. |
| 0x149 | `setiosys` | L1 L2 | Set the I/O system to mode L1 with rock value L2. For filter mode (1), L2 is the function address. |

The `getiosys` opcode has both `St` and `St2` flags set, indicating
that both operands are store destinations.

### 31.3.8 Miscellaneous (0x00, 0x0100–0x0111)

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x00 | `nop` | (none) | No operation. |
| 0x100 | `gestalt` | L1 L2 S1 | Query VM capability. L1 is the selector, L2 is an argument. Result stored in S1. |
| 0x101 | `debugtrap` | L1 | Trigger interpreter-specific debugging. Behaviour is implementation-defined. |
| 0x102 | `getmemsize` | S1 | Store the current total memory size (ENDMEM) into S1. |
| 0x103 | `setmemsize` | L1 S1 | Request memory resize to L1 bytes. S1 receives 0 on success, 1 on failure. L1 must be a multiple of 256 and ≥ ENDMEM. |
| 0x110 | `random` | L1 S1 | Generate a random number. If L1 > 0, result is in 0..(L1−1). If L1 < 0, result is in (L1+1)..0. If L1 = 0, result is any 32-bit value. |
| 0x111 | `setrandom` | L1 | Seed the random number generator. L1 = 0 selects a non-deterministic seed. |

The **gestalt** opcode is the primary mechanism for capability
detection. Important selectors include:

| Selector | Value | Query |
|---|---|---|
| GlulxVersion | 0 | Returns the VM version number. |
| TerpVersion | 1 | Returns the interpreter version number. |
| ResizeMem | 2 | Returns 1 if memory resizing is supported. |
| Undo | 3 | Returns 1 if undo is supported. |
| IOSystem | 4 | Returns 1 if I/O system L2 is supported. |
| Unicode | 5 | Returns 1 if Unicode features are supported. |
| MemCopy | 6 | Returns 1 if `mzero`/`mcopy` are supported. |
| MAlloc | 7 | Returns 1 if `malloc`/`mfree` are supported. |
| MAllocHeap | 8 | Returns heap start address or 0 if inactive. |
| Acceleration | 9 | Returns 1 if acceleration is supported. |
| AccelFunc | 10 | Returns 1 if accelerated function L2 is available. |
| Float | 11 | Returns 1 if floating-point opcodes are supported. |
| ExtUndo | 12 | Returns 1 if `hasundo`/`discardundo` are supported. |
| Double | 13 | Returns 1 if double-precision opcodes are supported. |

### 31.3.9 Game State (0x0120–0x0129)

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x120 | `quit` | (none) | Terminate the interpreter. |
| 0x121 | `verify` | S1 | Verify the game file integrity. S1 receives 0 if valid. |
| 0x122 | `restart` | (none) | Reset the VM to its initial state. |
| 0x123 | `save` | L1 S1 | Save the game state to Glk stream L1. S1 receives 0 on success, 1 on failure. On restore, S1 receives −1. |
| 0x124 | `restore` | L1 S1 | Restore game state from Glk stream L1. On success, execution resumes at the save point. S1 receives 1 on failure (success does not return here). |
| 0x125 | `saveundo` | S1 | Save the game state for undo. S1 receives 0 on success, 1 on failure. On undo-restore, S1 receives −1. |
| 0x126 | `restoreundo` | S1 | Restore the most recent undo state. S1 receives 1 on failure (success does not return here). |
| 0x127 | `protect` | L1 L2 | Protect the memory range [L1, L1+L2) from being affected by save/restore. Only one range can be protected at a time. |
| 0x128 | `hasundo` | S1 | Test whether an undo state is available. S1 receives 1 if yes, 0 if no. Requires Glulx 3.1.3+. |
| 0x129 | `discardundo` | (none) | Discard the most recent undo state. Requires Glulx 3.1.3+. |

### 31.3.10 Glk Interface (0x0130)

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x130 | `glk` | L1 L2 S1 | Call Glk function L1 with L2 arguments (taken from the stack). Store the return value in S1. |

This is the sole gateway between the Glulx program and the Glk I/O
library. Arguments must be pushed onto the stack before the instruction
executes (last argument pushed first, so the first argument is on top).
The compiler's internal opcode constant is `glk_gc` (value 68).

See §30.8 for a detailed discussion of the Glk I/O layer.

### 31.3.11 Search Opcodes (0x0150–0x0152)

Three opcodes provide efficient searching of structured data in
memory. These are frequently used by the compiler's veneer routines
for property table lookups.

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x150 | `linearsearch` | L1 L2 L3 L4 L5 L6 L7 S1 | Linear (sequential) search. |
| 0x151 | `binarysearch` | L1 L2 L3 L4 L5 L6 L7 S1 | Binary search on a sorted array. |
| 0x152 | `linkedsearch` | L1 L2 L3 L4 L5 L6 S1 | Linked list search. |

See §31.6 for the detailed parameter descriptions and semantics of
each search opcode.

### 31.3.12 Memory and Heap (0x170–0x179)

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x170 | `mzero` | L1 L2 | Zero L1 bytes of memory starting at address L2. |
| 0x171 | `mcopy` | L1 L2 L3 | Copy L1 bytes from address L2 to address L3. Handles overlapping regions correctly. |
| 0x178 | `malloc` | L1 S1 | Allocate L1 bytes of heap memory. S1 receives the address of the allocated block, or 0 on failure. |
| 0x179 | `mfree` | L1 | Free the heap block at address L1 (must be the exact address returned by `malloc`). |

These opcodes are gated behind `GOP_MemHeap` and require Glulx 3.1.0+.
The `malloc` and `mfree` opcodes manage memory beyond the initial
ENDMEM boundary. The `mzero` and `mcopy` opcodes operate on any
writable memory region.

### 31.3.13 Acceleration (0x180–0x181)

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x180 | `accelfunc` | L1 L2 | Register accelerated function L1 at address L2. The interpreter may replace calls to L2 with a native implementation of function L1. |
| 0x181 | `accelparam` | L1 L2 | Set acceleration parameter L1 to value L2. |

See §30.10 for the full acceleration parameter table and function
numbers.

### 31.3.14 Single-Precision Floating Point (0x190–0x1C9)

All single-precision opcodes are gated behind `GOP_Float` and require
Glulx 3.1.2+. Float values use IEEE-754 32-bit representation stored
in big-endian byte order within the same 32-bit word as integers.

#### Conversion

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x190 | `numtof` | L1 S1 | Convert signed integer L1 to float. |
| 0x191 | `ftonumz` | L1 S1 | Convert float L1 to signed integer (truncate toward zero). |
| 0x192 | `ftonumn` | L1 S1 | Convert float L1 to signed integer (round to nearest). |

#### Rounding

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x198 | `ceil` | L1 S1 | Round float L1 up to the nearest integer (result is still in float format). |
| 0x199 | `floor` | L1 S1 | Round float L1 down to the nearest integer (result is still in float format). |

#### Arithmetic

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x1A0 | `fadd` | L1 L2 S1 | S1 = L1 + L2 (float addition). |
| 0x1A1 | `fsub` | L1 L2 S1 | S1 = L1 − L2 (float subtraction). |
| 0x1A2 | `fmul` | L1 L2 S1 | S1 = L1 × L2 (float multiplication). |
| 0x1A3 | `fdiv` | L1 L2 S1 | S1 = L1 ÷ L2 (float division). Division by zero produces ±Infinity or NaN. |
| 0x1A4 | `fmod` | L1 L2 S1 S2 | S1 = remainder of L1 ÷ L2; S2 = quotient (truncated toward zero). Both in float format. |

The `fmod` opcode has both `St` and `St2` flags, indicating two store
destinations.

#### Transcendental Functions

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x1A8 | `sqrt` | L1 S1 | S1 = √L1. |
| 0x1A9 | `exp` | L1 S1 | S1 = e^L1. |
| 0x1AA | `log` | L1 S1 | S1 = ln(L1) (natural logarithm). |
| 0x1AB | `pow` | L1 L2 S1 | S1 = L1^L2 (L1 raised to the power L2). |
| 0x1B0 | `sin` | L1 S1 | S1 = sin(L1) (L1 in radians). |
| 0x1B1 | `cos` | L1 S1 | S1 = cos(L1). |
| 0x1B2 | `tan` | L1 S1 | S1 = tan(L1). |
| 0x1B3 | `asin` | L1 S1 | S1 = arcsin(L1). |
| 0x1B4 | `acos` | L1 S1 | S1 = arccos(L1). |
| 0x1B5 | `atan` | L1 S1 | S1 = arctan(L1). |
| 0x1B6 | `atan2` | L1 L2 S1 | S1 = arctan(L1/L2) using the signs of both arguments to determine the quadrant. |

#### Float Comparison and Special Value Tests

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x1C0 | `jfeq` | L1 L2 L3 Br1 | Branch if \|L1 − L2\| ≤ L3 (approximate equality with tolerance). |
| 0x1C1 | `jfne` | L1 L2 L3 Br1 | Branch if \|L1 − L2\| > L3 (not approximately equal). |
| 0x1C2 | `jflt` | L1 L2 Br1 | Branch if L1 < L2 (float comparison). |
| 0x1C3 | `jfle` | L1 L2 Br1 | Branch if L1 ≤ L2 (float comparison). |
| 0x1C4 | `jfgt` | L1 L2 Br1 | Branch if L1 > L2 (float comparison). |
| 0x1C5 | `jfge` | L1 L2 Br1 | Branch if L1 ≥ L2 (float comparison). |
| 0x1C8 | `jisnan` | L1 Br1 | Branch if L1 is NaN (Not a Number). |
| 0x1C9 | `jisinf` | L1 Br1 | Branch if L1 is ±Infinity. |

NaN comparisons follow IEEE-754 rules: NaN is not equal to anything,
including itself. `jfeq` with a NaN operand never branches (unless the
tolerance L3 is also NaN). `jfne` with a NaN operand always branches.

### 31.3.15 Double-Precision Floating Point (0x200–0x239)

All double-precision opcodes are gated behind `GOP_Double` and require
Glulx 3.1.3+. Double values are represented as pairs of 32-bit words
(high word first, low word second). Most double opcodes have more
operands than their single-precision counterparts to accommodate the
paired representation.

#### Conversion

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x200 | `numtod` | L1 S1 S2 | Convert integer L1 to double (S1 = high word, S2 = low word). |
| 0x201 | `dtonumz` | L1 L2 S1 | Convert double (L1:L2) to integer (truncate toward zero). |
| 0x202 | `dtonumn` | L1 L2 S1 | Convert double (L1:L2) to integer (round to nearest). |
| 0x203 | `ftod` | L1 S1 S2 | Convert single-precision float L1 to double (S1:S2). |
| 0x204 | `dtof` | L1 L2 S1 | Convert double (L1:L2) to single-precision float S1. |

#### Rounding

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x208 | `dceil` | L1 L2 S1 S2 | Round double (L1:L2) up; result in double format (S1:S2). |
| 0x209 | `dfloor` | L1 L2 S1 S2 | Round double (L1:L2) down; result in double format (S1:S2). |

#### Arithmetic

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x210 | `dadd` | L1 L2 L3 L4 S1 S2 | (L1:L2) + (L3:L4) → (S1:S2). |
| 0x211 | `dsub` | L1 L2 L3 L4 S1 S2 | (L1:L2) − (L3:L4) → (S1:S2). |
| 0x212 | `dmul` | L1 L2 L3 L4 S1 S2 | (L1:L2) × (L3:L4) → (S1:S2). |
| 0x213 | `ddiv` | L1 L2 L3 L4 S1 S2 | (L1:L2) ÷ (L3:L4) → (S1:S2). |
| 0x214 | `dmodr` | L1 L2 L3 L4 S1 S2 | Remainder of (L1:L2) ÷ (L3:L4) → (S1:S2). |
| 0x215 | `dmodq` | L1 L2 L3 L4 S1 S2 | Quotient of (L1:L2) ÷ (L3:L4) → (S1:S2), truncated toward zero. |

#### Transcendental Functions

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x218 | `dsqrt` | L1 L2 S1 S2 | √(L1:L2) → (S1:S2). |
| 0x219 | `dexp` | L1 L2 S1 S2 | e^(L1:L2) → (S1:S2). |
| 0x21A | `dlog` | L1 L2 S1 S2 | ln(L1:L2) → (S1:S2). |
| 0x21B | `dpow` | L1 L2 L3 L4 S1 S2 | (L1:L2)^(L3:L4) → (S1:S2). |
| 0x220 | `dsin` | L1 L2 S1 S2 | sin(L1:L2) → (S1:S2). |
| 0x221 | `dcos` | L1 L2 S1 S2 | cos(L1:L2) → (S1:S2). |
| 0x222 | `dtan` | L1 L2 S1 S2 | tan(L1:L2) → (S1:S2). |
| 0x223 | `dasin` | L1 L2 S1 S2 | arcsin(L1:L2) → (S1:S2). |
| 0x224 | `dacos` | L1 L2 S1 S2 | arccos(L1:L2) → (S1:S2). |
| 0x225 | `datan` | L1 L2 S1 S2 | arctan(L1:L2) → (S1:S2). |
| 0x226 | `datan2` | L1 L2 L3 L4 S1 S2 | arctan((L1:L2)/(L3:L4)) → (S1:S2). |

#### Double Comparison and Special Value Tests

| Code | Name | Operands | Semantics |
|---|---|---|---|
| 0x230 | `jdeq` | L1 L2 L3 L4 L5 L6 Br1 | Branch if \|(L1:L2) − (L3:L4)\| ≤ (L5:L6). |
| 0x231 | `jdne` | L1 L2 L3 L4 L5 L6 Br1 | Branch if \|(L1:L2) − (L3:L4)\| > (L5:L6). |
| 0x232 | `jdlt` | L1 L2 L3 L4 Br1 | Branch if (L1:L2) < (L3:L4). |
| 0x233 | `jdle` | L1 L2 L3 L4 Br1 | Branch if (L1:L2) ≤ (L3:L4). |
| 0x234 | `jdgt` | L1 L2 L3 L4 Br1 | Branch if (L1:L2) > (L3:L4). |
| 0x235 | `jdge` | L1 L2 L3 L4 Br1 | Branch if (L1:L2) ≥ (L3:L4). |
| 0x238 | `jdisnan` | L1 L2 Br1 | Branch if (L1:L2) is NaN. |
| 0x239 | `jdisinf` | L1 L2 Br1 | Branch if (L1:L2) is ±Infinity. |

### 31.3.16 Synthetic Opcodes (Compiler Macros)

The compiler provides four synthetic opcodes that are not real Glulx
instructions but are expanded into equivalent real instructions during
assembly. They are defined in the `opmacros_table_g` table:

| Macro | Expands To | Description |
|---|---|---|
| `pull` S1 | `copy sp S1` | Pop the top stack value into S1. |
| `push` L1 | `copy L1 sp` | Push L1 onto the stack. |
| `dload` L1 S1 S2 | Two `aload` instructions | Load a double (8-byte) value from address L1. |
| `dstore` L1 L2 L3 | Two `astore` instructions | Store a double (8-byte) value to address L1. |

The `pull` and `push` macros provide syntactic convenience for stack
operations. The `dload` and `dstore` macros support double-precision
floating-point values by loading or storing two consecutive 32-bit
words.

## 31.4 Function Call Conventions

Glulx supports two function types, distinguished by a type byte at the
beginning of the function's code in memory.

### 31.4.1 Function Types

**Type C1 (0xC1) — Local-argument functions.** This is the standard
function type used by the Inform compiler for most routines. Arguments
passed by the caller are copied into the function's local variables.
If more arguments are passed than the function declares, the extras are
discarded. If fewer arguments are passed, the remaining locals are
initialised to zero.

**Type C0 (0xC0) — Stack-argument functions.** Arguments remain on the
stack rather than being copied into locals. The function accesses its
arguments by popping them from the stack or using `stkpeek`. This type
is used for Inform functions whose first local is named
`_vararg_count`. In the compiled output,
the compiler emits a `@copy sp _vararg_count;` instruction at the start
of the function body to move the argument count from the stack into the
first local.

The compiler writes the function type byte as follows:

```c
if (stackargs)
    byteout(0xC0, 0); /* Stack-argument function */
else
    byteout(0xC1, 0); /* Local-argument function */
```

### 31.4.2 Function Format in Memory

A function in memory has the following layout:

```
  Byte 0:       Type byte (0xC0 or 0xC1)
  Bytes 1+:     LocalType/LocalCount pairs (2 bytes each)
                  LocalType = 4 (always, for 32-bit locals)
                  LocalCount = 1–255
                  Multiple pairs if more than 255 locals
                Terminated by a 0/0 pair
                Padded with 0/0 if necessary to reach 4-byte alignment
  Thereafter:   Executable opcodes
```

The compiler always uses 4-byte locals. If
a function has more than 255 locals, multiple LocalType/LocalCount
pairs are emitted, each with LocalType = 4 and LocalCount ≤ 255.

### 31.4.3 Calling Mechanisms

Functions can be called using several opcodes:

- **`call` (0x30):** The general calling opcode. Arguments must be
  pushed onto the stack before the call. The first operand is the
  function address, the second is the argument count, and the third
  is the store destination for the return value.

- **`callf` (0x160), `callfi` (0x161), `callfii` (0x162),
  `callfiii` (0x163):** Optimised calling opcodes for 0, 1, 2, or 3
  arguments respectively. Arguments are passed as inline operands,
  avoiding the need to push them onto the stack. The compiler prefers
  these for efficiency when the argument count is small and known at
  compile time.

- **`tailcall` (0x34):** A tail call that reuses the current stack
  frame. The function's return value is delivered directly to the
  original caller, saving one stack frame.

### 31.4.4 Exception Handling with Catch/Throw

The `catch` and `throw` opcodes provide a non-local return mechanism:

```inform6
! Catch/throw example:
@catch token ?label;
! ... code that may throw ...
.label;
! Execution continues here after catch or throw.
```

The `catch` opcode generates a unique token (stored in S1) and
branches to the label. If `throw` is later called with the same token,
the stack is unwound to the catch point and the thrown value is stored
as if `catch` had returned it.

## 31.5 Memory Operations

### 31.5.1 Array Access Patterns

The `aload`/`astore` family provides indexed access to arrays in
memory. The computed address formula depends on the element size:

| Opcode Pair | Element Size | Address Formula |
|---|---|---|
| `aload` / `astore` | 32-bit (4 bytes) | base + 4 × index |
| `aloads` / `astores` | 16-bit (2 bytes) | base + 2 × index |
| `aloadb` / `astoreb` | 8-bit (1 byte) | base + index |
| `aloadbit` / `astorebit` | 1 bit | byte (base + index/8), bit (index mod 8) |

In Inform source code, the `-->` operator compiles to `aload`/`astore`
and the `->` operator compiles to `aloadb`/`astoreb`.

### 31.5.2 Block Memory Operations

The `mzero` and `mcopy` opcodes operate on contiguous blocks of memory:

- `mzero` sets a region to all zeros. Useful for initialising arrays
  or clearing buffers.
- `mcopy` copies a block of bytes from one address to another. The
  operation handles overlapping regions correctly (similar to C's
  `memmove` rather than `memcpy`).

Both opcodes can only write to RAM (addresses ≥ RAMSTART).

### 31.5.3 Heap Management

The `malloc` and `mfree` opcodes provide dynamic memory allocation.
Memory is allocated from the extended memory region (above ENDMEM),
and the interpreter may need to grow the memory map to accommodate
allocations.

```inform6
! Allocate and free memory:
@malloc 1024 ptr;      ! Allocate 1024 bytes
if (ptr == 0) {
    ! Allocation failed
}
! ... use the memory ...
@mfree ptr;            ! Free the allocated block
```

The address passed to `mfree` must be exactly the address returned by
`malloc`. There is no realloc operation.

### 31.5.4 Memory Protection

The `protect` opcode designates a memory range that should not be
overwritten during save/restore operations. This is typically used to
preserve the current screen state or other transient data:

```inform6
@protect address length;
```

Only one region can be protected at a time. Calling `protect` again
replaces the previous protection. Calling `protect 0 0;` removes
protection entirely.

### 31.5.5 Memory Resizing

The `setmemsize` opcode requests that the interpreter grow (or shrink)
the total memory map:

```inform6
@setmemsize new_size result;
```

The new size must be a multiple of 256 and must be ≥ ENDMEM (the
original memory size). The result is 0 on success and 1 on failure.
This is primarily used internally by the heap manager.

## 31.6 Search Opcodes

The three search opcodes provide efficient searching of structured data
in memory. They are heavily used by the compiler's veneer routines for
object property table lookups. The compiler defines internal constants
for these opcodes: `linearsearch_gc` = 73, `binarysearch_gc` = 74,
`linkedsearch_gc` = 75.

### 31.6.1 Common Parameters

All three search opcodes share a set of common parameter conventions:

- **Key**: The value to search for, or (if `KeyIndirect` flag is set)
  the address of the key value in memory.
- **KeySize**: The size of the key in bytes (must be 1, 2, or 4).
- **Start**: The base address of the data structure(s) to search.
- **StructSize**: The size of each structure in bytes.
- **KeyOffset**: The byte offset within each structure where the key
  field is located.
- **Options**: A bitmask of option flags (see below).
- **Result**: The store destination for the search result.

### 31.6.2 Option Flags

| Flag | Value | Name | Description |
|---|---|---|---|
| 0x01 | 1 | `KeyIndirect` | The Key operand is an address pointing to the key value, not the value itself. |
| 0x02 | 2 | `ZeroKeyTerminates` | Stop searching when a structure with an all-zero key field is found (before comparing). |
| 0x04 | 4 | `ReturnIndex` | Return the index of the matching structure (0-based) instead of its address. If no match, return −1 (0xFFFFFFFF). |

### 31.6.3 linearsearch (0x0150)

```
linearsearch Key KeySize Start StructSize NumStructs KeyOffset Options Result
```

Searches sequentially through an array of structures starting at
`Start`. Each structure is `StructSize` bytes long. The search
compares the key field at `KeyOffset` within each structure against
`Key`. The search examines at most `NumStructs` structures. If
`NumStructs` is −1 (0xFFFFFFFF), the search continues until a match
is found or a zero-key terminator is encountered (the
`ZeroKeyTerminates` flag should typically be set in this case).

**Return value:** The address of the matching structure, or 0 if no
match is found. If `ReturnIndex` is set, returns the index or −1.

All three option flags (`KeyIndirect`, `ZeroKeyTerminates`,
`ReturnIndex`) are valid for `linearsearch`.

### 31.6.4 binarysearch (0x0151)

```
binarysearch Key KeySize Start StructSize NumStructs KeyOffset Options Result
```

Performs a binary search on an array of structures that must be sorted
in ascending order by key value. Keys are compared as big-endian
unsigned integers of the specified `KeySize`. The array must not
contain duplicate keys, and `NumStructs` must be the exact number of
structures (not −1).

**Return value:** The address of the matching structure, or 0 if no
match. If `ReturnIndex` is set, returns the index or −1.

Only `KeyIndirect` and `ReturnIndex` are valid for `binarysearch`.
The `ZeroKeyTerminates` flag is not supported.

The compiler's veneer uses `binarysearch` for property table lookups:

```inform6
! Example from the veneer (simplified):
@binarysearch id 2 otab 10 max 0 0 res;
```

### 31.6.5 linkedsearch (0x0152)

```
linkedsearch Key KeySize Start KeyOffset NextOffset Options Result
```

Searches a linked list of structures. Unlike the array-based search
opcodes, `linkedsearch` does not take `StructSize` or `NumStructs`
parameters. Instead, it takes `NextOffset`, the byte offset within
each structure of a 4-byte pointer to the next structure. A pointer
value of 0 marks the end of the list.

**Return value:** The address of the matching structure, or 0 if no
match.

Only `KeyIndirect` and `ZeroKeyTerminates` are valid for
`linkedsearch`. The `ReturnIndex` flag is not supported (since linked
lists do not have a natural index ordering).

### 31.6.6 Search Opcode Summary

| Feature | `linearsearch` | `binarysearch` | `linkedsearch` |
|---|---|---|---|
| Data structure | Array | Sorted array | Linked list |
| Operand count | 8 | 8 | 7 |
| KeyIndirect | ✓ | ✓ | ✓ |
| ZeroKeyTerminates | ✓ | ✗ | ✓ |
| ReturnIndex | ✓ | ✓ | ✗ |
| Unlimited count | ✓ (NumStructs = −1) | ✗ | N/A |
| Key comparison | Byte-level | Big-endian unsigned | Byte-level |
| Requires sorted data | No | Yes | No |

## 31.7 I/O Through Glk

This section describes how programs interact with the Glk I/O
library at the assembly level. For the architectural overview of the
Glk I/O layer, see §30.8.

### 31.7.1 The @glk Instruction

The `@glk` assembly instruction is the primary mechanism for calling
Glk API functions:

```inform6
@glk selector argcount result;
```

The arguments to the Glk function must be pushed onto the stack before
the `@glk` instruction is executed. Arguments are pushed in reverse
order so that the first argument is on top of the stack:

```inform6
! Call glk_put_char('A'):
@push 65;              ! Push the character 'A'
@glk $0080 1 result;   ! selector=$0080 (glk_put_char), 1 argument
```

For Glk functions that take no arguments:

```inform6
! Call glk_gestalt(gestalt_Version, 0):
@push 0;               ! Push second argument
@push 0;               ! Push first argument (gestalt_Version = 0)
@glk $0004 2 result;   ! selector=$0004 (glk_gestalt), 2 arguments
```

### 31.7.2 Glk Function Selectors

The Glk function selector is a numeric identifier for each Glk API
function. Common selectors include:

| Selector | Glk Function | Description |
|---|---|---|
| `$0004` | `glk_gestalt` | Query Glk capabilities. |
| `$0040` | `glk_window_iterate` | Iterate over windows. |
| `$0047` | `glk_window_open` | Open a new window. |
| `$002F` | `glk_window_get_size` | Get window dimensions. |
| `$0080` | `glk_put_char` | Output a single character. |
| `$0081` | `glk_put_char_stream` | Output a character to a stream. |
| `$0086` | `glk_put_buffer` | Output a buffer of characters. |
| `$00A0` | `glk_char_to_lower` | Convert character to lowercase. |
| `$00D0` | `glk_request_line_event` | Request line input. |
| `$00D2` | `glk_request_char_event` | Request character input. |
| `$00E0` | `glk_select` | Wait for and retrieve an event. |
| `$0160` | `glk_fileref_create_by_prompt` | Prompt user for a file. |

These selectors are defined in the `infglk.h` header file. The
complete set of selectors is documented in the Glk specification.

### 31.7.3 The Glk__Wrap Veneer Routine

When Inform source code calls `glk()` as a system function, the
compiler emits a call to the `Glk__Wrap` veneer routine rather than
an inline `@glk` instruction. This is because the high-level `glk()`
call uses Inform's standard calling convention (push all arguments
including the selector onto the stack), while the `@glk` opcode
requires the selector as an inline operand.

The `Glk__Wrap` veneer routine bridges these calling conventions:

1. It receives all arguments on the stack (selector + Glk arguments).
2. It pops the selector into a local variable.
3. It adjusts the argument count.
4. It calls `@glk` with the selector, count, and result.
5. It returns the result.

### 31.7.4 Setting the I/O System

Before any output can occur, the program must select an I/O system.
The standard library does this during initialisation, but programs
that bypass the library must do it manually:

```inform6
! Enable Glk I/O:
@setiosys 2 0;

! Check current I/O system:
@getiosys mode rock;
! mode = 2 (Glk), rock = 0
```

The `setiosys` opcode takes two operands: the mode number and a "rock"
value. For Glk mode, the rock is unused (conventionally 0). For filter
mode, the rock is the address of the filter function.

### 31.7.5 String Types

The `streamstr` opcode decodes and outputs strings from memory. Three
string formats are supported, identified by the first byte:

| Type Byte | Format | Description |
|---|---|---|
| 0xE0 | C-style | Null-terminated sequence of 8-bit characters. |
| 0xE1 | Huffman | Compressed string decoded using the string decoding table. |
| 0xE2 | Unicode | Three bytes of padding, then null-terminated 32-bit characters. |

The Inform compiler typically generates E1 (compressed) strings to
reduce game file size, using the Huffman table whose address is stored
in the header at offset 0x1C.

## 31.8 The `@` Assembly Notation in Inform 6

Inform 6 allows programmers to embed Glulx assembly instructions
directly in source code using the `@` prefix. This section describes
the syntax and conventions.

### 31.8.1 Basic Syntax

Assembly instructions follow the pattern:

```inform6
@opcode operand1 operand2 ... operandN;
```

Each operand can be:

- A **constant**: a decimal number, hex number (`$FF`), or binary
  number (`$$1010`).
- A **variable**: a local or global variable name.
- The **stack pointer**: `sp` (variable 0).
- A **label**: prefixed with `?` for branch targets.

Store destinations (the operand where a result is written) are simply
written as a variable name or `sp` — no special prefix is needed.

### 31.8.2 Branch Targets

For branch instructions, the last operand is a branch target,
specified as `?label`:

```inform6
@jz x ?is_zero;
! ... code for non-zero case ...
.is_zero;
! ... code for zero case ...
```

Two special branch targets generate return instructions:

- `?rtrue` — return 1 (true) from the current function.
- `?rfalse` — return 0 (false) from the current function.

```inform6
@jeq a b ?rtrue;    ! Return true if a == b
```

### 31.8.3 Common Assembly Patterns

**Copying values:**

```inform6
@copy source destination;  ! Copy a 32-bit value
@copy sp temp;             ! Pop stack into temp
@copy value sp;            ! Push value onto stack
```

**Using the synthetic pull/push macros:**

```inform6
@pull temp;                ! Equivalent to @copy sp temp;
@push value;               ! Equivalent to @copy value sp;
```

**Calling functions:**

```inform6
! Call with inline arguments (preferred for 0-3 args):
@callfi MyFunc 42 result;        ! Call MyFunc(42), store result
@callfii MyFunc 1 2 result;      ! Call MyFunc(1, 2)
@callfiii MyFunc 1 2 3 result;   ! Call MyFunc(1, 2, 3)

! General call with stack arguments:
@push arg3;
@push arg2;
@push arg1;
@call func 3 result;             ! Call func(arg1, arg2, arg3)
```

**Array access:**

```inform6
@aload base index result;       ! result = base-->index
@aloadb base index result;      ! result = base->index
@astore base index value;       ! base-->index = value
@astoreb base index value;      ! base->index = value
```

**Glk calls:**

```inform6
! Put a character (push args in reverse order):
@push 65;                        ! Character 'A'
@glk $0080 1 result;            ! glk_put_char('A')

! Open a window (5 arguments):
@push 0;                         ! rock
@push 0;                         ! wintype (pair)
@push 0;                         ! method
@push 0;                         ! size
@push 0;                         ! split
@glk $0023 5 win;               ! glk_window_open(...)
```

**I/O setup:**

```inform6
@setiosys 2 0;                   ! Enable Glk I/O
```

**Gestalt queries:**

```inform6
@gestalt 11 0 result;            ! Check float support
if (result) {
    ! Floating-point opcodes are available
}

@gestalt 0 0 version;            ! Get Glulx VM version
```

**Float operations:**

```inform6
@numtof 42 fval;                 ! Convert integer 42 to float
@fadd fval $3F800000 fval;       ! fval = fval + 1.0
@fmul fval fval fsquared;        ! fsquared = fval * fval
@ftonumz fsquared ival;          ! Convert float to integer
```

**Stack manipulation:**

```inform6
@stkcount n;                     ! How many values on the stack?
@stkpeek 0 top;                  ! Read top value without popping
@stkswap;                        ! Swap top two values
@stkroll 3 1;                    ! Rotate top 3 values by 1 position
```

### 31.8.4 Custom Opcode Syntax

If the compiler does not recognise an opcode name, it can still
assemble the instruction using a numeric opcode specification. This
allows programs to use opcodes that the compiler does not have built-in
support for:

```inform6
@"1:130" selector argcount result;
```

The format is `@"S:NNNN"` where `S` is the operand count category and
`NNNN` is the opcode number in hexadecimal. This is primarily useful
for experimenting with new or interpreter-specific opcodes.

### 31.8.5 Differences from Z-Machine Assembly

Programmers familiar with Z-machine assembly (`@` notation in Z-code
mode) should note these key differences when writing Glulx assembly:

| Feature | Z-Machine | Glulx |
|---|---|---|
| Opcode names | Z-machine mnemonics (e.g., `add`, `je`) | Glulx mnemonics (e.g., `add`, `jeq`) |
| Variable encoding | Variable number (0–255) | Modes: stack, local offset, RAM offset |
| Store operand | Written as `-> variable` | Written as just `variable` or `sp` |
| Branch operand | Written as `?label` or `?~label` | Written as `?label` (no negation) |
| Packed addresses | Used for routines/strings | Not used — direct byte addresses |
| I/O opcodes | Many (print, read, etc.) | Few (`streamchar`, `streamnum`, `streamstr`) — rest via `@glk` |
| String operand | In-line Z-encoded text | Not supported — use `streamstr` with address |
