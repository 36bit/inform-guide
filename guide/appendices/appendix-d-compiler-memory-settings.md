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

# Appendix D: Compiler Memory Settings Reference

This appendix provides a complete reference for all compiler settings
controllable through the `$` command syntax in Inform 6.44. These settings
govern dictionary format, object layout, code generation strategy, diagnostic
output, and other compiler behavior. Each entry is verified against the
compiler source code (`Inform6-6.44/options.c`).

For an overview of how to invoke the compiler and pass these settings, see
Chapter 11 (Invoking the Compiler) and Chapter 12 (Compiler Switches and
Settings).

---

## §D.1 Setting Syntax

Compiler memory settings use the `$` prefix and are specified in one of three
places, listed here in order of increasing precedence:

1. **Compiled-in defaults** — built into the compiler itself (precedence 0).
2. **Header comments** — an `!%` line at the very start of the source file
   (precedence 1).
3. **Command line** — passed directly to the compiler invocation
   (precedence 2).

A setting from a higher-precedence source overrides the same setting from a
lower-precedence source.

### Setting a Value

```
$SETTING_NAME=value
```

On the command line, this is passed as a single argument:

```
inform6 $MAX_ABBREVS=80 game.inf
```

In a source file header comment (must appear before any other source text):

```inform6
!% $MAX_ABBREVS=80
```

Setting names are case-insensitive (they are uppercased internally before
lookup).

### Querying a Setting

To display the current value and description of a setting, use `$?`:

```
inform6 $?MAX_ABBREVS
```

### Listing All Settings

To display all active settings and their current values for the selected
target platform, use `$LIST`:

```
inform6 $LIST
inform6 -G $LIST
```

The first command shows Z-machine settings; the second shows Glulx settings.

### Defining Symbols

The `$#` prefix defines a compile-time symbol (equivalent to a `Constant`
directive):

```
inform6 $#DEBUG=1 game.inf
```

### Trace Options

The `$!` prefix controls diagnostic trace output. See §D.5 for the full list.

```
inform6 $!DICT game.inf
inform6 $!ASM=2 game.inf
```

---

## §D.2 Active Settings — Complete Table

The following table lists all 27 active compiler settings as of Inform 6.44.
The **Platform** column indicates whether a setting applies to Z-code only
[Z-machine], Glulx only [Glulx], or both [All]. The **Limit** column
describes value constraints enforced by the compiler.

| Setting | Z Default | Glulx Default | Platform | Limit | Description |
|---------|-----------|---------------|----------|-------|-------------|
| `MAX_ABBREVS` | 64 | — | [Z-machine] | 0–96 | Maximum number of declared abbreviations |
| `NUM_ATTR_BYTES` | 6 | 7 | [Glulx] | Multiple of 4, plus 3 | Space (in bytes) used to store attribute flags; each byte stores 8 attributes |
| `DICT_WORD_SIZE` | 6 | 9 | [Glulx] | Any ≥ 0 | Number of characters in a dictionary word |
| `DICT_CHAR_SIZE` | 1 | 1 | [Glulx] | 1 or 4 | Byte size of one character in the dictionary (4 enables full Unicode input) |
| `GRAMMAR_VERSION` | 1 | 2 | [All] | Validated later | Grammar table format: 1 = Infocom format, 2 = Inform standard |
| `GRAMMAR_META_FLAG` | 0 | 0 | [All] | 0–1 | If 1, meta actions are indicated by value (≤ `#largest_meta_action`) rather than dict word flags |
| `MAX_DYNAMIC_STRINGS` | 32 | 100 | [All] | 0–96 in Z-code; unlimited in Glulx | Maximum number of string substitution variables (`@00` or `@(0)`) |
| `HASH_TAB_SIZE` | 512 | 512 | [All] | Any ≥ 0 | Size of hash tables used for the heaviest symbol banks |
| `ZCODE_HEADER_EXT_WORDS` | 3 | — | [Z-machine] | Any ≥ 0 | Number of words in the Z-code header extension table (Z-Spec 1.0); the `-W` switch also sets this |
| `ZCODE_HEADER_FLAGS_3` | 0 | — | [Z-machine] | Any ≥ 0 | Value to store in the Flags 3 word of the header extension table (Z-Spec 1.1) |
| `ZCODE_FILE_END_PADDING` | 1 | — | [Z-machine] | Any ≥ 0 | If non-zero, pads game file length to a multiple of 512 bytes |
| `ZCODE_LESS_DICT_DATA` | 0 | — | [Z-machine] | Any ≥ 0 | If non-zero, provides each dict word with 2 data bytes rather than 3 |
| `ZCODE_MAX_INLINE_STRING` | 32 | — | [Z-machine] | Any ≥ 0 | Maximum length of string literals that can be inlined in assembly opcodes |
| `ZCODE_COMPACT_GLOBALS` | 0 | — | [Z-machine] | Any ≥ 0 | If non-zero, reuses space from unused global variables in the globals segment |
| `INDIV_PROP_START` | 64 | 256 | [All] | Any ≥ 0 | Properties 1 to N−1 are common properties; N and above are individual properties |
| `MEMORY_MAP_EXTENSION` | — | 0 | [Glulx] | Rounded up to multiple of 256 | Number of zero-filled bytes to map into memory after the game file |
| `GLULX_OBJECT_EXT_BYTES` | — | 0 | [Glulx] | Any ≥ 0 | Additional space (in bytes) added to each object record, initialized to zero |
| `MAX_STACK_SIZE` | — | 4096 | [Glulx] | Rounded up to multiple of 256 | Maximum size (in bytes) of the interpreter stack during gameplay |
| `TRANSCRIPT_FORMAT` | 0 | 0 | [All] | 0–1 | If 1, adjusts `gametext.txt` transcript: each line is prefixed by its context |
| `WARN_UNUSED_ROUTINES` | 0 | 0 | [All] | 0–2 | 0 = no warnings; 1 = warn for game code only; 2 = warn for all code including library |
| `OMIT_UNUSED_ROUTINES` | 0 | 0 | [All] | 0–1 | If 1, avoids compiling unused routines into the game file |
| `STRIP_UNREACHABLE_LABELS` | 1 | 1 | [All] | 0–1 | If 1, skips labels in unreachable statements (more optimized code); if 0, compiles all labels |
| `OMIT_SYMBOL_TABLE` | 0 | 0 | [All] | 0–1 | If 1, skips compiling debug symbol names into the game file |
| `DICT_IMPLICIT_SINGULAR` | 0 | 0 | [All] | 0–1 | If 1, dict words in noun context get the `//s` flag when the `//p` flag is not set |
| `DICT_TRUNCATE_FLAG` | 0 | 0 | [All] | 0–1 | If 1, sets bit 6 of a dict word when truncated; if 0, bit 6 is set for verbs (legacy) |
| `LONG_DICT_FLAG_BUG` | 1 | 1 | [All] | 0–1 | If 0, fixes the bug that ignores `//p` on long dictionary words; if 1, retains buggy behavior |
| `SERIAL` | 0 | 0 | [All] | 0–999999 | Six-digit serial number written into the output file header (only applied when explicitly set) |

---

## §D.3 Detailed Setting Descriptions

### §D.3.1 Dictionary and Vocabulary Settings

#### `MAX_ABBREVS`

**Platform:** Z-machine only
**Default:** 64
**Limit:** 0–96

The maximum number of declared abbreviations. Abbreviations are short strings
that the compiler can substitute with single-byte tokens in Z-encoded text,
saving space. The Z-machine specification limits this to 96. This setting is
not meaningful in Glulx, where abbreviations are unlimited.

```inform6
!% $MAX_ABBREVS=96
Abbreviate "the ";
Abbreviate "you ";
```

#### `DICT_WORD_SIZE`

**Platform:** Glulx only (fixed at 6 in Z-code, 4 in Z-machine v3)
**Default:** 9

The number of characters stored per dictionary word. In Z-code, this is fixed
by the virtual machine specification (6 characters in v4+, 4 in v3). In Glulx,
this can be set to any value. Increasing this value allows the parser to
distinguish longer words, at the cost of a larger dictionary table.

```inform6
!% $DICT_WORD_SIZE=12
```

#### `DICT_CHAR_SIZE`

**Platform:** Glulx only (fixed at 1 in Z-code)
**Default:** 1
**Limit:** 1 or 4

The byte size of a single character in dictionary entries. The default of 1
uses one byte per character (Latin-1 range). Setting this to 4 enables full
Unicode dictionary words, where each character occupies four bytes. Values of
0, 2, or 3 are normalized to 1 by the compiler.

```inform6
!% $DICT_CHAR_SIZE=4
```

#### `DICT_IMPLICIT_SINGULAR`

**Platform:** All
**Default:** 0
**Limit:** 0–1

When set to 1, dictionary words used in noun context automatically receive the
singular flag (`//s`) unless the plural flag (`//p`) is already set. This is
useful for languages or library configurations that rely on singular/plural
distinctions in dictionary words.

#### `DICT_TRUNCATE_FLAG`

**Platform:** All
**Default:** 0
**Limit:** 0–1

Controls the meaning of bit 6 in dictionary word flags. When set to 1, bit 6
indicates that the word was truncated (it extends beyond `DICT_WORD_SIZE`
characters in the source). When set to 0 (the default, legacy behavior), bit 6
indicates that the word is used as a verb.

#### `LONG_DICT_FLAG_BUG`

**Platform:** All
**Default:** 1
**Limit:** 0–1

Historical versions of the compiler contained a bug that caused the `//p`
(plural) flag to be ignored on dictionary words longer than `DICT_WORD_SIZE`.
Setting this to 0 fixes the bug. The default of 1 retains the buggy behavior
for backward compatibility with existing games that may depend on it.

#### `ZCODE_LESS_DICT_DATA`

**Platform:** Z-machine only
**Default:** 0

When set to a non-zero value, each dictionary entry carries only 2 bytes of
associated data instead of the standard 3. This saves memory in the dictionary
table but removes one byte of per-word data available to the game.

### §D.3.2 Grammar Settings

#### `GRAMMAR_VERSION`

**Platform:** All
**Z-machine Default:** 1
**Glulx Default:** 2

Selects the table format used for verb grammar. Version 1 uses a format based
on Infocom's original grammar tables. Version 2 is the Inform standard format,
which is more flexible and supports features like the `meta` keyword in grammar
definitions. The default is 1 for Z-code (for Infocom compatibility) and 2 for
Glulx.

#### `GRAMMAR_META_FLAG`

**Platform:** All
**Default:** 0
**Limit:** 0–1

When set to 1, meta actions (actions that do not consume a game turn) are
indicated by having action values less than or equal to
`#largest_meta_action`, rather than relying on dictionary word flags. This
allows individual actions to be precisely marked as meta.

### §D.3.3 Object and Property Settings

#### `NUM_ATTR_BYTES`

**Platform:** Glulx only (fixed at 6 in Z-code, 4 in v3)
**Default:** 7

The number of bytes used to store attribute flags in each object record. Each
byte provides 8 attribute slots, so the default of 7 provides 56 attributes
(though only attributes 0–54 are typically usable because the value must be a
multiple of 4, plus 3). In Z-code this is fixed at 6 (48 attributes; only 32
used in v3).

```inform6
!% $NUM_ATTR_BYTES=11
```

This would provide 88 attribute slots. The value must satisfy the constraint:
`(NUM_ATTR_BYTES - 3)` must be a multiple of 4.

#### `INDIV_PROP_START`

**Platform:** All
**Z-machine Default:** 64
**Glulx Default:** 256

Defines the boundary between common properties and individual properties.
Properties numbered 1 through `INDIV_PROP_START - 1` are common properties
(shared across all objects and accessible by property number). Properties
numbered `INDIV_PROP_START` and above are individual properties (per-object,
accessed by identifier). The class-system individual properties (`create`,
`recreate`, `destroy`, `remaining`, `copy`, `call`, `print`,
`print_to_array`) are assigned starting at `INDIV_PROP_START`.

#### `GLULX_OBJECT_EXT_BYTES`

**Platform:** Glulx only
**Default:** 0

The number of additional bytes to add to each object record beyond the
standard fields. These extra bytes are initialized to zero and are available
for the game to use as it sees fit. This is only meaningful in Glulx; the
Z-machine object format is fixed by the specification.

### §D.3.4 String and Dynamic Text Settings

#### `MAX_DYNAMIC_STRINGS`

**Platform:** All
**Z-machine Default:** 32
**Glulx Default:** 100
**Limit:** 0–96 in Z-code; unlimited in Glulx

The maximum number of dynamic string substitution variables. These are
referenced in strings with the `@00` through `@31` syntax (or `@(N)` for
higher numbers). The `string` statement assigns text to these variables at
runtime. In Z-code the maximum is 96.

```inform6
!% $MAX_DYNAMIC_STRINGS=64
```

#### `ZCODE_MAX_INLINE_STRING`

**Platform:** Z-machine only
**Default:** 32

The maximum length (in source characters) of a string literal that can be
inlined directly into assembly opcodes. Strings longer than this are stored
separately and referenced by address. Increasing this value may improve
performance for short strings at the cost of larger code size.

### §D.3.5 Z-Code Header Settings

#### `ZCODE_HEADER_EXT_WORDS`

**Platform:** Z-machine only
**Default:** 3

The number of 16-bit words in the Z-code header extension table, as defined
in Z-Spec 1.0. The default of 3 accommodates the standard extension fields
including the Unicode translation table pointer. Can be set to 0 if no
Unicode translation table is needed. The `-W` command-line switch also sets
this value.

#### `ZCODE_HEADER_FLAGS_3`

**Platform:** Z-machine only
**Default:** 0

The value to place in the Flags 3 word of the header extension table, as
defined in Z-Spec 1.1. This is a bitmask of optional feature flags.

#### `ZCODE_FILE_END_PADDING`

**Platform:** Z-machine only
**Default:** 1

When set to a non-zero value, the compiler pads the output story file to a
length that is a multiple of 512 bytes. This is traditional behavior expected
by some interpreters. Set to 0 to produce a file with no padding.

#### `ZCODE_COMPACT_GLOBALS`

**Platform:** Z-machine only
**Default:** 0

When set to a non-zero value, the compiler reuses space from any unused global
variables in the 240-entry global variable segment. Global variables that are
never referenced are compacted out, and arrays begin immediately after the
last used global. This can save a small amount of space in the output file.

### §D.3.6 Glulx Runtime Settings

#### `MEMORY_MAP_EXTENSION`

**Platform:** Glulx only
**Default:** 0
**Limit:** Rounded up to a multiple of 256

The number of zero-filled bytes to map into memory after the end of the game
file. This reserved region can be used by the game at runtime for dynamic
memory allocation. The value is always rounded up to the nearest multiple of
256 bytes.

```inform6
!% $MEMORY_MAP_EXTENSION=4096
```

#### `MAX_STACK_SIZE`

**Platform:** Glulx only
**Default:** 4096
**Limit:** Rounded up to a multiple of 256

The maximum size (in bytes) of the interpreter stack during gameplay. The
default of 4096 provides roughly enough space for 90 nested function calls
with 8 local variables each, comparable to the Z-Spec's recommendation for
Z-machine stack size. Games with deep recursion or complex data structures
on the stack may need to increase this value. Inform 7-generated code
typically sets this to 65536.

```inform6
!% $MAX_STACK_SIZE=65536
```

### §D.3.7 Compilation Behavior Settings

#### `HASH_TAB_SIZE`

**Platform:** All
**Default:** 512

The size of the hash tables used for the compiler's heaviest symbol banks
(identifiers, dictionary words, etc.). Increasing this value can improve
compilation speed for very large source files by reducing hash collisions, at
the cost of increased compiler memory usage.

#### `SERIAL`

**Platform:** All
**Default:** 0 (not applied unless explicitly set)
**Limit:** 0–999999

When explicitly set, this six-digit number is used as the serial number
written into the header of the output file. If not set, the compiler uses the
current date in `YYMMDD` format. The `Serial` directive in source code is an
alternative way to set this.

```inform6
!% $SERIAL=240101
```

### §D.3.8 Code Optimization Settings

#### `WARN_UNUSED_ROUTINES`

**Platform:** All
**Default:** 0
**Limit:** 0–2

Controls warnings about unused routines:

- **0** — No warnings (default).
- **1** — Warn about unused routines in game code only (not in system/library
  files).
- **2** — Warn about all unused routines, including those in the standard
  library and system files.

This analysis includes routines that are only called from other unused
routines (transitively unreachable code).

#### `OMIT_UNUSED_ROUTINES`

**Platform:** All
**Default:** 0
**Limit:** 0–1

When set to 1, the compiler omits unused routines from the output file
entirely. This can significantly reduce story file size for projects that
include large libraries where many routines go unused. The compiler performs
a reachability analysis to determine which routines are actually called.

```inform6
!% $OMIT_UNUSED_ROUTINES=1
```

#### `STRIP_UNREACHABLE_LABELS`

**Platform:** All
**Default:** 1
**Limit:** 0–1

When set to 1 (the default), labels that appear in unreachable code (code
after an unconditional `return`, `jump`, `quit`, etc.) are stripped from the
output. This produces more optimized code. Jumping to a stripped label is a
compile-time error. Setting this to 0 causes all labels to be compiled
regardless of reachability, which may be needed for unusual control flow
patterns.

#### `OMIT_SYMBOL_TABLE`

**Platform:** All
**Default:** 0
**Limit:** 0–1

When set to 1, the compiler does not write debug symbol names into the output
game file. This reduces the file size but means that runtime error messages
and debugging tools cannot display meaningful symbol names. Useful for
release builds where file size is a priority.

```inform6
!% $OMIT_SYMBOL_TABLE=1
```

#### `TRANSCRIPT_FORMAT`

**Platform:** All
**Default:** 0
**Limit:** 0–1

Controls the format of the `gametext.txt` compilation transcript:

- **0** — Classic format (default).
- **1** — Machine-processing format, where each line is prefixed by a context
  indicator showing what kind of text it represents.

---

## §D.4 Obsolete Settings

The following settings have been withdrawn and are no longer functional. The
compiler will print a warning if they are used.

### §D.4.1 Withdrawn Inform 5 Settings

These Inform 5 memory settings have been withdrawn entirely:

| Setting | Original Purpose |
|---------|-----------------|
| `BUFFER_LENGTH` | Input buffer size |
| `MAX_BANK_SIZE` | Memory bank size |
| `BANK_CHUNK_SIZE` | Memory bank allocation chunk |
| `MAX_OLDEPTH` | Object loop nesting depth |
| `MAX_ROUTINES` | Maximum number of routines |
| `MAX_GCONSTANTS` | Maximum global constants |
| `MAX_FORWARD_REFS` | Maximum forward references |
| `STACK_SIZE` | Evaluation stack size |
| `STACK_LONG_SLOTS` | Long stack slot count |
| `STACK_SHORT_LENGTH` | Short stack length |

### §D.4.2 Withdrawn Inform 6 Settings

These Inform 6 memory settings are no longer needed. The compiler now manages
these resources dynamically:

| Setting | Original Purpose |
|---------|-----------------|
| `MAX_QTEXT_SIZE` | Quoted text buffer size |
| `MAX_SYMBOLS` | Maximum number of symbols |
| `SYMBOLS_CHUNK_SIZE` | Symbol table allocation chunk |
| `MAX_OBJECTS` | Maximum number of objects |
| `MAX_ACTIONS` | Maximum number of actions |
| `MAX_ADJECTIVES` | Maximum number of adjectives |
| `MAX_DICT_ENTRIES` | Maximum dictionary entries |
| `MAX_STATIC_DATA` | Maximum static data size |
| `MAX_PROP_TABLE_SIZE` | Property table size |
| `MAX_ARRAYS` | Maximum number of arrays |
| `MAX_EXPRESSION_NODES` | Expression tree node limit |
| `MAX_VERBS` | Maximum number of verbs |
| `MAX_VERBSPACE` | Verb grammar storage space |
| `MAX_LABELS` | Maximum number of labels |
| `MAX_LINESPACE` | Line storage space |
| `MAX_NUM_STATIC_STRINGS` | Maximum static string count |
| `MAX_STATIC_STRINGS` | Static string storage size |
| `MAX_ZCODE_SIZE` | Maximum Z-code output size |
| `MAX_LINK_DATA_SIZE` | Link data buffer size |
| `MAX_LOW_STRINGS` | Maximum low strings |
| `MAX_TRANSCRIPT_SIZE` | Transcript buffer size |
| `MAX_CLASSES` | Maximum number of classes |
| `MAX_INCLUSION_DEPTH` | Maximum include nesting depth |
| `MAX_SOURCE_FILES` | Maximum source files |
| `MAX_INDIV_PROP_TABLE_SIZE` | Individual property table size |
| `MAX_OBJ_PROP_TABLE_SIZE` | Object property table size |
| `MAX_OBJ_PROP_COUNT` | Per-object property count limit |
| `MAX_LOCAL_VARIABLES` | Maximum local variables per routine |
| `MAX_GLOBAL_VARIABLES` | Maximum global variables |
| `ALLOC_CHUNK_SIZE` | General allocation chunk size |
| `MAX_UNICODE_CHARS` | Maximum Unicode character entries |

The size commands `$SMALL`, `$LARGE`, and `$HUGE` have also been withdrawn.

---

## §D.5 Trace Options (`$!`)

Trace options control diagnostic output during compilation. They are set with
the `$!` prefix and accept an optional numeric level (defaulting to 1). Higher
levels produce more verbose output. These options are primarily useful for
compiler development and debugging.

```
inform6 $!OPTION game.inf
inform6 $!OPTION=2 game.inf
```

Use `$!` with no argument to display the full list of available trace options.

| Option | Level 1 | Level 2 | Level 3+ | Equivalent Switch |
|--------|---------|---------|----------|-------------------|
| `ACTIONS` | Show all actions | Also list actions as they are defined | — | — |
| `ASM` | Trace assembly | Also show hex dumps | Level 3: branch optimization info; Level 4: more verbose branch info | `-a` |
| `BPATCH` | Show backpatch results | Also show markers added | — | — |
| `DICT` | Display the dictionary table | Also show byte encoding of entries | — | — |
| `EXPR` | Show expression trees | More verbose | Level 3: even more verbose | — |
| `FILES` | Show files opened | — | — | — |
| `FINDABBREVS` | Show abbreviation optimization decisions | Also show three-letter-block decisions | — | Only meaningful with `-u` |
| `FREQ` | Show abbreviation efficiency | — | — | `-f` (only meaningful with `-e`) |
| `MAP` | Print memory map of the virtual machine | Also show percentage of VM per segment | Level 3: also show byte counts per segment | `-z` |
| `MEM` | Show internal memory allocations | — | — | — |
| `OBJECTS` | Display the object table | — | — | — |
| `PROPS` | Show attributes and properties defined | — | — | — |
| `RUNTIME` | Show game function calls at runtime | Also show library calls (Z-machine only) | Level 3: also show veneer calls (Z-machine only) | `-g` |
| `STATS` | Give compilation statistics | — | — | `-s` |
| `SYMBOLS` | Display the symbol table | Also show compiler-defined symbols | — | — |
| `SYMDEF` | Show when symbols are noticed and defined | — | — | — |
| `TOKENS` | Show token lexing | Also show token types | Level 3: also show lexical context | — |
| `VERBS` | Display the verb grammar table | — | — | — |

### Accepted Synonyms

Several trace options accept alternative spellings:

| Canonical Name | Accepted Synonyms |
|---------------|-------------------|
| `ASM` | `ASSEMBLY` |
| `ACTIONS` | `ACTION` |
| `BPATCH` | `BACKPATCH` |
| `DICT` | `DICTIONARY` |
| `EXPR` | `EXPRESSION`, `EXPRESSIONS` |
| `FILES` | `FILE` |
| `FINDABBREVS` | `FINDABBREV` |
| `FREQ` | `FREQUENCY`, `FREQUENCIES` |
| `MEM` | `MEMORY` |
| `OBJECTS` | `OBJECT`, `OBJS`, `OBJ` |
| `PROPS` | `PROP`, `PROPERTY`, `PROPERTIES` |
| `STATS` | `STATISTICS`, `STAT` |
| `SYMBOLS` | `SYMBOL` |
| `SYMDEF` | `SYMBOLDEF` |
| `TOKENS` | `TOKEN` |
| `VERBS` | `VERB` |

---

## §D.6 Predefined Symbol Definition (`$#`)

The `$#` prefix defines a compile-time symbol, equivalent to writing a
`Constant` directive in source code. This is useful for setting conditional
compilation flags from the command line:

```
inform6 $#DEBUG=1 $#MY_FLAG=42 game.inf
```

This is equivalent to having the following at the top of `game.inf`:

```inform6
Constant DEBUG = 1;
Constant MY_FLAG = 42;
```

The symbol name must be a valid Inform identifier (alphanumeric characters and
underscores, not starting with a digit). If no `=value` is provided, the
symbol is defined with the value 0.

---

## §D.7 Value Limits and Normalization

The compiler enforces constraints on setting values. Understanding these rules
helps avoid unexpected behavior.

### Numeric Parsing

All numeric setting values are parsed as decimal integers. Values with more
than 9 digits are clamped to ±999999999 with an overflow warning. Whitespace
before and after the number is permitted.

### Limit Types

The compiler uses several limit categories:

| Limit Type | Behavior |
|-----------|----------|
| Any non-negative | Value must be ≥ 0; no upper bound |
| 0 to N | Value is clamped to the range [0, N] |
| 0 to N (Z-code only) | In Z-code, clamped to [0, N]; in Glulx, unlimited |
| Multiple of 256 | Value is rounded up to the next multiple of 256 |
| String | Value is a string, not a number |

### Platform-Specific Defaults

Many settings have different defaults for Z-machine and Glulx targets. When
a setting is explicitly set by the user, the compiler stores the value for
both platforms, except for the `MAX_DYNAMIC_STRINGS` setting which applies the
Z-code limit only to the Z-code value while leaving the Glulx value unlimited.

### Precedence

If a setting is specified at multiple precedence levels, only the
highest-precedence value takes effect:

1. **Default** (precedence 0) — The compiled-in value.
2. **Header comment** (precedence 1) — A `!% $SETTING=value` line in source.
3. **Command line** (precedence 2) — A `$SETTING=value` argument on the
   command line.

A command-line setting always overrides a header comment setting for the same
option.

---

## §D.8 Quick Reference by Category

### Dictionary and Vocabulary

| Setting | Description |
|---------|-------------|
| `DICT_CHAR_SIZE` | Byte size of dictionary characters (1 or 4) |
| `DICT_IMPLICIT_SINGULAR` | Auto-apply singular flag to noun words |
| `DICT_TRUNCATE_FLAG` | Use bit 6 for truncation instead of verb marking |
| `DICT_WORD_SIZE` | Characters per dictionary word |
| `LONG_DICT_FLAG_BUG` | Retain/fix plural flag bug on long words |
| `MAX_ABBREVS` | Maximum abbreviation count |
| `ZCODE_LESS_DICT_DATA` | Reduce per-word data bytes from 3 to 2 |

### Grammar

| Setting | Description |
|---------|-------------|
| `GRAMMAR_META_FLAG` | Use action values for meta indication |
| `GRAMMAR_VERSION` | Grammar table format (1 or 2) |

### Objects and Properties

| Setting | Description |
|---------|-------------|
| `GLULX_OBJECT_EXT_BYTES` | Extra bytes per object record |
| `INDIV_PROP_START` | Common/individual property boundary |
| `NUM_ATTR_BYTES` | Attribute flag storage size |

### Strings and Text

| Setting | Description |
|---------|-------------|
| `MAX_DYNAMIC_STRINGS` | Maximum dynamic string variables |
| `ZCODE_MAX_INLINE_STRING` | Maximum inlineable string length |

### Z-Code Header

| Setting | Description |
|---------|-------------|
| `ZCODE_COMPACT_GLOBALS` | Reuse unused global variable space |
| `ZCODE_FILE_END_PADDING` | Pad output to 512-byte boundary |
| `ZCODE_HEADER_EXT_WORDS` | Header extension table size |
| `ZCODE_HEADER_FLAGS_3` | Flags 3 header extension value |

### Glulx Runtime

| Setting | Description |
|---------|-------------|
| `MAX_STACK_SIZE` | Interpreter stack size |
| `MEMORY_MAP_EXTENSION` | Post-file memory mapping |

### Code Generation

| Setting | Description |
|---------|-------------|
| `OMIT_SYMBOL_TABLE` | Strip debug symbol names |
| `OMIT_UNUSED_ROUTINES` | Remove unreachable routines |
| `STRIP_UNREACHABLE_LABELS` | Remove labels in dead code |
| `WARN_UNUSED_ROUTINES` | Warn about unused routines |

### Compiler Behavior

| Setting | Description |
|---------|-------------|
| `HASH_TAB_SIZE` | Symbol hash table size |
| `SERIAL` | Output file serial number |
| `TRANSCRIPT_FORMAT` | Compilation transcript format |

---

*Source: All settings are defined in `Inform6-6.44/options.c`, lines 106–382.
Variable declarations are in `Inform6-6.44/memory.c`, lines 256–281 and
`Inform6-6.44/inform.c`, line 94. Trace options are handled in
`Inform6-6.44/memory.c`, lines 320–444. Setting precedence constants are
defined in `Inform6-6.44/header.h`, lines 598–600.*
