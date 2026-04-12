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

# Chapter 29: Z-Machine Instruction Set (Assembly)

**[Z-machine]** This chapter documents the Z-machine instruction set as
supported by the compiler. It covers the binary encoding of instructions,
the complete set of opcodes organised by category, and the assembly
language syntax that allows programmers to embed Z-machine instructions
directly in their source code.

## 29.1 Encoding Format: Opcodes, Operand Types

Every Z-machine instruction consists of an opcode (identifying the
operation) followed by zero or more operands, and optionally a store
destination and/or a branch offset. The encoding of each part depends
on the instruction's **operand count category**.

### 29.1.1 Operand Count Categories

The Z-machine groups instructions into categories based on the number
of operands they accept. The compiler defines these internally as:

| Category | Constant | Operand Count | Description |
|----------|----------|---------------|-------------|
| 2OP | `TWO` | Exactly 2 (sometimes 1–4) | Two-operand instructions |
| VAR | `VAR` | 0 to 4 | Variable-operand instructions |
| VAR_LONG | `VAR_LONG` | 0 to 8 | Long variable-operand instructions |
| 1OP | `ONE` | Exactly 1 | One-operand instructions |
| 0OP | `ZERO` | Exactly 0 | Zero-operand instructions |
| EXT | `EXT` | 0 to 4 | Extended instructions |
| EXT_LONG | `EXT_LONG` | 0 to 8 | Long extended instructions |

### 29.1.2 Opcode Byte Encoding

The first byte (or bytes) of an instruction encode the opcode and
sometimes the operand types. The compiler writes the opcode byte in
`assemblez_instruction()` using the following scheme:

**2OP short form** (when both operands are small constants or variables):

```
  Bit:  7  6  5  4  3  2  1  0
       [0] [op1type] [---opcode---]
```

- Bit 7: always 0
- Bit 6: first operand type (0 = small constant, 1 = variable)
- Bit 5: second operand type (0 = small constant, 1 = variable)
- Bits 4–0: opcode number (0x00–0x1F)

If either operand is a **long constant** (2 bytes), the instruction
is automatically promoted to VAR form by adding `0xC0` to the opcode
byte and including a types byte.

**1OP form:**

```
  Bit:  7  6  5  4  3  2  1  0
       [1] [0] [type] [--opcode--]
```

- Bits 7–6: `10` (identifies 1OP)
- Bits 5–4: operand type
- Bits 3–0: opcode number (0x00–0x0F)

**0OP form:**

```
  Bit:  7  6  5  4  3  2  1  0
       [1] [0] [1] [1] [--opcode--]
```

- Bits 7–4: `1011` (identifies 0OP; top bits = `0xB0`)
- Bits 3–0: opcode number (0x00–0x0F)

**VAR form:**

```
  Byte 1:  [1] [1] [---opcode---]     (top bits = 0xC0)
  Byte 2:  [types for up to 4 operands]
```

- Bits 7–6 of byte 1: `11` (identifies VAR)
- Bits 4–0 of byte 1: opcode number (0x00–0x1F)
- Byte 2: operand types, 2 bits each for up to 4 operands

**VAR_LONG form:** Same as VAR but with **two** type bytes, supporting
up to 8 operands.

**EXT form:**

```
  Byte 1:  0xBE (extended prefix)
  Byte 2:  [opcode number]
  Byte 3:  [types for up to 4 operands]
```

EXT instructions always begin with the prefix byte `0xBE`, followed
by the opcode number and a types byte.

**EXT_LONG form:** Same as EXT but with **two** type bytes, supporting
up to 8 operands.

### 29.1.3 Operand Type Encoding

Each operand's type is encoded as a 2-bit value:

| Bits | Type | Size | Description |
|------|------|------|-------------|
| `00` | Large constant | 2 bytes | A 16-bit constant value |
| `01` | Small constant | 1 byte | An 8-bit constant value (0–255) |
| `10` | Variable | 1 byte | A variable number (0 = stack, 1–15 = locals, 16–255 = globals) |
| `11` | Omitted | 0 bytes | No operand in this position |

In types bytes, operand types are packed from the most significant bits
downward. For a VAR instruction with a single types byte:

```
  Bit:  7  6  5  4  3  2  1  0
       [op1 ] [op2 ] [op3 ] [op4 ]
```

Unused operand positions are filled with `11` (omitted). For a
VAR_LONG instruction, a second types byte provides positions for
operands 5–8.

### 29.1.4 Store Destinations

Instructions that produce a result (flagged with `St` in the opcode
table) include a **store byte** after the operands. This byte gives
the variable number where the result is stored:

- 0: push onto the stack
- 1–15: local variable
- 16–255: global variable

### 29.1.5 Branch Offsets

Instructions that can branch (flagged with `Br`) include a branch
specification after the operands (and after any store byte).

**Short branch (1 byte):**

```
  Bit:  7     6    5  4  3  2  1  0
       [pol] [1]  [------offset------]
```

- Bit 7: branch polarity (1 = branch on true, 0 = branch on false)
- Bit 6: 1 indicates short form
- Bits 5–0: 6-bit offset, or special values:
  - 0 = return false from current routine
  - 1 = return true from current routine
  - 2–63 = branch forward by (offset − 2) bytes

**Long branch (2 bytes):**

```
  Byte 1:  [pol] [0] [-----high 6 bits-----]
  Byte 2:  [----------low 8 bits-----------]
```

- Bit 7 of byte 1: branch polarity
- Bit 6 of byte 1: 0 indicates long form
- Remaining 14 bits: signed offset from the address after the branch
  bytes (offset − 2 gives the actual displacement)

## 29.2 Instruction Categories and Flags

### 29.2.1 Opcode Flags

Each opcode in the compiler's tables carries a set of flags describing
its behaviour:

| Flag | Value | Meaning |
|------|-------|---------|
| `St` | 1 | Instruction stores a result |
| `Br` | 2 | Instruction has a branch |
| `Rf` | 4 | Execution never continues past this instruction (return or unconditional jump) |
| `St2` | 8 | Second-to-last operand is a store destination (Glulx only) |

An instruction may combine flags. For example, `get_sibling` has flags
`St+Br` (3), meaning it both stores a result and branches.

### 29.2.2 Special Operand Rules

Some opcodes have unusual operand assembly rules, indicated by the
`op_rules` field:

| Rule | Value | Meaning |
|------|-------|---------|
| `VARIAB` | 1 | First operand is a variable number, assembled as a small constant |
| `TEXT` | 2 | One text operand, Z-encoded inline in the instruction stream |
| `LABEL` | 3 | One operand is a label, assembled as a long constant offset |
| `CALL` | 4 | First operand is a routine name, assembled as a packed address |

**`VARIAB` rule:** Used by opcodes like `inc`, `dec`, `store`, `load`,
`pull`, `inc_chk`, and `dec_chk`. The first operand names a variable
(given as a variable name in assembly), but it is encoded as a small
constant holding the variable's number rather than as a variable
reference.

**`TEXT` rule:** Used only by `print` and `print_ret`. The operand is a
double-quoted string that is Z-encoded directly into the instruction
stream.

**`LABEL` rule:** Used only by `jump`. The operand is a label name,
encoded as a 2-byte offset.

**`CALL` rule:** Used by all call instructions (`call`, `call_vs`,
`call_2s`, `call_1s`, `call_vn`, `call_2n`, `call_1n`, `call_vs2`,
`call_vn2`). The first operand is a routine name that is assembled as
a packed address.

### 29.2.3 Flags 2 Requirements

Some opcodes set a bit in the story file's Flags 2 header word when
they are used. This informs the interpreter that the game requires
certain features. The `flags2_set` field in the opcode table specifies
which bit to set:

| Bit | Feature | Opcodes |
|-----|---------|---------|
| 3 | Pictures | `draw_picture`, `picture_data`, `erase_picture`, `picture_table` |
| 4 | Undo | `save_undo`, `restore_undo` |
| 5 | Mouse | `read_mouse`, `mouse_window` |
| 6 | Colours | `set_colour` |
| 7 | Sound effects | `sound_effect` |
| 8 | Menus | `make_menu` |

## 29.3 Complete Opcode Listing

The following sections list every Z-machine opcode known to the Inform
6.44 compiler, organised by the version in which each opcode was
introduced. The information is taken directly from the compiler's internal
opcode table.

Each entry shows:
- **Name**: the opcode's assembly name
- **Code**: the opcode number within its category
- **Category**: the operand count category (2OP, 1OP, 0OP, VAR, EXT)
- **Flags**: St (store), Br (branch), Rf (return/never continues)
- **Rules**: any special operand rules

### 29.3.1 Version 3 Opcodes

These opcodes are available in all Z-machine versions (3 and later).

**2OP instructions:**

| Name | Code | Flags | Rules | Description |
|------|------|-------|-------|-------------|
| `je` | `0x01` | Br | — | Jump if equal (up to 4 operands in VAR form) |
| `jl` | `0x02` | Br | — | Jump if less than (signed) |
| `jg` | `0x03` | Br | — | Jump if greater than (signed) |
| `dec_chk` | `0x04` | Br | VARIAB | Decrement variable and branch if less than value |
| `inc_chk` | `0x05` | Br | VARIAB | Increment variable and branch if greater than value |
| `jin` | `0x06` | Br | — | Jump if object *a* is a direct child of object *b* |
| `test` | `0x07` | Br | — | Branch if all bits in *bitmap* are set in *flags* |
| `or` | `0x08` | St | — | Bitwise OR |
| `and` | `0x09` | St | — | Bitwise AND |
| `test_attr` | `0x0A` | Br | — | Branch if object has attribute |
| `set_attr` | `0x0B` | — | — | Set attribute on object |
| `clear_attr` | `0x0C` | — | — | Clear attribute on object |
| `store` | `0x0D` | — | VARIAB | Store value into variable |
| `insert_obj` | `0x0E` | — | — | Move object to become first child of destination |
| `loadw` | `0x0F` | St | — | Load word from array: `array-->index` |
| `loadb` | `0x10` | St | — | Load byte from array: `array->index` |
| `get_prop` | `0x11` | St | — | Get property value of object |
| `get_prop_addr` | `0x12` | St | — | Get byte address of property data |
| `get_next_prop` | `0x13` | St | — | Get number of next property after given one |
| `add` | `0x14` | St | — | Signed addition |
| `sub` | `0x15` | St | — | Signed subtraction |
| `mul` | `0x16` | St | — | Signed multiplication |
| `div` | `0x17` | St | — | Signed integer division |
| `mod` | `0x18` | St | — | Signed remainder |

**VAR instructions:**

| Name | Code | Flags | Rules | Description |
|------|------|-------|-------|-------------|
| `call` | `0x20` | St | CALL | Call routine with 0–3 arguments (v3 only name; `call_vs` from v4) |
| `storew` | `0x21` | — | — | Store word into array: `array-->index = value` |
| `storeb` | `0x22` | — | — | Store byte into array: `array->index = value` |
| `put_prop` | `0x23` | — | — | Set property value of object |
| `read` | `0x24` | — | — | Read a line of input (v3 "sread" form: no store) |
| `print_char` | `0x25` | — | — | Print ZSCII character |
| `print_num` | `0x26` | — | — | Print signed number in decimal |
| `random` | `0x27` | St | — | Generate random number, or seed the generator |
| `push` | `0x28` | — | — | Push value onto the stack |
| `pull` | `0x29` | — | VARIAB | Pull value from stack into variable (v3–5; changes in v6) |
| `split_window` | `0x2A` | — | — | Split screen into upper and lower windows |
| `set_window` | `0x2B` | — | — | Select output window |
| `output_stream` | `0x33` | — | — | Enable or disable an output stream |
| `input_stream` | `0x34` | — | — | Select an input stream |
| `sound_effect` | `0x35` | — | — | Play or stop a sound effect (sets Flags 2, bit 7) |

**1OP instructions:**

| Name | Code | Flags | Rules | Description |
|------|------|-------|-------|-------------|
| `jz` | `0x00` | Br | — | Branch if value is zero |
| `get_sibling` | `0x01` | St+Br | — | Get sibling of object; branch if it exists |
| `get_child` | `0x02` | St+Br | — | Get first child of object; branch if it exists |
| `get_parent` | `0x03` | St | — | Get parent of object |
| `get_prop_len` | `0x04` | St | — | Get length of property data in bytes |
| `inc` | `0x05` | — | VARIAB | Increment variable |
| `dec` | `0x06` | — | VARIAB | Decrement variable |
| `print_addr` | `0x07` | — | — | Print string at byte address |
| `remove_obj` | `0x09` | — | — | Remove object from object tree |
| `print_obj` | `0x0A` | — | — | Print short name of object |
| `ret` | `0x0B` | Rf | — | Return value from current routine |
| `jump` | `0x0C` | Rf | LABEL | Unconditional jump to label |
| `print_paddr` | `0x0D` | — | — | Print string at packed address |
| `load` | `0x0E` | St | VARIAB | Load value of variable |
| `not` | `0x0F` | St | — | Bitwise NOT (v3 only; changes in v4+) |

**0OP instructions:**

| Name | Code | Flags | Rules | Description |
|------|------|-------|-------|-------------|
| `rtrue` | `0x00` | Rf | — | Return true (1) from current routine |
| `rfalse` | `0x01` | Rf | — | Return false (0) from current routine |
| `print` | `0x02` | — | TEXT | Print inline Z-encoded string |
| `print_ret` | `0x03` | Rf | TEXT | Print inline string, print newline, return true |
| `nop` | `0x04` | — | — | No operation |
| `save` | `0x05` | Br | — | Save game state (v3 only; changes in v4+) |
| `restore` | `0x06` | Br | — | Restore game state (v3 only; changes in v4+) |
| `restart` | `0x07` | — | — | Restart the game |
| `ret_popped` | `0x08` | Rf | — | Return top-of-stack value |
| `pop` | `0x09` | — | — | Discard top-of-stack value (v3–4 only) |
| `quit` | `0x0A` | Rf | — | Terminate the program |
| `new_line` | `0x0B` | — | — | Print a newline character |
| `show_status` | `0x0C` | — | — | Update the status line (v3 only) |
| `verify` | `0x0D` | Br | — | Verify the story file checksum |

### 29.3.2 Version 4 Opcodes

These opcodes are available from version 4 onwards.

**2OP instructions:**

| Name | Code | Flags | Rules | Description |
|------|------|-------|-------|-------------|
| `call_2s` | `0x19` | St | CALL | Call routine with 1 argument, store result |

**VAR instructions:**

| Name | Code | Flags | Rules | Description |
|------|------|-------|-------|-------------|
| `call_vs` | `0x20` | St | CALL | Call routine with 0–3 arguments, store result (replaces `call`) |
| `read` | `0x24` | St | — | Read input (v4+ "aread" form: stores result) |
| `call_vs2` | `0x2C` | St | CALL | Call routine with 0–7 arguments, store result (VAR_LONG) |
| `erase_window` | `0x2D` | — | — | Erase a window |
| `erase_line` | `0x2E` | — | — | Erase to end of line |
| `set_cursor` | `0x2F` | — | — | Set cursor position |
| `get_cursor` | `0x30` | — | — | Get cursor position into a table |
| `set_text_style` | `0x31` | — | — | Set text style (bold, italic, fixed-width, etc.) |
| `buffer_mode` | `0x32` | — | — | Enable or disable output buffering |
| `read_char` | `0x36` | St | — | Read a single keypress |
| `scan_table` | `0x37` | St+Br | — | Search a table for a value |

**1OP instructions:**

| Name | Code | Flags | Rules | Description |
|------|------|-------|-------|-------------|
| `call_1s` | `0x08` | St | CALL | Call routine with 0 arguments, store result |

### 29.3.3 Version 5 Opcodes

These opcodes are available from version 5 onwards (and also in versions
7 and 8, which use instruction set 5).

**2OP instructions:**

| Name | Code | Flags | Rules | Description |
|------|------|-------|-------|-------------|
| `call_2n` | `0x1A` | — | CALL | Call routine with 1 argument, discard result |
| `set_colour` | `0x1B` | — | — | Set foreground and background colours (sets Flags 2, bit 6) |
| `throw` | `0x1C` | — | — | Throw a value to a catch frame |

**VAR instructions:**

| Name | Code | Flags | Rules | Description |
|------|------|-------|-------|-------------|
| `call_vn` | `0x39` | — | CALL | Call routine with 0–3 arguments, discard result |
| `call_vn2` | `0x3A` | — | CALL | Call routine with 0–7 arguments, discard result (VAR_LONG) |
| `tokenise` | `0x3B` | — | — | Tokenise text buffer into parse buffer |
| `encode_text` | `0x3C` | — | — | Encode text to dictionary form |
| `copy_table` | `0x3D` | — | — | Copy or zero a table |
| `print_table` | `0x3E` | — | — | Print a rectangular area of text |
| `check_arg_count` | `0x3F` | Br | — | Branch if argument *n* was provided to current routine |

**1OP instructions:**

| Name | Code | Flags | Rules | Description |
|------|------|-------|-------|-------------|
| `call_1n` | `0x0F` | — | CALL | Call routine with 0 arguments, discard result |

**0OP instructions:**

| Name | Code | Flags | Rules | Description |
|------|------|-------|-------|-------------|
| `catch` | `0x09` | St | — | Catch the current stack frame for later `throw` |
| `piracy` | `0x0F` | Br | — | Branch if story file is genuine (always branches in practice) |

**EXT instructions:**

| Name | Code | Flags | Rules | Description |
|------|------|-------|-------|-------------|
| `log_shift` | `0x02` | St | — | Logical shift (positive = left, negative = right) |
| `art_shift` | `0x03` | St | — | Arithmetic shift (sign-extending right shift) |
| `set_font` | `0x04` | St | — | Set font and return previous font number |
| `save_undo` | `0x09` | St | — | Save undo state (sets Flags 2, bit 4) |
| `restore_undo` | `0x0A` | St | — | Restore undo state (sets Flags 2, bit 4) |

### 29.3.4 Version 6 Opcodes

These opcodes are available only in version 6 (the graphical version).
All are in the EXT category.

| Name | Code | Flags | Rules | Flags2 | Description |
|------|------|-------|-------|--------|-------------|
| `draw_picture` | `0x05` | — | — | 3 | Draw a picture resource |
| `picture_data` | `0x06` | Br | — | 3 | Get picture dimensions; branch if picture exists |
| `erase_picture` | `0x07` | — | — | 3 | Erase a picture area |
| `set_margins` | `0x08` | — | — | — | Set left and right margins |
| `move_window` | `0x10` | — | — | — | Move a window |
| `window_size` | `0x11` | — | — | — | Resize a window |
| `window_style` | `0x12` | — | — | — | Set window style flags |
| `get_wind_prop` | `0x13` | St | — | — | Get a window property |
| `scroll_window` | `0x14` | — | — | — | Scroll a window |
| `pop_stack` | `0x15` | — | — | — | Pop values from a stack |
| `read_mouse` | `0x16` | — | — | 5 | Read mouse position and buttons |
| `mouse_window` | `0x17` | — | — | 5 | Set window for mouse input |
| `push_stack` | `0x18` | Br | — | — | Push a value onto a stack; branch on success |
| `put_wind_prop` | `0x19` | — | — | — | Set a window property |
| `print_form` | `0x1A` | — | — | — | Print a formatted table |
| `make_menu` | `0x1B` | Br | — | 8 | Create a menu; branch on success |
| `picture_table` | `0x1C` | — | — | 3 | Preload picture resources |
| `buffer_screen` | `0x1D` | St | — | — | Control screen buffering |

### 29.3.5 Z-Machine Specification Standard 1.0 Opcodes

These EXT opcodes were introduced in version 1.0 of the Z-Machine
Specification Standard and are available from instruction set 5 onwards:

| Name | Code | Flags | Description |
|------|------|-------|-------------|
| `print_unicode` | `0x0B` | — | Print a Unicode character |
| `check_unicode` | `0x0C` | St | Check if a Unicode character can be printed and read |

### 29.3.6 Z-Machine Specification Standard 1.1 Opcodes

These EXT opcodes were introduced in version 1.1 of the specification:

| Name | Code | Flags | Description |
|------|------|-------|-------------|
| `set_true_colour` | `0x0D` | — | Set foreground and background using true-colour values (v5+) |
| `buffer_screen` | `0x1D` | St | Control screen buffering (v6 only) |

### 29.3.7 Version-Variant Opcodes

Several opcodes change their encoding or behaviour between Z-machine
versions. The compiler tracks these in the `extension_table_z[]` array.
When the compiler encounters one of these opcodes, it selects the
appropriate form based on the target version.

**`not` — Bitwise NOT:**

| Version | Category | Code | Flags |
|---------|----------|------|-------|
| 3 | 1OP | `0x0F` | St |
| 4 | 1OP | `0x0F` | St |
| 5+ | VAR | `0x38` | St |

In version 3, `not` is a 1OP instruction. In version 4 it remains 1OP
but with store. From version 5 onwards, it becomes a VAR instruction
at opcode `0x38`.

**`save` — Save game state:**

| Version | Category | Code | Flags |
|---------|----------|------|-------|
| 3 | 0OP | `0x05` | Br |
| 4 | 0OP | `0x05` | St |
| 5+ | EXT | `0x00` | St |

In version 3, `save` is a 0OP instruction that branches on
success/failure. In version 4 it becomes a 0OP that stores a result.
From version 5 onwards, it is an EXT instruction that stores a result.

**`restore` — Restore game state:**

| Version | Category | Code | Flags |
|---------|----------|------|-------|
| 3 | 0OP | `0x06` | Br |
| 4 | 0OP | `0x06` | St |
| 5+ | EXT | `0x01` | St |

Follows the same pattern as `save`.

**`pull` — Pull from stack:**

| Version | Category | Code | Flags | Rules |
|---------|----------|------|-------|-------|
| 3–5 | VAR | `0x29` | — | VARIAB |
| 6 | VAR | `0x29` | St | — |

In versions 3–5, `pull` takes a variable reference as its operand
(the variable to receive the popped value). In version 6, it stores
the result using the standard store mechanism.

## 29.4 The `@` Assembly Notation in Inform 6

Inform 6 allows Z-machine instructions to be embedded directly in
source code using the `@` prefix. The assembly statement begins with
`@`, followed by the opcode name and its operands, and terminates with
a semicolon.

### 29.4.1 Basic Syntax

The general form of an assembly statement is:

```inform6
@opcode operand1 operand2 ... ;
```

For example:

```inform6
@add 3 4 -> result;
@jz x ?is_zero;
@print "Hello, world!";
```

The operands are separated by spaces. There are no commas between
operands.

### 29.4.2 Store Destinations

For opcodes that store a result (those with the `St` flag), the store
destination is specified with `->` followed by a variable name or `sp`
(to push the result onto the stack):

```inform6
@add x y -> result;        ! Store sum in 'result'
@random 100 -> sp;          ! Push random number onto stack
@get_parent obj -> parent;  ! Store parent object number
```

If no explicit `->` is given, the compiler automatically uses the last
operand as the store destination (which must be a variable). Using
explicit `->` is recommended for clarity.

### 29.4.3 Branch Destinations

For opcodes that branch (those with the `Br` flag), the branch
destination is specified with `?` (branch on true) or `?~` (branch on
false), followed by a label name:

```inform6
@je x y ?equal;              ! Branch to 'equal' if x == y
@test_attr obj a ?~no_attr;  ! Branch to 'no_attr' if object lacks attribute
@jz count ?done;             ! Branch to 'done' if count is zero
```

Two special branch targets are available:

```inform6
@je x y ?rtrue;              ! Return true if x == y
@jz count ?rfalse;           ! Return false if count is zero
```

`?rtrue` and `?rfalse` cause the routine to return true (1) or false (0)
respectively, rather than branching to a label. These are encoded
efficiently as special values in the branch offset.

### 29.4.4 Labels

Assembly labels are defined using the standard Inform label syntax (a
dot followed by the label name) and referenced in branch operands:

```inform6
[ MyRoutine x;
    @jz x ?is_zero;
    @print_num x;
    @new_line;
    @rtrue;
  .is_zero;
    @print "Zero!";
    @new_line;
    @rfalse;
];
```

For the `jump` opcode, the operand is a label name (not prefixed with
`?`):

```inform6
@jump done;
...
.done;
```

### 29.4.5 Text Operands

The `print` and `print_ret` opcodes take a double-quoted string as their
operand. The string is Z-encoded directly into the instruction stream:

```inform6
@print "Hello, world!";
@print_ret "Goodbye!";    ! Also prints newline and returns true
```

### 29.4.6 Indirect Addressing

Opcodes with the `VARIAB` operand rule (such as `inc`, `dec`, `store`,
`load`, `pull`, `inc_chk`, `dec_chk`) take a variable reference as
their first operand. The variable name is assembled as a small constant
holding the variable's number.

To use *indirect* addressing — where the variable number is itself stored
in another variable — enclose the first operand in square brackets:

```inform6
@inc [x];       ! Increment the variable whose number is stored in x
@load [x] -> y; ! Load the value of the variable numbered by x into y
```

Without brackets, a variable name is treated as a literal reference to
that specific variable.

### 29.4.7 The `sp` Variable

In assembly statements, the name `sp` refers to the stack pointer
(variable number 0). It can be used as an operand to push or pop
values:

```inform6
@push value;        ! Same as: push value onto stack
@pull sp;           ! Pop stack (discarding the value)
@add x y -> sp;     ! Push the sum onto the stack
@storew table 0 sp; ! Store top-of-stack into table entry
```

## 29.5 Custom Opcode Syntax

The Inform compiler allows emission of arbitrary Z-machine opcodes
that may not be in the compiler's built-in opcode table. This is done
using a double-quoted opcode specification string after `@`.

### 29.5.1 Syntax

```inform6
@"TYPE:NUMBERflags" operands... ;
```

Where:
- **TYPE** is one of: `0OP`, `1OP`, `2OP`, `VAR`, `EXT`, `VAR_LONG`,
  `EXT_LONG`
- **NUMBER** is the opcode number (decimal) within the type's range
- **flags** are optional single characters:
  - `B` — the opcode has a branch
  - `S` — the opcode stores a result
  - `T` — the opcode takes a text operand
  - `I` — the first operand uses indirect addressing (variable number)
  - `F`*n* — set bit *n* in Flags 2

### 29.5.2 Opcode Number Ranges

Each type has a valid range of opcode numbers:

| Type | Minimum | Maximum |
|------|---------|---------|
| `0OP` | 0 | 15 |
| `1OP` | 0 | 15 |
| `2OP` | 0 | 31 |
| `VAR` | 32 | 63 |
| `VAR_LONG` | 32 | 63 |
| `EXT` | 0 | 255 |
| `EXT_LONG` | 0 | 255 |

### 29.5.3 Examples

```inform6
! Emit a VAR opcode number 40 that stores a result:
@"VAR:40S" arg1 arg2 -> result;

! Emit an EXT opcode number 20 that branches:
@"EXT:20B" value ?label;

! Emit a 2OP opcode number 30 with no special flags:
@"2OP:30" x y;

! Emit an EXT opcode that sets Flags 2 bit 9:
@"EXT:100F9" arg1 arg2;
```

This feature is useful for experimenting with opcodes defined in newer
versions of the Z-Machine Specification that the compiler may not yet
have built-in support for, or for testing custom interpreter extensions.

## 29.6 Inline Byte and Word Assembly

Beyond structured opcode assembly, Inform 6 allows raw bytes or words
to be emitted directly into the code stream using `@->` and `@-->`.

### 29.6.1 Byte Assembly (`@->`)

```inform6
@-> value1 value2 value3 ... ;
```

Each value is truncated to 8 bits and emitted as a raw byte into the
code area. Values must be known constants (no backpatchable references
are allowed). The compiler issues a warning if any value is 256 or
greater.

Example:

```inform6
! Emit three raw bytes into the code stream:
@-> $BE $05 $FF;
```

### 29.6.2 Word Assembly (`@-->`)

```inform6
@--> value1 value2 ... ;
```

Each value is emitted as a 2-byte big-endian word. Unlike byte assembly,
word values may contain backpatchable references (such as routine
addresses or string addresses), because the marker is recorded on the
high byte.

Example:

```inform6
! Emit two raw words into the code stream:
@--> $1234 MyRoutine;
```

### 29.6.3 Use Cases

Inline byte and word assembly is rarely needed in ordinary programming.
Potential uses include:

- Emitting raw data that must appear at a specific position in the code
  area
- Constructing instruction sequences that the compiler's assembly parser
  does not directly support
- Embedding lookup tables in the code stream

**Warning:** Inline assembly is not checked for correctness. Emitting
invalid bytes can produce a story file that crashes the interpreter.
These features should be used only when no alternative exists.

## 29.7 Common Assembly Patterns

This section illustrates common patterns for using Z-machine assembly
in Inform 6 code. Assembly is useful for performance-critical code,
direct hardware access, or operations that cannot be expressed in
high-level Inform 6.

### 29.7.1 Stack Manipulation

```inform6
@push value;          ! Push a value onto the stack
@pull variable;       ! Pop the stack into a variable

! Using the stack as a temporary:
@push x;
! ... do something that modifies x ...
@pull x;              ! Restore x from the stack
```

### 29.7.2 Object Tree Traversal

```inform6
! Get the parent, child, and sibling of an object:
@get_parent obj -> p;
@get_child obj -> c ?has_children;
@get_sibling obj -> s ?has_sibling;

! Traverse all children of an object:
[ PrintChildren obj child;
    @get_child obj -> child ?~done;
  .loop;
    @print_obj child;
    @new_line;
    @get_sibling child -> child ?loop;
  .done;
];
```

### 29.7.3 Direct Property Access

```inform6
! Get a property value:
@get_prop obj prop_num -> value;

! Set a property value:
@put_prop obj prop_num new_value;

! Get the address and length of property data:
@get_prop_addr obj prop_num -> addr;
@get_prop_len addr -> len;
```

### 29.7.4 Memory Access

```inform6
! Read from and write to memory:
@loadw base index -> value;     ! value = base-->index
@loadb base index -> value;     ! value = base->index
@storew base index new_value;   ! base-->index = new_value
@storeb base index new_value;   ! base->index = new_value
```

### 29.7.5 Conditional Branching

```inform6
! Compare two values:
@je x y ?are_equal;
@jl x y ?x_is_less;
@jg x y ?x_is_greater;

! Test a value against zero:
@jz x ?is_zero;

! Test attributes:
@test_attr obj light ?has_light;

! Test if object is a child of another:
@jin child parent ?is_child;

! Test a bitmask:
@test flags mask ?all_bits_set;
```

### 29.7.6 Arithmetic

```inform6
@add x y -> sum;
@sub x y -> diff;
@mul x y -> product;
@div x y -> quotient;
@mod x y -> remainder;

! Increment and decrement with branch:
@inc_chk score 100 ?reached_max;
@dec_chk lives 0 ?game_over;

! Bitwise operations:
@or x y -> result;
@and x y -> result;
@not x -> result;         ! In v5+, this is: @"VAR:56S" x -> result;

! Shifting (v5+):
@log_shift x 2 -> result;    ! Logical shift left by 2
@log_shift x -3 -> result;   ! Logical shift right by 3
@art_shift x -1 -> result;   ! Arithmetic shift right by 1
```

### 29.7.7 I/O Operations

```inform6
! Output:
@print "Hello!";
@print_char 65;           ! Print 'A'
@print_num 42;            ! Print "42"
@print_obj obj;           ! Print object's short name
@new_line;

! Input (v5+):
@read text_buffer parse_buffer -> result;
@read_char 1 -> ch;       ! Read a single keypress

! Unicode (ZSpec 1.0+):
@print_unicode $00e9;     ! Print 'é' by Unicode value
```

### 29.7.8 Calling Routines

The Z-machine provides several calling conventions that differ in
whether they store the return value or discard it, and in how many
arguments they accept:

```inform6
! Calls that store the result:
@call_vs routine arg1 arg2 arg3 -> result;  ! 0–3 args (VAR)
@call_vs2 routine a1 a2 a3 a4 -> result;   ! 0–7 args (VAR_LONG)
@call_2s routine arg1 -> result;            ! exactly 1 arg (2OP)
@call_1s routine -> result;                 ! 0 args (1OP)

! Calls that discard the result:
@call_vn routine arg1 arg2;                 ! 0–3 args (VAR)
@call_vn2 routine a1 a2 a3 a4 a5;          ! 0–7 args (VAR_LONG)
@call_2n routine arg1;                      ! exactly 1 arg (2OP)
@call_1n routine;                           ! 0 args (1OP)
```

The "s" variants (store) are used when the return value is needed. The
"n" variants (no-store) are used when the return value can be discarded,
saving one byte in the instruction encoding.

### 29.7.9 Save and Restore

```inform6
! Save/restore undo (v5+):
@save_undo -> result;
@jz result ?undo_not_available;
! result: 0 = failed, 1 = saved, 2 = restored from undo

@restore_undo -> result;
! If successful, execution continues from the matching save_undo
! with result = 2

! Save/restore to file (v5+, EXT form):
@save table bytes name prompt -> result;
@restore table bytes name prompt -> result;
```

### 29.7.10 Catch and Throw (v5+)

```inform6
! Catch the current stack frame:
@catch -> frame;

! ... later, unwind the stack back to that frame:
@throw return_value frame;
```

`catch` stores a stack frame identifier. `throw` unwinds the stack to
the identified frame and causes the routine that executed `catch` to
return with the given value. This provides a non-local return mechanism.

## 29.8 Opcode Reference Tables

The following tables present the complete opcode set in a format
suitable for quick reference. Each table is organised by operand count
category.

### 29.8.1 2OP Opcodes

| # | Name | Code | Ver | Flags | Rules | F2 |
|---|------|------|-----|-------|-------|----|
| 0 | `je` | `$01` | 3+ | Br | — | — |
| 1 | `jl` | `$02` | 3+ | Br | — | — |
| 2 | `jg` | `$03` | 3+ | Br | — | — |
| 3 | `dec_chk` | `$04` | 3+ | Br | VARIAB | — |
| 4 | `inc_chk` | `$05` | 3+ | Br | VARIAB | — |
| 5 | `jin` | `$06` | 3+ | Br | — | — |
| 6 | `test` | `$07` | 3+ | Br | — | — |
| 7 | `or` | `$08` | 3+ | St | — | — |
| 8 | `and` | `$09` | 3+ | St | — | — |
| 9 | `test_attr` | `$0A` | 3+ | Br | — | — |
| 10 | `set_attr` | `$0B` | 3+ | — | — | — |
| 11 | `clear_attr` | `$0C` | 3+ | — | — | — |
| 12 | `store` | `$0D` | 3+ | — | VARIAB | — |
| 13 | `insert_obj` | `$0E` | 3+ | — | — | — |
| 14 | `loadw` | `$0F` | 3+ | St | — | — |
| 15 | `loadb` | `$10` | 3+ | St | — | — |
| 16 | `get_prop` | `$11` | 3+ | St | — | — |
| 17 | `get_prop_addr` | `$12` | 3+ | St | — | — |
| 18 | `get_next_prop` | `$13` | 3+ | St | — | — |
| 19 | `add` | `$14` | 3+ | St | — | — |
| 20 | `sub` | `$15` | 3+ | St | — | — |
| 21 | `mul` | `$16` | 3+ | St | — | — |
| 22 | `div` | `$17` | 3+ | St | — | — |
| 23 | `mod` | `$18` | 3+ | St | — | — |
| 24 | `call_2s` | `$19` | 4+ | St | CALL | — |
| 25 | `call_2n` | `$1A` | 5+ | — | CALL | — |
| 26 | `set_colour` | `$1B` | 5+ | — | — | 6 |
| 27 | `throw` | `$1C` | 5+ | — | — | — |

### 29.8.2 1OP Opcodes

| # | Name | Code | Ver | Flags | Rules |
|---|------|------|-----|-------|-------|
| 0 | `jz` | `$00` | 3+ | Br | — |
| 1 | `get_sibling` | `$01` | 3+ | St+Br | — |
| 2 | `get_child` | `$02` | 3+ | St+Br | — |
| 3 | `get_parent` | `$03` | 3+ | St | — |
| 4 | `get_prop_len` | `$04` | 3+ | St | — |
| 5 | `inc` | `$05` | 3+ | — | VARIAB |
| 6 | `dec` | `$06` | 3+ | — | VARIAB |
| 7 | `print_addr` | `$07` | 3+ | — | — |
| 8 | `call_1s` | `$08` | 4+ | St | CALL |
| 9 | `remove_obj` | `$09` | 3+ | — | — |
| 10 | `print_obj` | `$0A` | 3+ | — | — |
| 11 | `ret` | `$0B` | 3+ | Rf | — |
| 12 | `jump` | `$0C` | 3+ | Rf | LABEL |
| 13 | `print_paddr` | `$0D` | 3+ | — | — |
| 14 | `load` | `$0E` | 3+ | St | VARIAB |
| 15 | `not` / `call_1n` | `$0F` | 3 / 5+ | St / — | — / CALL |

Note: opcode `$0F` in the 1OP range is `not` in version 3 (with store),
and `call_1n` from version 5 onwards. In version 4, `not` moves to
opcode `$0F` with store but is still 1OP. From version 5, `not` becomes
a VAR instruction at `$38`.

### 29.8.3 0OP Opcodes

| # | Name | Code | Ver | Flags | Rules |
|---|------|------|-----|-------|-------|
| 0 | `rtrue` | `$00` | 3+ | Rf | — |
| 1 | `rfalse` | `$01` | 3+ | Rf | — |
| 2 | `print` | `$02` | 3+ | — | TEXT |
| 3 | `print_ret` | `$03` | 3+ | Rf | TEXT |
| 4 | `nop` | `$04` | 3+ | — | — |
| 5 | `save` | `$05` | 3 | Br | — |
| 6 | `restore` | `$06` | 3 | Br | — |
| 7 | `restart` | `$07` | 3+ | — | — |
| 8 | `ret_popped` | `$08` | 3+ | Rf | — |
| 9 | `pop` / `catch` | `$09` | 3–4 / 5+ | — / St | — |
| 10 | `quit` | `$0A` | 3+ | Rf | — |
| 11 | `new_line` | `$0B` | 3+ | — | — |
| 12 | `show_status` | `$0C` | 3 | — | — |
| 13 | `verify` | `$0D` | 3+ | Br | — |
| 15 | `piracy` | `$0F` | 5+ | Br | — |

Note: opcode `$09` is `pop` in versions 3–4 and `catch` (with store)
from version 5 onwards. `save` and `restore` at `$05`/`$06` are version
3 only as 0OP instructions; they move to EXT in later versions (see
§29.3.7).

### 29.8.4 VAR Opcodes

| # | Name | Code | Ver | Form | Flags | Rules | F2 |
|---|------|------|-----|------|-------|-------|----|
| 0 | `call` / `call_vs` | `$20` | 3+ / 4+ | VAR | St | CALL | — |
| 1 | `storew` | `$21` | 3+ | VAR | — | — | — |
| 2 | `storeb` | `$22` | 3+ | VAR | — | — | — |
| 3 | `put_prop` | `$23` | 3+ | VAR | — | — | — |
| 4 | `read` | `$24` | 3+ | VAR | — (v3) / St (v4+) | — | — |
| 5 | `print_char` | `$25` | 3+ | VAR | — | — | — |
| 6 | `print_num` | `$26` | 3+ | VAR | — | — | — |
| 7 | `random` | `$27` | 3+ | VAR | St | — | — |
| 8 | `push` | `$28` | 3+ | VAR | — | — | — |
| 9 | `pull` | `$29` | 3+ | VAR | — (v3–5) / St (v6) | VARIAB (v3–5) | — |
| 10 | `split_window` | `$2A` | 3+ | VAR | — | — | — |
| 11 | `set_window` | `$2B` | 3+ | VAR | — | — | — |
| 12 | `call_vs2` | `$2C` | 4+ | VAR_LONG | St | CALL | — |
| 13 | `erase_window` | `$2D` | 4+ | VAR | — | — | — |
| 14 | `erase_line` | `$2E` | 4+ | VAR | — | — | — |
| 15 | `set_cursor` | `$2F` | 4+ | VAR | — | — | — |
| 16 | `get_cursor` | `$30` | 4+ | VAR | — | — | — |
| 17 | `set_text_style` | `$31` | 4+ | VAR | — | — | — |
| 18 | `buffer_mode` | `$32` | 4+ | VAR | — | — | — |
| 19 | `output_stream` | `$33` | 3+ | VAR | — | — | — |
| 20 | `input_stream` | `$34` | 3+ | VAR | — | — | — |
| 21 | `sound_effect` | `$35` | 3+ | VAR | — | — | 7 |
| 22 | `read_char` | `$36` | 4+ | VAR | St | — | — |
| 23 | `scan_table` | `$37` | 4+ | VAR | St+Br | — | — |
| 24 | `not` | `$38` | 5+ | VAR | St | — | — |
| 25 | `call_vn` | `$39` | 5+ | VAR | — | CALL | — |
| 26 | `call_vn2` | `$3A` | 5+ | VAR_LONG | — | CALL | — |
| 27 | `tokenise` | `$3B` | 5+ | VAR | — | — | — |
| 28 | `encode_text` | `$3C` | 5+ | VAR | — | — | — |
| 29 | `copy_table` | `$3D` | 5+ | VAR | — | — | — |
| 30 | `print_table` | `$3E` | 5+ | VAR | — | — | — |
| 31 | `check_arg_count` | `$3F` | 5+ | VAR | Br | — | — |

### 29.8.5 EXT Opcodes

| # | Name | Code | Ver | Flags | F2 |
|---|------|------|-----|-------|----|
| 0 | `save` | `$00` | 5+ | St | — |
| 1 | `restore` | `$01` | 5+ | St | — |
| 2 | `log_shift` | `$02` | 5+ | St | — |
| 3 | `art_shift` | `$03` | 5+ | St | — |
| 4 | `set_font` | `$04` | 5+ | St | — |
| 5 | `draw_picture` | `$05` | 6 | — | 3 |
| 6 | `picture_data` | `$06` | 6 | Br | 3 |
| 7 | `erase_picture` | `$07` | 6 | — | 3 |
| 8 | `set_margins` | `$08` | 6 | — | — |
| 9 | `save_undo` | `$09` | 5+ | St | 4 |
| 10 | `restore_undo` | `$0A` | 5+ | St | 4 |
| 11 | `print_unicode` | `$0B` | 5+ (ZSpec 1.0) | — | — |
| 12 | `check_unicode` | `$0C` | 5+ (ZSpec 1.0) | St | — |
| 13 | `set_true_colour` | `$0D` | 5+ (ZSpec 1.1) | — | — |
| 16 | `move_window` | `$10` | 6 | — | — |
| 17 | `window_size` | `$11` | 6 | — | — |
| 18 | `window_style` | `$12` | 6 | — | — |
| 19 | `get_wind_prop` | `$13` | 6 | St | — |
| 20 | `scroll_window` | `$14` | 6 | — | — |
| 21 | `pop_stack` | `$15` | 6 | — | — |
| 22 | `read_mouse` | `$16` | 6 | — | 5 |
| 23 | `mouse_window` | `$17` | 6 | — | 5 |
| 24 | `push_stack` | `$18` | 6 | Br | — |
| 25 | `put_wind_prop` | `$19` | 6 | — | — |
| 26 | `print_form` | `$1A` | 6 | — | — |
| 27 | `make_menu` | `$1B` | 6 | Br | 8 |
| 28 | `picture_table` | `$1C` | 6 | — | 3 |
| 29 | `buffer_screen` | `$1D` | 6 | St | — |

### 29.8.6 Table Legend

The columns in the reference tables above are:

- **#**: Index in the compiler's opcode table (for cross-referencing)
- **Name**: The assembly language name used with the `@` prefix
- **Code**: The opcode number (hexadecimal) within its category
- **Ver**: The Z-machine version(s) where this opcode is available
- **Form**: The operand count category (if different from the table's
  default)
- **Flags**: `St` = stores result, `Br` = branches, `Rf` = never
  continues (return/jump), `St+Br` = stores and branches
- **Rules**: `VARIAB` = first operand is a variable number,
  `CALL` = first operand is a routine packed address,
  `TEXT` = inline Z-encoded string, `LABEL` = label offset
- **F2**: If non-zero, the Flags 2 header bit that is set when this
  opcode is assembled
