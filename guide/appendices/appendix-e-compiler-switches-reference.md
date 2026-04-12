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

# Appendix E: Compiler Switches Reference

This appendix provides a complete reference for every command-line switch,
path option, and configuration command accepted by the Inform 6.44 compiler.
Each entry is verified against the compiler source code. For detailed
discussion of compiler invocation, see Chapter 11 (Invoking the Compiler)
and Chapter 12 (Compiler Switches and Settings). For the full reference on
`$SETTING=value` memory settings, see Appendix D (Compiler Memory Settings
Reference).

---

## §E.1 Overview

The Inform compiler accepts three families of command-line options, each
distinguished by its prefix character:

| Prefix | Family | Example |
|--------|--------|---------|
| `-` | Dash switches | `-Des` |
| `+` | Path options | `+include_path=/usr/share/inform` |
| `$` | Dollar commands | `$MAX_ABBREVS=80` |

Additionally, the compiler accepts GNU-style double-dash long-form options
(e.g., `--help`, `--opt SETTING=value`) that map to the above families. ICL
(Inform Command Language) files and parenthesized file references
`(filename)` can also contain any of these commands.

### Where Switches Can Be Specified

Switches and settings can come from four sources, listed in order of
increasing precedence:

1. **Compiled-in defaults** — built into the compiler itself (precedence 0).
2. **Header comments** — `!%` lines at the very start of a source file
   (precedence 1). These are read before compilation begins.
3. **Command line** — passed directly to the compiler invocation or through
   an ICL file (precedence 2).
4. **`Switches` directive** — a source-level directive (deprecated; see
   below).

A setting from a higher-precedence source overrides the same setting from a
lower-precedence source.

### The `Switches` Directive (Deprecated)

The `Switches` directive can appear in Inform source code:

```inform6
Switches "dexs";
```

However, this directive is **deprecated** as of Inform 6.44. The compiler
issues an obsolete-usage warning when it is encountered. The directive must
appear before the first `Constant` definition and before the first routine
definition. Certain switches (`-k`, `-r`, `-u`, `-G`) cannot be set through
this directive at all. Use command-line arguments or `!%` header comments
instead.

The `-i` switch causes the compiler to ignore any `Switches` directive found
in the source file.

### Switch Negation

Most boolean (on/off) dash switches can be negated by prefixing them with
`~`. For example:

```
inform -~S game.inf
```

This disables the `-S` (strict error checking) switch. Negation sets the
associated variable to `FALSE` (0) instead of `TRUE` (1).

---

## §E.2 Dash Switches (Lowercase)

Lowercase dash switches control compilation behavior, diagnostic output, and
target version selection. Multiple switches can be combined in a single dash
group (e.g., `-dexs`).

The following table lists all lowercase switches as implemented in the
`switches()` function (`inform.c`). The "Default" column shows the value set
by `reset_switch_settings()`.

| Switch | Values | Variable | Default | Description |
|--------|--------|----------|---------|-------------|
| `-a` | 1–4 | `asm_trace_setting` | 0 (off) | Trace assembly language output. `-a` or `-a1`: basic trace. `-a2`: include hex dumps. `-a3`: include branch optimization info. `-a4`: verbose branch info. |
| `-c` | on/off | `concise_switch` | off | Use more concise error messages. |
| `-d` | 1–2 | `double_space_setting` | 0 (off) | Contract double spaces after full stops in text. `-d` or `-d1`: after periods only. `-d2`: also after `!` and `?` marks. |
| `-e` | on/off | `economy_switch` | off | Economy mode (slower compilation): make use of declared abbreviations to compress text. |
| `-f` | on/off | `frequencies_setting` | 0 (off) | Show how useful each declared abbreviation is (frequencies mode). Only meaningful with `-e`. |
| `-g` | 1–3 | `trace_fns_setting` | 0 (off) | Insert runtime function-call tracing code. `-g` or `-g1`: trace calls to game functions. `-g2`: also trace library functions. `-g3`: also trace veneer functions. |
| `-h` | 0–2 | — | — | Print help and exit. `-h` or `-h0`: general help summary. `-h1`: help on filenames and path options. `-h2`: full list of all switches. |
| `-i` | on/off | `ignore_switches_switch` | off | Ignore the `Switches` directive if found in the source file. |
| `-k` | on/off | `debugfile_switch` | off | Output debugging information to the debug file (typically `gameinfo.dbg`). **Cannot** be set from the `Switches` directive. |
| `-q` | on/off | `obsolete_switch` | off | Keep quiet about obsolete usage warnings (suppress warnings for withdrawn settings, deprecated directives, etc.). |
| `-r` | on/off | `transcript_switch` | off | Record all game text to a transcript file (typically `gametext.txt`). **Cannot** be set from the `Switches` directive. |
| `-s` | on/off | `statistics_switch` | off | Print compilation statistics at the end of a successful compile. |
| `-u` | on/off | `optimise_switch` | off | Work out the most useful text abbreviations (very slow). Also sets `store_the_text` internally. **Cannot** be set from the `Switches` directive. |
| `-v` | 3–8 | `version_number` | 5 | Select Z-machine version for the output story file. See §E.2.1 below. In Glulx mode (`-G`), `-v` instead sets the Glulx version number in `X.Y.Z` format. |
| `-w` | on/off | `nowarnings_switch` | off | Disable all warning messages. |
| `-x` | on/off | `hash_switch` | off | Print a `#` character for every 100 lines compiled (progress indicator). |
| `-z` | on/off | `memory_map_setting` | 0 (off) | Print a memory map of the virtual machine after compilation. |

### §E.2.1 Z-Machine Version Selection (`-v`)

The `-v` switch selects the Z-machine version:

| Switch | Version | Nickname | Notes |
|--------|---------|----------|-------|
| `-v3` | 3 | "Standard" / "ZIP" | 4-character dictionary words; no bold/italic; strict runtime checking disabled by default |
| `-v4` | 4 | "Plus" / "EZIP" | 6-character dictionary words |
| `-v5` | 5 | "Advanced" / "XZIP" | **Default.** 9-character dictionary words; full feature set |
| `-v6` | 6 | Graphical / "YZIP" | Graphical support; extended memory map |
| `-v7` | 7 | Expanded "Advanced" | Extended memory map; 8× packed address scale factor |
| `-v8` | 8 | Expanded "Advanced" | Like v5 but with 8× packed address scale factor for larger files |

When `-v` is used with a version below 5, the compiler automatically disables
strict runtime error checking (`-S`) unless `-S` has been explicitly set.

In Glulx mode (`-G`), the `-v` switch takes a version string in `X.Y.Z`
format (e.g., `-v3.1.2`) rather than a single digit.

---

## §E.3 Dash Switches (Uppercase)

Uppercase dash switches control compilation target, character encoding, error
format, and debugging features.

| Switch | Values | Variable | Default | Description |
|--------|--------|----------|---------|-------------|
| `-B` | on/off | `oddeven_packing_switch` | off | Use big memory model with odd/even packing for large V6/V7 story files. |
| `-C` | 0–9, `u` | `character_set_setting` / `character_set_unicode` | 1 (ISO Latin-1) | Set the source text character set. See §E.3.1 below. |
| `-D` | on/off | `define_DEBUG_switch` | off | Automatically insert `Constant DEBUG;` at the start of compilation. |
| `-E` | 0–2 | `error_format` | platform-dependent | Set the error message format. `-E0`: Archimedes style. `-E1`: Microsoft style. `-E2`: Macintosh MPW style. Without a digit, defaults to `-E1`. |
| `-G` | on/off | `glulx_mode` | off | Compile to Glulx format instead of Z-machine. **Cannot** be set from the `Switches` directive. Must appear **before** any `-v` switch on the command line. |
| `-H` | on/off | `compression_switch` | on | Use Huffman encoding to compress strings in Glulx output. Enabled by default; use `-~H` to disable. |
| `-R` | 0–1 | `riscos_file_type_format` | 0 | [RISC OS only] Set the file type for output files. `-R0`: filetype 060 + version number. `-R1`: official Acorn filetype 11A. Only available when compiled with `ARCHIMEDES` defined. |
| `-S` | on/off | `runtime_error_checking_switch` | on | Compile strict error-checking code at runtime. Enabled by default. Use `-~S` to disable. |
| `-T` | on/off | `throwback_switch` | off | [RISC OS only] Enable throwback of errors in the Desktop Development Environment (DDE). Only available when compiled with `ARC_THROWBACK` defined. |
| `-V` | — | — | — | Print the compiler version and date, then exit immediately. |
| `-W` | 3–99 | `ZCODE_HEADER_EXT_WORDS` | 3 | [Z-machine only] Set the minimum size of the header extension table in words. Accepts a one- or two-digit number (e.g., `-W3`, `-W16`). Also settable via `$ZCODE_HEADER_EXT_WORDS`. |
| `-X` | on/off | `define_INFIX_switch` | off | Compile with INFIX debugging facilities present, enabling interactive debugging in supporting interpreters. |

### §E.3.1 Character Set Selection (`-C`)

The `-C` switch selects the character encoding used for source file
interpretation:

| Switch | Character Set |
|--------|--------------|
| `-C0` | Plain ASCII only |
| `-C1` | ISO 8859-1 (Latin-1) — **default** |
| `-C2` | ISO 8859-2 (Latin-2, Central European) |
| `-C3` | ISO 8859-3 (Latin-3, South European) |
| `-C4` | ISO 8859-4 (Latin-4, North European) |
| `-C5` | ISO 8859-5 (Cyrillic) |
| `-C6` | ISO 8859-6 (Arabic) |
| `-C7` | ISO 8859-7 (Greek) |
| `-C8` | ISO 8859-8 (Hebrew) |
| `-C9` | ISO 8859-9 (Latin-5, Turkish) |
| `-Cu` | UTF-8 (Unicode) |

When `-Cu` is selected, `character_set_unicode` is set to `TRUE` and
`character_set_setting` remains at 1 (Latin-1), because the first block of
Unicode matches ISO 8859-1.

---

## §E.4 Plus-Sign Path Options

Plus-sign options control the directories where the compiler looks for input
files and writes output files. They also set the names of certain special
output files.

### Syntax

| Form | Meaning |
|------|---------|
| `+dir` | Shorthand for `+include_path=dir` — sets the include search path |
| `+name=path` | Set the named path variable to the given value |
| `++dir` | Shorthand for `++include_path=dir` — prepends to the include search path |
| `++name=path` | Prepend the given directory to the existing path (adds as a new alternative tried first) |

Path variable names are case-insensitive (they are lowercased internally
before comparison).

### Path Variables

The following eight path and filename variables are recognized, as
implemented in `set_path_command()` (`inform.c`):

| Variable | Type | Description |
|----------|------|-------------|
| `source_path` | directory path | Directory to search for the main source file specified on the command line. |
| `include_path` | directory path | Directory (or list of directories) to search for files referenced by `Include` directives. This is the default path set by the bare `+dir` syntax. |
| `code_path` | directory path | Directory for the compiled output story file. |
| `icl_path` | directory path | Directory to search for ICL (Inform Command Language) setup files referenced by `(filename)` syntax. |
| `debugging_name` | filename | The name of the debug information output file (used with `-k`). |
| `transcript_name` | filename | The name of the game text transcript file (used with `-r`). |
| `language_name` | filename | The name of the language definition file included by the library (e.g., `English` for `English.h`). |
| `charset_map` | filename | The name of the character set mapping file. |

### Alternative Paths

The four directory path variables (`source_path`, `include_path`,
`code_path`, `icl_path`) can contain multiple alternative directories
separated by the platform's alternative-path character (typically `,` on most
platforms). When searching for a file, the compiler tries each alternative
directory in left-to-right order until the file is found.

Using `++` (double plus) prepends a new directory to the existing list rather
than replacing it. For example:

```
inform ++include_path=/my/extra/libs game.inf
```

This adds `/my/extra/libs` as the first alternative in the include path,
before any previously set directories.

---

## §E.5 Dollar Memory Settings (Summary)

The `$SETTING=value` syntax controls compiler memory allocation and code
generation parameters. Inform 6.44 defines 27 active settings. For full
descriptions, value limits, and platform-specific defaults, see **Appendix D
(Compiler Memory Settings Reference)**.

The following table provides a quick-reference summary. The "Target" column
indicates whether a setting applies to Z-machine only [Z], Glulx only [G],
or both [All].

| Setting | Target | Z Default | G Default | Brief Description |
|---------|--------|-----------|-----------|-------------------|
| `MAX_ABBREVS` | [Z] | 64 | — | Maximum declared abbreviations (max 96) |
| `NUM_ATTR_BYTES` | [G] | 6 | 7 | Bytes for attribute flags |
| `DICT_WORD_SIZE` | [G] | 6 | 9 | Characters per dictionary word |
| `DICT_CHAR_SIZE` | [G] | 1 | 1 | Byte size of one dictionary character (1 or 4) |
| `GRAMMAR_VERSION` | [All] | 1 | 2 | Grammar table format version |
| `GRAMMAR_META_FLAG` | [All] | 0 | 0 | Use action-value ordering for meta actions |
| `MAX_DYNAMIC_STRINGS` | [All] | 32 | 100 | Maximum string substitution variables (max 96 in Z) |
| `HASH_TAB_SIZE` | [All] | 512 | 512 | Size of symbol hash tables |
| `ZCODE_HEADER_EXT_WORDS` | [Z] | 3 | — | Header extension table size in words |
| `ZCODE_HEADER_FLAGS_3` | [Z] | 0 | — | Value for Flags 3 word (Z-Spec 1.1) |
| `ZCODE_FILE_END_PADDING` | [Z] | 1 | — | Pad file to multiple of 512 bytes |
| `ZCODE_LESS_DICT_DATA` | [Z] | 0 | — | Use 2 data bytes per dict word instead of 3 |
| `ZCODE_MAX_INLINE_STRING` | [Z] | 32 | — | Max length for inline string literals in opcodes |
| `ZCODE_COMPACT_GLOBALS` | [Z] | 0 | — | Reuse space from unused globals |
| `INDIV_PROP_START` | [All] | 64 | 256 | First individual property number |
| `MEMORY_MAP_EXTENSION` | [G] | — | 0 | Extra zero bytes appended to memory map |
| `GLULX_OBJECT_EXT_BYTES` | [G] | — | 0 | Extra bytes per object record |
| `MAX_STACK_SIZE` | [G] | — | 4096 | Interpreter stack size in bytes |
| `TRANSCRIPT_FORMAT` | [All] | 0 | 0 | Transcript format (0=classic, 1=prefixed) |
| `WARN_UNUSED_ROUTINES` | [All] | 0 | 0 | Warn about unused routines (1=game, 2=all) |
| `OMIT_UNUSED_ROUTINES` | [All] | 0 | 0 | Omit unused routines from output |
| `STRIP_UNREACHABLE_LABELS` | [All] | 1 | 1 | Skip labels in unreachable code |
| `OMIT_SYMBOL_TABLE` | [All] | 0 | 0 | Omit debug symbol names from output |
| `DICT_IMPLICIT_SINGULAR` | [All] | 0 | 0 | Auto-set `//s` flag if `//p` is absent |
| `DICT_TRUNCATE_FLAG` | [All] | 0 | 0 | Use bit 6 for truncation (not verb) flag |
| `LONG_DICT_FLAG_BUG` | [All] | 1 | 1 | Retain legacy `//p` bug in long dict words |
| `SERIAL` | [All] | 0 | 0 | Override the six-digit serial number |

---

## §E.6 Dollar Special Commands

Beyond `$SETTING=value`, the dollar-sign prefix introduces several special
commands. These are processed by `execute_dollar_command()` (`memory.c`).

| Command | Description |
|---------|-------------|
| `$LIST` | List all current setting values and their sources. |
| `$?SETTING` | Display a brief explanation of what `SETTING` does. |
| `$SETTING=value` | Set a compiler memory setting to the given numeric value. Setting names are case-insensitive. |
| `$!` | List all available trace options with their descriptions. |
| `$!TRACEOPT` | Enable trace option `TRACEOPT` at level 1. |
| `$!TRACEOPT=N` | Enable trace option `TRACEOPT` at level N. |
| `$#SYMBOL=value` | Define `SYMBOL` as a compile-time constant with the given numeric value. Equivalent to `Constant SYMBOL = value;` in source. |
| `$HUGE` | Withdrawn size command. Accepted but ignored (with a warning unless `-q` is set). |
| `$LARGE` | Withdrawn size command. Accepted but ignored (with a warning unless `-q` is set). |
| `$SMALL` | Withdrawn size command. Accepted but ignored (with a warning unless `-q` is set). |

### Symbol Definition (`$#`)

The `$#SYMBOL=value` command defines a compile-time constant that can be
tested in the source using `Ifdef`, `Ifndef`, `Iftrue`, and `Iffalse`
directives. The symbol name must begin with a letter or underscore and
contain only alphanumeric characters and underscores. The value is a decimal
integer; if omitted, it defaults to 0. For example:

```
inform $#BETA_TEST=1 game.inf
```

This is equivalent to writing `Constant BETA_TEST = 1;` at the start of the
source, but the constant is set before any source code is processed.

---

## §E.7 Trace Options (`$!`)

Trace options provide detailed diagnostic output from various compiler
subsystems. They are set using the `$!TRACEOPT` or `$!TRACEOPT=N` syntax.
Higher numeric levels produce more verbose output.

The following table lists all 18 trace options as implemented in
`set_trace_option()` (`memory.c`). The "Synonyms" column lists alternative
names accepted for each option. Trace option names are case-insensitive (they
must be uppercase or are uppercased before matching).

| Option | Synonyms | Levels | Dash Switch | Description |
|--------|----------|--------|-------------|-------------|
| `ACTIONS` | `ACTION` | 1–2 | — | Show all actions. Level 2: also list actions as they are defined. |
| `ASM` | `ASSEMBLY` | 1–4 | `-a` | Trace assembly language output. Level 2: also show hex dumps. Level 3: also show branch optimization info. Level 4: more verbose branch info. |
| `BPATCH` | `BACKPATCH` | 1–2 | — | Show backpatch results. Level 2: also show markers as they are added. |
| `DICT` | `DICTIONARY` | 1–2 | — | Display the dictionary table after compilation. Level 2: also show byte encoding of entries. |
| `EXPR` | `EXPRESSION`, `EXPRESSIONS` | 1–3 | — | Show expression trees during parsing. Level 2: more verbose. Level 3: even more verbose. |
| `FILES` | `FILE` | 1 | — | Show files as they are opened during compilation. |
| `FINDABBREVS` | `FINDABBREV` | 1–2 | — | Show selection decisions during abbreviation optimization. Only meaningful with `-u`. Level 2: also show three-letter-block decisions. |
| `FREQ` | `FREQUENCY`, `FREQUENCIES` | 1+ | `-f` | Show how efficient abbreviations were. Only meaningful with `-e`. |
| `MAP` | — | 1–3 | `-z` | Print memory map of the virtual machine. Level 2: also show percentage of VM occupied by each segment. Level 3: also show byte counts per segment. |
| `MEM` | `MEMORY` | 1+ | — | Show internal memory allocations made by the compiler. |
| `OBJECTS` | `OBJECT`, `OBJS`, `OBJ` | 1+ | — | Display the object table after compilation. |
| `PROPS` | `PROP`, `PROPERTY`, `PROPERTIES` | 1+ | — | Show attributes and properties as they are defined. |
| `RUNTIME` | — | 1–3 | `-g` | Show game function calls at runtime. Level 2: also show library calls (not supported in Glulx). Level 3: also show veneer calls (not supported in Glulx). |
| `STATS` | `STATISTICS`, `STAT` | 1+ | `-s` | Give compilation statistics. |
| `SYMBOLS` | `SYMBOL` | 1–2 | — | Display the symbol table after compilation. Level 2: also show compiler-defined symbols. |
| `SYMDEF` | `SYMBOLDEF` | 1+ | — | Show when symbols are noticed and defined during compilation. |
| `TOKENS` | `TOKEN` | 1–3 | — | Show token lexing during compilation. Level 2: also show token types. Level 3: also show lexical context. |
| `VERBS` | `VERB` | 1+ | — | Display the verb grammar table after compilation. |

Where a trace option has an equivalent dash switch (shown in the "Dash
Switch" column), both forms set the same internal variable. For example,
`$!ASM=2` is exactly equivalent to `-a2`.

For practical debugging techniques using trace options, see Chapter 36
(Debugging and Testing).

---

## §E.8 Double-Dash Long-Form Options

The compiler accepts GNU-style double-dash options as an alternative to the
traditional single-character switch syntax. Each long-form option is
internally converted to its equivalent ICL command before execution.

The following table lists all 11 long-form options as implemented in
`execute_dashdash_command()` (`inform.c`):

| Long Option | Argument | Equivalent | Description |
|-------------|----------|------------|-------------|
| `--help` | — | `-h` | Print the general help summary and exit. |
| `--list` | — | `$LIST` | List all current compiler settings and their values. |
| `--size SIZE` | `HUGE`, `LARGE`, or `SMALL` | `$HUGE` etc. | Withdrawn size setting. Accepted but prints a warning that it is no longer needed. |
| `--opt SETTING=N` | `SETTING=value` | `$SETTING=N` | Set a compiler memory setting. The argument must contain an `=` sign. |
| `--helpopt SETTING` | setting name | `$?SETTING` | Display a brief explanation of the named setting. |
| `--define SYMBOL=N` | `SYMBOL=value` | `$#SYMBOL=N` | Define a compile-time constant symbol. |
| `--path NAME=dir` | `NAME=path` | `+NAME=dir` | Set a path variable. The argument must contain an `=` sign. |
| `--addpath NAME=dir` | `NAME=path` | `++NAME=dir` | Prepend a directory to an existing path variable. The argument must contain an `=` sign. |
| `--config FILE` | filename | `(FILE)` | Read and execute an ICL setup file. |
| `--trace TRACEOPT` | trace option | `$!TRACEOPT` | Set a trace option. Accepts `TRACEOPT` or `TRACEOPT=N`. |
| `--helptrace` | — | `$!` | List all available trace options. |

---

## §E.9 Obsolete Dollar Settings

The compiler recognizes a number of settings from earlier versions of Inform
that are no longer used. These settings are accepted without error, but the
compiler prints a warning message (suppressed by `-q`). No value is actually
changed.

### Withdrawn Inform 5 Settings (10)

The following settings were used by Inform 5 and have been withdrawn:

| Setting |
|---------|
| `BUFFER_LENGTH` |
| `MAX_BANK_SIZE` |
| `BANK_CHUNK_SIZE` |
| `MAX_OLDEPTH` |
| `MAX_ROUTINES` |
| `MAX_GCONSTANTS` |
| `MAX_FORWARD_REFS` |
| `STACK_SIZE` |
| `STACK_LONG_SLOTS` |
| `STACK_SHORT_LENGTH` |

### Withdrawn Inform 6 Settings (31)

The following settings were used by earlier versions of Inform 6 but have
been withdrawn. These limits are now handled dynamically by the compiler:

| Setting | Setting | Setting |
|---------|---------|---------|
| `MAX_QTEXT_SIZE` | `MAX_SYMBOLS` | `SYMBOLS_CHUNK_SIZE` |
| `MAX_OBJECTS` | `MAX_ACTIONS` | `MAX_ADJECTIVES` |
| `MAX_DICT_ENTRIES` | `MAX_STATIC_DATA` | `MAX_PROP_TABLE_SIZE` |
| `MAX_ARRAYS` | `MAX_EXPRESSION_NODES` | `MAX_VERBS` |
| `MAX_VERBSPACE` | `MAX_LABELS` | `MAX_LINESPACE` |
| `MAX_NUM_STATIC_STRINGS` | `MAX_STATIC_STRINGS` | `MAX_ZCODE_SIZE` |
| `MAX_LINK_DATA_SIZE` | `MAX_LOW_STRINGS` | `MAX_TRANSCRIPT_SIZE` |
| `MAX_CLASSES` | `MAX_INCLUSION_DEPTH` | `MAX_SOURCE_FILES` |
| `MAX_INDIV_PROP_TABLE_SIZE` | `MAX_OBJ_PROP_TABLE_SIZE` | `MAX_OBJ_PROP_COUNT` |
| `MAX_LOCAL_VARIABLES` | `MAX_GLOBAL_VARIABLES` | `ALLOC_CHUNK_SIZE` |
| `MAX_UNICODE_CHARS` | | |

---

## §E.10 ICL File Format

An ICL (Inform Command Language) file is a text file containing compiler
commands, one per line. ICL files provide a way to store commonly-used
switch combinations and compile multiple source files in sequence.

### Referencing ICL Files

ICL files can be loaded from the command line using parentheses:

```
inform (setup.icl) game.inf
```

Or with the long-form option:

```
inform --config setup.icl game.inf
```

The compiler searches for ICL files in the directory specified by `icl_path`.

### ICL File Syntax

Each line of an ICL file must be one of the following:

| Line Format | Description |
|-------------|-------------|
| `-switches` | A dash switch list (e.g., `-Des`) |
| `+path_command` | A plus-sign path option (e.g., `+include_path=/opt/inform/lib`) |
| `$command` | A dollar command (e.g., `$MAX_ABBREVS=80`) |
| `(filename)` | Load another ICL file |
| `compile source [output]` | Compile a source file, optionally specifying an output filename |
| `! comment` | A comment (ignored); also accepted after a command on the same line |
| *(empty line)* | Ignored |

The `compile` command is only valid inside ICL files, not on the command line
or in header comments. It triggers a full compilation of the specified
source file.

### Example ICL File

```
! setup.icl — Standard compilation settings
-Des
+include_path=/usr/share/inform/lib
$OMIT_UNUSED_ROUTINES=1
$WARN_UNUSED_ROUTINES=2
compile game.inf game.z5
```

An ICL command is recognized as any line starting with `+`, `-`, `$`, or
`(`. Any other non-empty, non-comment line is expected to be a `compile`
command.

---

## §E.11 Header Comment Switches (`!%`)

Source files can embed ICL commands in special header comments at the very
start of the file. These lines begin with `!%` and are processed before
compilation begins.

### Syntax

```inform6
!% -G
!% +include_path=../lib
!% $MAX_DYNAMIC_STRINGS=64
```

Each `!%` line contains exactly one ICL command (using the same syntax as
command-line arguments or ICL file lines). A regular comment may follow the
command on the same line, preceded by `!`.

### Rules

1. Header comment lines must appear at the **very beginning** of the source
   file, before any other content (including blank lines without `!%`).
2. The compiler stops reading header comments as soon as it encounters a line
   that does not begin with `!%`.
3. All ICL command types are accepted: `-switches`, `+path`, `$settings`,
   and `(filename)`.
4. Header comment settings have precedence 1, which is higher than compiled-in
   defaults (precedence 0) but lower than command-line arguments
   (precedence 2). A command-line setting always overrides a header comment
   setting.
5. Dollar-sign settings from header comments use the header-comment precedence
   level (`HEADCOM_OPTPREC`), while the same settings from the command line
   or ICL files use the command-line precedence level (`CMDLINE_OPTPREC`).

### Example

```inform6
!% -DG
!% +include_path=../inform6lib
!% $DICT_WORD_SIZE=12

! This is a regular comment — not a header comment.
! The compiler has already stopped reading !% lines.

Constant Story "My Game";
Constant Headline "^An interactive fiction.^";

Include "Parser";
Include "VerbLib";
Include "Grammar";
```

In this example, the header comments enable the `DEBUG` constant (`-D`),
select Glulx output (`-G`), set the include path, and increase the
dictionary word size to 12 characters — all before any source code is
processed.

---

*This appendix covers Inform compiler version 6.44. All switch names,
variables, defaults, and behaviors have been verified against the compiler
source code.*
