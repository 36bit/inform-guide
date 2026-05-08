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

# Chapter 19: Common Attributes

The standard library defines a set of **common attributes** that control
the fundamental characteristics of every object in the game. Attributes
are boolean flags — each object either has or lacks a given attribute.
This chapter provides a complete reference for every library-defined
attribute.

## 19.1 Overview

Attributes are single-bit flags stored on each object. The
language provides two operations for working with them:

- **`give`** — sets or clears an attribute on an object.
- **`has` / `hasnt`** — tests whether an object has or lacks an
  attribute.

```inform6
give lamp light;          ! Set the 'light' attribute
give lamp ~light;         ! Clear the 'light' attribute

if (lamp has light)       ! Test for the attribute
    print "The lamp is lit.";

if (room hasnt visited)   ! Test for absence
    print "You haven't been here before.";
```

The Z-machine supports up to 48 attributes per object; Glulx has no
practical limit. The library defines 31 standard attributes (plus one
conditional attribute), leaving room for game-specific attributes.

Attributes are declared in `linklpa.h` with the `Attribute` directive:

```inform6
Attribute animate;
Attribute open;
Attribute container;
```

The library, the parser, and the action routines in `verblib.h` all
inspect these attributes to determine how objects behave. Setting the
right combination of attributes is how the game author tells the library
what kind of thing an object is — a container, a door, a wearable item,
a light source, and so on.

## 19.2 Physical State Attributes

These attributes describe the physical state of objects — whether they
are open, locked, switched on, or being worn.

### 19.2.1 `open`

Indicates that a container or door is currently open. The library
manages `open` through the `Open` and `Close` actions for `openable`
objects. Game code can also set it directly.

```inform6
Object  trapdoor "trapdoor" cellar
  with  name 'trapdoor' 'trap' 'door',
        description "A heavy wooden trapdoor in the floor.",
  has   door openable;    ! Starts closed (no 'open')
```

### 19.2.2 `openable`

Indicates that the object can be opened and closed by the player.
Without `openable`, attempts to open the object are refused.

### 19.2.3 `locked`

Indicates that a container or door is currently locked. A locked object
cannot be opened until it is unlocked with the correct key (specified by
`with_key`).

### 19.2.4 `lockable`

Indicates that the object can be locked and unlocked. Without
`lockable`, the `Lock` and `Unlock` actions are refused.

### 19.2.5 `on`

Indicates that a switchable device is currently on. Managed by the
`SwitchOn` and `SwitchOff` actions.

### 19.2.6 `switchable`

Indicates that the object can be switched on and off. The library uses
`when_on` and `when_off` properties (Section 18.2.10) to describe
switchable objects in room descriptions.

```inform6
Object  ceiling_fan "ceiling fan" porch
  with  name 'ceiling' 'fan',
        description [;
            if (self has on) "The fan whirs lazily overhead.";
            "A still ceiling fan hangs overhead.";
        ],
  has   switchable scenery;    ! Starts off
```

### 19.2.7 `worn`

Indicates that a piece of clothing is currently being worn. Set by the
`Wear` action, cleared by `Disrobe`. When the player tries to drop,
give, put, insert, throw, push, or pull a `worn` item, the library
performs an **implicit `Disrobe`** first (via `ImplicitDisrobe`) and
then continues with the requested action; the player is not required
to take the item off explicitly. See §17.7.

### 19.2.8 `clothing`

Indicates that the object can be worn by the player. The `clothing`
attribute is the capability; `worn` is the current state.

```inform6
Object  cloak "dark cloak" cloakroom
  with  name 'dark' 'cloak',
        description "A flowing dark cloak.",
  has   clothing;
```

## 19.3 Object Type Attributes

These attributes define what kind of thing an object is — its
fundamental identity in the world model.

### 19.3.1 `container`

Marks an object as a container — other objects can be placed **inside**
it using the `Insert` action ("put X in Y"). The contents of a container
are visible if the container is `open` or `transparent`.

An object cannot be both `container` and `supporter`.

```inform6
Object  chest "wooden chest" attic
  with  name 'wooden' 'chest',
        description "A battered wooden chest.",
        capacity 10,
  has   container openable;
```

### 19.3.2 `supporter`

Marks an object as a supporter — other objects can be placed **on top**
of it using the `PutOn` action ("put X on Y"). The contents of a
supporter are always visible.

```inform6
Object  table "oak table" dining_room
  with  name 'oak' 'table',
        description "A sturdy oak table.",
        capacity 8,
  has   supporter;
```

### 19.3.3 `door`

Marks an object as a door connecting two rooms. A door must also have
`door_to`, `door_dir`, and `found_in` properties. See Section 17.6.

### 19.3.4 `enterable`

Indicates that the player can enter or get on this object. For a
`container`, the player gets **inside**; for a `supporter`, the player
gets **on** it.

```inform6
Object  bathtub "bathtub" bathroom
  with  name 'bathtub' 'bath' 'tub',
        inside_description "You sit in the bathtub. How relaxing.",
        capacity 5,
  has   container enterable open;
```

### 19.3.5 `edible`

Indicates that the object can be eaten. The `Eat` action removes the
object from play.

```inform6
Object  mushroom "red mushroom" forest
  with  name 'red' 'mushroom',
        before [;
          Eat: deadflag = 1;
               "You eat the mushroom. The world spins and fades...";
        ],
  has   edible;
```

### 19.3.6 `animate`

Marks an object as a living being (NPC). An animate object cannot be
taken, can receive commands ("guard, go north"), and triggers the
`life` property for personal actions. At least one gender attribute
should be set.

```inform6
Object  wizard "old wizard" tower
  with  name 'old' 'wizard',
        life [;
          Ask: "The wizard strokes his beard thoughtfully.";
        ],
  has   animate male;
```

See Section 17.10 for the full NPC system.

### 19.3.7 `talkable`

Allows the player to address the object in conversation even though it
is not `animate`. Used for telephones, computers, or magical items.
An object with `talkable` can receive `Ask`, `Tell`, and `Answer`
actions and can have an `orders` property.

## 19.4 Visibility and Listing Attributes

These attributes control whether and how objects appear in room
descriptions, and whether they provide light.

### 19.4.1 `scenery`

Marks an object as part of the room scenery. A scenery object:

- Is **not listed** separately by `Look` — it should be mentioned in
  the room's `description` text instead.
- **Cannot be taken** — attempts produce a refusal message.

```inform6
Object  fireplace "stone fireplace" parlour
  with  name 'stone' 'fireplace' 'hearth',
        description "A large stone fireplace, cold and dark.",
  has   scenery;
```

Scenery objects are still in scope and can be examined or otherwise
interacted with; they are simply not listed automatically.

### 19.4.2 `static`

Prevents the player from taking the object but does not suppress it
from room listings. Use `static` for heavy, fixed, or immovable objects
that should still appear when the library lists the room's contents.

```inform6
Object  cannon "brass cannon" battlement
  with  name 'brass' 'cannon',
        description "A brass cannon pointed out to sea.",
  has   static;
```

The difference from `scenery`: a `static` object **is** listed by
`Look`; a `scenery` object is not.

### 19.4.3 `concealed`

Suppresses the object from room listings but allows the player to refer
to it by name. Often combined with `scenery`. Game code can reveal the
object later by clearing the attribute:

```inform6
Object  secret_button "tiny button" library
  with  name 'tiny' 'button',
        before [;
          Push: "You hear a grinding noise behind the bookshelf.";
        ],
  has   concealed scenery;

! Later:
! give secret_button ~concealed;
! give secret_button ~scenery;
```

### 19.4.4 `transparent`

Indicates that the contents of a closed container are visible to the
player. Without `transparent`, a closed container hides its contents
from view (and from scope — the player cannot refer to them).

```inform6
Object  glass_case "glass display case" museum
  with  name 'glass' 'display' 'case',
        description "A glass display case with a brass lock.",
        with_key museum_key,
        capacity 5,
  has   container openable lockable locked transparent;
```

The diamond inside the locked glass case can be seen (and referred to)
but not taken until the case is unlocked and opened.

### 19.4.5 `light`

Indicates that the object provides illumination. The library's
`OffersLight` routine checks for `light` to determine whether the player
can see.

- A **room** with `light` is always illuminated.
- A **portable object** with `light` illuminates whatever room it is in.
- Light propagates through the containment tree: a lit torch inside an
  open box still lights the room.

```inform6
Object  cave "Dark Cave"
  with  description "A damp cave with moss-covered walls.",
  has   ;    ! No 'light' — this room is dark by default

Object  torch "flaming torch" cave
  with  name 'flaming' 'torch',
        description "A torch burning with a steady flame.",
  has   light;
```

See Section 17.4 for the complete light model.

## 19.5 Gender and Number Attributes

These attributes control the pronouns and articles the library uses
when referring to objects.

### 19.5.1 `male`

Causes the library to use masculine pronouns (he/him/his).

### 19.5.2 `female`

Causes the library to use feminine pronouns (she/her/hers).

### 19.5.3 `neuter`

Causes the library to use neuter pronouns (it/its). This is the default
for inanimate objects but should be set explicitly on animate objects
that are not male or female.

```inform6
Object  golem "clay golem" workshop
  with  name 'clay' 'golem',
        description "A massive figure of living clay.",
  has   animate neuter;
```

### 19.5.4 `pluralname`

Indicates that the object has a plural name and should use plural
pronouns (they/them/their). The library adjusts verb agreement.

```inform6
Object  scissors "pair of scissors" desk
  with  name 'pair' 'of' 'scissors',
        article "a",
  has   pluralname;
```

### 19.5.5 `proper`

Indicates a proper noun name — printed **without** an article. Without
`proper`, the library prints "a lamp" or "the lamp"; with `proper`, it
prints just "Captain Marsh" or "Excalibur".

```inform6
Object  excalibur "Excalibur" stone
  with  name 'excalibur' 'sword',
        description "The legendary sword, gleaming with inner light.",
  has   proper;
```

## 19.6 State-Tracking Attributes

These attributes track the history and state of objects during gameplay.
They are primarily set by the library itself, though game code may read
or modify them.

### 19.6.1 `moved`

Set by the library when the player picks up an object for the first
time. Once `moved` is set, the `initial` property is no longer used in
room descriptions. Also interacts with scoring: the first time an
object with `scored` is taken, `OBJECT_SCORE` points are awarded.

### 19.6.2 `visited`

Set by the library when the player enters a room for the first time.
Used by `Look` to decide between verbose and brief room descriptions.
Also interacts with scoring: the first time a `scored` room is entered,
`ROOM_SCORE` points are awarded.

### 19.6.3 `scored`

Marks an object or room for automatic scoring:

- **On a room**: `ROOM_SCORE` points (default 5) on first entry.
- **On an object**: `OBJECT_SCORE` points (default 4) when first taken.

```inform6
Object  treasure_room "Treasure Room"
  with  description "Gold glitters from every surface.",
  has   light scored;
```

See Section 17.12.4 for details.

### 19.6.4 `general`

A general-purpose flag with no predefined meaning. The library never
reads or modifies `general`; it exists for the game author's use.

```inform6
Object  lever "lever" control_room
  with  name 'lever',
        before [;
          Pull:
            if (self has general) "Already pulled.";
            give self general;
            "You pull the lever. Gears grind behind the wall.";
          Push:
            if (self hasnt general) "Already in position.";
            give self ~general;
            "You push the lever back.";
        ],
  has   static;
```

For objects needing multiple states, use the `number` property or a
custom property instead.

## 19.7 Internal Attributes

These attributes are used internally by the library. Game code should
rarely need to set or test them directly.

### 19.7.1 `absent`

Suppresses a floating object from appearing in its `found_in` rooms.
When set, `MoveFloatingObjects` skips the object. Has an alias:
`non_floating`.

```inform6
Object  companion "faithful dog"
  with  name 'faithful' 'dog',
        found_in [; rtrue; ],    ! Follows the player everywhere
  has   animate;

! Later: give companion absent;    ! Dog disappears
!        give companion ~absent;   ! Dog returns
```

See Section 17.11 for the floating-object system.

### 19.7.2 `workflag`

A temporary internal flag used by library routines — particularly the
object-listing code — to mark objects during processing. The library
sets and clears `workflag` freely. Game code should **not** rely on it
for persistent state; use `general` instead.

### 19.7.3 `infix__watching`

A conditional attribute that only exists when the compiler's **Infix**
debugging system is enabled. It is used internally by the Infix debugger
to track which objects are being monitored. Game code should never
reference this attribute.

```inform6
! Only defined when compiled with Infix support:
#Ifdef INFIX;
#Ifndef infix__watching;
Attribute infix__watching;
#Endif;
#Endif;
```

## 19.8 Complete Attribute Reference Table

The following table lists every attribute declared in `linklpa.h`, in
declaration order.

| Attribute         | Category     | Set by             | Meaning                                        |
|-------------------|--------------|--------------------|-------------------------------------------------|
| `animate`         | Object type  | Game author        | Object is a living being / NPC                  |
| `absent`          | Internal     | Game author / lib  | Suppresses floating object from appearing       |
| `clothing`        | Physical     | Game author        | Object can be worn                              |
| `concealed`       | Visibility   | Game author        | Object hidden from listings but referable       |
| `container`       | Object type  | Game author        | Object can hold other objects inside             |
| `door`            | Object type  | Game author        | Object is a door between rooms                  |
| `edible`          | Object type  | Game author        | Object can be eaten                             |
| `enterable`       | Object type  | Game author        | Player can enter or get on the object            |
| `general`         | State        | Game author        | General-purpose flag (free for author use)       |
| `light`           | Visibility   | Game author        | Object provides illumination                     |
| `lockable`        | Physical     | Game author        | Object can be locked/unlocked                    |
| `locked`          | Physical     | Game author / lib  | Object is currently locked                       |
| `moved`           | State        | Library            | Object has been picked up by player              |
| `on`              | Physical     | Library            | Switchable device is currently on                |
| `open`            | Physical     | Game author / lib  | Container/door is currently open                 |
| `openable`        | Physical     | Game author        | Object can be opened and closed                  |
| `proper`          | Gender       | Game author        | Proper noun — no article printed                 |
| `scenery`         | Visibility   | Game author        | Part of room description, not listed, not takeable|
| `scored`          | State        | Game author        | Contributes to automatic scoring                 |
| `static`          | Visibility   | Game author        | Cannot be taken                                  |
| `supporter`       | Object type  | Game author        | Objects can be placed on top                     |
| `switchable`      | Physical     | Game author        | Object can be switched on/off                    |
| `talkable`        | Object type  | Game author        | Object can be addressed in conversation          |
| `transparent`     | Visibility   | Game author        | Contents visible when closed                     |
| `visited`         | State        | Library            | Room has been entered by the player              |
| `workflag`        | Internal     | Library            | Temporary flag used by library routines          |
| `worn`            | Physical     | Library            | Clothing is currently being worn                 |
| `male`            | Gender       | Game author        | Masculine pronouns (he/him)                      |
| `female`          | Gender       | Game author        | Feminine pronouns (she/her)                      |
| `neuter`          | Gender       | Game author        | Neuter pronouns (it/its)                         |
| `pluralname`      | Gender       | Game author        | Plural pronouns (they/them)                      |

**Alias:** `non_floating` is an alias for `absent`. Both names refer to
the same underlying attribute.

> **Note:** The `infix__watching` attribute is only defined when the
> game is compiled with Infix debugging support (`#Ifdef INFIX`). It is
> not listed in the table above because it is not available in standard
> compilations.
