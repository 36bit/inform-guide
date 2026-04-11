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

# Chapter 20: Actions and the Action Processing Pipeline

Every command the player types — and every command issued by game code —
ultimately becomes an **action**. Actions are the central mechanism by
which the game world changes. This chapter describes how actions are
defined, how the library processes them from raw parser output to final
result, and how game authors can intercept, modify, or initiate actions
at every stage of the pipeline. The information here is derived from
`parser.h`, `verblib.h`, and `grammar.h` in library version 6.12.8.

## 20.1 What Is an Action?

An action is a named game-world event. When the player types `TAKE
LAMP`, the parser identifies the verb and the object, then triggers the
`Take` action with `lamp` as the noun. The library dispatches the action
to a routine called `TakeSub`, which carries out the effect.

Every action has three components:

1. **An action number** — a compile-time constant written `##ActionName`.
   The compiler assigns these automatically.
2. **An action routine** — a function named `ActionNameSub` that
   implements the default behaviour.
3. **A grammar pattern** (optional) — one or more grammar lines in a
   `Verb` directive that tell the parser how to recognise the action from
   player input.

When an action fires, the library sets several global variables that
describe it:

| Variable | Purpose |
|----------|---------|
| `action` | The action number (e.g. `##Take`). |
| `noun`   | The first object, or a numeric value, or `0`. |
| `second` | The second object, or a numeric value, or `0`. |
| `inp1`   | Like `noun`, but `1` when the value is a number. |
| `inp2`   | Like `second`, but `1` when the value is a number. |
| `actor`  | The person performing the action (usually `player`). |
| `meta`   | True if the action is a meta-action (see §20.5). |

The distinction between `noun`/`second` and `inp1`/`inp2` matters when
the parser reads a numeric value: `noun` holds the number itself, while
`inp1` is set to `1` to indicate "this is a number, not an object."
For object references, `noun` and `inp1` hold the same value.

```inform6
! Testing what action is being performed:
before [;
    Take:
        if (noun == golden_idol)
            "You hesitate — the idol looks trapped.";
    Lock, Unlock:
        "This object has no keyhole.";
],
```

## 20.2 The Full Processing Sequence

When the player types a command and presses Enter, a precisely ordered
sequence of events occurs. Understanding this sequence is essential for
writing correct `before` and `after` handlers.

### Step 1: Parsing

The parser reads the input, matches it against grammar lines, and
resolves noun phrases to objects. On success it sets `action`, `noun`,
`second`, and related variables. (Chapter 21 covers parsing in detail.)

### Step 2: Action Transformations

Before dispatching, the library applies several transformations to
normalise reversed and redirected actions:

- **`GiveR` → `Give`** and **`ShowR` → `Show`**: Grammar lines using
  `reverse` (e.g. `* creature held -> Give reverse`) produce `GiveR`
  and `ShowR`. The library swaps `noun` and `second` and changes the
  action to `Give` or `Show`.
- **`Tell` redirect**: `TELL NPC ABOUT X` becomes `ASK NPC ABOUT X`
  when the NPC is the actor.
- **`AskFor` redirect**: `ASK NPC FOR X` becomes an order to the NPC
  to `GIVE X TO ME`.

```inform6
! In parser.h the transformation looks like this (simplified):
if (action == ##GiveR or ##ShowR) {
    ! Swap noun and second
    i = inputobjs-->2; inputobjs-->2 = inputobjs-->3;
    inputobjs-->3 = i;
    if (action == ##GiveR) action = ##Give;
    else                   action = ##Show;
}
```

### Step 3: NPC Orders

If `actor` is not the player (e.g. the command was `TROLL, TAKE SWORD`),
the library routes the action through `actor_act()` instead of the
normal pipeline. This routine:

1. Calls `player.orders` — if it returns true, the order is handled.
2. Calls `actor.orders` — if it returns true, the NPC handles it.
3. Falls back to `RunLife(actor, ##Order)` for old-style `life`
   property handling.

If none of these handle the order, the library prints a refusal message.

### Step 4: `begin_action()`

For actions performed by the player (or actions invoked from code), the
library calls `begin_action()`. This is the heart of the pipeline:

#### 4a. `BeforeRoutines()`

The library calls the following handlers **in order**. If any handler
returns `true`, the action is **blocked** — `ActionPrimitive()` is never
called, and no `after` processing occurs.

1. **`GamePreRoutine()`** — a game-wide hook. If the game defines this
   entry-point routine and it returns `true`, the action is blocked.
2. **`LibraryExtensions.RunWhile(ext_gamepreroutine)`** — extensions get
   a chance to intercept the action.
3. **`player.orders`** — the player object's `orders` property is
   checked (this handles some special cases).
4. **Scope search for `react_before`** — every object in scope that has
   a `react_before` property gets a chance to react. If any returns
   `true`, the action is blocked.
5. **`location.before`** — the current room's `before` property is
   called. Return `true` to block the action.
6. **`inp1.before`** (i.e. `noun.before`) — the first object's `before`
   property is called. Return `true` to block the action.

```inform6
! Simplified from parser.h:
[ BeforeRoutines rv;
    if (GamePreRoutine()) rtrue;
    rv = LibraryExtensions.RunWhile(ext_gamepreroutine, 0);
    if (rv) rtrue;
    if (RunRoutines(player, orders)) rtrue;
    scope_reason = REACT_BEFORE_REASON;
    parser_one = 0;
    SearchScope(ScopeCeiling(player), player, 0);
    scope_reason = PARSING_REASON;
    if (parser_one) rtrue;
    if (location && RunRoutines(location, before)) rtrue;
    if (inp1 > 1 && RunRoutines(inp1, before)) rtrue;
    rfalse;
];
```

#### 4b. `ActionPrimitive()`

If no `before` handler blocked the action, the library dispatches to
the action routine (`ActionNameSub`) via a lookup in the actions table:

```inform6
[ ActionPrimitive;
    (#actions_table-->action)();
];
```

The action routine carries out the default behaviour — printing messages,
moving objects, changing attributes, and so on.

#### 4c. `AfterRoutines()`

After the action routine completes, the library calls `after` handlers.
Returning `true` from an `after` handler suppresses the default success
message (the action itself has already occurred).

1. **Scope search for `react_after`** — every object in scope with a
   `react_after` property is checked.
2. **`location.after`** — the current room's `after` property.
3. **`inp1.after`** (i.e. `noun.after`) — the first object's `after`
   property.
4. **`GamePostRoutine()`** — a game-wide post-hook.
5. **`LibraryExtensions.RunWhile(ext_gamepostroutine)`** — extensions
   get a final chance to react.

```inform6
! Simplified from parser.h:
[ AfterRoutines rv;
    scope_reason = REACT_AFTER_REASON;
    parser_one = 0;
    SearchScope(ScopeCeiling(player), player, 0);
    scope_reason = PARSING_REASON;
    if (parser_one) rtrue;
    if (location && RunRoutines(location, after)) rtrue;
    if (inp1 > 1 && RunRoutines(inp1, after)) rtrue;
    rv = GamePostRoutine();
    if (rv == false)
        rv = LibraryExtensions.RunWhile(ext_gamepostroutine, false);
    return rv;
];
```

### Step 5: End-of-Turn Sequence

For non-meta actions only, the library runs the end-of-turn sequence
after action processing completes. At each stage, if `deadflag` becomes
non-zero (the player has died or won), the sequence halts.

1. **`AdvanceWorldClock()`** — increment `turns` and advance the clock.
2. **`RunTimersAndDaemons()`** — fire all active timers and daemons.
3. **`RunEachTurnProperties()`** — call `each_turn` on relevant objects.
4. **`TimePasses()`** — game entry point for end-of-turn processing.
5. **`LibraryExtensions.RunAll(ext_timepasses)`** — extensions hook.
6. **`AdjustLight()`** — recalculate light and darkness.
7. **`NoteObjectAcquisitions()`** — track newly acquired objects for
   scoring.

## 20.3 Before and After Processing

The `before` and `after` properties are the primary way game authors
customise action behaviour. Both are **additive** properties, meaning
inherited values from class definitions are combined rather than
replaced.

### Blocking with `before`

A `before` handler receives the action in the global variable `action`
and can test it with a `switch` statement or action-name labels.
Returning `true` blocks the action entirely — the action routine is
never called.

```inform6
Object  rusty_gate "rusty gate" garden
  with  name 'rusty' 'gate' 'iron',
        before [;
            Open:
                if (self hasnt general) {
                    give self general;
                    "The gate screeches as you force it open
                     for the first time.";
                }
            Push, Pull:
                "The gate won't budge that way.";
        ],
  has   door openable locked;
```

When the handler prints a string (which implicitly returns `true`), the
action is blocked and no default message is printed.

### Suppressing messages with `after`

An `after` handler runs **after** the action has taken effect. Returning
`true` tells the library to suppress its default success message,
allowing the game to substitute a custom one.

```inform6
Object  diamond "diamond" cave
  with  name 'diamond' 'gem' 'stone',
        after [;
            Take:
                "You carefully pocket the diamond,
                 your hands trembling.";
            Drop:
                "The diamond clatters on the stone floor.";
        ],
  has   scored;
```

### Room-Level Handling

The room's `before` and `after` properties handle actions that occur
within the room, regardless of which object is involved:

```inform6
Object  library "Library" 
  with  description "Walls of books surround you.",
        before [;
            Take:
                if (noun == rare_book)
                    "The librarian glares at you warningly.";
        ],
        after [;
            Drop:
                "It thuds onto the library carpet.";
        ],
  has   light;
```

### Scope-Wide Reactions

The `react_before` and `react_after` properties let an object react to
any action that occurs while it is in scope — even actions directed at
other objects:

```inform6
Object  guard "burly guard" throne_room
  with  name 'burly' 'guard',
        react_before [;
            Take:
                if (noun == crown)
                    "~Hands off the crown!~ bellows the guard.";
        ],
  has   animate;
```

## 20.4 The `life` Property

The `life` property is called on **animate** objects when certain actions
target them directly. It works like `before`, but is specific to
person-oriented actions. The library calls `RunLife(object, ##Action)`
from inside the relevant action routines.

Actions that trigger `life`:

| Action | Called on | Typical use |
|--------|-----------|-------------|
| `##Give` | `second` | NPC receives an object |
| `##Show` | `second` | NPC is shown an object |
| `##Ask` | `noun` | NPC is asked about a topic |
| `##Tell` | `noun` | NPC is told about a topic |
| `##Answer` | `second` | NPC hears speech |
| `##Attack` | `noun` | NPC is attacked |
| `##Kiss` | `noun` | NPC is kissed |
| `##WakeOther` | `noun` | NPC is woken up |
| `##ThrowAt` | `second` | Something is thrown at NPC |
| `##Order` | `actor` | NPC receives a command |

```inform6
Object  wizard "old wizard" tower
  with  name 'old' 'wizard' 'mage',
        life [;
            Give:
                if (noun == spellbook) {
                    remove spellbook;
                    "~Ah, my spellbook! Thank you!~
                     The wizard smiles warmly.";
                }
                "~I have no use for that,~ he says.";
            Ask:
                if (second == 'dragon' or 'beast')
                    "~The dragon? It lurks in the eastern caves.~";
                "The wizard shrugs.";
            Attack:
                "The wizard vanishes in a puff of smoke!";
            default:
                rfalse;  ! Let the library handle it
        ],
  has   animate;
```

If `life` returns `true`, the action is considered handled. If it
returns `false`, the library falls through to its default response.

## 20.5 Meta vs Non-Meta Actions

**Meta actions** are out-of-world commands that do not consume a game
turn. They do not trigger `before`/`after` processing or the end-of-turn
sequence (no timers, daemons, or `each_turn` routines fire).

Meta actions are defined with the `Verb meta` directive:

```inform6
Verb meta 'score'
    *                       -> Score;

Verb meta 'save'
    *                       -> Save;
```

Inside `begin_action()`, the library checks the `meta` flag. When it is
set, `BeforeRoutines()` is skipped entirely and the action routine is
called directly. After the routine returns, no `AfterRoutines()` or
end-of-turn processing occurs.

The standard library defines these meta actions:

| Action | Verb words |
|--------|------------|
| `LMode1` | `brief` |
| `LMode2` | `verbose`, `long` |
| `LMode3` | `superbrief`, `short` |
| `LModeNormal` | `normal` |
| `NotifyOn` / `NotifyOff` | `notify` |
| `Pronouns` | `pronouns`, `nouns` |
| `Quit` | `quit`, `q`, `die` |
| `Restart` | `restart` |
| `Restore` | `restore` |
| `Save` | `save` |
| `Score` | `score` |
| `FullScore` | `fullscore`, `full` |
| `ScriptOn` / `ScriptOff` | `script`, `transcript`, `noscript` |
| `CommandsOn` / `CommandsOff` / `CommandsRead` | `recording`, `replay` |
| `Verify` | `verify` |
| `Version` | `version` |
| `Places` | `places` |
| `Objects` | `objects` |

Debug-mode meta actions (compiled only with `DEBUG`) include `trace`,
`actions`, `routines`, `timers`, `changes`, `showobj`, `showverb`,
`showdict`, `scope`, `goto`, `gonear`, `abstract`, `purloin`, `tree`,
and `glklist` (Glulx only).

## 20.6 Fake Actions

A **fake action** has an action number but no grammar and no `Sub`
routine. Fake actions are declared with `Fake_action`:

```inform6
Fake_action LetGo;
Fake_action Receive;
Fake_action ThrownAt;
```

Fake actions are used internally by the library to signal special
situations via `before` and `after` handlers. The game author can test
for them exactly like real actions:

```inform6
before [;
    Receive:
        if (noun == something_heavy)
            "The shelf can't support that much weight.";
    LetGo:
        "You can't remove items from the display case.";
],
```

The standard library declares these fake actions in `parser.h`:

| Fake action | Purpose |
|-------------|---------|
| `LetGo` | An object is being removed from a container. Sent to the container's `before`. |
| `Receive` | An object is being placed in a container or on a supporter. Sent to the container/supporter's `before`. |
| `ThrownAt` | An object has been thrown at this object. |
| `Order` | An NPC is being given an order. Sent to `life`. |
| `TheSame` | Disambiguation: the parser is comparing two objects. Sent to `parse_name`. |
| `PluralFound` | Signals that `parse_name` matched a plural word. |
| `ListMiscellany` | Internal: library message selection for object lists. |
| `Miscellany` | Internal: general library messages. |
| `RunTimeError` | Internal: run-time error messages. |
| `Prompt` | The library is about to print the command prompt. |
| `NotUnderstood` | The parser could not understand the command (sent to NPC `orders`). |
| `Going` | Sent to room `before` when the player is about to leave via `Go`. Contains direction information. |
| `Places` | Internal: used by the `places` meta-verb. |
| `Objects` | Internal: used by the `objects` meta-verb. |

## 20.7 Calling Actions from Code

Game code can trigger actions directly using angle-bracket notation.
This is essential for redirecting actions, implementing complex
behaviours, and creating cause-and-effect chains.

### Basic Forms

```inform6
<Look>;                    ! Perform Look with no objects
<Examine mirror>;          ! Perform Examine on the mirror
<PutOn ball table>;        ! Perform PutOn with ball and table
```

Each of these calls `begin_action()`, running the full pipeline
including `before` and `after` handlers.

### Return-True Form

Double angle brackets perform the action **and** return `true`. This is
commonly used inside `before` handlers to redirect one action to
another:

```inform6
before [;
    Push:
        <<Pull self>>;     ! Redirect Push to Pull and block
];
```

The `<<...>>` form is equivalent to `<...>; rtrue;`.

### Directed Actions

An action can be directed at a specific actor by adding the actor after
a comma:

```inform6
<Give sword, troll>;       ! The troll gives the sword (to player)
<Take lamp, servant>;      ! Order the servant to take the lamp
```

### Summary of Syntax

| Syntax | Effect |
|--------|--------|
| `<Action>` | Perform action, no objects |
| `<Action noun>` | Perform action with first object |
| `<Action noun second>` | Perform action with both objects |
| `<<Action>>` | Perform action and return `true` |
| `<<Action noun>>` | Perform with noun and return `true` |
| `<<Action noun second>>` | Perform with both and return `true` |
| `<Action noun, actor>` | Directed at actor |
| `<Action noun second, actor>` | Both objects, directed at actor |

### Practical Examples

```inform6
! A button that triggers a Look when pushed:
Object  red_button "red button" control_room
  with  name 'red' 'button',
        before [;
            Push:
                print "Click! The lights flicker.^";
                <Look>;    ! Re-describe the room
                rtrue;     ! Block the default Push message
        ],
  has   scenery;

! A magic mirror that redirects Examine to Search:
Object  magic_mirror "mirror" hallway
  with  name 'magic' 'mirror',
        before [;
            Examine:
                <<Search self>>;
        ];
```

## 20.8 Action Groups

The library organises actions into three groups based on their nature.
The grouping is determined by how the action is declared in `grammar.h`.

### Group 1: Standard Actions

These are the regular, in-world actions that advance the game clock.
They go through the full processing pipeline including `before`/`after`
handlers and end-of-turn processing. Examples: `Take`, `Drop`, `Look`,
`Go`, `Open`, `Examine`.

### Group 2: Meta Actions

Out-of-world commands declared with `Verb meta`. They do not consume a
game turn and bypass `before`/`after` processing. Examples: `Score`,
`Save`, `Quit`, `Version`, `Undo`.

### Group 3: Debug Actions

Available only when the game is compiled with the `DEBUG` flag. These
are always meta actions. Examples: `XPurloin`, `XAbstract`, `XTree`,
`Goto`, `GoNear`, `Scope`, `TraceOn`.

## 20.9 Complete Action Listing

The following table lists every action defined by the standard library
(version 6.12.8). "Group" indicates: **1** = standard, **2** = meta,
**3** = debug (meta). "Objects" shows the expected noun/second pattern.
"F" marks fake actions (no grammar, no `Sub` routine).

### Standard Actions (Group 1)

| Action | Objects | Verb words |
|--------|---------|------------|
| `Answer` | topic → creature | `answer`, `say`, `shout`, `speak` |
| `Ask` | creature, topic | `ask` |
| `AskFor` | creature, noun | `ask` |
| `AskTo` | creature, topic | `ask`, `tell` |
| `Attack` | noun | `attack`, `break`, `crack`, `destroy`, `fight`, `hit`, `kill`, `murder`, `punch`, `smash`, `thump`, `torture`, `wreck` |
| `Blow` | held | `blow` |
| `Burn` | noun (held) | `burn`, `light` |
| `Buy` | noun | `buy`, `purchase` |
| `Climb` | noun | `climb`, `scale` |
| `Close` | noun | `close`, `cover`, `shut` |
| `Consult` | noun, topic | `consult`, `look`, `read` |
| `Cut` | noun | `cut`, `chop`, `prune`, `slice` |
| `Dig` | noun (held) | `dig` |
| `Disrobe` | worn/multiheld | `disrobe`, `doff`, `shed`, `take`, `remove` |
| `Drink` | noun | `drink`, `sip`, `swallow` |
| `Drop` | multiheld | `drop`, `discard`, `put` |
| `Eat` | held | `eat` |
| `Empty` | noun | `empty` |
| `EmptyT` | noun, noun | `empty` |
| `Enter` | noun | `enter`, `cross`, `get`, `go`, `sit`, `lie`, `stand` |
| `Examine` | noun | `examine`, `x`, `check`, `describe`, `watch`, `look`, `read` |
| `Exit` | (noun) | `exit`, `out`, `outside`, `get`, `stand`, `leave` |
| `Fill` | noun (noun) | `fill` |
| `GetOff` | noun | `get` |
| `Give` | held → creature | `give`, `feed`, `offer`, `pay` |
| `Go` | direction | `go`, `run`, `walk`, `leave` |
| `GoIn` | — | `enter`, `in`, `inside` |
| `Inv` | — | `inventory`, `inv`, `i`, `take`, `carry`, `hold` |
| `InvTall` | — | `inventory` |
| `InvWide` | — | `inventory` |
| `Insert` | multiexcept → noun | `insert`, `drop`, `put` |
| `Jump` | — | `jump`, `hop`, `skip` |
| `JumpIn` | noun | `jump` |
| `JumpOn` | noun | `jump` |
| `JumpOver` | noun | `jump` |
| `Kiss` | creature | `kiss`, `embrace`, `hug` |
| `Listen` | (noun) | `listen`, `hear` |
| `Lock` | noun, held | `lock` |
| `Look` | — | `look`, `l` |
| `LookUnder` | noun | `look` |
| `Mild` | — | `bother`, `curses`, `darn`, `drat` |
| `No` | — | `no` |
| `Open` | noun | `open`, `uncover`, `undo`, `unwrap` |
| `Pray` | — | `pray` |
| `Pull` | noun | `pull`, `drag` |
| `Push` | noun | `push`, `clear`, `move`, `press`, `shift` |
| `PushDir` | noun, noun | `push`, `move` |
| `PutOn` | multiexcept → noun | `drop`, `put` |
| `Remove` | multiinside, noun | `remove`, `get`, `take`, `carry`, `hold` |
| `Rub` | noun | `rub`, `clean`, `dust`, `polish`, `scrub`, `shine`, `sweep`, `wipe` |
| `Search` | noun | `search`, `look` |
| `Set` | noun | `set`, `adjust` |
| `SetTo` | noun, special | `set`, `adjust` |
| `Show` | held → creature | `show`, `display`, `present` |
| `Sing` | — | `sing` |
| `Sleep` | — | `sleep`, `nap` |
| `Smell` | (noun) | `smell`, `sniff` |
| `Sorry` | — | `sorry` |
| `Squeeze` | noun | `squeeze`, `squash` |
| `Strong` | — | profanity verbs |
| `Swim` | — | `swim`, `dive` |
| `Swing` | noun | `swing` |
| `SwitchOff` | noun | `switch`, `close`, `turn` |
| `SwitchOn` | noun | `switch`, `turn` |
| `Take` | multi | `take`, `get`, `carry`, `hold`, `pick`, `peel` |
| `Taste` | noun | `taste` |
| `Tell` | creature, topic | `tell` |
| `Think` | — | `think` |
| `ThrowAt` | held → noun | `throw` |
| `Tie` | noun (noun) | `tie`, `attach`, `connect`, `fasten`, `fix` |
| `Touch` | noun | `touch`, `feel`, `fondle`, `grope` |
| `Transfer` | noun → noun | `transfer`, `push` |
| `Turn` | noun | `turn`, `rotate`, `screw`, `twist`, `unscrew` |
| `Unlock` | noun, held | `unlock`, `open`, `pry`, `prise`, `prize`, `lever`, `jemmy`, `force` |
| `VagueGo` | — | `go`, `run`, `walk`, `leave` |
| `Wait` | — | `wait`, `z` |
| `Wake` | — | `wake`, `awake`, `awaken` |
| `WakeOther` | creature | `wake`, `awake`, `awaken` |
| `Wave` | noun | `wave` |
| `WaveHands` | — | `wave` |
| `Wear` | multiheld | `wear`, `don`, `put` |
| `Yes` | — | `yes`, `y` |

### Meta Actions (Group 2)

| Action | Verb words |
|--------|------------|
| `CommandsOff` | `recording` |
| `CommandsOn` | `recording` |
| `CommandsRead` | `replay` |
| `FullScore` | `fullscore`, `full` |
| `LMode1` | `brief` |
| `LMode2` | `verbose`, `long` |
| `LMode3` | `superbrief`, `short` |
| `LModeNormal` | `normal` |
| `NotifyOff` | `notify` |
| `NotifyOn` | `notify` |
| `Objects` | `objects` |
| `Places` | `places` |
| `Pronouns` | `pronouns`, `nouns` |
| `Quit` | `quit`, `q`, `die` |
| `Restart` | `restart` |
| `Restore` | `restore` |
| `Save` | `save` |
| `Score` | `score` |
| `ScriptOff` | `script`, `noscript`, `unscript` |
| `ScriptOn` | `script`, `transcript` |
| `Verify` | `verify` |
| `Version` | `version` |

### Debug Actions (Group 3)

| Action | Verb words |
|--------|------------|
| `ActionsOn` / `ActionsOff` | `actions` |
| `ChangesOn` / `ChangesOff` | `changes` |
| `Glklist` | `glklist` (Glulx only) |
| `GoNear` | `gonear` |
| `Goto` | `goto` |
| `Predictable` | `random` |
| `RoutinesOn` / `RoutinesOff` / `RoutinesVerbose` | `routines`, `messages` |
| `Scope` | `scope` |
| `ShowDict` | `showdict`, `dict` |
| `ShowObj` | `showobj` |
| `ShowVerb` | `showverb` |
| `TimersOn` / `TimersOff` | `timers`, `daemons` |
| `TraceOn` / `TraceOff` / `TraceLevel` | `trace` |
| `XAbstract` | `abstract` |
| `XPurloin` | `purloin` |
| `XTree` | `tree` |

### Fake Actions

| Fake action | Purpose |
|-------------|---------|
| `Going` | Sent to room `before` when the player is leaving. |
| `LetGo` | Sent to container `before` when an object is removed. |
| `ListMiscellany` | Internal: object-list message selection. |
| `Miscellany` | Internal: general library messages. |
| `NotUnderstood` | Sent to NPC `orders` when the command is unrecognised. |
| `Objects` | Internal: used by `objects` meta-verb. |
| `Order` | Sent to NPC `life` when given a command. |
| `Places` | Internal: used by `places` meta-verb. |
| `PluralFound` | Internal: `parse_name` signalling. |
| `Prompt` | Sent before the command prompt is printed. |
| `Receive` | Sent to container/supporter `before` when receiving an object. |
| `RunTimeError` | Internal: run-time error messages. |
| `TheSame` | Internal: disambiguation comparison. |
| `ThrownAt` | Sent to target when an object is thrown at it. |
