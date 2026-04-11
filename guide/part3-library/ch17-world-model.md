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

# Chapter 17: The World Model

The standard library models the interactive fiction world as a tree of
objects connected by containment relationships. Rooms form the geography,
items populate the landscape, and a set of attributes and properties give
objects their physical characteristics — light, openability, wearability,
and so on. This chapter describes the world model that the library
(version 6.12.8) builds on top of the Inform 6 object system.

## 17.1 The Object Tree

Every object in an Inform 6 program exists in a single global **object
tree**. The tree is rooted at a virtual object 0 (the "root object"),
which is never referenced directly. Each object has exactly one parent
(or no parent, meaning it sits at the top level), and may have children
and siblings.

Four built-in functions navigate the tree:

| Function        | Returns                                       |
|-----------------|-----------------------------------------------|
| `parent(obj)`   | The object that contains `obj`, or `nothing`  |
| `child(obj)`    | The first child of `obj`, or `nothing`        |
| `sibling(obj)`  | The next sibling of `obj`, or `nothing`       |
| `children(obj)` | The number of direct children of `obj`        |

Two statements modify the tree:

```inform6
move lantern to kitchen;    ! Make lantern a child of kitchen
remove lantern;             ! Detach lantern (set its parent to nothing)
```

After `remove`, the object has no parent and is effectively out of play.
It still exists and can be moved back into the tree at any time.

## 17.2 Rooms and the Map

Rooms are the top-level containers of the world model. A room is simply an
object with no parent — it sits at the root level of the object tree.
Everything the player can interact with is ultimately contained within a
room (or within something contained within a room).

### 17.2.1 Direction Properties

Rooms are connected by **direction properties**. The library defines twelve
standard directions:

| Property  | Direction   | Property  | Direction   |
|-----------|-------------|-----------|-------------|
| `n_to`    | North       | `s_to`    | South       |
| `e_to`    | East        | `w_to`    | West        |
| `ne_to`   | Northeast   | `nw_to`   | Northwest   |
| `se_to`   | Southeast   | `sw_to`   | Southwest   |
| `u_to`    | Up          | `d_to`    | Down        |
| `in_to`   | In          | `out_to`  | Out         |

A direction property can hold several types of value:

- **A room object** — the player moves to that room.
- **A door object** — the library consults the door's `door_to` property.
- **A routine** — called at travel time; if it returns an object, the
  player moves there; if it prints a message and returns `true`, travel
  is blocked; if it returns `false` or `nothing`, the default "can't go"
  message is printed.
- **A string** — printed as a refusal message (travel is blocked).
- **`0` (false)** — no exit in that direction.

```inform6
Object  kitchen "Kitchen"
  with  description "A tidy kitchen with a door to the north.",
        n_to  dining_room,
        s_to  "The wall is solid brick.",
        e_to  [;
            if (player in wheelchair) "The doorway is too narrow.";
            return conservatory;
        ],
  has   light;
```

### 17.2.2 Customizing "You can't go that way"

When the player tries to go in a direction with no exit, the library
prints a default message. The `cant_go` property on a room overrides this:

```inform6
Object  cliff_edge "Cliff Edge"
  with  description "A narrow ledge overlooking the sea.",
        cant_go "One wrong step and you'd plummet into the waves.",
        n_to  path,
  has   light;
```

`cant_go` may also be a routine, which is useful for providing
direction-sensitive responses:

```inform6
  cant_go [;
      if (noun == d_obj) "It's a sheer drop!";
      "Dense undergrowth blocks your way.";
  ],
```

## 17.3 The Player Object

The library defines a default player object called `selfobj`:

```inform6
SelfClass selfobj "(self object)";
```

`selfobj` is an instance of the internal `SelfClass`, which provides
defaults for `short_name` (`YOURSELF__TX`, typically "yourself"),
`capacity` (`MAX_CARRIED`, default 100), and `narrative_voice` (2, for
second person). The global variable `player` is set to `selfobj` during
initialization. Game code normally refers to `player`, not `selfobj`.

### 17.3.1 Reassigning the Player

The player variable can be reassigned to a different object, typically via
`ChangePlayer(obj)`. A game that starts with a custom player object
sets it in `Initialise`:

```inform6
Object  captain "Captain Marsh"
  with  name 'captain' 'marsh',
        short_name "Captain Marsh",
        description "A weathered sea captain.",
        capacity MAX_CARRIED,
  has   animate proper transparent;

[ Initialise;
    location = bridge;
    ChangePlayer(captain);
];
```

### 17.3.2 `location` vs `real_location`

The library maintains two global variables for the player's position:

- **`real_location`** — always holds the actual room the player is in.
- **`location`** — normally the same as `real_location`, but set to
  `thedark` when the player is in an unlit room.

Game code that needs the true room should use `real_location`. The
`location` variable is what the library uses for description printing and
scope calculations, so in darkness it reflects the `thedark` pseudo-room.

## 17.4 Light and Darkness

The library determines whether the player can see by calling
`OffersLight`, which recursively checks whether any object in the
player's environment provides illumination.

### 17.4.1 The `light` Attribute

Any object with the `light` attribute provides illumination. A room with
`light` is always lit. A portable object with `light` (such as a torch)
illuminates whatever room it is in — or, more precisely, whatever
containment chain it is part of.

```inform6
Object  torch "brass torch"
  with  name 'brass' 'torch',
        description "A sturdy brass torch.",
  has   light;
```

### 17.4.2 How `OffersLight` Works

`OffersLight` starts from the player's immediate parent (typically the
room) and checks:

1. Does this object have the `light` attribute? If so, return true.
2. Does any child of this object have a light source? (Checked via
   `HasLightSource`, which recurses into see-through containers.)
3. If the object is a closed opaque container, stop — light cannot
   escape. Otherwise, recurse upward to the parent.

The effect is that a lit torch inside an open box on a table in a dark
room will illuminate the room, but a lit torch inside a closed opaque
chest will not.

### 17.4.3 The `thedark` Object

When `OffersLight` returns false, the library sets `location` to the
special `thedark` object:

```inform6
Object  thedark "(darkness object)"
  with  initial 0,
        short_name DARKNESS__TX,
        description [; return L__M(##Miscellany, 17); ];
```

`thedark` acts as a pseudo-room. When the player examines it, the library
prints `DARKNESS__TX` (default: "Darkness") as the room name and the
`##Miscellany` message 17 (default: "It is pitch dark, and you can't see
a thing.") as the description.

### 17.4.4 `AdjustLight`

At the end of every turn, the library calls `AdjustLight` to re-evaluate
lighting:

- If it was dark and is now light, `location` is restored to
  `real_location` and `<Look>` is performed.
- If it was light and is now dark, `location` is set to `thedark` and a
  darkness message is printed.

`AdjustLight` is also called by `PlayerTo` when the player moves to a
new room.

### 17.4.5 The `DarkToDark` Entry Point

When the player moves from one dark room to another dark room, the
library calls the optional `DarkToDark` entry point. This is commonly
used to kill the player for stumbling in the dark:

```inform6
[ DarkToDark;
    deadflag = 1;
    "You stumble in the darkness and fall into a chasm!";
];
```

If `DarkToDark` returns false (or is not defined), the library passes
control to `LibraryExtensions.RunAll(ext_darktodark)`.

## 17.5 Containers and Supporters

Two attributes give objects the ability to hold other objects physically:

- **`container`** — other objects can be placed **inside** (with the
  `Insert` action, "put X in Y").
- **`supporter`** — other objects can be placed **on top** (with the
  `PutOn` action, "put X on Y").

An object may not have both `container` and `supporter` simultaneously.

### 17.5.1 Related Attributes

| Attribute      | Effect                                              |
|----------------|-----------------------------------------------------|
| `open`         | Container is currently open (contents accessible)   |
| `openable`     | Container can be opened and closed by the player    |
| `locked`       | Container is locked shut                            |
| `lockable`     | Container can be locked and unlocked                |
| `transparent`  | Contents visible even when closed                   |
| `enterable`    | Player can enter the container or get on the supporter |

### 17.5.2 The `capacity` Property

The `capacity` property limits how many objects a container or supporter
can hold. The default value is 100. For the player object, `capacity`
limits how many objects can be carried (set to `MAX_CARRIED`, which
defaults to 100).

```inform6
Object  wooden_box "wooden box" kitchen
  with  name 'wooden' 'box',
        description "A sturdy wooden box with a hinged lid.",
        capacity 5,
  has   container openable open;

Object  mantelpiece "mantelpiece" parlour
  with  name 'mantelpiece' 'mantel',
        description "A stone mantelpiece above the fireplace.",
        capacity 3,
  has   supporter scenery;
```

### 17.5.3 Scope and Containment

Containment affects scope — the set of objects the parser considers
available. Objects inside a **closed opaque container** are not in scope:
the player cannot refer to them. If the container is `transparent` or
`open`, the contents are in scope.

## 17.6 Doors

A **door** is an object that connects two rooms. The player interacts with
it as a physical object (opening, closing, locking) and travels through it
as a map connection.

### 17.6.1 Door Attributes and Properties

| Attribute/Property | Purpose                                        |
|--------------------|------------------------------------------------|
| `door` attribute   | Marks the object as a door                     |
| `openable`         | The door can be opened and closed               |
| `open`             | The door is currently open                      |
| `lockable`         | The door can be locked                          |
| `locked`           | The door is currently locked                    |
| `with_key`         | The object required to lock/unlock              |
| `door_to`          | The destination room (see below)                |
| `door_dir`         | The direction property this door occupies       |
| `found_in`         | The two rooms the door appears in               |

### 17.6.2 A Complete Door Example

A two-way door must determine which room the player is going *to*, based
on which room they are coming *from*:

```inform6
Object  oak_door "oak door"
  with  name 'oak' 'door',
        description [;
            print "A heavy oak door";
            if (self has open) ". It stands open.";
            if (self has locked) ". It is locked shut.";
            ". It is closed.";
        ],
        when_open  "An oak door stands open here.",
        when_closed "A heavy oak door bars the way.",
        door_to [;
            if (location == hallway) return study;
            return hallway;
        ],
        door_dir [;
            if (location == hallway) return n_to;
            return s_to;
        ],
        found_in hallway study,
        with_key brass_key,
  has   door openable lockable locked static;
```

The rooms reference the door in their direction properties:

```inform6
Object  hallway "Hallway"
  with  description "A long hallway stretching east to west.",
        n_to oak_door,
  has   light;

Object  study "Study"
  with  description "A book-lined study.",
        s_to oak_door,
  has   light;
```

## 17.7 Clothing

Objects with the `clothing` attribute can be worn by the player.

| Attribute  | Meaning                                |
|------------|----------------------------------------|
| `clothing` | The object can be worn                 |
| `worn`     | The object is currently being worn     |

The `Wear` action puts on a piece of clothing (adding the `worn`
attribute); the `Disrobe` action removes it. An object that is `worn`
cannot be dropped — the player must take it off first.

```inform6
Object  raincoat "yellow raincoat" cloakroom
  with  name 'yellow' 'raincoat' 'coat' 'rain',
        description "A bright yellow raincoat.",
  has   clothing;
```

## 17.8 Edible and Switchable Objects

### 17.8.1 Edible Objects

An object with the `edible` attribute can be eaten by the player. The
`Eat` action removes the object from play:

```inform6
Object  apple "red apple" kitchen
  with  name 'red' 'apple',
        description "A crisp red apple.",
  has   edible;
```

After `>EAT APPLE`, the apple is removed from the object tree. A `before`
rule on the `Eat` action can customize or prevent this.

### 17.8.2 Switchable Objects

An object with the `switchable` attribute has an on/off state controlled
by the `SwitchOn` and `SwitchOff` actions.

| Attribute    | Meaning                                 |
|--------------|-----------------------------------------|
| `switchable` | The object can be switched on and off   |
| `on`         | The object is currently switched on     |

The library automatically uses the `when_on` and `when_off` properties
for room descriptions of switchable objects:

```inform6
Object  desk_lamp "desk lamp" study
  with  name 'desk' 'lamp',
        description [;
            print "A green-shaded desk lamp";
            if (self has on) ". It casts a warm glow.";
            ". It is off.";
        ],
        when_on  "A desk lamp casts a warm glow here.",
        when_off "A desk lamp sits dark on the table.",
  has   switchable;
```

A switchable object can also provide `light` that depends on its state.
Combine `switchable` with a `before` rule:

```inform6
Object  flashlight "flashlight" toolshed
  with  name 'flashlight' 'flash' 'light' 'torch',
        description "A heavy-duty flashlight.",
        before [;
          SwitchOn:  give self light;
          SwitchOff: give self ~light;
        ],
  has   switchable;
```

## 17.9 Scenery and Static Objects

Three attributes control whether objects are listed in room descriptions
and whether they can be picked up.

### 17.9.1 `scenery`

An object with `scenery` is considered part of the room's scenery. It is:

- **Not listed** separately by `Look` (it should be mentioned in the
  room's `description` text instead).
- **Not takeable** — attempts to take it print a refusal message.

```inform6
Object  fountain "marble fountain" courtyard
  with  name 'marble' 'fountain',
        description "Water cascades down tiers of carved marble.",
  has   scenery;
```

### 17.9.2 `static`

An object with `static` cannot be taken, but is otherwise listed normally
by `Look`. Use `static` for heavy or fixed objects that should still
appear in the room's object list:

```inform6
Object  anvil "iron anvil" forge
  with  name 'iron' 'anvil',
        description "A massive iron anvil, far too heavy to move.",
  has   static;
```

### 17.9.3 `concealed`

An object with `concealed` is not listed by `Look` but **can** be
referred to by name. This is useful for hidden objects that the player
might discover through interaction:

```inform6
Object  hidden_switch "small switch" hallway
  with  name 'small' 'switch' 'hidden',
        description "A small switch, barely visible.",
        before [;
          Push: give oak_door ~locked;
                "Click! Something unlocks nearby.";
        ],
  has   concealed scenery;
```

## 17.10 Animate Objects and NPCs

### 17.10.1 The `animate` Attribute

The `animate` attribute marks an object as a living being. Animate objects
differ from inanimate ones in several ways:

- The parser accepts commands directed at them ("guard, drop the sword").
- Actions like `Take` are refused ("I don't suppose the guard would care
  for that.").
- The `life` property intercepts actions performed on the NPC.

### 17.10.2 Gender Attributes

Four attributes specify how the library refers to animate objects:

| Attribute    | Pronoun  | Example         |
|--------------|----------|-----------------|
| `male`       | he/him   | a king          |
| `female`     | she/her  | a queen         |
| `neuter`     | it       | a robot         |
| `pluralname` | they     | the guards      |

At least one gender attribute should be set on every animate object.

### 17.10.3 The `life` Property

The `life` property handles actions directed **at** the NPC (such as
attacking, kissing, or giving). It receives the action in the `action`
variable:

```inform6
Object  guard "burly guard" gatehouse
  with  name 'burly' 'guard',
        description "A burly guard in chain mail.",
        life [;
          Attack: "The guard parries your blow effortlessly.";
          Kiss:   "The guard recoils in horror.";
          Give:
            if (noun == gold_coin) {
                remove gold_coin;
                "The guard pockets the coin and nods.";
            }
            "~I don't want that,~ the guard growls.";
        ],
  has   animate male;
```

### 17.10.4 The `orders` Property

The `orders` property handles commands given **to** the NPC (such as
"guard, go north" or "guard, take the sword"). It receives `action` as
the requested action:

```inform6
Object  robot "service robot" lab
  with  name 'service' 'robot',
        description "A battered service robot.",
        orders [;
          Go:     "The robot whirs but doesn't move.";
          Take:   move noun to self;
                  print "The robot picks up ", (the) noun, ".^";
                  rtrue;
          Drop:   if (noun notin self) "The robot isn't carrying that.";
                  move noun to parent(self);
                  print "The robot puts down ", (the) noun, ".^";
                  rtrue;
          default: "The robot bleeps uncomprehendingly.";
        ],
  has   animate neuter;
```

### 17.10.5 The `talkable` Attribute

The `talkable` attribute allows the player to address an object in
conversation even if it is not `animate`. This is useful for objects like
telephones, intercoms, or talking statues:

```inform6
Object  intercom "intercom" airlock
  with  name 'intercom',
        description "A wall-mounted intercom.",
        life [;
          Ask: "A voice crackles: ~I can't help you with that.~";
          Tell: "The voice says: ~Understood.~";
        ],
  has   talkable static;
```

### 17.10.6 Conversation Actions

The library provides three actions for NPC conversation:

- **`Ask`** — "ask guard about sword" (`noun` = guard, `second` = topic)
- **`Tell`** — "tell guard about dragon" (`noun` = guard, `second` = topic)
- **`Answer`** — "say yes to guard" (`noun` = topic, `second` = guard)

These are typically handled in the NPC's `life` property.

## 17.11 Found-In (Floating Objects)

Some objects should appear in multiple rooms — the sky, background
sounds, or a companion who follows the player. The `found_in` property
makes an object "float" into the correct room automatically.

### 17.11.1 Using `found_in`

The `found_in` property takes a list of rooms, a class, or a routine:

```inform6
! Appear in specific rooms
Object  sky "sky"
  with  name 'sky',
        description "A clear blue sky stretches overhead.",
        found_in  garden courtyard cliff_edge,
  has   scenery;

! Appear in all outdoor rooms (using a class)
Object  birdsong "birdsong"
  with  name 'birdsong' 'bird' 'song' 'singing',
        description "A pleasant chorus of birdsong.",
        found_in [;
            if (location has light && location provides outdoor
                && location.outdoor == true) rtrue;
            rfalse;
        ],
  has   scenery;
```

When `found_in` is a routine, it should return true if the object should
be present in the current `location`, or false otherwise. When `found_in`
is a list, each entry may be a room object or a class (the object floats
into any room that is an instance of that class).

### 17.11.2 How `MoveFloatingObjects` Works

Each time the player moves, the library calls `MoveFloatingObjects`. This
routine iterates over every object in the game. For each object with a
`found_in` property (that does not have the `absent` attribute):

1. If `found_in` is a routine, call it and check the return value.
2. If `found_in` is a list, check whether `location` matches any entry.
3. If the object should be present, `move` it to `location`.
4. If not, `remove` it from the tree.

### 17.11.3 The `absent` Attribute

Giving a floating object the `absent` attribute (aliased as
`non_floating`) suppresses its floating behaviour — `MoveFloatingObjects`
removes any object with `absent`, regardless of `found_in`:

```inform6
before [;
  Take:
    give sky absent;
    "You reach up and... no, you really can't take the sky.";
];
```

## 17.12 The `general`, `moved`, `visited`, and `scored` Attributes

Several attributes track world state automatically or provide free-use
flags for the game author.

### 17.12.1 `general`

The `general` attribute has no built-in meaning. It is reserved for game
authors as a free-use boolean flag on any object. Each object has its own
independent `general` flag:

```inform6
Object  lever "rusty lever" engine_room
  with  name 'rusty' 'lever',
        description [;
            if (self has general) "The lever is pulled down.";
            "The lever is in the upright position.";
        ],
        before [;
          Pull:
            if (self has general) "It's already pulled down.";
            give self general;
            "You pull the lever down with a grinding screech.";
          Push:
            if (self hasnt general) "It's already upright.";
            give self ~general;
            "You push the lever back into its upright position.";
        ],
  has   static;
```

### 17.12.2 `moved`

The `moved` attribute is set automatically by the library when the player
picks up an object. `NoteObjectAcquisitions` (called at the end of each
turn) gives `moved` to any newly held object. Additionally, when an
object with both `moved` unset and `scored` set first enters the player's
inventory, the library adds `OBJECT_SCORE` to the total score.

The library uses `moved` to select description properties: an object
**without** `moved` is described using its `initial` property; one
**with** `moved` is listed by the standard inventory lister instead:

```inform6
Object  letter "sealed letter" desk
  with  name 'sealed' 'letter',
        initial "A sealed letter rests on the desk.",
        description "A cream envelope with a wax seal.",
  has   ;
```

After the player takes and drops the letter, `Look` lists it as "You can
see a sealed letter here" rather than using the `initial` text.

### 17.12.3 `visited`

The `visited` attribute is given to a room when the player enters it for
the first time. `ScoreArrival` (called from `PlayerTo`) checks:

```inform6
if (location hasnt visited) {
    give location visited;
    if (location has scored) {
        score = score + ROOM_SCORE;
        places_score = places_score + ROOM_SCORE;
    }
}
```

After the first visit, the room is marked `visited` and subsequent
arrivals skip scoring. The `Look` action also uses `visited` to decide
whether to print a verbose or brief room description.

### 17.12.4 `scored`

The `scored` attribute interacts with the library's automatic scoring
system:

- On a **room**: when the player first visits a room with `scored`, the
  library adds `ROOM_SCORE` (default 5) to the total score.
- On an **object**: when the player first picks up an object with
  `scored`, the library adds `OBJECT_SCORE` (default 4) to the total
  score.

```inform6
Object  treasure_room "Treasure Room"
  with  description "Gold glitters from every surface.",
  has   light scored;

Object  diamond "diamond" treasure_room
  with  name 'diamond',
        description "A flawless diamond, catching the light.",
  has   scored;
```

The score constants can be overridden before including the library:

```inform6
Constant ROOM_SCORE   10;
Constant OBJECT_SCORE  5;
```
