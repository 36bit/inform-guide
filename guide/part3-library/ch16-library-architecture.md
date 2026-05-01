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

# Chapter 16: Library Architecture and Initialization

This chapter describes the overall architecture of the standard library
(version 6.12.8): how the library files fit together, how the game loop
operates, and how programs configure and initialize the library.

## 16.1 Overview

The standard library provides the complete runtime framework for interactive
fiction. It supplies:

- **Parsing** — reading and interpreting player commands.
- **The world model** — rooms, objects, containment, light, and scope.
- **Actions** — dispatching player commands to action routines.
- **Printing** — object names, room descriptions, inventory lists, and
  library messages.
- **Timekeeping** — turn counting, timers, daemons, and each-turn processing.

Most programs are built on top of this library. A game author
writes objects, rooms, and a handful of entry-point routines; the library
handles everything else. Understanding the library's architecture makes it
easier to customize behavior and debug problems.

## 16.2 The Three-File Include Pattern

A standard program includes the library in three pieces, in a
fixed order:

```inform6
Include "Parser";     ! First: parser and core infrastructure
Include "VerbLib";    ! Second: action routines and utilities
Include "Grammar";    ! Third: grammar tables (must be last)
```

Each `Include` directive pulls in one of the library's main header files:

- **Parser** (`parser.h`) — the parser engine, the main game loop, scope
  handling, and the `InformLibrary` and `InformParser` objects. This is the
  largest library file (over 7,000 lines) and forms the backbone of the
  runtime.
- **VerbLib** (`verblib.h`) — all standard action routines (such as
  `TakeSub`, `DropSub`, `LookSub`), as well as utility routines like
  `PlayerTo`, `WriteListFrom`, and `YesOrNo`.
- **Grammar** (`grammar.h`) — all `Verb` and `Extend` directives that
  define the player's vocabulary. This file also includes `english.h` (or
  the appropriate language definition file) for language-specific grammar
  and messages.

The game's own source code is placed **between** these includes:

```inform6
Include "Parser";

! --- Game-specific constants, attributes, properties ---
Constant Story "Ruins";
Constant Headline "^An Interactive Catastrophe^";

! --- Game objects, classes, and routines ---
Object  startroom "Jungle Clearing"
  with  description "Dense vegetation surrounds a clearing.",
  has   light;

[ Initialise;
    location = startroom;
];

Include "VerbLib";

! --- Additional routines, if needed ---

Include "Grammar";

! --- Custom Verb/Extend directives, if needed ---
```

### 16.2.1 Why the Order Matters

The three-file order is not arbitrary. `parser.h` defines global variables,
constants, and core objects that the rest of the library and the game source
depend on. `verblib.h` relies on those definitions and also on the game's
object tree being already declared (so that `Default` constants like
`MAX_CARRIED` can be overridden beforehand). `grammar.h` must come last
because `Verb` and `Extend` directives reference action routines defined in
`verblib.h` and the game source.

### 16.2.2 Library Stages

The library defines stage constants that track compilation progress:

| Constant          | Value | Set after including |
|-------------------|-------|---------------------|
| `BEFORE_PARSER`   | 0     | *(initial state)*   |
| `AFTER_PARSER`    | 1     | `Parser`            |
| `AFTER_VERBLIB`   | 2     | `VerbLib`           |
| `AFTER_GRAMMAR`   | 3     | `Grammar`           |

These are primarily used internally by the library to guard conditional
compilation blocks.

## 16.3 The `Initialise` Entry Point

`Initialise` is the only **mandatory** entry point a game must define.
The library calls it once during startup, from `GamePrologue`, before the
first room description is printed.

### 16.3.1 Requirements

At minimum, `Initialise` must set the global variable `location` to the
player's starting room:

```inform6
[ Initialise;
    location = startroom;
];
```

If `location` is not set, the player begins in darkness with no valid
room, which is almost never the intended behavior.

### 16.3.2 Return Values

The return value of `Initialise` controls banner printing:

| Return value | Effect                                              |
|--------------|-----------------------------------------------------|
| 0 (or false) | Print the banner and perform the initial `<Look>`   |
| 1 (or true)  | Print the banner, perform the initial `<Look>`      |
| 2            | Skip the banner entirely (initial `<Look>` still occurs unless `NOINITIAL_LOOK` is defined) |

Returning 2 is useful when the game prints its own custom title screen
before play begins.

### 16.3.3 A Typical `Initialise` Routine

```inform6
[ Initialise;
    location = study;
    "^^^Welcome to the mystery.
     You awaken in your study, the lamp burning low.^";
];
```

The printed string appears before the first room description. Because the
routine falls off the end after the print statement, it returns false (0),
so the banner prints normally.

### 16.3.4 Advanced Initialization

`Initialise` may also:

- Set `player` to a different object (replacing the default `selfobj`).
- Start timers and daemons with `StartTimer` and `StartDaemon`.
- Move objects into position.
- Set `score`, `the_time`, or other global state.
- Seed the random number generator with `random(-seed)` for testing.

```inform6
[ Initialise;
    location = bridge;
    player = captain;            ! play as a custom PC object
    StartDaemon(reactor);        ! begin reactor monitoring
    the_time = 8 * 60 + 30;     ! start at 08:30
];
```

## 16.4 The Main Game Loop

The heart of the library is the `InformLibrary.play` method. The compiler
generates a `Main` routine that simply calls this method:

```inform6
[ Main; InformLibrary.play(); ];
```

The `play` method divides execution into three phases.

### 16.4.1 Phase 1: Prologue

Before the main loop begins, `play` performs virtual machine initialization
and then calls `GamePrologue`:

```inform6
#Ifdef TARGET_ZCODE;
ZZInitialise();
#Ifnot; ! TARGET_GLULX
GGInitialise();
#Endif;
GamePrologue();
```

> **[Z-machine]** `ZZInitialise` sets up Z-machine-specific state such as
> the transcript and header flags.

> **[Glulx]** `GGInitialise` opens Glk windows and initialises Glulx I/O
> channels.

`GamePrologue` performs the following steps:

1. Seeds the random number generator.
2. Sets `player` to `selfobj` and `selfobj.capacity` to `MAX_CARRIED`.
3. Calls `LanguageInitialise` (if defined by the language module).
4. Runs `LibraryExtensions.RunAll(ext_initialise)`.
5. Calls the game's `Initialise` entry point.
6. Moves the player into `location` and sets up `real_location`.
7. Computes initial lighting.
8. Prints the banner (unless `Initialise` returned 2).
9. Performs an initial `<Look>` action (unless `NOINITIAL_LOOK` is defined).

### 16.4.2 Phase 2: The Main Loop

The main loop runs while `deadflag` is false (zero):

```
while (~~deadflag) {
    1. Score notification
    2. Parse player input
    3. Action transformations
    4. Action processing (or NPC orders)
    5. Multiple-object handling
    6. End-of-turn sequence
}
```

Each iteration represents one player turn. The steps are:

**Step 1 — Score notification.** If `score` has changed since the last
turn and `notify_mode` is on, the library prints a score notification.

**Step 2 — Parse player input.** The parser
(`InformParser.parse_input`) reads a line of input, tokenizes it, matches
it against the grammar, and writes the results into the `inputobjs` array:

| Entry            | Contains                                |
|------------------|-----------------------------------------|
| `inputobjs-->0`  | The action number                       |
| `inputobjs-->1`  | Number of matched objects (0, 1, or 2)  |
| `inputobjs-->2`  | First noun (or 0, or 1 for a number)    |
| `inputobjs-->3`  | Second noun (or 0, or 1 for a number)   |

The parser also sets the `meta` flag for meta-verbs (such as `save` and
`score`).

**Step 3 — Action transformations.** Before dispatching the action, the
library normalizes several reversed or compound grammar forms:

- **`##GiveR` → `##Give`** and **`##ShowR` → `##Show`**: The grammar
  "give fred biscuit" parses with reversed objects; the library swaps
  `noun` and `second` so action routines always see "give biscuit to fred".
- **`##Tell` → `##Ask`**: "tell P about X" is converted to "ask P about X"
  when the addressee is the player.
- **`##AskFor` → `##Give`**: "ask P for X" becomes an order "P, give X to
  me".

**Step 4 — Action processing.** If `actor` is the player, the library
calls `begin_action`, which runs the full action pipeline (see §16.5). If
`actor` is an NPC (a command like "fred, take lamp"), the library calls
`actor_act`, which invokes the NPC's `orders` and `life` properties.

**Step 5 — Multiple-object handling.** When the parser matched "all" or a
multi-noun token, the library iterates through the `multiple_object` list,
running the action once for each object with a printed name prefix.

**Step 6 — End-of-turn sequence.** After the action is resolved and
provided the verb was not meta, `end_turn_sequence` is called (see
§16.4.3).

### 16.4.3 The End-of-Turn Sequence

The `end_turn_sequence` method advances the world state. Each step checks
`deadflag` and returns immediately if the player has died:

```inform6
end_turn_sequence [;
    AdvanceWorldClock();        if (deadflag) return;
    RunTimersAndDaemons();      if (deadflag) return;
    RunEachTurnProperties();    if (deadflag) return;
    if (TimePasses() == 0)
        LibraryExtensions.RunAll(ext_timepasses);
                                if (deadflag) return;
    AdjustLight();              if (deadflag) return;
    NoteObjectAcquisitions();
];
```

The individual steps are:

- **`AdvanceWorldClock`** — increments the `turns` counter. If real-time
  mode is active (`the_time` is set), advances the clock by `time_rate`
  minutes per turn, wrapping at 1440 (midnight).
- **`RunTimersAndDaemons`** — iterates through the active timers array.
  Daemons call `RunRoutines(obj, daemon)` each turn. Timers decrement a
  countdown and call `RunRoutines(obj, time_out)` when they reach zero.
- **`RunEachTurnProperties`** — calls the `each_turn` property on the
  current location, then on every object in scope.
- **`TimePasses`** — the game's optional entry point, called once per turn.
  If undefined or returning false, `LibraryExtensions` gets a chance via
  `ext_timepasses`.
- **`AdjustLight`** — recalculates whether the player can see. If light
  has changed, sets `location` to `thedark` or `real_location` as
  appropriate and prints a message.
- **`NoteObjectAcquisitions`** — gives the `moved` attribute to any
  object newly acquired by the player.

### 16.4.4 Phase 3: Death and Epilogue

When `deadflag` becomes nonzero, the main loop exits. The library then:

1. If `deadflag ~= 2` (not a victory), calls `AfterLife`. This optional
   game entry point can clear `deadflag` to resurrect the player.
2. If `AfterLife` returns false, runs
   `LibraryExtensions.RunAll(ext_afterlife)`.
3. If `deadflag` was cleared, the loop resumes.
4. Otherwise, calls `GameEpilogue`, which prints the death/victory message
   and offers RESTART, RESTORE, UNDO, or (on victory) AMUSING if
   `AMUSING_PROVIDED` is defined.

## 16.5 The Action Processing Pipeline

When the player issues a command, `begin_action` processes it through a
three-stage pipeline. (The full details of action handling are covered in
Chapter 20; this section provides a structural overview.)

### 16.5.1 Before Routines

`BeforeRoutines` gives the game and the world model a chance to intercept
the action before the default behavior runs. The checks proceed in this
order, stopping if any returns true:

1. `GamePreRoutine()` — a global game entry point.
2. `LibraryExtensions.RunWhile(ext_gamepreroutine, 0)` — extension hooks.
3. `player.orders` — the player object's `orders` property.
4. `react_before` — the `react_before` property on every object in scope.
5. `location.before` — the current room's `before` property.
6. `noun.before` — the first noun's `before` property.

If any routine returns true, the action is intercepted and the action
primitive does not run.

### 16.5.2 The Action Primitive

If no before routine intercepted the action, the library dispatches to the
appropriate action routine via a table lookup:

> **[Z-machine]** `(#actions_table-->action)()` — a direct call through
> the actions table.

> **[Glulx]** `(#actions_table-->(action+1))()` — Glulx uses a slightly
> different table layout with an offset.

The action routine (e.g., `TakeSub`, `DropSub`, `ExamineSub`) carries out
the default behavior for the verb.

### 16.5.3 After Routines

After the action primitive runs, `AfterRoutines` gives the game a chance
to react. The checks proceed in order:

1. `react_after` — the `react_after` property on every object in scope.
2. `location.after` — the current room's `after` property.
3. `noun.after` — the first noun's `after` property.
4. `GamePostRoutine()` — a global game entry point.
5. `LibraryExtensions.RunWhile(ext_gamepostroutine, false)` — extension
   hooks.

## 16.6 Library Version Constants

The file `version.h` defines four constants that identify the library
release:

```inform6
Constant LibSerial       "251226";    ! Date-based serial number
Constant LibRelease      "6.12.8";    ! Human-readable version string
Constant LIBRARY_VERSION  612;        ! Numeric version (6.12 = 612)
Constant Grammar__Version 2;          ! Grammar table format version
```

`LIBRARY_VERSION` is the constant most commonly tested for feature
detection in library extensions:

```inform6
#Ifndef LIBRARY_VERSION;
Message error "This extension requires the Inform 6 standard library.";
#Endif;
#Iftrue (LIBRARY_VERSION < 612);
Message error "This extension requires library 6.12 or later.";
#Endif;
```

`Grammar__Version` is used internally to select between grammar table
formats. Version 2 is standard in all modern programs.

## 16.7 Compilation Constants

The game can define constants **before** `Include "Parser"` (or before
`Include "VerbLib"` where noted) to configure library behavior. If the
game does not define a constant, the library provides a default value using
the `Default` directive.

### 16.7.1 Debugging and Diagnostics

| Constant       | Effect                                             |
|----------------|----------------------------------------------------|
| `DEBUG`        | Enables trace commands (`actions`, `timers`, `scope`, etc.) at runtime |
| `INFIX`        | Enables the Infix debugger for inspecting and modifying game state interactively |

These are typically set using compiler switches (`-D` or `$DEFINE`) rather
than in source code.

### 16.7.2 Scoring and Tasks

| Constant         | Default | Effect                                       |
|------------------|---------|----------------------------------------------|
| `MAX_SCORE`      | 0       | Maximum possible score (0 disables the score display on the status line) |
| `NUMBER_TASKS`   | 1       | Number of scored tasks                        |
| `TASKS_PROVIDED` | 1       | Enables the task-based scoring system         |
| `ROOM_SCORE`     | 5       | Points awarded for visiting a scored room     |
| `OBJECT_SCORE`   | 4       | Points awarded for acquiring a scored object  |
| `NO_SCORE`       | —       | If defined, disables scoring entirely         |

### 16.7.3 Inventory and Carrying

| Constant       | Default | Effect                                         |
|----------------|---------|------------------------------------------------|
| `MAX_CARRIED`  | 100     | Maximum objects the player can carry (sets `selfobj.capacity`) |
| `SACK_OBJECT`  | 0       | Object number of an implicit container; if nonzero, the library automatically puts objects into this container when the player's hands are full (0 disables) |

### 16.7.4 Game Features

| Constant              | Effect                                          |
|-----------------------|-------------------------------------------------|
| `AMUSING_PROVIDED`    | If defined, offer "AMUSING" option at victory   |
| `NO_PLACES`           | If defined, disables the `PLACES` and `OBJECTS` meta-verbs |
| `MANUAL_PRONOUNS`     | If defined, prevents the library from automatically updating pronoun references (`it`, `him`, `her`) |
| `NOINITIAL_LOOK`      | If defined, suppresses the initial room description at game start |
| `DEATH_MENTION_UNDO`  | If defined, mentions UNDO in the death message  |

### 16.7.5 Timers and World Clock

| Constant       | Default | Effect                                         |
|----------------|---------|------------------------------------------------|
| `MAX_TIMERS`   | 32      | Maximum number of simultaneously active timers and daemons |
| `START_MOVE`   | 0       | Starting value for the `turns` counter (0 = Infocom convention; 1 = Inform convention) |

### 16.7.6 Target Platform

The compiler predefines one of these constants to indicate the target
virtual machine:

| Constant        | Meaning                                          |
|-----------------|--------------------------------------------------|
| `TARGET_ZCODE`  | Compiling for the Z-machine                      |
| `TARGET_GLULX`  | Compiling for Glulx                              |

These are set by the compiler, not by the game source. The library uses
them extensively in `#Ifdef` blocks to handle platform differences.

## 16.8 Library Objects

The library defines several built-in objects that form the core of the
runtime infrastructure.

### 16.8.1 `InformLibrary`

The main loop controller. Its `play` method (§16.4) is called from `Main`
and drives the entire game. It also provides the `begin_action`,
`actor_act`, and `end_turn_sequence` methods.

```inform6
Object  InformLibrary "(Inform Library)"
  with  play [ ... ],
        end_turn_sequence [ ... ],
        actor_act [ ... ],
        begin_action [ ... ],
  has   proper;
```

### 16.8.2 `InformParser`

The parser engine. Its `parse_input` method reads a line of player input,
tokenizes it, matches it against the grammar, resolves ambiguity, and
writes the results into the `inputobjs` array. The parser is covered in
detail in Chapter 19.

### 16.8.3 `Compass`

The parent object for all direction objects (`n_obj`, `s_obj`, `e_obj`,
etc.). The parser checks children of `Compass` when resolving direction
words. The `Compass` object and the individual direction objects are
defined by the language definition file (`english.h`).

### 16.8.4 `thedark`

A pseudo-room representing darkness. When the player is in an unlit
location, the library sets `location` to `thedark` so that the status line
displays "Darkness" and room contents are hidden. The real room is always
available in the global `real_location`:

```inform6
Object  thedark "(darkness object)"
  with  initial 0,
        short_name DARKNESS__TX,
        description [; return L__M(##Miscellany, 17); ];
```

### 16.8.5 `selfobj` (the Player)

The default player character, an instance of `SelfClass`:

```inform6
Class  SelfClass
  with  short_name YOURSELF__TX,
        description [; return L__M(##Miscellany, 19); ],
        before NULL, after NULL, life NULL,
        each_turn NULL, time_out NULL, describe NULL,
        capacity 100,
        narrative_voice 2,
        narrative_tense PRESENT_TENSE,
        nameless true,
  has   concealed animate proper transparent;

SelfClass selfobj "(self object)";
```

The `player` global initially points to `selfobj`. A game can replace the
player character by setting `player` to a different `animate` object in
`Initialise`. The library sets `selfobj.capacity` to `MAX_CARRIED` during
`GamePrologue`.

## 16.9 Library Extensions

The `LibraryExtensions` object provides a hook-based mechanism for
extending library behavior without modifying library source. An extension
object is placed inside `LibraryExtensions` and provides properties that
match named extension points:

```inform6
Object  MyExtension "(my extension)" LibraryExtensions
  with  ext_initialise [;
            ! Code to run during game initialisation
        ],
        ext_afterlife [;
            ! Code to run when the player dies
        ];
```

### 16.9.1 Execution Methods

`LibraryExtensions` provides three methods for invoking hooks:

- **`RunAll(prop)`** — calls `prop` on every child object that provides
  it. All extensions run regardless of return values.
- **`RunUntil(prop, exitval)`** — calls `prop` on children until one
  returns `exitval`, then stops.
- **`RunWhile(prop, exitval)`** — calls `prop` on children as long as
  they return `exitval`, stopping when one returns something different.

### 16.9.2 Available Extension Points

The library defines extension points across the full range of library
operations. Each hook has a calling convention indicated by two codes:

- **C1** = called in all cases; **C2** = called if the corresponding
  entry point is undefined or returns false; **C3** = called if undefined
  or returns −1.
- **R1** = all extensions run; **R2** = extensions run while returning
  false; **R3** = extensions run while returning −1.

Selected extension points include:

| Property                | Convention | Purpose                         |
|-------------------------|------------|---------------------------------|
| `ext_initialise`        | C1/R1      | Runs before `Initialise`        |
| `ext_gamepreroutine`    | C2/R2      | Alternative to `GamePreRoutine` |
| `ext_gamepostroutine`   | C2/R2      | Alternative to `GamePostRoutine`|
| `ext_timepasses`        | C2/R1      | Runs if `TimePasses` returns 0  |
| `ext_afterlife`         | C2/R1      | Runs after `AfterLife`          |
| `ext_beforeparsing`     | C2/R2      | Runs before each parse attempt  |
| `ext_newroom`           | C2/R1      | Runs when entering a new room   |
| `ext_deathmessage`      | C2/R1      | Custom death messages           |
| `ext_amusing`           | C2/R1      | Custom amusing text at victory  |
| `ext_parsererror`       | C2/R2      | Custom parser error handling    |

> **[Glulx]** Additional Glulx-specific extension points are available:
> `ext_handleglkevent`, `ext_identifyglkobject`, and `ext_initglkwindow`.
> These allow extensions to participate in Glk event handling and window
> management.

The `LibraryExtensions` mechanism allows multiple independent extensions to
coexist without conflicting, since each extension object can provide its own
set of hooks. The `BetweenCalls` property can be set to a routine (such as
the built-in `RestoreWN`) that is called between successive extension
invocations to reset shared state.
