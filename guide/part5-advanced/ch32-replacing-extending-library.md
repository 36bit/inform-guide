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

# Chapter 32: Replacing and Extending the Library

The library provides a comprehensive framework for interactive fiction,
but nearly every game needs to customize it in some way. This chapter
covers the compiler directives and design patterns that allow a game to
replace individual library routines, define new actions and verbs, extend
the class hierarchy, intercept library messages, and write reusable
library extensions.

## 32.1 The Replace Directive

The `Replace` directive tells the compiler that a routine defined later in
the source should take the place of a routine with the same name that would
otherwise be defined in an included system file. It has two forms:

```inform6
Replace routine_name;
Replace routine_name preserved_name;
```

### 32.1.1 Basic Replacement

The first form marks the symbol `routine_name` with the internal flag
`REPLACE_SFLAG` (value 2 in the compiler's symbol table). When the
compiler later encounters a routine definition for that name inside a
**system file** (a file that begins with the `System_file` directive), it
skips the entire definition — reading and discarding tokens until it
reaches the closing `]` — and uses the game's own definition instead.

The key mechanism works as follows:

1. The `Replace` directive sets
   `symbols[token_value].flags |= REPLACE_SFLAG` on the named symbol.
2. When the compiler begins parsing a routine definition,
   it checks whether the symbol has `REPLACE_SFLAG` set **and** the
   current file is a system file (via `is_systemfile()`).
3. If both conditions are true and there is no rename mapping, the
   compiler sets `dont_enter_into_symbol_table = TRUE` and consumes
   tokens until `EOF_TT` or the closing `]`, effectively discarding the
   system file's version.
4. The game's own definition of the same routine — which appears outside
   any system file — is compiled normally.

The `Replace` directive must appear **before** the routine it replaces is
defined. In practice this means it must appear before the relevant
`Include` directive that pulls in the system file. It is a compile-time
error to place `Replace` after the routine has already been compiled.

### 32.1.2 Replacement with Preservation

The two-argument form preserves the original routine under a new name:

```inform6
Replace Banner OriginalBanner;
```

This tells the compiler to do everything the one-argument form does, but
additionally to compile the system file's definition under the name
`preserved_name` rather than discarding it. The replacement mapping is
stored in an array of `value_pair_t` structures:

```c
typedef struct value_pair_struct {
    int original_symbol;
    int renamed_symbol;
} value_pair_t;
```

When the compiler encounters the routine definition in the system file, it
finds the rename mapping via `find_symbol_replacement()` and compiles the
routine body under the renamed symbol instead of skipping it entirely. The
game can then call `preserved_name()` to invoke the original behavior.

### 32.1.3 Constraints

Several constraints apply to the `Replace` directive:

- **Ordering**: `Replace` must appear before the routine is defined. The
  compiler does not retroactively replace routines.
- **No chaining**: A routine cannot be replaced more than once. If symbol
  A has already been added to the replacement table, attempting to replace
  it again produces an error.
- **No self-replacement**: The preserved name cannot be the same as the
  original name.
- **System files only**: The skip-and-discard mechanism only activates
  inside system files. Replacing a routine defined in the game's own
  source has no effect through this mechanism — the compiler will simply
  report a duplicate definition.

### 32.1.4 Example: Replacing Banner

A common use of `Replace` is to provide a custom banner:

```inform6
Replace Banner;

Include "Parser";
Include "VerbLib";

[ Banner;
    print "The Mysterious Castle^";
    print "An interactive fiction by J. Author^";
    print "Release 1 / Serial number 260115^";
    new_line;
];

Include "Grammar";
```

The library's own `Banner` routine (which is
included via `VerbLib`) is skipped because `Replace Banner` was issued
before the `Include`. The game's `Banner` routine is compiled in its
place.

## 32.2 Replacing Library Routines

Building on the mechanics described in §32.1, this section covers
practical patterns for replacing library routines.

### 32.2.1 The Standard Template

The standard source file structure places `Replace` directives
at the very top, before any library includes:

```inform6
Constant Story "My Game";
Constant Headline "^An Interactive Example^";

Replace Banner;
Replace DrawStatusLine;

Include "Parser";
Include "VerbLib";

! ... game objects and routines ...

[ Banner;
    print (string) Story;
    print (string) Headline;
    print "Release 1^";
];

[ DrawStatusLine;
    @set_cursor 1 1;
    @split_window 1;
    @set_window 1;
    @set_cursor 1 1;
    style reverse;
    spaces (0->33 - 1);
    @set_cursor 1 2;
    print (name) location;
    @set_cursor 1 (0->33 - 14);
    print "Score: ", score;
    style roman;
    @set_window 0;
];

Include "Grammar";
```

### 32.2.2 Using the Two-Argument Form

The two-argument form of `Replace` is especially useful when you want to
augment the original behavior rather than completely replacing it:

```inform6
Replace Banner OriginalBanner;

Include "Parser";
Include "VerbLib";

[ Banner;
    OriginalBanner();
    new_line;
    print "Visit our website at example.com^";
];

Include "Grammar";
```

Here `OriginalBanner` is the library's own `Banner` routine, compiled
under a different name. The game's `Banner` calls it first, then appends
additional output. This pattern avoids duplicating code and ensures that
the replacement stays compatible with future library updates.

### 32.2.3 The System_file Directive

The `System_file` directive, placed at the top of a source file, marks
that file as a system file. Only routines defined inside system files can
be silently replaced by the `Replace` mechanism. The standard library
files (`Parser.h`, `VerbLib.h`, `Grammar.h`, and the files they include)
all begin with `System_file`.

If you write your own library extension and want its routines to be
replaceable by game authors, place `System_file` at the top of your
extension file (see §32.6).

### 32.2.4 Commonly Replaced Routines

The following library routines are frequently replaced by games:

| Routine | Defined In | Purpose |
|---------|-----------|---------|
| `Banner` | `verblib.h` | Prints the game banner at startup |
| `DrawStatusLine` | `parser.h` | Draws the status line (Z-machine) |
| `PrintRank` | `parser.h` | Prints the player's rank based on score |
| `BeforeParsing` | Stub in `grammar.h` | Hook called before each input is parsed |
| `Initialise` | Game-defined (not in library) | Game setup (typically defined by the game) |

Note that routines declared via `Stub` in `grammar.h` (see §32.3) are
intended to be defined by the game directly rather than replaced — since
the `Stub` only creates a default if the game does not provide one.

## 32.3 The Stub Directive

The `Stub` directive creates a minimal do-nothing routine if — and only
if — no routine of that name has already been defined. Its syntax is:

```inform6
Stub routine_name N;
```

where `N` is an integer from 0 to 4, specifying the number of local
variables the stub routine should declare.

### 32.3.1 Compiler Mechanics

When the compiler processes a `Stub` directive:

1. It checks whether `routine_name` already exists in the symbol table as
   a defined routine.
2. If the routine **already exists**, the `Stub` is silently ignored —
   the existing definition takes precedence.
3. If the routine **does not exist**, the compiler:
   - Sets `STUB_SFLAG` (value 16) on the symbol.
   - Generates a trivial routine body with `N` dummy local variables
     named `dummy1` through `dummy4` (as needed).
   - Emits a `rfalse` instruction (Z-machine) or a `return 0`
     instruction (Glulx), so the stub always returns `false` (0).
   - Marks the dummy variables as used to suppress "unused variable"
     compiler warnings.

The generated code is equivalent to:

```inform6
! For Stub SomeRoutine 2;
[ SomeRoutine dummy1 dummy2;
    rfalse;
];
```

### 32.3.2 Purpose in the Library

The primary use of `Stub` is in the library's `grammar.h` file, where it
declares default implementations for all the **entry point routines** that
a game may optionally define. Because `grammar.h` is included last (after
the game's own source), any routine the game defines will already exist in
the symbol table, causing the corresponding `Stub` to be skipped. If the
game does not define a particular entry point, the `Stub` provides a safe
default that simply returns false.

The complete list of stubs from `grammar.h`:

```inform6
Stub AfterLife         0;
Stub AfterPrompt       0;
Stub Amusing           0;
Stub BeforeParsing     0;
Stub ChooseObjects     2;
Stub DarkToDark        0;
Stub DeathMessage      0;
Stub Epilogue          0;
Stub GamePostRoutine   0;
Stub GamePreRoutine    0;
Stub InScope           1;
Stub LookRoutine       0;
Stub NewRoom           0;
Stub ObjectDoesNotFit  2;
Stub ParseNumber       2;
Stub ParserError       1;
Stub PrintTaskName     1;
Stub PrintVerb         1;
Stub TimePasses        0;
Stub UnknownVerb       1;
Stub AfterSave         1;
Stub AfterRestore      1;
```

**[Glulx]** Three additional stubs are defined when compiling for Glulx:

```inform6
#Ifdef TARGET_GLULX;
Stub HandleGlkEvent    2;
Stub IdentifyGlkObject 4;
Stub InitGlkWindow     1;
#Endif;
```

These Glulx-only entry points handle Glk event processing, object
identification during save/restore, and window initialization
respectively.

### 32.3.3 Entry Point Summary

Each stubbed entry point is called at a specific moment during gameplay.
The following table summarises their purpose and the number of arguments
each receives:

| Entry Point | Args | Called When |
|------------|------|------------|
| `AfterLife` | 0 | After the player dies, before "restart/restore" prompt |
| `AfterPrompt` | 0 | After the command prompt is printed |
| `Amusing` | 0 | When player wins and types `AMUSING` |
| `BeforeParsing` | 0 | Before each command is parsed |
| `ChooseObjects` | 2 | During disambiguation to rank objects |
| `DarkToDark` | 0 | When the player moves from one dark room to another |
| `DeathMessage` | 0 | To print a custom death message |
| `Epilogue` | 0 | After winning or dying, before "restart/restore" prompt |
| `GamePostRoutine` | 0 | After each action, if not already handled |
| `GamePreRoutine` | 0 | Before each action is processed |
| `InScope` | 1 | To add objects to scope for a given actor |
| `LookRoutine` | 0 | After each room description is printed |
| `NewRoom` | 0 | When the player enters a new room |
| `ObjectDoesNotFit` | 2 | When an object is too large for a container |
| `ParseNumber` | 2 | To handle numeric input during parsing |
| `ParserError` | 1 | To intercept or customize parser error messages |
| `PrintTaskName` | 1 | To print the name of a scored task |
| `PrintVerb` | 1 | To print a verb the library does not recognize |
| `TimePasses` | 0 | At the end of each turn |
| `UnknownVerb` | 1 | To handle a verb the parser does not recognize |
| `AfterSave` | 1 | After a save operation (argument: success flag) |
| `AfterRestore` | 1 | After a restore operation (argument: success flag) |

## 32.4 Adding New Actions and Verbs

The `Verb`, `Extend`, and `Fake_Action` directives allow a game to define
new commands and actions beyond those provided by the standard library.
For related discussion of how the parser processes grammar at runtime, see
§20.

### 32.4.1 The Verb Directive

The `Verb` directive defines a new verb and its associated grammar. Its
general syntax is:

```inform6
Verb 'word1' 'word2' ...
    * token token ... -> ActionName
    * token token ... -> ActionName
    ...;
```

Each quoted word is a dictionary entry that the player can type to invoke
the verb. The grammar lines (beginning with `*`) describe the patterns of
input that the parser will accept, and the `->` arrow specifies which
action to generate when a pattern matches.

A complete example:

```inform6
[ DanceSub;
    "You dance a little jig.";
];

Verb 'dance' 'jig' 'boogie'
    *                           -> Dance;
```

This defines a verb that the player can invoke by typing `DANCE`, `JIG`,
or `BOOGIE`. The grammar line `*` (with no tokens) matches the verb word
alone with no further input. When matched, the parser generates the
`Dance` action, which is handled by the `DanceSub` routine.

The action routine must be named with the suffix `Sub` appended to the
action name. The action value itself can be referred to in code as
`##Dance`.

### 32.4.2 The meta Keyword

A verb can be declared as **meta** by placing the `meta` keyword before
the first dictionary word:

```inform6
Verb meta 'version'
    *                           -> Version;
```

Meta actions are "out-of-world" actions that do not consume a game turn.
They are typically used for commands like `SAVE`, `RESTORE`, `SCORE`, and
`QUIT`. A meta action does not trigger `GamePreRoutine`,
`GamePostRoutine`, `TimePasses`, or the `each_turn` daemon cycle.

### 32.4.3 Grammar Tokens

Each grammar line consists of zero or more **tokens** that describe what
the parser should expect after the verb word. The most commonly used
tokens are:

| Token | Meaning |
|-------|---------|
| `noun` | Any object in scope |
| `held` | An object held by the player |
| `multi` | One or more objects in scope |
| `multiheld` | One or more held objects |
| `multiexcept` | Multiple objects except the second noun |
| `multiinside` | Multiple objects inside the second noun |
| `creature` | An animate object in scope |
| `topic` | Any sequence of words (for `ASK`/`TELL`) |
| `number` | A numeric value |
| `scope=Routine` | Objects returned by a custom scope routine |
| `noun=Routine` | Objects filtered by a custom routine |
| `'word'` | A literal dictionary word |
| `attribute` | An object attribute (for debugging verbs) |

Tokens can be combined to form complex grammar patterns:

```inform6
Verb 'give' 'pay' 'offer' 'feed'
    * held 'to' creature       -> Give
    * creature held             -> Give reverse
    * 'up'                      -> Quit;
```

The `reverse` keyword after the action name swaps the values of `noun`
and `second` when the action is generated, allowing alternative word
orders without separate action routines.

### 32.4.4 The Extend Directive

The `Extend` directive adds grammar lines to an existing verb without
replacing its entire grammar. It has several modes:

**Default (append last):**

```inform6
Extend 'take'
    * 'a' 'bath'                -> Bathe
    * 'a' 'nap'                 -> Sleep;
```

New grammar lines are appended after the verb's existing lines. Since the
parser tries lines in order, existing patterns take priority.

**Explicit modes:**

```inform6
Extend 'look' first
    * 'through' noun            -> Search;
```

The `first` keyword inserts the new lines before the existing grammar, so
they are tried first during parsing.

```inform6
Extend 'look' last
    * 'behind' noun             -> LookBehind;
```

The `last` keyword is the default — new lines are appended after existing
grammar.

```inform6
Extend 'get' replace
    * multi                     -> Take
    * 'out'/'off'               -> Exit
    * multiinside 'from' noun   -> Remove;
```

The `replace` keyword discards **all** existing grammar for the verb and
substitutes the new lines. Use this with care, as it removes all standard
behavior for that verb word.

**The `only` keyword:**

```inform6
Extend only 'get' replace
    * multi                     -> Take
    * 'out'/'off'               -> Exit
    * multiinside 'from' noun   -> Remove;
```

The `only` keyword detaches a single dictionary word from its verb group
before extending it. Normally, `'get'` and `'take'` share the same verb
entry; `Extend only 'get'` separates `'get'` so it can have its own
independent grammar while `'take'` retains the original grammar.

The `/` character between quoted words (e.g., `'out'/'off'`) means "either
word" — the parser accepts either `OUT` or `OFF` in that position.

### 32.4.5 The Fake_Action Directive

The `Fake_Action` directive creates an action constant without any
associated grammar:

```inform6
Fake_Action Sneeze;
```

This allocates an action number for `Sneeze` (accessible as `##Sneeze`)
but does not create a verb or grammar for it. Fake actions are used for
internal signaling between objects — for example, a `before` handler
might respond to `##Sneeze` even though the player can never type
a command that generates it directly. Game code can trigger it with:

```inform6
<Sneeze>;
```

or the longer form:

```inform6
action = ##Sneeze;
noun = 0; second = 0;
ActionPrimitive();
```

Fake actions are also used by the library itself for internal actions like
`##Receive`, `##ThrownAt`, `##Order`, and others that are never generated
directly by the parser (see §20.5).

### 32.4.6 VM Limits on Verbs and Actions

It is important to distinguish between **verbs** (dictionary words that drive grammar, like "take" or "drop") and **actions** (the resulting subroutine triggered by the parser, like `##Take`).

**[Z-machine]** The Z-machine format limits the game to a maximum of 255
verb entries. Each `Verb` directive or `Extend only` operation that creates
a new verb entry counts against this limit. (This is because the Z-machine's dictionary format relies on a single byte to track verb references.) In practice, the standard library uses approximately 60 verb entries, leaving ample room for game-defined verbs.

However, the limit on *actions* is much higher. Under Grammar Version 2 (GV2) and Version 3 (GV3), the Z-machine supports up to **1024 distinct actions**. (Under the older Grammar Version 1, actions are limited to 256).

**[Glulx]** Although the VM itself has no architectural limitation on the number of verbs, the Inform 6 compiler enforces a fixed maximum of 65,535 verbs for Glulx programs. In practice, this is effectively unlimited. Glulx also supports a virtually unlimited number of actions.

## 32.5 Extending Classes

The language supports single and multiple inheritance through its class
system. Games can define new classes, inherit from library classes, and
combine behaviors from multiple parent classes.

### 32.5.1 Defining Classes

A class is defined with the `Class` directive:

```inform6
Class Treasure
    with  value 0,
          description "It looks valuable.",
    has   scored;
```

Classes serve as templates: any object (or further class) that inherits
from `Treasure` will receive the `value` property, the default
`description`, and the `scored` attribute.

Objects inherit from a class by naming it in a `class` segment:

```inform6
Object -> ruby_ring "ruby ring"
    class Treasure
    with  name 'ruby' 'ring',
          value 50,
          description "A ring set with a large ruby.";
```

The `ruby_ring` inherits the `scored` attribute from `Treasure`. Its own
`value` and `description` override the defaults provided by the class.

### 32.5.2 Multiple Inheritance

An object or class can inherit from more than one class:

```inform6
Class Container
    has   container open,
    with  capacity 10;

Class Lockable
    has   lockable,
    with  with_key 0;

Object -> strongbox "strongbox"
    class Container Lockable
    with  description "A heavy iron strongbox.",
         with_key brass_key;
```

When multiple classes are listed, inheritance proceeds left to right. For
**non-additive** properties, if two classes define the same property, the
value from the later (rightmost) class takes precedence. For **additive**
properties (see §7.5), values from all classes and the object's own
definition are accumulated.

Attributes are inherited from all listed classes via a union — if any
class provides an attribute, the inheriting object receives it.

The compiler tracks class information through an internal `classinfo`
structure:

```c
static int   *classes_to_inherit_from;
classinfo    *class_info;
```

Each class entry records the class's prototype object, its property block
offset, and its symbol table index. During object construction, the
compiler walks the class list, copying properties and attributes in order.

### 32.5.3 Constraints on Inheritance

- A class cannot inherit from itself. The compiler checks for this and
  reports an error: "A class cannot inherit from itself."
- Circular inheritance (A inherits from B, B inherits from A) is
  prevented by the requirement that a class must be defined before it can
  be used as a parent.
- The number of classes is limited by a fixed compiler constant in Inform 6.44 (`VENEER_CONSTRAINT_ON_CLASSES_Z` caps it at 256 for Z-Machine, `VENEER_CONSTRAINT_ON_CLASSES_G` caps it at 32,768 for Glulx).

### 32.5.4 Runtime Class Testing

At runtime, two operators test class membership:

**`ofclass`** tests whether an object is a member of a given class:

```inform6
if (obj ofclass Container) print "It's a container.^";
```

This returns `true` if `obj` inherits from `Container`, whether directly
or through a chain of intermediate classes.

**`metaclass`** returns the metaclass of a value:

```inform6
x = metaclass(obj);
```

The possible return values are:

| Return Value | Meaning |
|-------------|---------|
| `Object` | A regular object |
| `Class` | A class object |
| `Routine` | A routine |
| `String` | A string |
| `nothing` (0) | Not a valid object/routine/string |

These operators are compiled into efficient opcodes on both the Z-machine
and Glulx (see §28 and §29).

### 32.5.5 Overriding Inherited Properties

When an object provides its own value for an inherited property, the
object's value completely replaces the class's value (for non-additive
properties). For routine-valued properties, this means the object's
routine is called instead of the class's routine:

```inform6
Class Furniture
    with  description [; "A piece of furniture."; ],
          before [;
              Take: "It's fixed in place.";
          ],
    has   static scenery;

Object -> chair "wooden chair"
    class Furniture
    with  description "A rickety wooden chair.",
          name 'wooden' 'chair';
```

The `chair` inherits `before` and the attributes `static` and `scenery`
from `Furniture`, but its own `description` (a string) overrides the
class's `description` (a routine).

## 32.6 Library Contributions and Extensions

A library extension is a reusable source file that provides additional
functionality to games that include it. Well-written extensions follow
several conventions.

### 32.6.1 The System_file Directive

An extension that wants its routines to be replaceable by game authors
should place the `System_file` directive at the top of the file:

```inform6
System_file;
```

This marks all routine definitions in the file as belonging to a system
file. Game authors can then use `Replace` to override individual routines,
exactly as they would with standard library routines (see §32.1).

### 32.6.2 Include Guard Pattern

To prevent double-inclusion (which would cause duplicate symbol errors),
extensions should wrap their contents in a conditional compilation guard:

```inform6
#Ifndef MY_EXTENSION;
Constant MY_EXTENSION;

System_file;

! ... extension code ...

#Endif; ! MY_EXTENSION
```

If the file is included a second time, the constant `MY_EXTENSION` will
already exist and the body will be skipped.

### 32.6.3 Include Paths

The `Include` directive searches for files in the following order:

1. The current directory (the directory containing the file that issued
   the `Include`).
2. Each directory specified by the `+include_path` compiler switch, in
   the order given on the command line.

Multiple include paths can be specified:

```
inform6 +include_path=./lib,./extensions MyGame.inf
```

Within source code, `Include` takes a string filename:

```inform6
Include "MyExtension";
```

The compiler appends the appropriate file extension (`.h` or `.inf`)
depending on the platform and settings.

### 32.6.4 Version Checking

Extensions can check which version of the library is present using
predefined constants:

| Constant | Type | Example Value | Meaning |
|----------|------|--------------|---------|
| `LIBRARY_VERSION` | Numeric | 612 | Library version number (6.12 = 612) |
| `LibRelease` | String | `"6.12.8"` | Human-readable version string |
| `LibSerial` | String | `"251226"` | Serial number (date-based: YYMMDD) |

```inform6
#Ifdef LIBRARY_VERSION;
#Iftrue LIBRARY_VERSION >= 612;
    ! Code that requires library 6.12 or later
#Endif;
#Endif;
```

The outer `#Ifdef` check ensures the code compiles even outside the
library context. The `#Iftrue` directive performs a numeric comparison.

### 32.6.5 Extension Best Practices

- **Document dependencies**: state which library version is required.
- **Namespace your symbols**: use a prefix derived from the extension name
  for constants, globals, and routines to avoid collisions with other
  extensions or the game itself.
- **Use `System_file`**: so game authors can replace individual routines.
- **Use include guards**: to prevent double-inclusion (see §32.6.2).
- **Avoid modifying globals at load time**: defer initialization to
  a routine that the game can call from `Initialise`.
- **Provide `Replace`-friendly routines**: keep routines small and
  focused so they can be individually replaced without copying large
  amounts of code.

### 32.6.6 Extension Structure Example

A typical extension file:

```inform6
! myextension.h — Provides the Gizmo class.
! Requires library 6.12 or later.

#Ifndef MYEXTENSION_H;
Constant MYEXTENSION_H;

System_file;

#Ifdef LIBRARY_VERSION;
#Iftrue LIBRARY_VERSION < 612;
Message error "myextension.h requires Inform library 6.12 or later.";
#Endif;
#Endif;

Class Gizmo
    with  name 'gizmo',
          description "A mysterious gizmo.",
          activate [;
              give self on;
              "You activate ", (the) self, ".";
          ],
          deactivate [;
              give self ~on;
              "You deactivate ", (the) self, ".";
          ],
    has   switchable;

[ GizmoActivateSub;
    if (noun ofclass Gizmo)
        noun.activate();
    else
        "That's not a gizmo.";
];

[ GizmoDeactivateSub;
    if (noun ofclass Gizmo)
        noun.deactivate();
    else
        "That's not a gizmo.";
];

Verb 'activate'
    * noun                      -> GizmoActivate;

Verb 'deactivate'
    * noun                      -> GizmoDeactivate;

#Endif; ! MYEXTENSION_H
```

The game includes it with:

```inform6
Include "Parser";
Include "VerbLib";

Include "myextension";

! ... game code ...

Include "Grammar";
```

## 32.7 Customizing LibraryMessages

The standard library generates many messages during gameplay — responses
to actions, parser errors, status reports, and so on. The
`LibraryMessages` mechanism provides a clean way to customize these
messages without replacing entire library routines.

### 32.7.1 The LibraryMessages Object

A game can define an object named `LibraryMessages` to intercept library
messages before they reach the default message handler:

```inform6
Object LibraryMessages
    with before [;
        Take:
            if (lm_n == 1)
                print_ret "You grab ", (the) noun, ".";
        Drop:
            if (lm_n == 1)
                print_ret "You toss ", (the) noun, " aside.";
        Look:
            if (lm_n == 1)
                print_ret ""; ! suppress "You can see..." preamble
    ];
```

The `LibraryMessages` object must be defined in the game source (not
inside a class or contained within another object). It has no parent, no
name, and no attributes — only the `before` property matters.

### 32.7.2 Dispatch Mechanism

When the library needs to print a message, the following sequence occurs:

1. The library sets the action variable to the relevant action (e.g.,
   `##Take`) and the global `lm_n` to a sub-message number.
2. If the game has defined a `LibraryMessages` object, the library calls
   `LibraryMessages.before()`.
3. If `before` returns `true` (i.e., it printed a message and used
   `print_ret`, `rtrue`, or returned a non-zero value), the library
   considers the message handled and skips its default.
4. If `before` returns `false` (the entry point was not intercepted, or
   there is no `LibraryMessages` object), the library calls
   `LanguageLM(n, x1, x2)` in `english.h`, which prints the default
   English-language message.

### 32.7.3 The lm_n Variable

The global variable `lm_n` (declared in `parser.h`) identifies which
specific message within an action is being printed. Each action can have
multiple messages corresponding to different situations. For example, the
`Take` action defines several `lm_n` values:

| `lm_n` | Situation |
|--------|-----------|
| 1 | Successfully taken |
| 2 | The player already has it |
| 3 | Carried by an NPC |
| 4 | Object is not accessible |
| 5 | Object is part of the player |
| 6 | Belongs to an NPC |
| 7 | Part of a scenery object |
| 8 | NPC action taken |
| 9 | In a closed container |
| 10 | Object is fixed in place |
| 11 | Object is animate |
| 12 | Taking too many items (inventory full) |
| 13 | A container-specific refusal |

Not every action uses all 13 or more sub-messages; some actions have only
one or two. The exact set of `lm_n` values for each action is defined by
the corresponding case in `LanguageLM` in `english.h`.

### 32.7.4 Additional Context Variables

The `LanguageLM` function receives two additional parameters, `x1` and
`x2`, that carry context-specific information. The library also uses the
companion globals `lm_o` and `lm_s` (both declared in `parser.h`
alongside `lm_n`) to pass additional context in some messages. The exact
meaning of these parameters varies by action and `lm_n` value — consult
the source of `LanguageLM` in `english.h` for the specific action.

For example, in some messages `x1` may be the object that caused the
action to fail, while in others it may be a count of items.

### 32.7.5 LanguageLM and Internationalization

The default message handler, `LanguageLM`, is defined in `english.h`
and contains a large switch statement
covering every action that the library can generate. Each case handles the
various `lm_n` sub-messages for that action:

```inform6
[ LanguageLM n x1 x2;
    Answer, Ask:
        print_ret "There is no reply.";
    Attack:
        print_ret "Violence isn't the answer to this one.";
    ! ... many more cases ...
];
```

The relationship between `LibraryMessages` and `LanguageLM` is
straightforward:

- **`LibraryMessages`** provides **game-level** customization. A game
  author uses it to adjust specific messages for their particular game.
- **`LanguageLM`** provides **language-level** defaults. A translator
  replacing `english.h` with (for example) `french.h` would rewrite
  `LanguageLM` to produce French-language messages.

### 32.7.6 Comprehensive Example

The following example customizes messages for several common actions:

```inform6
Object LibraryMessages
    with before [;
        Take:
            if (lm_n == 1)
                print_ret "Taken.  (", (the) noun, ")";
            if (lm_n == 12) {
                print "Your hands are full";
                if (noun has pluralname)
                    print " and you can't carry them all.^";
                else
                    print " and you can't carry it too.^";
                rtrue;
            }
        Drop:
            if (lm_n == 1)
                print_ret (The) noun, " tumbles to the ground.";
        Examine:
            if (lm_n == 1) {
                print "You see nothing special about ";
                print (the) noun, ".^";
                rtrue;
            }
        Search:
            if (lm_n == 7)
                print_ret "You find nothing of interest.";
        Eat:
            if (lm_n == 1)
                print_ret "You devour ", (the) noun, ". Not bad.";
        Inv:
            if (lm_n == 1)
                print_ret "You are empty-handed.";
            if (lm_n == 2)
                print "You are carrying";
    ];
```

This object intercepts specific messages while allowing all unhandled
messages to fall through to the library defaults. The game author only
needs to handle the `lm_n` values they want to change.

### 32.7.7 Replacing vs. LibraryMessages

There are two approaches to customizing library output:

| Approach | When to Use |
|----------|------------|
| `LibraryMessages` | Changing the text of specific messages while keeping the library's action-processing logic intact |
| `Replace` | Changing the logic of how an action is processed, not just its textual output |

In general, prefer `LibraryMessages` for text changes. It is less
invasive, does not require understanding the internal structure of library
routines, and is more resilient to library version changes. Use `Replace`
when you need to alter control flow, add new conditions, or completely
restructure how a routine works.
