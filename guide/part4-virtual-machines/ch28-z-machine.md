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

The Z-machine is the original virtual machine for Inform. It is a portable
abstract processor designed to execute interactive fiction story files across
a wide range of hardware platforms. The information in this chapter derives
from the Z-Machine Standards Document (version 1.1) and the compiler source
code (Inform 6.44). The Z-machine defines a complete execution environment
including memory layout, instruction set, text encoding, input/output
facilities, and a persistent object model. Inform 6 compiles source code
into Z-machine instructions by default, producing a story file that any
conforming interpreter can execute.

## §28.1 Overview and Version History

The Z-machine was designed by Joel Berez and Marc Blank at Infocom in 1979
as the runtime engine for their text adventure games. The name derives from
"Zork," the first game developed using the system. The design prioritized
portability: a single story file could run on any platform that had a
Z-machine interpreter, from early 8-bit microcomputers to modern desktop
systems.

### §28.1.1 Version Timeline

Infocom created versions 1 through 6 of the Z-machine during the
company's active years (1979–1989). Versions 7 and 8 were later added
by the Inform community to address story file size limitations.

| Version | Origin | Key Characteristics |
| ------- | ------ | ------------------- |
| 1 | Infocom (1979) | Original format; no longer in active use |
| 2 | Infocom (1981) | Minor revisions; no longer in active use |
| 3 | Infocom (1982) | Status line, 255 objects, 32 attributes, 128 KB limit |
| 4 | Infocom (1985) | Timed input, 65535 objects, 48 attributes, 256 KB limit |
| 5 | Infocom (1987) | Extended opcodes, 96 abbreviations, 256 KB limit |
| 6 | Infocom (1988) | Graphics, split windows, mouse input, 512 KB limit |
| 7 | Inform | Offset addressing for routines/strings, 320 KB limit |
| 8 | Inform | Larger packing factor, 512 KB limit |

### §28.1.2 Version Selection in Inform 6

Inform 6 can target versions 3, 4, 5, 6, 7, and 8 using the `-v` compiler
switch. Version 5 is the default and is the most commonly used target for
new works. Version 8 provides the largest story file capacity for Z-machine
targets.

```inform6
! Compile for version 3 (classic Infocom-style):
!   inform -v3 source.inf

! Compile for version 5 (default):
!   inform -v5 source.inf

! Compile for version 8 (large story files):
!   inform -v8 source.inf
```

Version 6 is architecturally unique among Z-machine versions. It supports
bitmap graphics, proportional fonts, multiple split windows, and mouse
input. However, it is rarely used in practice because few modern
interpreters fully support its display model.

### §28.1.3 Conditional Compilation by Version

Inform 6 provides compile-time directives to conditionally include code
based on the target version:

```inform6
#IfV3;
! Code compiled only for version 3
print "This is a V3 story file.^";
#EndIf;

#IfV5;
! Code compiled for version 5 and later (5, 6, 7, 8)
print "This is a V5+ story file.^";
#EndIf;

#Ifdef TARGET_ZCODE;
! Code compiled only when targeting the Z-machine (any version)
print "Running on the Z-machine.^";
#EndIf;
```

## §28.2 Memory Map

The Z-machine's address space is divided into three contiguous regions.
Each region has different access permissions, and the boundaries between
them are defined by values stored in the story file header.

### §28.2.1 Dynamic Memory

Dynamic memory begins at byte address 0 and extends up to the static
memory base address. This region contains:

- The 64-byte header
- The abbreviations table
- The property defaults table
- The object table (including property data)
- Global variables (240 two-byte words)
- Arrays allocated with `Array` directives

Dynamic memory can be both read and written at runtime. It is the only
region affected by the `@save` and `@restore` instructions, which
capture and reinstate the entire dynamic memory state.

### §28.2.2 Static Memory

Static memory begins at the address specified in header bytes `$0E`–`$0F`
and extends to the end of the accessible memory region. This region
contains:

- The dictionary
- Grammar tables (action routines, parse tables)
- Static strings created with the `Lowstring` directive
- Any data placed here by the compiler

Static memory can be read but not written at runtime. Attempting to write
to a static memory address produces undefined behavior.

### §28.2.3 High Memory

High memory begins at the address specified in header bytes `$04`–`$05`.
It contains:

- Compiled routines (function code)
- Printed strings (text referenced by `print` and `print_ret`)

High memory is accessible only through packed addresses. The program
counter operates within high memory when executing routines. High memory
may overlap with static memory; the overlap region can be read as data
using byte/word addresses but also contains executable code addressed
through packed addresses.

### §28.2.4 Memory Layout Diagram

```
Address 0          +--------------------------+
                   |        Header            |  64 bytes
                   +--------------------------+
                   |   Abbreviations Table    |
                   +--------------------------+
                   |   Property Defaults      |
                   +--------------------------+
                   |      Object Table        |
                   +--------------------------+
                   |    Global Variables      |  240 words
                   +--------------------------+
                   |    Dynamic Arrays        |
Static base addr   +--------------------------+
                   |      Dictionary          |
                   +--------------------------+
                   |    Grammar Tables        |
                   +--------------------------+
                   |    Static Strings        |
High memory addr   +--------------------------+
                   |   Routines & Strings     |
                   |    (packed addresses)    |
End of file        +--------------------------+
```

### §28.2.5 Memory Size Limits

The total addressable memory varies by Z-machine version:

| Version | Maximum Memory |
| ------- | -------------- |
| 3       | 128 KB         |
| 4       | 256 KB         |
| 5       | 256 KB         |
| 6       | 512 KB         |
| 7       | 320 KB         |
| 8       | 512 KB         |

Dynamic memory is limited to 64 KB in all versions because byte addresses
in dynamic memory are 16-bit values.

## §28.3 The Header

The first 64 bytes of a Z-machine story file constitute the header. The
header contains essential metadata that interpreters read at startup and
that the compiler populates during compilation. Some fields are set by the
compiler; others are written by the interpreter at runtime to communicate
capabilities.

### §28.3.1 Header Field Table

| Bytes     | Field                          | Description |
| --------- | ------------------------------ | ----------- |
| `$00`     | Version number                 | Z-machine version (3–8) |
| `$01`     | Flags 1                        | Bit field: status line type, story file split, censorship, etc. |
| `$02–$03` | Release number                 | Game release number (set by `Release` directive) |
| `$04–$05` | High memory base               | Byte address where high memory begins |
| `$06–$07` | Initial PC                     | Byte address of first instruction to execute (v1–5) or packed address of main routine (v6) |
| `$08–$09` | Dictionary address             | Byte address of the dictionary |
| `$0A–$0B` | Object table address           | Byte address of the object table |
| `$0C–$0D` | Global variable table address  | Byte address of the global variables table |
| `$0E–$0F` | Static memory base             | Byte address where static memory begins |
| `$10–$11` | Flags 2                        | Bit field: transcription on/off, fixed-pitch request, etc. |
| `$12–$17` | Serial number                  | Six ASCII characters; usually the compilation date (YYMMDD) |
| `$18–$19` | Abbreviations table address    | Byte address of the abbreviations table |
| `$1A–$1B` | File length                    | Story file length divided by the packing factor |
| `$1C–$1D` | Checksum                       | Sum of all bytes in the file (mod 65536), excluding the header |
| `$1E`     | Interpreter number             | Identifies the interpreter (set by interpreter at runtime) |
| `$1F`     | Interpreter version            | Version of the interpreter (set by interpreter at runtime) |
| `$20`     | Screen height (lines)          | Set by interpreter |
| `$21`     | Screen width (characters)      | Set by interpreter |
| `$22–$23` | Screen width (units)           | Set by interpreter (v6+) |
| `$24–$25` | Screen height (units)          | Set by interpreter (v6+) |
| `$26`     | Font width (units, v6) / height (v5) | Set by interpreter |
| `$27`     | Font height (units, v6) / width (v5) | Set by interpreter |
| `$28–$29` | Routines offset                | Offset for packed routine addresses (v6–7 only) |
| `$2A–$2B` | Strings offset                 | Offset for packed string addresses (v6–7 only) |
| `$2C`     | Default background colour      | Set by interpreter |
| `$2D`     | Default foreground colour      | Set by interpreter |
| `$2E–$2F` | Terminating characters table   | Byte address of table (v5+) |
| `$30–$31` | Output stream 3 width          | Total width of text sent to stream 3 (v6) |
| `$32–$33` | Standard revision number       | Z-Machine Standard revision number |
| `$34–$35` | Alphabet table address         | Byte address of custom alphabet table (v5+), or 0 for default |
| `$36–$37` | Header extension table address | Byte address of header extension table (v5+) |
| `$38–$3F` | Inform version identifier      | Compiler version string (set by Inform) |

### §28.3.2 Flags 1 (Byte $01)

The meaning of flags 1 bits varies by version:

**Version 3:**

| Bit | Meaning |
| --- | ------- |
| 1   | Status line type: 0 = score/turns, 1 = hours:minutes |
| 2   | Story file split across two discs |
| 3   | Censorship flag (the story is "tandy-approved") |
| 4   | Status line not available (set by interpreter) |
| 5   | Screen-splitting available (set by interpreter) |
| 6   | Variable-pitch font is default (set by interpreter) |

**Versions 4+:**

| Bit | Meaning |
| --- | ------- |
| 0   | Colours available (set by interpreter) |
| 1   | Picture display available (set by interpreter) |
| 2   | Bold face available (set by interpreter) |
| 3   | Italic available (set by interpreter) |
| 4   | Fixed-space style available (set by interpreter) |
| 5   | Sound effects available (set by interpreter) |
| 7   | Timed keyboard input available (set by interpreter) |

### §28.3.3 Flags 2 (Bytes $10–$11)

| Bit | Meaning |
| --- | ------- |
| 0   | Transcription on (set by game or interpreter) |
| 1   | Force fixed-pitch printing (set by game) |
| 2   | Interpreter requests screen redraw (v6) |
| 3   | Request: use pictures (set by game) |
| 4   | Request: use undo opcodes (set by game) |
| 5   | Request: use mouse (set by game) |
| 7   | Request: use sound effects (set by game) |
| 8   | Request: use menus (v6, set by game) |

### §28.3.4 Accessing the Header from Inform 6

Inform 6 provides symbolic constants and low-level memory access for
reading header fields at runtime:

```inform6
[ ShowHeaderInfo version release serial;
    ! The version number is at byte 0
    version = 0->0;
    print "Z-machine version: ", version, "^";

    ! The release number is a word at bytes $02-$03
    release = 0-->1;
    print "Release: ", release, "^";

    ! The serial number is 6 ASCII bytes at $12-$17
    print "Serial: ";
    serial = $12;
    print (char) serial->0, (char) serial->1, (char) serial->2,
          (char) serial->3, (char) serial->4, (char) serial->5, "^";

    ! Screen dimensions (set by interpreter)
    print "Screen: ", 0->$21, " columns x ", 0->$20, " rows^";
];
```

The notation `0->N` reads byte N from address 0 (i.e., header byte N).
The notation `0-->N` reads word N from address 0 (i.e., header bytes
2×N and 2×N+1 as a 16-bit big-endian value).

## §28.4 Data Types and Addressing

The Z-machine is a 16-bit architecture. All registers, stack entries,
local variables, global variables, and most operands are 16-bit values.
Whether a value is interpreted as signed (−32768 to 32767) or unsigned
(0 to 65535) depends on the instruction that uses it.

### §28.4.1 Byte Addresses

Byte addresses are plain 16-bit values that index individual bytes in
memory. They are used to access dynamic and static memory directly:

```inform6
! Read byte at address addr
value = addr->0;

! Write byte at address addr
addr->0 = value;
```

In versions 1–5, byte addresses cover the range 0–65535, limiting directly
addressable memory to 64 KB. In versions 6–8, the addressable range is
extended through packed addresses and offsets.

### §28.4.2 Word Addresses

Word addresses refer to 16-bit words. A word address W corresponds to
byte address W × 2. Word addresses are used primarily for the global
variable table:

```inform6
! Global variable N is stored at word address (globals_base / 2) + N
! The global variables table begins at the address in header $0C-$0D.
! Globals are numbered 0-239.
! In Inform 6, global variables are accessed by name, not by address.

Global my_counter;

[ IncrementCounter;
    my_counter++;
];
```

### §28.4.3 Packed Addresses

Packed addresses are compressed references to routines and strings in high
memory. The Z-machine multiplies a packed address by a version-dependent
packing factor to obtain the actual byte address:

| Version | Packing Factor | Formula |
| ------- | -------------- | ------- |
| 3       | 2              | byte_addr = packed × 2 |
| 4–5     | 4              | byte_addr = packed × 4 |
| 6–7     | 4 + offset     | byte_addr = packed × 4 + offset |
| 8       | 8              | byte_addr = packed × 8 |

In versions 6 and 7, the offset is stored in the header: bytes `$28–$29`
for routine offsets and bytes `$2A–$2B` for string offsets. The offset is
multiplied by 8 before being added to the packed address calculation:

```
byte_addr = packed × 4 + (offset × 8)
```

This scheme allows versions 6 and 7 to address up to 512 KB and 320 KB
respectively despite using 16-bit packed addresses.

### §28.4.4 Signed vs. Unsigned Interpretation

Arithmetic instructions (`@add`, `@sub`, `@mul`, `@div`, `@mod`) treat
operands as signed 16-bit two's complement values. Comparison instructions
(`@jl`, `@jg`) also use signed comparison. Logical operations and
addressing operations treat values as unsigned.

```inform6
[ SignDemo x;
    ! Arithmetic: -1 + 1 = 0 (signed)
    @add (-1) 1 -> x;
    print "Result: ", x, "^";  ! Prints 0

    ! Unsigned comparison: 65535 is stored as $FFFF
    ! which is -1 in signed interpretation
    x = -1;
    if (x == $FFFF) print "Same bit pattern.^";
];
```

## §28.5 The Object Table

The Z-machine maintains a fixed-format object table in dynamic memory.
Every object in a story file has an entry in this table containing its
attributes, object-tree links (parent, sibling, child), and a pointer to
its individual property table.

### §28.5.1 Property Defaults Table

The property defaults table immediately precedes the object entries. It
contains one 16-bit word for each possible property number, providing a
default value that is returned when an object does not explicitly define
that property.

| Version | Properties | Table Size |
| ------- | ---------- | ---------- |
| 3       | 1–31       | 31 words (62 bytes) |
| 4+      | 1–63       | 63 words (126 bytes) |

### §28.5.2 Object Entries

Each object entry is a fixed-size record:

**Version 3 format (9 bytes per object):**

| Offset | Size    | Field |
| ------ | ------- | ----- |
| 0      | 4 bytes | 32 attribute flags (bits 0–31) |
| 4      | 1 byte  | Parent object number |
| 5      | 1 byte  | Sibling object number |
| 6      | 1 byte  | Child object number |
| 7      | 2 bytes | Property table address (byte address) |

**Version 4+ format (14 bytes per object):**

| Offset | Size    | Field |
| ------ | ------- | ----- |
| 0      | 6 bytes | 48 attribute flags (bits 0–47) |
| 6      | 2 bytes | Parent object number |
| 8      | 2 bytes | Sibling object number |
| 10     | 2 bytes | Child object number |
| 12     | 2 bytes | Property table address (byte address) |

Version 3 limits object numbers to 1–255 (one-byte fields). Version 4 and
later allow up to 65535 objects (two-byte fields).

### §28.5.3 Individual Property Tables

Each object's property table is a variable-length structure located in
dynamic memory, pointed to by the object entry. The table begins with:

1. A text-length byte specifying the length of the short name
2. The short name encoded as Z-characters
3. A sequence of property entries in descending order of property number

Each property entry consists of:

**Version 3:**
- A size byte encoding both the property number (bits 0–4) and the data
  length minus one (bits 5–7)
- 1–8 data bytes

**Version 4+:**
- A first size byte: bit 7 clear means 1 or 2 bytes of data (bit 6
  selects); bits 0–5 give the property number
- If bit 7 is set, a second size byte follows: bits 0–5 give the data
  length (0 means 64 bytes); bits 0–5 of the first byte still give the
  property number
- 1–64 data bytes

Properties are always stored in descending numerical order, which allows
the interpreter to perform a linear scan that terminates early when the
desired property number is passed.

```inform6
Object treasure_chest "chest"
  with  name 'chest' 'box' 'treasure',
        description "A sturdy wooden chest bound with iron.",
        capacity 10,
        before [;
            Open: if (self hasnt locked)
                      "You heave the lid open.";
                  "The chest is locked.";
        ],
  has   container openable lockable locked static;

[ ExamineProperties obj addr prop_num size;
    ! The property table address is in the object entry
    ! Use assembly to walk property data
    addr = obj.&name;       ! Address of 'name' property data
    size = obj.#name;       ! Size of 'name' property data in bytes
    print "name property at address ", addr,
          " with ", size, " bytes of data.^";

    ! Property number of 'capacity'
    prop_num = capacity;    ! The compiler resolves this to a number
    print "Capacity value: ", obj.capacity, "^";
];
```

### §28.5.4 Attribute Operations in Assembly

Attributes can be tested, set, and cleared using Z-machine instructions:

```inform6
[ AttributeDemo obj;
    ! Test if object has attribute 'open' (attribute number)
    @test_attr obj open ?is_open;
    print "The object is closed.^";
    jump done;
  .is_open;
    print "The object is open.^";
  .done;

    ! Set an attribute
    @set_attr obj open;

    ! Clear an attribute
    @clear_attr obj open;
];
```

### §28.5.5 Object Tree Operations in Assembly

The Z-machine provides instructions for navigating and modifying the
object tree:

```inform6
[ TreeDemo obj child_obj sibling_obj parent_obj;
    ! Get the parent of an object
    @get_parent obj -> parent_obj;
    if (parent_obj == 0)
        print (name) obj, " has no parent.^";

    ! Get the first child of an object
    @get_child obj -> child_obj ?has_children;
    print (name) obj, " has no children.^";
    return;
  .has_children;
    print "First child: ", (name) child_obj, "^";

    ! Iterate through siblings
    @get_sibling child_obj -> sibling_obj ?has_sibling;
    return;
  .has_sibling;
    print "Next sibling: ", (name) sibling_obj, "^";

    ! Move an object: remove from current parent, insert under new parent
    @remove_obj child_obj;
    @insert_obj child_obj parent_obj;
];
```

## §28.6 The Dictionary

The Z-machine dictionary is a table in static memory that maps encoded
word text to associated data. The parser uses this table to look up words
typed by the player during input processing.

### §28.6.1 Dictionary Structure

The dictionary is located at the byte address stored in header bytes
`$08–$09`. Its format is:

1. **Word separators**: a count byte N followed by N separator characters
   (typically `'.'`, `','`, `'"'`)
2. **Entry length**: a byte giving the total length of each dictionary entry
3. **Entry count**: a signed 16-bit word giving the number of entries. If
   negative, the absolute value is the count but entries are unsorted
   (requiring linear search instead of binary search)
4. **Entries**: a contiguous array of fixed-length dictionary entries

Each dictionary entry contains:

| Component | Size (v3) | Size (v4+) | Description |
| --------- | --------- | ---------- | ----------- |
| Encoded text | 4 bytes (2 words) | 6 bytes (3 words) | Word encoded as Z-characters |
| Data bytes | variable | variable | Flags and values defined by the compiler |

### §28.6.2 Word Encoding

Dictionary words are encoded using Z-characters, a 5-bit encoding scheme.
Three Z-characters are packed into each 16-bit word:

```
  Bit:  15  14 13 12 11 10  9  8  7  6  5  4  3  2  1  0
        [T] [--- char 1 ---] [--- char 2 ---] [--- char 3 ---]
```

Bit 15 is the termination flag; it is set in the last word of the encoded
text. This means:

- Version 3: 6 Z-characters packed into 2 words (4 bytes)
- Version 4+: 9 Z-characters packed into 3 words (6 bytes)

Words shorter than the maximum Z-character length are padded with shift
character 5 (pad character). Words longer than the maximum are truncated.

### §28.6.3 Dictionary Data Bytes

After the encoded text, each dictionary entry contains data bytes whose
meaning is defined by the compiler. Inform 6 uses these bytes to store
flags indicating the word's grammatical role:

| Bit (in data byte) | Meaning |
| ------------------- | ------- |
| Bit 7 of byte 0     | Word is a verb (action word) |
| Bit 6 of byte 0     | Word is used as a preposition |
| Bit 5 of byte 0     | Word is a noun (referring to an object) |
| Bit 2 of byte 0     | Word is a "meta" verb (not an in-game action) |
| Remaining bits       | Verb number, plural flags, etc. |

### §28.6.4 Accessing the Dictionary from Assembly

```inform6
[ DictionaryLookup text_addr result len;
    ! The dictionary can be scanned with @scan_table
    ! But typically you use @tokenise to parse input against it

    ! Read the dictionary address from the header
    len = 0-->4;   ! Header word at $08-$09 = dictionary address
    print "Dictionary at address: ", len, "^";

    ! Number of word separators
    print "Word separators: ", len->0, "^";
];
```

## §28.7 The Instruction Set

The Z-machine instruction set consists of approximately 120 instructions
organized by operand count. Instructions are variable-length: a one- or
two-byte opcode is followed by operands whose types are encoded in the
opcode byte or in a separate operand-type byte.

### §28.7.1 Instruction Forms

Each instruction belongs to one of four forms based on operand count:

| Form | Operand Count | Opcode Range | Description |
| ---- | ------------- | ------------ | ----------- |
| 0OP  | 0             | `$B0`–`$BF` | No operands |
| 1OP  | 1             | `$80`–`$AF` | One operand |
| 2OP  | 2             | `$00`–`$7F` | Two operands |
| VAR  | 0–4 (or 0–8)  | `$C0`–`$FF` | Variable number of operands |
| EXT  | variable      | `$BE` + byte | Extended opcodes (v5+) |

### §28.7.2 Operand Types

Each operand is one of four types:

| Type | Encoding | Size | Description |
| ---- | -------- | ---- | ----------- |
| Large constant | `%00` | 2 bytes | 16-bit value |
| Small constant | `%01` | 1 byte  | 8-bit value (0–255) |
| Variable       | `%10` | 1 byte  | Variable number (0 = stack, 1–15 = local, 16–255 = global) |
| Omitted        | `%11` | 0 bytes | No operand (terminates VAR list) |

### §28.7.3 Zero-Operand Instructions (0OP)

| Opcode | Mnemonic      | Description |
| ------ | ------------- | ----------- |
| `$B0`  | `@rtrue`      | Return true (1) from current routine |
| `$B1`  | `@rfalse`     | Return false (0) from current routine |
| `$B2`  | `@print`      | Print literal string (follows inline) |
| `$B3`  | `@print_ret`  | Print literal string, print newline, return true |
| `$B4`  | `@nop`        | No operation |
| `$B8`  | `@ret_popped` | Return value popped from stack |
| `$B9`  | `@catch`      | Catch current stack frame (v5+, store) |
| `$BA`  | `@quit`       | Halt the interpreter |
| `$BB`  | `@new_line`   | Print a newline character |
| `$BD`  | `@verify`     | Verify story file checksum (branch) |
| `$BF`  | `@piracy`     | Branch (always true in modern interpreters) |

### §28.7.4 One-Operand Instructions (1OP)

| Opcode | Mnemonic        | Description |
| ------ | --------------- | ----------- |
| `$80`  | `@jz`           | Branch if operand is zero |
| `$81`  | `@get_sibling`  | Get sibling object (store, branch) |
| `$82`  | `@get_child`    | Get child object (store, branch) |
| `$83`  | `@get_parent`   | Get parent object (store) |
| `$84`  | `@get_prop_len` | Get length of property data (store) |
| `$85`  | `@inc`          | Increment variable |
| `$86`  | `@dec`          | Decrement variable |
| `$87`  | `@print_addr`   | Print string at byte address |
| `$88`  | `@call_1s`      | Call routine with 0 args (store, v4+) |
| `$89`  | `@remove_obj`   | Remove object from its parent |
| `$8A`  | `@print_obj`    | Print short name of object |
| `$8B`  | `@ret`          | Return operand from current routine |
| `$8C`  | `@jump`         | Unconditional jump (signed offset) |
| `$8D`  | `@print_paddr`  | Print string at packed address |
| `$8E`  | `@load`         | Load value of variable (store) |
| `$8F`  | `@call_1n`      | Call routine with 0 args, discard result (v5+) |

### §28.7.5 Two-Operand Instructions (2OP)

| Opcode  | Mnemonic         | Description |
| ------- | ---------------- | ----------- |
| `$01`   | `@je`            | Branch if first operand equals any other |
| `$02`   | `@jl`            | Branch if signed less than |
| `$03`   | `@jg`            | Branch if signed greater than |
| `$04`   | `@dec_chk`       | Decrement variable and branch if less than |
| `$05`   | `@inc_chk`       | Increment variable and branch if greater than |
| `$06`   | `@jin`           | Branch if object A is child of object B |
| `$07`   | `@test`          | Branch if all bits in bitmap are set |
| `$08`   | `@or`            | Bitwise OR (store) |
| `$09`   | `@and`           | Bitwise AND (store) |
| `$0A`   | `@test_attr`     | Branch if object has attribute |
| `$0B`   | `@set_attr`      | Set attribute of object |
| `$0C`   | `@clear_attr`    | Clear attribute of object |
| `$0D`   | `@store`         | Store value in variable |
| `$0E`   | `@insert_obj`    | Insert object as first child of destination |
| `$0F`   | `@loadw`         | Load word from array (store) |
| `$10`   | `@loadb`         | Load byte from array (store) |
| `$11`   | `@get_prop`      | Get object property value (store) |
| `$12`   | `@get_prop_addr` | Get address of property data (store) |
| `$13`   | `@get_next_prop` | Get next property number (store) |
| `$14`   | `@add`           | Signed addition (store) |
| `$15`   | `@sub`           | Signed subtraction (store) |
| `$16`   | `@mul`           | Signed multiplication (store) |
| `$17`   | `@div`           | Signed division (store) |
| `$18`   | `@mod`           | Signed remainder (store) |
| `$19`   | `@call_2s`       | Call routine with 1 arg (store, v4+) |
| `$1A`   | `@call_2n`       | Call routine with 1 arg, discard result (v5+) |
| `$1B`   | `@set_colour`    | Set foreground/background colours (v5+) |
| `$1C`   | `@throw`         | Throw value to a catch frame (v5+) |

### §28.7.6 Variable-Operand Instructions (VAR)

| Opcode  | Mnemonic          | Description |
| ------- | ----------------- | ----------- |
| `$E0`   | `@call_vs`        | Call routine with up to 3 args (store) |
| `$E1`   | `@storew`         | Store word in array |
| `$E2`   | `@storeb`         | Store byte in array |
| `$E3`   | `@put_prop`       | Set object property value |
| `$E4`   | `@read`/`@aread`  | Read input from keyboard (v4: `@read`; v5+: `@aread`, store) |
| `$E5`   | `@print_char`     | Print a ZSCII character |
| `$E6`   | `@print_num`      | Print a signed number |
| `$E7`   | `@random`         | Generate random number or seed RNG (store) |
| `$E8`   | `@push`           | Push value onto the stack |
| `$E9`   | `@pull`           | Pull value from the stack (store, or variable in v6) |
| `$EA`   | `@split_window`   | Split screen into upper/lower windows |
| `$EB`   | `@set_window`     | Select active window (0 = lower, 1 = upper) |
| `$EC`   | `@call_vs2`       | Call routine with up to 7 args (store, v4+) |
| `$ED`   | `@erase_window`   | Erase a window (v4+) |
| `$EE`   | `@erase_line`     | Erase current line in window (v4+) |
| `$EF`   | `@set_cursor`     | Set cursor position in upper window (v4+) |
| `$F0`   | `@get_cursor`     | Get cursor position (v4+) |
| `$F1`   | `@set_text_style` | Set text style (bold, italic, etc., v4+) |
| `$F2`   | `@buffer_mode`    | Enable/disable output buffering (v4+) |
| `$F3`   | `@output_stream`  | Select/deselect an output stream |
| `$F4`   | `@input_stream`   | Select an input stream |
| `$F5`   | `@sound_effect`   | Play a sound effect (v3+) |
| `$F6`   | `@read_char`      | Read a single character (store, v4+) |
| `$F7`   | `@scan_table`     | Scan a table for a value (store, branch, v4+) |
| `$F8`   | `@not`            | Bitwise NOT (store, v5+) |
| `$F9`   | `@call_vn`        | Call routine with up to 3 args, discard result (v5+) |
| `$FA`   | `@call_vn2`       | Call routine with up to 7 args, discard result (v5+) |
| `$FB`   | `@tokenise`       | Tokenise text buffer against dictionary (v5+) |
| `$FC`   | `@encode_text`    | Encode text to dictionary form (v5+) |
| `$FD`   | `@copy_table`     | Copy or zero a block of memory (v5+) |
| `$FE`   | `@print_table`    | Print a rectangular area of text (v5+) |
| `$FF`   | `@check_arg_count`| Branch if argument N was provided (v5+) |

### §28.7.7 Extended Instructions (EXT, v5+)

Extended instructions use a two-byte opcode: `$BE` followed by the
extended opcode number.

| EXT # | Mnemonic          | Description |
| ----- | ----------------- | ----------- |
| 0     | `@save`           | Save game state (store) |
| 1     | `@restore`        | Restore game state (store) |
| 2     | `@log_shift`      | Logical bit shift (store) |
| 3     | `@art_shift`      | Arithmetic bit shift (store) |
| 4     | `@set_font`       | Set font (store) |
| 5     | `@draw_picture`   | Draw a picture (v6) |
| 6     | `@picture_data`   | Get picture dimensions (v6, branch) |
| 7     | `@erase_picture`  | Erase a picture (v6) |
| 8     | `@set_margins`    | Set window margins (v6) |
| 9     | `@save_undo`      | Save state for undo (store) |
| 10    | `@restore_undo`   | Restore undo state (store) |
| 11    | `@print_unicode`  | Print a Unicode character |
| 12    | `@check_unicode`  | Check if Unicode char is available (store) |
| 13    | `@set_true_colour`| Set true colour (v5+, spec 1.1) |
| 16    | `@move_window`    | Move a window (v6) |
| 17    | `@window_size`    | Set window size (v6) |
| 18    | `@window_style`   | Set window style (v6) |
| 19    | `@get_wind_prop`  | Get window property (v6, store) |
| 20    | `@scroll_window`  | Scroll a window (v6) |
| 21    | `@pop_stack`      | Pop N values from stack (v6) |
| 22    | `@read_mouse`     | Read mouse data (v6) |
| 23    | `@mouse_window`   | Set mouse window (v6) |
| 24    | `@push_stack`     | Push to user stack (v6, branch) |
| 25    | `@put_wind_prop`  | Set window property (v6) |
| 26    | `@print_form`     | Print formatted table (v6) |
| 27    | `@make_menu`      | Define a menu (v6, branch) |
| 28    | `@picture_table`  | Preload pictures (v6) |
| 29    | `@buffer_screen`  | Buffer screen operations (v6, store) |

### §28.7.8 Branch and Store Semantics

**Store instructions** place their result in a variable specified by a
single byte following the operands. Variable 0 means the stack; variables
1–15 are locals; variables 16–255 are globals.

**Branch instructions** include a branch offset after the operands (and
after the store byte, if the instruction also stores). The branch offset
specifies:

- Whether to branch on true or false (bit 7 of first byte)
- The offset itself: 0 means return false; 1 means return true;
  other values give a signed offset from the current PC

In Inform 6 assembly, the `?label` syntax branches on true and `?~label`
branches on false:

```inform6
[ BranchDemo x;
    @jz x ?is_zero;        ! Branch to is_zero if x == 0
    print "x is non-zero.^";
    return;
  .is_zero;
    print "x is zero.^";
];

[ BranchNegateDemo x;
    @jz x ?~not_zero;      ! Branch to not_zero if x != 0
    print "x is zero.^";
    return;
  .not_zero;
    print "x is non-zero.^";
];
```

## §28.8 Assembly Language in Inform 6

Inform 6 supports inline Z-machine assembly instructions using the `@`
prefix. Assembly statements can be freely mixed with high-level Inform
code within any routine.

### §28.8.1 Basic Syntax

The general form of an assembly statement is:

```
@opcode operand1 operand2 ... -> result ?label
```

Where:

- `@opcode` is the Z-machine instruction mnemonic
- Operands are expressions, constants, or variable names
- `-> variable` specifies where to store the result (for store instructions)
- `?label` specifies the branch target (for branch instructions)
- `?~label` specifies a branch target with inverted condition

```inform6
[ AssemblyBasics a b result;
    ! Store instruction: add a and b, store in result
    @add a b -> result;
    print "Sum: ", result, "^";

    ! Branch instruction: jump to label if a equals b
    @je a b ?equal;
    print "Not equal.^";
    return;
  .equal;
    print "Equal.^";
];
```

### §28.8.2 Reading Single Characters

The `@read_char` instruction reads a single keypress without waiting for
a full line of input:

```inform6
[ WaitForKey key;
    print "Press any key to continue...";
    @read_char 1 -> key;
    print "^You pressed character ", key, ".^";
    return key;
];

[ WaitForKeyWithTimeout key;
    ! Read a character with a 5-second timeout
    ! Operands: device (1=keyboard), time (tenths of seconds), routine
    #IfV5;
    @read_char 1 50 0 -> key;
    if (key == 0) print "Timed out!^";
    else print "You pressed: ", (char) key, "^";
    #EndIf;
    return key;
];
```

### §28.8.3 Output Stream Control

The Z-machine provides four output streams. `@output_stream` activates or
deactivates them:

| Stream | Destination | Enable | Disable |
| ------ | ----------- | ------ | ------- |
| 1      | Screen      | `@output_stream 1` | `@output_stream -1` |
| 2      | Transcript  | `@output_stream 2` | `@output_stream -2` |
| 3      | Memory table| `@output_stream 3 table` | `@output_stream -3` |
| 4      | Script file | `@output_stream 4` | `@output_stream -4` |

Stream 3 is particularly useful for capturing printed output into a memory
buffer:

```inform6
Array text_buffer -> 160;

[ CaptureOutput txt;
    ! Redirect output to a memory table
    @output_stream 3 text_buffer;
    print "This text is captured into memory.";
    @output_stream -3;

    ! text_buffer-->0 now contains the number of characters written
    ! text_buffer->2 onwards contains the actual characters
    print "Captured ", text_buffer-->0, " characters.^";

    ! Print the captured text manually
    txt = text_buffer-->0;
    print "Captured text: ";
    for (: txt > 0 : txt--)
        print (char) text_buffer->(2 + text_buffer-->0 - txt);
    new_line;
];
```

Stream 3 can be nested up to 16 levels deep in most interpreters, allowing
output capture within output capture.

### §28.8.4 Table Scanning

The `@scan_table` instruction searches a table for a specific value:

```inform6
Array data_table -->  10 20 30 40 50 60 70 80 90 100;

[ FindInTable sought result;
    ! Scan data_table for the value 'sought'
    ! Arguments: value, table_addr, table_length, form
    ! form $82 = word entries, field width 2
    @scan_table sought data_table 10 $82 -> result ?found;
    print "Value ", sought, " not found.^";
    return 0;
  .found;
    print "Value ", sought, " found at address ", result, ".^";
    return result;
];
```

### §28.8.5 Memory Block Operations

The `@copy_table` instruction copies or zeroes blocks of memory:

```inform6
Array source_buf -> 64;
Array dest_buf -> 64;

[ CopyMemory;
    ! Copy 64 bytes from source_buf to dest_buf
    @copy_table source_buf dest_buf 64;

    ! Zero out 64 bytes of dest_buf (pass 0 as source)
    @copy_table dest_buf 0 64;
];
```

When the destination address is 0, the source table is zeroed. When the
size is negative, the copy proceeds backwards (safe for overlapping
regions).

### §28.8.6 Save and Restore

```inform6
[ SaveGame result;
    #IfV5;
    @save -> result;
    switch (result) {
        0: print "Save failed.^";
        1: print "Game saved.^";
        2: print "Game restored.^";
    }
    #EndIf;
    #IfV3;
    @save ?saved;
    print "Save failed.^";
    return;
  .saved;
    print "Game saved.^";
    #EndIf;
];
```

In version 5+, `@save` is a store instruction that returns 0 on failure,
1 on successful save, and 2 when execution resumes after a successful
restore. In version 3, `@save` is a branch instruction.

### §28.8.7 Undo Support

```inform6
[ SaveUndo result;
    @save_undo -> result;
    switch (result) {
        0: print "Undo is not available.^";
        1: print "Undo state saved.^";
        2: print "Undone.^";
    }
];

[ PerformUndo result;
    @restore_undo -> result;
    ! If we reach here, restore_undo failed
    print "Undo failed.^";
];
```

Like `@save`, `@save_undo` returns 2 when execution resumes after a
successful `@restore_undo`.

### §28.8.8 Tokenisation and Dictionary Encoding

The `@tokenise` instruction parses a text buffer against the dictionary:

```inform6
Array input_text -> 64;
Array parse_buffer -> 64;

[ ManualParse;
    ! Set up the text buffer: byte 0 = max chars
    input_text->0 = 62;

    ! Read input from the keyboard
    @aread input_text parse_buffer -> 0;

    ! parse_buffer now contains:
    !   byte 0: max entries
    !   byte 1: number of entries found
    !   each entry (4 bytes): dict addr (word), text length, text offset
    print "Words found: ", parse_buffer->1, "^";
];
```

The `@encode_text` instruction encodes a fragment of text into dictionary
form:

```inform6
Array coded_text -> 10;

[ EncodeWord;
    ! Encode 4 characters starting at position 0 of input_text+2
    ! into dictionary-encoded form in coded_text
    @encode_text input_text 4 0 coded_text;
];
```

### §28.8.9 Version-Conditional Assembly

Inform 6 provides directives for conditional compilation based on the
target Z-machine version:

```inform6
[ VersionDemo;
    #IfV3;
    print "Running on Z-machine version 3.^";
    ! V3 uses @save as branch, not store
    @save ?saved_ok;
    print "Save failed.^";
    return;
  .saved_ok;
    print "Saved.^";
    #EndIf;

    #IfV5;
    print "Running on Z-machine version 5 or later.^";
    ! V5 uses @save as store
    @save -> sp;
    #EndIf;
];
```

The `TARGET_ZCODE` symbol is defined when compiling for any Z-machine
version. The `TARGET_GLULX` symbol is defined when compiling for Glulx.
These can be used to write code portable across both virtual machines:

```inform6
[ PrintUnicodeChar ch;
    #Ifdef TARGET_ZCODE;
    @print_unicode ch;
    #Ifnot;
    @streamunichar ch;
    #EndIf;
];
```

## §28.9 Story File Size Limits

The Z-machine imposes hard upper limits on story file size. These limits
arise from the packed address mechanism and the 16-bit address space.

### §28.9.1 Size Limits by Version

| Version | Maximum Story File Size | Packing Factor | Practical Limit |
| ------- | ----------------------- | -------------- | --------------- |
| 3       | 128 KB                  | 2              | 128 KB          |
| 4       | 256 KB                  | 4              | 256 KB          |
| 5       | 256 KB                  | 4              | 256 KB          |
| 6       | 512 KB                  | 4 + offset     | 512 KB          |
| 7       | 320 KB                  | 4 + offset     | 320 KB          |
| 8       | 512 KB                  | 8              | 512 KB          |

These limits are absolute architectural constraints. A story file that
exceeds the limit for its version cannot be compiled. If a project
approaches the limit for version 5 (256 KB), switching to version 8
(512 KB) provides additional capacity. For projects that exceed even
version 8 limits, the Glulx virtual machine removes size restrictions
entirely.

### §28.9.2 Monitoring File Size

The Inform 6 compiler reports the story file size in its compilation
summary. Use the `-s` switch for detailed statistics or the `-z` switch
for Z-machine–specific information:

```inform6
! Command-line usage (not Inform 6 code):
!   inform -v5 -s source.inf
!   inform -v8 -z source.inf
```

The compiler issues an error if the compiled output exceeds the
architectural limit for the target version.

### §28.9.3 Strategies for Reducing File Size

When a project approaches size limits, the following strategies can
reduce the compiled output:

- Use the `Abbreviate` directive to define common text strings that
  the compiler can substitute with shorter references
- Minimize long `print` statements by reusing string constants
- Compile for version 8 instead of version 5
- Move to Glulx if the project exceeds Z-machine limits

## §28.10 Text Encoding (ZSCII)

The Z-machine uses its own character encoding system called ZSCII
(Zork Standard Code for Information Interchange) for all text processing.
ZSCII characters are encoded into compact 5-bit units called Z-characters
for storage in story files.

### §28.10.1 The ZSCII Character Set

ZSCII is an 8-bit character set (values 0–1023, though only a subset is
defined). The printable range largely corresponds to ISO 8859-1 (Latin-1).
Key ranges include:

| ZSCII Code | Character |
| ---------- | --------- |
| 0          | Null (output only: no effect) |
| 8          | Delete (input only) |
| 13         | Newline (carriage return) |
| 27         | Escape (input only) |
| 32–126     | Standard ASCII printable characters |
| 129–132    | Cursor keys (up, down, left, right — input only) |
| 133–144    | Function keys F1–F12 (input only) |
| 145–154    | Keypad 0–9 (input only) |
| 155–251    | Extra characters (accented letters, defined by Unicode translation table) |
| 252        | Menu click (v6, input only) |
| 253        | Double-click (v6, input only) |
| 254        | Single-click (v6, input only) |

### §28.10.2 Z-Characters and Alphabets

Text is stored in story files using Z-characters: 5-bit values packed
three to a 16-bit word. The Z-machine defines three alphabets:

**Alphabet A0 (default — lowercase):**

| Z-char | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 | 24 | 25 | 26 | 27 | 28 | 29 | 30 | 31 |
| ------ | - | - | - | - | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- |
| Char   | a | b | c | d | e  | f  | g  | h  | i  | j  | k  | l  | m  | n  | o  | p  | q  | r  | s  | t  | u  | v  | w  | x  | y  | z  |

**Alphabet A1 (uppercase):**

| Z-char | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 | 24 | 25 | 26 | 27 | 28 | 29 | 30 | 31 |
| ------ | - | - | - | - | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- |
| Char   | A | B | C | D | E  | F  | G  | H  | I  | J  | K  | L  | M  | N  | O  | P  | Q  | R  | S  | T  | U  | V  | W  | X  | Y  | Z  |

**Alphabet A2 (punctuation and special):**

| Z-char | 6     | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 | 24 | 25 | 26 | 27 | 28 | 29 | 30 | 31 |
| ------ | ----- | - | - | - | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- |
| Char   | (esc) | ^ | 0 | 1 | 2  | 3  | 4  | 5  | 6  | 7  | 8  | 9  | .  | ,  | !  | ?  | _  | #  | '  | "  | /  | \  | -  | :  | (  | )  |

Special Z-character values:

| Z-char | Meaning |
| ------ | ------- |
| 0      | Space character |
| 1      | Abbreviation (v2+): references abbreviation 0–31 |
| 2      | Abbreviation (v3+): references abbreviation 32–63 |
| 3      | Abbreviation (v3+): references abbreviation 64–95 |
| 4      | Shift to alphabet A1 (for next character only) |
| 5      | Shift to alphabet A2 (for next character only) |

### §28.10.3 The Escape Sequence

Z-character 6 in alphabet A2 introduces a 10-bit ZSCII escape sequence.
The next two Z-characters form a 10-bit value (first Z-char × 32 + second
Z-char) representing the ZSCII code of an arbitrary character. This
mechanism allows any ZSCII character to be encoded, even those not present
in the standard alphabets.

```
Encoding of ZSCII character 233 (é):
  Z-char 5      → shift to A2
  Z-char 6      → escape
  Z-char 7      → high 5 bits: 233 / 32 = 7
  Z-char 9      → low 5 bits:  233 mod 32 = 9
```

### §28.10.4 Abbreviations

Abbreviations provide a text compression mechanism. The story file
contains a table of up to 96 abbreviation strings (32 in version 3).
When a Z-character sequence 1/2/3 followed by a Z-character N appears in
encoded text, it is replaced by abbreviation string number
(Z-char − 1) × 32 + N.

| Z-char | Abbreviation Numbers |
| ------ | -------------------- |
| 1      | 0–31                 |
| 2      | 32–63                |
| 3      | 64–95                |

### §28.10.5 The Abbreviate Directive

In Inform 6, the `Abbreviate` directive defines abbreviation strings that
the compiler uses during text compression:

```inform6
Abbreviate "the ";
Abbreviate "you ";
Abbreviate "and ";
Abbreviate "The ";
Abbreviate "ing ";
Abbreviate "tion";
Abbreviate ". ";
Abbreviate ", ";
Abbreviate "have ";
Abbreviate "with ";
```

The compiler analyses all text in the story file and substitutes
abbreviation references wherever the abbreviated string appears. The
`-e` compiler switch reports on abbreviation efficiency. Up to 96
abbreviations may be defined (32 in version 3). Choosing effective
abbreviations can significantly reduce story file size, as each
abbreviation reference consumes only 2 Z-characters regardless of the
length of the abbreviated string.

### §28.10.6 Custom Alphabet Tables

Version 5 and later allow the story file to replace the default alphabets
with custom ones by setting the alphabet table address in header bytes
`$34–$35`. This is useful for languages that need different characters in
the primary alphabets:

```inform6
! The Zcharacter directive configures custom alphabets.
! Each alphabet definition is a string of 26 characters.

Zcharacter "abcdefghijklmnopqrstuvwxyz"
           "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
           " ^0123456789.,!?_#'\"/\\-:()";
```

The `Zcharacter` directive replaces one or more of the three alphabets.
Characters that appear frequently in the game text should be placed in
alphabet A0 for maximum compression.

### §28.10.7 Unicode Translation Table

The Z-machine maps ZSCII codes 155–251 to Unicode code points via a
translation table. The default table provides standard Western European
accented characters. The `Zcharacter` directive can also modify this
mapping:

```inform6
! Map ZSCII 155 onwards to specific Unicode code points
Zcharacter table + '@{e9}' '@{e8}' '@{ea}' '@{eb}';
```

The `@print_unicode` instruction (extended opcode 11) prints any Unicode
character directly by its code point, bypassing the ZSCII translation
table:

```inform6
[ PrintAccented;
    ! Print é (Unicode U+00E9) directly
    @print_unicode $E9;
    new_line;

    ! Check if a Unicode character is available for output
    @check_unicode $E9 -> sp;
    @jz sp ?not_available;
    print "Character is available.^";
    return;
  .not_available;
    print "Character is not available.^";
];
```
