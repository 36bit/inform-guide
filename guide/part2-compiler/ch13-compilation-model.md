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

# Chapter 13: Compilation Model

This chapter describes how the compiler transforms source code into a
finished story file. It covers the single-pass architecture, the
backpatching mechanism that resolves forward references, the veneer
routines injected as runtime support, and the construction of the major
data structures — dictionary, object tree, grammar tables, and memory
layout — that make up the final output.

---

## 13.1 Overview of the Compilation Process

Inform 6 is a **single-pass compiler with backpatching**. The compiler
reads source files sequentially, parsing directives and routine bodies,
generating code and building tables as it goes. Because only one pass
over the source is made, forward references — a routine calling another
routine defined later in the file, or an object property referring to a
not-yet-defined object — cannot be resolved immediately. Instead, the
compiler records placeholder values alongside **backpatch markers** that
indicate what kind of reference needs to be filled in. After the main
pass completes, a backpatching phase walks all generated code and data,
replacing placeholders with final addresses.

This design keeps the compiler fast (each source line is read exactly
once) while still allowing the flexible definition order that Inform
programmers expect. The cost is a modest amount of bookkeeping in the
form of backpatch tables, which are discarded once the story file is
assembled.

At a high level, compilation proceeds through three broad stages:

1. **Initialization** — parse command-line options, allocate memory,
   set up subsystem state.
2. **Main pass** — read and parse source code, generate bytecode,
   build symbol tables, object definitions, dictionary entries, and
   grammar tables.
3. **Finalization** — compile veneer routines, sort the dictionary,
   eliminate dead code, backpatch forward references, and assemble the
   story file.

---

## 13.2 Compilation Phases

The compiler's top-level entry point is the `compile()` function. It orchestrates the entire process:

```
compile()
  ├── execute_icl_header()        ICL command processing
  ├── set_compile_variables()     Configure target platform
  ├── init_vars()                 Reset all subsystem state
  ├── allocate_arrays()           Allocate working memory
  ├── run_pass()                  *** Main compilation ***
  ├── output_file()               Write story file to disk
  └── free memory / close files
```

### 13.2.1 Initialization

Before any source code is read, the compiler:

- Processes ICL (Inform Command Language) headers embedded in the
  source file or in separate `.icl` files.
- Selects the target platform (Z-machine version 3–8 or Glulx) and
  configures platform-dependent constants such as word size, maximum
  object count, and address scale factors.
- Calls `init_vars()` to reset every subsystem (lexer, assembler,
  symbol table, object manager, etc.) to its initial state.
- Allocates dynamic arrays for code storage, symbol tables, dictionary
  entries, and other working data.

### 13.2.2 The Main Pass

The heart of compilation is `run_pass()`, which executes the following
sequence:

```
run_pass()
  1.  lexer_begin_prepass()           Initialise lexer
  2.  files_begin_prepass()           Initialise file tracking
  3.  load_sourcefile()               Open the main source file
  4.  begin_pass()                    Initialise all subsystems
        ├── compile_initial_routine() Create Main__ entry point
        └── make_class() × 4         Create Class, Object,
                                       Routine, String metaclasses
  5.  parse_program()                 *** Parse source code ***
  6.  ensure_builtin_globals()        Guarantee required globals exist
  7.  find_the_actions()              Identify action routines
  8.  issue_unused_warnings()         Warn about unused symbols
  9.  compile_veneer()                Generate veneer routines
  10. lexer_endpass()                 Finalise lexer state
  11. close_all_source()              Close all source files
  12. sort_dictionary()               Sort dictionary into order
  13. sort_actions()                  Sort actions (if needed)
  14. locate_dead_functions()         Dead code elimination
  15. locate_dead_grammar_lines()     Remove unused grammar
  16. construct_storyfile()           Assemble the output file
```

Step 4 deserves special attention: before any user code is parsed, the
compiler creates the `Main__` wrapper routine (which calls the user's
`Main` and then quits) and the four built-in metaclass objects — `Class`,
`Object`, `Routine`, and `String` — that underpin Inform's object
system. These are always objects 1–4 in every story file.

Step 5 (`parse_program`) is where the compiler spends most of its time.
It reads directives (`Object`, `Class`, `Verb`, `Array`, `Global`,
`Constant`, etc.), compiles routine bodies into bytecode, and records
dictionary words, grammar rules, and symbol definitions as it goes.

### 13.2.3 Finalization

After all source has been parsed, the compiler performs several
post-processing steps before the story file can be written:

- **Veneer compilation** (§13.5) generates any runtime support routines
  that were referenced but not user-defined.
- **Dictionary sorting** (§13.6) flattens the red-black insertion tree
  into the sorted order required by the story file format.
- **Dead code elimination** (§13.10) optionally removes unreachable
  routines.
- **Story file construction** (§13.9) lays out all data areas, applies
  backpatches, and writes the final binary.

---

## 13.3 Symbol Resolution and Forward References

The compiler maintains a **symbol table** that maps identifiers to their
types and values. When a symbol is first encountered — whether in a
definition or a reference — it is entered into the table. Symbols carry
a type (constant, global variable, routine, object, class, attribute,
property, action, etc.) and a value that depends on the type.

### Permitted Forward References

Because Inform is single-pass, a symbol may be *referenced* before it is
*defined*. The compiler handles this by inserting a backpatch marker at
the point of use:

```inform6
! Forward reference: TreasureRoom is not yet defined
Object -> Corridor "Corridor"
  with description "A long hallway leading to the treasure room.",
       door_to TreasureRoom;       ! SYMBOL_MV marker recorded

! ... later in the source ...

Object TreasureRoom "Treasure Room"
  with description "Glittering gold lines every surface.";
```

Most forward references are legal:

- Routine calls to not-yet-defined routines
- Object references in property values
- Constants used in expressions
- Dictionary words (always resolved at sort time)

### Restrictions

Some constructs require their targets to be defined first:

- A class used as a superclass in a `Class` or `Object` definition must
  already be defined, because the compiler needs to copy inherited
  properties immediately.
- `Attribute` and `Property` directives must precede any use in object
  definitions.
- `Ifdef` / `Ifndef` tests are evaluated immediately, so the tested
  symbol's definition status matters at that point in the source.

---

## 13.4 Backpatching

Backpatching is the mechanism by which the compiler resolves addresses
and values that were unknown during the main pass.

### 13.4.1 Backpatch Markers

When the compiler generates code or data containing a forward reference,
it stores a **marker type** alongside the placeholder value. Each marker
type tells the backpatcher what kind of final value to substitute:

| Marker | Value | Meaning |
| ------ | ----- | ------- |
| `NULL_MV` | 0 | No backpatch needed |
| `DWORD_MV` | 1 | Dictionary word address |
| `STRING_MV` | 2 | Static string address |
| `INCON_MV` | 3 | System constant (table address) |
| `IROUTINE_MV` | 4 | Internal routine address |
| `VROUTINE_MV` | 5 | Veneer routine address |
| `ARRAY_MV` | 6 | Dynamic array address |
| `NO_OBJS_MV` | 7 | Total number of objects |
| `INHERIT_MV` | 8 | Inherited common property value |
| `INHERIT_INDIV_MV` | 9 | Inherited individual property value |
| `MAIN_MV` | 10 | Address of `Main` routine |
| `SYMBOL_MV` | 11 | Forward reference to any symbol |
| `VARIABLE_MV` | 12 | Global variable reference [Glulx] |
| `IDENT_MV` | 13 | Property identifier number |
| `INDIVPT_MV` | 14 | Individual property table address |
| `ACTION_MV` | 15 | Action number |
| `OBJECT_MV` | 16 | Object number [Glulx] |
| `STATIC_ARRAY_MV` | 17 | Static array address |
| `ERROR_MV` | 18 | Error placeholder |

Additional internal markers (`LABEL_MV`, `DELETED_MV`, `BRANCH_MV`)
are used during code assembly for jump targets and branch optimization
but are never written to the story file.

### 13.4.2 Backpatch Tables

The compiler maintains three backpatch tables that record where markers
appear:

- **`zcode_backpatch_table`** — markers in the routine code area
- **`zmachine_backpatch_table`** — markers in other story file areas
  (property tables, grammar data, etc.)
- **`staticarray_backpatch_table`** — markers in static array data

Each table entry records the marker type and the byte offset within its
area where the placeholder value was written.

### 13.4.3 Resolution

During story file construction, the backpatcher walks each table and
calls `backpatch_value_z()` (Z-machine) or `backpatch_value_g()` (Glulx)
to compute the final value for each marker. The resolution logic differs
by marker type:

- **`STRING_MV`**: the placeholder (a string number) is converted to
  the string's address in the strings area, scaled by the platform's
  packing factor.
- **`IROUTINE_MV`**: the placeholder (a routine index) is converted to
  the routine's address in the code area, again with packing.
- **`VROUTINE_MV`**: looked up in `veneer_routine_address[]`, then
  offset into the code area.
- **`DWORD_MV`**: the placeholder (a dictionary accession number) is
  converted to the word's byte offset in the dictionary table.
- **`SYMBOL_MV`**: the symbol table is consulted to find the symbol's
  final value, which may itself require further interpretation based
  on the symbol's type.
- **`NO_OBJS_MV`**: replaced with the actual count of objects.

[Z-machine] Addresses in the code and string areas are **packed** by
dividing by a scale factor (2 for version 3, 4 for versions 4–7, 8 for
version 8). The backpatcher applies this scaling automatically.

[Glulx] All addresses are full 32-bit byte offsets; no packing is
needed.

---

## 13.5 Veneer Routines

The compiler injects a set of **veneer routines** — small, compiler-
generated functions that provide runtime support for language features
that are too complex to expand inline. These routines are defined in
the compiler's veneer source module as Inform source text, compiled by
the compiler itself during the finalization phase.

### 13.5.1 Why Veneer Routines Exist

Many Inform language features translate to routine calls rather than
inline instruction sequences. For example, the expression `obj.prop`
(reading an object property) may require searching common property
tables, checking individual properties, consulting class inheritance,
and raising a runtime error if the property is not found. Generating
this logic inline at every use site would bloat the output enormously.
Instead, the compiler emits a call to the veneer routine `RV__Pr`, which
handles all the complexity in one place.

### 13.5.2 Demand-Driven Compilation

Veneer routines are compiled **on demand**. Each of the 48 possible
veneer routines has a state:

| State | Constant | Meaning |
| ----- | -------- | ------- |
| Unused | `VR_UNUSED` | Not referenced; will not be compiled |
| Called | `VR_CALLED` | Referenced by user code or another veneer; needs compilation |
| Compiled | `VR_COMPILED` | Already compiled into the code area |

When the compiler encounters a reference to a veneer routine (for
example, while generating code for a property access), it marks the
routine as `VR_CALLED`. After the main pass, `compile_veneer()` loops
through all 48 routines, compiling any that are in the `VR_CALLED`
state. Because compiling one veneer routine may reference others, the
loop repeats until no new routines are marked:

```
compile_veneer():
    repeat
        changed = false
        for each veneer routine V:
            if V.state == VR_CALLED:
                parse and compile V's source text
                V.state = VR_COMPILED
                changed = true
    until not changed
```

This ensures that the transitive closure of all required veneer routines
is compiled, but routines that are never needed are omitted entirely.
A typical game using the standard library might include 20–30 of the 48
veneer routines; a minimal program may need only a handful.

### 13.5.3 Catalog of Veneer Routines

The following table lists the major categories of veneer routines. The
full set of 48 is defined in the `VRs_z[]` and `VRs_g[]` arrays in the
compiler's veneer module.

**Statement support:**

| Routine | Purpose |
| ------- | ------- |
| `Box__Routine` | Implement the `box` statement [Z-machine only] |

**Default library stubs:**

| Routine | Purpose |
| ------- | ------- |
| `R_Process` | Default action handler (prints action tuple) |
| `DefArt` | Print definite article ("the") |
| `InDefArt` | Print indefinite article ("a") |
| `CDefArt` | Print capitalized definite article ("The") |
| `CInDefArt` | Print capitalized indefinite article ("A") |
| `PrintShortName` | Print an object's short name |
| `EnglishNumber` | Print a number in English words |
| `Print__PName` | Print a property name (for debugging) |

These stubs provide minimal default behavior when the standard library
is not included. The library replaces them with fuller implementations.

**Property access and message passing:**

| Routine | Purpose |
| ------- | ------- |
| `RV__Pr` | Read the value of an object's property |
| `WV__Pr` | Write a value to an object's property |
| `CA__Pr` | Call a property as a routine (message sending) |
| `RA__Pr` | Read the address of a property value |
| `RL__Pr` | Read the length (in bytes) of a property value |
| `OP__Pr` | Test whether an object provides a property |
| `CP__Tab` | Search common property tables |

**Object and class operations:**

| Routine | Purpose |
| ------- | ------- |
| `OC__Cl` | Test `ofclass` — is an object a member of a class? |
| `Copy__Primitive` | Deep-copy an object's properties |
| `Cl__Ms` | Handle class messages (`create`, `destroy`, `copy`, etc.) |
| `Z__Region` | Determine value type (object / routine / string / nothing) |
| `Meta__class` | Return the metaclass of a value |
| `Unsigned__Compare` | Unsigned integer comparison |

**Property increment/decrement:**

| Routine | Purpose |
| ------- | ------- |
| `IB__Pr` | Pre-increment a property (`++obj.prop`) |
| `IA__Pr` | Post-increment a property (`obj.prop++`) |
| `DB__Pr` | Pre-decrement a property (`--obj.prop`) |
| `DA__Pr` | Post-decrement a property (`obj.prop--`) |

**Superclass access:**

| Routine | Purpose |
| ------- | ------- |
| `RA__Sc` | Implement the superclass (`::`) operator |

**Runtime error checking:**

| Routine | Purpose |
| ------- | ------- |
| `RT__Err` | Report runtime errors (with error code lookup) |

When strict error checking is enabled (`-S` switch), additional veneer
routines (`RT__ChT`, `RT__ChR`, `RT__ChG`, `RT__ChLDB`, `RT__ChLDW`,
`RT__ChSTB`, `RT__ChSTW`, `RT__ChPrintC`, `RT__ChPrintA`,
`RT__ChPrintS`, `RT__ChPrintO`, etc.) are compiled to guard property
reads, array accesses, and print operations with bounds and type checks.

### 13.5.4 Replacing Veneer Routines

A veneer routine can be replaced by a user-defined version using the
`Replace` directive:

```inform6
Replace RT__Err;

[ RT__Err crime obj id size;
    ! Custom error handler: log to a buffer instead of printing
    ...
];
```

When a veneer routine is replaced, the compiler uses the user's
definition and does not compile the built-in version. This is rarely
needed but can be useful for custom runtime error handling or
internationalization of article printing.

---

## 13.6 Vocabulary (Dictionary) Construction

The story file dictionary is one of the most important data structures
for interactive fiction, as it is used by the parser to recognize words
in player input. The compiler builds it incrementally during the main
pass and sorts it during finalization.

### 13.6.1 Insertion: Red-Black Tree

Every time the compiler encounters a dictionary word — in a `Verb`
directive, a `name` property, a `'word'` literal, or a `Dictionary`
directive — it inserts the word into a **red-black tree** (a self-
balancing binary search tree). This data structure ensures that insertion and lookup are O(log n) operations
even as the dictionary grows to thousands of entries.

Each tree node stores left/right child indices and a color bit. The
actual word data (encoded sort keys and dictionary flags) is stored in
parallel arrays indexed by accession number.

If the same word is inserted more than once, the dictionary flags are
**merged** by bitwise OR. This is how a word can simultaneously be
marked as a verb, a noun, and a preposition if it appears in multiple
contexts.

### 13.6.2 Sorting

After the main pass, `sort_dictionary()` performs an in-order traversal
of the red-black tree, assigning each node a position in the final
sorted order. The result is stored in `final_dict_order[]`, an array
that maps each word's accession number to its position in the output
dictionary.

The story file format requires the dictionary to be sorted because the
interpreter performs binary search on it when parsing player input.

### 13.6.3 Dictionary Format

[Z-machine] The dictionary table begins with a header:

```
Byte  0:       Number of word-separator characters (typically 3)
Bytes 1–N:     The separator characters (usually '.', ',', '"')
Byte  N+1:     Length of each entry in bytes
Bytes N+2–N+3: Number of entries (big-endian 16-bit)
Bytes N+4+:    Dictionary entries in sorted order
```

Each entry contains the word's encoded text (4 bytes in version 3, 6
bytes in versions 4+) followed by flag bytes indicating the word's
grammatical roles.

[Glulx] The dictionary uses a 4-byte header (number of entries as a
32-bit value), followed by entries whose word fields use either 1-byte
or 4-byte characters depending on the `DICT_CHAR_SIZE` setting.

---

## 13.7 Object Tree Construction

Objects are the central data structure of most Inform programs. The
compiler builds the object tree — the hierarchy of parent, child, and
sibling pointers — as objects are defined during the main pass.

### 13.7.1 Object Numbering

Objects are assigned sequential numbers starting from 1 as they are
defined. The first four objects (1–4) are always the built-in metaclass
objects created during initialization:

| Object # | Name | Role |
| -------- | ---- | ---- |
| 1 | `Class` | The metaclass of all classes |
| 2 | `Object` | The default metaclass of all objects |
| 3 | `Routine` | The metaclass of routine values |
| 4 | `String` | The metaclass of string values |

User-defined objects start from number 5.

### 13.7.2 Tree Structure

Each object record stores three tree pointers: **parent**, **child**
(first child), and **sibling** (next sibling). Together, these encode an
ordered tree:

```
                    ┌─────────┐
                    │ nothing │  (object 0 = no object)
                    └────┬────┘
                         │ child
                    ┌────┴────┐
                    │ Kitchen │
                    └────┬────┘
              ┌──────────┼──────────┐
              │ child    │          │ sibling
         ┌────┴────┐    │    ┌─────┴─────┐
         │  Table  │    │    │   Chair    │
         └────┬────┘    │    └────┬──────┘
              │ child              │ child
         ┌────┴────┐         ┌────┴────┐
         │  Lamp   │         │ Cushion │
         └─────────┘         └─────────┘
```

Arrow notation in source code controls placement:

```inform6
Object Kitchen "Kitchen";

Object -> Table "table";       ! child of Kitchen

Object -> -> Lamp "lamp";      ! child of Table (2 arrows = 2 deep)

Object -> Chair "chair";       ! sibling of Table (1 arrow = child of Kitchen)

Object -> -> Cushion "cushion"; ! child of Chair
```

The arrow depth is relative to the most recently defined object at the
enclosing depth level.

### 13.7.3 Class Instances

When an object is defined as an instance of a class (or multiple
classes), the compiler copies inherited property values and attribute
settings from the class definition. It also records the class membership
in a table used at runtime by the `ofclass` operator and the
`metaclass()` function.

### 13.7.4 Platform Differences

[Z-machine] Each object table entry has a fixed size: 9 bytes in
version 3 (32 attributes, 1-byte tree pointers) or 14 bytes in versions
4+ (48 attributes, 2-byte tree pointers). A separate property table
holds each object's property values, pointed to by an offset in the
object entry.

[Glulx] Object entries are variable-length, with configurable attribute
count (`NUM_ATTR_BYTES`), 4-byte tree pointers, and additional extension
bytes (`GLULX_OBJECT_EXT_BYTES`). The property table format is more
flexible, supporting larger property numbers and values.

---

## 13.8 Grammar Table Generation

The grammar table encodes the verb definitions that drive the parser.
It is built during the main pass as `Verb` and `Extend` directives are
processed, and finalized during story file construction.

### 13.8.1 Structure

The grammar system has three levels:

```
Grammar Table
  └── Verb entries (one per verb word group)
        └── Grammar lines (one per syntax pattern)
              └── Tokens (individual parsing elements)
```

A `Verb` directive defines one or more verb synonyms and a set of
grammar lines. Each grammar line is a sequence of tokens describing a
valid command syntax, ending with an action to invoke:

```inform6
Verb 'take' 'grab' 'pick'
    * noun                          -> Take
    * 'up' noun                     -> Take
    * noun 'from' noun              -> Remove
    * multiheld 'from' noun         -> Remove;
```

### 13.8.2 Token Types

Grammar tokens specify what kind of input the parser should expect at
each position. Common token types include:

- **Noun tokens** (`noun`, `held`, `multi`, `multiheld`, etc.) — match
  an object the player can refer to, with various scope filters.
- **Literal tokens** (`'word'`) — match a specific dictionary word.
- **Routine tokens** (`scope=Routine`) — call a user-defined routine to
  determine scope or parse custom input.
- **Attribute tokens** (`creature`) — match objects with a specific
  attribute.

### 13.8.3 Grammar Versions

The compiler supports three grammar table formats, selected by the
`$GRAMMAR_VERSION` setting:

- **Version 1**: the original format, now obsolete. Stores grammar lines
  with fixed-size token entries.
- **Version 2** (default): stores grammar lines as flat arrays of
  variable-length tokens. More compact and flexible than version 1.
- **Version 3**: an extended version of the format. Used when
  `$GRAMMAR_META_FLAG` is set; supports action sorting and metadata.

The grammar table is written into the static memory area of the story
file, and its starting address is recorded in the file header.

---

## 13.9 Code Area, String Area, and Static Data Layout

The final story file is a single binary blob with a carefully specified
layout. The compiler's `construct_storyfile()` function
assembles all the pieces.

### 13.9.1 Z-Machine Story File Layout

[Z-machine] The story file is divided into three regions:

```
┌──────────────────────────────────────┐  Byte 0
│              Header (64 bytes)       │
├──────────────────────────────────────┤
│  Abbreviation strings table          │
│  Header extension table              │
│  Property defaults (62/126 bytes)    │
│  Object table                        │
│  Object property values              │
│  Class number table                  │
│  Individual property table           │
│  Global variables (240 × 2 bytes)    │
│  Dynamic arrays                      │  ← Dynamic memory
├──────────────────────────────────────┤  static_memory_offset
│  Grammar table                       │
│  Action routines table               │
│  Preaction routines table            │
│  Adjectives table                    │
│  Dictionary                          │
│  Static arrays                       │  ← Static memory
├──────────────────────────────────────┤  code_offset
│  Compiled routine bytecode           │  ← High memory (code area)
├──────────────────────────────────────┤  strings_offset
│  Static string data (Z-encoded)      │  ← High memory (strings area)
└──────────────────────────────────────┘
```

Dynamic memory (below `static_memory_offset`) can be read and written
by the game at runtime. Static memory and high memory are read-only.
The interpreter may swap high memory out to disk and page it in on
demand.

Routine addresses in the code area and string addresses in the strings
area are stored as **packed addresses**: the actual byte offset divided
by a scale factor that depends on the Z-machine version (2 for v3, 4 for
v4–7, 8 for v8). This packing allows 16-bit pointers to address more
than 64 KB of code and strings.

### 13.9.2 Glulx Story File Layout

[Glulx] The story file has a different organization:

```
┌──────────────────────────────────────┐  Byte 0
│          Header (36 bytes)           │
├──────────────────────────────────────┤
│  Compiled routine bytecode           │
│  Compressed string data              │  ← ROM (read-only)
├──────────────────────────────────────┤  RAMSTART
│  Global variables (N × 4 bytes)      │
│  Dynamic arrays                      │
│  String-reference table              │
│  Object table                        │
│  Property values                     │
│  Individual property table           │
│  Class number table                  │
│  Identifier names (optional)         │
│  Static arrays                       │  ← RAM (read-write)
├──────────────────────────────────────┤  EXTSTART
│  (extensible memory)                 │  ← Available for runtime use
└──────────────────────────────────────┘
```

In Glulx, all addresses are full 32-bit byte offsets — there is no
packing. The ROM/RAM boundary is recorded in the header, allowing the
interpreter to enforce read-only protection on code and strings. The
`EXTSTART` value marks the end of the initial data; memory beyond this
point can be allocated at runtime.

### 13.9.3 String Compression

[Z-machine] Strings are encoded using the Z-character encoding scheme:
characters are packed three at a time into 2-byte words using a 5-bit
alphabet. Common substrings can be replaced with abbreviation codes for
further compression (enabled with the `-e` switch).

[Glulx] Strings can be stored uncompressed (as Latin-1 or Unicode
sequences) or compressed using Huffman encoding. The compiler builds a
Huffman tree during finalization and stores it in the ROM area. The
`compress_game_text()` function in `text.c` handles this process.

### 13.9.4 Header Fields

Both platforms store critical metadata in the story file header.

[Z-machine] The 64-byte header includes (among others):

| Offset | Size | Field |
| ------ | ---- | ----- |
| 0 | 1 | Z-machine version number (3–8) |
| 4–5 | 2 | Base of high memory (code area) |
| 6–7 | 2 | Initial program counter |
| 8–9 | 2 | Dictionary table address |
| 10–11 | 2 | Object table address |
| 12–13 | 2 | Global variables table address |
| 14–15 | 2 | Base of static memory |
| 24–29 | 6 | Serial number (compilation date as YYMMDD) |
| 26–27 | 2 | File length (divided by scale factor) |
| 28–29 | 2 | Checksum |

[Glulx] The 36-byte header includes version numbers, the RAMSTART
offset, the EXTSTART offset, the initial stack pointer, the entry point
address, and the string-decoding table address.

---

## 13.10 Dead Code Elimination

Starting with compiler version 6.40, Inform supports optional **dead
code elimination**: the removal of routines that are never called. This
feature is controlled by the `$OMIT_UNUSED_ROUTINES` setting.

### 13.10.1 How It Works

When `$OMIT_UNUSED_ROUTINES` is enabled (set to 2 to activate), the
compiler performs the following steps after the main pass:

1. **Record routine boundaries.** During code generation, the assembler
   calls `df_note_function_start()` and `df_note_function_end()` to
   record the start and end address of every compiled routine.

2. **Build a call graph.** The compiler tracks which routines call or
   reference which other routines, building a directed graph of
   dependencies.

3. **Trace from entry points.** Starting from known entry points — the
   `Main` routine, action routines, veneer routines, property values
   that hold routine addresses, and other roots — the compiler performs
   a reachability analysis.

4. **Mark unreachable routines.** Any routine not reachable from an
   entry point is marked as dead. The function `locate_dead_functions()`
   orchestrates this process.

5. **Strip dead code.** During story file construction, dead routines
   are omitted from the code area. The backpatcher uses
   `df_stripped_address_for_address()` to adjust all remaining routine
   addresses to account for the removed code.

### 13.10.2 Interaction with Backpatching

Dead code elimination interacts carefully with the backpatching system.
When `OMIT_UNUSED_ROUTINES` is active, the backpatch functions
(`backpatch_value_z` and `backpatch_value_g`) call
`df_stripped_address_for_address()` to translate pre-stripping routine
addresses to their post-stripping equivalents. This ensures that all
routine references in the final story file point to the correct
(compacted) addresses.

### 13.10.3 Limitations

- Dead code elimination is conservative: if the compiler cannot prove
  a routine is unreachable, it is kept. Routines stored in variables,
  passed as arguments, or referenced through computed expressions may
  not be eliminable.
- The feature adds compilation time due to the call graph analysis.
- Similarly, `locate_dead_grammar_lines()` removes grammar lines whose
  action routines are known to be unused, keeping the grammar table
  compact.

---

## 13.11 Putting It All Together

To summarise the complete lifecycle of a compilation:

```
Source Code (.inf)
       │
       ▼
 ┌─────────────────────┐
 │  1. Initialization   │  Parse options, allocate memory
 └──────────┬──────────┘
            ▼
 ┌─────────────────────┐
 │  2. Main Pass        │  Parse source, generate bytecode,
 │                      │  build symbols, objects, dictionary,
 │                      │  grammar; record backpatch markers
 └──────────┬──────────┘
            ▼
 ┌─────────────────────┐
 │  3. Veneer           │  Compile referenced veneer routines
 └──────────┬──────────┘
            ▼
 ┌─────────────────────┐
 │  4. Post-processing  │  Sort dictionary, sort actions,
 │                      │  eliminate dead code/grammar
 └──────────┬──────────┘
            ▼
 ┌─────────────────────┐
 │  5. Storyfile        │  Lay out memory regions, apply
 │     Construction     │  backpatches, write header,
 │                      │  compute checksum
 └──────────┬──────────┘
            ▼
      Story File (.z5 / .ulx)
```

The single-pass design with backpatching gives Inform 6 its
characteristic speed — even large games compile in under a second on
modern hardware — while the veneer system and flexible table
construction allow the compiler to support a rich object-oriented
language without requiring a complex multi-pass architecture.
