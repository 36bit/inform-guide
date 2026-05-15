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

# Chapter 25: Scope, Light, and Visibility

**Scope** determines which objects the player can refer to in commands.
**Light** determines whether the player can see. Together, these two
systems form the visibility model that governs what the player can
perceive and interact with. This chapter provides a complete reference
for both systems: the scope-determination algorithm, visibility
ceilings, light propagation, and the routines that query and
manipulate scope.

## 25.1 The Scope Determination Algorithm

Scope is not a fixed set — it is computed dynamically each time the
parser needs to resolve a noun phrase, and again during end-of-turn
processing for `each_turn`, `react_before`, and `react_after`.
The algorithm answers one question: "Which objects can the current
actor refer to right now?"

### 25.1.1 When Scope Is Computed

Scope is computed in several contexts, each identified by a
**scope reason** constant:

| Constant               | Value | Context                             |
|----------------------- |------ |------------------------------------ |
| `PARSING_REASON`       | 0     | Normal command parsing              |
| `TALKING_REASON`       | 1     | NPC conversation                    |
| `EACH_TURN_REASON`     | 2     | Running `each_turn` properties      |
| `REACT_BEFORE_REASON`  | 3     | Running `react_before` properties   |
| `REACT_AFTER_REASON`   | 4     | Running `react_after` properties    |
| `LOOPOVERSCOPE_REASON` | 5     | `LoopOverScope()` iteration         |
| `TESTSCOPE_REASON`     | 6     | `TestScope()` query                 |

The global variable `scope_reason` holds the current reason. During
parsing, the scope callback (`ScopeWithin_O`) attempts to match each
in-scope object against the player's noun phrase. During non-parsing
contexts, it invokes the appropriate property or callback instead.

### 25.1.2 The Scope Ceiling

Before searching for objects, the library determines the **scope
ceiling** — the highest point in the object tree from which the
search begins. The routine `ScopeCeiling(person)` walks up the tree
from `person`'s location, passing through any object that is
transparent, a supporter, or an open container, and stops when it
reaches an object that blocks visibility:

```inform6
[ ScopeCeiling person act;
    act = parent(person);
    if (act == 0) return person;
    if (person == player && location == thedark) return thedark;
    while (parent(act)~=0 && (act has transparent || act has supporter ||
                             (act has container && act has open)))
        act = parent(act);
    return act;
];
```

For most games, the scope ceiling is the room the player is in. But
if the player is inside a closed, opaque container, the scope ceiling
is that container — the player can only see and interact with objects
inside the container.

The `IsSeeThrough` helper determines whether an object allows scope
to pass through:

```inform6
[ IsSeeThrough o;
    if (o has supporter or transparent ||
       (o has container && o has open))
        rtrue;
    rfalse;
];
```

An object is "see-through" if it is:
- A **supporter** (objects on top are always visible),
- **Transparent** (the `transparent` attribute is set), or
- An **open container** (both `container` and `open` are set).

### 25.1.3 `SearchScope`: The Main Algorithm

The core of the scope system is `SearchScope(domain1, domain2,
context)`. It populates the set of in-scope
objects by searching two domains (typically the player's location and
the player object itself) and respecting any custom scope overrides.

The algorithm proceeds in this order:

**Priority 1 — Custom scope token:** If the parser is matching a
grammar token that specifies a custom scope routine (via the
`scope=RoutineName` token syntax; see §22), that routine is called
first with `scope_stage` set to 2. If it returns `true`, the search
ends.

**Priority 2 — `MULTIINSIDE` special case:** If the grammar context
is `MULTIINSIDE_TOKEN` (for commands like "take all from box"), scope
is restricted to the contents of the specified container.

**Priority 3 — `InScope()` entry point:** If the actor is one of the
search domains, the library calls the game author's `InScope(actor)`
entry point (if defined). If `InScope` returns `true`, the standard
search is skipped entirely — the game has complete control over what
is in scope.

**Priority 4 — Standard search:** The library searches `domain1` and
`domain2` using `ScopeWithin`:

1. If the domain is a container or supporter, its own entry is
   processed by `ScopeWithin_O`.
2. `ScopeWithin` iterates over the domain's children, calling
   `ScopeWithin_O` for each.
3. `ScopeWithin_O` processes a single object:
   - If the scope reason is parsing, it attempts to match the object
     against the current noun phrase.
   - If the scope reason is non-parsing, it invokes the appropriate
     callback (`DoScopeAction`).
   - If the object has an `add_to_scope` property, its contents are
     also brought into scope (see §25.5.3).
   - If the object is "see-through", `ScopeWithin_O` recurses into
     its children.

**Priority 5 — Darkness rule:** Even in darkness, the actor is always
in scope to themselves. If `thedark` is one of the domains, the library
explicitly adds the actor and (if the actor is in a container or on a
supporter) the actor's immediate parent.

**Priority 6 — Compass:** When parsing (not in `indef_mode`, i.e.,
not processing "all"), the compass directions are always in scope in
rooms. `ScopeWithin` adds the `Compass` object's children
(the direction objects) to scope.

### 25.1.4 `ScopeWithin` and `ScopeWithin_O`

`ScopeWithin(domain, nosearch, context)` iterates over the children of
`domain`:

```inform6
[ ScopeWithin domain nosearch context x y;
    if (domain == 0) rtrue;

    if (indef_mode==0 && domain==actors_location
        && scope_reason==PARSING_REASON && context~=CREATURE_TOKEN)
            ScopeWithin(Compass);

    x = child(domain);
    while (x ~= 0) {
        y = sibling(x);
        ScopeWithin_O(x, nosearch, context);
        x = y;
    }
];
```

Note that `ScopeWithin` captures each object's sibling *before*
calling `ScopeWithin_O`, in case `ScopeWithin_O` triggers code that
moves the object in the tree (e.g., via an `each_turn` routine).

`ScopeWithin_O` processes a single object. It handles:

- **Pronoun tracking:** updates `it_obj`, `him_obj`, `them_obj` based
  on the object's attributes.
- **`add_to_scope`:** if the object provides this property, additional
  objects are brought into scope (see §25.5.3).
- **Recursion:** if the object is see-through (`IsSeeThrough`), the
  routine recurses into its children.

### 25.1.5 Debug Mode and Scope

When the game is compiled in debug mode and a debugging verb (such as
`TREE`, `SHOWOBJ`, or `GONEAR`) is being parsed, the scope algorithm
is replaced entirely: *every object in the game* is placed in scope.
This allows debug commands to refer to any object regardless of
location.

## 25.2 Light and Darkness

The library models light as a property that propagates through the
object tree. A location is lit if any light source is reachable
through open or transparent containers from the player's position. In
darkness, the player cannot see objects (though they remain in the
player's inventory scope) and room descriptions are suppressed.

### 25.2.1 The `light` Attribute

An object with the `light` attribute is a light source. This can be a
room (providing ambient illumination), a lamp, a torch, or any other
object:

```inform6
Object  torch "flaming torch" cave
  with  name 'flaming' 'torch',
  has   light;
```

If a room has `light`, it is always lit. If a room does not have
`light`, it is lit only if a light source is reachable from the
player's position through the containment tree.

### 25.2.2 `OffersLight(obj)`

The `OffersLight` routine determines whether a given position in the
object tree is illuminated. Starting from `obj`, it checks:

1. Does `obj` itself have the `light` attribute? If so, return `true`.
2. Does any child of `obj` provide light (via `HasLightSource`)? If
   so, return `true`.
3. Can light reach `obj` from its parent? This depends on the
   containment relationship:
   - If `obj` is a **container**, light passes through only if `obj`
     is `open` or `transparent`.
   - Otherwise (`obj` is not a container), light passes through only
     if `obj` is `enterable`, `transparent`, or a `supporter`.
4. If light can pass through, recurse to `parent(obj)`.

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

The recursion walks *up* the tree toward the room, checking whether
light can reach the player's position from any light source above.

### 25.2.3 `HasLightSource(obj)`

`HasLightSource` checks whether `obj` directly or indirectly provides
light:

1. If `obj` has the `light` attribute, return `true`.
2. If `obj` is enterable or see-through, and does not hide its
   contents (see `HidesLightSource`), recursively check all children.
3. If `obj` has an `add_to_scope` property, check those objects too.

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
            ats_flag = 0; if (ats_hls == 1) rtrue;
        }
        else {
            for (j=0 : (WORDSIZE*j)<i.#add_to_scope : j++)
                if (HasLightSource(ad-->j) == 1) rtrue;
        }
    }
    rfalse;
];
```

### 25.2.4 `HidesLightSource(obj)`

Determines whether `obj` prevents light from escaping to its parent:

```inform6
[ HidesLightSource obj;
    if (obj == player) rfalse;
    if (obj has transparent or supporter) rfalse;
    if (obj has container) return (obj hasnt open);
    return (obj hasnt enterable);
];
```

A closed container hides light. Transparent objects, supporters, and
the player never hide light. A non-container, non-enterable object
(such as a plain `Object`) also hides light from its contents — though
in practice such objects rarely have contents.

### 25.2.5 `AdjustLight()`

The library calls `AdjustLight()` at the end of each turn (step 5 of
the end-of-turn sequence; see §24.6) to detect transitions between
light and darkness:

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

If the lighting state changes, the library transitions between light
and darkness:

- **Light acquired:** `location` is updated from `thedark` to
  `real_location`, and a `Look` is performed to show the room.
- **Light lost:** `location` is set to `thedark`, `NoteArrival()` is
  called to run the `initial` property and `NewRoom()` entry point,
  and a darkness message is printed.

The global `lightflag` records whether the player's current position
is lit (`1`) or dark (`0`). The `location` variable holds `thedark`
when the player is in darkness, while `real_location` always holds
the actual room object.

## 25.3 The `thedark` Object

When the player is in darkness, the library sets `location` to a
special object called `thedark`. This object acts as a proxy location:
the parser uses it as the scope domain, and the `Look` action
displays its description (the standard "It is pitch dark" message).

Key properties of `thedark`:

- `short_name`: prints "Darkness" (or the language-specific
  equivalent).
- `description`: prints "It is pitch dark, and you can't see a thing."

The actual room is still stored in `real_location`. Library routines
that need to operate on the real room (e.g., action processing) use
`real_location` when `location == thedark`.

## 25.4 Querying and Manipulating Scope

The library provides several routines for querying scope from game
code and for customizing what is in scope.

### 25.4.1 `TestScope(obj, actor)`

Tests whether `obj` is currently in scope for `actor`. If `actor` is
`0` or omitted, the player is used.

Returns `1` if `obj` is in scope, `0` if not.

```inform6
[ TestScope obj act a al sr x y;
    x = parser_one; y = parser_two;
    parser_one = obj; parser_two = 0; a = actor; al = actors_location;
    sr = scope_reason; scope_reason = TESTSCOPE_REASON;
    if (act == 0) actor = player; else actor = act;
    actors_location = ScopeCeiling(actor);
    SearchScope(actors_location, actor, 0); scope_reason = sr; actor = a;
    actors_location = al; parser_one = x; x = parser_two; parser_two = y;
    return x;
];
```

The routine works by performing a full scope search with
`TESTSCOPE_REASON`. During this search, `DoScopeAction` checks whether
each scanned object matches `parser_one` (the target) and sets
`parser_two` to `1` if found.

Usage:

```inform6
if (TestScope(golden_key))
    print "You can see the golden key.^";
```

### 25.4.2 `LoopOverScope(routine, actor)`

Calls `routine(obj)` for every object in scope for `actor`. If `actor`
is `0`, the player is used.

```inform6
[ LoopOverScope routine act x y a al;
    x = parser_one; y = scope_reason; a = actor; al = actors_location;
    parser_one = routine;
    if (act == 0) actor = player; else actor = act;
    actors_location = ScopeCeiling(actor);
    scope_reason = LOOPOVERSCOPE_REASON;
    SearchScope(actors_location, actor, 0);
    parser_one = x; scope_reason = y; actor = a; actors_location = al;
];
```

The routine performs a full scope search with `LOOPOVERSCOPE_REASON`.
For each in-scope object, `DoScopeAction` calls `parser_one(obj)` —
that is, the user-supplied routine.

Usage:

```inform6
[ CountLightSources obj;
    if (obj has light) light_count++;
];

[ HowManyLights;
    light_count = 0;
    LoopOverScope(CountLightSources);
    print "There are ", light_count, " light sources in scope.^";
];
```

### 25.4.3 `PlaceInScope(thing)`

Manually adds `thing` to the current scope. This routine is only
meaningful when called from within an `InScope()` entry point, a
custom scope token routine, or an `add_to_scope` property routine.
Outside these contexts, it has no effect.

```inform6
[ PlaceInScope thing;
    if (scope_reason~=PARSING_REASON or TALKING_REASON) {
        DoScopeAction(thing); rtrue;
    }
    wn = match_from; TryGivenObject(thing); placed_in_flag = 1;
];
```

During parsing, `PlaceInScope` attempts to match `thing` against the
current noun phrase. During non-parsing contexts (e.g.,
`EACH_TURN_REASON`), it invokes `DoScopeAction` directly.

### 25.4.4 `ScopeWithin(obj)`

When called from a custom scope routine (see §25.5.2), `ScopeWithin`
adds all the children of `obj` to scope, recursing into see-through
containers. This is useful for bringing the entire contents of an
object into scope.

## 25.5 Custom Scope Routines

The library provides three ways to customize what is in scope: the
`InScope()` entry point, custom scope tokens, and the `add_to_scope`
property.

### 25.5.1 The `InScope()` Entry Point

The game author can define an `InScope(actor)` routine that is called
during every scope determination. It receives the actor whose scope is
being computed (usually `player`). Inside `InScope`, the game can call
`PlaceInScope(obj)` to add objects to scope.

If `InScope` returns `true`, the library's standard scope search is
**skipped entirely** — the game has full control over what is in scope.
If it returns `false`, the library performs the standard search in
addition to any objects placed by `InScope`.

```inform6
[ InScope person;
    if (person == player) {
        PlaceInScope(magic_mirror);   ! Always in scope
        rfalse;                       ! Also do standard scope
    }
];
```

To completely replace the standard scope for a given situation:

```inform6
[ InScope person;
    if (person == player && location == dream_world) {
        PlaceInScope(dream_self);
        PlaceInScope(dream_door);
        PlaceInScope(dream_key);
        rtrue;    ! Skip standard scope entirely
    }
    rfalse;       ! Otherwise, use standard scope
];
```

### 25.5.2 Custom Scope Tokens

Grammar lines can include a `scope=RoutineName` token, which defines a
custom scope for that specific grammar line. The routine is called with
`scope_stage` set to indicate the current phase:

| `scope_stage` | Meaning                                                           |
|-------------- |------------------------------------------------------------------ |
| 1             | Query: return `1` if this token matches multiple objects, else `0`|
| 2             | Search: call `PlaceInScope`/`ScopeWithin` to populate scope       |
| 3             | Error: no objects matched — print an error message                |

During stage 2, the routine should call `PlaceInScope(obj)` for each
object that should be matchable, or `ScopeWithin(domain)` to add all
children of a domain. Returning `true` from stage 2 suppresses the
library's standard scope search for this grammar line.

```inform6
[ MyScopeRoutine;
    switch (scope_stage) {
        1: rfalse;              ! Token matches a single object (not multi)
        2: ScopeWithin(limbo);  ! All objects in limbo are matchable
           rtrue;               ! Don't also do standard scope
        3: "You can't see any such thing in the spirit world.";
    }
];

Verb 'summon'
    * scope=MyScopeRoutine -> Summon;
```

### 25.5.3 The `add_to_scope` Property

An object with an `add_to_scope` property can bring additional objects
into scope whenever it is itself in scope. The property can be either:

- **A list of objects:** those objects are automatically added to scope
  (and checked for light sources).

  ```inform6
  Object  keyring "keyring" player
    with  name 'keyring' 'key' 'ring',
          add_to_scope brass_key iron_key silver_key;
  ```

- **A routine:** the routine is called, and it should call
  `AddToScope(obj)` for each additional object. `AddToScope` is the
  proper helper inside `add_to_scope` routines: it both adds `obj` to
  scope (when called during scope searching) *and* participates in
  light-source detection (when `HasLightSource` is probing the object).
  Using `PlaceInScope` here would skip the light-source check.

  ```inform6
  Object  toolbelt "tool belt" player
    with  name 'tool' 'belt',
          add_to_scope [;
              AddToScope(hammer);
              AddToScope(wrench);
              if (self has open) AddToScope(screwdriver);
          ];
  ```

Objects brought into scope via `add_to_scope` are treated as if they
were physically present — the parser can match them, `each_turn` runs
on them, and they participate in `react_before`/`react_after` checks.
However, they are *not* physically moved in the object tree; their
actual parent remains unchanged.

## 25.6 `FindVisibilityLevels()`

The routine `FindVisibilityLevels()`
determines the **visibility ceiling** for display purposes. It returns
the nesting depth from the player to the ceiling and stores the
ceiling object in the global `visibility_ceiling`.

```inform6
[ FindVisibilityLevels visibility_levels;
    visibility_levels = 1;
    visibility_ceiling = parent(player);
    while ((parent(visibility_ceiling)) &&
           (visibility_ceiling hasnt container ||
            visibility_ceiling has open or transparent)) {
        visibility_ceiling = parent(visibility_ceiling);
        visibility_levels++;
    }
    return visibility_levels;
];
```

This is used primarily by `LookSub` to determine how to format
the room description and by the status line drawing code.

## 25.7 Light and Scope Interaction

Scope and light interact in a specific way:

- **In light**, the standard scope algorithm includes all objects
  reachable through see-through containment from the scope ceiling.

- **In darkness** (`location == thedark`):
  - The actor is always in scope to themselves.
  - The actor's inventory is in scope (by virtue of searching the
    actor as `domain2`).
  - Objects in the room are generally *not* in scope, because the
    scope ceiling is `thedark` rather than the actual room.
  - The actor's immediate container (if any) is in scope.

This means that in darkness, the player can still interact with
carried objects ("drop lamp", "turn on torch") but cannot see or
interact with room contents.

A game can override this behavior through the `InScope()` entry
point — for example, to allow the player to feel around in the dark:

```inform6
[ InScope person;
    if (person == player && location == thedark) {
        ! Allow touching nearby objects even in darkness
        ScopeWithin(real_location);
        rfalse;   ! Also add standard (inventory) scope
    }
];
```

## 25.8 Summary of Routines and Properties

| Routine / Property             | Purpose                                    |
|------------------------------- |------------------------------------------- |
| `TestScope(obj, actor)`        | Is `obj` in scope for `actor`?             |
| `LoopOverScope(routine, actor)`| Call `routine(obj)` for each in-scope obj  |
| `PlaceInScope(thing)`          | Add `thing` to current scope search        |
| `AddToScope(obj)`              | Helper for `add_to_scope` routines         |
| `ScopeWithin(domain)`          | Add all children of `domain` to scope      |
| `ScopeCeiling(person)`         | Return highest see-through ancestor        |
| `IsSeeThrough(obj)`            | Is `obj` transparent/open/supporter?       |
| `OffersLight(obj)`             | Is position `obj` illuminated?             |
| `HasLightSource(obj)`          | Does `obj` provide or contain light?       |
| `HidesLightSource(obj)`        | Does `obj` block light from contents?      |
| `AdjustLight()`                | Recalculate and apply lighting changes     |
| `FindVisibilityLevels()`       | Get nesting depth to visibility ceiling    |
| `InScope(actor)`               | Entry point: customize scope               |
| `add_to_scope` property        | Bring extra objects into scope             |
| `light` attribute              | Object provides illumination               |
| `transparent` attribute        | Object contents visible / light passes     |
| `thedark` object               | Proxy location during darkness             |
| `location` global              | Current location (may be `thedark`)        |
| `real_location` global         | Actual room (never `thedark`)              |
| `lightflag` global             | `1` if player position is lit              |
| `scope_reason` global          | Reason for current scope search            |
