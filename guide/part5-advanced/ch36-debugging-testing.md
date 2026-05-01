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

# Chapter 36: Debugging and Testing

The language provides a layered set of debugging and testing facilities
that span compile time, link time, and runtime. At the compiler level,
switches control which debugging infrastructure is included in the
compiled story file: the `DEBUG` constant, strict runtime error checking,
and the Infix source-level debugger. At runtime, the library provides a
suite of debug verbs for inspecting parser state, object properties,
scope, and the object tree. The compiler can also emit an XML debug
information file for use by external debugger tools, and a variety of
trace directives and switches allow fine-grained inspection of the
compilation process itself.

This chapter documents each of these facilities, the trade-offs they
impose on story file size and performance, and practical strategies for
using them together during development.


## 36.1 Compile-Time Debugging Switches

The compiler provides three principal switches that control
the inclusion of debugging infrastructure in the compiled story file.
Each corresponds to a Boolean variable in the compiler's switch state,
toggled from the command line or from `!%` directives in source code.


### 36.1.1 -D (Define DEBUG)

The `-D` switch causes the compiler to define the `DEBUG` constant
before compilation begins. The compiler's internal `define_DEBUG_switch`
variable defaults to `FALSE`. Passing `-D` on the command line sets
it to `TRUE`, which in turn defines the symbol `DEBUG` as a
compile-time constant.

The primary effect is to enable conditional compilation blocks throughout
the library that are guarded by `#Ifdef DEBUG`. Most importantly, the
library's `grammar.h` file uses this guard to conditionally include all
debug verb definitions:

```inform6
#Ifdef DEBUG;
Verb meta 'trace'
    *                               -> TraceOn
    * number                        -> TraceLevel
    * 'on'                          -> TraceOn
    * 'off'                         -> TraceOff;

Verb meta 'actions'
    *                               -> ActionsOn
    * 'on'                          -> ActionsOn
    * 'off'                         -> ActionsOff;

! ... showobj, showverb, scope, tree, gonear, purloin, abstract, etc.
#Endif;
```

When compiling without `-D`, none of the debug verbs (`trace`,
`showobj`, `showverb`, `scope`, `tree`, `gonear`, `purloin`,
`abstract`, `actions`, `routines`, `timers`, `changes`, `showdict`)
exist in the compiled game. This saves significant space — both the
grammar table entries and the implementing routines are omitted
entirely.

To enable debug verbs from within source code rather than the command
line:

```inform6
!% -D
```

This directive must appear before any library inclusion.

> **Note:** The `-D` switch only defines the `DEBUG` constant. It does
> not by itself enable strict runtime checking (see §36.1.2) or the
> Infix debugger (see §36.1.3). These are independent switches.


### 36.1.2 -S (Runtime Error Checking — STRICT Mode)

The `-S` switch controls the `runtime_error_checking_switch` variable,
which defaults to `TRUE`. This means strict runtime checking is
**enabled by default**. To disable it, use `-~S` on the command line.

When enabled, the compiler inserts runtime bounds-checking and
type-checking code at various points during code generation. The checks
cover several
categories:

**Array bounds checking.** Every array access is preceded by a check
that the index is within the valid range (0 to length−1 for `->` and
`-->` arrays, 1 to length for table arrays). Out-of-bounds accesses
generate a runtime error message rather than silently corrupting memory.

**Property access validation.** When reading or writing an individual
property of an object, the compiler inserts code to verify that the
property actually exists on that object. Accessing a nonexistent
individual property produces a runtime error rather than returning an
undefined value.

**Object validity checking.** Operations that take an object as an
argument (such as `move`, `remove`, `parent`, `child`, `sibling`,
`objectloop`, and property/attribute access) verify that the object
number is valid — not zero and within the range of defined objects.

**Indirect call validation.** When calling a routine through a variable
(`indirect(addr)` or the `addr()` syntax), the compiler checks that the
address is a valid routine entry point.

When a runtime error is detected, the library prints a diagnostic
message that includes the error type and, where possible, the source
location:

```
[** Programming error: tried to access property 47 of nothing (0) **]
[** Programming error: array index 15 out of bounds (0..9) **]
```

The trade-off is straightforward: strict mode adds code size overhead
(typically 5–15% depending on the program) but catches many common bugs
that would otherwise manifest as mysterious behavior. The recommended
practice is to leave `-S` enabled (the default) during development and
disable it with `-~S` only for final release builds where code size is
critical.

```
inform6 -~S game.inf    ! Disable strict checking for release
```


### 36.1.3 -X (Infix Debugger)

The `-X` switch controls the `define_INFIX_switch` variable, which
defaults to `FALSE`. Passing `-X` on the command line sets this
variable to `TRUE`, which defines the `INFIX` constant and enables the
Infix interactive debugger.

The Infix debugger is a substantially more powerful debugging facility
than the standard debug verbs; it is described in detail in §36.2.

The `-X` switch implies `-D` — enabling Infix automatically enables
`DEBUG` as well, since the Infix debugger depends on the debug verb
infrastructure.

```
inform6 -X game.inf    ! Enable Infix debugger (implies -D)
```


## 36.2 The Infix Debugger

The Infix debugger is an interactive source-level debugger that is built
into the compiled story file. Unlike the standard debug verbs (§36.3),
which provide discrete inspection commands, Infix provides a unified
debugging environment with the ability to inspect and modify any object,
property, attribute, global variable, array, or routine at runtime.

Infix is enabled by the `-X` compiler switch (see §36.1.3), which
defines the `INFIX` constant. The debugger has both a compiler-side
component (code generated by the compiler to support runtime inspection)
and a library-side component (the `infix.h` header file that implements
the interactive interface).


### 36.2.1 Compiler-Side Effects

When `define_INFIX_switch` is `TRUE`, the compiler makes several
modifications to the generated code:

**The `infix__watching` attribute.** The compiler creates a special
attribute named `infix__watching` that can be set on
any object to enable change tracking for that object. This attribute
is allocated at attribute slot 0, which means it is reserved and
unavailable for normal use.

> **[Z-machine]** With `infix__watching` occupying slot 0, only 47
> attributes remain available for the program (down from the normal 48).

> **[Glulx]** On Glulx, the attribute space is much larger (4096
> slots), so reserving slot 0 for `infix__watching` leaves 4095
> attributes for normal use — rarely a practical constraint.

**Property modification tracing.** When Infix is enabled, the compiler
modifies the veneer routines to include property-change tracing. The
`RT__TrPS` (trace property set) veneer function is called whenever a property is modified on a watched
object. This function reports the property name, old value, and new
value.

**Instruction-level tracing.** When Infix is enabled, the compiler
generates additional code to track execution points. This
enables Infix to report which routines are being entered and exited,
and to support single-stepping through code.

**Symbol table preservation.** The Infix debugger requires access to
the names of objects, properties, attributes, global variables, arrays,
and routines at runtime. The compiler includes additional symbol table
data in the story file to make this possible.


### 36.2.2 Library-Side Support (infix.h)

The library file `infix.h` implements the interactive debugging
interface. It is included conditionally:

```inform6
#Ifdef INFIX;
Include "infix";
#Endif;
```

The `infix.h` file defines routines for:

- **Object inspection:** displaying all properties, attributes, and
  the object tree position of any object by name or number
- **Property modification:** changing property values at runtime
- **Attribute manipulation:** setting and clearing attributes on any
  object
- **Global variable inspection and modification:** reading and changing
  the value of any global variable by name
- **Routine invocation:** calling any routine with specified arguments
- **Watchpoint management:** setting and clearing watchpoints on objects
  (via the `infix__watching` attribute) to monitor property changes

The Infix debugger integrates with the parser to add its own set of
debugging verbs, which supplement the standard debug verbs described in
§36.3.


### 36.2.3 The `debug_flag` Variable

The `debug_flag` global variable controls runtime debugging behavior.
Individual bits enable or disable different categories of tracing (see
§36.3.1 for the bit definitions). The Infix system can manipulate
`debug_flag` programmatically to enable or disable tracing features as
part of a debugging session.


### 36.2.4 Trade-Offs

Enabling Infix has a significant impact on the compiled story file:

- **Code size increase:** the additional tracing code, symbol tables,
  and Infix library routines add substantial bulk to the story file.
  This can be especially problematic on the Z-machine, where story file
  size limits are tight (see §36.5.4).
- **Attribute slot consumption:** reserving slot 0 for
  `infix__watching` reduces the available attribute space.
- **Performance overhead:** instruction-level tracing code runs even
  when not actively debugging, adding a constant performance cost.

Infix should only be used during active debugging sessions. It should
never be enabled in release builds. For most debugging tasks, the
standard debug verbs (§36.3) combined with strict mode (§36.1.2) are
sufficient.


## 36.3 Debug Verbs

The library defines a comprehensive set of debug verbs that
are available at runtime when the game is compiled with `-D`. All debug
verb definitions are guarded by `#Ifdef DEBUG` in `grammar.h`
and their implementing routines appear in `verblib.h` and `parser.h`.

All debug verbs are declared as `meta` verbs, which means they do not
consume a game turn — the turn counter is not incremented, and
daemons/timers are not run when a debug verb is executed.


### 36.3.1 Debug Flag Constants

The library defines a set of bit constants for the `debug_flag` global
variable. Each constant corresponds to a category of runtime tracing
that can be independently enabled or disabled:

```inform6
Constant DEBUG_MESSAGES  $0001;   ! Routine entry/exit tracing
Constant DEBUG_ACTIONS   $0002;   ! Action processing tracing
Constant DEBUG_TIMERS    $0004;   ! Timer/daemon firing tracing
Constant DEBUG_CHANGES   $0008;   ! Property/attribute change tracing
Constant DEBUG_VERBOSE   $0080;   ! Verbose mode flag

Global debug_flag;
```

These constants are defined in `parser.h`. The
`debug_flag` variable is a bitmask: multiple categories can be active
simultaneously by setting multiple bits.


### 36.3.2 Parser Tracing

**Verbs:** `trace`, `trace on`, `trace off`, `trace N`

**Grammar** (grammar.h):

```inform6
Verb meta 'trace'
    *                               -> TraceOn
    * number                        -> TraceLevel
    * 'on'                          -> TraceOn
    * 'off'                         -> TraceOff;
```

**Implementation:**

- `TraceOnSub` sets `parser_trace = 1`, enabling parser tracing at
  level 1.
- `TraceLevelSub` sets `parser_trace = noun`, where `noun` is the
  numeric argument. Trace levels range from 1 to 5, with higher levels
  producing increasingly detailed output.
- `TraceOffSub` sets `parser_trace = 0`, disabling parser tracing.

**Effect:** When parser tracing is active, the parser prints its
decision-making process for each line of player input. This includes:

- Tokenization of the input line into dictionary words
- Grammar line matching attempts and their results
- Object disambiguation decisions and scoring
- Scope queries and their results
- The final parsed action, noun, and second noun

Parser tracing is invaluable for diagnosing "I don't understand that
sentence" errors and disambiguation problems. Level 1 provides a
high-level overview; levels 2–5 add progressively more internal detail.

Example output at level 1:

```
> get key
[ "get" "key" ]
[ Trying verb 'get': line 0 token 1: noun ... ]
[ action Take noun key (27) ]
```


### 36.3.3 Action Listing

**Verbs:** `actions`, `actions on`, `actions off`

**Grammar** (grammar.h):

```inform6
Verb meta 'actions'
    *                               -> ActionsOn
    * 'on'                          -> ActionsOn
    * 'off'                         -> ActionsOff;
```

**Implementation:** `ActionsOnSub` sets the
`DEBUG_ACTIONS` bit (`$0002`) in `debug_flag`; `ActionsOffSub` clears
it.

**Effect:** When active, the library prints the action name, noun, and
second noun for every action processed through the action-handling
pipeline. This includes actions generated by the parser, `<action>`
statements in source code, and actions generated by `react_before` and
`react_after` processing.

Example output:

```
[ Action Take (noun: brass key) ]
[ Action Insert (noun: brass key) (second: wooden box) ]
```

Action tracing is useful for verifying that the correct action is being
generated and for tracking the flow of action processing through
`before`, `after`, and `react_before` rules. See §22 for the complete
action processing pipeline.


### 36.3.4 Routine/Message Listing

**Verbs:** `routines`, `messages`, `routines on`, `routines off`

**Grammar** (grammar.h):

```inform6
Verb meta 'routines' 'messages'
    *                               -> RoutinesOn
    * 'on'                          -> RoutinesOn
    * 'off'                         -> RoutinesOff;
```

**Implementation:** `RoutinesOnSub` sets the
`DEBUG_MESSAGES` bit (`$0001`) in `debug_flag`; `RoutinesOffSub` clears
it.

**Effect:** When active, the library prints the name of every routine
as it is entered and exited:

```
[Entering routine InformLibrary.play ...]
[Entering routine ChooseObjects ...]
[Exiting routine ChooseObjects (return value: 0)]
```

This is useful for understanding the call sequence during action
processing, parser execution, or any complex multi-routine interaction.
The output can be very verbose — a single turn may involve dozens of
routine calls — so this tracing is best used in targeted sessions.

> **Note:** The `routines` and `messages` verbs are synonyms; they
> both toggle the same `DEBUG_MESSAGES` flag.


### 36.3.5 Timer/Daemon Listing

**Verbs:** `timers`, `daemons`, `timers on`, `timers off`

**Grammar** (grammar.h):

```inform6
Verb meta 'timers' 'daemons'
    *                               -> TimersOn
    * 'on'                          -> TimersOn
    * 'off'                         -> TimersOff;
```

**Implementation:** `TimersOnSub` sets the
`DEBUG_TIMERS` bit (`$0004`) in `debug_flag`; `TimersOffSub` clears it.

**Effect:** When active, the library prints each timer and daemon as it
fires during the end-of-turn processing:

```
[Timer for clock: fired (time left: 3)]
[Daemon for guardian: running]
```

This is useful for verifying that timers are counting down correctly and
that daemons are running when expected. See §24 for the timer/daemon
system.


### 36.3.6 Change Tracking

**Verbs:** `changes`, `changes on`, `changes off`

**Implementation:** `ChangesOnSub` sets the `DEBUG_CHANGES` bit
(`$0008`) in `debug_flag`; `ChangesOffSub` clears it.

**Effect:** When active, the library reports whenever an object's
attributes or properties change:

```
[brass key: "moved" attribute set]
[wooden box: "number_of_items" property changed from 2 to 3]
```

Change tracking is useful for detecting unintended state modifications,
especially in complex games with many interacting objects and rules.


### 36.3.7 Object Inspector: showobj

**Verbs:** `showobj`, `showobj N`, `showobj <object>`

**Grammar** (grammar.h):

```inform6
Verb meta 'showobj'
    *                               -> ShowObj
    * anynumber                     -> ShowObj
    * multi                         -> ShowObj;
```

**Implementation:** `ShowObjSub` displays a
comprehensive dump of the specified object (or the current location if
no argument is given). The output includes:

- The object's class, internal name, object number, and parent object
- All attributes that are currently set on the object
- All properties with their current values, including inherited defaults
  and individual properties
- All children of the object (one level deep)

Example output:

```
> showobj brass key
Object "brass key" (27) in "Dusty Room" (12)
  has  light moved
  with name 'brass' 'key',
       description "A small brass key, slightly tarnished.",
       before NULL,
       after NULL,
       ...
  Children: (none)
```

`showobj` without an argument displays the current room. `showobj N`
displays the object with internal number N, which is useful when you
know the object number from other debugging output.


### 36.3.8 Verb Inspector: showverb

**Verbs:** `showverb <word>`

**Grammar** (grammar.h):

```inform6
Verb meta 'showverb'
    * special                       -> ShowVerb;
```

**Implementation:** `ShowVerbSub` displays the
complete grammar definition for the specified verb word. The output
includes:

- Whether the verb is `meta` (does not consume a turn)
- All dictionary words that map to the same verb number (synonyms)
- All grammar lines associated with the verb, showing each token and
  the target action for each line

Example output:

```
> showverb take
Verb 'take' (synonyms: 'get' 'carry' 'hold')
    * multi                         -> Take
    * multi 'from' noun             -> Remove
    * 'off' worn                    -> Disrobe
    * 'inventory'                   -> Inv
```

`showverb` is invaluable for diagnosing grammar conflicts and for
verifying that custom verb definitions have been correctly installed.
See §20 for the grammar system.


### 36.3.9 Dictionary Inspector: showdict

**Verbs:** `showdict`, `showdict <word>`

**Grammar** (grammar.h):

```inform6
Verb meta 'showdict' 'dict'
    *                               -> ShowDict
    * topic                         -> ShowDict;
```

**Implementation:** `ShowDictSub` displays
dictionary entries. Without an argument, it lists all words in the
dictionary. With a word argument, it displays the flags set on that
specific dictionary entry.

Dictionary word flags include:

- **verb** — the word heads a verb grammar definition
- **noun** — the word appears in an object's `name` property
- **plural** — the word is a plural synonym (from `plural` in `name`)
- **meta** — the word heads a meta verb
- **preposition** — the word is used as a grammar token (e.g., `'from'`,
  `'with'`)

> **[Z-machine]** Dictionary words are truncated to 6 characters in
> Z-machine versions 1–3 and 9 characters in versions 4–8. The
> `showdict` output reflects the truncated forms stored in the
> dictionary.

> **[Glulx]** Glulx dictionary words are stored at their full length
> (or a configurable maximum set by `$DICT_WORD_SIZE`).


### 36.3.10 Scope Inspector: scope

**Verbs:** `scope`, `scope N`, `scope <object>`

**Grammar** (grammar.h):

```inform6
Verb meta 'scope'
    *                               -> Scope
    * anynumber                     -> Scope
    * noun                          -> Scope;
```

**Implementation:** `ScopeSub` lists all objects
currently in scope. Without an argument, it lists objects in scope for
the player. With an object argument, it lists objects in scope from that
object's perspective.

Scope determines which objects the parser considers as potential
referents for nouns in the player's input. An object is in scope if it
is visible and reachable according to the rules described in §21.

Example output:

```
> scope
Objects in scope for player:
  Dusty Room (12)
  brass key (27)
  wooden box (31)
  old lamp (33)
```

The `scope` verb is essential for diagnosing "You can't see any such
thing" errors. If an object that should be accessible is not listed by
`scope`, the problem lies in the scope calculation — check `found_in`,
`add_to_scope`, container transparency, and the `light` attribute.


### 36.3.11 Object Tree: tree

**Verbs:** `tree`, `tree N`, `tree <object>`

**Grammar** (grammar.h):

```inform6
Verb meta 'tree'
    *                               -> XTree
    * anynumber                     -> XTree
    * noun                          -> XTree;
```

**Implementation:** `XTreeSub` displays the object
containment hierarchy. Without arguments, it shows all top-level objects
(objects with no parent). With an argument, it shows that object and all
its descendants, indented to reflect containment depth.

Example output:

```
> tree
Dusty Room (12)
  brass key (27)
  wooden box (31)
    old map (34)
  player (1)
    old lamp (33)
Dark Cellar (15)
  rusty sword (40)
```

The `tree` verb is useful for verifying object placement after complex
move operations, and for getting an overview of the game world structure.


### 36.3.12 Teleportation: gonear

**Verbs:** `gonear N`, `gonear <object>`

**Grammar** (grammar.h):

```inform6
Verb meta 'gonear'
    * anynumber                     -> GoNear
    * noun                          -> GoNear;
```

**Implementation:** `GoNearSub` moves the
player to the room that contains the specified object. The routine
walks up the object tree from the target object until it finds a room
(an object without a parent, or an object that `has light` and is at
the top level), then calls `PlayerTo` to move the player there.

`gonear` is useful for rapidly navigating to distant parts of the game
world during testing, without needing to type the sequence of movement
commands.

```
> gonear rusty sword
[Transporting to Dark Cellar]

Dark Cellar
A damp underground room...
```


### 36.3.13 Object Theft: purloin

**Verbs:** `purloin N`, `purloin <object>`

**Grammar** (grammar.h):

```inform6
Verb meta 'purloin'
    * anynumber                     -> XPurloin
    * multi                         -> XPurloin;
```

**Implementation:** `XPurloinSub` moves the
specified object directly into the player's inventory, bypassing all
`before` and `after` rules, capacity checks, and any other restrictions.

The routine:

1. Checks that the target is not the player object itself
2. Removes the object from its current position in the object tree
3. Moves it to the player
4. Sets the `moved` attribute on the object
5. Clears the `concealed` attribute (if set)
6. Prints a confirmation message

`purloin` is useful for acquiring objects that are otherwise
inaccessible — locked behind puzzles, in unreachable rooms, or guarded
by NPCs. Unlike the normal `Take` action, `purloin` never fails.

```
> purloin golden idol
[Purloined.]
```


### 36.3.14 Object Transfer: abstract

**Verbs:** `abstract N to N`, `abstract <object> to <object>`

**Grammar** (grammar.h):

```inform6
Verb meta 'abstract'
    * anynumber 'to' anynumber      -> XAbstract
    * noun 'to' noun                -> XAbstract;
```

**Implementation:** `XAbstractSub` moves the first
object into the second object, bypassing all `before` and `after`
rules. This is a more general version of `purloin` — it allows
moving any object to any other object, not just to the player.

```
> abstract golden idol to wooden box
[Abstracted.]
```

`abstract` is useful for setting up specific test scenarios: placing
objects in containers, moving NPCs to particular rooms, or constructing
game states that would be difficult to reach through normal play.


## 36.4 Debug File Format (gameinfo.dbg)

The compiler can generate an external debug information file
that records the mapping between compiled story file addresses and
source-level constructs (routine names, object names, source file
locations, etc.). This file is consumed by external debugger tools
that provide graphical source-level debugging.


### 36.4.1 Enabling Debug File Generation

The debug file is generated when the `-k` switch is used on the command
line. The output file is named `gameinfo.dbg`.

```
inform6 -k game.inf
```

This produces two output files: the story file (`game.z5` or
`game.ulx`) and the debug information file (`gameinfo.dbg`).


### 36.4.2 XML Format

The debug file is an XML document. The file begins with an XML
declaration and a root element that identifies the compiler version:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<inform-story-file version="1.0"
    content-creator="Inform"
    content-creator-version="6.44">
    <source index="0">
        <given-path>game.inf</given-path>
        <resolved-path>/path/to/game.inf</resolved-path>
        <language>Inform 6</language>
    </source>
    <!-- ... symbol, routine, object, and line data ... -->
</inform-story-file>
```

The compiler opens the file and writes the XML declaration and root
element. Subsequent compiler passes add child elements as compilation
proceeds.


### 36.4.3 Writing Functions

The compiler uses several internal functions to write the debug file:

- A `printf`-style function writes formatted text to the debug file,
  used for most debug file output.
- An entity-escaping function writes strings to the debug file,
  replacing XML special characters with their entity references:

  | Character | Entity     |
  |-----------|------------|
  | `"`       | `&quot;`   |
  | `&`       | `&amp;`    |
  | `'`       | `&apos;`   |
  | `<`       | `&lt;`     |
  | `>`       | `&gt;`     |

- Base64 encoding functions are available for embedding binary data
  (such as compiled routine bytecode) in the XML file.


### 36.4.4 Source File Tracking

Each source file included during compilation receives a `<source>`
element in the debug file. The element includes both the path as given
in the `Include` directive (`<given-path>`) and the resolved absolute
path (`<resolved-path>`). Source files are indexed sequentially,
starting from 0 for the main source file.

```xml
<source index="0">
    <given-path>game.inf</given-path>
    <resolved-path>/home/user/game/game.inf</resolved-path>
    <language>Inform 6</language>
</source>
<source index="1">
    <given-path>Parser</given-path>
    <resolved-path>/usr/share/inform6/lib/parser.h</resolved-path>
    <language>Inform 6</language>
</source>
```

Line number records in subsequent elements reference source files by
their index number.


### 36.4.5 Symbol and Address Data

The debug file contains elements that map between compiled addresses
and source-level constructs:

- **Routine records:** map each compiled routine to its source file,
  line number, and name
- **Object records:** map each object to its number, name, and source
  location
- **Global variable records:** map each global to its storage address
  and name
- **Array records:** map each array to its address, element count, and
  name
- **Line number records:** map compiled instruction addresses to source
  file line numbers, enabling source-level single-stepping


### 36.4.6 External Debugger Support

Several interactive fiction interpreters and development tools can
consume the `gameinfo.dbg` file to provide source-level debugging:

- Setting breakpoints at specific source lines or routine entry points
- Single-stepping through source code
- Inspecting and modifying variables, object properties, and attributes
- Viewing the call stack with source locations

The debug file is designed to be a complete and self-contained
description of the mapping between source and compiled code. The
debugger does not need access to the original source files to display
routine and object names (though source files are needed for
line-by-line display).


## 36.5 Common Bugs and Troubleshooting

This section documents common programming mistakes and the
techniques for diagnosing them. Many of these issues are caught
automatically by strict mode (§36.1.2); others require manual
investigation using the debug verbs (§36.3).


### 36.5.1 Syntax Errors

**Missing semicolons.** The single most common syntax error.
Because the compiler uses semicolons as statement terminators, a missing
semicolon causes the compiler to attempt to parse the next statement as
a continuation of the current one. The error is therefore reported at
the location of the *next* statement, not where the semicolon should
have been. When investigating a syntax error, always check the line
*above* the reported location.

```inform6
! Bug: missing semicolon after print
[ MyRoutine;
    print "Hello"        ! <- Missing semicolon here
    return true;         ! Error reported on this line
];
```

**Mismatched brackets.** Mismatched `[` `]` (routine delimiters) or
`{` `}` (block delimiters) cause the compiler to lose track of nesting.
A mismatched `[` can cause the compiler to treat the rest of the file
as part of a routine, producing cascading errors. When you see a large
number of errors after a point in the file, look for a missing closing
bracket near the first error.

**Assignment in conditions.** Using `=` (assignment) instead of `==`
(equality test) in a condition is a common mistake. Here, `=`
in a condition is syntactically valid but performs an assignment:

```inform6
! Bug: assigns 5 to x, always true
if (x = 5) print "x is five";

! Correct: tests equality
if (x == 5) print "x is five";
```

The compiler may warn about this pattern when strict mode (`-S`) is
enabled.


### 36.5.2 Object and Property Errors

**Confusing attributes and properties.** Attributes are Boolean flags
tested with `has`/`hasnt` and modified with `give`. Properties are
named values accessed with the `.` operator. A common mistake is using
`has` when you mean to test a property, or vice versa:

```inform6
! Bug: light is an attribute, not a property
if (lamp.light) ...

! Correct
if (lamp has light) ...
```

**Property access on the wrong object.** Accessing an individual
property (one not defined in a class or with a default) on an object
that does not provide it will produce a runtime error under strict mode:

```
[** Programming error: tried to access property 47 of "brass key" **]
```

Use `provides` to test before accessing (see §5):

```inform6
if (obj provides special_action)
    obj.special_action();
```

**Common property count limits.** The Z-machine allows a maximum of 63
common properties (numbered 1–63). Glulx has an effectively unlimited
number. If you exceed the Z-machine limit, the compiler will report an
error. Consider migrating to Glulx (see §28) or converting some common
properties to individual properties.

**Object tree modification during `objectloop`.** The `objectloop`
construct iterates over the object tree using `child`/`sibling` links.
If the loop body modifies the tree (by `move` or `remove` operations on
objects in the iteration set), the loop can skip objects or visit them
twice. Use a temporary variable to save the next sibling before
modifying the tree:

```inform6
! Safe pattern for tree-modifying loops
objectloop (obj in container) {
    next_obj = sibling(obj);
    if (should_remove(obj))
        remove obj;
    obj = next_obj;
}
```

Or collect objects into an array first, then process the array.


### 36.5.3 Scope and Visibility Issues

**Objects not in scope.** When the parser responds with "You can't see
any such thing" for an object that should be accessible, use the
`scope` debug verb (§36.3.10) to see what is actually in scope. Common
causes of scope problems include:

- The object is in a closed, non-transparent container (the `open`
  and `transparent` attributes control this)
- The object is in a dark room without a light source (the `light`
  attribute must be set on either the room or a visible object)
- The object's `found_in` property does not include the current room
- An `add_to_scope` routine is not adding the expected objects
- The object is a component (child of another non-container object)
  and is not explicitly added to scope

**Light calculation problems.** The library's light model is
straightforward but can produce unexpected results. A room is lit if
it `has light`, or if any object in scope `has light`. A container
or supporter provides light to its parent only if it is `open` or
`transparent`. Use `showobj` to verify which objects have `light` set,
and `scope` to verify which objects are visible.

See §21 for the complete scope rules and §23 for the light model.


### 36.5.4 Z-machine Limits

The Z-machine imposes hard limits on several resources. Exceeding any
of these limits produces a compiler error. The limits vary by Z-machine
version:

| Resource                  | V3    | V5    | V8       |
|---------------------------|-------|-------|----------|
| Story file size           | 128KB | 256KB | 512KB    |
| Maximum objects           | 255   | 65535 | 65535    |
| Global variables          | 240   | 240   | 240      |
| Attributes                | 32    | 48    | 48       |
| Common properties         | 31    | 63    | 63       |
| Dictionary word length    | 6     | 9     | 9        |
| Maximum verb count        | 255   | 255   | 255      |

> **[Glulx]** Glulx removes or greatly relaxes all of these limits.
> Story file size is limited only by available memory. Objects,
> attributes, properties, and dictionary words have effectively no
> practical upper bound. If you are hitting Z-machine limits, consider
> targeting Glulx instead (see §28).

When approaching the story file size limit, use the optimization
techniques described in §35 (abbreviations, economy mode, unused
routine omission, symbol table stripping) to reduce code size. Use the
`-s` statistics switch to monitor resource usage during development.


### 36.5.5 Runtime Error Messages

With strict mode enabled (the default; see §36.1.2), the library
generates runtime error messages when it detects programming errors.
These messages always begin with `[** Programming error:` and describe
the nature of the error:

```
[** Programming error: tried to access property 47 of nothing (0) **]
```

The object number 0 ("nothing") indicates a null object reference —
typically caused by using an uninitialized variable or the result of a
failed `objectloop`.

```
[** Programming error: array index 15 out of bounds (0..9) **]
```

An array access with an index outside the declared bounds. The range
shown is the valid range.

```
[** Programming error: tried to read from an illegal address **]
```

A memory access at an address that does not correspond to any valid
data structure. This usually indicates a corrupted pointer or an
uninitialized variable used as an address.

```
[** Programming error: "word" has no property xyz **]
```

An attempt to read an individual property that does not exist on the
specified object. Use `provides` to test before accessing.

All runtime error messages indicate bugs that should be investigated
and fixed before release. Even if the game appears to work correctly
despite the error, the underlying cause may produce more serious
problems in other situations or on other interpreters.


## 36.6 Testing Strategies

This section covers compile-time diagnostic tools and practical
methodologies for testing programs.


### 36.6.1 The Trace Directive

The `Trace` directive controls compile-time diagnostic output from
within source code. Unlike the runtime `trace` verb (§36.3.2), the
`Trace` directive operates during compilation and reveals the
compiler's internal processing.

The directive supports the following keywords:

**Assembly tracing:**

```inform6
Trace assembly;       ! Enable assembly output at level 1
Trace assembly 2;     ! Enable at level 2 (more detail)
Trace assembly 3;     ! Enable at level 3 (with hex dumps)
Trace assembly 4;     ! Maximum detail
```

When assembly tracing is active, the compiler prints the generated
Z-machine or Glulx instructions for each statement. This is useful for
understanding how high-level constructs map to virtual machine
instructions, and for diagnosing code generation issues.

**Expression tracing:**

```inform6
Trace expressions;    ! Enable expression parsing output at level 1
Trace expressions 2;  ! Level 2
Trace expressions 3;  ! Maximum detail
```

Shows the compiler's expression parser at work: operator precedence
decisions, type checking, and the expression tree structure.

**Token tracing:**

```inform6
Trace tokens;         ! Enable lexer token output at level 1
Trace tokens 2;       ! Level 2
Trace tokens 3;       ! Maximum detail
```

Shows the lexer's tokenization of the source file: each token's type,
value, and source location.

**Data structure inspection:**

```inform6
Trace dictionary;     ! Display the dictionary at this point
Trace objects;        ! Display the object tree at this point
Trace symbols;        ! Display the symbol table
Trace symbols 2;      ! Symbol table with more detail
Trace verbs;          ! Display the verb/grammar table
```

These forms print a snapshot of the compiler's internal data structures
at the point in the source where the directive appears. This is useful
for verifying that objects, dictionary words, symbols, and grammar have
been defined as expected up to that point in the compilation.

**Global toggle:**

```inform6
Trace on;             ! Enable all tracing
Trace off;            ! Disable all tracing
```

The `Trace` directive can be placed at any point in the source file.
A common technique is to bracket a problematic section with
`Trace assembly;` and `Trace off;` to inspect only the generated code
for that section:

```inform6
Trace assembly;
[ ProblematicRoutine x y;
    ! ... code to inspect ...
];
Trace off;
```


### 36.6.2 Trace Settings ($! Options)

The compiler supports a set of trace settings that can be specified on
the command line using the `$!` prefix. These correspond to the same
tracing categories available through the `Trace` directive, but are set
externally rather than embedded in source code.

The available settings are:

| Setting          | Switch  | Description                             |
|------------------|---------|-----------------------------------------|
| `$!ASM`          | `-a`    | Assembly trace level (0–4)              |
| `$!EXPR`         |         | Expression trace level (0–3)            |
| `$!TOKENS`       |         | Token trace level (0–3)                 |
| `$!BPATCH`       |         | Backpatching trace                      |
| `$!SYMDEF`       |         | Symbol definition trace                 |
| `$!FINDABBREVS`  |         | Abbreviation search trace               |
| `$!RUNTIME`      | `-g`    | Runtime function call tracing (0–3)     |
| `$!FILES`        |         | File inclusion trace                    |
| `$!VERBS`        |         | Verb table output                       |
| `$!DICT`         |         | Dictionary output                       |
| `$!OBJECTS`       |         | Object tree output                      |
| `$!SYMBOLS`      |         | Symbol table output                     |

Settings are specified as `$!NAME=N` on the command line, where `N` is
the trace level:

```
inform6 $!ASM=2 $!EXPR=1 game.inf
```

This compiles `game.inf` with assembly tracing at level 2 and
expression tracing at level 1. The output goes to standard output
(or standard error, depending on the setting) and can be redirected
to a file for analysis.

For settings that have a corresponding single-letter switch (like
`$!ASM` and `-a`), either form may be used; they are equivalent.


### 36.6.3 The -a Switch (Assembly Tracing)

The `-a` switch enables assembly tracing from the command line, as an
alternative to using `Trace assembly;` in source code or `$!ASM` on
the command line. The switch accepts an optional numeric level:

| Switch | Level | Output                                         |
|--------|-------|------------------------------------------------|
| `-a`   | 1     | Generated assembly instructions                |
| `-a2`  | 2     | Assembly with stack depth tracking              |
| `-a3`  | 3     | Assembly with hex dumps of generated bytecode   |
| `-a4`  | 4     | Maximum detail                                  |

Assembly tracing is useful for:

- Understanding how source code maps to Z-machine or Glulx
  instructions (see §29 and §30 for the instruction sets)
- Diagnosing unexpected behavior caused by code generation issues
- Verifying that compiler optimizations are working correctly
- Comparing code generation between different compiler versions

The `-a` switch applies globally to the entire compilation. To trace
only a specific section, use the `Trace assembly;` directive in source
code instead.


### 36.6.4 Testing Methodology

Effective testing of programs combines compile-time checking,
runtime debug verbs, and systematic play-testing. The following
methodology covers the recommended practices:

**Development compilation.** During development, always compile with
both `-D` and `-S` (strict mode is on by default):

```
inform6 -D game.inf
```

This enables the full set of debug verbs and runtime error checking.
Any runtime error messages that appear during testing indicate bugs that
should be fixed.

**Parser debugging.** When the parser misinterprets player input or
reports "I don't understand," use the `trace` verb at runtime:

```
> trace 2
[Parser tracing on, level 2.]
> get the brass key from the box
[ "get" "the" "brass" "key" "from" "the" "box" ]
[ Trying verb 'get': line 0 ... ]
...
```

The trace output shows exactly how the parser tokenizes the input,
which grammar lines it tries, and where matching fails. See §36.3.2.

**Object state inspection.** Use `showobj` (§36.3.7) to verify that
objects have the expected attributes and property values at any point
during gameplay. This is especially useful after complex state changes:

```
> showobj lamp
```

**Scope verification.** When objects are unexpectedly inaccessible, use
`scope` (§36.3.10) to list what the parser can currently see:

```
> scope
```

Compare the scope list against your expectations. If an object is
missing, check the scope rules described in §21.

**Action pipeline tracking.** Use `actions` (§36.3.3) to verify that
the correct actions are being generated and to trace the flow through
`before` and `after` rules:

```
> actions on
> put key in box
[ Action Insert (noun: brass key) (second: wooden box) ]
```

**Rapid traversal.** Use `purloin` (§36.3.13) and `gonear` (§36.3.12)
to quickly acquire objects and move to distant locations during testing.
Use `abstract` (§36.3.14) to set up specific test scenarios without
playing through prerequisites.

**Debug file generation.** For complex debugging sessions, use `-k` to
generate the `gameinfo.dbg` file (§36.4), which can be loaded by
external debugger tools for source-level debugging with breakpoints and
variable inspection.

**Game transcript recording.** Use the `-r` switch to record all game
text to `gametext.txt`. This file captures both player input and game
output and can be used for:

- Reviewing test sessions after the fact
- Comparing output between compiler or library versions
- Documenting bugs with full reproduction context

**Automated test scripts.** Write a text file containing a sequence of
game commands (one per line) and pipe it to the interpreter for
automated regression testing:

```
echo "take key
unlock door with key
open door
north
look" | dumb-frotz game.z5
```

This technique can be combined with `diff` to detect output changes
between versions:

```
echo "take key
look" | dumb-frotz game.z5 > output-new.txt
diff output-old.txt output-new.txt
```

**Release compilation.** For the final release build, compile without
`-D` and optionally without strict mode to minimize story file size:

```
inform6 -~D -~S game.inf
```

Or combine with optimization switches (see §35):

```
inform6 -~D -~S -esu $OMIT_UNUSED_ROUTINES=1 game.inf
```

> **Note:** Always perform a final round of play-testing on the release
> build. Some bugs may only manifest when debug infrastructure is
> absent (for example, code that inadvertently depends on the `DEBUG`
> constant being defined, or timing-sensitive behavior affected by the
> removal of tracing code).
