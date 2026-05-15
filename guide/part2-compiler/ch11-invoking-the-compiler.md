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

# Chapter 11: Invoking the Compiler

The compiler translates source files into
story files for either the Z-machine or Glulx virtual machine. This
chapter describes how to invoke the compiler from the command line, how it
locates source and output files, and how to control its behavior through
switches, path settings, and configuration files.

---

## 11.1 Overview

A typical invocation of the compiler looks like this:

```
inform [options...] <source> [<output>]
```

The compiler reads the source file, processes any `Include` directives to
pull in additional files, and produces a single output story file. By
default, the compiler targets the Z-machine (version 5); the `-G` switch
selects Glulx instead.

A minimal compilation requires only a source filename:

```
inform adventure.inf
```

This compiles `adventure.inf` and produces `adventure.z5` (Z-machine
version 5) in the current directory. To compile for Glulx:

```
inform -G adventure.inf
```

This produces `adventure.ulx` instead.

---

## 11.2 Command-Line Syntax

The general form of the command line is:

```
inform [options...] <file1> [<file2>]
```

`<file1>` is the source file to compile. `<file2>`, if given,
specifies the output filename exactly as written — the compiler will not
alter it or add an extension. Options may appear in any order, and may
be interleaved with the filename arguments.

Options come in four styles, each identified by its leading character:

| Prefix | Style | Example |
| ------ | ----- | ------- |
| `-`    | Switches (one or two letters) | `-v8 -s -D` |
| `+`    | Path settings | `+include_path=mylib` |
| `$`    | Compiler settings | `$DICT_WORD_SIZE=12` |
| `(…)`  | ICL file inclusion | `(setup.icl)` |

Multiple single-letter switches can be combined after a single dash:

```
inform -dexs adventure.inf
```

This enables the `-d`, `-e`, `-x`, and `-s` switches simultaneously.

### 11.2.1 Long Options

As of compiler 6.35, Unix-style long options are also accepted on the
command line. Each long option begins with `--` and takes zero or one
argument:

| Long option | Equivalent | Description |
| ----------- | ---------- | ----------- |
| `--help` | `-h` | Print help information |
| `--list` | `$LIST` | List current compiler settings |
| `--opt SETTING=number` | `$SETTING=number` | Set a compiler setting |
| `--define SYMBOL=number` | `$#SYMBOL=number` | Define a constant |
| `--helpopt SETTING` | `$?SETTING` | Explain a setting |
| `--path PATH=dir` | `+PATH=dir` | Set a path |
| `--addpath PATH=dir` | `++PATH=dir` | Add to a path |
| `--config file.icl` | `(file.icl)` | Read an ICL file |
| `--trace TRACEOPT` | `$!TRACEOPT` | Set a trace option |
| `--trace TRACEOPT=N` | `$!TRACEOPT=N` | Set trace level |
| `--helptrace` | `$!` | List all trace options |

Long options are only available on the command line; they cannot appear in
ICL files or `!%` header comments.

---

## 11.3 Source Files and Include Paths

### 11.3.1 Source File Conventions

The compiler reads a single top-level source file specified on the command
line. By convention, source files use the extension `.inf` or
`.i6`, and included header files use `.h`, though these conventions vary
by platform. If the given filename has no extension, the compiler appends
the default source extension (`.inf`) before opening the file.

### 11.3.2 Include Search Order

Additional source files are brought in with the `Include` directive
(§10.9). Where the compiler looks for the file depends on the form of
the directive:

- If the filename already contains a directory separator, it is used
  as-is and no search path is applied.
- For `Include ">filename"`, the compiler looks **only** in the
  directory of the file that issued the `Include` (the `>` form is
  described in the next subsection).
- For an ordinary `Include "filename"`, the compiler searches the
  directories listed in the **include path**, in the order given (left
  to right). If the include path is unset (empty), the filename is
  used relative to the current working directory.

### 11.3.3 The `>` Prefix

The `Include` directive supports a special `>` prefix that restricts the
search to the directory of the including file:

```inform6
Include ">prologue.h";
```

This searches only in the directory of the file containing this
directive. It is useful when a source file needs to include a companion
file that sits alongside it, regardless of how the include path is
configured.

### 11.3.4 Setting Include Paths

Include paths are set with `+` options:

```
inform +include_path=i6lib adventure.inf
```

Multiple directories can be specified as a comma-separated list:

```
inform +include_path=i6lib,contrib,extras adventure.inf
```

To add a directory to the existing path rather than replacing it, use a
double `+`:

```
inform ++include_path=mylib adventure.inf
```

The newly added paths are searched first, before any previously
configured paths.

### 11.3.5 Other Path Variables

The compiler maintains several path variables, each governing where a
particular kind of file is found or created:

| Path variable | Purpose |
| ------------- | ------- |
| `source_path` | Directory for the top-level source file |
| `include_path` | Directories for included files |
| `code_path` | Directory for the output story file |
| `icl_path` | Directory for ICL command files |

Each can be set with `+path_name=value`. For example:

```
inform +code_path=build adventure.inf
```

If a path variable is unset (empty), the current working directory is
used.

---

## 11.4 Output File Naming

### 11.4.1 Default Behavior

When only one filename is given on the command line, the compiler derives
the output filename from the source filename by stripping any directory
path and extension, then appending the appropriate output extension.

> **[Z-machine]** The output extension depends on the target version:
>
> | Version | Extension |
> | ------- | --------- |
> | 3       | `.z3`     |
> | 4       | `.z4`     |
> | 5 (default) | `.z5` |
> | 6       | `.z6`     |
> | 7       | `.z7`     |
> | 8       | `.z8`     |

> **[Glulx]** The output extension is `.ulx`.

For example:

```
inform -v8 adventure.inf       → adventure.z8
inform -G  adventure.inf       → adventure.ulx
```

If `code_path` is set, the output file is placed in that directory rather
than the current directory.

### 11.4.2 Explicit Output Filename

When a second filename is given on the command line, it is used as the
output filename exactly as specified — no extension is added or
modified:

```
inform adventure.inf release/story.z5
```

This writes the compiled story to `release/story.z5` regardless of the
target version.

---

## 11.5 ICL (Inform Command Language) Files

ICL files provide a way to store frequently used compiler options in a
reusable file rather than typing them on the command line each time.

### 11.5.1 ICL File Format

An ICL file is a plain text file (conventionally with the `.icl`
extension) containing one command per line. Each line begins with a
character that identifies its type:

| Prefix | Meaning | Example |
| ------ | ------- | ------- |
| `-`    | Switch option | `-v8` |
| `+`    | Path setting | `+include_path=i6lib` |
| `$`    | Compiler setting | `$DICT_WORD_SIZE=12` |
| `!`    | Comment (ignored) | `! This is a comment` |
| `(`…`)` | Nested ICL file | `(other.icl)` |

Lines that do not begin with one of these characters are treated as
errors, with one exception: the `compile` command can appear in an ICL
file to trigger compilation of a specified source file (this is rarely
used in practice).

### 11.5.2 Loading an ICL File

To load an ICL file from the command line, enclose its name in
parentheses:

```
inform (setup.icl) adventure.inf
```

Or use the long-option form:

```
inform --config setup.icl adventure.inf
```

### 11.5.3 Example ICL File

A typical ICL file for a Glulx project might contain:

```
! glulx-setup.icl — standard options for our Glulx project
-G
-D
-s
+include_path=i6lib,extensions
$DICT_WORD_SIZE=12
```

---

## 11.6 The `!%` Header Comment Convention

Since compiler version 6.30, compilation options can be embedded directly
in the source file using special header comments. This is convenient
because the file carries its own build configuration.

### 11.6.1 Syntax

Lines at the very beginning of a source file that start with `!%` are
parsed as ICL commands. The `!%` prefix is stripped and the remainder
of the line is processed exactly as if it appeared in an ICL file or on
the command line:

```inform6
!% -G
!% -s
!% +include_path=i6lib
!% $MAX_DYNAMIC_STRINGS=200

Include "Parser";
Include "VerbLib";

[ Initialise;
  location = Kitchen;
];

Object Kitchen "Kitchen"
  with description "It's the kitchen.",
  has light;

Include "Grammar";
```

### 11.6.2 Placement Rules

The `!%` lines must appear at the very start of the source file. The
compiler reads lines sequentially and stops scanning for `!%` commands as
soon as it encounters a line that does not begin with `!%`. This means:

- Multiple `!%` lines are permitted; they are processed in order.
- Any non-`!%` line (including blank lines and regular `!` comments)
  terminates the header scanning. Subsequent `!%` lines in the file are
  treated as ordinary comments and have no effect on compilation.

### 11.6.3 Character Encoding

The `!%` header section is parsed before the source character encoding is
determined — indeed, the header may itself specify the encoding (for
example, `!% -Cu` to select UTF-8). Therefore, `!%` lines must contain
only plain ASCII characters.

---

## 11.7 Option Precedence

### 11.7.1 Processing Order

Options are applied to the compiler's settings in the following order:

1. **Built-in defaults** — the compiler's internal default settings
   (Z-machine version 5, no debug flags, etc.).
2. **Command-line arguments** — processed left to right, including any
   ICL files loaded via `(file.icl)` or `--config`.
3. **`!%` header comments** — options embedded at the top of the source
   file are applied last, just before compilation begins.

### 11.7.2 Overriding Behavior

For ordinary single- and two-letter switches (e.g. `-v8`, `-D`, `-S`),
later applications simply overwrite the previous value, so the last
setting in processing order wins. Within the command line itself,
arguments are processed strictly left to right:

```
inform -v5 -v8 adventure.inf
```

This compiles to version 8, because `-v8` appears after `-v5`.

Because `!%` header comments are applied after the command line, an
option set in the header overrides the same switch given on the command
line. For example, if `adventure.inf` begins with `!% -v5`, and you run:

```
inform -v8 adventure.inf
```

This produces a version-5 story file because the `!% -v5` header in the
source file is processed after the command-line `-v8`, overriding it.

Compiler settings introduced with `$` (such as `$DICT_WORD_SIZE=12`)
behave differently. Each setting carries a precedence value that records
where it came from, and a lower-precedence source can never overwrite a
value already set by a higher-precedence source. The command line has
higher precedence than `!%` header comments, so for `$` settings the
direction of override is reversed: a `$` setting given on the command
line takes priority over the same setting in an `!%` header.

### 11.7.3 The `-i` Switch

The `-i` (ignore) switch causes the compiler to skip any `Switches`
directive found inside the source file. It does not affect `!%` header
comments or command-line options. This is useful when a source file
contains a `Switches` directive that you want to override without editing
the file.

---

## 11.8 Compiler Output and Feedback

### 11.8.1 Banner

When the compiler starts, it prints a banner line identifying itself:

```
Inform 6.44 for Unix (11th September 2025)
```

The banner includes the version number, the platform, and the release
date. After a successful compilation, it prints a summary:

```
Inform 6.44 for Unix (11th September 2025)
In:  1 source code files                53 syntactic lines
    53 textual lines                  2120 characters (ISO 8859-1 Latin1)
Allocated:
   120 symbols                      204800 bytes of memory
Out:   Version 5 "Advanced" story.z5 1.870225 (3K long):
```

### 11.8.2 Statistics (`-s`)

The `-s` switch produces a detailed statistical summary of the compiled
story file. This is invaluable for understanding how large your game is
and where resources are being consumed:

```
inform -s adventure.inf
```

The output includes counts and sizes for all major areas of the story
file: objects, properties, dictionary entries, action routines, grammar
entries, code size, string space, and more. Each line shows how much of
the available resource has been used, making it easy to see if you are
approaching any limits.

> **[Z-machine]** Statistics include Z-machine-specific areas such as the
> abbreviation table, alphabet table, and the breakdown of high memory
> into readable, code, and string areas.

> **[Glulx]** Statistics include Glulx-specific information such as the
> amount of RAM vs. ROM, string compression ratios (when Huffman encoding
> is enabled with `-H`), and the stack size allocation.

### 11.8.3 Memory Map (`-z`)

The `-z` switch (or the `$!MAP` trace option) prints a memory map showing
the layout of the story file in the virtual machine's address space. This
can be useful for advanced debugging or for understanding the structure of
the compiled output.

### 11.8.4 Progress Indicator (`-x`)

The `-x` switch prints a `#` character for every 100 lines of source code
compiled. This provides a simple progress indicator for large
compilations.

### 11.8.5 Concise Mode (`-c`)

The `-c` switch produces more concise error messages, omitting the source
context that is normally shown around each error. This is useful when
piping compiler output to tools that parse error messages.

### 11.8.6 Error Message Formats (`-E`)

The format of error messages can be adjusted with the `-E` switch to suit
different development environments:

| Switch | Format |
| ------ | ------ |
| `-E0`  | Archimedes-style (default) |
| `-E1`  | Microsoft-style |
| `-E2`  | Macintosh MPW-style |

### 11.8.7 Warning Suppression (`-w`)

The `-w` switch suppresses all warning messages. The compiler still
reports errors, but non-fatal warnings (such as unused variables or
deprecated usage) are silenced.

### 11.8.8 Version Information (`-V`)

The `-V` switch causes the compiler to exit immediately after the banner
has been printed, without compiling anything. Since the banner already
includes the version number, platform, and release date (see §11.8.1),
this serves as a "print version and quit" command.

---

## 11.9 Commonly Used Switch Reference

The following table summarises the most frequently used switches. A full
list is available via `inform -h2`.

| Switch | Description |
| ------ | ----------- |
| `-c`   | Concise error messages |
| `-d`   | Contract double spaces after full stops in text |
| `-e`   | Economy mode: use declared abbreviations |
| `-h`   | Print help information (`-h1` filenames, `-h2` switches) |
| `-i`   | Ignore in-file `Switches` directive |
| `-k`   | Output debugging information to `gameinfo.dbg` |
| `-q`   | Quiet about obsolete usages |
| `-r`   | Record all text to `gametext.txt` |
| `-s`   | Print compilation statistics |
| `-u`   | Work out most useful abbreviations (very slow) |
| `-v3`…`-v8` | Set Z-machine version (default `-v5`) |
| `-w`   | Suppress warning messages |
| `-x`   | Print `#` for every 100 lines compiled |
| `-z`   | Print memory map of the virtual machine |
| `-B`   | Big memory model (V6/V7 only) |
| `-C0`…`-C9`, `-Cu` | Select source character set |
| `-D`   | Automatically define `DEBUG` constant |
| `-E0`…`-E2` | Error message format |
| `-G`   | Compile to Glulx format |
| `-H`   | Huffman string compression (Glulx, on by default) |
| `-S`   | Strict runtime error checking (on by default) |
| `-V`   | Print version and exit |
| `-W`*n* | Set header extension table size (Z-code) |
| `-X`   | Compile with INFIX debugging (Z-machine only) |

Switches can be negated with a tilde prefix. For example, `-~S` turns
off strict error checking, and `-~H` disables Huffman compression in
Glulx mode.

---

## 11.10 Putting It All Together

Here are several complete invocation examples showing typical use cases.

**Basic Z-machine compilation (version 5, the default):**

```
inform adventure.inf
```

**Compile to Z-machine version 8 with statistics:**

```
inform -v8 -s adventure.inf
```

**Compile to Glulx with debugging enabled:**

```
inform -G -D adventure.inf
```

**Set a custom include path and compile:**

```
inform +include_path=i6lib,contrib adventure.inf
```

**Use an ICL configuration file:**

```
inform (project.icl) adventure.inf
```

**Override output filename explicitly:**

```
inform -G adventure.inf builds/latest.ulx
```

**Embed all options in the source file using `!%`:**

```inform6
!% -G
!% -D
!% -s
!% +include_path=i6lib

Include "Parser";
Include "VerbLib";
! ... game source ...
Include "Grammar";
```

Then compile with just:

```
inform adventure.inf
```

**Override an in-file `!%` `$` setting from the command line:**

Because `$` compiler settings on the command line have higher precedence
than those in `!%` header comments (§11.7.2), a command-line `$` setting
overrides the same setting embedded in the source. For example, if a
Glulx project's source contains `!% $DICT_WORD_SIZE=9`, you can still
build with a 12-character dictionary by writing:

```
inform -G $DICT_WORD_SIZE=12 adventure.inf
```

(The `$DICT_WORD_SIZE` setting is meaningful only for Glulx targets; in
Z-code it is ignored, and specifying any value other than 6 is currently
a fatal error — see §15.7.1.)

Note that this reversal applies *only* to `$` settings. For ordinary
switches such as `-v8`, the `!%` header is applied last and so always
wins over the command line; switches embedded in `!%` cannot be
overridden from the command line.

**Use long options (6.35+):**

```
inform -G --path include_path=i6lib --opt DICT_WORD_SIZE=12 --trace STATS adventure.inf
```
