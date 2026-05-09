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

# Chapter 28: The Z-Machine Architecture

**[Z-machine]** This chapter describes the architecture of the Z-machine
virtual machine as targeted by the compiler. The Z-machine is the primary
compilation target and was originally designed by Infocom in the early
1980s to run interactive fiction portably across many different hardware
platforms. The compiler produces story files that conform to the Z-machine
specification, and understanding the underlying architecture is valuable
for advanced optimization, debugging, and writing assembly-level code.

All details in this chapter have been verified against the Inform 6.44
compiler source code. Where the compiler's behavior differs from or
extends the Z-Machine Standard document, those differences are noted.

## 28.1 History and Versions

The Z-machine exists in several versions, numbered 3 through 8. Each
version extends the previous one with additional capabilities, larger
address spaces, or new instructions. The compiler supports all six
versions and assigns each an internal name.

The compiler's `select_version()` function configures
the following parameters for each version:

| Version | Internal Name | Scale Factor | Length Scale Factor | Extended Memory Map | Instruction Set |
|---------|---------------|-------------|--------------------|--------------------|-----------------|
| 3 | Standard | 2 | 2 | No | 3 |
| 4 | Plus | 4 | 4 | No | 4 |
| 5 | Advanced | 4 | 4 | No | 5 |
| 6 | Graphical | 4 | 8 | Yes | 6 |
| 7 | Expanded Advanced | 4 | 8 | Yes | 5 |
| 8 | Expanded Advanced | 8 | 8 | No | 5 |

The **scale factor** is the packed address multiplier. Routine and string
addresses stored in the story file are *packed addresses* that must be
multiplied by the scale factor to obtain the actual byte address. A larger
scale factor allows addressing a larger memory space but requires that
routines and strings be aligned to the corresponding boundary.

The **length scale factor** is used to convert the file length stored in
the header (bytes 26–27) into actual bytes. For most versions it equals
the scale factor, but versions 6 and 7 use a length scale factor of 8
regardless of their scale factor.

The **extended memory map** applies only to versions 6 and 7. These
versions use separate routine and string offset values (stored in header
bytes 40–43) to extend the addressable space for packed addresses.

### 28.1.1 Instruction Sets vs. Version Numbers

An important distinction in the compiler is that the **version number** and
the **instruction set number** are not always the same. Versions 7 and 8
both use instruction set 5. Version 6 has its own instruction set (6)
containing graphical opcodes. The compiler sets this as follows:

```
version_number:        3  4  5  6  7  8
instruction_set_number: 3  4  5  6  5  5
```

This means that versions 7 and 8 accept the same opcodes as version 5.
Version 6 accepts all version 5 opcodes plus the graphical opcodes
(`draw_picture`, `picture_data`, `move_window`, etc.) that are
exclusive to version 6.

### 28.1.2 Version Selection

The target version is selected with the `-v` compiler switch:

- `-v3` through `-v8` select the corresponding Z-machine version
- `-v5` is the default when no version is specified

The version may also be set with `!% -v5` at the top of a source file
(the ICL header comment convention) or via an ICL command file.

## 28.2 Memory Map: Dynamic, Static, and High Memory

A Z-machine story file is divided into three regions, each with
different access rules at runtime:

1. **Dynamic memory** — from byte 0 to the start of static memory.
   This region can be both read and written at runtime. It contains the
   header, abbreviations table, object table, property values, global
   variables, and dynamic arrays.

2. **Static memory** — from the static memory base address (header
   bytes 14–15) to the start of high memory. This region is read-only
   at runtime. It contains the grammar tables, action routine addresses,
   the dictionary, and static arrays.

3. **High memory** — from the high memory base address (header bytes
   4–5) to the end of the file. This region contains executable code
   (routines) and the static string pool. High memory is addressed via
   packed addresses rather than byte addresses.

### 28.2.1 Construction Order

The compiler builds the story file sequentially. The following listing shows the order in which sections
are laid out, grouped by memory region:

**Dynamic memory (byte `0x00` to `grammar_table_at - 1`):**

1. Header — 64 bytes (`0x00`–`0x3F`)
2. Low strings pool — starts at `0x40`, with a default "empty" string
3. Abbreviations table — 96 entries × 2 bytes (192 bytes)
4. Header extension table — variable length
5. Character set table — 78 bytes (only if the alphabet has been modified)
6. Unicode translation table — variable length (only if ZSCII definitions
   have been modified)
7. Property defaults table — 31 words for v3, 63 words for v4+
8. Object table — 9 bytes per object for v3, 14 bytes per object for v4+
9. Property value tables — variable length, one per object
10. Class prototype numbers table — 2 bytes per class, plus 2 trailing zeros
11. Identifier names table (unless `$OMIT_SYMBOL_TABLE` is set)
12. Individual property values table
13. Global variables and dynamic arrays
14. Terminating characters table (v5+ only)

**Static memory (`grammar_table_at` to `Write_Code_At - 1`):**

15. Grammar tables — index table plus per-verb grammar lines
16. Action routine addresses — 2 bytes per action
17. General parsing routine addresses (GV1/GV3 only)
18. Adjectives table (GV1/GV3 only)
19. Dictionary
20. Static arrays
21. Padding to code area boundary

**High memory (`Write_Code_At` onwards):**

22. Code area — compiled routines
23. String pool — static strings

The boundary between dynamic and static memory is recorded in header
bytes 14–15 as the address of the grammar table. The boundary between
static and high memory is recorded in header bytes 4–5 as the
`Write_Code_At` address.

### 28.2.2 Alignment Requirements

The code area must start at an address that is a multiple of the
`length_scale_factor`. The compiler inserts padding bytes (zeros) between
the static arrays and the code area to achieve this alignment. It also
ensures that the first routine has a packed address of at least 256,
so that all routine packed addresses are large enough to be encoded
as long constants.

When the `-B` (oddeven packing) switch is used, the compiler additionally
requires the code area start to be aligned to twice the scale factor,
and the string pool to be offset by one scale factor from code, allowing
routines and strings to occupy interleaved address ranges.

## 28.3 The Header

The first 64 bytes of every Z-machine story file form the header. The
compiler writes these bytes in `construct_storyfile_z()` after all
other sections have been assembled, so that all addresses are known.
The following table documents every header field as set by the Inform
6.44 compiler:

| Bytes | Field | Description |
|-------|-------|-------------|
| 0 | Version number | Z-machine version (3–8) |
| 1 | Flags 1 | Bit 1: status line type (0 = score/turns, 1 = hours:mins) |
| 2–3 | Release number | The `Release` directive value, big-endian |
| 4–5 | High memory base | Byte address where high memory begins (`Write_Code_At`) |
| 6–7 | Initial PC / Main | v3–5, v7–8: byte address of first instruction in `Main__` (= `Write_Code_At + 1`). v6: packed address of `Main__` |
| 8–9 | Dictionary | Byte address of the dictionary table |
| 10–11 | Object table | Byte address of the property defaults table (objects follow immediately) |
| 12–13 | Global variables | Byte address of the global variables area |
| 14–15 | Static memory base | Byte address where static memory begins (= grammar table address) |
| 16–17 | Flags 2 | Feature flags, auto-populated from assembled opcodes |
| 18–23 | Serial number | Six ASCII characters in YYMMDD format |
| 24–25 | Abbreviations table | Byte address of the abbreviations table |
| 26–27 | File length | Length of the file divided by the length scale factor (filled in at output time) |
| 28–29 | Checksum | Unsigned sum of all bytes from `0x40` onwards (filled in at output time) |
| 30–31 | Interpreter number/version | Set to 0 by the compiler; filled in by the interpreter at runtime |
| 32–33 | Screen height/width | Set to 0; filled in by the interpreter |
| 34–37 | Screen dimensions in units | Set to 0; filled in by the interpreter |
| 38–39 | Font dimensions | Set to 0; filled in by the interpreter |
| 40–41 | Routines offset | v6/v7 only: offset for computing routine packed addresses |
| 42–43 | Strings offset | v6/v7 only: offset for computing string packed addresses |
| 44–45 | Default background colour | Set to 0 by the compiler |
| 46–47 | Terminating characters table | v5+: byte address of the terminating characters table |
| 48–49 | Output stream 3 width | Set to 0; used at runtime |
| 50–51 | Standard revision number | Set to 0 by the compiler |
| 52–53 | Alphabet table address | Byte address of the custom alphabet table, if `alphabet_modified` is true; otherwise 0 |
| 54–55 | Header extension table | Byte address of the header extension table |
| 56–59 | (Reserved) | Set to 0 |
| 60–63 | Inform version | Four ASCII characters encoding the compiler version (e.g., `6.44`) |

### 28.3.1 Flags 2

The Flags 2 word (header bytes 16–17) is automatically populated by
the compiler based on which opcodes have been assembled. Each opcode
in the compiler's opcode table has a `flags2_set` field; if an opcode
with a non-zero `flags2_set` value is assembled, the corresponding bit
in Flags 2 is set. The known flag bits are:

| Bit | Meaning | Set by opcode |
|-----|---------|---------------|
| 3 | Game wants pictures | `draw_picture`, `picture_data`, `erase_picture` |
| 4 | Game wants undo | `save_undo`, `restore_undo` |
| 5 | Game wants mouse | `read_mouse`, `mouse_window` |
| 6 | Game wants colour | `set_colour` |
| 7 | Game wants sound | `sound_effect` |
| 8 | Game wants menus | `make_menu` |

### 28.3.2 Header Extension Table

The header extension table provides additional header fields beyond the
base 64 bytes. Its address is stored in header bytes 54–55. The first
word of the table gives the number of extension words that follow. The
compiler uses 1-based indexing for the extension words:

| Word | Field | Description |
|------|-------|-------------|
| 0 | Length | Number of extension words following this word |
| 1 | (Reserved) | Set to 0 |
| 2 | (Reserved) | Set to 0 |
| 3 | Unicode table | Byte address of the Unicode translation table (if ZSCII has been modified) |
| 4 | Flags 3 | Additional feature flags (ZSpec 1.1); controlled by `$ZCODE_HEADER_FLAGS_3` |

The compiler ensures the extension table has at least 3 words if a
Unicode translation table is needed, and at least 4 words if Flags 3
are in use.

## 28.4 Object Table Layout

The object table is one of the most important data structures in the
Z-machine. It consists of two parts: the **property defaults table**
and the **object entries** themselves.

### 28.4.1 Property Defaults Table

The property defaults table immediately precedes the object entries. It
provides default values for common properties (those that an object does
not explicitly define):

- **Version 3:** 31 entries (properties 1–31), each a 2-byte word.
  Total size: 62 bytes.
- **Version 4+:** 63 entries (properties 1–63), each a 2-byte word.
  Total size: 126 bytes.

Property 1 is unused (the first entry in the table is always zero). The
table must be word-aligned in the story file.

### 28.4.2 Object Entries

Each object entry is a fixed-size record in the object table. The format
depends on the Z-machine version:

**Version 3 (9 bytes per object):**

| Offset | Size | Field |
|--------|------|-------|
| 0–3 | 4 bytes | Attribute flags (32 attributes, bit 7 of byte 0 = attribute 0) |
| 4 | 1 byte | Parent object number (0–255) |
| 5 | 1 byte | Sibling object number (0–255) |
| 6 | 1 byte | Child object number (0–255) |
| 7–8 | 2 bytes | Property table address (byte address) |

**Version 4+ (14 bytes per object):**

| Offset | Size | Field |
|--------|------|-------|
| 0–5 | 6 bytes | Attribute flags (48 attributes, bit 7 of byte 0 = attribute 0) |
| 6–7 | 2 bytes | Parent object number (0–65535) |
| 8–9 | 2 bytes | Sibling object number (0–65535) |
| 10–11 | 2 bytes | Child object number (0–65535) |
| 12–13 | 2 bytes | Property table address (byte address) |

The single-byte tree pointers in version 3 limit the total number of
objects to 255. Versions 4+ use 2-byte pointers, supporting up to
65535 objects.

### 28.4.3 Attribute Storage

Attributes are stored as bit fields in the first bytes of each object
entry. The bit ordering places attribute 0 in the most significant bit
(bit 7) of the first byte. Thus attribute *n* is stored in bit
`(7 - (n % 8))` of byte `(n / 8)`.

### 28.4.4 Property Tables

Each object's property table is located at the address stored in the
object entry. The property table begins with a text length byte followed
by Z-encoded text giving the object's short name, then a sequence of
property entries.

Each property entry has a size-and-number byte (or bytes in v4+ for
properties longer than 8 bytes of data) followed by the property data.
Properties are stored in descending order of property number, terminated
by a zero byte.

The internal compiler structure for a Z-code object is:

```c
typedef struct objecttz {
    uchar atts[6];       /* attribute bytes */
    int parent;          /* parent object number */
    int next;            /* sibling object number */
    int child;           /* child object number */
    int propsize;        /* size of property table */
} objecttz;
```

## 28.5 Dictionary Layout

The dictionary table stores the game's vocabulary — every word that the
parser can recognize. The table has three parts:

### 28.5.1 Separator List

The dictionary begins with a list of word-separator characters. The
compiler always writes three separators: `.` `,` `"`. The first byte
gives the count (3), followed by the separator characters.

### 28.5.2 Entry Length and Count

After the separators, one byte gives the length of each dictionary entry,
followed by a 2-byte word giving the total number of entries.

### 28.5.3 Dictionary Entries

Each entry consists of Z-encoded text followed by data bytes:

**Version 3 (7 bytes per entry):**

| Offset | Size | Field |
|--------|------|-------|
| 0–3 | 4 bytes | Z-encoded text (6 Z-characters = up to 6 resolved characters) |
| 4–6 | 3 bytes | Data bytes (flags and grammar information) |

**Version 4+ (9 bytes per entry, or 8 with `$ZCODE_LESS_DICT_DATA`):**

| Offset | Size | Field |
|--------|------|-------|
| 0–5 | 6 bytes | Z-encoded text (9 Z-characters = up to 9 resolved characters) |
| 6–8 | 3 bytes | Data bytes (flags and grammar information) |

When the compiler setting `$ZCODE_LESS_DICT_DATA` is enabled, v4+
entries use only 2 data bytes (8 bytes total), omitting one data byte.

Dictionary entries are stored in sorted order (alphabetical by their
Z-encoded form) to allow binary search at runtime. The compiler
computes the final sort order and writes entries in that order.

## 28.6 Grammar Table Layout

The grammar table defines the verb grammar that the parser uses to
interpret player input. The compiler supports three grammar
table formats: GV1 (grammar version 1), GV2, and GV3. The format is
selected at compile time and affects the structure of the grammar lines.

### 28.6.1 Grammar Table Index

All grammar versions share the same index structure. The table begins
with `no_Inform_verbs × 2` bytes — a word address for each verb
pointing to that verb's grammar line table.

Each per-verb table starts with a 1-byte line count, followed by the
grammar lines for that verb. If a verb was marked as unused by the
dead-code eliminator, its line count is 0.

### 28.6.2 GV1: Fixed-Length Records

Grammar version 1 uses fixed-length 8-byte records for each grammar
line:

| Offset | Size | Field |
|--------|------|-------|
| 0 | 1 byte | Number of noun tokens that precede the action |
| 1–6 | 6 bytes | Token data (up to 6 tokens, unused slots are zero) |
| 7 | 1 byte | Action number |

In GV1, tokens with values less than 180 are counted as "noun" tokens.
Routine addresses and action numbers are backpatched after the story
file is assembled.

### 28.6.3 GV2: Variable-Length Records

Grammar version 2 uses variable-length records. Each grammar line
consists of:

1. **Action word** (2 bytes): encodes the action number and token
   information
2. **Token sequence**: pairs of bytes, one per token
3. **Terminator**: a byte with value 15 (`0x0F`) marks the end of the
   token sequence

The action word encodes:
- Bits 0–9: action number (0–1023)
- Bit 10: reverse flag
- Bits 11–15: not used in GV2 (token count is implicit from the
  terminator)

### 28.6.4 GV3: Compact Variable-Length Records

Grammar version 3 is similar to GV2 but omits the `0x0F` terminator
byte. Instead, the token count is encoded directly in the action word:

1. **Action word** (2 bytes): bits 0–9 = action number, bits 3–7 of the
   high byte encode the token count (extracted as `(high_byte & 0xF8) >> 3`)
2. **Token sequence**: exactly `token_count` pairs of bytes

This format is more compact than GV2 because it eliminates the per-line
terminator byte.

## 28.7 The Story File Format

A Z-machine story file is a single binary file containing all of the
data structures described in this chapter. The compiler assembles the
file in a single pass through `construct_storyfile_z()`, writing
sections in the order described in §28.2.

### 28.7.1 Address Resolution and Backpatching

During compilation, many addresses cannot be determined until all code
has been generated. The compiler uses a **backpatching** system to
record these unresolved references and fix them up after the story file
is fully assembled.

The backpatching system uses **marker
values** to classify each reference that needs resolution. When a value
is written into the story file or code area, it may be tagged with a
marker indicating what kind of address adjustment is needed.

The following marker types are used:

| Marker | Description |
|--------|-------------|
| `NULL_MV` | No adjustment needed |
| `DWORD_MV` | Dictionary word — adjusted to byte address within dictionary |
| `STRING_MV` | String literal — adjusted by `strings_offset / scale_factor` |
| `INCON_MV` | System constant — resolved to its value |
| `IROUTINE_MV` | Internal routine — adjusted by `code_offset / scale_factor` |
| `VROUTINE_MV` | Veneer routine — resolved via the veneer routine address table, then adjusted by code offset |
| `ARRAY_MV` | Internal array — adjusted by `variables_offset` (with compact globals adjustment if applicable) |
| `STATIC_ARRAY_MV` | Static array — adjusted by `static_arrays_offset` |
| `NO_OBJS_MV` | Number of objects — resolved to the total object count |
| `INHERIT_MV` | Inherited common property value — looked up from the property values table |
| `INDIVPT_MV` | Individual property table address — adjusted by `individuals_offset` |
| `INHERIT_INDIV_MV` | Inherited individual property value — resolved from the individual property table |
| `MAIN_MV` | Reference to `Main` routine — resolved to the Main symbol's value |
| `SYMBOL_MV` | General symbol reference — resolved to the symbol's value |
| `VARIABLE_MV` | Global variable (Glulx only) |
| `ACTION_MV` | Action reference (Glulx only) |
| `OBJECT_MV` | Internal object (Glulx only) |
| `ERROR_MV` | Error marker — indicates a prior compilation error |

After the entire story file has been laid out in memory, the compiler
calls `backpatch_zmachine_image_z()` to walk through all recorded
backpatch entries and apply the appropriate adjustments. It also
separately backpatches the symbol name table, action routine addresses,
and grammar token routine addresses.

### 28.7.2 Code Area

The code area contains all compiled routines, packed sequentially. Each
routine begins with a header (described in §29) followed by the routine's
instructions. Routine addresses in the story file are stored as packed
addresses — that is, the byte address divided by the scale factor.

### 28.7.3 String Pool

The string pool follows the code area (or is interleaved with it when
oddeven packing is used). It contains all static string literals used in
the program, Z-encoded and packed. String addresses are also stored as
packed addresses.

### 28.7.4 File Size Limits

After assembling the complete file, the compiler checks whether it
exceeds the version-specific size limit:

| Version | Maximum Size |
|---------|-------------|
| 3 | 128 KB (`0x20000` bytes) |
| 4, 5 | 256 KB (`0x40000` bytes) |
| 6, 7, 8 | 512 KB (`0x80000` bytes) |

If the file exceeds the limit, a fatal error is reported. For versions
6 and 7 with extended memory maps, the compiler additionally checks
that the code area and strings area individually fit within the packed
address range (`scale_factor × 0x10000` bytes each).

## 28.8 Z-Machine Limitations by Version

The following table summarises the key limitations that vary between
Z-machine versions, as determined by the compiler source code:

| Capability | v3 | v4 | v5 | v6 | v7 | v8 |
|-----------|----|----|----|----|----|----|
| Maximum story file size | 128 KB | 256 KB | 256 KB | 512 KB | 512 KB | 512 KB |
| Packed address scale factor | 2 | 4 | 4 | 4 | 4 | 8 |
| Maximum objects | 255 | 65535 | 65535 | 65535 | 65535 | 65535 |
| Attributes per object | 32 | 48 | 48 | 48 | 48 | 48 |
| Common properties | 31 | 63 | 63 | 63 | 63 | 63 |
| Global variables | 240 | 240 | 240 | 240 | 240 | 240 |
| Maximum verbs | 255 | 255 | 255 | 255 | 255 | 255 |
| Dictionary word resolution | 6 chars | 9 chars | 9 chars | 9 chars | 9 chars | 9 chars |
| Object entry size | 9 bytes | 14 bytes | 14 bytes | 14 bytes | 14 bytes | 14 bytes |
| Dict entry size | 7 bytes | 9 bytes | 9 bytes | 9 bytes | 9 bytes | 9 bytes |
| Instruction set | 3 | 4 | 5 | 6 | 5 | 5 |
| Extended memory map | No | No | No | Yes | Yes | No |
| Status line | Score/time | Score/time | — | — | — | — |
| Terminating chars table | No | No | Yes | Yes | Yes | Yes |

### 28.8.1 Readable Memory Limit

Regardless of the maximum file size, the Z-machine imposes a limit on
**readable memory** (dynamic + static memory). This area must fit within
the first 64 KB (`0xFFFE` bytes) of the story file. If the combined
size of dynamic and static memory exceeds this limit, the compiler
reports an error:

> This program has overflowed the maximum readable-memory size of the
> Z-machine format.

This can occur with very large games that have many objects, properties,
grammar lines, or dictionary entries.

### 28.8.2 Global Variables

All Z-machine versions support exactly 240 global variables, stored as
16-bit words in a 480-byte area. The compiler defines
`MAX_ZCODE_GLOBAL_VARS` as 240. Variable numbers 0–15 are local
variables; variable 0 is the stack pointer. Variable numbers 16–255
map to the 240 globals.

When the `$ZCODE_COMPACT_GLOBALS` setting is enabled, the compiler
writes only the globals that are actually used, followed immediately
by dynamic arrays. This can save space in the dynamic memory area,
but the compiler must adjust backpatch calculations to account for
the different layout.

## 28.9 ZSCII Character Encoding

The Z-machine uses its own character encoding called **ZSCII** (Zork
Standard Code for Information Interchange). The compiler handles
conversion between several character representations during
compilation.

### 28.9.1 Character Representations

The compiler works with six different
character representations:

| Representation | Size | Range | Description |
|---------------|------|-------|-------------|
| ASCII | 7-bit | `$20`–`$7E` | Plain ASCII printable characters |
| Source | 8-bit | `$00`–`$FF` | Raw bytes from the source file |
| ISO | 8-bit | `$00`–`$FF` | ASCII or ISO 8859-1 through 8859-9, depending on `character_set_setting` |
| ZSCII | 10-bit | 0–1023 | The Z-machine's native character set |
| Textual | sequence | — | Escape sequences such as `@'e` (e-acute) or `@$03A3` (Greek sigma) |
| Unicode | 16-bit | 0–65535 | A unifying representation for all characters |

Conversion can always proceed downward through this list (ASCII → Source
→ ISO → ZSCII → Unicode) but generally not upward, since information may
be lost.

### 28.9.2 The Z-Machine Alphabet

ZSCII text in the story file is encoded using **Z-characters**, a
compressed representation based on three 26-letter alphabets:

| Alphabet | Name | Default Contents |
|----------|------|-----------------|
| A0 | Lowercase | `a b c d e f g h i j k l m n o p q r s t u v w x y z` |
| A1 | Uppercase | `A B C D E F G H I J K L M N O P Q R S T U V W X Y Z` |
| A2 | Punctuation | (special) `0 1 2 3 4 5 6 7 8 9 . , ! ? _ # ' / \ - : ( )` and more |

Each alphabet contains 26 entries. Special Z-character values are used
to switch between alphabets:

- Z-char 0: space
- Z-char 1: abbreviation (table 0)
- Z-char 2: abbreviation (table 1)  [v3: shift to A1]
- Z-char 3: abbreviation (table 2)  [v3: shift to A2]
- Z-char 4: shift to A1  [v3: shift-lock to A1]
- Z-char 5: shift to A2  [v3: shift-lock to A2]
- Z-chars 6–31: the 26 letters of the current alphabet

In alphabet A2, position 0 (Z-char 6) is special: it introduces a
two-Z-char literal ZSCII code (used for characters not in any alphabet).
Position 1 (Z-char 7) represents a newline character.

The alphabet can be customized using the `Zcharacter` directive, which
modifies the `alphabet[][]` array in the compiler. If the alphabet is
modified, the custom alphabet table is written into the story file and
its address is recorded in header bytes 52–53.

### 28.9.3 Z-Character Encoding

Z-characters are 5 bits each. They are packed three per 2-byte word in
big-endian format:

```
  Bit:  15  14 13 12 11 10  9  8  7  6  5  4  3  2  1  0
       [T] [---char 1---] [---char 2---] [---char 3---]
```

- Bit 15: termination flag. Set to 1 in the last word of the string; 0
  in all preceding words.
- Bits 14–10: first Z-character
- Bits 9–5: second Z-character
- Bits 4–0: third Z-character

If a string's length is not a multiple of three Z-characters, it is
padded with Z-char 5 (shift-to-A2) characters.

### 28.9.4 Accented Characters

ZSCII values 155 and above represent accented and special characters.
The compiler defines 69 standard accented characters starting at ZSCII
155, as specified by the Z-Machine Standard version 0.2:

| ZSCII | Character | ZSCII | Character |
|-------|-----------|-------|-----------|
| 155 | ä (a-umlaut) | 169 | á (a-acute) |
| 156 | ö (o-umlaut) | 170 | é (e-acute) |
| 157 | ü (u-umlaut) | 171 | í (i-acute) |
| 158 | Ä (A-umlaut) | 172 | ó (o-acute) |
| 159 | Ö (O-umlaut) | 173 | ú (u-acute) |
| 160 | Ü (U-umlaut) | 174 | ý (y-acute) |
| 161 | ß (eszett) | 175 | À (A-grave) |
| 162 | » (right guillemet) | 176 | È (E-grave) |
| 163 | « (left guillemet) | ... | ... |

The complete list of 69 characters is defined by the compiler's internal
`accents` string. These characters can be entered in
source code using the `@` escape notation (e.g., `@:a` for ä, `@'e`
for é).

### 28.9.5 Unicode Translation Table

When ZSCII character definitions are modified (using the `Zcharacter`
directive), the compiler writes a **Unicode translation table** into
the story file. This table maps ZSCII values 155 and above to their
corresponding Unicode code points.

The table format is:

| Offset | Size | Field |
|--------|------|-------|
| 0 | 1 byte | Number of entries (*N*) |
| 1 | 2×*N* bytes | Unicode code points, each as a 2-byte big-endian value |

The table's address is stored in word 3 of the header extension table.
Each entry maps ZSCII value `155 + i` to the Unicode code point at
offset `1 + 2×i`.

### 28.9.6 Conversion Grids

The compiler maintains several lookup tables (called "grids") for
efficient character conversion:

- `source_to_iso_grid[256]` — filters source characters into valid ISO
- `iso_to_unicode_grid[256]` — maps ISO characters to Unicode code points
- `iso_to_alphabet_grid[256]` — maps ISO characters to positions in the
  Z-machine alphabet (0–77), or to negative values for characters that
  exist in ZSCII but not in the alphabet, or to −5 for characters not
  in ZSCII at all
- `zscii_to_unicode_grid[256]` — maps ZSCII values to Unicode code
  points

These grids must be kept consistent with each other. Changing one
structure (such as the alphabet table or the character set setting)
may require rebuilding several others. These dependency relationships
are maintained internally by the compiler.
