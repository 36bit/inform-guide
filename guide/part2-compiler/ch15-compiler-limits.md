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

# Chapter 15: Compiler Limits and Memory Settings

This chapter describes the limits that constrain a program: both the hard
constraints imposed by the target virtual machine and the configurable
settings that control the compiler's own behavior. It collects the
practical information a programmer needs when a project grows beyond
default limits, when choosing between Z-machine and Glulx, or when
fine-tuning the compiler for a production release.

---

## 15.1 Overview

Limits on a program come from two distinct sources:

1. **Virtual machine constraints.** The Z-machine and Glulx each define
   a fixed architecture — address widths, object table formats, and
   instruction encodings — that impose absolute upper bounds. These
   limits cannot be changed by compiler settings; they are inherent in
   the target platform.

2. **Compiler configuration.** The compiler itself maintains internal
   tables and parameters that govern dictionary layout, attribute
   storage, stack size, and similar concerns. These are controlled with
   `$SETTING=value` options (see §12.4 for the syntax).

Since compiler version 6.36, the second category has been dramatically
simplified. Most internal tables that formerly required fixed-size
allocations now grow dynamically as needed. The settings that remain are
those with semantic significance — they affect the format of the
compiled output, not just how much workspace the compiler reserves
during compilation.

The practical consequence is that modern projects rarely need to
adjust compiler settings at all. When limits are encountered, they are
almost always VM-imposed constraints that require either choosing a
different Z-machine version or switching to Glulx.

---

## 15.2 Z-Machine Story File Size Limits

The Z-machine exists in several versions, each with its own size
constraints. These are architectural limits defined by the Z-Machine
Standards Document and cannot be overridden by compiler settings.

### 15.2.1 Size and Object Limits by Version

| Version | Max file size | Max objects | Max attributes | Max properties | Max globals |
| ------- | ------------- | ----------- | -------------- | -------------- | ----------- |
| **V3**  | 128 KB        | 255         | 32             | 31             | 240         |
| **V4**  | 256 KB        | 65,535      | 48             | 63             | 240         |
| **V5**  | 256 KB        | 65,535      | 48             | 63             | 240         |
| **V6**  | 512 KB        | 65,535      | 48             | 63             | 240         |
| **V7**  | 512 KB        | 65,535      | 48             | 63             | 240         |
| **V8**  | 512 KB        | 65,535      | 48             | 63             | 240         |

The "max properties" column reflects the Z-Machine architectural limit
(property numbers 1–31 in V3, 1–63 in V4+). The compiler reserves a
small number of property slots for internal use (`name`, class
inheritance, individual property table), so the practical limit on
user-declarable common properties is **29** in V3 and **61** in V4+.
Exceeding this limit produces the error `All 29 properties already
declared` (or `All 61 …`). Similarly, the "max globals" column shows
the architectural total of 240; the compiler reserves 7 of those slots,
so the user limit is **233** (see §15.2.2).

**V3** is the original Infocom format. Its 128 KB file size limit, 255
maximum objects, and 32 attributes make it unsuitable for large modern
games. **V5** is the default target and the best choice for most
Z-machine projects. **V8** doubles the packed address scale factor,
allowing larger routine and string segments within a 512 KB file; it is
the recommended choice for large Z-code games that exceed V5's 256 KB
limit.

### 15.2.2 Other Z-Machine Limits

Several additional limits apply across all Z-machine versions:

- **Local variables:** A maximum of 15 local variables per routine.
  This is a hard limit of the Z-machine instruction encoding.

- **Dictionary word resolution:** In V3, dictionary words are resolved
  to 6 characters (stored in 4 bytes using Z-character encoding). In V4
  and later versions, words are resolved to 9 characters (stored in 6
  bytes). Words longer than the resolution limit are silently truncated
  by the parser.

- **Packed address scale factors:** Routine and string addresses are
  stored as packed values. The scale factor determines the granularity
  of addressing and therefore the maximum addressable space:

  | Version | Packed address scale | Effective routine/string limit |
  | ------- | -------------------- | ------------------------------ |
  | V3      | ×2                   | 128 KB                         |
  | V4–V5   | ×4                   | 256 KB                         |
  | V6–V7   | ×4 (with offsets)    | 512 KB                         |
  | V8      | ×8                   | 512 KB                         |

  V6 and V7 use separate routine and string offset values in the header
  to extend addressing beyond the basic scale factor.

- **Global variables:** The Z-machine architecture provides 240 global
  variable slots. The compiler reserves 7 of them for internal use
  (four temporaries, `self`, `sender`, and `sw__var`), leaving at most
  **233** user-declarable globals. See §3.1 for details.

- **Abbreviations and dynamic strings:** The Z-machine allocates a
  shared pool of exactly 96 slots for these two features combined.
  The defaults are `$MAX_ABBREVS=64` and `$MAX_DYNAMIC_STRINGS=32`
  (summing to 96). If only one value is changed, the compiler
  automatically adjusts the other to keep the sum at 96; if both are
  set to values that do not sum to 96, the compiler warns and resets
  both to their defaults.

- **Property values per property:** On the Z-machine, a single property
  may hold at most **32 values** (64 bytes). For V3 common properties the
  limit is tighter: **4 values** (8 bytes). These are hard limits enforced
  by the compiler with an error such as "Limit (of 32 values) exceeded for
  property". In practice they are rarely reached.

---

## 15.3 Glulx Limits

**[Glulx]** Glulx is a 32-bit virtual machine designed to overcome the
Z-machine's size constraints. Its limits are far more generous:

- **Address space:** 32-bit addressing provides a theoretical maximum
  of 4 GB for the story file. In practice, file sizes are limited by
  interpreter memory and the Glulx specification's use of signed
  offsets, but multi-megabyte games are routine.

- **Objects:** There is no hard limit on the number of objects. The
  compiler allocates object records dynamically.

- **Attributes:** Configurable via `$NUM_ATTR_BYTES`. The default of 7
  bytes provides 56 attributes. The value must follow the formula
  4*n* + 3 (so 7, 11, 15, 19, ...) up to a maximum of 39 bytes
  (312 attributes).

- **Properties:** There is no architectural limit on properties. The
  setting `$INDIV_PROP_START` (default 256) determines the boundary
  between common and individual properties; properties numbered below
  it are common, those at or above it are individual. A single property
  may hold at most **32,768 values**; this is a compiler-imposed limit
  that is effectively never reached in practice.

- **Dictionary word resolution:** Controlled by `$DICT_WORD_SIZE`
  (default 9). Unlike the Z-machine, this is freely adjustable.
  `$DICT_CHAR_SIZE` can be set to 4 for full Unicode dictionary entries.

- **Local variables:** `MAX_LOCAL_VARIABLES` is set to 119 (including the
  internal `sp` slot), giving a maximum of **118** usable local variables
  per routine (compared to 15 in the Z-machine).

- **Stack size:** Controlled by `$MAX_STACK_SIZE` (default 4096 bytes).
  This can be increased for deeply recursive programs.

- **Global variables:** No fixed limit (dynamically allocated by the
  compiler).

- **Memory extension:** `$MEMORY_MAP_EXTENSION` appends extra zero
  bytes to the story file for runtime use. `$GLULX_OBJECT_EXT_BYTES`
  adds extra bytes to each object record for custom data.

The combination of these generous limits makes Glulx the natural choice
for large projects, games requiring extensive world models, or
applications needing full Unicode support.

---

## 15.4 Configurable Compiler Settings

The following table lists all `$SETTING=value` options recognized by
the current compiler (version 6.44). Settings marked "fixed" in the
Z-code column are determined by the Z-machine specification and cannot
be changed when compiling for Z-code. Settings marked "N/A" do not
apply to that target.

For the syntax used to set these values, see §12.4.1. To display all
current values, run `inform $LIST`.

### 15.4.1 Dictionary and Text Settings

| Setting | Z-code default | Glulx default | Description |
| ------- | -------------- | ------------- | ----------- |
| `$DICT_WORD_SIZE` | 6 (fixed) | 9 | Characters stored per dictionary word. Longer words are truncated during parsing. |
| `$DICT_CHAR_SIZE` | 1 (fixed) | 1 | Bytes per character in dictionary words. Set to 4 in Glulx for Unicode dictionary entries. |
| `$MAX_ABBREVS` | 64 | N/A | Maximum number of `Abbreviate` directives. Z-code hard limit is 96. Not meaningful in Glulx (no abbreviation limit). |
| `$MAX_DYNAMIC_STRINGS` | 32 | 100 | Maximum number of string substitution variables (`@00`, `@(0)`, etc.). Z-code hard limit is 96. |

### 15.4.2 Grammar and Object Settings

| Setting | Z-code default | Glulx default | Description |
| ------- | -------------- | ------------- | ----------- |
| `$GRAMMAR_VERSION` | 1 | 2 | Grammar table format. Version 2 is the standard format used by modern libraries; version 1 is the older Infocom-style format. |
| `$INDIV_PROP_START` | 64 (fixed) | 256 | First individual property number. Properties below this are common; those at or above are individual. |
| `$NUM_ATTR_BYTES` | 6 (fixed) | 7 | Bytes used for attribute flags. Each byte provides 8 attributes (Z-code: 48; Glulx default: 56). |

### 15.4.3 Glulx-Specific Settings

**[Glulx]** These settings apply only when compiling for Glulx:

| Setting | Default | Description |
| ------- | ------- | ----------- |
| `$MAX_STACK_SIZE` | 4096 | Interpreter stack size in bytes. Increase for deeply recursive programs. |
| `$MEMORY_MAP_EXTENSION` | 0 | Extra zero bytes appended to the story file, available as writable memory at runtime. |
| `$GLULX_OBJECT_EXT_BYTES` | 0 | Additional bytes per object record. Must be a multiple of 4. Available for custom use by the game or library. |

### 15.4.4 Z-Code-Specific Settings

**[Z-machine]** These settings apply only when compiling for the
Z-machine:

| Setting | Default | Description |
| ------- | ------- | ----------- |
| `$ZCODE_HEADER_EXT_WORDS` | 3 | Words in the Z-code header extension table. |
| `$ZCODE_HEADER_FLAGS_3` | 0 | Value of Flags 3 in the header extension (Z-Machine Standard 1.1). |
| `$ZCODE_FILE_END_PADDING` | 1 | If 1, pad the story file to a multiple of 512 bytes. Set to 0 to omit padding. |
| `$ZCODE_MAX_INLINE_STRING` | 32 | Maximum length of a string literal assembled inline rather than placed in the string pool. |
| `$ZCODE_COMPACT_GLOBALS` | 0 | If 1, reuse global variable slots that are declared but never written. |
| `$ZCODE_LESS_DICT_DATA` | 0 | If 1, use 2 data bytes per dictionary entry instead of 3, saving space in large dictionaries. |

### 15.4.5 Compiler Behavior Settings

These settings control compiler internals and output features:

| Setting | Default | Description |
| ------- | ------- | ----------- |
| `$HASH_TAB_SIZE` | 512 | Size of the symbol hash table. Larger values may speed compilation of very large programs. |
| `$TRANSCRIPT_FORMAT` | 0 | Format of the `gametext.txt` file: 0 for human-readable, 1 for machine-parseable. |
| `$SERIAL` | auto | Six-digit serial number for the story file header. Default uses the compilation date (YYMMDD). |

### 15.4.6 Optimization and Output Flags

These boolean settings (0 or 1) control compiler optimization passes
and output format. See §12.5 for detailed descriptions of each.

| Setting | Default | Description |
| ------- | ------- | ----------- |
| `$OMIT_UNUSED_ROUTINES` | 0 | If 1, strip routines that are never called (dead-code elimination). |
| `$WARN_UNUSED_ROUTINES` | 0 | If 1, warn about unused routines in game source; if 2, also in library files. |
| `$STRIP_UNREACHABLE_LABELS` | 1 | If 1, remove labels in generated code that can never be reached. |
| `$OMIT_SYMBOL_TABLE` | 0 | If 1, omit property, array, and symbol names from the output file. |

### 15.4.7 Dictionary and Grammar Flags

These boolean settings (0 or 1) control dictionary entry flags and
grammar table features. See §12.5 for detailed descriptions.

| Setting | Default | Description |
| ------- | ------- | ----------- |
| `$DICT_IMPLICIT_SINGULAR` | 0 | If 1, dictionary words in noun context automatically get the singular flag. |
| `$DICT_TRUNCATE_FLAG` | 0 | If 1, flag bit 6 marks words truncated beyond `$DICT_WORD_SIZE`. |
| `$LONG_DICT_FLAG_BUG` | 1 | If 1, retain legacy buggy handling of `//p` on long dictionary words. Set to 0 for correct behavior. |
| `$GRAMMAR_META_FLAG` | 0 | If 1, enable per-action meta flags in the grammar table. |

---

## 15.5 Memory Allocation Model

Understanding how the compiler manages its own memory helps explain why
most historical memory settings are no longer needed.

### 15.5.1 Dynamic Memory Lists

**[Compiler 6.36+]** Since version 6.36, the compiler uses **dynamic
memory lists** for nearly all internal tables. A memory list is an
array that starts empty (or with a small initial allocation) and grows
on demand. When a table needs more space, the compiler calls
`ensure_memory_list_available()`, which allocates room according to a
simple growth formula:

```
new_count = 2 × requested_count + 8
```

This exponential-with-headroom strategy means that a table that grows
incrementally (one item at a time) will double in capacity each time it
overflows, amortizing the cost of reallocation. The `+ 8` term ensures
that very small tables get a reasonable minimum allocation without
requiring a separate code path.

Memory lists never shrink during compilation. They are freed in bulk
when the compiler finishes.

### 15.5.2 Tracking Memory Usage

The compiler tracks total allocated memory in the internal variable
`malloced_bytes`. You can inspect memory usage with the trace option:

```
inform $!MEM adventure.inf
```

This prints a line for each internal allocation and deallocation,
showing the purpose and size. For a summary of all compiler statistics
including memory, use the `-s` switch:

```
inform -s adventure.inf
```

The statistics output includes the total memory allocated during
compilation, which is useful for verifying that a build stays within
the resources available on constrained systems.

### 15.5.3 What Remains Fixed

Not everything is dynamically allocated. The settings listed in §15.4
that affect the **output format** — dictionary word size, attribute
byte count, stack size, and similar parameters — must be known before
compilation begins because they determine the layout of the story file.
These are genuine configuration values, not arbitrary workspace limits.

---

## 15.6 Removed and Obsolete Settings

Over the course of compiler versions 6.36 and 6.40, more than 40
memory settings were removed as the compiler moved to dynamic
allocation. Attempting to use a removed setting produces a warning
rather than an error, so that old build scripts continue to work:

**Inform 6 settings (removed in 6.36):**

```
$ALLOC_CHUNK_SIZE        $MAX_ACTIONS             $MAX_ADJECTIVES
$MAX_ARRAYS              $MAX_CLASSES             $MAX_DICT_ENTRIES
$MAX_EXPRESSION_NODES    $MAX_GLOBAL_VARIABLES    $MAX_INCLUSION_DEPTH
$MAX_INDIV_PROP_TABLE_SIZE  $MAX_LABELS           $MAX_LINESPACE
$MAX_LINK_DATA_SIZE      $MAX_LOCAL_VARIABLES     $MAX_LOW_STRINGS
$MAX_NUM_STATIC_STRINGS  $MAX_OBJECTS             $MAX_OBJ_PROP_COUNT
$MAX_OBJ_PROP_TABLE_SIZE $MAX_PROP_TABLE_SIZE     $MAX_QTEXT_SIZE
$MAX_SOURCE_FILES        $MAX_STATIC_DATA         $MAX_STATIC_STRINGS
$MAX_SYMBOLS             $MAX_TRANSCRIPT_SIZE     $MAX_UNICODE_CHARS
$MAX_VERBS               $MAX_VERBSPACE           $MAX_ZCODE_SIZE
$SYMBOLS_CHUNK_SIZE
```

Using any of these produces:

> The Inform 6 memory setting "X" is no longer needed and has been
> withdrawn.

**Inform 5 settings (long obsolete):**

```
$BANK_CHUNK_SIZE         $BUFFER_LENGTH           $MAX_BANK_SIZE
$MAX_FORWARD_REFS        $MAX_GCONSTANTS          $MAX_OLDEPTH
$MAX_ROUTINES            $STACK_LONG_SLOTS        $STACK_SHORT_LENGTH
$STACK_SIZE
```

Using any of these produces:

> The Inform 5 memory setting "X" has been withdrawn.

**Module and linking settings (removed in 6.40):**

The module compilation system (`-M` switch) and linking (`Link`
directive) were removed entirely in version 6.40. Any settings related
to module linking are also obsolete.

If you are porting a project from an older compiler, you can safely
delete all of these settings from your build scripts and ICL files.
The compiler now allocates memory dynamically for all of these areas.

---

## 15.7 When and How to Adjust Settings

Most projects will never need to change any compiler setting. When you
do, it is usually in response to a specific compiler message or a
conscious design decision about the output format. Here are the most
common scenarios.

### 15.7.1 Enlarging Dictionary Word Resolution

**[Glulx]** The default dictionary resolution of 9 characters is
sufficient for most English-language games. For games in languages with
long compound words (such as German or Finnish), or for games that need
to distinguish very similar long words, increase `$DICT_WORD_SIZE`:

```inform6
! In an ICL file or on the command line:
! $DICT_WORD_SIZE=12

! The parser will now distinguish the first 12 characters
! of each word, rather than 9.
```

For full Unicode dictionary support (necessary if your game accepts
input in non-Latin scripts), also set `$DICT_CHAR_SIZE=4`:

```
inform -G $DICT_WORD_SIZE=12 $DICT_CHAR_SIZE=4 adventure.inf
```

**[Z-machine]** Dictionary word size cannot be changed in Z-code; it
is fixed at 6 characters (V3) or 9 characters (V4+) by the VM
specification.

### 15.7.2 Enabling Dead-Code Stripping

For production releases, enabling dead-code elimination can
substantially reduce the story file size, especially when using the
standard library:

```
inform -G $OMIT_UNUSED_ROUTINES=1 adventure.inf
```

To preview which routines would be removed without actually removing
them, use the warning mode first:

```
inform -G $WARN_UNUSED_ROUTINES=2 adventure.inf
```

Level 2 reports unused routines in both the game source and library
files. Level 1 reports only game-source routines.

### 15.7.3 Increasing the Glulx Stack Size

**[Glulx]** If your game uses deep recursion or complex nested function
calls, you may encounter a stack overflow at runtime. The solution is
to increase the stack allocation:

```
inform -G $MAX_STACK_SIZE=8192 adventure.inf
```

The value is in bytes. The default of 4096 is sufficient for most
games; values of 8192 or 16384 handle all but the most extreme cases.

### 15.7.4 Adding Extra Attributes

**[Glulx]** If 56 attributes (the default) are not enough, increase
`$NUM_ATTR_BYTES`. The value must follow the pattern 4*n* + 3:

```
inform -G $NUM_ATTR_BYTES=11 adventure.inf
```

This provides 88 attributes (11 × 8). The maximum is 39 bytes (312
attributes).

**[Z-machine]** The number of attribute bytes is fixed at 6 (48
attributes) by the Z-machine specification and cannot be changed.

### 15.7.5 Reserving Runtime Memory

**[Glulx]** The `$MEMORY_MAP_EXTENSION` setting appends extra writable
bytes at the end of the story file. This memory is available at runtime
for custom data storage by the game or library:

```
inform -G $MEMORY_MAP_EXTENSION=4096 adventure.inf
```

Similarly, `$GLULX_OBJECT_EXT_BYTES` reserves extra bytes in each
object record (must be a multiple of 4):

```
inform -G $GLULX_OBJECT_EXT_BYTES=8 adventure.inf
```

These extra bytes are initialized to zero and can be read and written
using assembly-level memory access.

### 15.7.6 Reducing Z-Code File Size

**[Z-machine]** When a Z-code game is pressing against size limits,
several settings can reclaim small amounts of space:

- `$ZCODE_LESS_DICT_DATA=1` saves one byte per dictionary entry by
  reducing the per-word data from 3 bytes to 2.
- `$ZCODE_COMPACT_GLOBALS=1` reuses global variable slots that are
  declared but never written.
- `$OMIT_UNUSED_ROUTINES=1` strips routines that are never called.
- `$OMIT_SYMBOL_TABLE=1` omits symbol names from the output.
- `$ZCODE_FILE_END_PADDING=0` omits the trailing padding that rounds
  the file to a 512-byte boundary.

These techniques are useful when a game is a few kilobytes over the V5
or V8 limit and switching to Glulx is not an option.

---

## 15.8 Practical Guidance

### 15.8.1 Choosing a Target Platform

The choice between Z-machine versions and Glulx is the single most
important decision affecting limits. Here is a summary of the
trade-offs:

| Criterion | Z-machine V5 | Z-machine V8 | Glulx |
| --------- | ------------ | ------------ | ----- |
| Max file size | 256 KB | 512 KB | Effectively unlimited |
| Max objects | 65,535 | 65,535 | No hard limit |
| Max attributes | 48 | 48 | 56+ (configurable) |
| Max properties | 63 | 63 | No hard limit |
| Integer size | 16-bit | 16-bit | 32-bit |
| Local variables | 15 | 15 | 118 |
| Dictionary resolution | 9 chars | 9 chars | Configurable |
| Interpreter support | Excellent | Good | Good |
| Multimedia | Limited | Limited | Full (via Glk) |

**V5** is the right default for small to medium games. It offers the
widest interpreter compatibility and is the compiler's default target.

**V8** extends the file size limit to 512 KB without changing the
instruction set. Choose V8 when your game exceeds 256 KB but you want
to remain in the Z-machine ecosystem.

**Glulx** removes nearly all size constraints. Choose it for large
games, games requiring many attributes or properties, games needing
32-bit arithmetic, games with multimedia, or games targeting non-Latin
scripts with Unicode support.

### 15.8.2 General Recommendations

1. **Start with defaults.** The compiler's default settings are
   appropriate for the vast majority of projects. Do not adjust
   settings pre-emptively.

2. **Let the compiler tell you.** When a limit is reached, the compiler
   produces a clear error or warning message identifying the constraint.
   Adjust the specific setting mentioned.

3. **Prefer Glulx for large projects.** If you find yourself adjusting
   multiple Z-machine limits, consider switching to Glulx with `-G`
   instead. One target change often eliminates several limit workarounds.

4. **Clean up old settings.** If you inherit a project with many
   `$MAX_*` settings from a pre-6.36 compiler, remove them. They are
   ignored by the current compiler and produce warning messages.

5. **Use `-s` to monitor.** The statistics switch shows how much of
   each VM resource your game consumes. Run it periodically during
   development to catch approaching limits early:

   ```
   inform -s adventure.inf
   ```

6. **Use `$LIST` to review.** To see every configurable setting and
   its current value:

   ```
   inform $LIST
   ```

   This is especially useful when diagnosing unexpected behavior
   caused by a setting inherited from an ICL file or header comment.

---

*See also:* §11 (Invoking the Compiler), §12 (Compiler Switches and
Settings), §13 (Compilation Model), §14 (Error Messages and
Diagnostics).
