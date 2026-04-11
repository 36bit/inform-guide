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

# Chapter 23: Time, Daemons, and Each-Turn Processing

The Inform library provides several mechanisms for making the game world
respond to the passage of time. **Daemons** execute every turn
regardless of the player's actions. **Timers** (also called fuses)
count down toward a one-time event. The **`each_turn`** property
provides automatic per-turn processing for objects currently in scope.
Together, these systems allow the game world to evolve independently of
player commands. The information in this chapter is derived from
`parser.h` and `verblib.h` in library version 6.12.8, with
VM-specific constants drawn from `linklpa.h`.

## §23.1 The Turn Counter

The library tracks the number of turns played in the global variable
`turns`, declared in `parser.h` (line 259):

```inform6
Global turns = START_MOVE;
```

The constant `START_MOVE` is defined at line 256 of `parser.h`:

```inform6
Constant START_MOVE 0;     ! Traditionally 0 for Infocom, 1 for Inform
```

A game can override `START_MOVE` before including the library to change
the initial turn number. The default is `0`.

### §23.1.1 When `turns` Is Incremented

The `turns` counter is incremented once per successful non-meta action,
at the start of the end-of-turn sequence (see §23.2). Specifically,
`AdvanceWorldClock()` increments `turns` immediately before running
timers, daemons, and `each_turn` processing.

Actions that do not count as a turn include:

- **Meta actions** — commands such as `SAVE`, `RESTORE`, `SCORE`, and
  `QUIT` do not consume a game turn. The end-of-turn sequence is
  skipped entirely for meta actions.
- **Implicit takes** — when the library automatically takes an object
  before performing a requested action, the implicit take does not
  increment the turn counter. Only the main action counts.
- **Actions blocked by `before`** — if a `before` handler returns
  `true`, the action is blocked but the turn is still consumed. The
  end-of-turn sequence runs normally.

### §23.1.2 The Status Line Display

The library displays the turn counter on the status line through the
`DisplayStatus` routine (parser.h, line 5756):

```inform6
[ DisplayStatus;
    if (sys_statusline_flag == 0) {
        sline1 = score;
        sline2 = turns;
    }
    else {
        sline1 = the_time/60;
        sline2 = the_time%60;
    }
];
```

When `sys_statusline_flag` is `0` (score/turns mode), the right side of
the status line shows the current score and turn count. When
`sys_statusline_flag` is `1` (time mode), it shows hours and minutes
instead. See §23.6 for details on timed games.

## §23.2 The End-of-Turn Sequence

After each non-meta action completes, the library runs the
**end-of-turn sequence**. This is the mechanism by which the world
advances: clocks tick, daemons fire, timers count down, and lighting
conditions are re-evaluated.

The end-of-turn sequence executes the following steps in order:

1. **`AdvanceWorldClock()`** — increments `turns` and advances the game
   clock if time-based play is enabled (see §23.6).
2. **`RunTimersAndDaemons()`** — fires all active daemons and
   decrements all active timers, calling `time_out` on any that reach
   zero (see §23.3 and §23.4).
3. **`RunEachTurnProperties()`** — calls the `each_turn` property on
   every in-scope object (see §23.5).
4. **`TimePasses()`** — calls the game author's entry point, if defined
   (see §23.7). If `TimePasses` returns `0`, the library additionally
   calls `LibraryExtensions.RunAll(ext_timepasses)`.
5. **`AdjustLight()`** — recalculates whether the player's location is
   lit or dark, and issues `##Darkness` or room descriptions as needed.
6. **`NoteObjectAcquisitions()`** — tracks any objects newly acquired by
   the player for scoring purposes.

### §23.2.1 The `deadflag` Check

Between every step, the library checks whether `deadflag` has become
non-zero. If it has, the end-of-turn sequence halts immediately:

```inform6
AdvanceWorldClock();        if (deadflag) return;
RunTimersAndDaemons();      if (deadflag) return;
RunEachTurnProperties();    if (deadflag) return;
if (TimePasses() == 0)
    LibraryExtensions.RunAll(ext_timepasses);
                            if (deadflag) return;
AdjustLight();              if (deadflag) return;
NoteObjectAcquisitions();
```

This means that if a daemon kills the player (sets `deadflag` to `1`),
the `each_turn` routines will not fire on that turn. Similarly, if a
timer's `time_out` routine ends the game, `TimePasses` is never called.
The order of operations is deterministic and guaranteed by the library.

## §23.3 Daemons

A **daemon** is a routine attached to an object that fires once per
turn for as long as it is active. Unlike `each_turn` (§23.5), a daemon
runs regardless of whether its object is in scope — it fires even if
the player is in a different room.

### §23.3.1 The `daemon` Property

A daemon is declared using the `daemon` property on an object:

```inform6
Object  security_guard "security guard" guard_room
  with  name 'security' 'guard',
        daemon [;
            ! Move the guard to a random adjacent room
            if (children(parent(self)) > 1)
                move self to random_adjacent_room(parent(self));
            if (self in location)
                "^The security guard strolls into view.";
        ],
        description "A burly security guard in a blue uniform.",
  has   animate;
```

The `daemon` property holds a routine. The library does not inspect or
call this routine automatically — it is only invoked after
`StartDaemon` has been called to register the object.

### §23.3.2 `StartDaemon(obj)`

The `StartDaemon` routine registers an object's daemon for per-turn
execution. Its implementation (parser.h, lines 5735–5744):

```inform6
[ StartDaemon obj i;
    for (i=0 : i<active_timers : i++)
        if (the_timers-->i == WORD_HIGHBIT + obj) rfalse;
    for (i=0 : i<active_timers : i++)
        if (the_timers-->i == 0) jump FoundTSlot3;
    i = active_timers++;
    if (i >= MAX_TIMERS) RunTimeError(4);
  .FoundTSlot3;
    the_timers-->i = WORD_HIGHBIT + obj;
];
```

Key details:

- **Duplicate check**: the first loop scans `the_timers` for an
  existing entry of `WORD_HIGHBIT + obj`. If found, `StartDaemon`
  returns `false` without adding a duplicate.
- **Slot reuse**: the second loop looks for an empty slot (`0`) in the
  existing array. Empty slots arise when timers or daemons are stopped.
- **Array growth**: if no empty slot exists, `active_timers` is
  incremented to extend the used portion of the array. If this exceeds
  `MAX_TIMERS`, a runtime error is raised.
- **Storage format**: daemons are stored as `WORD_HIGHBIT + obj`. The
  high bit distinguishes daemons from timers in the shared
  `the_timers` array (see §23.3.4).

### §23.3.3 `StopDaemon(obj)`

The `StopDaemon` routine removes a daemon from the active list.
Its implementation (parser.h, lines 5746–5752):

```inform6
[ StopDaemon obj i;
    for (i=0 : i<active_timers : i++)
        if (the_timers-->i == WORD_HIGHBIT + obj) jump FoundTSlot4;
    rfalse;
  .FoundTSlot4;
    the_timers-->i = 0;
];
```

The routine scans `the_timers` for `WORD_HIGHBIT + obj`. If found, the
slot is set to `0`, effectively deregistering the daemon. If the daemon
is not active, `StopDaemon` returns `false` silently.

Note that `StopDaemon` does not shrink `active_timers` — the slot
remains as a zero entry that can be reused by a later `StartDaemon` or
`StartTimer` call.

### §23.3.4 Internal Representation

Daemons and timers share a single array, `the_timers`, declared in
`parser.h` (lines 265–268):

```inform6
Constant MAX_TIMERS  32;
Array  the_timers  --> MAX_TIMERS;
Global active_timers;
```

The `active_timers` global tracks the high-water mark of the used
portion of the array. The array stores both timers and daemons,
distinguished by the `WORD_HIGHBIT` flag:

- **Daemon entry**: `WORD_HIGHBIT + obj` — the high bit is set.
- **Timer entry**: `obj` — the high bit is clear. The object reference
  is stored directly.
- **Empty slot**: `0` — a previously stopped timer or daemon.

The `WORD_HIGHBIT` constant is defined in `linklpa.h` and differs by
target platform:

**[Z-machine]** `Constant WORD_HIGHBIT = $8000;`

**[Glulx]** `Constant WORD_HIGHBIT = $80000000;`

The `RunTimersAndDaemons` routine tests each entry with
`j & WORD_HIGHBIT` to determine whether it is a daemon or a timer
(see §23.4.3).

### §23.3.5 `MAX_TIMERS`

The `MAX_TIMERS` constant (default `32`) sets the maximum number of
simultaneously active timers and daemons. Timers and daemons share
this pool. A game that needs more than 32 concurrent timers and
daemons can increase this limit:

```inform6
Constant MAX_TIMERS 64;
Include "parser.h";
```

The constant must be defined before `parser.h` is included.

### §23.3.6 Example: A Wandering NPC

The following example implements an NPC that wanders between rooms
every turn while the daemon is active:

```inform6
Object  old_cat "old cat" kitchen
  with  name 'old' 'cat' 'tabby',
        daemon [;
            if (random(3) == 1) {
                ! One-in-three chance of moving each turn
                switch (random(3)) {
                    1: move self to kitchen;
                    2: move self to hallway;
                    3: move self to garden;
                }
            }
            if (parent(self) == location)
                "^The old tabby cat stretches and yawns.";
        ],
        description "A grizzled old tabby cat.",
        before [;
            Take:
                "The cat hisses and darts away.";
        ],
  has   animate;

[ Initialise;
    location = kitchen;
    StartDaemon(old_cat);
    "^You are in your cottage. An old cat dozes nearby.";
];
```

When the game starts, `Initialise` calls `StartDaemon(old_cat)` to
activate the daemon. Each turn, the cat has a one-in-three chance of
relocating. If the cat is in the player's current location, a message
is printed. The daemon runs regardless of where the player is.

To stop the cat's wandering (for example, if the player feeds it):

```inform6
StopDaemon(old_cat);
move old_cat to location;
"The cat settles into your lap, purring contentedly.";
```

## §23.4 Timers (Fuses)

A **timer** (historically called a **fuse**) is a countdown mechanism
attached to an object. When started, the timer counts down from a
specified value. When the countdown reaches zero, a designated routine
is called and the timer is automatically stopped.

### §23.4.1 Timer Properties

Timers use two properties:

- **`time_left`** — the current countdown value. Decremented by the
  library each turn. When it reaches zero, `time_out` is called.
- **`time_out`** — the routine called when the countdown expires.

Both properties must be declared on any object used with timers. The
`time_left` property must be explicitly provided (even if set to `0`)
because `StartTimer` checks for its existence via `obj.&time_left` and
raises a runtime error if it is absent.

```inform6
Object  bomb "ticking bomb" vault
  with  name 'ticking' 'bomb',
        time_out [;
            deadflag = 1;
            "^BOOM! The bomb explodes, killing you instantly!";
        ],
        time_left 0,
        description [;
            if (self.time_left > 0) {
                print "A bomb with a display reading ";
                print self.time_left;
                " seconds.";
            }
            "A bomb. Its display is dark.";
        ],
  has   ;
```

### §23.4.2 `StartTimer(obj, value)`

The `StartTimer` routine registers an object's timer and sets its
initial countdown. Its implementation (parser.h, lines 5714–5724):

```inform6
[ StartTimer obj timer i;
    for (i=0 : i<active_timers : i++)
        if (the_timers-->i == obj) rfalse;
    for (i=0 : i<active_timers : i++)
        if (the_timers-->i == 0) jump FoundTSlot;
    i = active_timers++;
    if (i >= MAX_TIMERS) { RunTimeError(4); return; }
  .FoundTSlot;
    if (obj.&time_left == 0) { RunTimeError(5, obj, time_left); return; }
    the_timers-->i = obj; obj.time_left = timer;
];
```

Key details:

- **Duplicate check**: the first loop prevents registering the same
  timer twice. If `obj` is already in `the_timers`, `StartTimer`
  returns `false`.
- **Property check**: `obj.&time_left == 0` tests whether the object
  provides the `time_left` property. If not, runtime error 5 is raised.
- **Initial countdown**: `obj.time_left` is set to the `timer`
  argument. The countdown begins on the next turn.
- **Storage**: the object reference is stored directly in `the_timers`
  (without `WORD_HIGHBIT`), distinguishing timers from daemons.

### §23.4.3 How Timers Are Processed

Each turn, `RunTimersAndDaemons` processes every active entry in
`the_timers`. For timer entries (those without `WORD_HIGHBIT` set), the
following logic applies:

```inform6
if (j & WORD_HIGHBIT) {
    RunRoutines(j & ~WORD_HIGHBIT, daemon);
}
else {
    if (j.time_left == 0) {
        StopTimer(j);
        RunRoutines(j, time_out);
    }
    else
        j.time_left = j.time_left - 1;
}
```

The processing for each timer entry is:

1. If `time_left` is greater than zero, decrement it by one.
2. If `time_left` is zero, stop the timer and call `time_out`.

Note the order: decrementing happens first. If `StartTimer(obj, 3)` is
called, the sequence is:

| Turn | `time_left` at start | Action | `time_left` at end |
|------|---------------------|--------|--------------------|
| 1    | 3                   | Decrement | 2               |
| 2    | 2                   | Decrement | 1               |
| 3    | 1                   | Decrement | 0               |
| 4    | 0                   | Fire `time_out`, stop timer | — |

The timer fires on the fourth turn after it is started when set to 3,
meaning the effective countdown is `value + 1` turns from start to
firing.

### §23.4.4 `StopTimer(obj)`

The `StopTimer` routine removes a timer from the active list and resets
its countdown. Its implementation (parser.h, lines 5726–5733):

```inform6
[ StopTimer obj i;
    for (i=0 : i<active_timers : i++)
        if (the_timers-->i == obj) jump FoundTSlot2;
    rfalse;
  .FoundTSlot2;
    if (obj.&time_left == 0) { RunTimeError(5, obj, time_left); return; }
    the_timers-->i = 0; obj.time_left = 0;
];
```

Key details:

- The slot in `the_timers` is set to `0` (freeing it for reuse).
- `obj.time_left` is explicitly set to `0`.
- If the object lacks a `time_left` property, runtime error 5 is raised
  (the same check as in `StartTimer`).
- If the timer is not active, `StopTimer` returns `false` silently.

### §23.4.5 The Complete `RunTimersAndDaemons` Routine

The full implementation of `RunTimersAndDaemons` (parser.h, lines
5451–5482) processes both daemons and timers in a single pass through
the `the_timers` array:

```inform6
[ RunTimersAndDaemons i j;
    #Ifdef DEBUG;
    if (debug_flag & DEBUG_TIMERS) {
        for (i=0 : i<active_timers : i++) {
            j = the_timers-->i;
            if (j ~= 0) {
                print (name) (j & ~WORD_HIGHBIT), ": ";
                if (j & WORD_HIGHBIT) print "daemon";
                else
                    print "timer with ", j.time_left,
                          " turns to go";
                new_line;
            }
        }
    }
    #Endif; ! DEBUG

    for (i=0 : i<active_timers : i++) {
        if (deadflag) return;
        j = the_timers-->i;
        if (j ~= 0) {
            if (j & WORD_HIGHBIT)
                RunRoutines(j & ~WORD_HIGHBIT, daemon);
            else {
                if (j.time_left == 0) {
                    StopTimer(j);
                    RunRoutines(j, time_out);
                }
                else
                    j.time_left = j.time_left - 1;
            }
        }
    }
];
```

Important details:

- **`deadflag` check**: the `if (deadflag) return;` inside the loop
  means that if a daemon or timer sets `deadflag`, no further timers or
  daemons are processed on that turn. Earlier entries in the array fire
  before later ones.
- **Debug output**: when `DEBUG` is defined and `DEBUG_TIMERS` is set
  in `debug_flag`, the routine prints the state of all active timers
  and daemons before processing them.
- **Array order**: entries are processed in array index order, which
  corresponds to insertion order (the order in which `StartDaemon` and
  `StartTimer` were called), modified by slot reuse.

### §23.4.6 Example: A Bomb with a Fuse

A complete example of a timer-based bomb:

```inform6
Object  time_bomb "time bomb" laboratory
  with  name 'time' 'bomb' 'device',
        time_out [;
            deadflag = 1;
            "^The bomb detonates with a blinding flash!
             Everything goes white...";
        ],
        time_left 0,
        before [;
            Examine:
                if (self.time_left > 0) {
                    print "A cylindrical device with a red LED
                           display reading ";
                    print self.time_left;
                    ".";
                }
                "A cylindrical device. The display is dark.";
            SwitchOn:
                if (self.time_left > 0)
                    "It's already armed!";
                StartTimer(self, 5);
                "You press the activation switch. The display
                 lights up: 5... 4...";
            SwitchOff:
                if (self.time_left == 0)
                    "It isn't running.";
                StopTimer(self);
                "You frantically press the deactivation switch.
                 The display goes dark. Phew.";
        ],
  has   switchable;
```

When the player types `SWITCH ON BOMB`, the timer starts with a
countdown of 5. The bomb explodes on the sixth turn (after five
decrements bring `time_left` to zero, then `time_out` fires). The
player can defuse the bomb by typing `SWITCH OFF BOMB` before the
countdown expires.

### §23.4.7 Example: A Reusable Egg Timer

An object whose timer can be started multiple times:

```inform6
Object  egg_timer "egg timer" kitchen
  with  name 'egg' 'timer',
        time_out [;
            if (self in location)
                "^Ding! The egg timer goes off.";
        ],
        time_left 0,
        description [;
            if (self.time_left > 0) {
                print "The timer shows ";
                print self.time_left;
                " minutes remaining.";
            }
            "The egg timer is idle.";
        ],
        before [;
            Turn:
                StopTimer(self);
                StartTimer(self, 5);
                "You wind the egg timer. It begins ticking.";
        ],
  has   ;
```

Note the pattern of calling `StopTimer` before `StartTimer` when
restarting. Because `StartTimer` returns `false` if the timer is
already active (the duplicate check), you must stop the existing timer
first to change the countdown value. See §23.8 for more on this
interaction.

## §23.5 The `each_turn` Property

The `each_turn` property provides automatic per-turn processing for
objects in scope. Unlike daemons, which must be explicitly started and
stopped and which fire regardless of scope, `each_turn` fires
automatically for any object that is currently visible to the player.

### §23.5.1 Declaration

The `each_turn` property can hold either a routine or a string:

```inform6
Object  dripping_faucet "dripping faucet" bathroom
  with  name 'dripping' 'faucet' 'tap',
        each_turn [;
            if (random(4) == 1)
                "^Drip... drip... drip...";
        ],
        description "A faucet with a slow, persistent drip.",
  has   static scenery;
```

If `each_turn` is a string, it is printed every turn while the object
is in scope:

```inform6
Object  ticking_clock "ticking clock" study
  with  name 'ticking' 'clock',
        each_turn "^Tick... tock... tick... tock...",
        description "A clock on the mantelpiece.",
  has   static scenery;
```

Because `each_turn` is declared as **additive** in `linklpa.h`,
multiple `each_turn` routines can be layered through class inheritance.
When a class defines `each_turn` and a subclass also defines it, both
routines are called.

### §23.5.2 How `RunEachTurnProperties` Works

The library processes `each_turn` properties in the
`RunEachTurnProperties` routine (parser.h, lines 5484–5489):

```inform6
[ RunEachTurnProperties;
    scope_reason = EACH_TURN_REASON; verb_word = 0;
    DoScopeAction(location);
    SearchScope(ScopeCeiling(player), player, 0);
    scope_reason = PARSING_REASON;
];
```

The processing steps are:

1. **Set scope context**: `scope_reason` is set to `EACH_TURN_REASON`
   (constant value `2`, defined at parser.h line 600) and `verb_word`
   is cleared.
2. **Process current room**: `DoScopeAction(location)` calls
   `PrintOrRun(location, each_turn)` for the current room. This means
   the room's `each_turn` property is always called first, regardless
   of scope calculations.
3. **Process in-scope objects**: `SearchScope` walks the scope tree
   from `ScopeCeiling(player)` (normally the room) and calls
   `DoScopeAction` for each object found. `DoScopeAction` calls
   `PrintOrRun(thing, each_turn)` for objects whose `scope_reason` is
   `EACH_TURN_REASON`.
4. **Restore scope context**: `scope_reason` is reset to
   `PARSING_REASON`.

### §23.5.3 Scope and `each_turn`

An object's `each_turn` routine fires only while the object is in
the player's scope. The practical consequence is:

- If the player leaves the room containing the object, `each_turn`
  stops firing (unless the object is a floating object via `found_in`).
- If the object is inside a closed opaque container, it is not in
  scope and `each_turn` does not fire.
- Objects carried by the player are in scope and their `each_turn`
  routines fire normally.

This makes `each_turn` self-managing — there is no need to start or
stop it explicitly. When an object enters scope, processing begins
automatically. When it leaves scope, processing stops.

```inform6
Object  music_box "music box" parlour
  with  name 'music' 'box' 'musical',
        each_turn [;
            "^The music box plays a tinkling melody.";
        ],
        before [;
            Close:
                give self ~open;
                "You snap the music box shut. Silence.";
            Open:
                give self open;
                "You open the music box. A delicate melody
                 begins to play.";
        ],
        description "An ornate music box.",
  has   container open openable;
```

In this example, the music box plays each turn while it is in scope.
If the player puts it inside a closed opaque container or leaves the
room, the music stops (because the object leaves scope). No explicit
start/stop logic is required.

### §23.5.4 Room `each_turn`

The current room's `each_turn` is treated specially: it is always
called first, via `DoScopeAction(location)`, before the general scope
search processes other objects. This guarantees that room-level
per-turn effects execute before object-level effects.

```inform6
Object  windy_cliff "Windy Cliff"
  with  description
            "A narrow cliff path. The wind howls around you.",
        each_turn [;
            switch (random(3)) {
                1: "^A gust of wind nearly knocks you off
                    your feet.";
                2: "^The wind shrieks through the crags.";
                3: "^Pebbles skitter along the path.";
            }
        ],
  has   light;
```

### §23.5.5 Example: A Grandfather Clock

A clock that chimes at regular intervals while the player is present:

```inform6
Object  grandfather_clock "grandfather clock" hallway
  with  name 'grandfather' 'clock' 'pendulum',
        each_turn [;
            if (turns % 5 == 0)
                "^The grandfather clock chimes the quarter hour.";
            if (random(6) == 1)
                "^The pendulum swings with a steady tick-tock.";
        ],
        description
            "A tall grandfather clock with a brass pendulum.",
  has   static scenery;
```

## §23.6 Timed Games and the Status Line

The library supports two status line modes: **score/turns** and
**time of day**. By default, the status line shows score and turns.
Games can switch to time-based display using the `Statusline`
directive.

### §23.6.1 Enabling Time Display

To display time on the status line instead of score and turns, include
the following directive in the game source:

```inform6
Statusline time;
```

This sets `sys_statusline_flag` to `1`, causing `DisplayStatus` to
show hours and minutes on the status line.

### §23.6.2 The `the_time` Global

The `the_time` global (parser.h, line 260) holds the current game time
as minutes since midnight:

```inform6
Global the_time = NULL;
```

When `the_time` is `NULL` (the default), time-based processing is
inactive. The `Statusline time` directive sets `the_time` to `0`
(midnight). Game code can set `the_time` directly to establish the
starting time:

```inform6
[ Initialise;
    location = bedroom;
    the_time = 8 * 60 + 30;   ! 8:30 AM = 510 minutes
];
```

Time values are always in minutes since midnight and wrap at 1440
(24 hours). The `AdvanceWorldClock` routine handles the wrapping:

```inform6
the_time = the_time % 1440;
```

### §23.6.3 The `time_rate` Global

The `time_rate` global (parser.h, line 261) controls how quickly game
time advances:

```inform6
Global time_rate = 1;
```

The interpretation of `time_rate` depends on its sign:

| `time_rate` value | Meaning |
|-------------------|---------|
| Positive *n*      | Each turn advances time by *n* minutes |
| `0`               | Time does not advance automatically |
| Negative *-n*     | Time advances 1 minute every *n* turns |

Examples:

- `time_rate = 1;` — default: one minute per turn.
- `time_rate = 5;` — five minutes per turn (fast-paced game).
- `time_rate = 0;` — time stands still unless manually advanced.
- `time_rate = -3;` — one minute every three turns (slow-paced game).

### §23.6.4 The `AdvanceWorldClock` Routine

The complete implementation (parser.h, lines 5436–5449):

```inform6
[ AdvanceWorldClock;
    turns++;
    if (the_time ~= NULL) {
        if (time_rate >= 0) the_time = the_time + time_rate;
        else {
            time_step--;
            if (time_step == 0) {
                the_time++;
                time_step = -time_rate;
            }
        }
        the_time = the_time % 1440;
    }
];
```

For negative `time_rate`, the library uses the `time_step` auxiliary
global as a sub-turn counter. When `time_step` reaches zero, one
minute is added and `time_step` is reset to `-time_rate`. This
provides fractional minutes-per-turn timing.

### §23.6.5 The `sline1` and `sline2` Globals

The `sline1` and `sline2` globals (parser.h, lines 190–191) hold the
values displayed on the right side of the status line:

```inform6
Global sline1;    ! Score or hours
Global sline2;    ! Turns or minutes
```

These are set by `DisplayStatus`:

- In **score/turns mode** (`sys_statusline_flag == 0`): `sline1 =
  score`, `sline2 = turns`.
- In **time mode** (`sys_statusline_flag == 1`): `sline1 =
  the_time/60` (hours), `sline2 = the_time%60` (minutes).

Game code can override `DisplayStatus` to customise the status line
(see §26 for replacing library routines).

### §23.6.6 Example: A Timed Game

A game where time matters and different events happen at different
times of day:

```inform6
Statusline time;

[ Initialise;
    location = town_square;
    the_time = 9 * 60;           ! 9:00 AM
    time_rate = 5;               ! 5 minutes per turn
    "^The church clock strikes nine.";
];

Object  town_square "Town Square"
  with  description [;
            print "The town square is ";
            if (the_time >= 6*60 && the_time < 18*60)
                print "bathed in sunlight";
            else
                print "shrouded in darkness";
            ".";
        ],
        each_turn [;
            if (the_time >= 12*60 && the_time < 12*60 + time_rate)
                "^The church bell tolls noon.";
            if (the_time >= 18*60 && the_time < 18*60 + time_rate)
                "^The sun sets. Shadows lengthen across the
                 square.";
        ],
  has   light;
```

### §23.6.7 Manually Advancing Time

When `time_rate` is `0`, game code must advance time explicitly:

```inform6
time_rate = 0;   ! Disable automatic advancement

! In an action handler:
the_time = the_time + 30;
the_time = the_time % 1440;
"Half an hour passes while you work.";
```

This gives the game full control over how much time each action
consumes. Different actions can advance time by different amounts.

## §23.7 The `TimePasses` Entry Point

`TimePasses` is a game-author **entry point** — a routine that the
library calls if the game defines it. It is called once per turn during
the end-of-turn sequence, after `RunTimersAndDaemons` and
`RunEachTurnProperties` have completed.

### §23.7.1 Stub Definition

The library provides a stub in `grammar.h` (line 566):

```inform6
Stub TimePasses 0;
```

The `Stub` directive defines a do-nothing version of `TimePasses` that
takes no arguments and returns `0`. If the game defines its own
`TimePasses`, the game's version replaces the stub.

### §23.7.2 Usage

`TimePasses` is called in the end-of-turn sequence as follows:

```inform6
if (TimePasses() == 0)
    LibraryExtensions.RunAll(ext_timepasses);
```

If `TimePasses` returns `0` (or `false`), the library additionally runs
any registered library extensions via `ext_timepasses`. If it returns a
non-zero value, the extensions are skipped. This allows `TimePasses` to
suppress extension processing when needed.

### §23.7.3 Example

A `TimePasses` routine that tracks hunger:

```inform6
Global hunger_level = 0;

[ TimePasses;
    hunger_level++;
    switch (hunger_level) {
        10: "^You are getting hungry.";
        20: "^You are very hungry. You should eat something.";
        30: "^You are starving!";
        40: deadflag = 1;
            "^You collapse from hunger.";
    }
    rfalse;
];
```

Note the `rfalse` at the end — this returns `0`, allowing library
extensions to also run their `ext_timepasses` hooks. If the hunger
system should be the exclusive end-of-turn handler, return `true`
instead.

### §23.7.4 `TimePasses` vs. Daemons vs. `each_turn`

| Feature           | `TimePasses`          | Daemon                | `each_turn`            |
|-------------------|-----------------------|-----------------------|------------------------|
| Scope required    | No                    | No                    | Yes                    |
| Per-object        | No (global)           | Yes                   | Yes                    |
| Start/stop        | Always runs           | `StartDaemon`/`StopDaemon` | Automatic (scope) |
| Number allowed    | One                   | Up to `MAX_TIMERS`    | Unlimited              |
| Firing order      | After timers/daemons  | Array order           | Room first, then scope |

Use `TimePasses` for global game-state changes that are not tied to a
specific object. Use daemons for per-object recurring behaviour that
must fire regardless of scope. Use `each_turn` for per-object behaviour
that should only fire when the player can observe the object.

## §23.8 Interaction Between Mechanisms

The library's time-processing mechanisms interact in specific, defined
ways. Understanding these interactions is essential for building
reliable turn-based systems.

### §23.8.1 Order of Processing

The end-of-turn sequence (§23.2) defines a strict order:

1. `AdvanceWorldClock()` — turn counter and game clock.
2. `RunTimersAndDaemons()` — all daemons and timers.
3. `RunEachTurnProperties()` — `each_turn` on in-scope objects.
4. `TimePasses()` — global entry point.
5. `AdjustLight()` — lighting recalculation.
6. `NoteObjectAcquisitions()` — score tracking.

This means:

- Daemons and timers fire **before** `each_turn` routines.
- If a daemon sets `deadflag`, `each_turn` routines are skipped.
- `TimePasses` runs **after** both daemons and `each_turn`.
- The turn counter is already incremented when daemons fire.

### §23.8.2 Within `RunTimersAndDaemons`

Daemons and timers are processed in `the_timers` array order. The
array order is determined by insertion order (the order of
`StartDaemon` and `StartTimer` calls), subject to slot reuse:

```inform6
! If started in this order:
StartDaemon(guard);     ! Slot 0
StartTimer(bomb, 5);    ! Slot 1
StartDaemon(cat);       ! Slot 2
```

The guard's daemon fires first, then the bomb's timer is processed,
then the cat's daemon fires. If the guard's daemon is later stopped
and a new daemon started, the new daemon may reuse slot 0 and fire
first.

Within the processing loop, `deadflag` is checked before each entry:

```inform6
for (i=0 : i<active_timers : i++) {
    if (deadflag) return;
    j = the_timers-->i;
    ...
}
```

If the guard's daemon sets `deadflag = 1`, neither the bomb's timer
nor the cat's daemon will fire on that turn.

### §23.8.3 Scope Changes During Processing

If a daemon or timer moves the player to a different room (or moves
objects in and out of scope), this affects which `each_turn` routines
fire later in the end-of-turn sequence. The scope is calculated at the
time `RunEachTurnProperties` runs, not at the start of the end-of-turn
sequence.

```inform6
Object  teleporter "teleporter" lab
  with  daemon [;
            if (turns % 10 == 0) {
                PlayerTo(garden, 1);
                print "^The teleporter activates!^";
            }
        ];
```

If this daemon teleports the player from the lab to the garden, any
`each_turn` properties on objects in the lab will not fire (they are no
longer in scope), while `each_turn` properties on objects in the garden
will fire.

### §23.8.4 Restarting an Active Timer

Because `StartTimer` checks for an existing entry and returns `false`
if the timer is already active, you cannot change a running timer's
countdown by calling `StartTimer` again. You must stop the timer first:

```inform6
! Wrong — does nothing if timer is already running:
StartTimer(bomb, 10);

! Correct — stop first, then restart:
StopTimer(bomb);
StartTimer(bomb, 10);
```

This is a common source of bugs. If a timer should be restartable, the
stop-then-start pattern must be used consistently.

### §23.8.5 Running Both a Daemon and a Timer on the Same Object

An object can have both a `daemon` property and a `time_out`/`time_left`
pair. However, they are independent registrations in `the_timers`:

```inform6
Object  candle "candle" cellar
  with  name 'candle',
        daemon [;
            if (self in location && random(3) == 1)
                "^The candle flame flickers.";
        ],
        time_out [;
            give self ~light;
            StopDaemon(self);
            if (self in location)
                "^The candle sputters and goes out.";
        ],
        time_left 0,
  has   light;

[ LightCandle;
    StartDaemon(candle);
    StartTimer(candle, 20);
    give candle light;
    "You light the candle.";
];
```

Here the daemon provides ambient flicker messages while the timer
controls the candle's lifespan. When the timer expires, `time_out`
stops the daemon and removes the `light` attribute. The daemon and
timer occupy two separate slots in `the_timers` and count toward
`MAX_TIMERS` independently.

### §23.8.6 `each_turn` on Floating Objects

Objects with `found_in` (see §18.6) may be present in multiple
locations. Their `each_turn` routines fire in every room where they
are in scope:

```inform6
Object  background_music "background music"
  with  found_in [; rtrue; ],
        each_turn [;
            if (random(5) == 1)
                "^You hear faint music in the distance.";
        ],
  has   scenery;
```

This object is in scope everywhere (its `found_in` always returns
`true`), so its `each_turn` fires every turn regardless of location.

### §23.8.7 Daemons and `each_turn` Together

An object can have both a `daemon` and an `each_turn` property. The
daemon fires every turn (when started) regardless of scope, and
`each_turn` fires only when in scope. Both fire on the same turn when
the object is in scope and the daemon is active:

```inform6
Object  robot "cleaning robot" workshop
  with  name 'cleaning' 'robot',
        daemon [;
            ! Internal logic: always runs
            self.charge--;
            if (self.charge <= 0) {
                StopDaemon(self);
                give self ~animate;
            }
        ],
        each_turn [;
            ! Visible feedback: only when in scope
            if (random(3) == 1)
                "^The robot whirs past, polishing the floor.";
        ],
        charge 100,
        description "A small cleaning robot.",
  has   animate;
```

In this design, the daemon handles the robot's internal battery
depletion (always runs), while `each_turn` provides atmosphere text
(only when the player can see the robot).

### §23.8.8 Debug Support

When compiling with the `DEBUG` flag, the `TIMERS` debug verb provides
visibility into active timers and daemons. Setting `DEBUG_TIMERS` in
`debug_flag` causes `RunTimersAndDaemons` to print the state of each
entry before processing:

```
> TIMERS
candle: daemon
bomb: timer with 3 turns to go
egg_timer: timer with 0 turns to go
```

This output is generated by the debug block at the top of
`RunTimersAndDaemons`, which iterates over `the_timers` and prints
each non-zero entry, distinguishing daemons (entries with
`WORD_HIGHBIT`) from timers (entries without).

### §23.8.9 Summary of Limits and Defaults

| Constant / Global  | Default Value | Purpose                                      |
|---------------------|---------------|----------------------------------------------|
| `MAX_TIMERS`        | `32`          | Maximum concurrent timers + daemons          |
| `START_MOVE`        | `0`           | Initial value of `turns`                     |
| `turns`             | `START_MOVE`  | Current turn number                          |
| `the_time`          | `NULL`        | Minutes since midnight (`NULL` = no clock)   |
| `time_rate`         | `1`           | Minutes per turn (or negative: turns/minute) |
| `time_step`         | `0`           | Sub-turn counter for negative `time_rate`    |
| `active_timers`     | `0`           | High-water mark in `the_timers` array        |
| `sline1`            | `0`           | Status line right-side value 1               |
| `sline2`            | `0`           | Status line right-side value 2               |
| `WORD_HIGHBIT` (Z)  | `$8000`       | High bit for 16-bit words                    |
| `WORD_HIGHBIT` (G)  | `$80000000`   | High bit for 32-bit words                    |
| `EACH_TURN_REASON`  | `2`           | Scope reason constant for `each_turn`        |
