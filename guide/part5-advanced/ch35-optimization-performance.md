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

# Chapter 35: Optimization and Performance

The compiler provides a number of facilities for reducing story file size,
improving text compression, and eliminating unused code. These range from
text abbreviation systems and economy switches to dead code elimination
and memory layout tuning. This chapter documents each optimization
mechanism, the compiler switches and settings that control it, and
practical guidance for producing compact release builds.

Most of these optimizations are off by default. During development, the
unoptimized output is easier to debug; for release builds, combining
several of the techniques described here can yield significant size
reductions.


## 35.1 Abbreviation Optimization

The `Abbreviate` directive declares text abbreviations that the compiler
can substitute into compiled strings to reduce their encoded size. When
economy mode is enabled (see §35.2), the compiler replaces occurrences of
each abbreviation's text with a compact two-Z-character reference,
saving space whenever the original text would have cost more than two
Z-characters to encode.

### 35.1.1 Declaring Abbreviations

Abbreviations are declared at the top level of the source file using the
`Abbreviate` directive. Each argument is a string literal:

```inform6
Abbreviate ". " ", " "the " "The " "you " "and ";
```

Multiple strings may appear on a single `Abbreviate` line, or across
multiple directives. The order of declaration determines priority when
two abbreviations could apply to the same text span.

### 35.1.2 The Abbreviation Structure

Internally, the compiler maintains an array of abbreviation records. Each
entry has the following fields:

```c
typedef struct abbreviation_s {
    int value;      /* compiled string value */
    int quality;    /* Z-chars saved (original length - 2) */
    int freq;       /* frequency of use */
    int textpos;    /* position in abbreviations_text */
    int textlen;    /* length of abbreviation text */
} abbreviation;
```

The `quality` field is the key metric: it represents the number of
Z-characters saved each time the abbreviation is used, calculated as the
Z-character cost of the original text minus 2 (the fixed cost of an
abbreviation reference). If quality is zero or negative, the abbreviation
provides no benefit and the compiler issues a warning.

### 35.1.3 Creating Abbreviations: make_abbreviation()

The `make_abbreviation()` function processes each
`Abbreviate` argument. It performs the following steps:

1. Checks that the maximum abbreviation count (`$MAX_ABBREVS`) has not
   been exceeded.
2. If `economy_switch` is false (i.e. `-e` was not given), returns
   immediately — the abbreviation is parsed but not stored.
3. Computes the Z-character encoding cost of the abbreviation text.
4. Calculates quality as `(Z-char cost) - 2`.
5. If quality is less than or equal to zero, issues a warning that the
   abbreviation saves no space.
6. Stores the abbreviation in the abbreviation table with its text
   position, length, quality, and an initial frequency of zero.

### 35.1.4 Automatic Abbreviation Computation (-u)

The `-u` switch instructs the compiler to compute an optimal set of
abbreviations automatically, rather than relying on manually declared
ones. This process is thorough but computationally expensive — it
analyses every string in the game to find substrings that would yield the
greatest total compression.

The optimization algorithm works as follows:

1. **Candidate generation**: the compiler examines all game text to
   identify frequently occurring substrings. Three-letter blocks are used
   as seeds for finding longer candidate strings.

2. **Scoring**: each candidate is scored using the formula:

   ```
   score = (matches - 1) * (scrabble - 2)
   ```

   where `matches` is the number of times the substring appears across
   all game text, and `scrabble` is the Z-character encoding cost of the
   substring. The `- 1` accounts for the fact that one occurrence must
   remain unabbreviated (the abbreviation definition itself), and the
   `- 2` accounts for the cost of the abbreviation reference.

3. **Forced first abbreviations**: the substrings `". "` and `", "` are
   always included in the abbreviation set because they appear
   frequently in nearly all English-language interactive fiction and
   reliably save space.

4. **Selection**: candidates are ranked by score. The top-scoring
   candidates are selected up to the `$MAX_ABBREVS` limit.

5. **Overlap avoidance**: the `any_overlap()` function checks whether a
   newly selected abbreviation would conflict with one already chosen —
   for example, if `"the "` and `"there"` both appear as candidates, the
   compiler ensures that selecting one does not invalidate occurrences of
   the other. Overlapping candidates are skipped in favor of
   higher-scoring alternatives.

The algorithm produces good results but can be very slow on large games,
as it must examine every possible substring of every string in the
source. For large projects, it may be practical to run `-u` once,
capture the computed abbreviations from the compiler output, and then
declare them explicitly with `Abbreviate` in subsequent compilations.

Compiler invocation:

```
inform6 -eu game.inf    ! Economy mode + compute abbreviations
```

### 35.1.5 $MAX_ABBREVS

The `$MAX_ABBREVS` setting controls the maximum number of abbreviations
the compiler will use. The default is 64.

**[Z-machine]** The Z-machine architecture supports exactly 96
abbreviations, arranged in three groups of 32. This is a hard limit
imposed by the virtual machine: the two-byte abbreviation reference
encodes a group number (0–2) and an index within the group (0–31),
yielding 96 possible values. Setting `$MAX_ABBREVS` higher than 96 for
Z-code targets has no effect.

**[Glulx]** Glulx does not use Z-character abbreviation encoding.
String compression on Glulx is handled by Huffman encoding (see §35.2),
so the abbreviation system is primarily relevant to Z-code compilation.

### 35.1.6 Abbreviation Frequency Statistics (-f)

The `-f` switch causes the compiler to display frequency statistics for
each abbreviation after compilation. The output shows, for each declared
abbreviation:

- The abbreviation text
- The number of times it was substituted
- The total number of bytes saved

This is useful for evaluating the effectiveness of manually chosen
abbreviations and for pruning abbreviations that provide little benefit.

```
inform6 -f game.inf     ! Show abbreviation frequency statistics
```

When combined with `-u`, the `-f` output shows the performance of the
automatically computed abbreviation set:

```
inform6 -efu game.inf   ! Compute abbreviations, show frequencies
```


## 35.2 Economy Mode (-e)

The `-e` switch enables **economy mode**: the compiler actively uses
declared abbreviations during text compilation, substituting abbreviation
references for matching substrings in all compiled strings.

Without `-e`, `Abbreviate` directives are parsed and validated (so that
syntax errors are still caught), but no text substitution occurs. The
abbreviation table is populated but never consulted during string
encoding.

### 35.2.1 The economy_switch Variable

The `economy_switch` variable controls whether
abbreviation substitution is active:

- Initialized to `FALSE`.
- Set to `TRUE` when the `-e` switch is provided on the command line.
- Checked in `make_abbreviation()`: if `!economy_switch`,
  the function returns immediately without recording the abbreviation.
- Checked during text translation: when true, the
  compiler scans each text segment against the abbreviation table for
  potential substitution.

### 35.2.2 Text Compression Process

When economy mode is active and the compiler encounters a string to
encode, it performs the following for each position in the string:

1. Checks the current position against all declared abbreviations,
   longest first.
2. If a match is found and the abbreviation's quality is positive (i.e.
   the substitution saves space), the compiler emits a two-Z-character
   abbreviation reference instead of the full text.
3. The abbreviation's frequency counter is incremented.
4. The scan position advances past the matched text.

The quality metric ensures that only beneficial substitutions are made: an
abbreviation whose text encodes to two or fewer Z-characters would cost
the same or more as a reference, so it is skipped.

### 35.2.3 Switch Interactions

Several switches interact with economy mode:

| Switch | Effect |
|--------|--------|
| `-e` | Use manually declared abbreviations |
| `-u` | Compute optimal abbreviations automatically |
| `-e -u` | Compute and then use the computed abbreviations |
| `-f` | Display abbreviation frequency statistics |
| `-H` | **[Glulx]** Enable Huffman string compression (see §35.2.4) |

Without `-e`, the `-u` switch will still compute and display the optimal
abbreviation set, but they will not be applied to the compiled output.
Combining `-e` and `-u` is the recommended approach for maximum text
compression in Z-code builds.

### 35.2.4 Huffman Compression (-H)

**[Glulx]** Glulx uses a fundamentally different string compression
mechanism from the Z-machine. Instead of an abbreviation table, Glulx
strings are compressed using Huffman encoding, which assigns shorter bit
sequences to more frequently occurring characters and substrings.

The `-H` switch controls Huffman compression for Glulx targets. It is
enabled by default; passing `-H` disables it (the switch toggles the
setting). With Huffman compression enabled, the compiler:

1. Analyses the frequency of all characters and common substrings across
   all game text.
2. Builds a Huffman tree that assigns variable-length bit codes to each
   symbol.
3. Encodes all strings using the Huffman tree, and embeds the tree in the
   story file header so that the interpreter can decode strings at
   runtime.

Huffman compression is generally more effective than the Z-machine
abbreviation system, as it can exploit character-level frequency patterns
across the entire text corpus rather than relying on a fixed set of 96
substring abbreviations. For Glulx targets, Huffman compression is
normally sufficient and the Z-machine abbreviation system is not used.

### 35.2.5 Double-Space Contraction (-d and -d2)

The `-d` switch enables double-space contraction: wherever two
consecutive space characters appear in compiled text (typically after a
full stop), the compiler contracts them to a single space. This is a
minor space-saving measure that can affect a large number of strings in
text-heavy games.

The `-d2` variant extends the contraction to also apply after exclamation
marks and question marks, not just full stops.

These switches affect only the compiled output; the source text remains
unchanged. Note that double-space contraction changes the visual
formatting of the game's output, so it should be tested before use.


## 35.3 Dead Code Elimination

The compiler performs basic dead code elimination by tracking
the reachability of each statement during assembly. When a statement is
determined to be unreachable — because it follows an unconditional return,
jump, or other flow-terminating opcode — the compiler can skip it
entirely, reducing the size of the compiled output.

### 35.3.1 Execution State Tracking

The compiler maintains an execution state for the current point in the
code being assembled. This state is stored in the
`execution_never_reaches_here` variable and uses a set of
flag constants:

| Constant | Value | Meaning |
|----------|-------|---------|
| `EXECSTATE_REACHABLE` | 0 | Normal execution; code is compiled |
| `EXECSTATE_UNREACHABLE` | 1 | Execution cannot reach this line |
| `EXECSTATE_ENTIRE` | 2 | The entire statement block is unreachable |
| `EXECSTATE_NOWARN` | 4 | Suppress unreachability warnings (bit flag) |

The `EXECSTATE_NOWARN` flag is a bit flag that can be combined with the
other states using bitwise OR. This allows the compiler to mark code as
unreachable without generating a warning — useful for code patterns where
unreachable code is intentional, such as after a `switch` statement with
a `default` case that returns.

### 35.3.2 How Dead Code Detection Works

The assembler updates the execution state as it processes opcodes:

1. When the assembler encounters an opcode marked with the `Rf`
   (returns/final) flag — such as `return`, `rtrue`, `rfalse`, `quit`,
   or an unconditional `jump` — it sets `execution_never_reaches_here`
   to `EXECSTATE_UNREACHABLE`.

2. Subsequent statements are checked against this flag. If the execution
   state is unreachable, the compiler either skips the code or issues a
   warning (depending on the `EXECSTATE_NOWARN` flag).

3. When a label or branch target is encountered, the execution state is
   reset to `EXECSTATE_REACHABLE`, since a jump or branch from elsewhere
   in the code could transfer control to that point.

4. Certain control structures — such as `if`/`else` blocks where both
   branches return — propagate unreachability to the code that follows
   them.

### 35.3.3 $STRIP_UNREACHABLE_LABELS

The `$STRIP_UNREACHABLE_LABELS` setting controls
whether labels in unreachable code are themselves stripped:

- **Default: 1 (on)**. When enabled, labels that appear in unreachable
  code are skipped entirely. If any code elsewhere attempts to jump to a
  stripped label, the compiler reports a compilation error, since the
  target no longer exists.

- **Value 0 (off)**. All labels are compiled into the output even if
  they appear in unreachable code. This ensures that jumps to those
  labels remain valid, at the cost of retaining some dead code.

The stripping logic checks three conditions before
removing a label:

1. The label is not referenced by any reachable jump or branch.
2. The current execution state includes `EXECSTATE_ENTIRE` (the entire
   block is unreachable, not just a single statement).
3. `$STRIP_UNREACHABLE_LABELS` is set to 1.

If all three conditions hold, the label is skipped. This is generally
safe for well-structured code, but can cause errors in code that uses
computed jumps or other unusual control flow patterns. In such cases,
setting `$STRIP_UNREACHABLE_LABELS=0` preserves all labels.

### 35.3.4 Unused Routine Detection

The compiler can detect and optionally remove routines that are never
called. Two settings control this behavior:

**`$WARN_UNUSED_ROUTINES`** controls warning generation:

| Value | Behavior |
|-------|-----------|
| 0 | No warnings about unused routines (default) |
| 1 | Warn about unused routines in game code only |
| 2 | Warn about all unused routines, including library routines |

Setting this to 1 is useful during development to identify dead code in
the game source. Setting it to 2 is more aggressive and will also flag
library routines that are included but never called — this can be noisy
for standard library builds, since the library defines many routines that
are only used in specific configurations.

**`$OMIT_UNUSED_ROUTINES`** controls code generation:

| Value | Behavior |
|-------|-----------|
| 0 | Compile all routines regardless of usage (default) |
| 1 | Omit routines that are never called from the compiled output |

When set to 1, the compiler performs a reachability analysis after
parsing all source code. Any routine that is not reachable from the
program's entry points (the `Main` routine, action routines, property
routines, timers, daemons, and other entry points) is excluded from the
compiled story file. This can significantly reduce story file size,
particularly in games that include the full standard library but use only
a subset of its features.

These settings can be specified on the command line or in source file
headers:

```
$WARN_UNUSED_ROUTINES=2
$OMIT_UNUSED_ROUTINES=1
```

Or in the source file:

```inform6
!% $WARN_UNUSED_ROUTINES=2
!% $OMIT_UNUSED_ROUTINES=1
```

It is common to use `$WARN_UNUSED_ROUTINES=1` during development (to
identify dead code) and `$OMIT_UNUSED_ROUTINES=1` in release builds (to
remove it).

Note that the unused routine analysis is conservative: if a routine's
address is taken (e.g. stored in a variable or property), it is
considered reachable even if it is never explicitly called. This prevents
the compiler from incorrectly removing routines that are invoked
indirectly.


## 35.4 Memory Setting Tuning

The compiler uses a set of configurable memory settings that
control the sizes of internal tables, buffers, and limits. These settings
can be adjusted to accommodate larger or more complex games, or to
fine-tune the compiler's resource usage.

### 35.4.1 Specifying Memory Settings

Memory settings are specified using the `$SETTING=value` syntax, either
on the command line or in source file headers:

On the command line:

```
inform6 $MAX_DYNAMIC_STRINGS=200 $MAX_STACK_SIZE=8192 game.inf
```

In source file headers (must appear before any Inform 6 directives):

```inform6
!% $MAX_DYNAMIC_STRINGS=200
!% $MAX_STACK_SIZE=8192
```

Settings specified on the command line override those in source file
headers. See §12.1 for the full syntax of compiler switches and ICL
commands.

### 35.4.2 Key Memory Settings

The following table lists the most commonly tuned memory settings:

| Setting | Default (Z-code) | Default (Glulx) | Description |
|---------|------------------|-----------------|-------------|
| `MAX_ABBREVS` | 64 | 64 | Maximum number of text abbreviations |
| `HASH_TAB_SIZE` | 512 | 512 | Size of the symbol hash table |
| `MAX_DYNAMIC_STRINGS` | 32 | 100 | Maximum number of `@NN` dynamic strings |
| `MAX_STACK_SIZE` | N/A | 4096 | Glulx interpreter stack size (in bytes) |
| `MAX_LOCAL_VARIABLES` | 16 | 118 | Maximum local variables per routine (obsolete in 6.44; compiled-in limit) |
| `MEMORY_MAP_EXTENSION` | N/A | 0 | Extra zero-bytes appended to Glulx story file |
| `DICT_WORD_SIZE` | 6 (V3) / 9 (V5+) | varies | Dictionary word resolution length |

### 35.4.3 Local Variable Limits

**[Z-machine]** The Z-machine architecture imposes a hard limit of 16
local variables per routine. This is encoded in the routine header as a
4-bit field, allowing values 0–15. In practice, one local is reserved for
the implicit result of certain operations, leaving 15 usable locals. The
`$MAX_LOCAL_VARIABLES` setting cannot exceed 16 for Z-code targets.

**[Glulx]** Glulx supports a much larger number of local variables per
routine. The `$MAX_LOCAL_VARIABLES` setting defaults to 119 and can be
increased further. Each local variable occupies 4 bytes in the routine's
stack frame, so very large numbers of locals increase stack consumption.
The `$MAX_STACK_SIZE` setting may need to be increased correspondingly.

### 35.4.4 When to Adjust Settings

The compiler reports specific error messages when a setting is too small
for the game being compiled. Common situations include:

- **"Too many abbreviations"**: increase `$MAX_ABBREVS` (or reduce the
  number of `Abbreviate` directives). Note the Z-machine limit of 96.
- **"Too many dynamic strings"**: increase `$MAX_DYNAMIC_STRINGS` when
  the game uses many `@NN` print variable references.
- **"Stack size exceeded"**: **[Glulx]** increase `$MAX_STACK_SIZE` for
  games with deep recursion or many simultaneous local variables.
- **"Too many local variables"**: refactor the routine to use fewer
  locals, or (on Glulx) increase `$MAX_LOCAL_VARIABLES`.

In general, the defaults are adequate for most games. Large or complex
projects — particularly those with extensive use of library extensions,
deep object hierarchies, or procedural text generation — may need to
increase several settings.

### 35.4.5 Hash Table Size

The `$HASH_TAB_SIZE` setting controls the size of the hash table used for
symbol lookup during compilation. A larger table reduces the likelihood
of hash collisions, which can speed up compilation for games with very
large numbers of symbols (objects, routines, constants, variables, etc.).

The default of 512 is sufficient for most games. For games with thousands
of symbols, increasing this to 1024 or 2048 can reduce compilation time
by improving symbol lookup performance. The value should be a power of 2
for optimal hash distribution.


## 35.5 Story File Size Reduction Techniques

This section provides a practical overview of all available optimization
techniques, summarizing the mechanisms described in this chapter and
elsewhere in the guide. The goal is to help authors produce the smallest
possible story file for release builds.

### 35.5.1 Optimization Checklist

The following techniques can be combined for maximum size reduction.
They are listed roughly in order of impact:

1. **Abbreviations** (§35.1): Use the `-u` switch to compute an optimal
   abbreviation set, and `-e` to enable abbreviation substitution.
   Alternatively, declare abbreviations manually based on `-f` frequency
   analysis. On Z-code targets, up to 96 abbreviations can be used.

2. **Economy mode** (§35.2): Always enable `-e` for release builds.
   Without it, no abbreviation substitution occurs and one of the most
   effective compression mechanisms is wasted.

3. **Unused routine omission** (§35.3.4): Set `$OMIT_UNUSED_ROUTINES=1`
   to exclude routines that are never called. This is particularly
   effective when using the full standard library, which defines many
   routines that may not be needed by a specific game.

4. **Symbol table omission** (§35.6.1): Set `$OMIT_SYMBOL_TABLE=1` to
   remove debug symbol names from the compiled output. This can save
   5–15% of total story file size, depending on the number and length of
   symbol names.

5. **Dictionary data reduction** (§35.6.2): **[Z-machine]** Set
   `$ZCODE_LESS_DICT_DATA=1` to save one byte per dictionary entry.

6. **Compact globals** (§35.7): **[Z-machine]** Set
   `$ZCODE_COMPACT_GLOBALS=1` to reclaim unused global variable slots
   for array storage.

7. **Double-space contraction** (§35.2.5): Use `-d` (or `-d2`) to
   contract consecutive spaces after full stops (and optionally after
   exclamation and question marks).

8. **Huffman compression** (§35.2.4): **[Glulx]** Enabled by default.
   Compresses all strings using Huffman encoding for efficient storage.

### 35.5.2 Release Build Example

A release build command incorporating multiple optimizations:

```
inform6 -esu $OMIT_UNUSED_ROUTINES=1 $OMIT_SYMBOL_TABLE=1 game.inf
```

This command:

- `-e` — enables economy mode (abbreviation substitution)
- `-s` — displays compilation statistics (useful for verifying size)
- `-u` — computes optimal abbreviations automatically
- `$OMIT_UNUSED_ROUTINES=1` — removes unreachable routines
- `$OMIT_SYMBOL_TABLE=1` — strips debug symbol names

For Z-code targets with additional size pressure, add dictionary
optimizations:

```
inform6 -esu $OMIT_UNUSED_ROUTINES=1 $OMIT_SYMBOL_TABLE=1 \
    $ZCODE_LESS_DICT_DATA=1 $ZCODE_COMPACT_GLOBALS=1 game.inf
```

### 35.5.3 Measuring the Impact

Use the `-s` switch to display compilation statistics, which include the
total story file size and the sizes of major sections (strings, code,
dictionary, objects, etc.). Comparing the output of `-s` between
optimized and unoptimized builds shows the impact of each technique.

Use the `-f` switch (see §35.1.6) to evaluate abbreviation effectiveness
and identify abbreviations that contribute little to compression.

The `$WARN_UNUSED_ROUTINES=1` setting (see §35.3.4) identifies routines
that would be removed by `$OMIT_UNUSED_ROUTINES=1`, allowing the author
to assess the impact before enabling routine omission.


## 35.6 $OMIT_SYMBOL_TABLE and $ZCODE_LESS_DICT_DATA

These two compiler settings remove optional data from the compiled story
file, reducing its size at the cost of some debugging capability or
dictionary flexibility.

### 35.6.1 $OMIT_SYMBOL_TABLE

The `$OMIT_SYMBOL_TABLE` setting controls whether
the compiler includes symbolic debugging information in the story file.

| Value | Behavior |
|-------|-----------|
| 0 | Include all debug symbol names in the story file (default) |
| 1 | Omit all debug symbol names from the story file |

When set to 1, the compiler strips the names of all functions, global
variables, constants, arrays, and other symbols from the compiled output.
The resulting story file is smaller but contains no symbolic information
for runtime debugging tools.

**What is removed:**

- Routine names (used by debug verbs like `showobj` and `showverb`)
- Global variable names
- Constant names
- Array names
- Class and object internal names (where stored symbolically)

**What is not affected:**

- Object short names (the printed names of in-game objects) — these are
  game content, not debug symbols
- Dictionary words — these are part of the parser vocabulary
- All game logic and behavior — stripping symbols has no effect on
  gameplay

**Typical size savings:** 5–15% of total story file size, depending on
the number and average length of symbols in the game. Games with many
long routine and constant names see larger savings.

**Trade-off:** with `$OMIT_SYMBOL_TABLE=1`, runtime debugging is
effectively impossible. Debug verbs that display routine names or
variable names will show numeric addresses instead. This setting should
be used only for final release builds.

Usage:

```
inform6 $OMIT_SYMBOL_TABLE=1 game.inf
```

Or in source:

```inform6
!% $OMIT_SYMBOL_TABLE=1
```

### 35.6.2 $ZCODE_LESS_DICT_DATA

**[Z-machine]** only.

The `$ZCODE_LESS_DICT_DATA` setting reduces the
size of each dictionary entry by one byte.

| Value | Behavior |
|-------|-----------|
| 0 | Standard dictionary entry format (default) |
| 1 | Reduced dictionary entry format (one fewer data byte) |

**Dictionary entry format:** each dictionary entry consists of encoded
text followed by data bytes that store flags and grammatical information
(noun/verb/preposition status, verb number, etc.):

| Version | Encoded text | Data bytes (normal) | Data bytes (reduced) | Total (normal) | Total (reduced) |
|---------|-------------|--------------------|--------------------|---------------|----------------|
| V3 | 4 bytes | 3 bytes | 2 bytes | 7 bytes | 6 bytes |
| V5+ | 6 bytes | 3 bytes | 2 bytes | 9 bytes | 8 bytes |

The byte length formula is:

```
entry_length = ((version == 3) ? 4 : 6) + (ZCODE_LESS_DICT_DATA ? 2 : 3)
```

The omitted byte is the third data/flags byte in each dictionary entry.
This byte is used for additional grammatical information; games that do
not rely on it can safely enable this setting.

**Size savings:** for a game with 1000 dictionary entries, enabling
`$ZCODE_LESS_DICT_DATA` saves approximately 1000 bytes (one byte per
entry). For games with large vocabularies (2000+ words), the savings are
proportionally larger.

**Compatibility note:** this setting changes the dictionary entry format,
which may affect library code or extensions that access dictionary entry
data bytes directly. The standard library is compatible with this
setting, but custom code that reads the third data byte of dictionary
entries will need to be updated.

Usage:

```
inform6 $ZCODE_LESS_DICT_DATA=1 game.inf
```

Or in source:

```inform6
!% $ZCODE_LESS_DICT_DATA=1
```


## 35.7 Compact Globals

**[Z-machine]** only.

The `$ZCODE_COMPACT_GLOBALS` setting enables a more
efficient memory layout for the global variable area in Z-code story
files.

### 35.7.1 Standard Global Variable Layout

The Z-machine reserves a contiguous block of memory for global variables,
starting at the address stored in the story file header. The architecture
supports up to 240 global variables (numbered 0x10 through 0xFF in the
variable encoding scheme), each occupying 2 bytes. This means the global
variable block is always 480 bytes long, regardless of how many globals
the game actually uses.

Following the global variable block, the compiler places arrays and other
dynamic data. In the standard layout, arrays start at offset 0x1E0
(decimal 480) from the beginning of the global variable area.

### 35.7.2 Compact Layout

When `$ZCODE_COMPACT_GLOBALS` is set to 1, the compiler changes this
layout:

| Value | Behavior |
|-------|-----------|
| 0 | Standard layout: 480-byte global block, arrays follow (default) |
| 1 | Compact layout: arrays begin immediately after the last used global |

Instead of reserving the full 480 bytes for globals, the compiler counts
the actual number of global variables used by the game and places arrays
immediately after the last one. The unused global variable slots are
reclaimed for array storage.

For example, if a game uses 50 global variables (100 bytes), arrays begin
at offset 100 instead of offset 480, saving 380 bytes in the dynamic
memory area.

### 35.7.3 Implementation Details

The compact layout requires adjustments in several parts of the compiler:

- **Array allocation**: the starting
  offset for arrays is calculated based on the actual number of globals
  rather than the fixed maximum of 240.
- **Backpatching**: references to array addresses are
  adjusted to account for the compact layout. The backpatcher must use
  the correct base offset when resolving symbolic references to array
  elements.
- **Table generation**: the final story file
  tables are assembled using the compact offsets.

The Z-machine itself is unaware of the compact layout; it accesses
globals by index (0x10–0xFF) and the interpreter translates these to
memory addresses using the header's global variable table pointer. The
compact layout works because the interpreter only accesses the global
slots that exist — unused slots beyond the last defined global are never
read or written. The space they would have occupied is now used for
arrays, which are accessed by absolute address rather than by global
variable index.

### 35.7.4 When to Use Compact Globals

This optimization is most effective for games that use relatively few
global variables. The standard library defines a substantial number of
globals, so the savings are typically modest (50–200 bytes) for games
that include the library. Games that use a custom minimal library or no
library at all see larger benefits.

The setting has no effect if the game uses all 240 global variable slots
(which is rare). It also has no effect on Glulx targets, which use a
different memory layout for global variables (see §30.3).

Usage:

```
inform6 $ZCODE_COMPACT_GLOBALS=1 game.inf
```

Or in source:

```inform6
!% $ZCODE_COMPACT_GLOBALS=1
```

### 35.7.5 Combining Compact Globals with Other Optimizations

Compact globals can be combined with all other Z-code optimization
settings. A fully optimized Z-code build might use:

```
inform6 -esu $OMIT_UNUSED_ROUTINES=1 $OMIT_SYMBOL_TABLE=1 \
    $ZCODE_LESS_DICT_DATA=1 $ZCODE_COMPACT_GLOBALS=1 game.inf
```

This combines abbreviation optimization (`-eu`), unused routine omission,
symbol table stripping, dictionary data reduction, and compact globals
for the smallest possible Z-code story file. The `-s` switch displays
statistics that can be used to verify the cumulative impact.
