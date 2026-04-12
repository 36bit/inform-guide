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

# Appendix F: Veneer Routines Reference

This appendix provides a complete listing and documentation of every
compiler-generated **veneer routine** in Inform 6.44. The veneer is a layer of
run-time support code that the compiler automatically inserts into the output
file. It bridges the gap between Inform 6 language constructs and the
underlying virtual machine capabilities.

All information in this appendix is verified against the compiler source file
`veneer.c` and the header `header.h` in Inform 6.44.

---

## §F.1 What Are Veneer Routines?

When you write Inform 6 code that uses high-level features — property access,
message passing, `metaclass()`, `move`, `give`, article printing, and so on —
the compiler must translate these into sequences of virtual machine
instructions. Many of these operations are too complex for inline code, so the
compiler emits small **veneer routines**: self-contained functions compiled from
built-in source text held inside the compiler itself.

The key characteristics of veneer routines are:

1. **Demand-driven compilation.** A veneer routine is only compiled into the
   output if the program actually uses the language feature that requires it.
   The compiler tracks which routines are needed during code generation and
   emits only those at the end of compilation.

2. **Dependency chains.** Many veneer routines call other veneer routines. When
   one is marked as needed, all its dependencies are recursively marked as
   well. For example, marking `CA__Pr` (the message-passing routine) also
   marks `Z__Region`, `Cl__Ms`, `RT__Err`, and others.

3. **Replaceable.** If the program (or a library it includes) defines a routine
   with the same name as a veneer routine, the compiler uses the
   programmer-supplied version instead of the built-in one. The standard
   library replaces several veneer routines with more capable versions (e.g.,
   `DefArt`, `InDefArt`, `PrintShortName`, `R_Process`, `EnglishNumber`).

4. **Platform-specific implementations.** Each veneer routine has separate
   Z-machine and Glulx implementations. The Z-machine versions use Z-code
   opcodes; the Glulx versions use Glulx opcodes and memory layout. The
   compiler selects the appropriate set based on the compilation target.

5. **Debug-aware.** Many veneer routines include conditional tracing code,
   compiled only when `DEBUG` or `INFIX` is defined. This allows the run-time
   to report object moves, property changes, attribute changes, and message
   sends to aid debugging.

There are exactly **48** veneer routines (numbered 0–47), defined by the
constant `VENEER_ROUTINES` in `header.h`.

---

## §F.2 How Veneer Routines Are Compiled

The compilation of veneer routines follows this process:

1. **During code generation**, when the compiler encounters a language construct
   that requires a veneer routine (e.g., the `.` property-read operator
   compiling to a call to `RV__Pr`), it calls `veneer_routine(code)` which
   marks the routine as `VR_CALLED` and recursively marks all its dependencies.

2. **At the end of the compilation pass**, `compile_veneer()` is invoked. It
   iterates over all 48 veneer slots. For each routine marked `VR_CALLED`:

   - If the routine's name is **unknown** (not yet defined in the symbol
     table), the compiler concatenates the routine's built-in source fragments
     and compiles them as an Inform routine.
   - If the routine's name is **already defined** as a `Routine` symbol, the
     compiler uses the existing definition. This is how `Replace` works for
     veneer routines — the programmer's definition takes priority.
   - If the name is defined but not as a routine, the compiler reports an
     error.

3. This process repeats in a loop because compiling one veneer routine may mark
   additional dependencies as needed. The loop continues until no new routines
   are marked.

### Using `Replace` with Veneer Routines

Any veneer routine can be overridden by defining a routine with the same name
in your source code. The standard library does this for several routines. When
you write:

```inform6
Replace PrintShortName;

[ PrintShortName obj;
    ! Your custom implementation
    ...
];
```

The compiler will use your version instead of the built-in veneer code. This
works because veneer routines are compiled only at the end of the pass; if the
symbol is already defined, the veneer source is skipped.

---

## §F.3 Complete Veneer Routine Listing

The following table lists all 48 veneer routines by their index number, name,
and functional category.

| Index | Name | Category | Both VMs? |
|-------|------|----------|-----------|
| 0 | `Box__Routine` | Statement support | Yes (different implementations) |
| 1 | `R_Process` | Action processing | Yes |
| 2 | `DefArt` | Printing | Yes |
| 3 | `InDefArt` | Printing | Yes |
| 4 | `CDefArt` | Printing | Yes |
| 5 | `CInDefArt` | Printing | Yes |
| 6 | `PrintShortName` | Printing | Yes |
| 7 | `EnglishNumber` | Printing | Yes |
| 8 | `Print__PName` | Printing | Yes (different implementations) |
| 9 | `WV__Pr` | Property write | Yes |
| 10 | `RV__Pr` | Property read | Yes |
| 11 | `CA__Pr` | Message passing / property call | Yes (different implementations) |
| 12 | `IB__Pr` | Property pre-increment | Yes |
| 13 | `IA__Pr` | Property post-increment | Yes |
| 14 | `DB__Pr` | Property pre-decrement | Yes |
| 15 | `DA__Pr` | Property post-decrement | Yes |
| 16 | `RA__Pr` | Property address read | Yes (different implementations) |
| 17 | `RL__Pr` | Property length read | Yes (different implementations) |
| 18 | `RA__Sc` | Superclass (::) operator | Yes (different implementations) |
| 19 | `OP__Pr` | `provides` operator | Yes (different implementations) |
| 20 | `OC__Cl` | `ofclass` operator | Yes (different implementations) |
| 21 | `Copy__Primitive` | Object deep copy | Yes (different implementations) |
| 22 | `RT__Err` | Run-time error reporting | Yes (different implementations) |
| 23 | `Z__Region` | Value type detection | Yes (different implementations) |
| 24 | `Unsigned__Compare` | Unsigned comparison | Yes (different implementations) |
| 25 | `Meta__class` | `metaclass()` function | Yes |
| 26 | `CP__Tab` | Common property table search | Yes (different implementations) |
| 27 | `Cl__Ms` | Class message dispatch | Yes (different implementations) |
| 28 | `RT__ChT` | Checked `move` | Yes (different implementations) |
| 29 | `RT__ChR` | Checked `remove` | Yes (different implementations) |
| 30 | `RT__ChG` | Checked `give` | Yes (different implementations) |
| 31 | `RT__ChGt` | Checked `give ~` (remove attribute) | Yes (different implementations) |
| 32 | `RT__ChPS` | Checked property store (`.` write) | Yes (different implementations) |
| 33 | `RT__ChPR` | Checked property read (`.` read) | Yes |
| 34 | `RT__TrPS` | Trace property setting | Yes |
| 35 | `RT__ChLDB` | Checked byte load (`->` read) | Yes (different implementations) |
| 36 | `RT__ChLDW` | Checked word load (`-->` read) | Yes (different implementations) |
| 37 | `RT__ChSTB` | Checked byte store (`->` write) | Yes (different implementations) |
| 38 | `RT__ChSTW` | Checked word store (`-->` write) | Yes (different implementations) |
| 39 | `RT__ChPrintC` | Checked `print (char)` | Yes (different implementations) |
| 40 | `RT__ChPrintA` | Checked `print (address)` | Yes (different implementations) |
| 41 | `RT__ChPrintS` | Checked `print (string)` | Yes |
| 42 | `RT__ChPrintO` | Checked `print (object)` | Yes (different implementations) |
| 43 | `OB__Move` | Low-level object move | **Glulx only** |
| 44 | `OB__Remove` | Low-level object remove | **Glulx only** |
| 45 | `Print__Addr` | Print dictionary word by address | **Glulx only** |
| 46 | `Glk__Wrap` | Glk dispatch wrapper | **Glulx only** |
| 47 | `Dynam__String` | Dynamic string (printing variable) | **Glulx only** |

**Note on Glulx-only routines (43–47):** The Z-machine has built-in opcodes for
object tree manipulation (`@insert_obj`, `@remove_obj`), dictionary word
printing (`@print_addr`), and no Glk or dynamic string system. Glulx lacks
these opcodes, so the veneer must provide them in software.

---

## §F.4 Printing Routines (0–8)

These routines implement print rule syntax and article printing. The standard
library is expected to replace most of them with more capable versions. The
veneer provides minimal fallbacks so that programs which do not include the
library can still compile.

### §F.4.1 `Box__Routine` (Index 0)

**Purpose:** Implements the `box` statement, which displays quoted text in a
centered box on the upper window.

**Invoked by:** The `box` statement.

**Signature:** `Box__Routine(maxw, table)`

**Parameters:**

- `maxw` — the width (in characters) of the widest line in the box.
- `table` — address of a word array. Entry 0 is the number of lines; entries
  1 through *n* are packed string addresses for each line.

**Behavior:**

- [Z-machine] Splits the screen, opens the upper window, centers the box text
  using `@set_cursor` and `style reverse`, then transcribes the box content to
  the transcript stream (`@output_stream $ffff`).
- [Glulx] Calls `glk($0086, 7)` to set a block-quote style, prints each line
  with a newline, then restores `glk($0086, 0)` for normal style. This is a
  simplified implementation; the library typically provides a more
  sophisticated version.

**Typically replaced by:** The standard library, which provides a more
full-featured box display.

---

### §F.4.2 `R_Process` (Index 1)

**Purpose:** Processes a generated action at run-time. This is the entry point
for action execution via the `<<Action>>` and `<Action>` syntax.

**Invoked by:** Action statements (`<<Action noun second>>`).

**Signature:** `R_Process(a, b, c, d)`

**Parameters:**

- `a` — the action number.
- `b` — the noun (first argument).
- `c` — the second argument.
- `d` — optional; if nonzero, the actor for the action.

**Veneer behavior:** The minimal veneer implementation simply prints a
diagnostic: `Action <a b c>` or `Action <a b c, d>`. This allows
non-library programs to compile action statements without errors, though the
actions will not be functionally processed.

**Typically replaced by:** The standard library's action processing system.

---

### §F.4.3 `DefArt` (Index 2)

**Purpose:** Prints the definite article form of an object's name (e.g., "the
lamp").

**Invoked by:** The `print (the) obj` and `print (DefArt) obj` syntax.

**Signature:** `DefArt(obj)`

**Veneer behavior:** Prints `the ` followed by the object's short name.

**Typically replaced by:** The standard library, which handles the `proper`
attribute, `article` property, plural forms, and language-specific rules.

---

### §F.4.4 `InDefArt` (Index 3)

**Purpose:** Prints the indefinite article form of an object's name (e.g., "a
lamp").

**Invoked by:** The `print (a) obj` and `print (InDefArt) obj` syntax.

**Signature:** `InDefArt(obj)`

**Veneer behavior:** Prints `a ` followed by the object's short name.

**Typically replaced by:** The standard library, with proper article handling.

---

### §F.4.5 `CDefArt` (Index 4)

**Purpose:** Prints the capitalized definite article form (e.g., "The lamp").

**Invoked by:** The `print (The) obj` and `print (CDefArt) obj` syntax.

**Signature:** `CDefArt(obj)`

**Veneer behavior:** Prints `The ` followed by the object's short name.

**Typically replaced by:** The standard library.

---

### §F.4.6 `CInDefArt` (Index 5)

**Purpose:** Prints the capitalized indefinite article form (e.g., "A lamp").

**Invoked by:** The `print (A) obj` and `print (CInDefArt) obj` syntax.

**Signature:** `CInDefArt(obj)`

**Veneer behavior:** Prints `A ` followed by the object's short name.

**Typically replaced by:** The standard library.

---

### §F.4.7 `PrintShortName` (Index 6)

**Purpose:** Prints the short name of an object, or a type description for
non-object values.

**Invoked by:** The `print (name) obj` and `print (PrintShortName) obj` syntax.
Also used internally by other veneer routines for diagnostic output.

**Signature:** `PrintShortName(obj)`

**Behavior:** Uses `metaclass()` to determine the type of the value:

- `0` (nothing): prints `nothing`
- `Object`: prints the object's short name via `@print_obj` [Z-machine] or
  by streaming the name string from `GOBJFIELD_NAME` [Glulx]
- `Class`: prints `class ` followed by the object's short name
- `Routine`: prints `(routine at <address>)`
- `String`: prints `(string at <address>)`

**Typically replaced by:** The standard library, which provides more
sophisticated name printing including `short_name` property support and
language-specific handling.

---

### §F.4.8 `EnglishNumber` (Index 7)

**Purpose:** Prints a number in English words (e.g., 5 → "five").

**Invoked by:** The `print (number) n` and `print (EnglishNumber) n` syntax.

**Signature:** `EnglishNumber(obj)`

**Veneer behavior:** Simply prints the number in decimal digits. The name
"English" is historical; the veneer version does not actually produce English
words.

**Typically replaced by:** The standard library's `LanguageNumber` routine
(routed through `EnglishNumber`), which prints numbers as English words for
values up to a language-specific limit.

---

### §F.4.9 `Print__PName` (Index 8)

**Purpose:** Prints the name of a property, given its identifier number. Used
for diagnostic messages (e.g., in `RT__Err`).

**Invoked by:** The `print (property) id` syntax.

**Signature:** `Print__PName(prop)`

**Parameters:**

- `prop` — the property identifier. May include class-encoding bits for
  inherited properties.

**Behavior:**

If the property identifier has class-encoding bits set (the upper bits encode a
class index from `#classes_table`), the routine first prints the class name
followed by `::`, then decodes the actual property number.

- [Z-machine] If `prop` has `$c000` set, the class index is in the low byte
  (`prop & $ff`) and the property index is in bits 8–13 or 8–14 depending on
  whether bit 15 is set.
- [Glulx] If `prop` has any bits set in the upper 16 bits (`$FFFF0000`), the
  class index is in the low 16 bits and the property index is obtained by
  shifting right 16.

After decoding, the routine looks up the property name in `#identifiers_table`.
If `$OMIT_SYMBOL_TABLE` is defined, it prints `<number N>` instead.

---

## §F.5 Object-Oriented System Routines (9–20)

These routines implement the run-time half of Inform's object-oriented
programming system. They handle property access, message passing, the class
system operators (`provides`, `ofclass`), and the superclass `::` operator.

> **Note:** These routines are never needed for programs that use only Inform 5
> features (simple property access via common properties and no class system).

### §F.5.1 `WV__Pr` (Index 9) — Write Value to Property

**Purpose:** Writes a value to an individual property of an object.

**Invoked by:** The `obj.prop = value` assignment when `prop` is an individual
(non-common) property.

**Signature:** `WV__Pr(obj, identifier, value)`

**Behavior:**

1. Looks up the property address via `obj..&identifier` [Z-machine] or
   `obj.&id` [Glulx].
2. If the address is 0 (property not found), calls `RT__Err` with `"write to"`
   and returns.
3. In debug mode, if tracing is active, calls `RT__TrPS` to trace the property
   setting.
4. Writes `value` to the first word of the property data (`x-->0 = value`).

**Dependencies:** `RT__Err`, `RT__TrPS`

---

### §F.5.2 `RV__Pr` (Index 10) — Read Value from Property

**Purpose:** Reads a value from an individual property of an object.

**Invoked by:** The `obj.prop` expression when `prop` is an individual
(non-common) property.

**Signature:** `RV__Pr(obj, identifier)`

**Behavior:**

1. Looks up the property address.
2. If not found:
   - [Z-machine] If `identifier` is in the common property range (1–63) and
     the property length ≤ 2, returns the common property value directly
     (`obj.identifier`). Otherwise calls `RT__Err`.
   - [Glulx] If `identifier` is in the common property range (1 to
     `INDIV_PROP_START`-1), returns the default value from `#cpv__start-->id`.
     Otherwise calls `RT__Err`.
3. [Z-machine] If found but the property data length > 2, calls `RT__Err`
   (error code 2 in v3, error with extra arg in v5+) because the `.` operator
   requires a 1- or 2-byte property.
4. Returns the first word of the property data (`x-->0`).

**Dependencies:** `RT__Err`

---

### §F.5.3 `CA__Pr` (Index 11) — Call/Print/Read Property

**Purpose:** The central message-passing routine. Implements `obj.prop(args)`
for all forms: calling a routine stored in a property, printing a string, or
returning a bare value.

**Invoked by:** The `obj.prop()` and `obj.prop(a, b, ...)` message-send syntax.

**Signature:**

- [Z-machine] `CA__Pr(obj, id, a, b, c, d, e, f)` — up to 6 explicit
  arguments (uses `@check_arg_count` to detect how many were passed).
- [Glulx] `CA__Pr(obj, id, ...)` — variable-argument using `_vararg_count`.

**Behavior:**

This is the most complex veneer routine. Its logic is:

1. **Non-object values.** If `obj` is not a valid object (outside the object
   range), `Z__Region` is called:
   - Region 2 (routine): If `id == call`, sets `sender`/`self` and calls the
     routine via `indirect()` [Z-machine] or `@call` [Glulx].
   - Region 3 (string): If `id == print`, prints the string. If
     `id == print_to_array`, captures the string output into a byte array.
   - Otherwise: jumps to `Call__Error`.

2. **Class methods.** If `obj` is a class (i.e., `obj in Class`), and `id` is
   one of the five built-in class messages (`create`, `recreate`, `destroy`,
   `remaining`, `copy`), delegates to `Cl__Ms`.

3. **Debug tracing.** If `DEBUG` is defined and `debug_flag & 1` is set, prints
   a trace message showing the object, property, and arguments.

4. **Common properties (id 1–63).** Looks up the property data. If not found,
   falls back to the property defaults table.

5. **Individual properties.** Looks up via `obj..&id`. If not found, reports an
   error via `RT__Err`.

6. **Iterates over the property data words.** For each word in the property
   data:
   - If the value is `$ffff` (Z-machine) or `-1`, returns false.
   - If the value is a routine (region 2), saves/restores `sender`, `self`,
     and `sw__var`, then calls the routine with all passed arguments. If the
     routine returns nonzero, that value is returned.
   - If the value is a string (region 3), prints it with a newline and returns
     true.
   - Otherwise (bare value), returns the value.

7. If all property data entries were processed without a nonzero return,
   returns false.

**Dependencies:** `Z__Region`, `Cl__Ms`, `RT__Err`, `RA__Pr` [Glulx],
`RL__Pr` [Glulx], `PrintShortName` [Glulx], `Print__PName` [Glulx],
`Glk__Wrap` [Glulx]

**Z-machine limitation:** In Z-code version 3, `CA__Pr` generates a compile
error: "Object message calls are not supported in v3."

---

### §F.5.4 `IB__Pr` (Index 12) — Pre-Increment Property

**Purpose:** Implements `++(obj.prop)` for individual properties.

**Invoked by:** The `++obj.prop` expression.

**Signature:** `IB__Pr(obj, identifier)`

**Behavior:**

1. Looks up the property address via `obj..&identifier` [Z-machine] or
   `obj.&identifier` [Glulx].
2. If not found, calls `RT__Err("increment", obj, identifier)`.
3. In debug mode, traces the new value.
4. Returns `++(x-->0)` — increments the value in place and returns the new
   value.

**Dependencies:** `RT__Err`, `RT__TrPS`

---

### §F.5.5 `IA__Pr` (Index 13) — Post-Increment Property

**Purpose:** Implements `(obj.prop)++` for individual properties.

**Invoked by:** The `obj.prop++` expression.

**Signature:** `IA__Pr(obj, identifier)`

**Behavior:** Same as `IB__Pr` except returns `(x-->0)++` — returns the old
value and then increments.

**Dependencies:** `RT__Err`, `RT__TrPS`

---

### §F.5.6 `DB__Pr` (Index 14) — Pre-Decrement Property

**Purpose:** Implements `--(obj.prop)` for individual properties.

**Invoked by:** The `--obj.prop` expression.

**Signature:** `DB__Pr(obj, identifier)`

**Behavior:** Same structure as `IB__Pr` but decrements: returns `--( x-->0)`.

**Dependencies:** `RT__Err`, `RT__TrPS`

---

### §F.5.7 `DA__Pr` (Index 15) — Post-Decrement Property

**Purpose:** Implements `(obj.prop)--` for individual properties.

**Invoked by:** The `obj.prop--` expression.

**Signature:** `DA__Pr(obj, identifier)`

**Behavior:** Same structure as `IA__Pr` but decrements: returns `(x-->0)--`.

**Dependencies:** `RT__Err`, `RT__TrPS`

---

### §F.5.8 `RA__Pr` (Index 16) — Read Property Address

**Purpose:** Returns the byte address of a property's data for a given object,
or 0 if the object does not provide that property.

**Invoked by:** The `obj.&prop` expression for individual properties.

**Signature:** `RA__Pr(obj, identifier)`

**Behavior:**

- [Z-machine] Handles three cases based on bit-flags in `identifier`:
  1. **Common properties** (identifier 1–63): returns `obj.&identifier`
     directly using the Z-machine's built-in property mechanism.
  2. **Inherited common properties** (`$4000` flag): Looks up the class from
     `#classes_table`, verifies `ofclass`, and navigates the class's property
     table using `CP__Tab`.
  3. **Inherited individual properties** (`$8000` flag): Looks up the class,
     verifies `ofclass`, and walks the individual property table counting
     entries.
  4. **Simple individual properties**: Searches `obj.3` (the individual
     property table) for a matching identifier, considering the private-access
     flag (`$8000` on the identifier when `self == obj`).

- [Glulx] The logic is simpler because of the uniform property table format:
  1. If `id` has upper 16 bits set, it's an inherited property (class::prop).
     Decodes the class index, verifies `ofclass`, shifts to get the real
     property id, and searches the class's property table.
  2. Calls `CP__Tab(obj, id)` to search the object's property table.
  3. If `obj` is a class (not via inheritance lookup), only class-system
     properties (INDIV_PROP_START through INDIV_PROP_START+7) are accessible.
  4. Checks the private-access bit (bit 72 of the property entry, i.e.,
     `@aloadbit prop 72`). If the property is private and `self ~= obj`,
     returns 0.
  5. Returns `prop-->1` (the data address field of the property entry).

**Dependencies:** `CP__Tab` [Z-machine], `OC__Cl` [Glulx], `CP__Tab` [Glulx]

---

### §F.5.9 `RL__Pr` (Index 17) — Read Property Length

**Purpose:** Returns the byte length of a property value for a given object,
or 0 if the property is not provided.

**Invoked by:** The `obj.#prop` expression for individual properties.

**Signature:** `RL__Pr(obj, identifier)`

**Behavior:**

- [Z-machine] For common properties (1–63), returns `obj.#identifier`. For
  individual properties, looks up via `obj..&identifier`, then decodes the
  length from the property header byte. The encoding differs between v3
  (length = 1 + high bits / 32) and v5+ (two-byte size field with `$C0` mask).

- [Glulx] Same class-inheritance and private-access checks as `RA__Pr`.
  Returns `WORDSIZE * ix` where `ix` is the length field from the property
  entry structure (`@aloads prop 1 ix`).

**Dependencies:** `OC__Cl` [Glulx], `CP__Tab` [Glulx]

---

### §F.5.10 `RA__Sc` (Index 18) — Superclass Operator

**Purpose:** Implements the `class::property` (superclass) operator. Returns a
compound property identifier that encodes both the class and the property.

**Invoked by:** The `Class::prop` expression.

**Signature:** `RA__Sc(cla, identifier)`

**Behavior:**

- [Z-machine] Verifies that `cla` is indeed a class (either in metaclass 1 or
  is one of the four built-in classes, or has object number ≤ 4). Searches
  `#classes_table` for the class. Returns a compound identifier:
  - For common properties (< 64): `$4000 + identifier * $100 + j` where `j`
    is the class's index in `#classes_table`.
  - For individual properties: `$8000 + k * $100 + j` where `k` is the
    position of the property within the class's individual property table.

- [Glulx] Returns `id * $10000 + j` where `j` is the class's index in
  `#classes_table`. The simpler encoding reflects Glulx's 32-bit word size.

**Dependencies:** `RT__Err` [both], `OC__Cl` [Glulx]

---

### §F.5.11 `OP__Pr` (Index 19) — Object Provides Property

**Purpose:** Tests whether a given object provides a given property. Implements
the `provides` operator.

**Invoked by:** The `obj provides prop` expression.

**Signature:** `OP__Pr(obj, identifier)`

**Return value:** True if the object provides the property, false otherwise.

**Behavior:**

- [Z-machine]
  1. If `obj` is out of the valid object range, checks `Z__Region`:
     - Region 2 (routine): provides `call` only.
     - Region 3 (string): provides `print` and `print_to_array` only.
     - Otherwise: false.
  2. For common properties (< 64): returns true if `obj.&identifier ~= 0`.
  3. For individual properties: returns true if `obj..&identifier ~= 0`.
  4. For class-system properties (64–71) on class objects (`obj in 1`):
     returns true.

- [Glulx] Uses `Z__Region` for type checking and `RA__Pr` for the actual
  property lookup. Class-system properties (`INDIV_PROP_START` to
  `INDIV_PROP_START+7`) are always provided by classes.

**Dependencies:** `Z__Region` [Z-machine], `RA__Pr` [Glulx], `Z__Region`
[Glulx]

---

### §F.5.12 `OC__Cl` (Index 20) — Object Of Class

**Purpose:** Tests whether a given object is a member of a given class.
Implements the `ofclass` operator.

**Invoked by:** The `obj ofclass cla` expression.

**Signature:** `OC__Cl(obj, cla)`

**Return value:** True if `obj` is of class `cla`, false otherwise.

**Behavior:**

- [Z-machine]
  1. If `obj` is outside the object range:
     - If `cla` is 3 (String metaclass) or 4 (Routine metaclass), checks
       `Z__Region` for a match.
     - Otherwise: false.
  2. If `cla == 1` (Class metaclass): true if `obj` is a class (obj ≤ 4 or
     obj is in the Class object).
  3. If `cla == 2` (Object metaclass): true if `obj` is an ordinary object
     (not a class).
  4. If `cla == 3` or `4`: false (only non-object values can be String or
     Routine).
  5. If `cla` is not itself a class (`cla notin 1`): error.
  6. Reads property 2 (`class` property) of `obj`, which contains a word array
     of class references. Searches this array for `cla`.

- [Glulx] Same logical structure but uses `Z__Region` for type detection,
  checks against the four named metaclass objects (`Class`, `Object`,
  `String`, `Routine`), and reads the class list from property 2 with
  `WORDSIZE`-based length calculation.

**Dependencies:** `Z__Region`, `RT__Err` [Z-machine]; `RA__Pr`, `RL__Pr`,
`Z__Region`, `RT__Err` [Glulx]

---

## §F.6 Utility Routines (21–27)

These routines provide fundamental services used by many other veneer routines.

### §F.6.1 `Copy__Primitive` (Index 21) — Deep Object Copy

**Purpose:** Performs a deep copy of one object's data onto another. Used by the
class system for `create`, `recreate`, `destroy`, and `copy` operations.

**Invoked by:** `Cl__Ms` (class message dispatch).

**Signature:** `Copy__Primitive(o1, o2)` — copies o2's data onto o1.

**Behavior:**

- [Z-machine]
  1. Copies all 48 attributes from o2 to o1.
  2. Copies all common properties (1–63, except 2 and 3) where both objects
     have the property and the sizes match.
  3. Copies individual properties: walks o2's individual property table
     (`o2.3`), and for each entry, finds the matching entry in o1's table and
     copies the data bytes if the sizes match.

- [Glulx]
  1. Copies all attribute bytes (`NUM_ATTR_BYTES`).
  2. Iterates through o2's property table (accessed via `GOBJFIELD_PROPTAB`).
     For each property, finds the corresponding entry in o1 via `CP__Tab`. If
     found and lengths match, copies the property data and the flags word.

**Dependencies:** `CP__Tab` [Glulx]

---

### §F.6.2 `RT__Err` (Index 22) — Run-Time Error Reporter

**Purpose:** Prints diagnostic messages for run-time errors detected by the
veneer's checked operations.

**Invoked by:** Nearly all veneer routines that perform validity checks.

**Signature:** `RT__Err(crime, obj, id, size)`

**Parameters:**

- `crime` — the error code or a string address. When negative, jumps to the
  property-error handler. When a string, the string is used in the message.
- `obj` — the object involved (or a value, depending on the error).
- `id` — additional context (property id, target object, etc.).
- `size` — used for array bounds and property size errors.

**Error codes and their messages:**

| Code | Message |
|------|---------|
| 1 | `class <obj>: 'create' can have 0 to 3 parameters only` |
| 2 | `tried to test ~in~ or ~notin~ of <obj>` |
| 3 | `tried to test ~has~ or ~hasnt~ of <obj>` |
| 4–11 | `tried to find the ~parent/child/sibling/etc.~ of <obj>` |
| 12 | `tried to use ~objectloop~ of <obj>` |
| 13 | `tried to find the ~}~ at end of ~objectloop~ of <obj>` |
| 14 | `tried to ~give~ an attribute to <obj>` |
| 15 | `tried to ~remove~ <obj>` |
| 16 | `tried to ~move~ <obj> to <id>` (source invalid) |
| 17 | `tried to ~move~ <obj> to <id>` (destination invalid) |
| 18 | `tried to ~move~ <obj> to <id>, which would make a loop` |
| 19 | `tried to ~give~ or test ~has~ or ~hasnt~ with a non-attribute on <obj>` |
| 20 | `tried to divide by zero` |
| 21 | `tried to find the ~.&~ of <obj>` |
| 22 | `tried to find the ~.#~ of <obj>` |
| 23 | `tried to find the ~.~ of <obj>` |
| 24 | `tried to read outside memory using ->` |
| 25 | `tried to read outside memory using -->` |
| 26 | `tried to write outside memory using ->` |
| 27 | `tried to write outside memory using -->` |
| 28–31 | Array bounds errors with detailed array name and index information |
| 32 | `objectloop broken because the object <obj> was moved while the loop passed through it` |
| 33 | `tried to print (char) <obj>, which is not a valid ZSCII/Glk character code` |
| 34 | `tried to print (address) on something not the byte address of a string/dict word` |
| 35 | `tried to print (string) on something not a string` |
| 36 | `tried to print (object) on something not an object or class` |
| 37 | `tried to call Glulx print_to_array with only one argument` [Glulx only] |
| 40 | `tried to change printing variable <obj>; must be 0 to <max>` [Glulx only] |

**Property-error path:** When `crime` is negative (a string address), the
routine prints `(object number <obj>) has no property <id> to <crime>`. If
`OMIT_SYMBOL_TABLE` is not defined, it also reports whether the property
exists for any object.

**Array-error path (codes 28–31):** The `size` parameter encodes both the
array type (bits 0–2: 0/1=byte, 2=string, 3=table, 4=buffer) and the access
mode (bit 3 for `-->`, bit 4 for `->`). The `p` parameter is used as an index
into `#array_names_offset` to print the array name (unless `OMIT_SYMBOL_TABLE`
is defined).

---

### §F.6.3 `Z__Region` (Index 23) — Value Type Detection

**Purpose:** Determines what region of the address space a value falls into.
Returns 1 for objects, 2 for code (routine addresses), 3 for strings, and 0
for none of the above.

**Invoked by:** `metaclass()`, `provides`, `ofclass`, `CA__Pr`, and various
runtime checks.

**Signature:** `Z__Region(addr)`

**Return value:**

| Return | Meaning |
|--------|---------|
| 0 | Not a valid reference (or `nothing`/`-1`) |
| 1 | An object number |
| 2 | A code address (routine) |
| 3 | A string address |

**Behavior:**

- [Z-machine] Checks:
  1. If `addr` is 0 or -1, returns 0.
  2. If the address (after shifting for v6/v7 packed addressing) exceeds
     `$001A-->0` (the story file length), returns 0.
  3. If `addr` is in the object range (1 to `#largest_object-255`), returns 1.
  4. For v6/v7 with odd/even packing: tests bit 0 to distinguish strings from
     code. For other versions: compares against `#strings_offset` and
     `#code_offset` thresholds.

- [Glulx] Checks:
  1. If `addr < 36`, returns 0 (below minimum valid address).
  2. If `addr >= memory size` (`@getmemsize`), returns 0.
  3. Reads the byte at `addr`:
     - `$E0` or above → string (return 3)
     - `$C0` to `$DF` → code (return 2)
     - `$70` to `$7F` and `addr >= header field 2` (object area start) →
       object (return 1)
  4. Otherwise returns 0.

**Dependencies:** `Unsigned__Compare` [Z-machine]; `Unsigned__Compare` [Glulx]

---

### §F.6.4 `Unsigned__Compare` (Index 24) — Unsigned Integer Comparison

**Purpose:** Compares two values as unsigned integers. Returns 1 if x > y,
0 if x == y, -1 if x < y.

**Invoked by:** `Z__Region`, `RT__ChLDB`, `RT__ChLDW`, `RT__ChSTB`,
`RT__ChSTW`, `RT__ChPrintA`.

**Signature:** `Unsigned__Compare(x, y)`

**Behavior:**

- [Z-machine] Since Z-machine arithmetic is signed 16-bit, unsigned comparison
  requires special handling:
  1. If x == y, return 0.
  2. If x < 0 and y >= 0, x is larger (its high bit is set), return 1.
  3. If x >= 0 and y < 0, y is larger, return -1.
  4. Otherwise, compare `x & $7fff` vs `y & $7fff`.

- [Glulx] Uses native unsigned comparison opcodes:
  ```
  @jleu x y ?lesseq;
  return 1;
  .lesseq;
  @jeq x y ?equal;
  return -1;
  .equal;
  return 0;
  ```

---

### §F.6.5 `Meta__class` (Index 25) — Metaclass Function

**Purpose:** Returns the metaclass of a value. This is the implementation of
the `metaclass()` built-in function.

**Invoked by:** The `metaclass(x)` expression.

**Signature:** `Meta__class(obj)`

**Return value:** One of `Class`, `Object`, `Routine`, `String`, or 0 (for
invalid values).

**Behavior:**

Calls `Z__Region(obj)`:
- Region 2 → `Routine`
- Region 3 → `String`
- Region 1:
  - [Z-machine] If `obj in 1` (in the Class metaclass object) or `obj <= 4`
    → `Class`. Otherwise → `Object`.
  - [Glulx] If `obj in Class` or `obj` is one of `Class`, `String`,
    `Routine`, `Object` → `Class`. Otherwise → `Object`.
- Region 0 → false (0).

**Dependencies:** `Z__Region`

---

### §F.6.6 `CP__Tab` (Index 26) — Common Property Table Search

**Purpose:** Searches a property table for a given property identifier. This
imitates the behavior of the Z-machine's `@get_prop_addr` opcode for the
veneer's use, and provides the core property lookup mechanism in Glulx.

**Invoked by:** `RA__Pr`, `RL__Pr`, `Copy__Primitive`.

**Signature:**

- [Z-machine] `CP__Tab(x, id)` where `x` is the address of the property
  table.
- [Glulx] `CP__Tab(obj, id)` where `obj` is the object.

**Behavior:**

- [Z-machine] Walks the property table starting at address `x`. For each
  entry, reads the property number from the size/number byte:
  - v3: property number = `n & $1f`, length = `(n / $20) + 1`
  - v5+: if the size byte has bit 7 set, the next byte gives the length
    (masked with `$3f`); otherwise, bit 6 indicates length 2, else length 1.
    Property number = `n & $3f`.
  - If the property number matches `id`, returns the data address.
  - If `id == -1`, returns the address of the first byte past the table end
    (used for table-length calculations).

- [Glulx] Reads the property table from `obj-->GOBJFIELD_PROPTAB`. The table
  starts with a count word, followed by 10-byte property entries. Uses the
  `@binarysearch` opcode for efficient lookup:
  ```
  @binarysearch id 2 otab 10 max 0 0 res;
  ```
  Returns the address of the matching property entry structure (not the data
  address — the data address is at `res-->1`).

---

### §F.6.7 `Cl__Ms` (Index 27) — Class Message Dispatch

**Purpose:** Implements the five built-in message-receiving properties of Class
objects: `create`, `recreate`, `destroy`, `remaining`, and `copy`.

**Invoked by:** `CA__Pr` when the target object is a class and the property is
one of these five identifiers.

**Signature:**

- [Z-machine] `Cl__Ms(obj, id, y, a, b, c, d)` where `y` is the argument
  count.
- [Glulx] `Cl__Ms(obj, id, ...)` using variable arguments.

**Behavior:**

- `create`: If the class has more than one child (the first child is the
  prototype), removes the second child from the class, calls `create` on it
  if provided, and returns the new instance. The prototype (first child) is
  never removed.

- `recreate(a)`: Verifies `a ofclass obj`, calls `a.destroy()` if provided
  [Glulx only — the Z-machine version does not call destroy first],
  copies the prototype's data onto `a` via `Copy__Primitive`, and calls
  `a.create(...)` if provided.

- `destroy(a)`: Verifies `a ofclass obj`, calls `a.destroy()` if provided,
  copies the prototype's data onto `a`, and moves `a` back into the class
  (returning it to the pool).

- `remaining`: Returns `children(obj) - 1` (the number of available instances,
  excluding the prototype).

- `copy(a, b)`: Verifies both `a` and `b` are of class `obj`, then calls
  `Copy__Primitive(a, b)`.

**Z-machine limitation:** In v3, class messages generate a compile error.

**Dependencies:** `RT__Err`, `Copy__Primitive`, `OB__Remove` [Glulx],
`OB__Move` [Glulx], `OC__Cl` [Glulx], `OP__Pr` [Glulx]

---

## §F.7 Run-Time Checked Operations (28–42)

When the compiler switch `-S` (strict mode) is active (the default), or when
`DEBUG` is defined, many operations are compiled with run-time safety checks.
Instead of emitting the raw opcode directly, the compiler emits a call to a
checking veneer routine that validates the operation first.

When strict checking is disabled (with `-~S`), the compiler emits the raw
opcodes directly, bypassing these routines entirely. This produces smaller and
faster code but with no protection against invalid operations.

### §F.7.1 `RT__ChT` (Index 28) — Checked Move

**Purpose:** Validates and performs a `move obj1 to obj2` statement.

**Invoked by:** The `move` statement when strict checking is enabled.

**Signature:** `RT__ChT(obj1, obj2)`

**Checks performed:**

1. Verifies `obj1` is a valid, non-class object:
   - [Z-machine] `obj1 >= 5`, `obj1 <= #largest_object-255`, and not in
     metaclass 1 (`obj1 notin 1`). Error 16 if invalid.
   - [Glulx] `obj1 ~= 0`, `Z__Region(obj1) == 1`, not a metaclass object,
     and not in `Class`. Error 16 if invalid.
2. Verifies `obj2` similarly. Error 17 if invalid.
3. Checks that moving `obj1` into `obj2` would not create a containment loop
   by walking up from `obj2` through its parents. Error 18 if a loop would
   result.
4. In debug mode, prints a trace message.
5. Performs the move:
   - [Z-machine] `@insert_obj obj1 obj2`
   - [Glulx] `OB__Move(obj1, obj2)`

**Dependencies:** `RT__Err`; `Z__Region`, `OB__Move` [Glulx]

---

### §F.7.2 `RT__ChR` (Index 29) — Checked Remove

**Purpose:** Validates and performs a `remove obj1` statement.

**Invoked by:** The `remove` statement when strict checking is enabled.

**Signature:** `RT__ChR(obj1)`

**Checks performed:**

1. Verifies `obj1` is a valid, non-class object. Error 15 if invalid.
2. In debug mode, prints a trace message.
3. Performs the removal:
   - [Z-machine] `@remove_obj obj1`
   - [Glulx] `OB__Remove(obj1)`

**Dependencies:** `RT__Err`; `Z__Region`, `OB__Remove` [Glulx]

---

### §F.7.3 `RT__ChG` (Index 30) — Checked Give Attribute

**Purpose:** Validates and performs a `give obj1 a` statement.

**Invoked by:** The `give` statement when strict checking is enabled.

**Signature:** `RT__ChG(obj1, a)`

**Checks performed:**

1. Verifies `obj1` is a valid, non-class object. Error 14 if invalid.
2. Verifies the attribute number is in range:
   - [Z-machine] `a >= 0 && a < 48`.
   - [Glulx] `a >= 0 && a < NUM_ATTR_BYTES * 8`.
   Error 19 if out of range.
3. If `obj1` already has attribute `a`, returns without action (optimization).
4. In debug mode, prints a trace message (unless the attribute is `workflag`).
5. Gives the attribute:
   - [Z-machine] `@set_attr obj1 a`
   - [Glulx] `give obj1 a` (compiled as Inform statement)

**Dependencies:** `RT__Err`

---

### §F.7.4 `RT__ChGt` (Index 31) — Checked Remove Attribute

**Purpose:** Validates and performs a `give obj1 ~a` statement (removing an
attribute).

**Invoked by:** The `give obj ~attr` statement when strict checking is enabled.

**Signature:** `RT__ChGt(obj1, a)`

**Checks performed:** Same as `RT__ChG`, except:

- If `obj1` does **not** have attribute `a`, returns without action.
- Clears the attribute:
  - [Z-machine] `@clear_attr obj1 a`
  - [Glulx] `give obj1 ~a`

**Dependencies:** `RT__Err`

---

### §F.7.5 `RT__ChPS` (Index 32) — Checked Property Store

**Purpose:** Validates and performs a common property write (`obj.prop = val`).

**Invoked by:** Common property assignment with strict checking.

**Signature:** `RT__ChPS(obj, prop, val)`

**Checks performed:**

- [Z-machine] Verifies `obj` is valid, has the property, and the property
  length ≤ 2. Error `"set"` if invalid. Then `@put_prop obj prop val`.
- [Glulx] Verifies `obj` is valid and not a metaclass/class object. Delegates
  the actual write to `WV__Pr`.
- In debug mode, calls `RT__TrPS` to trace the change.

**Dependencies:** `RT__Err`, `RT__TrPS`; `WV__Pr` [Glulx]

---

### §F.7.6 `RT__ChPR` (Index 33) — Checked Property Read

**Purpose:** Validates and performs a common property read (`obj.prop`).

**Invoked by:** Common property read with strict checking.

**Signature:** `RT__ChPR(obj, prop)`

**Checks performed:**

- [Z-machine] Verifies `obj` is valid and property length ≤ 2. On error, uses
  obj=2 as a fallback. Then `@get_prop obj prop`.
- [Glulx] Verifies `obj` is valid. Delegates the actual read to `RV__Pr`.

**Dependencies:** `RT__Err`; `RV__Pr` [Glulx]

---

### §F.7.7 `RT__TrPS` (Index 34) — Trace Property Setting

**Purpose:** Prints a trace message when a property value is changed. This is
a debug-only helper.

**Invoked by:** `WV__Pr`, `IB__Pr`, `IA__Pr`, `DB__Pr`, `DA__Pr`, `RT__ChPS`
— always guarded by `#ifdef DEBUG` or `#ifdef INFIX`.

**Signature:** `RT__TrPS(obj, prop, val)`

**Output:** `[Setting <obj>.<prop> to <val>]` followed by a newline.

---

### §F.7.8 `RT__ChLDB` (Index 35) — Checked Byte Load

**Purpose:** Validates and performs a byte-array read (`base->offset`).

**Invoked by:** The `->` operator when strict checking is enabled.

**Signature:** `RT__ChLDB(base, offset)`

**Checks performed:**

- [Z-machine] Computes `a = base + offset`. If `a >= #readable_memory_offset`,
  error 24.
- [Glulx] Computes `a = base + offset`. If `a >= memory size`
  (`@getmemsize`), error 24.

**On success:** Returns the byte value via `@loadb` [Z-machine] or `@aloadb`
[Glulx].

**Dependencies:** `Unsigned__Compare`, `RT__Err`

---

### §F.7.9 `RT__ChLDW` (Index 36) — Checked Word Load

**Purpose:** Validates and performs a word-array read (`base-->offset`).

**Invoked by:** The `-->` operator when strict checking is enabled.

**Signature:** `RT__ChLDW(base, offset)`

**Checks performed:**

- [Z-machine] `a = base + 2*offset`. If `a >= #readable_memory_offset`,
  error 25.
- [Glulx] `a = base + WORDSIZE*offset`. If `a >= memory size`, error 25.

**On success:** Returns the word value via `@loadw` [Z-machine] or `@aload`
[Glulx].

**Dependencies:** `Unsigned__Compare`, `RT__Err`

---

### §F.7.10 `RT__ChSTB` (Index 37) — Checked Byte Store

**Purpose:** Validates and performs a byte-array write (`base->offset = val`).

**Invoked by:** The `->` store operator when strict checking is enabled.

**Signature:** `RT__ChSTB(base, offset, val)`

**Checks performed:**

- [Z-machine] Computes `a = base + offset`. Checks that `a` falls within one
  of the writable memory regions:
  - `#array__start` to `#array__end` (global arrays)
  - `#cpv__start` to `#cpv__end` (common property values)
  - `#ipv__start` to `#ipv__end` (individual property values)
  - Address `$0011` (the flags2 byte in the Z-machine header)
  Error 26 if outside all writable regions.

- [Glulx] Computes `a = base + offset`. Checks that `a < memory size`
  (`@getmemsize`) and `a >= RAM start` (header field at word 2). Error 26 if
  outside writable RAM.

**On success:** Stores via `@storeb` [Z-machine] or `@astoreb` [Glulx].

**Dependencies:** `Unsigned__Compare`, `RT__Err`

---

### §F.7.11 `RT__ChSTW` (Index 38) — Checked Word Store

**Purpose:** Validates and performs a word-array write (`base-->offset = val`).

**Invoked by:** The `-->` store operator when strict checking is enabled.

**Signature:** `RT__ChSTW(base, offset, val)`

**Checks performed:**

- [Z-machine] `a = base + 2*offset`. Same writable-region checks as
  `RT__ChSTB`, plus address `$0010` (flags2 word). Error 27 if outside.

- [Glulx] `a = base + WORDSIZE*offset`. Same RAM-range check as `RT__ChSTB`.
  Error 27 if outside.

**On success:** Stores via `@storew` [Z-machine] or `@astore` [Glulx].

**Dependencies:** `Unsigned__Compare`, `RT__Err`

---

### §F.7.12 `RT__ChPrintC` (Index 39) — Checked Print Char

**Purpose:** Validates and performs `print (char) c`.

**Invoked by:** The `print (char)` statement when strict checking is enabled.

**Signature:** `RT__ChPrintC(c)`

**Checks performed:**

- [Z-machine] The character must be one of: 0, 9, 11, 13, 32–126, or 155–251
  (the valid ZSCII output range). Error 33 otherwise.
- [Glulx] The character must not be in the control-character ranges:
  `c < 10`, or `10 < c < 32`, or `126 < c < 160`. Error 33 otherwise.
  If valid and `c < 256`, uses `@streamchar`; if `c >= 256`, uses
  `@streamunichar`.

**Dependencies:** `RT__Err`

---

### §F.7.13 `RT__ChPrintA` (Index 40) — Checked Print Address

**Purpose:** Validates and performs `print (address) a`.

**Invoked by:** The `print (address)` statement when strict checking is enabled.

**Signature:** `RT__ChPrintA(a)`

**Checks performed:**

- [Z-machine] If `a >= #readable_memory_offset`, error 34. Otherwise
  `@print_addr a`.
- [Glulx] If `a < 36` or `a >= memory size`, error 34. If `a->0 ~= $60`
  (not a dictionary word marker byte), error 34. Otherwise calls
  `Print__Addr(a)`.

**Dependencies:** `Unsigned__Compare`, `RT__Err`; `Print__Addr` [Glulx]

**Note:** In Glulx, `print (address)` can only print dictionary words (entries
beginning with `$60`), unlike the Z-machine where it prints any byte-addressed
string.

---

### §F.7.14 `RT__ChPrintS` (Index 41) — Checked Print String

**Purpose:** Validates and performs `print (string) a`.

**Invoked by:** The `print (string)` statement when strict checking is enabled.

**Signature:** `RT__ChPrintS(a)`

**Checks performed:** If `Z__Region(a) ~= 3`, error 35.

**On success:** `@print_paddr a` [Z-machine] or `@streamstr a` [Glulx].

**Dependencies:** `Z__Region`, `RT__Err`

---

### §F.7.15 `RT__ChPrintO` (Index 42) — Checked Print Object

**Purpose:** Validates and performs `print (object) a`.

**Invoked by:** The `print (object)` statement when strict checking is enabled.

**Signature:** `RT__ChPrintO(a)`

**Checks performed:** If `Z__Region(a) ~= 1`, error 36.

**On success:**
- [Z-machine] `@print_obj a`
- [Glulx] Loads the name string from `obj-->GOBJFIELD_NAME` and streams it
  with `@streamstr`.

**Dependencies:** `Z__Region`, `RT__Err`

---

## §F.8 Glulx-Only Routines (43–47)

These five routines exist only in the Glulx veneer. They implement operations
that the Z-machine handles via built-in opcodes but that Glulx must perform in
software.

### §F.8.1 `OB__Move` (Index 43) — Low-Level Object Move

**Purpose:** Moves an object within the Glulx object tree. This is the Glulx
equivalent of the Z-machine's `@insert_obj` opcode.

**Invoked by:** `RT__ChT` (checked move) and `Cl__Ms` (class message dispatch).

**Signature:** `OB__Move(obj, dest)`

**Behavior:**

1. Reads `obj`'s current parent from `obj-->GOBJFIELD_PARENT`.
2. If obj has a parent, unlinks obj from its parent's child list:
   - If obj is the first child, updates `parent-->GOBJFIELD_CHILD` to
     `obj-->GOBJFIELD_SIBLING`.
   - Otherwise, walks the sibling chain to find obj and patches the link.
3. Sets `obj-->GOBJFIELD_SIBLING` to `dest-->GOBJFIELD_CHILD` (the current
   first child of dest).
4. Sets `obj-->GOBJFIELD_PARENT` to `dest`.
5. Sets `dest-->GOBJFIELD_CHILD` to `obj` (making obj the new first child).

**Glulx object field offsets** (from `header.h`):

| Field | Word Offset |
|-------|-------------|
| `GOBJFIELD_CHAIN` | `1 + NUM_ATTR_BYTES/4` |
| `GOBJFIELD_NAME` | `2 + NUM_ATTR_BYTES/4` |
| `GOBJFIELD_PROPTAB` | `3 + NUM_ATTR_BYTES/4` |
| `GOBJFIELD_PARENT` | `4 + NUM_ATTR_BYTES/4` |
| `GOBJFIELD_SIBLING` | `5 + NUM_ATTR_BYTES/4` |
| `GOBJFIELD_CHILD` | `6 + NUM_ATTR_BYTES/4` |

---

### §F.8.2 `OB__Remove` (Index 44) — Low-Level Object Remove

**Purpose:** Removes an object from the Glulx object tree. This is the Glulx
equivalent of the Z-machine's `@remove_obj` opcode.

**Invoked by:** `RT__ChR` (checked remove) and `Cl__Ms`.

**Signature:** `OB__Remove(obj)`

**Behavior:**

1. Reads `obj-->GOBJFIELD_PARENT`. If 0 (no parent), returns immediately.
2. Unlinks obj from its parent's child list (same unlinking logic as
   `OB__Move`).
3. Sets `obj-->GOBJFIELD_SIBLING` to 0.
4. Sets `obj-->GOBJFIELD_PARENT` to 0.

---

### §F.8.3 `Print__Addr` (Index 45) — Print Dictionary Word Address

**Purpose:** Prints a dictionary word given its memory address. In Glulx,
`print (address)` can only print dictionary words, unlike the Z-machine where
it prints any byte-addressed string.

**Invoked by:** `RT__ChPrintA` (checked print address).

**Signature:** `Print__Addr(addr)`

**Behavior:**

1. If `addr->0 ~= $60` (not a dictionary word marker), prints a diagnostic:
   `(<addr>: not dict word)`.
2. Otherwise, iterates through the word data (bytes 1 to `DICT_WORD_SIZE`):
   - If `DICT_IS_UNICODE` is not defined, reads bytes with `addr->ix`.
   - If `DICT_IS_UNICODE` is defined, reads words with `addr-->ix`.
   - Stops at a zero character.
   - Prints each character with `print (char) ch`.

---

### §F.8.4 `Glk__Wrap` (Index 46) — Glk Dispatch Wrapper

**Purpose:** Wraps the `@glk` opcode to provide a callable-routine interface
for Glk calls. This allows Glk functions to be invoked from Inform source code
with a familiar calling convention.

**Invoked by:** The `glk($selector, args...)` syntax in Inform source code.

**Signature:** `Glk__Wrap(callid, ...)` — variable arguments.

**Behavior:**

1. Pops the first argument from the stack as `callid` (the Glk function
   selector).
2. Decrements `_vararg_count` by 1 (since `callid` is not passed to Glk).
3. Calls `@glk callid _vararg_count retval`.
4. Returns `retval`.

---

### §F.8.5 `Dynam__String` (Index 47) — Dynamic String Assignment

**Purpose:** Sets a dynamic string (printing variable) to a given value. Dynamic
strings are referenced by `@(N)` in printed text and can be changed at run-time.

**Invoked by:** The `string` statement (`string N "text"` or `string N routine`).

**Signature:** `Dynam__String(num, val)`

**Parameters:**

- `num` — the printing variable number (0-based index).
- `val` — the new value: a string address, routine address, or any printable
  value.

**Behavior:**

1. Validates that `num` is in range: `0 <= num < #dynam_string_table-->0`.
   Error 40 if out of range.
2. Sets `(#dynam_string_table)-->(num + 1) = val`. The dynamic string table's
   first entry is the count; entries 1 onward are the current values.

**Dependencies:** `RT__Err`

---

## §F.9 Veneer Routine Dependencies

When the compiler marks a veneer routine as needed, it recursively marks all
routines that the first routine calls. The following table shows the complete
dependency graph for each veneer routine, separated by target platform.

### §F.9.1 Z-Machine Dependencies

| Routine | Depends On |
|---------|-----------|
| `WV__Pr` | `RT__TrPS`, `RT__Err` |
| `RV__Pr` | `RT__Err` |
| `CA__Pr` | `Z__Region`, `Cl__Ms`, `RT__Err` |
| `IB__Pr` | `RT__Err`, `RT__TrPS` |
| `IA__Pr` | `RT__Err`, `RT__TrPS` |
| `DB__Pr` | `RT__Err`, `RT__TrPS` |
| `DA__Pr` | `RT__Err`, `RT__TrPS` |
| `RA__Pr` | `CP__Tab` |
| `RA__Sc` | `RT__Err` |
| `OP__Pr` | `Z__Region` |
| `OC__Cl` | `Z__Region`, `RT__Err` |
| `Z__Region` | `Unsigned__Compare` |
| `Metaclass` | `Z__Region` |
| `Cl__Ms` | `RT__Err`, `Copy__Primitive` |
| `RT__ChT` | `RT__Err` |
| `RT__ChR` | `RT__Err` |
| `RT__ChG` | `RT__Err` |
| `RT__ChGt` | `RT__Err` |
| `RT__ChPS` | `RT__Err`, `RT__TrPS` |
| `RT__ChPR` | `RT__Err` |
| `RT__ChLDB` | `Unsigned__Compare`, `RT__Err` |
| `RT__ChLDW` | `Unsigned__Compare`, `RT__Err` |
| `RT__ChSTB` | `Unsigned__Compare`, `RT__Err` |
| `RT__ChSTW` | `Unsigned__Compare`, `RT__Err` |
| `RT__ChPrintC` | `RT__Err` |
| `RT__ChPrintA` | `Unsigned__Compare`, `RT__Err` |
| `RT__ChPrintS` | `RT__Err`, `Z__Region` |
| `RT__ChPrintO` | `RT__Err`, `Z__Region` |

### §F.9.2 Glulx Dependencies

| Routine | Depends On |
|---------|-----------|
| `PrintShortName` | `Metaclass` |
| `Print__PName` | `PrintShortName` |
| `WV__Pr` | `RA__Pr`, `RT__TrPS`, `RT__Err` |
| `RV__Pr` | `RA__Pr`, `RT__Err` |
| `CA__Pr` | `RA__Pr`, `RL__Pr`, `PrintShortName`, `Print__PName`, `Z__Region`, `Cl__Ms`, `Glk__Wrap`, `RT__Err` |
| `IB__Pr` | `RT__Err`, `RT__TrPS` |
| `IA__Pr` | `RT__Err`, `RT__TrPS` |
| `DB__Pr` | `RT__Err`, `RT__TrPS` |
| `DA__Pr` | `RT__Err`, `RT__TrPS` |
| `RA__Pr` | `OC__Cl`, `CP__Tab` |
| `RL__Pr` | `OC__Cl`, `CP__Tab` |
| `RA__Sc` | `OC__Cl`, `RT__Err` |
| `OP__Pr` | `RA__Pr`, `Z__Region` |
| `OC__Cl` | `RA__Pr`, `RL__Pr`, `Z__Region`, `RT__Err` |
| `Copy__Primitive` | `CP__Tab` |
| `Z__Region` | `Unsigned__Compare` |
| `CP__Tab` | `Z__Region` |
| `Metaclass` | `Z__Region` |
| `Cl__Ms` | `OC__Cl`, `OP__Pr`, `RT__Err`, `Copy__Primitive`, `OB__Remove`, `OB__Move` |
| `RT__ChG` | `RT__Err` |
| `RT__ChGt` | `RT__Err` |
| `RT__ChR` | `RT__Err`, `Z__Region`, `OB__Remove` |
| `RT__ChT` | `RT__Err`, `Z__Region`, `OB__Move` |
| `RT__ChPS` | `RT__Err`, `RT__TrPS`, `WV__Pr` |
| `RT__ChPR` | `RT__Err`, `RV__Pr` |
| `RT__ChLDB` | `Unsigned__Compare`, `RT__Err` |
| `RT__ChLDW` | `Unsigned__Compare`, `RT__Err` |
| `RT__ChSTB` | `Unsigned__Compare`, `RT__Err` |
| `RT__ChSTW` | `Unsigned__Compare`, `RT__Err` |
| `RT__ChPrintC` | `RT__Err` |
| `RT__ChPrintA` | `Unsigned__Compare`, `RT__Err`, `Print__Addr` |
| `RT__ChPrintS` | `RT__Err`, `Z__Region` |
| `RT__ChPrintO` | `RT__Err`, `Z__Region` |
| `Print__Addr` | `RT__Err` |
| `Dynam__String` | `RT__Err` |

---

## §F.10 Language Constructs and Their Veneer Routines

This section maps Inform 6 language constructs to the veneer routines the
compiler invokes when compiling them.

| Language Construct | Veneer Routine(s) |
|-------------------|-------------------|
| `box "text"` | `Box__Routine` |
| `<<Action noun second>>` | `R_Process` |
| `print (the) obj` | `DefArt` |
| `print (a) obj` | `InDefArt` |
| `print (The) obj` | `CDefArt` |
| `print (A) obj` | `CInDefArt` |
| `print (name) obj` | `PrintShortName` |
| `print (number) n` | `EnglishNumber` |
| `print (property) id` | `Print__PName` |
| `print (char) c` | `RT__ChPrintC` (strict) |
| `print (address) a` | `RT__ChPrintA` (strict) / `Print__Addr` (Glulx, non-strict) |
| `print (string) a` | `RT__ChPrintS` (strict) |
| `print (object) a` | `RT__ChPrintO` (strict) |
| `obj.prop` (read, individual) | `RV__Pr` |
| `obj.prop` (read, common, strict) | `RT__ChPR` |
| `obj.prop = val` (write, individual) | `WV__Pr` |
| `obj.prop = val` (write, common, strict) | `RT__ChPS` |
| `obj.&prop` | `RA__Pr` |
| `obj.#prop` | `RL__Pr` |
| `obj.prop(args)` | `CA__Pr` |
| `++obj.prop` | `IB__Pr` |
| `obj.prop++` | `IA__Pr` |
| `--obj.prop` | `DB__Pr` |
| `obj.prop--` | `DA__Pr` |
| `Class::prop` | `RA__Sc` |
| `obj provides prop` | `OP__Pr` |
| `obj ofclass cla` | `OC__Cl` |
| `metaclass(x)` | `Meta__class` |
| `move obj to dest` (strict) | `RT__ChT` |
| `remove obj` (strict) | `RT__ChR` |
| `give obj attr` (strict) | `RT__ChG` |
| `give obj ~attr` (strict) | `RT__ChGt` |
| `base->offset` (strict) | `RT__ChLDB` |
| `base-->offset` (strict) | `RT__ChLDW` |
| `base->offset = val` (strict) | `RT__ChSTB` |
| `base-->offset = val` (strict) | `RT__ChSTW` |
| `glk($selector, args)` | `Glk__Wrap` [Glulx] |
| `string N val` | `Dynam__String` [Glulx] |

---

## §F.11 Library Replacements

The standard library (6.12.8) replaces several veneer routines with more
capable versions. When the library is included, its definitions take priority
over the built-in veneer code. The following routines are typically replaced:

| Veneer Routine | Library Replacement Location | Enhancement |
|---------------|------------------------------|-------------|
| `R_Process` | `parserm.h` | Full action processing with `before`/`after` hooks |
| `DefArt` | `english.h` / `parserm.h` | Handles `proper`, `article` property, plurals |
| `InDefArt` | `english.h` / `parserm.h` | Handles indefinite articles per language |
| `CDefArt` | `english.h` / `parserm.h` | Capitalized definite articles |
| `CInDefArt` | `english.h` / `parserm.h` | Capitalized indefinite articles |
| `PrintShortName` | `parserm.h` | Supports `short_name` property |
| `EnglishNumber` | `english.h` | Prints numbers as English words |
| `Box__Routine` | `linklpa.h` | Enhanced box quote display |

Programs that do not include the standard library get the minimal veneer
implementations, which provide basic functionality sufficient for simple
programs.

---

## §F.12 The Initial Routine (`Main__`)

In addition to the 48 veneer routines, the compiler generates one more piece of
veneer code: the **initial routine**. This is the very first routine in the
output file, placed at the start of the code area. It is named `Main__` (with
two underscores) and serves as the entry point for the virtual machine.

**Behavior:**

1. [Z-machine] Calls `Main` (the user's entry-point routine) using `@call_vs`
   (v4+) or `@call` (v3), storing the result in a temporary variable. Then
   executes `@quit` to end the program.

2. [Glulx] Calls `Main` using `@call` with zero arguments, discarding the
   result. Then executes `@return 0`.

The initial routine exists because the VM begins execution with an empty stack
frame and specific constraints (e.g., it must end with `quit` rather than
`return` in Z-code v1–5). By wrapping `Main` in this trampoline, the
compiler avoids imposing these constraints on the user's `Main` routine.

This routine does not appear in the veneer routine table and cannot be replaced
via `Replace`.
