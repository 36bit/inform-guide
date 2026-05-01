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

The compiler reports three categories of diagnostic messages during
compilation:

| Category    | Severity | Compilation continues? | Output produced? |
|-------------|----------|------------------------|------------------|
| Fatal error | Highest  | No — immediate halt    | No               |
| Error       | Medium   | Yes — to find more     | No               |
| Warning     | Lowest   | Yes                    | Yes              |

**Fatal errors** represent unrecoverable conditions: the compiler cannot
proceed and terminates immediately with a non-zero exit code.

**Errors** indicate definite problems in the source code — syntax
violations, undefined symbols, type mismatches, and so on. When an error
is encountered the compiler attempts to recover and continue parsing so
that it can report as many errors as possible in a single run. However,
no output file is produced.

**Warnings** flag potential problems that may or may not be actual bugs.
The source code is technically valid, and the compiler produces an output
file, but the programmer should review the flagged constructs. Warnings
can be suppressed with the `-w` switch.

All three categories include the source file name and line number (when
available) so that the programmer can locate the problem quickly.

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
| `Out of memory` | The compiler has exhausted available memory. Increase the relevant `$MAX_*` setting (see §12.5). |
| `Too many errors` | The error count has exceeded the configurable maximum (default 100). This prevents runaway error cascading. The limit can be changed with `$MAX_ERRORS`. |
| `Exceeded internal limit: description` | A hard-coded compiler table has overflowed. The message names the table and its current size. Use the corresponding `$` memory setting to raise the limit. |
| `I/O failure: couldn't write to output file` | The compiler cannot write the story file — typically a permissions or disk-space problem. |
| `This program is a Z-machine program, not Glulx` | A source file compiled with `-G` uses a feature that requires the opposite target, or vice versa. |

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
"filename", line N: Error: description
```

The format can be changed with the `-E` switch (see §14.5).

### 14.3.2 Error Recovery

After encountering an error, the compiler skips tokens until it reaches
a point where parsing can safely resume — typically the next semicolon,
closing brace, or directive keyword. This recovery is imperfect: a
single real mistake (such as a missing semicolon) can sometimes trigger
a cascade of spurious follow-on errors. As a rule of thumb, fix the
*first* reported error and recompile before tackling later ones.

If the total number of errors reaches the configurable maximum (default
100, set with `$MAX_ERRORS`), the compiler issues a fatal error and
halts to prevent unbounded output.

### 14.3.3 Common Errors

#### Syntax errors

```
"game.inf", line 42: Error: Expected ';'
```

A semicolon is missing at the end of a statement or directive. This is
by far the most common error.

```
"game.inf", line 18: Error: Unknown directive word "Obiect"
```

A top-level directive is misspelled — here `Object` was typed as
`Obiect`.

#### Undefined symbols

```
"game.inf", line 57: Error: Undefined symbol "lantern"
```

The identifier `lantern` has not been declared. Check spelling, and
ensure that the file defining the symbol is `Include`d before the point
of use.

#### Exceeding VM limits

```
"game.inf", line 200: Error: Too many attributes defined
```

The Z-machine allows a maximum of 48 attributes [Z-machine] while Glulx
raises this to a configurable higher limit [Glulx]. If you hit this
limit on the Z-machine, consider combining related attributes or
switching to Glulx with the `-G` switch.

```
"game.inf", line 310: Error: Too many common properties defined
```

Similarly, the Z-machine limits common properties to 63 [Z-machine].
Glulx permits many more [Glulx].

#### Type checking (Inform 6.36+)

Starting with Inform 6.36, the compiler performs optional type checking
when strict mode is enabled:

```
"game.inf", line 85: Error: Expression has wrong type
```

This indicates that a value is being used in a context that expects a
different type — for example, using a string where a routine address is
required.

#### Other common errors

| Message | Explanation |
|---------|-------------|
| `Expected 'something'` | The parser expected a specific token (keyword, bracket, operator) that was not found. |
| `Duplicate definition of "name"` | A symbol with this name already exists in the same scope. |
| `Property given twice in the same list` | An object definition provides the same property more than once. |
| `Illegal object tree operation` | An attempt to `move` an object in a way that would create a cycle. |
| `Routine contains too many local variables` | The Z-machine allows at most 15 locals per routine [Z-machine]; Glulx allows more [Glulx]. |
| `String too long` | A quoted string exceeds the compiler's internal buffer size. Increase `$MAX_STATIC_STRINGS` if needed. |
| `Include file not found` | An `Include` directive names a file that cannot be located on any search path. |

---

## 14.4 Warnings

Warnings indicate potential problems that do not prevent compilation.
The output file is still produced, but the programmer should review the
flagged code.

### 14.4.1 Format

```
"filename", line N: Warning: description
```

### 14.4.2 Suppressing Warnings

All warnings can be suppressed with the `-w` switch:

```
inform -w game.inf
```

This is useful when compiling legacy code that triggers many harmless
warnings, but in general it is better to fix the underlying causes.

### 14.4.3 Common Warnings

#### Unreachable code

```
"game.inf", line 90: Warning: Unreachable code
```

Code appears after a `return`, `rtrue`, `rfalse`, `jump`, or `quit`
statement where it can never be executed:

```inform6
[ MyRoutine;
    return 42;
    print "This line is never reached.^";   ! Warning here
];
```

#### Unused variables

```
"game.inf", line 12: Warning: Variable 'temp' not used
```

A local variable is declared but never referenced in the routine body.
Either remove the declaration or use the variable:

```inform6
[ MyRoutine temp;
    ! 'temp' is declared but the routine never mentions it
    print "Hello!^";
];
```

#### Obsolete usage

```
"game.inf", line 5: Warning: Obsolete usage: description
```

The source uses a syntax form that is deprecated and may be removed in
future compiler versions. The warning message describes the preferred
replacement.

#### Value overflow

```
"game.inf", line 77: Warning: Byte value overflow
```

A value larger than 255 is being stored in a byte array (`->`) or
a `byte` table, which will silently truncate to the low 8 bits:

```inform6
Array counts -> 10;

[ init;
    counts->0 = 300;   ! Warning: 300 > 255, stored as 44
];
```

#### Other common warnings

| Message | Explanation |
|---------|-------------|
| `Value assigned but never used` | A variable is written to but never subsequently read. |
| `Assignment in condition` | An `=` appears where `==` was probably intended inside an `if` test. |
| `This statement has no effect` | An expression is evaluated but its result is discarded and it has no side effects. |
| `Object "name" has no parent` | An object is defined without being placed in the object tree, which may be intentional but is often a mistake. |

---

## 14.5 Error Format Options

The `-E` switch controls the format of error and warning messages. This
is useful for integrating the compiler with editors and IDEs that parse
error output to jump to the relevant source line.

### 14.5.1 `-E0` — Archimedes / RISC OS Format

```
"game.inf", line 42: Error: Expected ';'
```

The original error format. Includes the filename in double
quotes followed by the line number. For errors in the main source file
the filename is omitted (showing only `line 42: Error: …`); filenames
are always shown for errors in included files. This is the default
format on Unix/Linux and most other platforms.

### 14.5.2 `-E1` — Microsoft Format

```
game.inf(42): Error: Expected ';'
```

Uses the `file(line)` format recognized by Microsoft Visual Studio and
many Windows-based editors. This is the default format on Windows.

### 14.5.3 `-E2` — Macintosh MPW Format

```
File "game.inf"; Line 42	# Error: Expected ';'
```

Uses the Macintosh Programmer's Workshop format. This is the default
format on classic Mac OS.

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
Trace N;
```

where *N* is an integer trace level:

| Level | Output |
|-------|--------|
| 0     | Tracing off (default) |
| 1     | Basic trace: tokens, statements |
| 2     | Detailed trace: expression trees, code generation |
| 3+    | Verbose: internal data structures, memory allocation |

### 14.6.2 Trace Aspects

Specific aspects of compilation can be traced independently using the
`Trace` directive with a keyword:

```inform6
Trace assembly on;
Trace expressions on;
Trace tokens on;
Trace symbols on;
```

Each aspect can be turned `on` or `off`, or given a numeric level. For
the full list of trace options and their corresponding command-line
switches, see §12.6.

### 14.6.3 Example

To diagnose how the compiler parses a particular expression:

```inform6
Trace expressions on;
x = (a + b) * c;
Trace expressions off;
```

The compiler will print the expression tree it builds for the
assignment, showing operator precedence and operand types. This output
goes to `stderr` and does not appear in the compiled game.

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
