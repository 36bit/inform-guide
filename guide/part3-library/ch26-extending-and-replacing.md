<!--
  Copyright (C) 2026 Software Freedom Conservancy, Inc.
  This file is part of The Inform 6 Programmer's Guide.

  The Inform 6 Programmer's Guide is free software: you can redistribute it
  and/or modify it under the terms of the GNU General Public License as
  published by the Free Software Foundation, either version 3 of the License,
  or (at your option) any later version.

  The Inform 6 Programmer's Guide is distributed in the hope that it will be
  useful, but WITHOUT ANY WARRANTY; without even the implied warranty of
  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General
  Public License for more details.

  You should have received a copy of the GNU General Public License along with
  The Inform 6 Programmer's Guide. If not, see <https://www.gnu.org/licenses/>.
-->

# Chapter 26: Extending and Replacing

## 26.1 Overview

The Inform 6 library is deliberately designed to be customized. Rather than
requiring you to modify library source files directly — which would make
upgrades difficult and introduce maintenance headaches — the library offers
several well-defined mechanisms for altering its behavior from your game code:

- **Replace directive** — completely replace a library routine with your own
  version.
- **Extend directive** — add new grammar lines to existing verbs, or replace
  their grammar entirely.
- **Before/after hooks** — intercept actions at various levels, from individual
  objects up to global game-wide routines.
- **Entry points** — game-defined routines that the library calls at specific
  moments if they exist.
- **LibraryMessages** — customize the text of any message the library prints
  (covered in detail in Chapter 27).
- **Compilation constants** — configure library behavior by defining constants
  before including the library files.

These mechanisms form a layered system. The lightest-weight approach (setting a
constant or writing a `before` rule) should be preferred when it suffices; the
heaviest (replacing an entire routine) should be reserved for cases where no
other approach will do.

## 26.2 The Replace Directive

The `Replace` directive tells the compiler that your definition of a routine
should be used instead of the library's definition. Any library routine can be
replaced this way.

### Basic Replacement

```inform6
Replace Banner;

Include "Parser";
Include "VerbLib";

[ Banner;
    print "^^Welcome to MY GAME^^";
    print "An interactive fiction by A. N. Author^^";
];
```

The `Replace` declaration must appear **before** the `Include` directive that
brings in the file containing the routine you are replacing. If you place it
after, the compiler will see two definitions and report an error.

### Replacement with Renaming (Inform 6.34+)

Starting with Inform 6.34, you can replace a routine while keeping the original
accessible under a new name:

```inform6
Replace Banner OriginalBanner;

Include "Parser";
Include "VerbLib";

[ Banner;
    OriginalBanner();
    print "^Release date: January 2026^";
];
```

This is extremely useful when you want to *wrap* or *augment* the library's
behavior rather than rewrite it from scratch. Your replacement calls the
original under its new name, then adds whatever extra behavior you need.

### Commonly Replaced Routines

Some routines are replaced more often than others:

| Routine          | Purpose                                    |
|------------------|--------------------------------------------|
| `Banner`         | Print the game banner at startup           |
| `DrawStatusLine` | Render the status bar                      |
| `LookSub`        | Handle the Look action                     |
| `InvSub`         | Handle the Inventory action                |
| `DarkToDark`     | React when player moves in darkness        |
| `PrintRank`      | Print the player's score rank              |
| `ParseNoun`      | Custom noun-parsing logic                  |

### Example: Custom Status Line

```inform6
Replace DrawStatusLine;

[ DrawStatusLine width;
    @split_window 1;
    @set_window 1;
    @set_cursor 1 1;
    style reverse;
    width = 0->33;
    spaces width;
    @set_cursor 1 2;
    print (name) location;
    @set_cursor 1 (width - 20);
    print "HP: ", player_hp, "/", player_max_hp;
    @set_cursor 1 1;
    style roman;
    @set_window 0;
];
```

## 26.3 Extending Actions with Extend

The `Extend` directive lets you modify the grammar table for existing verbs.
This is the correct way to add new syntax to built-in verbs or to redefine
their grammar entirely.

### Adding Grammar Lines

The simplest form appends a new grammar line to an existing verb:

```inform6
Extend 'examine' * edible -> Taste;
```

This adds a grammar line so that `examine [something edible]` is now also
parsed as the `Taste` action. New lines added this way have **lower** priority
than existing lines — the parser tries the original grammar first.

### Adding Lines with Higher Priority

Use the `first` keyword to insert grammar at the **beginning** of the verb's
grammar table, giving it higher priority:

```inform6
Extend 'examine' first
    * readable -> Read;
```

Now `examine [something readable]` will be parsed as `Read` before any of the
original `Examine` grammar is tried.

### Replacing All Grammar

The `replace` keyword removes all existing grammar for the verb and substitutes
yours:

```inform6
Extend 'take' replace
    * multi -> Take
    * multiinside 'from' noun -> Remove
    * 'inventory' -> Inv;
```

Use this with caution — you are responsible for providing complete grammar for
the verb.

### The `only` Keyword

By default, `Extend` modifies all synonyms of a verb. If `take` and `get` are
synonyms, extending `take` also extends `get`. The `only` keyword restricts the
change to the specific word named:

```inform6
Extend only 'get' first
    * 'up' -> Exit
    * 'off' noun -> GetOff;
```

This adds lines only to `get`, leaving `take` untouched.

### Placement

`Extend` directives must appear **after** the grammar has been defined — that
is, after `Include "Grammar"`:

```inform6
Include "Parser";
Include "VerbLib";
Include "Grammar";

Extend 'read' first
    * noun -> Examine;
```

### Example: Adding a "Read" Synonym

```inform6
Include "Grammar";

Extend 'read' replace
    * noun -> Examine;

Extend only 'examine'
    * readable -> Read;
```

## 26.4 Before and After Hooks

The library's action-processing system provides multiple layers where you can
intercept an action before or after it takes effect. These layers are checked
in a specific order, giving you fine-grained control over which actions to
intercept and at what scope.

### Object-Level Hooks

The `before` property on an object intercepts actions directed at that
specific object:

```inform6
Object brass_lamp "brass lamp"
    with name 'brass' 'lamp' 'lantern',
         before [;
            Take:
                if (self in player)
                    "You're already carrying it.";
            Rub:
                "The lamp glows faintly as you rub it.";
         ],
    has  light;
```

The `after` property works similarly but runs only after the action has
succeeded:

```inform6
Object fragile_vase "fragile vase"
    with name 'fragile' 'vase',
         after [;
            Drop:
                remove self;
                "The vase shatters into a thousand pieces!";
         ],
    has ;
```

### Reactive Hooks

The `react_before` and `react_after` properties fire for **any** action, as
long as the object providing them is in scope. This is useful for "observer"
objects:

```inform6
Object security_camera "security camera"
    with name 'security' 'camera',
         react_before [;
            Take, Drop:
                print "The security camera whirs, tracking your movement.^";
         ],
    has  scenery;
```

### Global Hooks

Two entry-point routines provide game-wide interception:

- `GamePreRoutine()` — called before every action. Return `true` to block the
  action.
- `GamePostRoutine()` — called after every action that was not already handled.
  Return `true` to suppress the default message.

```inform6
[ GamePreRoutine;
    if (action == ##Sleep)
        "You're far too anxious to sleep right now.";
    rfalse;
];
```

### Processing Order

The library processes action interception in this order:

1. `GamePreRoutine()`
2. Player's `orders` property (for orders to NPCs)
3. `react_before` on objects in scope
4. `before` on the room
5. `before` on the direct object
6. — *the action itself executes* —
7. `after` on the direct object
8. `after` on the room
9. `react_after` on objects in scope
10. `GamePostRoutine()`

At any point, returning `true` stops further processing.

## 26.5 The SACK_OBJECT Constant

When the player's inventory reaches the `MAX_CARRIED` limit, attempts to pick
up additional items normally fail with a "You're carrying too many things"
message. The `SACK_OBJECT` mechanism provides an automatic workaround.

Define a container object and set the `SACK_OBJECT` constant to it before
including the library:

```inform6
Object rucksack "rucksack"
    with name 'rucksack' 'sack' 'bag',
         description "A sturdy canvas rucksack.",
    has  container open openable;

Constant SACK_OBJECT = rucksack;

Include "Parser";
Include "VerbLib";
```

With this in place, when the player tries to pick up an item that would exceed
`MAX_CARRIED`, the library automatically transfers an item from the player's
hands into the rucksack, printing a message like "(first putting the old
newspaper into the rucksack)".

The sack object **must** have the `container` attribute. It should normally also
be `open` or `openable` so the library can put things into it.

## 26.6 MAX_CARRIED and Other Constants

The library's behavior can be configured by defining constants before including
the library files. These constants act as compile-time configuration switches.

### Inventory and Scoring

| Constant          | Default | Purpose                                      |
|-------------------|---------|----------------------------------------------|
| `MAX_CARRIED`     | 100     | Maximum items player can carry directly       |
| `MAX_SCORE`       | 0       | Maximum possible score (for "out of" display) |
| `NUMBER_TASKS`    | 1       | Number of scoring tasks                       |
| `TASKS_PROVIDED`  | —       | Enable task-based scoring                     |
| `ROOM_SCORE`      | —       | Award points for visiting rooms with `scored` |
| `OBJECT_SCORE`    | —       | Award points for taking objects with `scored`  |

### Feature Flags

| Constant             | Effect                                           |
|----------------------|--------------------------------------------------|
| `AMUSING_PROVIDED`   | Offer "amusing" option at successful game end     |
| `NO_PLACES`          | Suppress the `places` and `objects` meta-verbs    |
| `MANUAL_PRONOUNS`    | Disable automatic pronoun (`it`/`him`/`her`) updates |
| `WITHOUT_DIRECTIONS` | Omit compass direction objects from the world model |
| `SACK_OBJECT`        | Enable automatic sack for inventory overflow       |
| `DEATH_MENTION_UNDO` | Mention UNDO as an option when the player dies     |

### Example: Task-Based Scoring

```inform6
Constant MAX_SCORE = 50;
Constant NUMBER_TASKS = 3;
Constant TASKS_PROVIDED;

Global task_scores --> 3;  ! array of point values for each task

[ Initialise;
    task_scores-->0 = 10;  ! Task 0: worth 10 points
    task_scores-->1 = 20;  ! Task 1: worth 20 points
    task_scores-->2 = 20;  ! Task 2: worth 20 points
    location = StartRoom;
];

[ AwardTask n;
    if (task_done-->n == false) {
        task_done-->n = true;
        score = score + task_scores-->n;
    }
];
```

### Example: Limiting Inventory

```inform6
Constant MAX_CARRIED = 7;
Constant SACK_OBJECT = backpack;

Object backpack "backpack"
    with name 'backpack' 'pack',
    has  container open openable worn;
```

## 26.7 Writing Library Extensions

The library provides a formal mechanism for extensions — reusable modules that
hook into the library's processing without requiring `Replace` directives. This
mechanism uses the `LibraryExtensions` object.

### The LibraryExtensions Object

An extension registers itself by adding itself to the `LibraryExtensions`
object's child list. The library then calls specific properties on each
registered extension at the appropriate times.

```inform6
Object MyExtension "(my extension)"
    with ext_initialise [;
            print "My extension initialized.^";
         ],
         ext_gamepreroutine [;
            ! Called before each action
            rfalse;
         ],
         ext_gamepostroutine [;
            ! Called after each action
            rfalse;
         ],
    has  ;
```

### Available Hook Properties

Extensions can provide any of the following properties:

| Property                | Called When                                    |
|-------------------------|------------------------------------------------|
| `ext_initialise`        | During game initialization                     |
| `ext_gamepreroutine`    | Before each action (like GamePreRoutine)        |
| `ext_gamepostroutine`   | After each action (like GamePostRoutine)        |
| `ext_afterlife`         | When the player dies (chance to intervene)       |
| `ext_timepasses`        | Each turn, after action processing              |
| `ext_beforeparsing`     | Before the parser processes input               |
| `ext_afterprompt`       | After printing the command prompt               |

### Registering an Extension

To register your extension, move it into the `LibraryExtensions` object during
initialization:

```inform6
Object MyExtension "(my extension)"
    with ext_initialise [;
            print "Hint system ready.^";
         ],
         ext_timepasses [;
            if (turns % 10 == 0 && hint_available)
                print "(Type HINT if you're stuck.)^";
            rfalse;
         ],
    has  ;

[ Initialise;
    move MyExtension to LibraryExtensions;
    location = StartRoom;
];
```

### The RunWhile Mechanism

When the library calls extension hooks, it uses a "run while" mechanism. For
most hooks, the library calls each extension's property in turn and stops if
any returns `true`. This mirrors the behavior of `before` properties — returning
`true` means "I've handled this, stop processing."

For `ext_afterlife`, returning `true` means the player has been resurrected and
the game should continue rather than ending.

### Example: A Hint Extension

```inform6
Object HintExtension "(hint extension)"
    with ext_initialise [;
            hint_count = 0;
         ],
         ext_gamepreroutine [;
            if (action == ##Hint) {
                self.give_hint();
                rtrue;
            }
            rfalse;
         ],
         give_hint [;
            switch (location) {
                Kitchen:
                    "Try looking under the table.";
                Garden:
                    "Have you examined the roses?";
                default:
                    "No hints available here.";
            }
         ],
    has  ;

Global hint_count;

[ HintSub; <<Hint>>; ];

Verb 'hint'
    * -> Hint;
```

### Example: A Turn Counter Extension

```inform6
Object TurnLimit "(turn limit)"
    with max_turns 200,
         ext_timepasses [;
            if (turns >= self.max_turns) {
                print "^Time has run out!^";
                deadflag = 3;
            }
            rfalse;
         ],
    has  ;
```

## 26.8 Best Practices

When customizing the library, follow these guidelines to keep your code
maintainable and portable.

### Prefer Lighter-Weight Mechanisms

Use the lightest mechanism that achieves your goal:

1. **Constants** — if a constant controls the behavior you want to change, set
   it. This is the simplest and least error-prone approach.
2. **Before/after hooks** — for intercepting specific actions on specific
   objects, these are clean and self-documenting.
3. **Entry points** — `GamePreRoutine`, `GamePostRoutine`, `Initialise`, and
   friends are the right place for game-wide behavior changes.
4. **LibraryMessages** — for changing what the library *says*, not what it
   *does*.
5. **Extend** — for grammar changes.
6. **Replace** — only when none of the above will work. Replacing a routine
   ties your code to a specific version of the library and may break when the
   library is updated.

### Use Replace with Renaming

When you must use `Replace`, prefer the renaming form so you can call the
original:

```inform6
Replace LookSub OriginalLookSub;

[ LookSub;
    OriginalLookSub();
    if (location has scored && location hasnt visited)
        print "^[You sense this is a significant place.]^";
];
```

This insulates you from changes to the library's implementation — if the
original `LookSub` is updated, your wrapper automatically picks up the changes.

### Test on Multiple Targets

If your game targets both the Z-machine and Glulx, test your customizations on
both platforms. Some replaced routines (especially `DrawStatusLine`) use
platform-specific opcodes:

```inform6
Replace DrawStatusLine;

[ DrawStatusLine;
    #Ifdef TARGET_ZCODE;
        ! Z-machine status line code
        @split_window 1;
        @set_window 1;
        ! ...
    #Ifnot; ! TARGET_GLULX
        ! Glulx status line code using glk calls
        if (gg_statuswin == 0) return;
        glk_set_window(gg_statuswin);
        ! ...
    #Endif;
];
```

### Document Your Customizations

When replacing or extending library behavior, add comments explaining *why*
the change is needed. Future maintainers (including yourself) will thank you:

```inform6
! Replace Banner to include the game's version number,
! which is stored in a global and may change between releases.
Replace Banner OriginalBanner;

[ Banner;
    OriginalBanner();
    print "Build ", (string) BUILD_VERSION, "^";
];
```

### Keep Extensions Self-Contained

Library extensions should be designed as drop-in modules. Avoid depending on
global variables defined elsewhere in the game. If your extension needs
configuration, use properties on the extension object:

```inform6
Object TimerExtension "(timer)"
    with max_turns 500,       ! configurable
         warning_at 450,      ! configurable
         ext_timepasses [;
            if (turns == self.warning_at)
                print "^You feel time running short.^";
            if (turns >= self.max_turns) {
                deadflag = 3;
                rtrue;
            }
            rfalse;
         ],
    has  ;
```

The game author can then customize the extension simply by changing its
properties in `Initialise`:

```inform6
[ Initialise;
    move TimerExtension to LibraryExtensions;
    TimerExtension.max_turns = 300;
    TimerExtension.warning_at = 250;
    location = StartRoom;
];
```
