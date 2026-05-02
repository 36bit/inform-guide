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

# Chapter 14: Error Messages and Diagnostics

The compiler provides a rich set of diagnostic messages to help
programmers identify and fix problems in their source code. This chapter
catalogs the three categories of diagnostic messages — fatal errors,
errors, and warnings — and describes the compiler's debugging and
tracing facilities.

---

## 14.1 Overview

The compiler reports four categories of diagnostic messages during
compilation:

| Category       | Internal style | Compilation continues? | Output produced? |
|----------------|----------------|------------------------|------------------|
| Fatal error    | 0              | No — immediate halt    | No               |
| Error          | 1              | Yes — to find more     | No               |
| Warning        | 2              | Yes                    | Yes              |
| Compiler error | 4              | Yes (usually)          | No               |

(Style 3, "linkage error", existed historically but is no longer used.)

**Fatal errors** represent unrecoverable conditions: the compiler cannot
proceed and terminates immediately with exit code 1.

**Errors** indicate definite problems in the source code — syntax
violations, undefined symbols, type mismatches, and so on. When an error
is encountered the compiler attempts to recover and continue parsing so
that it can report as many errors as possible in a single run. However,
no output file is produced.

**Warnings** flag potential problems that may or may not be actual bugs.
The source code is technically valid, and the compiler produces an output
file, but the programmer should review the flagged constructs. Warnings
can be suppressed with the `-w` switch and re-enabled with `-w-`. (The
compiler still counts suppressed warnings and reports the count at the
end of compilation.)

**Compiler errors** are an internal-only category, prefixed with
`*** Compiler error:`, that indicate a bug in Inform itself rather than
in the source program. They print a "sorry" message inviting the user
to file a bug report.

All three user-facing categories include the source file name and line
number (when available) so that the programmer can locate the problem
quickly. At the end of compilation the compiler prints a summary line
of the form `Compiled with N error(s) and M warning(s)`, including a
suppressed-warning count when applicable.

---

## 14.2 Fatal Errors

Fatal errors represent conditions from which the compiler cannot
recover. When one occurs, compilation stops immediately and no story
file is written.

### 14.2.1 Format

Fatal errors are printed in the form:

```
Fatal error: description
```

The compiler then exits with a non-zero exit code, which can be tested
by build scripts and Makefiles.

### 14.2.2 Common Fatal Errors

| Message | Cause |
|---------|-------|
| `Couldn't open source file "filename"` | The named source file does not exist or is not readable. |
| `Run out of memory allocating N bytes for X` | The compiler has exhausted available memory while allocating an internal table. Increase the relevant `$MAX_*` setting (see §12.5). |
| `Too many errors: giving up` | The error count has reached `MAX_ERRORS` (a compile-time constant of 100). This prevents runaway error cascading. |
| `Too many compiler errors: giving up` | An internal compiler error has recurred 100 times. |

### 14.2.3 Example

```
Fatal error: Couldn't open source file "story.inf"
```

This typically means the filename is misspelled, the file is in a
different directory, or the include path (set with `+include_path=...`)
does not point to the right location. See §11.3 for path configuration.

---

## 14.3 Errors

Errors indicate definite problems in the source code. The compiler
reports the error but attempts to recover and continue parsing so that
multiple errors can be found in a single compilation run.

### 14.3.1 Format

The default error format is:

```
"filename", line N: Error:  description
```

(Note the two spaces after the colon.) When the error is in the main
source file, the filename portion is omitted, leaving just
`line N: Error:  description`. The format can be changed with the `-E`
switch (see §14.5).

### 14.3.2 Error Recovery

After encountering an error, the compiler skips tokens until it reaches
a point where parsing can safely resume — typically the next semicolon,
closing brace, or directive keyword. This recovery is imperfect: a
single real mistake (such as a missing semicolon) can sometimes trigger
a cascade of spurious follow-on errors. As a rule of thumb, fix the
*first* reported error and recompile before tackling later ones.

If the total number of errors reaches the built-in maximum (100,
defined as the compile-time constant `MAX_ERRORS` in `header.h`), the
compiler issues the fatal error `Too many errors: giving up` and halts
to prevent unbounded output.

### 14.3.3 Common Errors

#### Syntax errors

The most common errors come from the `ebf_*` family ("expected but
found"), which produce messages of the form:

```
"game.inf", line 42: Error:  Expected ';' but found <token>
```

For string and character tokens the wording adjusts: `Expected X but
found string "..."`, `Expected X but found char '...'`, or `Expected X
but found dict word '...'`.

#### Symbol clashes

```
"lantern" is a name already in use and may not be used as a <kind>
(<oldkind> "lantern" was defined at "game.inf", line 12)
```

This is produced by `ebf_symbol_error` whenever a name is reused for
something incompatible with its earlier definition.

#### Exceeding VM limits

```
Too many local variables for a routine; max is 15
```

This message comes from `syntax.c` and reflects the Z-machine cap of
15 locals per routine [Z-machine]; Glulx allows more [Glulx].

```
The number of abbreviations has exceeded 96, the limit in Z-code
```
```
The number of abbreviations has exceeded MAX_ABBREVS. Increase MAX_ABBREVS.
```

These come from `error_max_abbreviations` in `errors.c`.

#### Other common errors

| Message | Source | Explanation |
|---------|--------|-------------|
| `Expected X but found Y` | `ebf_error`, `ebf_curtoken_error` | The parser expected a specific token and found something else. |
| `Duplicate definition of label: "name"` | `states.c` | A label with this name already exists in the routine. |
| `Property given twice in the same declaration: "name"` | `objects.c` | An object definition provides the same property more than once. |
| `Evaluating this has no effect: ...` | `expressc.c` | An expression statement is evaluated but its result is discarded and it has no side effects. |
| `Only dynamic strings @(00) to @(NN) may be used ...` | `error_max_dynamic_strings` | The dynamic-string index exceeds `$MAX_DYNAMIC_STRINGS` or, in Z-code, the hard cap of 96. |

---

## 14.4 Warnings

Warnings indicate potential problems that do not prevent compilation.
The output file is still produced, but the programmer should review the
flagged code.

### 14.4.1 Format

```
"filename", line N: Warning:  description
```

(Note the two spaces after the colon — the same preamble as for errors,
just with the `Warning:` keyword.)

### 14.4.2 Suppressing Warnings

All warnings can be suppressed with the `-w` switch:

```
inform -w game.inf
```

The switch sets the internal flag `nowarnings_switch`. Suppressed
warnings are still counted (in `no_suppressed_warnings`) and reported in
the final summary line. Re-enable warnings explicitly with `-w-`. The
related switch `-w` does not affect compiler errors or fatal errors.

The `Obsolete usage:` warnings (issued via `obsolete_warning`) can also
be quieted by the `-q` (`obsolete_switch`) switch independently of `-w`.
Inside files marked as system files (typically library headers),
obsolete-usage warnings are suppressed unconditionally.

Suppressing warnings is useful when compiling legacy code that triggers
many harmless warnings, but in general it is better to fix the
underlying causes.

### 14.4.3 Common Warnings

#### Declared but not used

```
Local variable "temp" declared but not used
```

Issued by `dbnu_warning` when a local, global, attribute, property, or
similar named entity is declared but never referenced. With
`-u` (`OMIT_UNUSED_ROUTINES`) enabled, unreferenced routines are also
flagged with `Routine "name" unused and omitted` (or `unused (not
omitted)` if `-u` is off), produced by `uncalled_routine_warning`.

#### Obsolete usage

```
Obsolete usage: description
```

Issued by `obsolete_warning` when source uses a syntax form that has
been superseded. Examples found in the source:

- `'#a$Act' is now superseded by '##Act'`
- `ignoring 'print_paddr': use 'print (string)' instead`
- `ignoring 'print_addr': use 'print (address)' instead`
- `ignoring 'print_char': use 'print (char)' instead`

These warnings are silently dropped when the offending text is inside a
system file (such as a library header).

#### Assignment in a condition

```
'=' used as condition: '==' intended?
```

An `=` appears where `==` was probably intended inside an `if` test
(`expressp.c`).

#### Other common warnings

| Message | Source | Explanation |
|---------|--------|-------------|
| `Evaluating this has no effect: ...` | `expressc.c` | An expression statement has no observable effect. |
| `Logical expression has no side-effects` | `expressc.c` | A `&&`/`\|\|` expression discards its result and the operands have no side effects. |
| `The 'box' statement has no effect in a version 3 game` | `states.c` | Version 3 Z-machines lack box-quote support. |
| `Without bracketing, the minus sign '-' is ambiguous` | `expressp.c` | Add parentheses to clarify operator precedence. |
| `Bare property name found. "self.prop" intended?` | `expressp.c` | A property name appears as a bare identifier. |
| `Property name in expression is not qualified by object` | `expressp.c` | Same family — a property is referenced without `obj.prop`. |
| `Ignoring spurious leading comma` / `trailing comma` | `expressp.c` | Stray comma in an argument list. |
| `Using '->' to access a --> or table array` | `expressc.c` | Operator/array-kind mismatch (and the converse `'-->' to access a -> or string array`). |
| `In <context>, expected X but found Y "name"` | `symtype_warning` | Soft type mismatch; the symbol is used in a context expecting another kind. |

---

## 14.5 Error Format Options

The `-E` switch controls the format of error and warning messages. This
is useful for integrating the compiler with editors and IDEs that parse
error output to jump to the relevant source line.

### 14.5.1 `-E0` — Archimedes / RISC OS Format

```
"game.inf", line 42: Error:  Expected ';' but found ...
```

The original error format. Includes the filename in double
quotes followed by the line number. For errors in the main source file
the filename is omitted (showing only `line 42: Error:  …`); filenames
are always shown for errors in included files. This is the default
format on Unix/Linux and most other platforms.

When an error originates inside an included file, the preamble appends
a secondary location of the form `("includer", N:C)` showing the file,
line, and column of the surrounding context.

### 14.5.2 `-E1` — Microsoft Format

```
game.inf(42): Error:  Expected ';' but found ...
```

Uses the `file(line)` format recognized by Microsoft Visual Studio and
many Windows-based editors. The filename always appears (even for the
main source file), and the source-extension is appended automatically
when the input filename has no extension. Secondary include locations
are appended after a `|` separator. This is the default format on
Windows.

### 14.5.3 `-E2` — Macintosh MPW Format

```
File "game.inf"; Line 42	# Error:  Expected ';' but found ...
```

Uses the Macintosh Programmer's Workshop format. The separator before
`Error:` is a literal tab. This is the default format on classic Mac
OS.

> **Note:** The default error format is platform-dependent: `-E0` on
> Unix/Linux, `-E1` on Windows, `-E2` on Mac OS. It can always be
> overridden explicitly.

---

## 14.6 The Trace Directive

The `Trace` directive controls diagnostic trace output during
compilation. It is primarily useful for compiler developers and for
diagnosing obscure compilation problems.

### 14.6.1 Basic Usage

```inform6
Trace;        ! same as: Trace assembly 1
Trace N;      ! Trace assembly N
```

A bare `Trace` (or `Trace N` with a number) sets the assembly trace
level. Otherwise the directive takes a keyword naming the aspect to
trace.

### 14.6.2 Trace Aspects

The keywords accepted by `Trace` are:

| Keyword       | Behavior |
|---------------|----------|
| `assembly`    | Set ongoing trace level for assembly output (`on`/`off`/N). Equivalent to the `-a` command-line switch. |
| `expressions` | Set ongoing trace level for expression-tree output (`on`/`off`/N). |
| `tokens`      | Set ongoing trace level for lexer-token output (`on`/`off`/N). |
| `dictionary`  | Immediately display the current dictionary table. |
| `symbols`     | Immediately display the current symbol table. |
| `objects`     | Immediately display the current object table. |
| `verbs`       | Immediately display the current verb/grammar table. |
| `lines`       | Reserved keyword; not implemented. |
| `linker`      | Reserved keyword; no longer implemented. |

The first four "table" keywords (`dictionary`, `symbols`, `objects`,
`verbs`) print a one-shot dump at the point of the directive; they do
not affect ongoing tracing. The three "level" keywords (`assembly`,
`expressions`, `tokens`) take an optional `on`, `off`, or numeric level
following the keyword (default 1).

### 14.6.3 Command-Line Equivalents

Ongoing trace settings can also be selected at the command line:

| Switch | Variable | Effect |
|--------|----------|--------|
| `-a`, `-a1`..`-a4` | `asm_trace_setting` | Initial value of `asm_trace_level`. |

Token and expression initial settings are configurable through the
`$!TOKENS` and `$!EXPR` ICL settings rather than dedicated short
switches.

### 14.6.4 Example

To diagnose how the compiler parses a particular expression:

```inform6
Trace expressions on;
x = (a + b) * c;
Trace expressions off;
```

The compiler will print the expression tree it builds for the
assignment, showing operator precedence and operand types. This output
goes to `stdout` and does not appear in the compiled game.

---

## 14.7 Debug File Output

The `-k` switch instructs the compiler to produce a debug information
file alongside the story file. This file maps addresses in the compiled
output back to source-code locations, enabling source-level debugging.

### 14.7.1 File Format

The debug file is named `gameinfo.dbg` (where `gameinfo` matches the
story file's base name) and uses an XML-based format. It contains:

- **Source file table** — paths to all source files included during
  compilation, with checksums for change detection.
- **Routine records** — the name, source location, and compiled address
  of every routine.
- **Global variable records** — names and storage addresses of all
  global variables.
- **Array records** — names, addresses, and extents of all arrays.
- **Object records** — names and object numbers for all game objects.
- **Map records** — mappings between story-file addresses and source
  file/line numbers.

### 14.7.2 Usage

```
inform -k game.inf
```

This produces both `game.z5` (or `game.ulx`) and `game.dbg`. The debug
file can then be loaded by a debugger tool alongside the story file.

### 14.7.3 Consumers

The debug information file is used by:

- **Inform 7's debugging tools** — the Inform 7 IDE uses it to map
  generated I6 code back to I7 source text.
- **Third-party debuggers** — tools that provide source-level debugging
  for interactive fiction, such as those integrated into some IDE
  front-ends.

The debug file format is documented in the Inform Technical Manual and
the compiler source.

---

## 14.8 Infix Debugger Support

The Infix debugger is an interactive debugging system that is compiled
directly into the story file. It allows the programmer to inspect and
manipulate the game world at runtime.

### 14.8.1 Enabling Infix

Infix is enabled with the `-X` switch or by defining the `INFIX`
constant:

```
inform -X game.inf
```

or in source:

```inform6
Constant INFIX;
```

When `INFIX` is defined, the compiler automatically sets `DEBUG` as
well. The Infix support code is drawn from `infix.h` in the standard
library and adds approximately 1200 lines of library code to the
compiled game.

### 14.8.2 Infix Commands

At runtime, Infix commands are entered at the game prompt prefixed with
`;` (semicolon). The following commands are available:

| Command | Description |
|---------|-------------|
| `; expression` | Evaluate an expression and print the result. |
| `; showobj obj` | Display all properties and attributes of *obj*. |
| `; showverb "verb"` | Show the grammar table for *verb*. |
| `; watch obj` | Monitor changes to *obj* — report when properties or attributes change. |
| `; unwatch obj` | Stop monitoring *obj*. |
| `; inventory` | List all watched objects. |
| `; give obj attr` | Set attribute *attr* on *obj*. |
| `; give obj ~attr` | Clear attribute *attr* on *obj*. |
| `; move obj to dest` | Move *obj* to *dest* in the object tree. |
| `; remove obj` | Remove *obj* from the object tree. |

### 14.8.3 Example Session

```
West of House
You are standing in an open field west of a white house.

>; showobj white_house
Object "white house" (23) in "West of House"
  has  container open static
  with name 'white' 'house' 'building',
       description "The house is a beautiful colonial house.",
       ...

>; give white_house locked
[Setting attribute "locked" of "white house"]

>; 2 + 3
5
```

### 14.8.4 Limitations

- Infix significantly increases the size of the compiled game. It is
  intended for development only, not for released games.
- Infix is primarily designed for Z-machine games [Z-machine]. Glulx
  support is partial [Glulx].
- Because Infix compiles symbol-table information into the story file,
  it may push small games over Z-machine size limits.

---

## 14.9 Runtime Error Messages

In addition to compile-time diagnostics, the compiler can insert runtime
checks into the generated code. These checks catch common programming
errors during play.

### 14.9.1 Enabling Runtime Checks

Runtime checking is controlled by the `-S` switch (strict mode) or
equivalently by defining `STRICT_MODE`:

```
inform -S game.inf
```

Strict mode is enabled by default. It can be disabled with `-~S` when
maximum performance is needed, but this is not recommended during
development.

### 14.9.2 What Is Checked

With strict mode enabled, the compiler inserts checks for:

- **Array bounds violations** — accessing an array element outside its
  declared range.
- **Invalid object references** — reading properties or testing
  attributes of object number 0 (nothing) or an out-of-range object.
- **Division by zero** — integer division or modulo with a zero divisor.
- **Invalid object-tree operations** — attempting to `move` an object to
  itself, or other illegal tree manipulations.
- **Stack underflow** — popping more values than have been pushed
  (primarily a concern for hand-written assembly).

### 14.9.3 Runtime Error Format

When a runtime error occurs, the veneer routine `RT__Err` is called. It
prints a message of the form:

```
[** Programming error: description **]
```

The game continues running after most runtime errors, though its state
may be inconsistent.

### 14.9.4 Runtime Error Codes

The `RT__Err` routine uses numeric error codes internally. The following
table lists the most important codes and their meanings:

| Code | Description |
|------|-------------|
| 1    | `create` message called with more than 3 arguments |
| 2    | Test `in`/`notin` on invalid object |
| 3    | Test `has`/`hasnt` on invalid object |
| 4    | Find `parent` of invalid object |
| 5    | Find `eldest` of invalid object |
| 6    | Find `child` of invalid object |
| 7    | Find `younger` of invalid object |
| 8    | Find `sibling` of invalid object |
| 9    | Find `children` of invalid object |
| 10   | Find `youngest` of invalid object |
| 11   | Find `elder` of invalid object |
| 12   | Use `objectloop` with invalid object |
| 13   | `}` at end of `objectloop` with invalid object |
| 14   | `give` attribute to invalid object |
| 15   | `remove` invalid object |
| 16   | `move` with invalid source object |
| 17   | `move` with invalid destination object |
| 18   | `move` would create a loop in the object tree |
| 19   | `give`/`has`/`hasnt` with non-attribute value |
| 20   | Division by zero |
| 21   | Read `.&` (property address) of invalid object |
| 22   | Read `.#` (property length) of invalid object |
| 23   | Read `.` (property value) of invalid object |
| 24   | Read outside memory bounds using `->` |
| 25   | Read outside memory bounds using `-->` |
| 26   | Write outside memory bounds using `->` |
| 27   | Write outside memory bounds using `-->` |
| 28   | Read past end of `->` array |
| 29   | Read past end of `-->` array |
| 30   | Write past end of `->` array |
| 31   | Write past end of `-->` array |
| 32   | `objectloop` broken because object was moved during traversal |
| 33   | `print (char)` with invalid character code |
| 34   | `print (address)` on non-string address |
| 35   | `print (string)` on non-string value |
| 36   | `print (object)` on non-object value |
| 37   | Glulx `print_to_array` called with only one argument [Glulx] |
| 40   | Invalid dynamic string variable number [Glulx] |

String-based error codes (not shown as numeric codes above) are also
used for property access errors: the veneer calls `RT__Err` with a
string such as `"read"`, `"write to"`, or `"send message"` to report
property-related errors like accessing a non-existent individual
property.

### 14.9.5 Example

The following code triggers a runtime error:

```inform6
[ TestBug x;
    x = nothing;
    print (the) x;    ! Runtime error: printing nothing as object
];
```

At runtime:

```
[** Programming error: tried to print (the) of nothing **]
```

### 14.9.6 Disabling Individual Checks

There is no mechanism to disable individual runtime checks — strict mode
is all-or-nothing. For production releases, some authors disable strict
mode entirely with `-~S` to eliminate the size and performance overhead
of the checks. This is safe only after thorough testing.

---

## 14.10 Best Practices

1. **Fix errors from the top.** The first error is the most reliable;
   later errors may be cascading artefacts. Fix the first error and
   recompile before investigating subsequent messages.

2. **Don't suppress warnings carelessly.** The `-w` switch hides all
   warnings. Instead, fix the underlying code. Unused variables and
   unreachable code often indicate logic errors.

3. **Use strict mode during development.** The `-S` switch (on by
   default) catches many bugs that would otherwise manifest as bizarre
   in-game behavior. Only disable it for final release builds after
   thorough testing.

4. **Use the debug file for complex projects.** The `-k` switch
   produces a debug information file that source-level debuggers can
   use to step through compiled code.

5. **Set `-E1` for IDE integration.** If your editor can parse compiler
   output and jump to error locations, the Microsoft-format `-E1`
   option provides the widest compatibility.

6. **Increase limits rather than work around them.** If you hit a
   "too many" or "exceeded limit" error, use `$MAX_*` memory settings
   to raise the limit rather than restructuring your code to fit
   arbitrary constraints. See §12.5 for memory settings.

7. **Consider Glulx for large projects.** Many Z-machine limits on
   attributes, properties, objects, and code size are dramatically
   higher or absent in Glulx [Glulx]. Switching targets with `-G` can
   eliminate entire categories of limit errors.

---

*See also:* §11 (Invoking the Compiler), §12 (Compiler Switches and
Settings), §13 (Compilation Model).
