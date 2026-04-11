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

# Chapter 22: Scope, Light, and Line of Sight

Scope, light, and touchability are three distinct but related systems
that govern what objects the player (and other actors) can perceive,
refer to, and physically manipulate during play. Scope determines which
objects the parser considers when resolving noun phrases and which
objects receive periodic processing such as `each_turn`,
`react_before`, and `react_after`. Light determines whether a location
is illuminated or plunged into darkness. Touchability determines
whether an actor can physically reach an object, even if that object is
visible. The information in this chapter is derived from `parser.h` and
`verblib.h` in library version 6.12.8.

## §22.1 What Is Scope?

**Scope** is the set of objects that are available for reference at any
given moment during parsing. When the player types a command such as
`take the lamp`, the parser must determine which objects in the game
world could be meant by "the lamp." Only objects that are currently **in
scope** are candidates for matching.

Scope is not the same as physical containment or visibility. An object
can be physically present in the same room as the player yet out of
scope (if scope has been customised to exclude it). Conversely, an
object in a distant location can be brought into scope through
customisation. Scope is a parser-level concept: it defines the universe
of objects that the parser is allowed to consider.

Scope serves two distinct purposes:

1. **Noun resolution.** During parsing, the parser searches scope to
   find objects whose `name` properties (or `parse_name` routines)
   match the words the player typed. Only in-scope objects are
   candidates.

2. **Periodic processing.** After each turn, the library rebuilds scope
   for non-parsing reasons to determine which objects should have their
   `each_turn`, `react_before`, and `react_after` properties called.
   Only objects currently in scope receive this processing.

Scope is rebuilt from scratch every time it is needed. There is no
persistent "scope list" that carries over between turns. Each call to
`SearchScope` constructs the set afresh based on the current state of
the object tree, the actor's location, and any customisations in
effect.

### §22.1.1 Scope vs. Visibility vs. Containment

Consider a gold coin inside a closed glass box (a `transparent
container`). The coin is:

- **In scope** — the parser can match "gold coin" because the box is
  transparent, so `IsSeeThrough` returns true and the coin is added to
  scope.
- **Visible** — the player can see the coin through the glass.
- **Not touchable** — the player cannot physically reach the coin
  because the box is closed. `ObjectIsUntouchable` returns true.

Now consider a magic orb that a wizard can sense remotely via an
`InScope` entry point. The orb is:

- **In scope** — it was explicitly added by `InScope`.
- **Not visible** in the normal sense — it may be in another room
  entirely.
- **Not touchable** — there is no common ancestor in the object tree.

These distinctions are fundamental to understanding the systems
described in this chapter.

## §22.2 The Scope-Building Algorithm

The library builds scope through a chain of routines in `parser.h`:
`SearchScope` is the entry point, `ScopeWithin` iterates over
children, and `ScopeWithin_O` processes individual objects. Together
they traverse the object tree and determine which objects are available.

### §22.2.1 ScopeCeiling

Before scope is searched, the library determines the **scope ceiling**
— the highest point in the object tree from which scope is visible.
`ScopeCeiling` traverses upward from an actor through `transparent`
objects, `supporter`s, and `open container`s until it reaches an
opaque boundary or a room.

```inform6
! ScopeCeiling(person) — returns the topmost visible ancestor
! If person is the player and location is thedark, returns thedark.
! Otherwise walks up through transparent/open/supporter parents.

[ ScopeCeiling person act;
    act = parent(person);
    if (act == 0) return person;
    if (person == player && location == thedark) return thedark;
    while (parent(act) ~= 0
           && (act has transparent || act has supporter
               || (act has container && act has open)))
        act = parent(act);
    return act;
];
```

For example, if the player is on a table (a `supporter`) which is in a
room, `ScopeCeiling` returns the room. If the player is inside a
closed opaque box, `ScopeCeiling` returns the box.

### §22.2.2 SearchScope

`SearchScope` is the primary scope-building routine. It is called by
the parser during noun resolution and by the library during each-turn
processing.

```inform6
SearchScope(domain1, domain2, context)
```

The parameters are:

| Parameter | Meaning |
|-----------|---------|
| `domain1` | Primary search domain (typically `actors_location`) |
| `domain2` | Secondary search domain (typically the `actor` object) |
| `context` | The grammar token type being parsed (e.g. `NOUN_TOKEN`, `HELD_TOKEN`) |

The routine performs the following steps in order:

1. **Debug verbs.** If `DEBUG` is defined and the current verb is a
   debugging verb, all objects in the game are placed in scope.

2. **Custom scope token.** If `scope_token` is set (from a
   `scope=Routine` grammar token — see §22.4), the custom scope
   routine is called with `scope_stage` set to 2. If it returns true,
   scope building is complete.

3. **MULTIINSIDE context.** If the context is `MULTIINSIDE_TOKEN` and
   `advance_warning` is set, only the contents of the advance warning
   object are added (if it is see-through).

4. **InScope entry point.** If the actor is one of the domains, the
   `InScope(actor)` entry point is called (see §22.4). If it returns
   true, standard scope is replaced entirely. The library also checks
   `LibraryExtensions.RunWhile(ext_inscope, false, actor)`.

5. **Domain traversal.** For each domain that is a `supporter` or
   `container`, `ScopeWithin_O` is called on the domain itself, then
   `ScopeWithin` is called to process its children.

6. **Darkness rule.** If `thedark` is one of the domains, the actor
   is always placed in scope (so the player can refer to themselves),
   and if the actor's parent is a `supporter` or `container`, that
   parent is also placed in scope.

### §22.2.3 IsSeeThrough

The helper routine `IsSeeThrough` determines whether an object's
contents should be added to scope when the object itself is in scope:

```inform6
[ IsSeeThrough o;
    if (o has supporter or transparent
        || (o has container && o has open))
        rtrue;
    rfalse;
];
```

An object is see-through if it is a `supporter`, is `transparent`, or
is an `open container`. This means:

- A closed opaque container hides its contents from scope.
- An open container exposes its contents.
- A transparent container (open or closed) exposes its contents.
- A supporter always exposes the objects on top of it.

### §22.2.4 ScopeWithin

`ScopeWithin` iterates over the children of a domain and calls
`ScopeWithin_O` for each:

```inform6
ScopeWithin(domain, nosearch, context)
```

| Parameter | Meaning |
|-----------|---------|
| `domain` | Object whose children to process |
| `nosearch` | Object to skip (prevents circular search) |
| `context` | Grammar token context |

Before iterating children, `ScopeWithin` has a special rule: if the
domain is the `actors_location` and the scope reason is
`PARSING_REASON` and the context is not `CREATURE_TOKEN`, the compass
directions are added to scope by calling `ScopeWithin(Compass)`.

The iteration uses `child`/`sibling` pointers rather than `objectloop`
to avoid issues with objects moving during scope processing (which can
happen during `each_turn` processing).

### §22.2.5 ScopeWithin_O

`ScopeWithin_O` processes a single object for scope inclusion:

```inform6
ScopeWithin_O(domain, nosearch, context)
```

Its behaviour depends on `scope_reason`:

- **Non-parsing reasons** (`EACH_TURN_REASON`, `REACT_BEFORE_REASON`,
  etc.): the routine calls `DoScopeAction(domain)` immediately and
  does not attempt to match words. See §22.3.

- **Parsing reasons** (`PARSING_REASON`, `TALKING_REASON`): the
  routine attempts to match the object against the player's input by
  calling `TryGivenObject(domain)`. It also handles pronoun matching
  ("it", "them") via the `LanguagePronouns` table.

After processing the object itself, `ScopeWithin_O` does two more
things:

1. **Recursive descent.** If the object has children and is
   see-through (`IsSeeThrough` returns true) and is not the
   `nosearch` object, `ScopeWithin` is called recursively on this
   object's contents.

2. **`add_to_scope` processing.** If the object has an
   `add_to_scope` property, it is processed:

   - If the value is a **routine**: `ats_flag` is set to `2 + context`
     and the routine is called via `RunRoutines`. The routine should
     use `AddToScope(obj)` to add objects. After the routine returns,
     `ats_flag` is reset to 0.

   - If the value is a **list of objects**: each object in the list is
     passed to `ScopeWithin_O` for individual processing.

> **[Z-machine]** To determine whether `add_to_scope` contains a
> routine or a list, the library checks whether the first word of the
> property data exceeds `top_object`. If so, it is treated as a
> routine address.

> **[Glulx]** On Glulx, the library checks whether the first byte at
> the property address has the value `$70` (the Glulx function
> signature byte). If not, it is treated as an object list.

## §22.3 Scope Reasons

The global variable `scope_reason` controls the purpose for which scope
is currently being built. Different scope reasons cause different
actions to be taken when an object is found in scope. The constants are
defined in `parser.h`:

| Constant | Value | Purpose |
|----------|-------|---------|
| `PARSING_REASON` | 0 | Normal noun resolution during parsing |
| `TALKING_REASON` | 1 | Resolving nouns in dialogue/conversation context |
| `EACH_TURN_REASON` | 2 | Running `each_turn` properties on in-scope objects |
| `REACT_BEFORE_REASON` | 3 | Running `react_before` properties before the action |
| `REACT_AFTER_REASON` | 4 | Running `react_after` properties after the action |
| `LOOPOVERSCOPE_REASON` | 5 | Iterating over scope via `LoopOverScope` |
| `TESTSCOPE_REASON` | 6 | Testing scope membership via `TestScope` |

### §22.3.1 DoScopeAction

When `scope_reason` is not `PARSING_REASON` or `TALKING_REASON`,
`ScopeWithin_O` does not attempt word matching. Instead, it calls
`DoScopeAction(thing)`, which dispatches based on the current
`scope_reason`:

```inform6
[ DoScopeAction thing s p1;
    s = scope_reason; p1 = parser_one;
    switch (scope_reason) {
        REACT_BEFORE_REASON:
            if (thing.react_before == 0 or NULL) return;
            if (parser_one == 0)
                parser_one = RunRoutines(thing, react_before);
        REACT_AFTER_REASON:
            if (thing.react_after == 0 or NULL) return;
            if (parser_one == 0)
                parser_one = RunRoutines(thing, react_after);
        EACH_TURN_REASON:
            if (thing.#each_turn == 0) return;
            PrintOrRun(thing, each_turn);
        TESTSCOPE_REASON:
            if (thing == parser_one) parser_two = 1;
        LOOPOVERSCOPE_REASON:
            parser_one(thing);
            parser_one = p1;
    }
    scope_reason = s;
];
```

Key details:

- For `REACT_BEFORE_REASON` and `REACT_AFTER_REASON`, the routine
  checks that the object actually has the corresponding property before
  running it. It stores the result in `parser_one`; if `parser_one` is
  already non-zero (a previous object's reaction returned true), no
  further reactions are run. This means a `react_before` or
  `react_after` can halt the chain by returning true.

- For `EACH_TURN_REASON`, the `each_turn` property is executed via
  `PrintOrRun`, which either prints the string or calls the routine.

- For `TESTSCOPE_REASON`, the routine simply compares `thing` against
  `parser_one` (which holds the object being tested). If they match,
  `parser_two` is set to 1.

- For `LOOPOVERSCOPE_REASON`, the routine stored in `parser_one` is
  called with `thing` as the argument. After the call, `parser_one` is
  restored because the called routine may have changed parser globals.

### §22.3.2 Order of Processing

During the each-turn sequence, scope is rebuilt multiple times with
different reasons. The order is:

1. `scope_reason` = `REACT_BEFORE_REASON` — all in-scope objects with
   `react_before` are processed.
2. The action's `before` routine chain runs.
3. The action is executed.
4. `scope_reason` = `REACT_AFTER_REASON` — all in-scope objects with
   `react_after` are processed.
5. `scope_reason` = `EACH_TURN_REASON` — all in-scope objects with
   `each_turn` are processed.

Because scope is rebuilt for each phase, objects that move during an
action may or may not receive `react_after` or `each_turn` processing
depending on their location at the time scope is rebuilt.

## §22.4 Customising Scope

The library provides three mechanisms for customising which objects are
in scope: the `InScope` entry point, the `add_to_scope` property, and
the `scope=Routine` grammar token.

### §22.4.1 The InScope Entry Point

`InScope` is an optional entry point routine that the game can define.
It is called from `SearchScope` whenever the actor is one of the
search domains:

```inform6
[ InScope actor;
    ! Return false to augment standard scope.
    ! Return true to replace standard scope entirely.
];
```

When `InScope` is called, use `PlaceInScope(obj)` to add objects to
scope. If `InScope` returns `false`, the standard scope-building
algorithm continues and the objects added by `InScope` supplement the
normal scope. If `InScope` returns `true`, the standard algorithm is
skipped and only the objects explicitly placed in scope by `InScope`
are available.

**Example: telepathic awareness of a special object**

```inform6
Object crystal_ball "crystal ball" chamber
    with name 'crystal' 'ball' 'orb',
         description "A shimmering sphere of pure crystal.";

[ InScope actor;
    if (actor == player) {
        PlaceInScope(crystal_ball);  ! always in scope
    }
    rfalse;                          ! augment, don't replace
];
```

With this `InScope` routine, the player can always refer to the crystal
ball regardless of where it is in the game world. Because `InScope`
returns `false`, all the normal scope rules still apply in addition.

**Example: replacing scope entirely**

```inform6
[ InScope actor;
    if (actor == player && location == trance_room) {
        PlaceInScope(spirit_wolf);
        PlaceInScope(spirit_eagle);
        PlaceInScope(dream_gate);
        rtrue;   ! replace standard scope: only these three objects
    }
    rfalse;      ! everywhere else, use normal scope
];
```

In the trance room, the player can only refer to the three spirit
objects. Normal room contents, inventory, and compass directions are
all excluded.

`InScope` is called for every scope reason, not just `PARSING_REASON`.
This means objects added by `InScope` also receive `each_turn`,
`react_before`, and `react_after` processing. You can check
`scope_reason` inside `InScope` if you want different behaviour for
different reasons:

```inform6
[ InScope actor;
    if (scope_reason == PARSING_REASON && actor == player) {
        PlaceInScope(radio_transmitter);
    }
    rfalse;
];
```

Here, the radio transmitter is available for noun resolution but does
not receive `each_turn` processing (unless it is in scope for other
reasons).

### §22.4.2 The add_to_scope Property

The `add_to_scope` property on an object causes additional objects to
be brought into scope whenever the owning object is in scope. It can
hold either a list of objects or a routine.

**As a list of objects:**

```inform6
Object toolbelt "toolbelt" 
    with name 'tool' 'belt' 'toolbelt',
         add_to_scope hammer screwdriver wrench,
    has  worn clothing;
```

When the toolbelt is in scope, the hammer, screwdriver, and wrench are
also placed in scope, even if they are not children of the toolbelt in
the object tree. This is useful for modelling objects that are parts of
a larger whole, or accessories that are conceptually attached to
something.

**As a routine:**

When `add_to_scope` is a routine, it is called with `ats_flag` set to
`2 + context` (where `context` is the current grammar token type). The
routine should call `AddToScope(obj)` for each object to add:

```inform6
Object robot "robot" lab
    with name 'robot' 'machine',
         add_to_scope [;
             AddToScope(robot_arm);
             AddToScope(robot_sensor);
             if (robot has general)
                 AddToScope(robot_weapon);
         ];
```

The `AddToScope` routine (as distinct from `PlaceInScope`) is designed
for use inside `add_to_scope` routines:

```inform6
[ AddToScope obj;
    if (ats_flag >= 2)
        ScopeWithin_O(obj, 0, ats_flag - 2);
    if (ats_flag == 1) {
        if (HasLightSource(obj) == 1) ats_hls = 1;
    }
];
```

When `ats_flag >= 2`, the object is processed through `ScopeWithin_O`
with the appropriate context. When `ats_flag == 1`, the routine is
being called from `HasLightSource` to check whether any add-to-scope
objects provide light; in this case, `AddToScope` sets `ats_hls` to 1
if the object is a light source.

**Important:** `AddToScope` should be used inside `add_to_scope`
routines, while `PlaceInScope` should be used inside `InScope` entry
points and `scope=Routine` grammar token routines. Using the wrong one
can produce incorrect results because they handle the scope reason
differently.

### §22.4.3 The scope=Routine Grammar Token

A grammar line can include a `scope=Routine` token to provide a
custom scope for a specific verb. The routine is called at multiple
stages during parsing:

| `scope_stage` | Purpose | Expected behaviour |
|---------------|---------|-------------------|
| 1 | Initialisation | Return `false` (the library handles this stage) |
| 2 | Scope building | Call `PlaceInScope(obj)` for each object; return `true` to suppress standard scope, `false` to augment |
| 3 | Post-processing | Print error messages if needed; return `true` if handled |

**Example: a "locate" verb that can refer to any object in the game**

```inform6
[ LocateScope;
    switch (scope_stage) {
        1: rfalse;
        2: objectloop (x ofclass Object)
               PlaceInScope(x);
           rtrue;
        3: "You can't locate that.";
    }
];

Verb 'locate'
    * scope=LocateScope -> Locate;
```

When the player types `locate the golden key`, the parser uses
`LocateScope` to build scope. Stage 2 places every object in scope,
so the parser can match any object in the game. If no object matches,
stage 3 is called to print an error message.

**Example: a "remember" verb for previously visited rooms**

```inform6
Array visited_rooms --> 64;
Global visited_count = 0;

[ RememberScope i;
    switch (scope_stage) {
        1: rfalse;
        2: for (i = 0 : i < visited_count : i++)
               PlaceInScope(visited_rooms-->i);
           rtrue;
        3: "You don't remember any such place.";
    }
];

Verb 'remember'
    * scope=RememberScope -> Remember;
```

## §22.5 Scope Utility Routines

The library provides two utility routines for querying scope from game
code: `TestScope` and `LoopOverScope`. Both work by temporarily
rebuilding scope with a special scope reason and using `DoScopeAction`
to extract information.

### §22.5.1 TestScope

```inform6
TestScope(obj, actor)
```

Returns 1 if `obj` is currently in scope for `actor`, or 0 if it is
not. If `actor` is 0 or omitted, the player is used.

**Implementation:** `TestScope` saves the current parser state, sets
`scope_reason` to `TESTSCOPE_REASON`, stores `obj` in `parser_one`,
and calls `SearchScope`. During the scope search, `DoScopeAction`
checks each encountered object against `parser_one`. If a match is
found, `parser_two` is set to 1. After the search, all parser globals
are restored and `parser_two` is returned.

```inform6
[ TestScope obj act a al sr x y;
    x = parser_one; y = parser_two;
    parser_one = obj; parser_two = 0;
    a = actor; al = actors_location;
    sr = scope_reason; scope_reason = TESTSCOPE_REASON;
    if (act == 0) actor = player; else actor = act;
    actors_location = ScopeCeiling(actor);
    SearchScope(actors_location, actor, 0);
    scope_reason = sr; actor = a;
    actors_location = al;
    parser_one = x; x = parser_two; parser_two = y;
    return x;
];
```

**Example: conditional description based on scope**

```inform6
Object torch "torch" dungeon
    with name 'torch' 'flaming',
         description [;
             print "A flaming torch mounted on the wall.";
             if (TestScope(secret_door))
                 " Its light reveals a hidden door to the east!";
             "";
         ],
    has  light;
```

**Example: checking scope from a daemon**

```inform6
Object surveillance_system "(surveillance)"
    with daemon [;
             if (TestScope(player, security_guard))
                 print "^[The security guard watches you on a monitor.]^";
         ];
```

### §22.5.2 LoopOverScope

```inform6
LoopOverScope(routine, actor)
```

Calls `routine(obj)` for every object currently in scope for `actor`.
If `actor` is 0 or omitted, the player is used.

**Implementation:** `LoopOverScope` saves parser state, sets
`scope_reason` to `LOOPOVERSCOPE_REASON`, stores the routine address
in `parser_one`, and calls `SearchScope`. During the scope search,
`DoScopeAction` calls `parser_one(thing)` for each object found in
scope. After the search completes, all parser globals are restored.

```inform6
[ LoopOverScope routine act x y a al;
    x = parser_one; y = scope_reason;
    a = actor; al = actors_location;
    parser_one = routine;
    if (act == 0) actor = player; else actor = act;
    actors_location = ScopeCeiling(actor);
    scope_reason = LOOPOVERSCOPE_REASON;
    SearchScope(actors_location, actor, 0);
    parser_one = x; scope_reason = y;
    actor = a; actors_location = al;
];
```

**Example: counting objects in scope**

```inform6
Global scope_count;

[ CountScopeObj obj;
    obj = obj;   ! suppress unused-variable warning
    scope_count++;
];

[ CountInScope act;
    scope_count = 0;
    LoopOverScope(CountScopeObj, act);
    return scope_count;
];
```

**Example: finding the nearest light source in scope**

```inform6
Global found_light;

[ FindLightObj obj;
    if (obj has light && found_light == 0)
        found_light = obj;
];

[ NearestLight;
    found_light = 0;
    LoopOverScope(FindLightObj);
    return found_light;
];
```

**Example: listing all in-scope containers**

```inform6
[ ListContainer obj;
    if (obj has container)
        print (The) obj, " (", (name) obj, ")^";
];

[ ShowContainers;
    print "Containers in scope:^";
    LoopOverScope(ListContainer);
];
```

### §22.5.3 PlaceInScope

```inform6
PlaceInScope(thing)
```

Explicitly adds `thing` to scope. This routine is intended for use
within `InScope` entry points and `scope=Routine` grammar token
routines.

```inform6
[ PlaceInScope thing;
    if (scope_reason ~= PARSING_REASON or TALKING_REASON) {
        DoScopeAction(thing); rtrue;
    }
    wn = match_from; TryGivenObject(thing); placed_in_flag = 1;
];
```

When called during a non-parsing scope reason (such as
`EACH_TURN_REASON` or `TESTSCOPE_REASON`), `PlaceInScope` calls
`DoScopeAction` directly, which dispatches the appropriate behaviour
for that scope reason. When called during parsing, it attempts to
match `thing` against the player's input words by calling
`TryGivenObject`.

## §22.6 Light and Darkness

Light and darkness in the Inform library are determined by the `light`
attribute and a chain of routines that check whether light reaches the
player's location. When light is present, the player can see room
descriptions and object names. When light is absent, the player is in
darkness and the library substitutes a special `thedark` pseudo-
location.

### §22.6.1 The light Attribute

Any object in the game world can be a light source simply by being
given the `light` attribute:

```inform6
Object lantern "brass lantern"
    with name 'brass' 'lantern' 'lamp',
         description "A well-polished brass lantern.",
    has  light;
```

Rooms typically have `light` to indicate that they are inherently
illuminated:

```inform6
Object meadow "Sunny Meadow"
    with description "A wide meadow bathed in sunlight.",
    has  light;
```

A room without `light` is dark unless the player (or something in the
player's environment) carries a light source.

### §22.6.2 OffersLight

`OffersLight(obj)` determines whether an object or its environment
provides light. It is the primary routine called by `AdjustLight` to
determine whether the player's current location is lit.

```inform6
[ OffersLight i j;
    if (i == 0) rfalse;
    if (i has light) rtrue;
    objectloop (j in i)
        if (HasLightSource(j) == 1) rtrue;
    if (i has container) {
        if (i has open || i has transparent)
            return OffersLight(parent(i));
    }
    else {
        if (i has enterable || i has transparent || i has supporter)
            return OffersLight(parent(i));
    }
    rfalse;
];
```

The algorithm:

1. Return `false` if `obj` is 0 (nothing).
2. Return `true` if `obj` itself has the `light` attribute.
3. Check each child of `obj`: if any has a light source (via
   `HasLightSource`), return `true`.
4. If `obj` is a container, check whether light can pass through:
   only if the container is `open` or `transparent`, recurse upward
   to the parent.
5. If `obj` is not a container but is `enterable`, `transparent`, or
   a `supporter`, recurse upward to the parent.
6. Otherwise return `false` — the object blocks light propagation.

### §22.6.3 HasLightSource

`HasLightSource(obj)` performs a more thorough check than
`OffersLight`, including objects brought into scope via
`add_to_scope`:

```inform6
[ HasLightSource i j ad;
    if (i == 0) rfalse;
    if (i has light) rtrue;
    if (i has enterable || IsSeeThrough(i) == 1)
        if (~~(HidesLightSource(i)))
            objectloop (j in i)
                if (HasLightSource(j) == 1) rtrue;
    ad = i.&add_to_scope;
    if (parent(i) ~= 0 && ad ~= 0) {
        if (metaclass(ad-->0) == Routine) {
            ats_hls = 0; ats_flag = 1;
            RunRoutines(i, add_to_scope);
            ats_flag = 0;
            if (ats_hls == 1) rtrue;
        }
        else {
            for (j = 0 : (WORDSIZE * j) < i.#add_to_scope : j++)
                if (HasLightSource(ad-->j) == 1) rtrue;
        }
    }
    rfalse;
];
```

The `add_to_scope` check is significant: if an object has an
`add_to_scope` routine, that routine is called with `ats_flag` set to
1. Inside the routine, calls to `AddToScope` check each added object
for `light` and set `ats_hls` to 1 if found. This means a light
source added via `add_to_scope` can illuminate a room even if it is
not physically present in the object tree beneath the room.

### §22.6.4 HidesLightSource

`HidesLightSource(obj)` is a helper that determines whether an object
blocks light from its contents:

```inform6
[ HidesLightSource obj;
    if (obj == player) rfalse;
    if (obj has transparent or supporter) rfalse;
    if (obj has container) return (obj hasnt open);
    return (obj hasnt enterable);
];
```

The player never hides light (carrying a lamp always illuminates). A
`transparent` object or `supporter` never hides light. A closed
container hides light. A non-enterable, non-container object hides
light from its contents.

### §22.6.5 The thedark Object

When the player's location is dark, the library sets `location` to a
special pseudo-object called `thedark`:

```inform6
Object thedark "(darkness object)"
    with initial 0,
         short_name DARKNESS__TX,
         description [; return L__M(##Miscellany, 17); ];
```

- `short_name` is set to `DARKNESS__TX`, which is the translatable
  string "Darkness" (defined in the language definition file).
- `description` prints library message `Miscellany 17`, which is
  typically "It is pitch dark, and you can't see a thing."

When `location` is `thedark`, the global `real_location` holds the
actual room the player is in. This allows the library to:

- Determine the player's real position for movement purposes.
- Restore `location` when light returns.
- Process `each_turn` for the real room (since scope is built from
  `real_location`).

### §22.6.6 Light-Related Globals

| Global | Purpose |
|--------|---------|
| `lightflag` | `true` if the player currently has light; `false` if in darkness |
| `location` | Current display location; set to `thedark` in darkness |
| `real_location` | Always holds the actual room, even in darkness |
| `visibility_ceiling` | Highest visible object from the player's perspective |

**Example: a switchable light source**

```inform6
Object flashlight "flashlight" player
    with name 'flashlight' 'flash' 'light' 'torch',
         description "A heavy-duty flashlight.",
         before [;
             SwitchOn:
                 give self light;
                 "You switch on the flashlight.";
             SwitchOff:
                 give self ~light;
                 "You switch off the flashlight.";
         ],
    has  switchable;
```

After giving or removing `light`, call `AdjustLight()` to trigger the
light change detection (see §22.7). The library's built-in `SwitchOn`
and `SwitchOff` actions do this automatically, but if you manipulate
`light` directly in custom code, you must call `AdjustLight()`
yourself:

```inform6
[ BreakLanternSub;
    give lantern ~light;
    AdjustLight();
    "The lantern shatters! Darkness engulfs you.";
];
```

## §22.7 Light Change Detection

The library detects transitions between light and darkness using the
`AdjustLight` routine, which is called during the turn processing
sequence.

### §22.7.1 AdjustLight

```inform6
[ AdjustLight flag i;
    i = lightflag;
    lightflag = OffersLight(parent(player));

    if (i == 0 && lightflag == 1) {
        location = real_location;
        if (flag == 0) <Look>;
    }

    if (i == 1 && lightflag == 0) {
        real_location = location; location = thedark;
        if (flag == 0) {
            NoteArrival();
            return L__M(##Miscellany, 9);
        }
    }
    if (i == 0 && lightflag == 0) location = thedark;
];
```

The parameter `flag` controls whether automatic responses are
generated:

| `flag` | Behaviour |
|--------|-----------|
| 0 | Print room description on light appearing; print darkness message on light vanishing |
| non-zero | Suppress automatic messages (silent transition) |

`AdjustLight` handles four cases:

1. **Darkness → Light** (`i == 0`, `lightflag == 1`): The player has
   gained a light source. `location` is restored from
   `real_location`, and a `Look` action is generated to describe the
   newly visible room.

2. **Light → Darkness** (`i == 1`, `lightflag == 0`): The player has
   lost all light. The current `location` is saved to
   `real_location`, then `location` is set to `thedark`.
   `NoteArrival()` is called to handle arrival processing, and
   library message `Miscellany 9` is printed ("It is now pitch dark
   in here!").

3. **Darkness → Darkness** (`i == 0`, `lightflag == 0`): The player
   remains in darkness. `location` is kept as `thedark`.

4. **Light → Light** (no condition matches): No action is needed.

### §22.7.2 The DarkToDark Entry Point

When the player moves from one dark location to another (both lacking
light), the library calls the `DarkToDark` entry point. This is
detected during the `Go` action processing in `verblib.h`:

```inform6
if (OffersLight(location))
    lightflag = true;
else {
    lightflag = false;
    if (k == thedark) {
        if (DarkToDark() == false)
            LibraryExtensions.RunAll(ext_darktodark);
        if (deadflag) rtrue;
    }
    location = thedark;
}
```

The `DarkToDark` entry point is an optional routine the game can
define. It is typically used to warn the player of danger or to inflict
damage:

```inform6
[ DarkToDark;
    if (random(3) == 1) {
        deadflag = 1;
        "Stumbling blindly in the dark, you fall into a crevasse!";
    }
    "You stumble onwards in the darkness, feeling your way...";
];
```

If `DarkToDark` is not defined by the game, the library provides a
stub (via `grammar.h`) that does nothing:

```inform6
Stub DarkToDark 0;
```

If `DarkToDark` returns `false`, `LibraryExtensions.RunAll` is called
to allow library extensions to handle the event. If `deadflag` is set
(the player has died), the movement action returns immediately.

### §22.7.3 When AdjustLight Is Called

`AdjustLight` is called automatically in several situations:

- At the end of each turn during the turn processing sequence.
- During the `Go` action after moving the player.
- During the `SwitchOn` and `SwitchOff` action processing.
- During `Insert` and `Remove` actions when objects move.

If you manipulate the `light` attribute in custom code (giving or
removing it from objects), you should call `AdjustLight()` afterward
to ensure the library detects the change. Failure to do so may leave
`lightflag` and `location` in an inconsistent state until the next
turn.

## §22.8 Reachability and Touchability

Scope determines what objects the player can *refer to*. Touchability
determines what objects the player can *physically reach*. An object
can be in scope (visible, referable) but untouchable (behind a
physical barrier). The library enforces touchability for actions that
require physical contact, such as taking, dropping, or opening.

### §22.8.1 ObjectIsUntouchable

The primary touchability check is performed by
`ObjectIsUntouchable`, defined in `verblib.h`:

```inform6
ObjectIsUntouchable(item, flag1, flag2)
```

| Parameter | Meaning |
|-----------|---------|
| `item` | The object to test |
| `flag1` | If true, suppress error messages |
| `flag2` | If true, apply additional Take/Remove restrictions |
| **Return** | `true` if item is **unreachable**; `false` if it is touchable |

Note the return value convention: `true` means **untouchable** (there
is a barrier). This is the opposite of what you might expect — the
routine name asks "is it untouchable?" and returns a truthy answer if
yes.

### §22.8.2 The Touchability Algorithm

`ObjectIsUntouchable` works in three phases:

**Phase 1: Scope-added objects.** The routine calls
`CommonAncestor(actor, item)` to find the nearest shared parent in the
object tree. If there is no common ancestor (the actor and item are in
completely different branches), the item may have been placed in scope
by an `add_to_scope` property. The routine walks up the item's parent
chain calling `ObjectScopedBySomething` to find what object added it.
If found, it recursively checks whether *that* object is touchable.

```inform6
[ CommonAncestor o1 o2 i j;
    i = o1;
    while (i) {
        j = o2;
        while (j) {
            if (j == i) return i;
            j = parent(j);
        }
        i = parent(i);
    }
    return 0;
];
```

```inform6
[ ObjectScopedBySomething item i j k l m;
    i = item;
    objectloop (j .& add_to_scope) {
        l = j.&add_to_scope;
        k = (j.#add_to_scope) / WORDSIZE;
        if (l-->0 ofclass Routine) continue;
        for (m = 0 : m < k : m++)
            if (l-->m == i) return j;
    }
    rfalse;
];
```

`ObjectScopedBySomething` searches all objects in the game that have
an `add_to_scope` property. For each one with a list (not a routine),
it checks whether `item` is in the list. If found, it returns the
owning object. Note that this routine only checks list-based
`add_to_scope`, not routine-based ones.

**Phase 2: Barriers between actor and common ancestor.** Starting from
the actor's parent, the routine walks up the tree toward the common
ancestor. The only barrier is a closed container — if the actor is
inside a closed container, they cannot reach outside it.

```inform6
if (actor ~= ancestor) {
    i = parent(actor);
    while (i ~= ancestor) {
        if (i has container && i hasnt open) {
            if (flag1) rtrue;
            return L__M(##Take, 9, i, noun);  ! "X is closed."
        }
        i = parent(i);
    }
}
```

**Phase 3: Barriers between item and common ancestor.** Starting from
the item's parent, the routine walks up the tree toward the common
ancestor. Here, more types of barriers apply:

- A closed container blocks access (always).
- When `flag2` is set (Take/Remove restrictions), additional barriers
  include:
  - An `animate` object that is neither a container nor a supporter
    — the item belongs to someone.
  - A `transparent` non-container, non-supporter — visible but not
    reachable (like a display case that is part of scenery).
  - Any other non-container, non-supporter — a generic barrier.

Each barrier type produces a different library message if `flag1` is
not set.

### §22.8.3 Practical Touchability Examples

**A transparent closed container:**

```inform6
Object display_case "glass display case" museum
    with name 'glass' 'display' 'case',
         description "A tall glass display case.",
    has  container transparent openable ~open;

Object diamond "diamond" display_case
    with name 'diamond' 'gem';
```

In this scenario:
- `IsSeeThrough(display_case)` returns `true` (it is transparent).
- The diamond is in scope — the player can `examine diamond`.
- `ObjectIsUntouchable(diamond)` returns `true` — the case is closed.
- The player cannot `take diamond` until the case is opened.

**An object held by an NPC:**

```inform6
Object guard "guard" entrance
    with name 'guard' 'soldier',
    has  animate;

Object keyring "keyring" guard
    with name 'keyring' 'keys' 'key' 'ring';
```

The keyring is in scope (the guard is see-through for scope purposes
because `animate` objects are not containers). With `flag2` set (Take
restrictions), `ObjectIsUntouchable` blocks taking the keyring with
the message "That seems to belong to the guard."

**Checking touchability in custom code:**

```inform6
[ CanReach obj;
    if (ObjectIsUntouchable(obj, true))
        rfalse;   ! silently untouchable
    rtrue;         ! touchable
];

[ PushButtonSub;
    if (ObjectIsUntouchable(noun, true)) {
        "You can't reach ", (the) noun, " from here.";
    }
    "You press ", (the) noun, ".";
];
```

Pass `true` as `flag1` when you want to check touchability without
printing the library's default error messages, then print your own
custom message instead.

### §22.8.4 Scope vs. Touchability Summary

| Situation | In scope? | Touchable? |
|-----------|-----------|------------|
| Object in same room | Yes | Yes |
| Object in open container in room | Yes | Yes |
| Object in closed transparent container | Yes | **No** |
| Object in closed opaque container | **No** | **No** |
| Object on supporter | Yes | Yes |
| Object held by NPC | Yes | No (with `flag2`) |
| Object added by `InScope` | Yes | Depends on tree position |
| Object in different room | **No** (normally) | **No** |

## §22.9 The Compass and Directions in Scope

Compass directions are handled specially by the scope system. They are
always in scope during parsing (but not during `CREATURE_TOKEN`
context) so the player can type directional commands like `go north`
or `examine north wall`.

### §22.9.1 The Compass Object

The library defines a `Compass` object that serves as the parent of
all direction objects:

```inform6
Object Compass "compass" has concealed;
```

The individual direction objects are children of `Compass`:

```inform6
Object n_obj "north wall"  Compass with door_dir n_to, name 'n//' 'north';
Object s_obj "south wall"  Compass with door_dir s_to, name 's//' 'south';
Object e_obj "east wall"   Compass with door_dir e_to, name 'e//' 'east';
Object w_obj "west wall"   Compass with door_dir w_to, name 'w//' 'west';
! ... and so on for ne_obj, nw_obj, se_obj, sw_obj, u_obj, d_obj,
!     in_obj, out_obj
```

Each direction object has:

- A `door_dir` property pointing to the corresponding direction
  property (e.g., `n_to`).
- A `name` property for parser matching.
- `short_name` and `description` for examination.

### §22.9.2 How Directions Enter Scope

Compass directions are brought into scope by a special case in
`ScopeWithin`:

```inform6
if (indef_mode == 0 && domain == actors_location
    && scope_reason == PARSING_REASON
    && context ~= CREATURE_TOKEN)
    ScopeWithin(Compass);
```

This adds all children of the `Compass` object to scope when:

- The parser is not in indefinite mode (`indef_mode == 0`).
- The current domain is the actor's location.
- The scope reason is `PARSING_REASON`.
- The context is not `CREATURE_TOKEN` (so "north" is not matched when
  the parser is looking for a creature).

Because this check requires `domain == actors_location`, compass
directions are only in scope for location-level searches, not for
inventory searches or sub-container searches.

### §22.9.3 Direction Properties and Movement

Each room can define direction properties that determine where
movement leads:

```inform6
Object cave "Dark Cave"
    with description "A damp cave with exits north and east.",
         n_to passage,
         e_to ledge,
    has  ~light;
```

The direction properties can hold:

| Value | Meaning |
|-------|---------|
| An object (room) | Move player to that room |
| An object (door) | Check if door is open; move through if so |
| A routine | Call the routine; if it returns an object, move there |
| A string | Print the string (movement blocked) |
| 0 | "You can't go that way." |

When the player types a direction command, the parser matches the
direction word against the compass objects in scope, resolves the
direction to a `door_dir` property, and the `Go` action reads the
corresponding direction property from the current room to determine
the destination.

### §22.9.4 Removing Directions from Scope

Because compass directions are added via the standard `ScopeWithin`
mechanism, there is no built-in way to remove individual directions
from scope. However, an `InScope` entry point that returns `true`
(replacing standard scope) will exclude compass directions unless they
are explicitly added back:

```inform6
[ InScope actor i;
    if (actor == player && location == void_room) {
        ! In the void: only the player and inventory are in scope.
        ! No compass directions, no room contents.
        PlaceInScope(player);
        objectloop (i in player) PlaceInScope(i);
        rtrue;
    }
    rfalse;
];
```

Alternatively, individual direction objects can be removed from match
lists using a `ChooseObjects` entry point, though this affects
disambiguation scoring rather than scope itself.
