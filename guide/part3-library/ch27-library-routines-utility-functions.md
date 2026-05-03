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

# Chapter 27: Library Routines and Utility Functions

The standard library provides a collection of public routines that game
authors can call from their own code. These routines handle player
movement, object-tree queries, input parsing, list formatting, action
dispatch, and other common tasks. This chapter provides a complete
reference for each routine, organised by function.

## 27.1 Player Movement

### 27.1.1 `PlayerTo(newplace, flag)`

Moves the player to a new location, handling all the bookkeeping that
a location change requires: departure notification, floating object
repositioning, light recalculation, and optionally a room description.

**Defined in:** `verblib.h`

**Arguments:**

- `newplace` — the room (or enterable container/supporter) to move the
  player to.
- `flag` — controls what happens after the move:

| `flag` | Effect                                                   |
|------- |--------------------------------------------------------- |
| 0      | Perform a `Look` (print the room description).          |
| 1      | Call `NoteArrival()` and `ScoreArrival()` (award points  |
|        | for visiting new rooms), but do not print a description. |
| 2      | Call `LookSub(1)` — a full `Look` as if the player just |
|        | typed `LOOK`.                                            |

**Implementation:**

```inform6
[ PlayerTo newplace flag;
    NoteDeparture();
    move player to newplace;
    while (parent(newplace)) newplace = parent(newplace);
    location = real_location = newplace;
    MoveFloatingObjects(); AdjustLight(1);
    switch (flag) {
      0:    <Look>;
      1:    NoteArrival(); ScoreArrival();
      2:    LookSub(1);
    }
];
```

**Processing steps:**

1. `NoteDeparture()` records the departure from the current location.
2. `move player to newplace` physically places the player in the new
   object.
3. The loop `while (parent(newplace))` walks up the tree to find the
   outermost room — this handles the case where `newplace` is an
   enterable container or supporter inside a room.
4. `location` and `real_location` are set to the outermost room.
5. `MoveFloatingObjects()` repositions objects with `found_in`
   properties.
6. `AdjustLight(1)` recalculates lighting without performing a `Look`.
7. The `flag` determines whether a room description is printed.

**Usage:**

```inform6
PlayerTo(treasure_cave);        ! Move and describe
PlayerTo(treasure_cave, 1);     ! Move silently (no description)
```

**Notes:**
- `PlayerTo` is the correct way to move the player from game code. Do
  not use `move player to room` directly, as that skips floating
  objects, light adjustment, and scoring.
- `PlayerTo` calls the `NewRoom()` entry point (via the `Look` action)
  if flag is 0 or 2.

### 27.1.2 `MoveFloatingObjects()`

Repositions all **floating objects** — objects with a `found_in`
property — to their correct locations based on the player's current
position.

**Defined in:** `verblib.h`

**How it works:** The routine iterates over all objects that have a
non-zero `found_in` property. For each such object:

1. If `found_in` is a **routine**, it is called. If it returns `true`,
   the object is moved to the player's current `location`. If `false`,
   the object is removed from play.

2. If `found_in` is a **list of rooms**, the routine checks whether
   the player's `location` appears in the list. If so, the object is
   moved there; otherwise, it is removed.

Objects with the `non_floating` attribute (which is an alias for
`absent`) are skipped by the floating-object logic and are not
repositioned. In addition, `MoveFloatingObjects` removes **any**
object that has the `absent` attribute from the object tree, even
objects that do not have a `found_in` property.

**Usage:** `MoveFloatingObjects` is called automatically by `PlayerTo`
and should rarely need to be called directly.

## 27.2 Daemon and Timer Routines

These routines manage the daemon/timer pool. See §24 for a complete
discussion.

### 27.2.1 `StartDaemon(obj)`

Activates the daemon for `obj`. See §24.3.2.

### 27.2.2 `StopDaemon(obj)`

Deactivates the daemon for `obj`. See §24.3.3.

### 27.2.3 `StartTimer(obj, timer)`

Starts a countdown timer on `obj`. See §24.4.2.

### 27.2.4 `StopTimer(obj)`

Cancels the timer on `obj`. See §24.4.3.

## 27.3 Scope Routines

These routines query and manipulate the scope system. See §25 for a
complete discussion.

### 27.3.1 `TestScope(obj, actor)`

Tests whether `obj` is in scope for `actor`. See §25.4.1.

### 27.3.2 `LoopOverScope(routine, actor)`

Calls `routine(obj)` for every in-scope object. See §25.4.2.

### 27.3.3 `PlaceInScope(thing)`

Manually adds `thing` to the current scope. See §25.4.3.

### 27.3.4 `ScopeWithin(domain)`

Adds all children of `domain` to scope. See §25.4.4.

## 27.4 Object-Tree Queries

### 27.4.1 `ObjectIsUntouchable(item, flag1, flag2)`

Determines whether there is a physical barrier between the current
actor and `item`. This routine checks for barriers such as closed
containers and animate object possession that would prevent the actor
from physically manipulating the item.

**Defined in:** `verblib.h`

**Arguments:**

- `item` — the object to test.
- `flag1` — if `true` (1), suppress error messages; just return the
  result.
- `flag2` — if `true` (1), also check `Take`/`Remove` restrictions
  (e.g., objects held by NPCs or behind transparent barriers).

**Return value:**

| Value   | Meaning                                                |
|-------- |------------------------------------------------------- |
| `true`  | A barrier exists; `item` cannot be touched.            |
| `false` | No barrier; `item` is accessible.                      |

**Algorithm:**

1. Find the `CommonAncestor` of the actor and the item.
2. Walk up from the actor to the ancestor: if any closed container is
   found along the path, it is a barrier.
3. Walk up from the item to the ancestor: check for closed containers,
   and (if `flag2` is set) for NPC possession and transparent barriers.
4. If the item was added to scope by `add_to_scope`, the routine first
   checks whether the object that added it is itself touchable.

**Usage:**

```inform6
if (ObjectIsUntouchable(golden_key, true))
    "You can see the key but can't reach it.";
```

### 27.4.2 `IndirectlyContains(o1, o2)`

Tests whether `o1` indirectly contains `o2` — that is, whether `o2`
is a child, grandchild, or further descendant of `o1`.

**Defined in:** `verblib.h`

**Arguments:**

- `o1` — the potential ancestor.
- `o2` — the potential descendant.

**Return value:** `true` if `o1` contains `o2` (at any depth), `false`
otherwise.

**Implementation:**

```inform6
[ IndirectlyContains o1 o2;
    while (o2) {
        if (o1 == o2) rtrue;
        if (o2 ofclass Class) rfalse;
        o2 = parent(o2);
    }
    rfalse;
];
```

The routine walks up the tree from `o2`, comparing each parent to
`o1`. It stops if it reaches a `Class` object (which cannot contain
regular objects) or the root.

**Usage:**

```inform6
if (IndirectlyContains(player, golden_key))
    "You have the key somewhere on your person.";
```

### 27.4.3 `CommonAncestor(o1, o2)`

Finds the nearest common ancestor of two objects — the lowest object
in the tree that indirectly contains both `o1` and `o2`.

**Defined in:** `verblib.h`

**Arguments:**

- `o1` — the first object.
- `o2` — the second object.

**Return value:** The common ancestor object, or `0` if the two
objects share no common ancestor (i.e., they are in completely separate
parts of the tree).

**Implementation:**

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

For each ancestor of `o1`, the routine checks whether it is also an
ancestor of `o2`. The first match found is the nearest common
ancestor.

**Usage:**

```inform6
anc = CommonAncestor(player, treasure);
if (anc == location)
    "The treasure is right here in the room with you.";
```

## 27.5 Action Dispatch

### 27.5.1 `RunRoutines(obj, prop)`

Calls the property routine `prop` on object `obj`, with special
handling for the `thedark` proxy object.

**Defined in:** `parser.h`

**Arguments:**

- `obj` — the object whose property routine to call.
- `prop` — the property to invoke (e.g., `before`, `after`,
  `description`).

**Return value:** The return value of the property routine, or `false`
if the property does not exist.

**Implementation:**

```inform6
[ RunRoutines obj prop;
    if (obj == thedark && prop ~= initial or short_name or description)
        obj = real_location;
    if (obj.&prop == 0 && prop >= INDIV_PROP_START) rfalse;
    return obj.prop();
];
```

**Notes:**
- When `obj` is `thedark` and the property is not one of the display
  properties (`initial`, `short_name`, `description`), the routine
  substitutes `real_location`. This ensures that action-handling
  properties like `before` and `after` are dispatched to the actual
  room, not to the darkness proxy.
- If the property does not exist on the object (and it is an
  individual property), the routine returns `false` without calling
  anything.

### 27.5.2 `PrintOrRun(obj, prop, flag)`

Handles a property that can be either a string or a routine. If it is
a string, the string is printed. If it is a routine, the routine is
called.

**Defined in:** `parser.h`

**Arguments:**

- `obj` — the object.
- `prop` — the property.
- `flag` — if `0`, print a newline after a string value; if `1`, do
  not print a newline.

**Return value:** `true` if the property was a string (it was
printed), or the return value of the routine if it was a routine, or
`false` if the property value was `NULL`.

**Implementation:**

```inform6
[ PrintOrRun obj prop flag;
    if (obj.#prop > WORDSIZE) return RunRoutines(obj, prop);
    if (obj.prop == NULL) rfalse;
    switch (metaclass(obj.prop)) {
      Class, Object, nothing:
        return RunTimeError(2,obj,prop);
      String:
        print (string) obj.prop;
        if (flag == 0) new_line;
        rtrue;
      Routine:
        return RunRoutines(obj, prop);
    }
];
```

**Notes:**
- If the property data is longer than one word (`obj.#prop > WORDSIZE`),
  it is treated as a routine (multiple-entry properties are always
  routine code).
- A `NULL` value means the property is unset; `PrintOrRun` returns
  `false`.

### 27.5.3 `AfterRoutines()`

Runs the full chain of post-action processing: `react_after`
properties for all in-scope objects, the location's `after` property,
the noun's `after` property, and finally `GamePostRoutine()`.

**Defined in:** `parser.h`

**Return value:** `true` if any handler in the chain returned `true`.

**Implementation:**

```inform6
[ AfterRoutines rv;
    scope_reason = REACT_AFTER_REASON; parser_one = 0;
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

## 27.6 Input Parsing Utilities

### 27.6.1 `NextWord()`

Returns the dictionary word value at the current word position (`wn`)
in the parsed input and advances `wn` by one.

**Defined in:** `parser.h`

**Return value:** The dictionary word value, or `false` (0) if `wn` is
out of range or the word is not in the dictionary.

**[Z-machine] Implementation:**

```inform6
[ NextWord i j;
    if (wn <= 0 || wn > parse->1) { wn++; rfalse; }
    i = wn*2-1; wn++;
    j = parse-->i;
    if (j == ',//') j = comma_word;
    if (j == './/') j = THEN1__WD;
    return j;
];
```

**[Glulx] Implementation:**

```inform6
[ NextWord i j;
    if (wn <= 0 || wn > parse-->0) { wn++; rfalse; }
    i = wn*3-2; wn++;
    j = parse-->i;
    if (j == ',//') j = comma_word;
    if (j == './/') j = THEN1__WD;
    return j;
];
```

**Notes:**
- Commas and periods are normalized to `comma_word` and `THEN1__WD`
  respectively.
- The global variable `wn` (word number) tracks the current position,
  starting at 1.
- If the word is not in the dictionary, the parse array holds 0 and
  `NextWord` returns 0.

### 27.6.2 `NextWordStopped()`

Like `NextWord()`, but returns `-1` (rather than 0) when `wn` is past
the end of the input. This allows the caller to distinguish between
"no more words" and "word not in dictionary."

**Defined in:** `parser.h`

**Return value:** The dictionary word value, `0` if the word is not in
the dictionary, or `-1` if past the end of input.

### 27.6.3 `WordAddress(wordnum, parse_array, buffer_array)`

Returns the byte address of word `wordnum` in the input buffer.

**Defined in:** `parser.h`

**Arguments:**

- `wordnum` — the 1-based word index.
- `parse_array` — the parse array (0 for the default `parse`).
- `buffer_array` — the buffer array (0 for the default `buffer`).

**Return value:** The absolute byte address of the word's text in the
buffer.

**[Z-machine/Glulx difference]** The implementation differs because
the parse array layout differs between platforms. On the Z-machine,
each parse entry is 4 bytes; on Glulx, each entry is 3 words (12
bytes).

### 27.6.4 `WordLength(wordnum, parse_array)`

Returns the byte length of word `wordnum` in the input buffer.

**Defined in:** `parser.h`

**Arguments:**

- `wordnum` — the 1-based word index.
- `parse_array` — the parse array (0 for the default `parse`).

**Return value:** The number of characters in the word.

### 27.6.5 `TryNumber(wordnum)`

Attempts to parse word `wordnum` as a numeric value.

**Defined in:** `parser.h`

**Arguments:** `wordnum` — the 1-based word index.

**Return value:**

| Value   | Meaning                                                |
|-------- |------------------------------------------------------- |
| 0–10000 | The parsed numeric value.                              |
| -1000   | The word is not a number.                              |
| 10000   | The word is a number but exceeds 4 digits.             |

**Algorithm:**

1. Check `NumberWord()` for English number words ("one", "two", ...,
   "twenty"). If found, return the value.
2. Call `ParseNumber()` (the game's entry point). If it returns
   non-zero, return that value.
3. Call any library extension `ext_parsenumber` routines.
4. Attempt to parse the word as a string of decimal digits. If all
   characters are digits and the number is 4 digits or fewer, return
   the value. If more than 4 digits, return 10000.
5. If any non-digit character is found, return -1000.

**Usage:**

```inform6
wn = 2;   ! Check the second word
n = TryNumber(wn);
if (n == -1000) "That's not a number.";
print "You typed the number ", n, ".^";
```

## 27.7 User Interaction

### 27.7.1 `YesOrNo()`

Reads a line of input and returns `true` for "yes" or `false` for
"no", prompting again if the response is neither.

**Defined in:** `verblib.h`

**Arguments:** One optional argument `noStatusRedraw` — if `true`,
suppress status line redraw during input.

**Return value:** `true` for yes, `false` for no.

**Implementation:**

```inform6
[ YesOrNo noStatusRedraw
    i j;
    for (::) {
        #Ifdef TARGET_ZCODE;
        if (location == nothing || parent(player) == nothing
            || noStatusRedraw)
            read buffer parse;
        else read buffer parse DrawStatusLine;
        j = parse->1;
        #Ifnot; ! TARGET_GLULX;
        noStatusRedraw = 0;
        KeyboardPrimitive(buffer, parse);
        j = parse-->0;
        #Endif;
        if (j) {
            i = parse-->1;
            if (i == YES1__WD or YES2__WD or YES3__WD) rtrue;
            if (i == NO1__WD  or NO2__WD  or NO3__WD) rfalse;
        }
        L__M(##Quit, 1); print "> ";
    }
];
```

**Notes:**
- The routine loops until a valid response is received. Invalid
  responses produce the message "Please answer yes or no." (or its
  language-specific equivalent).
- The recognized words are defined by language constants: `YES1__WD`,
  `YES2__WD`, `YES3__WD` (typically "yes", "y", "oui"), and
  `NO1__WD`, `NO2__WD`, `NO3__WD` (typically "no", "n").

## 27.8 Display and Output

### 27.8.1 `Banner()`

Prints the game's banner: the story title, headline, release number,
serial number, compiler version, and library version.

**Defined in:** `verblib.h`

**Arguments:** None.

**Return value:** None (the routine always prints).

**Format:** The banner has this general form:

```
Ruins
An Interactive Catastrophe
Release 1 / Serial number 260411 / Inform v6.44 / Library 6.12.8 SD
```

The final letters indicate compilation flags: `S` for strict mode,
`D` for debug mode, `X` for infix mode.

**Notes:**
- If a `LanguageBanner` routine is defined, it is called instead of
  the default banner code, giving the language module full control over
  the banner format.
- The `Story` and `Headline` constants must be defined in the game
  source.
- `Banner()` can be called at any time; the `VERSION` meta-action
  calls it.

### 27.8.2 `WriteListFrom(obj, style, depth)`

Prints a formatted list of objects starting from `obj`. This is the
library's primary list-formatting routine, used for inventory, room
contents, and any situation where a set of objects needs to be
displayed.

**Defined in:** `verblib.h`

**Arguments:**

- `obj` — the first object to list (the routine lists `obj` and its
  siblings).
- `style` — a bitmask of style flags controlling formatting.
- `depth` — recursion depth (usually 0 for the top level).

**Return value:** `true` if anything was printed, `false` (0) if the
list was empty.

**Style flags:**

| Constant         | Value    | Effect                                    |
|----------------- |--------- |------------------------------------------ |
| `NEWLINE_BIT`    | `$0001`  | Print a newline after each entry.         |
| `INDENT_BIT`     | `$0002`  | Indent each entry by depth.               |
| `FULLINV_BIT`    | `$0004`  | Full inventory info after each entry.     |
| `ENGLISH_BIT`    | `$0008`  | English sentence style (commas and "and").|
| `RECURSE_BIT`    | `$0010`  | Recurse into containers/supporters.       |
| `ALWAYS_BIT`     | `$0020`  | Always recurse (even closed containers).  |
| `TERSE_BIT`      | `$0040`  | Shorter English style.                    |
| `PARTINV_BIT`    | `$0080`  | Brief inventory info after each entry.    |
| `DEFART_BIT`     | `$0100`  | Use the definite article ("the").         |
| `WORKFLAG_BIT`   | `$0200`  | At top level, list only objects with       |
|                  |          | the `workflag` attribute.                 |
| `ISARE_BIT`      | `$0400`  | Print "is" or "are" before the list.      |
| `CONCEAL_BIT`    | `$0800`  | Omit `concealed` and `scenery` objects.   |
| `NOARTICLE_BIT`  | `$1000`  | Print no articles at all.                 |
| `ID_BIT`         | `$2000`  | Print object ID after each entry.         |

**Usage:**

```inform6
! Print room contents in English sentence form:
WriteListFrom(child(location),
    ENGLISH_BIT + RECURSE_BIT + PARTINV_BIT + CONCEAL_BIT);

! Print a full inventory:
WriteListFrom(child(player),
    FULLINV_BIT + ENGLISH_BIT + RECURSE_BIT + NEWLINE_BIT);
```

**Notes:**
- `WriteListFrom` sorts the objects using `SortOutList` before
  printing, grouping objects with matching `list_together` properties.
- The routine uses `WriteListR` for the actual recursive traversal and
  printing.

### 27.8.3 `DrawStatusLine()`

Redraws the status line at the top of the screen.

**Defined in:** `parser.h`

**[Z-machine]** The routine uses the Z-machine's window 1 (the status
window) to display the location name on the left and either the
score/moves or the time on the right. The exact layout depends on the
Z-machine version and screen width.

**[Glulx]** The routine uses Glk window operations to clear and
redraw the status window.

**Notes:**
- `DrawStatusLine` is called automatically during input. Games can
  also call it explicitly to force a status line update.
- The status line displays `location` if `location` provides a
  `short_name`, or prints the room name via `(name)`.
- In time-of-day mode (`Statusline time`), the right side shows
  hours and minutes from `the_time`.

## 27.9 Time and Scoring

### 27.9.1 `SetTime(time, rate)`

Initialises the game clock. See §24.7.2.

**Defined in:** `parser.h`

**Arguments:**

- `time` — minutes since midnight (0–1439).
- `rate` — minutes per turn (positive) or turns per minute (negative).

### 27.9.2 `DisplayStatus()`

Updates the status line variables `sline1` and `sline2`. If
`sys_statusline_flag` is 0, they hold the score and move count;
otherwise, they hold hours and minutes.

**Defined in:** `parser.h`

### 27.9.3 `ScoreArrival()`

Awards points the first time the player visits a room. Called
automatically by `PlayerTo` (with `flag = 1`) and by the standard
movement-handling code.

**Defined in:** `verblib.h`

**Implementation:**

```inform6
[ ScoreArrival;
    if (location hasnt visited) {
        give location visited;
        if (location has scored) {
            score = score + ROOM_SCORE;
            places_score = places_score + ROOM_SCORE;
        }
    }
];
```

**Notes:**
- The `visited` attribute is set on the room the first time it is
  scored, so subsequent visits do not re-award points.
- Rooms with the `scored` attribute add `ROOM_SCORE` to both `score`
  and `places_score`. Rooms without `scored` are still marked as
  `visited` but do not award points.

## 27.10 Player Identity

### 27.10.1 `ChangePlayer(obj, flag)`

Switches the active player to a different object. After the call,
`player` refers to `obj`, and the parser, scope, and display logic
treat `obj` as the protagonist.

**Defined in:** `parser.h`

**Arguments:**

- `obj` — the object to make the player. If `obj` is `nothing`, the
  default `selfobj` is used.
- `flag` — stored in the global `print_player_flag`, used by the
  library to control whether the new player's name is announced.

**Effects:**

1. The previous player loses the temporarily-set `transparent` and
   `concealed` attributes that flagged it as the current player.
2. The new player gains `transparent`, `concealed`, and `animate`.
3. `location` and `real_location` are recomputed from the new player's
   position by walking up the object tree to the outermost room.
4. `MoveFloatingObjects()` is called to reposition floating objects
   for the new player's location.
5. Lighting is recalculated; `location` is set to `thedark` if the new
   position offers no light.
6. If the previous `actor` was the previous player, `actor` is updated
   to the new player.
7. If `parent(obj) == 0` after the switch, runtime error 10 is raised.

**Usage:**

```inform6
ChangePlayer(robot, 0);
```

**Notes:**
- The new player object must be inside a room (somewhere with a
  non-zero parent chain ending at a room) before the call.
- `ChangePlayer` is the standard way to support multi-character games
  or temporary possession effects.

## 27.11 Action Invocation

### 27.11.1 Calling Actions from Code

The `<Action>` and `<<Action>>` syntax allows game code to invoke
actions programmatically (see §20 for a complete discussion):

```inform6
<Take lamp>;            ! Invoke Take action, continue execution
<<Take lamp>>;          ! Invoke Take action and return true
<Give lamp second>;     ! Two-noun action
```

These are compile-time constructs handled by the compiler, not library
routines.

## 27.12 Miscellaneous Utilities

### 27.12.1 `Random(n)` / `random(a, b, c, ...)`

Though not a library routine (it is a language built-in), `random` is
used extensively:

- `random(n)` returns a random integer from 1 to `n`.
- `random(a, b, c)` returns one of the listed values at random.

### 27.12.2 `metaclass(obj)`

Returns the metaclass of a value — `Object`, `Class`, `Routine`,
`String`, or `nothing`. See §7.9 for details.

## 27.13 Summary Table

| Routine                            | Purpose                         | §     |
|----------------------------------- |-------------------------------- |------ |
| `PlayerTo(place, flag)`            | Move player to location         | 27.1  |
| `MoveFloatingObjects()`            | Reposition floating objects     | 27.1  |
| `StartDaemon(obj)`                 | Activate daemon                 | 24.3  |
| `StopDaemon(obj)`                  | Deactivate daemon               | 24.3  |
| `StartTimer(obj, timer)`           | Start countdown timer           | 24.4  |
| `StopTimer(obj)`                   | Cancel timer                    | 24.4  |
| `TestScope(obj, actor)`            | Is object in scope?             | 25.4  |
| `LoopOverScope(routine, actor)`    | Iterate over scope              | 25.4  |
| `PlaceInScope(thing)`              | Add to scope                    | 25.4  |
| `ScopeWithin(domain)`              | Add children to scope           | 25.4  |
| `ObjectIsUntouchable(item,f1,f2)`  | Check for barriers              | 27.4  |
| `IndirectlyContains(o1, o2)`       | Containment test                | 27.4  |
| `CommonAncestor(o1, o2)`           | Find common ancestor            | 27.4  |
| `RunRoutines(obj, prop)`           | Call property routine            | 27.5  |
| `PrintOrRun(obj, prop, flag)`      | Print string or call routine    | 27.5  |
| `AfterRoutines()`                  | Post-action chain               | 27.5  |
| `NextWord()`                       | Get next parsed word            | 27.6  |
| `NextWordStopped()`                | Get next word (with stop)       | 27.6  |
| `WordAddress(n, p, b)`             | Address of word in buffer       | 27.6  |
| `WordLength(n, p)`                 | Length of word in buffer        | 27.6  |
| `TryNumber(wordnum)`               | Parse word as number            | 27.6  |
| `YesOrNo()`                        | Read yes/no response            | 27.7  |
| `Banner()`                         | Print game banner               | 27.8  |
| `WriteListFrom(obj, style, depth)` | Print formatted object list     | 27.8  |
| `DrawStatusLine()`                 | Redraw status line              | 27.8  |
| `SetTime(time, rate)`              | Initialise game clock           | 27.9  |
| `DisplayStatus()`                  | Update status variables         | 27.9  |
| `ScoreArrival()`                   | Award room-visit points         | 27.9  |
| `ChangePlayer(obj, flag)`          | Switch the active player        | 27.10 |
