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

# Chapter 18: Common Properties

The standard library (version 6.12.8) defines a set of **common
properties** in `linklpa.h` that are available on every object in the
game. These properties control how objects are named, described, parsed,
and how they interact with the player and the world model. This chapter
provides a complete reference for every library-defined common property.

## 18.1 Overview

Common properties are declared by the library using the `Property`
directive in `linklpa.h`. Because they are common properties (as opposed
to individual properties defined with `with`), every object in the game
has a slot for each one, initialized to a default value.

Key points about common properties:

- Each common property has a **default value**, typically `0`, `NULL`,
  or a specific constant such as `"a"` or `100`.
- Some properties are declared **additive** with `Property additive`.
  When a class defines an additive property and a subclass or instance
  also defines it, the values **accumulate** rather than being replaced.
  This is essential for action-handling properties like `before` and
  `after`.
- A property value can be a constant, a string, an object, a routine
  (embedded or named), or an array of values. The library interprets
  each property according to its own conventions.

```inform6
! From linklpa.h — examples of property declarations:
Property additive before NULL;      ! Additive, default NULL
Property article "a";               ! Non-additive, default "a"
Property capacity 100;              ! Non-additive, default 100
Property short_name 0;              ! Non-additive, default 0
```

A property whose default is `NULL` (the VM-specific all-bits-set value)
is treated differently from one whose default is `0`. The library tests
for `NULL` to determine whether a property has been set at all, while
`0` is often used as a valid "no value" or "use the default behavior"
sentinel.

## 18.2 Naming and Description Properties

These properties control how objects are named, printed, and described
to the player.

### 18.2.1 `name`

The `name` property holds a list of dictionary words that the parser
recognizes as referring to this object. The `name` property is
**additive**: when a class provides name words, instances inherit them
and can add more.

```inform6
Object  brass_lamp "brass lamp" cave
  with  name 'brass' 'lamp' 'lantern',
        description "A well-polished brass lamp.";
```

The player can refer to this object as "brass lamp", "lantern", or any
combination of the listed words. The `name` property is defined by the
compiler itself rather than by `linklpa.h`, but it is central to
library operation. See Chapter 7 for the object-definition syntax.

### 18.2.2 `short_name`

A string or routine used to print the object's name in inventory lists,
room descriptions, and parser messages. Default: `0`.

When `short_name` is `0`, the compiler uses the name given in the object
header. When it is a string, that string is printed instead. When it is
a routine, the routine prints the name and returns `true`, or returns
`false` to fall back to the default.

```inform6
Object  potion "potion" lab
  with  name 'potion' 'bottle',
        short_name [;
            if (self has general) print "empty bottle";
            else                 print "potion";
            rtrue;
        ];
```

### 18.2.3 `short_name_indef`

A string or routine used when printing the object's name in indefinite
contexts (e.g., "You can see a ... here"). Default: `0`. When `0`, the
library uses `short_name` for both contexts.

```inform6
Object  stranger "stranger" market
  with  name 'stranger' 'man' 'marcus',
        short_name "Marcus",
        short_name_indef "tall stranger",
  has   proper male;
```

### 18.2.4 `description`

A string or routine printed when the player examines the object (the
`Examine` action). For rooms, `description` is printed as part of the
`Look` action's room description.

```inform6
Object  painting "oil painting" gallery
  with  name 'oil' 'painting',
        description [;
            print "A large oil painting of a stormy sea.";
            if (magnifying_glass in player)
                " In the corner you notice a tiny signature.";
            "";
        ];
```

### 18.2.5 `article`

The article printed before the object's name in indefinite contexts
("a", "an", "some"). Default: `"a"`.

```inform6
Object  umbrella "umbrella" hallway
  with  name 'umbrella',
        article "an",
        description "A large black umbrella.";

Object  gravel "gravel" path
  with  name 'gravel',
        article "some",
        description "Loose grey gravel.",
  has   scenery;
```

Objects with the `proper` attribute skip the article entirely.

### 18.2.6 `articles`

An array property for language-specific article forms. Used by
non-English language modules that require multiple article forms (e.g.,
gendered or case-inflected articles). The format is language-dependent;
see the language definition file for details. Most English-language games
do not use this property.

### 18.2.7 `plural`

A dictionary word or routine for the plural form of the object's name.
Used by the library when listing multiple objects of the same kind.

```inform6
Class   Coin
  with  name 'coin',
        plural 'coins',
        short_name "coin",
        description "A small coin.";
```

When several `Coin` objects appear together, the library can print
"three coins" instead of listing each one separately.

### 18.2.8 `initial`

A string or routine printed in room descriptions when the object is in
its **initial state** — before the player has picked it up (before the
`moved` attribute is set). For doors and containers, the library uses
`when_open`/`when_closed` instead when the object is `openable`.

```inform6
Object  letter "sealed letter" desk
  with  name 'sealed' 'letter',
        initial "A sealed letter lies prominently on the desk.",
        description "A letter sealed with red wax.";
```

### 18.2.9 `when_open` and `when_closed`

Strings or routines printed in room descriptions for containers and
doors, depending on whether the object currently has the `open`
attribute.

```inform6
Object  wardrobe "oak wardrobe" bedroom
  with  name 'oak' 'wardrobe',
        when_open   "An oak wardrobe stands here, its doors flung wide.",
        when_closed "An oak wardrobe stands solidly against the wall.",
        description "A handsome oak wardrobe.",
        capacity 10,
  has   container openable;
```

These properties take priority over `initial` for openable objects.

### 18.2.10 `when_on` and `when_off`

Strings or routines printed in room descriptions for switchable objects,
depending on whether the object currently has the `on` attribute.

```inform6
Object  radio "transistor radio" lounge
  with  name 'transistor' 'radio',
        when_on  "A transistor radio is playing softly.",
        when_off "A transistor radio sits silent on the shelf.",
        description "A battered transistor radio.",
  has   switchable;
```

These properties take priority over `initial` for switchable objects.

### 18.2.11 `describe`

A routine called by the library when it wants to include this object in
a room description. **Additive**, default `NULL`.

If `describe` returns `true`, the library considers the object described
and does not print any default text. If it returns `false`, the library
falls back to `initial`, `when_open`/`when_closed`, or the standard
listing.

```inform6
Object  cat "tabby cat" garden
  with  name 'tabby' 'cat',
        describe [;
            if (self in kitchen) "A tabby cat sits on the counter.";
            if (self in garden)  "A tabby cat lounges in a sunny patch.";
            rfalse;
        ],
  has   animate;
```

### 18.2.12 `inside_description`

A string or routine printed when the player is **inside** this object
(an `enterable` container or supporter).

```inform6
Object  wardrobe "wardrobe" bedroom
  with  name 'wardrobe',
        inside_description
            "It's dark and cramped, surrounded by coats.",
        capacity 5,
  has   container enterable openable open;
```

### 18.2.13 `invent`

A routine called when the object is listed in the player's inventory.
It is called twice: first with `inventory_stage` set to `1` (printing
the name), then with `inventory_stage` set to `2` (printing additional
details). Returning `true` at either stage suppresses the default.

```inform6
Object  sword "magic sword" armoury
  with  name 'magic' 'sword',
        invent [;
            if (inventory_stage == 2) {
                if (self has light) print " (glowing brightly)";
                rtrue;
            }
        ];
```

### 18.2.14 `list_together`

Groups objects together in inventory and room-description listings. The
value can be:

- **A string** — objects with the same `list_together` string are
  grouped and the string is used as a heading.
- **A routine** — called to print the grouped listing. Like `invent`,
  it is called with `inventory_stage` set to `1` (before) and `2`
  (after).
- **A number** — objects with the same number are grouped together.

```inform6
Class   Coin
  with  name 'coin',
        list_together "coins",
        short_name "coin",
        description "A small coin.";

Object  -> gold_coin "gold coin"
  class Coin
  with  name 'gold',
        short_name "gold coin";

Object  -> silver_coin "silver coin"
  class Coin
  with  name 'silver',
        short_name "silver coin";
```

When both coins are visible, the library prints "two coins (a gold coin
and a silver coin)" rather than listing them separately.

## 18.3 Action-Handling Properties

These properties intercept actions before and after the library processes
them. They form the core of Inform 6's event-handling system. The full
details of action processing are covered in Chapter 20; this section
describes the properties themselves.

### 18.3.1 `before`

Called before the library processes an action on this object (when this
object is `noun` or `second`). **Additive**, default `NULL`.

The `before` routine receives the action in the `action` variable and
can test it with action-name constants:

```inform6
Object  vase "porcelain vase" parlour
  with  name 'porcelain' 'vase',
        before [;
          Take:
            if (self has general) "It's stuck to the shelf with glue!";
          Attack:
            remove self;
            "The vase shatters into a thousand pieces!";
        ],
        description "A delicate porcelain vase.";
```

If `before` returns `true`, the action is **stopped** — the library
does not execute its default behavior. If it returns `false`, processing
continues normally.

Because `before` is additive, a class hierarchy can layer multiple
handlers. An instance inherits all `before` routines from its class
chain and can add its own.

### 18.3.2 `after`

Called after the library has successfully processed an action on this
object, but before the default success message is printed. **Additive**,
default `NULL`.

If `after` returns `true`, the library's default success message is
suppressed (the routine should print its own message). If it returns
`false`, the default message is printed.

```inform6
Object  lever "rusty lever" engine_room
  with  name 'rusty' 'lever',
        after [;
          Pull:
            StartDaemon(steam_engine);
            "You heave the lever. Machinery rumbles to life!";
          Push:
            StopDaemon(steam_engine);
            "You push the lever back. The machinery falls silent.";
        ],
        description "A rusty lever set into the wall.",
  has   static;
```

### 18.3.3 `life`

Called for actions directed **at** an animate object — actions such as
`Attack`, `Kiss`, `Give`, `Show`, `Ask`, `Tell`, and `Answer`.
**Additive**, default `NULL`.

The `life` property is only consulted when the object has the `animate`
attribute (or `talkable` for conversation actions). See Section 17.10.3
for examples.

```inform6
Object  parrot "parrot" jungle
  with  name 'parrot' 'bird',
        life [;
          Attack: "The parrot squawks and flutters out of reach!";
          Ask:    "~Pieces of eight!~ squawks the parrot.";
        ],
  has   animate;
```

### 18.3.4 `react_before`

Called when **any** action occurs, provided this object is in scope
(even if it is not `noun` or `second`). If it returns `true`, the
action is stopped.

```inform6
Object  guard_dog "guard dog" yard
  with  name 'guard' 'dog',
        react_before [;
          Take:
            if (noun in yard && noun ~= self)
                "The guard dog growls as you reach for ",
                (the) noun, ".";
        ],
  has   animate;
```

### 18.3.5 `react_after`

Called after any action has been successfully processed, provided this
object is in scope. Analogous to `react_before` but fires during the
after phase.

### 18.3.6 `orders`

Handles commands given **to** this object as an NPC (for example,
"robot, take the spanner"). **Additive**.

The `orders` property receives the player's command with `action` set to
the requested action and `noun`/`second` set appropriately. If it
returns `true`, the command is considered handled. See Section 17.10.4
for a full example.

```inform6
Object  butler "butler" dining_room
  with  name 'butler',
        orders [;
          Give:   "~Very good, sir.~ The butler accepts ", (the) noun, ".";
          default: "~I'm afraid I can't do that, sir.~";
        ],
        description "An impeccably dressed butler.",
  has   animate male proper;
```

## 18.4 Direction Properties

These properties define the connections between rooms. They are described
in Section 17.2.1; this section provides a summary.

### 18.4.1 The Twelve Compass Directions

| Property  | Direction   | Property  | Direction   |
|-----------|-------------|-----------|-------------|
| `n_to`    | North       | `s_to`    | South       |
| `e_to`    | East        | `w_to`    | West        |
| `ne_to`   | Northeast   | `nw_to`   | Northwest   |
| `se_to`   | Southeast   | `sw_to`   | Southwest   |
| `u_to`    | Up          | `d_to`    | Down        |
| `in_to`   | In          | `out_to`  | Out         |

Each direction property can hold:

- **A room object** — the player moves to that room.
- **A door object** — the library consults the door's `door_to` property.
- **A routine** — called at travel time; returns a destination object,
  `true` (after printing a refusal), or `false` (use the default "can't
  go" message).
- **A string** — printed as a refusal message; travel is blocked.
- **`0` (false)** — no exit in that direction.

### 18.4.2 `door_to`

The room that a door leads to from the player's current location.
Usually a routine that checks `location`:

```inform6
door_to [;
    if (location == hallway) return study;
    return hallway;
],
```

See Section 17.6 for complete door examples.

### 18.4.3 `door_dir`

The direction property that a door occupies. Like `door_to`, this is
usually a routine when the door connects two rooms:

```inform6
door_dir [;
    if (location == hallway) return n_to;
    return s_to;
],
```

### 18.4.4 `cant_go`

A string or routine printed when the player tries to move in a direction
with no exit. Overrides the library's default "You can't go that way"
message. See Section 17.2.2.

```inform6
Object  island "Desert Island"
  with  description "A tiny desert island.",
        cant_go "The sea stretches endlessly in every direction.",
        n_to  beach,
  has   light;
```

## 18.5 Container and Capacity Properties

### 18.5.1 `capacity`

The maximum number of objects that a container, supporter, or the player
object can hold. Default: `100`.

For the player object, `capacity` limits the number of items that can be
carried simultaneously. It is initialized to `MAX_CARRIED` (default
100). For containers and supporters, it limits the number of child
objects.

```inform6
Object  toolbox "toolbox" workshop
  with  name 'toolbox' 'tool' 'box',
        description "A metal toolbox.",
        capacity 8,
  has   container openable open;
```

### 18.5.2 `with_key`

The key object required to lock or unlock a `lockable` container or
door. The `Lock` and `Unlock` actions check whether the player is
holding the correct key.

```inform6
Object  iron_chest "iron chest" cellar
  with  name 'iron' 'chest',
        description "A heavy iron chest with a complex lock.",
        with_key skeleton_key,
        capacity 10,
  has   container openable lockable locked;
```

If `with_key` is `nothing` (or not provided), the `Unlock` action will
accept any object as a key.

## 18.6 Scope and Parsing Properties

These properties affect how the parser identifies objects and what
objects are available for the player to refer to.

### 18.6.1 `parse_name`

A routine that provides custom parsing for the object's name. Default:
`0`.

When the parser is trying to match player input to an object, it calls
`parse_name` (if provided) instead of the default dictionary-word
matching. The routine examines the word stream using `NextWord()` and
returns the number of words matched, or `0` for no match, or `-1` to
fall back to the default name-matching algorithm.

```inform6
Object  numbered_locker "locker" changing_room
  with  parse_name [  w n;
            w = NextWord();
            if (w ~= 'locker') return 0;
            w = NextWord();
            n = TryNumber(wn - 1);
            if (n == self.number) return 2;
            return 1;
        ],
        number 0,
        description [; print "Locker number ", self.number, "."; ];
```

`parse_name` is also used for disambiguation. If it returns a value, the
parser uses that value to rank competing matches — a higher word count
means a better match.

### 18.6.2 `add_to_scope`

Adds related objects to scope when this object is in scope. The value
can be:

- **An object** — that object is added to scope.
- **A routine** — called with `PlaceInScope(obj)` to add objects
  programmatically.

```inform6
Object  key_ring "key ring" player
  with  name 'key' 'ring' 'keyring',
        add_to_scope brass_key iron_key,
        description "A ring holding several keys.";
```

When the player holds the key ring, `brass_key` and `iron_key` become
individually referable. With a routine form, call `PlaceInScope(obj)`
to add objects programmatically.

### 18.6.3 `grammar`

Provides custom grammar lines for an `animate` or `talkable` object.
When the player gives a command to this NPC, the library checks
`grammar` before the standard grammar tables. The routine can return
`true` to indicate the command has been parsed, or `false` to fall
through to standard grammar.

```inform6
Object  computer "computer terminal" lab
  with  name 'computer' 'terminal',
        grammar [;
            if (verb_word == 'run' or 'execute') {
                action = ##SwitchOn;
                rtrue;
            }
        ],
  has   talkable switchable;
```

### 18.6.4 `found_in`

A list of rooms, a routine, or a class that specifies where a "floating"
object appears. The library moves the object to the appropriate room
automatically during `MoveFloatingObjects`.

```inform6
Object  sky "sky"
  with  name 'sky',
        description "The sky stretches endlessly above.",
        found_in outdoor_room1 outdoor_room2 outdoor_room3,
  has   scenery;
```

When `found_in` is a routine, it returns `true` if the object should be
present in the current room. See Section 17.11 for details on floating
objects and the `absent` attribute.

## 18.7 Timer and Daemon Properties

These properties drive the library's turn-based scheduling system.

### 18.7.1 `daemon`

A routine that is called once per turn while the daemon is active.
Daemons are started and stopped with `StartDaemon(obj)` and
`StopDaemon(obj)`.

```inform6
Object  candle "wax candle" chapel
  with  name 'wax' 'candle',
        daemon [;
            self.time_left = self.time_left - 1;
            if (self.time_left == 0) {
                give self ~light;
                StopDaemon(self);
                if (self in location)
                    "^The candle gutters and goes out.";
            }
            if (self.time_left == 2 && self in location)
                "^The candle flickers weakly.";
        ],
        time_left 10,
        description "A stubby wax candle.",
  has   light;
```

### 18.7.2 `time_out`

A routine called when the object's timer expires (when `time_left`
reaches zero). **Additive**, default `NULL`.

Timers are started with `StartTimer(obj, duration)` and stopped with
`StopTimer(obj)`. Each turn, the library decrements `time_left`; when
it reaches zero, `time_out` is called.

```inform6
Object  bomb "ticking bomb" vault
  with  name 'ticking' 'bomb',
        time_out [;
            deadflag = 1;
            "^The bomb explodes with a deafening roar!";
        ],
        description [;
            print "A bomb with a digital display reading ";
            print self.time_left;
            " seconds.";
        ];
```

### 18.7.3 `time_left`

The number of turns remaining before a timer expires. Set automatically
by `StartTimer(obj, duration)` and decremented each turn by the
library. Can also be read or modified directly by game code.

```inform6
Object  egg_timer "egg timer" kitchen
  with  name 'egg' 'timer',
        time_out [;
            if (self in location) "^Ding! The egg timer goes off.";
        ],
        time_left 0,
        description [;
            if (self.time_left > 0)
                print_ret "The timer shows ", self.time_left, " minutes.";
            "The egg timer is not running.";
        ];

[ BoilEgg;
    StartTimer(egg_timer, 5);
    "You set the egg timer for five minutes.";
];
```

### 18.7.4 `number`

A general-purpose numeric property with no predefined meaning. Default:
`0`. Game code can use `number` to store any integer value — a locker
number, a score, a counter, or an internal ID.

```inform6
Object  locker_14 "locker 14" hallway
  with  name 'locker',
        number 14,
        description [;
            print_ret "Locker number ", self.number, ".";
        ];
```

The library does not inspect or modify `number` itself; it exists purely
for the game author's use.

### 18.7.5 `each_turn`

A routine called at the end of every turn while this object is in scope
(visible to the player). **Additive**, default `NULL`.

Unlike daemons, `each_turn` requires no explicit start/stop — it fires
automatically whenever the object is in scope.

```inform6
Object  grandfather_clock "grandfather clock" hallway
  with  name 'grandfather' 'clock',
        each_turn [;
            if (random(3) == 1) "^The grandfather clock chimes softly.";
        ],
        description "A tall grandfather clock with a swinging pendulum.",
  has   static scenery;
```

Because `each_turn` is additive, multiple routines can be layered
through class inheritance.

## 18.8 Complete Property Reference Table

The following table lists every common property declared in `linklpa.h`,
in the order they are defined.

| Property           | Default  | Additive? | Section | Purpose                                                 |
|--------------------|----------|-----------|---------|---------------------------------------------------------|
| `before`           | `NULL`   | Yes       | 18.3.1  | Intercepts actions before processing                    |
| `after`            | `NULL`   | Yes       | 18.3.2  | Intercepts actions after processing                     |
| `life`             | `NULL`   | Yes       | 18.3.3  | Handles actions on animate objects                      |
| `n_to`             | `0`      | No        | 18.4.1  | North exit                                              |
| `s_to`             | `0`      | No        | 18.4.1  | South exit                                              |
| `e_to`             | `0`      | No        | 18.4.1  | East exit                                               |
| `w_to`             | `0`      | No        | 18.4.1  | West exit                                               |
| `ne_to`            | `0`      | No        | 18.4.1  | Northeast exit                                          |
| `nw_to`            | `0`      | No        | 18.4.1  | Northwest exit                                          |
| `se_to`            | `0`      | No        | 18.4.1  | Southeast exit                                          |
| `sw_to`            | `0`      | No        | 18.4.1  | Southwest exit                                          |
| `u_to`             | `0`      | No        | 18.4.1  | Up exit                                                 |
| `d_to`             | `0`      | No        | 18.4.1  | Down exit                                               |
| `in_to`            | `0`      | No        | 18.4.1  | In exit                                                 |
| `out_to`           | `0`      | No        | 18.4.1  | Out exit                                                |
| `door_to`          | `0`      | No        | 18.4.2  | Destination through a door                              |
| `with_key`         | `0`      | No        | 18.5.2  | Key for locked container/door                           |
| `door_dir`         | `0`      | No        | 18.4.3  | Direction a door faces                                  |
| `invent`           | `0`      | No        | 18.2.13 | Custom inventory listing routine                        |
| `plural`           | `0`      | No        | 18.2.7  | Plural name form                                        |
| `add_to_scope`     | `0`      | No        | 18.6.2  | Adds objects to scope                                   |
| `list_together`    | `0`      | No        | 18.2.14 | Groups objects in listings                              |
| `react_before`     | `0`      | No        | 18.3.4  | Reacts to any action in scope (before)                  |
| `react_after`      | `0`      | No        | 18.3.5  | Reacts to any action in scope (after)                   |
| `grammar`          | `0`      | No        | 18.6.3  | Custom grammar for NPC                                  |
| `orders`           | `0`      | Yes       | 18.3.6  | Handles commands given to NPC                           |
| `initial`          | `0`      | No        | 18.2.8  | Description in initial state                            |
| `when_open`        | `0`      | No        | 18.2.9  | Description when open                                   |
| `when_closed`      | `0`      | No        | 18.2.9  | Description when closed                                 |
| `when_on`          | `0`      | No        | 18.2.10 | Description when on                                     |
| `when_off`         | `0`      | No        | 18.2.10 | Description when off                                    |
| `description`      | `0`      | No        | 18.2.4  | Printed on `Examine`                                    |
| `describe`         | `NULL`   | Yes       | 18.2.11 | Custom room-listing routine                             |
| `article`          | `"a"`    | No        | 18.2.5  | Indefinite article                                      |
| `cant_go`          | `0`      | No        | 18.4.4  | Message when movement blocked                           |
| `found_in`         | `0`      | No        | 18.6.4  | Rooms where floating object appears                     |
| `time_left`        | `0`      | No        | 18.7.3  | Turns remaining on timer                                |
| `number`           | `0`      | No        | 18.7.4  | General-purpose numeric value                           |
| `time_out`         | `NULL`   | Yes       | 18.7.2  | Called when timer expires                               |
| `daemon`           | `0`      | No        | 18.7.1  | Per-turn daemon routine                                 |
| `each_turn`        | `NULL`   | Yes       | 18.7.5  | Called each turn while in scope                         |
| `capacity`         | `100`    | No        | 18.5.1  | Max objects held                                        |
| `short_name`       | `0`      | No        | 18.2.2  | Object name string or routine                           |
| `short_name_indef` | `0`      | No        | 18.2.3  | Object name in indefinite contexts                      |
| `parse_name`       | `0`      | No        | 18.6.1  | Custom parsing routine                                  |
| `articles`         | `0`      | No        | 18.2.6  | Language-specific article forms                         |
| `inside_description`| `0`     | No        | 18.2.12 | Description when player is inside                       |
