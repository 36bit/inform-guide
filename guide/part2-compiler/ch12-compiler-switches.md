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

# Chapter 12: Compiler Switches and Settings

This chapter provides a comprehensive reference for every switch, path
option, memory setting, and trace option recognized by the compiler.
Chapter 11 explains how to invoke the compiler and where options may be
supplied (command line, ICL files, `!%` header comments); this chapter
documents what each option does in detail.

---

## 12.1 Dash Switches

Dash switches are single-character flags prefixed with `-`. They control
the compiler's target platform, output format, debugging features, and
diagnostic behavior. Many accept an optional numeric suffix to select a
specific level or mode; when no suffix is given, the switch simply
toggles on.

A switch can be negated by inserting a tilde (`~`) between the dash and
the letter. For example, `-~S` turns off strict error checking, and
`-~H` disables Huffman compression.

### 12.1.1 Lowercase Switches

| Switch | Meaning |
| ------ | ------- |
| `-a`   | Trace assembly language (see below) |
| `-c`   | Concise error messages |
| `-d`   | Contract double spaces after full stops in text (see below) |
| `-e`   | Economy mode — use declared abbreviations to compress text |
| `-f`   | Frequencies mode — show abbreviation frequency data |
| `-g`   | Trace function calls at runtime (see below) |
| `-h`   | Print help information (see below) |
| `-i`   | Ignore in-file `Switches` directive |
| `-k`   | Output debugging information file |
| `-q`   | Quiet about obsolete usages |
| `-r`   | Record all game text to `gametext.txt` |
| `-s`   | Print compilation statistics |
| `-u`   | Calculate most useful text abbreviations (slow) |
| `-v`   | Set target virtual machine version (see below) |
| `-w`   | Suppress all warning messages |
| `-x`   | Print `#` for every 100 lines compiled |
| `-z`   | Print memory map of the compiled story file |

### 12.1.2 Uppercase Switches

| Switch | Meaning |
| ------ | ------- |
| `-B`   | Big memory model — use separate packing offsets for routines and strings (V6/V7 only) |
| `-C`   | Character set selection (see below) |
| `-D`   | Define the `DEBUG` constant automatically |
| `-E`   | Error message format (see below) |
| `-G`   | Compile to Glulx format instead of Z-machine |
| `-H`   | Huffman string compression (Glulx; on by default) |
| `-R`   | RISC OS file type format (platform-specific) |
| `-S`   | Strict runtime error checking — define `STRICT_MODE` (on by default) |
| `-T`   | Throwback error reporting (Archimedes only) |
| `-V`   | Print compiler version and exit immediately |
| `-W`   | Z-code header extension table size (see below) |
| `-X`   | Compile with INFIX debugging support — define `INFIX` |

### 12.1.3 Switch Details

**`-a` — Assembly Trace**

Prints a trace of the assembly-language instructions generated for each
function. This is the primary tool for understanding how the compiler
translates source code into virtual machine instructions. The switch
accepts a numeric suffix to control verbosity:

| Level | Effect |
| ----- | ------ |
| `-a` or `-a1` | Show assembly instructions |
| `-a2` | Also show hex byte encoding of each instruction |
| `-a3` | Also show branch offset optimization decisions |
| `-a4` | Most verbose — include all internal detail |

Assembly tracing produces large amounts of output. It is most useful
when applied to a small test file or when debugging a specific routine.

**`-c` — Concise Errors**

Suppresses the source context lines that normally accompany error
messages. Useful when piping compiler output to an IDE or tool that
parses only the error location and message text.

**`-d` — Double Space Contraction**

Controls how the compiler handles consecutive spaces in game text:

| Level | Effect |
| ----- | ------ |
| `-d` or `-d1` | Contract double spaces after full stops (`.`) in printed text |
| `-d2` | Also contract double spaces after exclamation marks (`!`) and question marks (`?`) |

The `-d1` behavior is a legacy feature from early Z-machine games where
double spacing after sentence-ending punctuation was conventional.

**`-e` — Economy Mode**

Enables use of declared `Abbreviate` strings to compress game text.
Without this switch, `Abbreviate` directives are accepted but ignored.
Economy mode can reduce the size of the string pool significantly in
text-heavy games.

**`-f` — Frequencies**

Displays character and substring frequency data, useful when choosing
abbreviations. Only meaningful when used with `-e` (economy mode).

**`-g` — Runtime Function Tracing**

Inserts calls at the start of functions so that function names are
printed at runtime when they are called. The numeric suffix controls
which functions are traced:

| Level | Effect |
| ----- | ------ |
| `-g` or `-g1` | Trace calls to user-defined functions only |
| `-g2` | Also trace calls to library functions |
| `-g3` | Trace all function calls, including veneer routines |

This switch adds overhead to the compiled game and is intended for
debugging only.

**`-h` — Help**

Prints help information and exits:

| Level | Effect |
| ----- | ------ |
| `-h` or `-h0` | General help and usage summary |
| `-h1` | Show file-naming conventions |
| `-h2` | List all available dash switches |

**`-i` — Ignore In-File Switches**

Causes the compiler to skip any `Switches` directive found inside the
source file. It does not affect `!%` header comments or command-line
options. Useful when overriding a source file's settings without
editing it.

**`-k` — Debug File**

Outputs a debugging information file named `gameinfo.dbg`. Since
compiler version 6.33, this file is in XML format. The debug file
contains source-level information (function names, line numbers, local
variables) that an interpreter or debugging tool can use to provide
source-level debugging.

**`-q` — Quiet About Obsolete Usages**

Suppresses warnings about deprecated language features and obsolete
coding patterns.

**`-r` — Record Text**

Writes all compiled game text to a transcript file named `gametext.txt`.
The format of this file is controlled by the `$TRANSCRIPT_FORMAT`
setting: 0 for human-readable (default) and 1 for machine-parseable.

**`-s` — Statistics**

Prints a detailed statistical summary of the compiled story file,
including counts and sizes for objects, properties, dictionary entries,
action routines, grammar entries, code size, and string space.

**`-u` — Abbreviation Optimization**

Analyses all game text to determine the most space-efficient set of
abbreviations. This process is computationally expensive and may take
several minutes for a large game. The results can be used in `Abbreviate`
directives and then applied with `-e`.

**`-v` — Version Selection**

Sets the Z-machine version for the compiled story file:

| Switch | Z-machine version |
| ------ | ----------------- |
| `-v3`  | Version 3 |
| `-v4`  | Version 4 |
| `-v5`  | Version 5 (default) |
| `-v6`  | Version 6 |
| `-v7`  | Version 7 |
| `-v8`  | Version 8 |

When compiling for Glulx (with `-G`), the `-v` switch specifies the
Glulx VM version instead, using dotted notation: `-v3.1.0`, `-v3.1.2`,
etc. See §12.7 for details on what each version provides.

**`-w` — Suppress Warnings**

Suppresses all non-fatal warning messages. Errors are still reported.

**`-x` — Progress Indicator**

Prints a `#` character for every 100 lines of source code compiled,
providing a simple visual progress indicator for large compilations.

**`-z` — Memory Map**

Prints a map of the compiled story file's layout in the virtual machine's
address space. See the `$!MAP` trace option for more detailed variants.

**`-B` — Big Memory Model**

Enables separate packing offsets for routines and strings, used in
Z-machine versions 6 and 7. This allows the routine and string areas to
each occupy a larger portion of the address space. In other versions,
this switch has no effect.

**`-C` — Character Set**

Selects the character encoding used to interpret source file bytes:

| Switch | Encoding |
| ------ | -------- |
| `-C0`  | Plain ASCII (0x20–0x7E) |
| `-C1`  | ISO 8859-1 (Latin-1) — default |
| `-C2`  | ISO 8859-2 (Latin-2) |
| `-C3`  | ISO 8859-3 (Latin-3) |
| `-C4`  | ISO 8859-4 (Latin-4) |
| `-C5`  | ISO 8859-5 (Cyrillic) |
| `-C6`  | ISO 8859-6 (Arabic) |
| `-C7`  | ISO 8859-7 (Greek) |
| `-C8`  | ISO 8859-8 (Hebrew) |
| `-C9`  | ISO 8859-9 (Latin-5 / Turkish) |
| `-Cu`  | UTF-8 |

When `-Cu` is selected, the compiler reads source files as UTF-8 and
supports the full Unicode range in string literals and character
constants.

**`-D` — Define DEBUG**

Automatically defines the `DEBUG` constant at the start of compilation.
This is equivalent to placing `Constant DEBUG;` at the top of the source
file. When `DEBUG` is defined, the standard library enables its built-in
debugging verbs (PURLOIN, TREE, SHOWOBJ, etc.).

**`-E` — Error Format**

Selects the format of error and warning messages:

| Switch | Format style |
| ------ | ------------ |
| `-E0`  | Archimedes-style (default) |
| `-E1`  | Microsoft-style |
| `-E2`  | Macintosh MPW-style |

This is useful for integrating the compiler with IDEs that expect errors
in a particular format.

**`-G` — Glulx Target**

Compiles to Glulx format instead of Z-machine. When this switch is
active, several Z-machine-specific switches (such as `-v3`–`-v8` and
`-B`) become irrelevant, and Glulx-specific settings (such as
`$MAX_STACK_SIZE` and `$GLULX_OBJECT_EXT_BYTES`) become available.

**`-H` — Huffman Compression**

Enables Huffman encoding for string compression in Glulx output. This is
on by default when compiling for Glulx; use `-~H` to disable it. Has no
effect in Z-machine mode (which uses its own built-in string compression).

**`-S` — Strict Mode**

Defines the `STRICT_MODE` constant, which enables runtime error checking
in the compiled game (such as array bounds checking and argument count
validation). This is on by default; use `-~S` to disable it for
production releases where the slight performance gain matters.

**`-V` — Version**

Prints the compiler version string and exits immediately without
compiling.

**`-W` — Header Extension Words**

Sets the number of words in the Z-code header extension table. The
default is `-W3`. This is a specialized setting relevant only to
Z-machine output; most authors will never need to change it.

```
inform -W5 adventure.inf
```

**`-X` — INFIX Debugging**

Defines the `INFIX` constant and compiles the game with support for the
Infix debugger. Infix is a powerful source-level debugger built into the
game itself, allowing the player to examine and modify objects,
properties, variables, and routines at runtime. Available for Z-machine
only.

---

## 12.2 Compound Switches

Multiple single-character switches can be combined into a single
argument. The compiler processes each character in the string from left
to right:

```
inform -v5Ds adventure.inf
```

This is equivalent to:

```
inform -v5 -D -s adventure.inf
```

It sets the Z-machine version to 5, defines `DEBUG`, and enables
statistics output.

### 12.2.1 Ordering Rules

Order matters when a switch requires a following character as a value.
The value character must immediately follow its switch letter:

- `-v5D` is correct — version 5, then DEBUG
- `-Dv5` is correct — DEBUG, then version 5
- `-5v` is **incorrect** — `5` is not a valid switch

Similarly, `-E1s` sets the error format to Microsoft-style and turns on
statistics, but `-sE1` sets statistics and then error format 1.

### 12.2.2 Negation in Compound Switches

The tilde negation prefix applies to the immediately following character:

```
inform -D~Ss adventure.inf
```

This defines `DEBUG`, turns **off** strict mode, and turns on statistics.

---

## 12.3 Path Options

Path options use the `+` prefix and control where the compiler searches
for include files and where it writes output files. These options are
only available on the command line or in ICL files — they cannot appear
in `!%` header comments.

### 12.3.1 Setting Include Paths

The most common path option sets the search path for `Include` files:

```
inform +include_path=i6lib adventure.inf
```

The short form omits the `include_path=` prefix:

```
inform +i6lib adventure.inf
```

Both forms set the directory where the compiler looks for files specified
in `Include` directives.

### 12.3.2 Multiple Directories

Multiple directories can be specified in a single path option, separated
by commas:

```
inform +include_path=i6lib,contrib,extensions adventure.inf
```

The compiler searches the directories in the order listed, left to right.
The path separator is always a comma (`,`) regardless of the operating
system.

### 12.3.3 Adding to Paths

The `++` prefix (double plus) adds a directory to an existing path
rather than replacing it. The newly added directory is searched **first**,
before any previously set directories:

```
inform +include_path=i6lib ++include_path=mylib adventure.inf
```

Here `mylib` is searched first, then `i6lib`.

### 12.3.4 Other Path Variables

The compiler recognizes several path variable names:

| Path variable | Purpose |
| ------------- | ------- |
| `include_path` | Directories for `Include` file search |
| `source_path`  | Directory for the main source file |
| `code_path`    | Directory for compiled output files |
| `icl_path`     | Directory for ICL command files |

### 12.3.5 Long Option Syntax

Since compiler version 6.35, paths may also be set using long options:

```
inform --path include_path=i6lib adventure.inf
inform --addpath include_path=mylib adventure.inf
```

The `--addpath` form is equivalent to `++`.

---

## 12.4 Dollar Memory Settings

Dollar settings use the `$SETTING=value` syntax to control compiler
memory allocation, dictionary parameters, output format details, and
other numeric configuration values. They can appear on the command line,
in ICL files, or in `!%` header comments.

### 12.4.1 Syntax

```
inform $DICT_WORD_SIZE=9 adventure.inf
```

The setting name is case-sensitive and must match exactly. The value must
be a non-negative integer. The long option equivalent (since 6.35) is:

```
inform --opt DICT_WORD_SIZE=9 adventure.inf
```

To display all current settings and their values, use `$LIST`:

```
inform $LIST
```

### 12.4.2 Defining Arbitrary Constants

The `$#` prefix defines a numeric constant visible to the source code:

```
inform $#MAX_SCORE=1000 adventure.inf
```

This is equivalent to `Constant MAX_SCORE 1000;` at the top of the source
file. The long option form is `--define MAX_SCORE=1000`. This feature was
added in compiler version 6.35.

### 12.4.3 Dictionary and Text Settings

These settings control how the compiler builds the dictionary and handles
text data:

| Setting | Z-code default | Glulx default | Description |
| ------- | -------------- | ------------- | ----------- |
| `$DICT_WORD_SIZE` | ignored (auto: 4 in v3, 6 in v4+) | 9 | Bytes of encoded text per dictionary word. In Z-code the setting is ignored and the compiler auto-sizes entries (4 bytes / 6 Z-characters in v3; 6 bytes / 9 Z-characters in v4+); see note below. In Glulx, freely adjustable; longer values let the parser distinguish longer words. |
| `$DICT_CHAR_SIZE` | 1 | 1 | Bytes per character in dictionary words. Set to 4 in Glulx to support full Unicode dictionary entries. In Z-code, this is always 1. |
| `$MAX_ABBREVS` | 64 | N/A | Maximum number of `Abbreviate` directives. Z-code only; not meaningful in Glulx. In Z-code, `$MAX_ABBREVS` and `$MAX_DYNAMIC_STRINGS` share a pool of exactly 96 slots and must sum to 96. |
| `$MAX_DYNAMIC_STRINGS` | 32 | 100 | Maximum number of string substitution variables (`@00`, `@(0)`, etc.). In Z-code, `$MAX_ABBREVS` and `$MAX_DYNAMIC_STRINGS` share a pool of exactly 96 slots and must sum to 96. |

> **Note (Z-code `$DICT_WORD_SIZE`):** In Z-code the memory setting is
> ignored — the compiler auto-sizes dictionary entries to 4 bytes (v3)
> or 6 bytes (v4+). However, due to a current compiler quirk, explicitly
> specifying `$DICT_WORD_SIZE` on the command line (or in an `!%`
> header) for a Z-code target is a fatal error unless the value is
> exactly 6, *even if the value would match the version's natural byte
> size* (e.g., `$DICT_WORD_SIZE=4` for v3 still fails). The safest
> course is to omit the setting entirely for Z-code projects.

### 12.4.4 Grammar and Action Settings

| Setting | Z-code default | Glulx default | Description |
| ------- | -------------- | ------------- | ----------- |
| `$GRAMMAR_VERSION` | 1 | 2 | Grammar table format version. Version 1 is the classic Infocom format; version 2 is the standard Inform format; version 3 (added in 6.43) extends version 2 with additional features. |
| `$INDIV_PROP_START` | 64 | 256 | First individual property number. Properties below this value are common properties; those at or above it are individual properties. |

### 12.4.5 Z-Code Specific Settings

These settings apply only when compiling for the Z-machine:

| Setting | Default | Description |
| ------- | ------- | ----------- |
| `$NUM_ATTR_BYTES` | 6 | Number of bytes used for attribute flags. Each byte provides 8 attributes, so the default gives 48 attributes. The Z-machine specification fixes this at 6. |
| `$ZCODE_HEADER_EXT_WORDS` | 3 | Number of words in the Z-code header extension table. Most games use the default of 3. |
| `$ZCODE_HEADER_FLAGS_3` | 0 | Value of the Flags 3 word in the header extension table, as defined in the Z-Machine Standard 1.1. |
| `$ZCODE_FILE_END_PADDING` | 1 | If 1, pad the game file to a multiple of 512 bytes. Set to 0 to omit the padding. |
| `$ZCODE_MAX_INLINE_STRING` | 32 | Maximum length of a string literal that can be assembled inline (as a sequence of Z-characters) rather than placed in the string pool. |
| `$ZCODE_COMPACT_GLOBALS` | 0 | If 1, reuse global variable slots that are declared but never written. This can save a small amount of header space. |

### 12.4.6 Glulx Specific Settings

These settings apply only when compiling for Glulx:

| Setting | Default | Description |
| ------- | ------- | ----------- |
| `$NUM_ATTR_BYTES` | 7 | Number of bytes for attribute flags. Must be a multiple of 4 plus 3 (so 7, 11, 15, ...). The default of 7 provides 56 attributes. |
| `$MAX_STACK_SIZE` | 4096 | Maximum interpreter stack size in bytes. Increase this if you encounter stack overflow errors at runtime in deeply recursive programs. |
| `$GLULX_OBJECT_EXT_BYTES` | 0 | Additional bytes allocated in each object's data block. Must be a multiple of the word size (4). These extra bytes are available for custom use by the game or library. |
| `$MEMORY_MAP_EXTENSION` | 0 | Number of extra zero bytes appended to the end of the game file. Can be used to reserve writable memory for runtime use. |

### 12.4.7 Compiler Internals

| Setting | Default | Description |
| ------- | ------- | ----------- |
| `$HASH_TAB_SIZE` | 512 | Size of the hash tables used by the compiler's symbol banks. Larger values can speed up compilation of very large programs at the cost of memory. |
| `$TRANSCRIPT_FORMAT` | 0 | Format of the `gametext.txt` file produced by `-r`. Set to 0 for human-readable or 1 for machine-parseable output. |
| `$SERIAL` | 0 | A six-digit serial number (0–999999) to embed in the story file header. When set to 0 (the default), the compiler uses the current date in YYMMDD format. |

### 12.4.8 Removed Settings

Many memory settings that existed in earlier versions of the compiler
were removed in version 6.36, when the compiler switched to dynamic
memory allocation. Attempting to use a removed setting produces a
warning message explaining that it is obsolete. The removed settings
include:

`$ALLOC_CHUNK_SIZE`, `$MAX_OBJECTS`, `$MAX_CLASSES`, `$MAX_SYMBOLS`,
`$MAX_PROP_TABLE_SIZE`, `$MAX_INDIV_PROP_TABLE_SIZE`,
`$MAX_OBJ_PROP_COUNT`, `$MAX_OBJ_PROP_TABLE_SIZE`, `$MAX_ARRAYS`,
`$MAX_STATIC_DATA`, `$MAX_ADJECTIVES`, `$MAX_VERBS`, `$MAX_VERBSPACE`,
`$MAX_LABELS`, `$MAX_EXPRESSION_NODES`, `$MAX_SOURCE_FILES`,
`$MAX_INCLUSION_DEPTH`, `$MAX_ACTIONS`, `$MAX_LINESPACE`,
`$MAX_ZCODE_SIZE`, `$MAX_LINK_DATA_SIZE`, `$MAX_TRANSCRIPT_SIZE`,
`$MAX_DICT_ENTRIES`, `$MAX_NUM_STATIC_STRINGS`, `$MAX_UNICODE_CHARS`,
`$MAX_STATIC_STRINGS`, `$MAX_LOW_STRINGS`, `$MAX_GLOBAL_VARIABLES`,
`$MAX_LOCAL_VARIABLES`, `$MAX_QTEXT_SIZE`.

If you are porting a project that used these settings under an older
compiler, you can safely remove them. The compiler now allocates memory
dynamically for all of these areas.

---

## 12.5 Special Dollar Settings

Several dollar settings function as boolean flags (0 or 1) that enable
specific compiler features. These merit individual discussion because
they affect the compiled output in significant ways.

### 12.5.1 `$OMIT_UNUSED_ROUTINES`

```
inform $OMIT_UNUSED_ROUTINES=1 adventure.inf
```

When set to 1, the compiler performs dead-code elimination: any routine
that is never called (directly or indirectly from the main entry point)
is omitted from the compiled game file. This can substantially reduce
the output size, especially when using a large library where many
routines go unused.

The analysis is conservative — if a routine's address is stored in a
variable or property (i.e., it is used as a function value), it is
considered reachable even if it is never explicitly called.

See also `$WARN_UNUSED_ROUTINES` to identify unused routines without
removing them.

### 12.5.2 `$WARN_UNUSED_ROUTINES`

```
inform $WARN_UNUSED_ROUTINES=2 adventure.inf
```

Controls warnings about routines that are compiled but never called:

| Value | Behavior |
| ----- | --------- |
| 0     | No warnings (default) |
| 1     | Warn about unused routines in the game source only |
| 2     | Warn about unused routines in both game source and library files |

This is a diagnostic companion to `$OMIT_UNUSED_ROUTINES`. Running with
level 2 can reveal dead code in the library that may be eliminated.

### 12.5.3 `$OMIT_SYMBOL_TABLE`

```
inform $OMIT_SYMBOL_TABLE=1 adventure.inf
```

When set to 1, the compiler skips the compilation of property, array,
and symbol names into the output file. This reduces the game file size
but has several consequences:

- `print (property) p` prints a number instead of a name.
- Runtime error messages omit symbol names.
- Constants that reference symbol names are not available.

This is suitable for production releases where file size is more
important than debugging convenience.

### 12.5.4 `$ZCODE_LESS_DICT_DATA`

```
inform $ZCODE_LESS_DICT_DATA=1 adventure.inf
```

**Z-code only.** Reduces each dictionary entry from 3 data bytes to 2.
This saves one byte per dictionary word, which can be significant in
games with very large dictionaries pressing up against Z-machine size
limits.

Restrictions: When this setting is enabled, the `#dict_par3` system
constant is not available, and grammar version 1 cannot be used.

### 12.5.5 `$DICT_IMPLICIT_SINGULAR`

```
inform $DICT_IMPLICIT_SINGULAR=1 adventure.inf
```

When set to 1, dictionary words that appear in a noun context
automatically receive the singular (`//s`) flag unless the plural
(`//p`) flag has been explicitly set. This allows the library to
distinguish singular from plural nouns without requiring the author to
manually mark every singular word.

Added in compiler version 6.43.

### 12.5.6 `$DICT_TRUNCATE_FLAG`

```
inform $DICT_TRUNCATE_FLAG=1 adventure.inf
```

Controls the meaning of dictionary flag bit 6:

| Value | Behavior |
| ----- | --------- |
| 0     | Flag 6 is set on all verb words (legacy behavior, default) |
| 1     | Flag 6 is set on words that were truncated beyond `$DICT_WORD_SIZE` characters |

When set to 1, the flag provides useful information: it lets the game
detect that a player's input word was longer than the dictionary's
resolution and may have been ambiguous. Added in compiler version 6.43.

### 12.5.7 `$GRAMMAR_META_FLAG`

```
inform $GRAMMAR_META_FLAG=1 adventure.inf
```

When set to 1, enables per-action meta flags in the grammar table.
Normally, the "meta" attribute (meaning an action that is out-of-world,
like SAVE or QUIT) is stored on verb dictionary words. With this setting,
each individual action can be marked as meta, and meta action numbers are
sorted before non-meta ones.

When enabled, the compiler defines a constant
`#highest_meta_action_number` that the library can use at runtime to
quickly test whether an action is meta.

Test for this feature using `#ifdef GRAMMAR_META_FLAG` rather than
checking the value. Added in compiler version 6.43.

### 12.5.8 `$LONG_DICT_FLAG_BUG`

```
inform $LONG_DICT_FLAG_BUG=0 adventure.inf
```

Controls backward compatibility with a long-standing bug in how the
`//p` (plural) flag was applied to dictionary words longer than
`$DICT_WORD_SIZE`:

| Value | Behavior |
| ----- | --------- |
| 1     | Retain old buggy behavior (default) |
| 0     | Use correct flag handling |

The default of 1 preserves compatibility with existing games. Set to 0
only when developing new games or when you are certain the fix does not
affect your project.

### 12.5.9 `$STRIP_UNREACHABLE_LABELS`

```
inform $STRIP_UNREACHABLE_LABELS=1 adventure.inf
```

When set to 1 (the default), the compiler removes labels in generated
code that can never be reached. This is a minor optimization that
slightly reduces code size with no effect on behavior. Set to 0 only
if you suspect a compiler code-generation bug related to label stripping.

---

## 12.6 Trace Options

Trace options provide fine-grained control over compiler diagnostic
output. They use the `$!` prefix and were introduced in compiler version
6.40, replacing several older dash switches and the `Trace` directive.

### 12.6.1 Syntax

```
inform $!STATS adventure.inf
inform $!MAP=2 adventure.inf
```

The simple form `$!TRACEOPT` sets the named option to level 1. The form
`$!TRACEOPT=N` sets it to a specific level. All trace options default to
level 0 (off). The bare form `$!` (with no option name) prints a list of
all supported trace options.

The long option equivalents (since 6.35) are:

```
inform --trace STATS adventure.inf
inform --trace MAP=2 adventure.inf
inform --helptrace adventure.inf
```

### 12.6.2 Complete Trace Option Reference

| Trace option | Levels | Description |
| ------------ | ------ | ----------- |
| `$!ACTIONS`  | 1–2 | Show all actions defined. Level 2 also lists them as they are defined during compilation. |
| `$!ASM`      | 1–4 | Trace assembly (same as `-a`). Level 1: instructions. Level 2: add hex encoding. Level 3: add branch optimization. Level 4: maximum detail. |
| `$!BPATCH`   | 1–2 | Show backpatch results. Level 2 also shows markers as they are added during compilation. |
| `$!DICT`     | 1–2 | Display the dictionary table. Level 2 also shows the byte encoding of each entry. |
| `$!EXPR`     | 1–3 | Show expression parse trees. Higher levels show progressively more internal detail. |
| `$!FILES`    | 1 | List all files opened during compilation. |
| `$!FINDABBREVS` | 1–2 | Show decisions during abbreviation optimization (`-u`). Level 2 includes three-letter-block analysis. |
| `$!FREQ`     | 1 | Show abbreviation frequency data (same as `-f`). Only meaningful with `-e`. |
| `$!MAP`      | 1–3 | Print memory map (same as `-z`). Level 2 adds percentage utilization. Level 3 adds byte counts. |
| `$!MEM`      | 1 | Show the compiler's own internal memory allocations. |
| `$!OBJECTS`  | 1 | Display the object tree table. |
| `$!PROPS`    | 1 | Show all attributes and properties defined. |
| `$!RUNTIME`  | 1–3 | Insert runtime function call tracing (same as `-g`). Level 1: user functions. Level 2: add library. Level 3: add veneer. |
| `$!STATS`    | 1 | Print compilation statistics (same as `-s`). |
| `$!SYMBOLS`  | 1–2 | Display the symbol table. Level 2 also shows compiler-defined symbols. |
| `$!SYMDEF`   | 1 | Show each symbol as it is noticed and defined. Useful for debugging name conflicts. |
| `$!TOKENS`   | 1–3 | Show token lexing. Level 2 adds token types. Level 3 adds lexical context. |
| `$!VERBS`    | 1 | Display the verb grammar table. |

### 12.6.3 Relationship to Dash Switches

Several trace options have equivalent dash switches inherited from older
compiler versions. Both forms remain supported:

| Trace option | Equivalent switch |
| ------------ | ----------------- |
| `$!ASM`      | `-a` |
| `$!FREQ`     | `-f` |
| `$!MAP`      | `-z` |
| `$!RUNTIME`  | `-g` |
| `$!STATS`    | `-s` |

The trace option form is preferred in new projects because it supports
multiple levels and is more self-documenting. In older compiler versions
(before 6.40), additional dash switches existed for some of the
information now provided by trace options; these switches (`-b`, `-j`,
`-l`, `-m`, `-n`, `-o`, `-p`, `-t`) have been removed.

---

## 12.7 Target Selection

The compiler can produce output for two virtual machines: the Z-machine
and Glulx. The target is selected with the `-v` and `-G` switches and
affects nearly every aspect of compilation, from the available address
space to the dictionary format.

### 12.7.1 Z-Machine Versions

The Z-machine exists in several versions. The compiler supports versions
3 through 8, selected with the `-v` switch. The default is version 5.

| Version | Max size | Scale factor | Notable features |
| ------- | -------- | ------------ | ---------------- |
| **V3**  | 128 KB   | 2 | Original Infocom format. Status line. No bold/italic. 32 attributes. 1-byte property numbers. Score or time display. |
| **V4**  | 256 KB   | 4 | 64 KB of dynamic memory. Bold/italic text. Timed keyboard input. 48 attributes. |
| **V5**  | 256 KB   | 4 | The standard version for modern games. Full instruction set. Undo support. Unicode extensions. Recommended unless you have a reason to choose otherwise. |
| **V6**  | 512 KB   | 4 | Graphical version. Proportional fonts, pictures, sound, mouse input. Extended memory map. Rarely used; limited interpreter support. |
| **V7**  | 512 KB   | 4 | Extended memory map (like V6) with the V5 instruction set. No graphics. Rarely used. |
| **V8**  | 512 KB   | 8 | Largest Z-machine format. Same instruction set as V5 but with a higher scale factor that doubles the address space for routines and strings. Suitable for large text-heavy games that exceed V5 limits. |

**Internal parameters by version:**

| Parameter | V3 | V4 | V5 | V6 | V7 | V8 |
| --------- | -- | -- | -- | -- | -- | -- |
| Instruction set | 3 | 4 | 5 | 6 | 5 | 5 |
| Packed address scale | 2 | 4 | 4 | 4 | 4 | 8 |
| Length scale factor | 2 | 4 | 4 | 8 | 8 | 8 |
| Extended memory map | No | No | No | Yes | Yes | No |

### 12.7.2 Glulx

Glulx is selected with the `-G` switch:

```
inform -G adventure.inf
```

Glulx is a 32-bit virtual machine designed to overcome the Z-machine's
size limitations. It supports:

- A 32-bit address space (up to 4 GB, though practical limits are much
  lower).
- Unlimited objects, properties, attributes (beyond the Z-machine's fixed
  maximums).
- Full Unicode string handling.
- 32-bit integers (vs. 16-bit in the Z-machine).
- Larger stack and more local variables.
- Huffman-compressed strings (enabled by default with `-H`).
- The Glk I/O system for input, output, graphics, and sound.

When compiling for Glulx, several settings use different defaults
(for example, `$DICT_WORD_SIZE` defaults to 9 instead of 6), and
Z-machine-specific settings like `$ZCODE_HEADER_EXT_WORDS` are
ignored.

### 12.7.3 Specifying the Glulx VM Version

When in Glulx mode, the `-v` switch specifies the target Glulx VM
version using dotted notation:

```
inform -G -v3.1.2 adventure.inf
```

If no `-v` switch is given in Glulx mode, the compiler targets a default
Glulx version appropriate for the features used. The Glulx version
determines which VM opcodes and capabilities are available to the
compiled game; most interpreters support version 3.1.2 or later.

### 12.7.4 Choosing a Target

For most new interactive fiction projects:

- Use **Z-machine V5** (the default) for games of moderate size that
  need maximum interpreter compatibility. Nearly every Z-machine
  interpreter supports V5.
- Use **Z-machine V8** for large Z-machine games that exceed V5's 256 KB
  limit but must remain in Z-machine format.
- Use **Glulx** for games that need large dictionaries, many objects,
  full Unicode, or multimedia capabilities. Glulx interpreters are
  widely available on all major platforms.

Versions 3, 4, 6, and 7 are primarily of historical interest or for
specialized applications.

---

## 12.8 Quick Reference

### 12.8.1 All Dash Switches

```
-a  assembly trace          -B  big memory model (V6/V7)
-c  concise errors          -C  character set (-C0..-C9, -Cu)
-d  double-space contraction -D  define DEBUG
-e  economy mode            -E  error format (-E0..-E2)
-f  frequency data          -G  compile for Glulx
-g  trace function calls    -H  Huffman compression (Glulx)
-h  help                    -R  RISC OS file type format
-i  ignore Switches directive -S  strict mode (default on)
-k  debug file output       -T  throwback (Archimedes)
-q  quiet about obsolete    -V  print version and exit
-r  record text             -W  header extension words
-s  statistics              -X  INFIX debugger
-u  abbreviation optimiser
-v  version selection
-w  suppress warnings
-x  progress indicator
-z  memory map
```

### 12.8.2 All Dollar Settings

| Setting | Description | Details |
| ------- | ----------- | ------- |
| `$DICT_CHAR_SIZE` | Bytes per character in dictionary words (1 or 4) | §12.4.3 |
| `$DICT_IMPLICIT_SINGULAR` | Auto-set singular flag on noun words | §12.5.5 |
| `$DICT_TRUNCATE_FLAG` | Flag words truncated beyond `DICT_WORD_SIZE` | §12.5.6 |
| `$DICT_WORD_SIZE` | Characters stored per dictionary word | §12.4.3 |
| `$GLULX_OBJECT_EXT_BYTES` | Extra bytes per Glulx object data block | §12.4.6 |
| `$GRAMMAR_META_FLAG` | Per-action meta flags in grammar table | §12.5.7 |
| `$GRAMMAR_VERSION` | Grammar table format version (1, 2, or 3) | §12.4.4 |
| `$HASH_TAB_SIZE` | Compiler symbol hash table size | §12.4.7 |
| `$INDIV_PROP_START` | First individual property number | §12.4.4 |
| `$LONG_DICT_FLAG_BUG` | Retain long-word dictionary flag bug | §12.5.8 |
| `$MAX_ABBREVS` | Maximum number of `Abbreviate` directives | §12.4.3 |
| `$MAX_DYNAMIC_STRINGS` | Maximum string substitution variables | §12.4.3 |
| `$MAX_STACK_SIZE` | Glulx interpreter stack size in bytes | §12.4.6 |
| `$MEMORY_MAP_EXTENSION` | Extra zero bytes at end of Glulx file | §12.4.6 |
| `$NUM_ATTR_BYTES` | Bytes for attribute flags | §12.4.5, §12.4.6 |
| `$OMIT_SYMBOL_TABLE` | Skip symbol names in output file | §12.5.3 |
| `$OMIT_UNUSED_ROUTINES` | Dead-code elimination of uncalled routines | §12.5.1 |
| `$SERIAL` | Story file serial number (0 = use date) | §12.4.7 |
| `$STRIP_UNREACHABLE_LABELS` | Remove unreachable labels from code | §12.5.9 |
| `$TRANSCRIPT_FORMAT` | Game text transcript format (0 or 1) | §12.4.7 |
| `$WARN_UNUSED_ROUTINES` | Warn about uncalled routines (0, 1, or 2) | §12.5.2 |
| `$ZCODE_COMPACT_GLOBALS` | Reuse unused global variable slots | §12.4.5 |
| `$ZCODE_FILE_END_PADDING` | Pad Z-code file to 512-byte boundary | §12.4.5 |
| `$ZCODE_HEADER_EXT_WORDS` | Words in Z-code header extension table | §12.4.5 |
| `$ZCODE_HEADER_FLAGS_3` | Flags 3 value in header extension table | §12.4.5 |
| `$ZCODE_LESS_DICT_DATA` | Reduce Z-code dictionary entry size | §12.5.4 |
| `$ZCODE_MAX_INLINE_STRING` | Max length for inline Z-character strings | §12.4.5 |

### 12.8.3 All Trace Options

See §12.6.2 for full details including trace levels.

| Trace option | Description |
| ------------ | ----------- |
| `$!ACTIONS` | Show all actions defined |
| `$!ASM` | Trace assembly output (same as `-a`) |
| `$!BPATCH` | Show backpatch markers and results |
| `$!DICT` | Display the dictionary table |
| `$!EXPR` | Show expression parse trees |
| `$!FILES` | List all files opened during compilation |
| `$!FINDABBREVS` | Show abbreviation optimization decisions |
| `$!FREQ` | Show abbreviation frequency data (same as `-f`) |
| `$!MAP` | Print memory map (same as `-z`) |
| `$!MEM` | Show compiler internal memory allocations |
| `$!OBJECTS` | Display the object tree table |
| `$!PROPS` | Show all attributes and properties defined |
| `$!RUNTIME` | Insert runtime function call tracing (same as `-g`) |
| `$!STATS` | Print compilation statistics (same as `-s`) |
| `$!SYMBOLS` | Display the symbol table |
| `$!SYMDEF` | Show each symbol as it is noticed and defined |
| `$!TOKENS` | Show token lexing |
| `$!VERBS` | Display the verb grammar table |
