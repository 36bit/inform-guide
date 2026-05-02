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

# Chapter 24: Timers, Daemons, and Scheduling

The standard library provides three complementary mechanisms for
executing code at regular intervals or after a delay: **daemons**,
which run every turn until stopped; **timers**, which count down a
specified number of turns and then fire; and the **`each_turn`
property**, which runs every turn for objects currently in scope. All
three are processed during the end-of-turn sequence described in §16,
after the player's action has been resolved. This chapter provides a
complete reference for all three mechanisms, including the underlying
data structures, the turn-end execution order, and the real-time clock
used for time-of-day games.

## 24.1 Overview of Scheduled Events

Interactive fiction proceeds in discrete turns. After each player
command is parsed and executed, the library runs a sequence of
end-of-turn tasks. Three of those tasks invoke author-supplied code:

| Mechanism    | Property    | Activation            | Duration             |
|------------- |------------ |---------------------- |--------------------- |
| Daemon       | `daemon`    | `StartDaemon(obj)`    | Until `StopDaemon(obj)` |
| Timer        | `time_out`  | `StartTimer(obj, n)`  | Fires once after *n* turns |
| Each-turn    | `each_turn` | Automatic (in scope)  | Every turn while in scope |

Daemons and timers share a common activation pool and are managed by
four library routines. The `each_turn` property, by contrast, requires
no explicit activation — it runs automatically for every object
currently in scope.

## 24.2 The Timer/Daemon Pool

### 24.2.1 `the_timers` Array

Daemons and timers are tracked in a single word array called
`the_timers`, declared in `parser.h`:

```inform6
#Ifndef MAX_TIMERS;
Constant MAX_TIMERS  32;
#Endif;
Array  the_timers  --> MAX_TIMERS;
```

Each slot in `the_timers` holds either `0` (empty), an object
reference (for a timer), or an object reference with the high bit set
(for a daemon). The global variable `active_timers` records the
high-water mark — the number of slots that have been used, which
determines how far the library scans when processing the array.

The maximum number of concurrently active timers and daemons is
`MAX_TIMERS`, which defaults to 32. A game that needs more can define
a larger value before including the library:

```inform6
Constant MAX_TIMERS 64;
Include "Parser";
```

If a `StartDaemon` or `StartTimer` call would exceed the pool size,
the library raises runtime error 4 ("Too many timers").

### 24.2.2 The `WORD_HIGHBIT` Constant

Daemons and timers are distinguished in the pool by a single bit. For
each daemon entry, the library stores `WORD_HIGHBIT + obj` rather than
`obj` alone. The constant is platform-specific:

**[Z-machine]** `WORD_HIGHBIT` is `$8000` (bit 15 of a 16-bit word).

**[Glulx]** `WORD_HIGHBIT` is `$80000000` (bit 31 of a 32-bit word).

The library tests `the_timers-->i & WORD_HIGHBIT` (a bitwise AND) to
distinguish daemon entries from timer entries, and recovers the object
with `the_timers-->i & ~WORD_HIGHBIT` (masking off the high bit).

## 24.3 Daemons

A **daemon** is a background process attached to an object. Once
started, the object's `daemon` property routine is called every turn
until the daemon is explicitly stopped. Daemons are useful for
continuous processes: a ticking clock, a wandering NPC, a slowly
filling room, or any event that recurs indefinitely.

### 24.3.1 The `daemon` Property

An object that will use a daemon must provide a `daemon` property
containing a routine:

```inform6
Object  old_clock "grandfather clock" hallway
  with  name 'grandfather' 'clock' 'old',
        description "A tall grandfather clock with a swinging pendulum.",
        daemon [;
            if (self in location)
                print "^The grandfather clock ticks solemnly.^";
        ];
```

The `daemon` property is declared in `linklpa.h` as a common property.
Its default value is `0` (no daemon). The property routine receives no
arguments and its return value is ignored.

### 24.3.2 `StartDaemon(obj)`

Activates the daemon for `obj`. The object is added to the
`the_timers` pool with the `WORD_HIGHBIT` flag set. If the daemon is
already running, the call returns `false` and has no effect.

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

The routine first scans for a duplicate entry. If none is found, it
searches for an empty slot (a `0` entry). If no empty slot exists, it
extends the high-water mark. If that would exceed `MAX_TIMERS`,
runtime error 4 is raised.

Usage:

```inform6
StartDaemon(old_clock);
```

### 24.3.3 `StopDaemon(obj)`

Deactivates the daemon for `obj` by clearing its slot in the
`the_timers` array. If the daemon is not currently running, the call
returns `false`.

```inform6
[ StopDaemon obj i;
    for (i=0 : i<active_timers : i++)
        if (the_timers-->i == WORD_HIGHBIT + obj) jump FoundTSlot4;
    rfalse;
  .FoundTSlot4;
    the_timers-->i = 0;
];
```

Usage:

```inform6
StopDaemon(old_clock);
```

### 24.3.4 Daemon Example

A wandering NPC that moves to a random adjacent room each turn:

```inform6
Object  guard "palace guard" throne_room
  with  name 'palace' 'guard',
        description "A stern-looking guard in ceremonial armour.",
        daemon [ dest;
            dest = random(parent(self).n_to, parent(self).s_to,
                          parent(self).e_to, parent(self).w_to);
            while (dest == 0 or nothing)
                dest = random(parent(self).n_to, parent(self).s_to,
                              parent(self).e_to, parent(self).w_to);
            move self to dest;
            if (dest == location)
                print "^The palace guard marches in.^";
            if (parent(self) == location && dest ~= location)
                print "^The palace guard marches away.^";
        ],
  has   animate;

! In Initialise():
[ Initialise;
    location = throne_room;
    StartDaemon(guard);
];
```

## 24.4 Timers

A **timer** is a countdown attached to an object. When started, the
object's `time_left` property is set to a count. Each turn, the
library decrements `time_left` by one. When it reaches zero, the
object's `time_out` property routine is called and the timer is
automatically removed from the pool.

### 24.4.1 Timer Properties

Two properties govern timers:

- **`time_left`** — a numeric property that holds the number of turns
  remaining. The library decrements it each turn. The object *must*
  provide this property (the library checks with `.&time_left`); if it
  does not, runtime error 5 is raised.

- **`time_out`** — a property routine (declared as additive) that is
  called when `time_left` reaches zero. This is the timer's "fire"
  event.

```inform6
Object  bomb "ticking bomb" cellar
  with  name 'ticking' 'bomb',
        description [;
            print "A bomb with a fuse. ";
            if (self.time_left > 0)
                print "It will go off in ", self.time_left, " turns.^";
            else
                "It looks inert.";
        ],
        time_left 0,
        time_out [;
            deadflag = 1;
            "^The bomb explodes with a deafening roar!";
        ];
```

### 24.4.2 `StartTimer(obj, turns)`

Activates a timer on `obj`, setting its `time_left` property to
`turns`. The object is added to the `the_timers` pool. If the timer is
already running, the call returns `false` and has no effect.

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

The routine validates that `obj` provides a `time_left` property by
checking `obj.&time_left`. If the object lacks this property, runtime
error 5 is raised and the timer is not started.

Usage:

```inform6
StartTimer(bomb, 10);   ! Bomb explodes in 10 turns
```

### 24.4.3 `StopTimer(obj)`

Deactivates the timer for `obj`, clearing its slot in `the_timers` and
resetting `time_left` to zero. If the timer is not running, the call
returns `false`.

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

Usage:

```inform6
! Player defuses the bomb:
before [;
    Disarm:
        if (noun == bomb && bomb.time_left > 0) {
            StopTimer(bomb);
            "You carefully cut the fuse. The bomb is disarmed.";
        }
],
```

### 24.4.4 Restarting a Timer

Because `StartTimer` returns `false` if the timer is already running, a
timer cannot be restarted without first stopping it. To reset a running
timer to a new countdown:

```inform6
StopTimer(bomb);
StartTimer(bomb, 5);   ! Reset to 5 turns
```

Alternatively, you can directly modify the `time_left` property of a
running timer without stopping and restarting it:

```inform6
bomb.time_left = 5;    ! Reset countdown without removing from pool
```

This works because the library simply decrements `time_left` each turn;
it does not read the value from the pool.

### 24.4.5 Timer Example

A lantern with limited fuel:

```inform6
Object  lantern "brass lantern" startroom
  with  name 'brass' 'lantern' 'lamp',
        description "A well-polished brass lantern.",
        time_left 0,
        time_out [;
            give self ~light;
            give self ~on;
            StopDaemon(self);
            if (self in location || IndirectlyContains(player, self))
                "^Your lantern flickers and goes out!";
        ],
        before [;
            SwitchOn:
                StartTimer(self, 200);  ! 200 turns of fuel
                give self light;
            SwitchOff:
                StopTimer(self);
                give self ~light;
        ],
  has   switchable;
```

## 24.5 The `each_turn` Property

The `each_turn` property provides a simpler form of per-turn
processing that does not require explicit activation or deactivation.
Every turn, the library calls the `each_turn` routine of every object
currently in **scope** (see §25 for the full definition of scope). If
the object is out of scope — for example, in a different room — its
`each_turn` routine is not called.

### 24.5.1 Declaration

The `each_turn` property is declared additive in `linklpa.h`, meaning
that when a class defines `each_turn` and an instance also defines it,
both routines run. The property can be either a routine or a string:

```inform6
Object  dripping_tap "dripping tap" bathroom
  with  name 'dripping' 'tap' 'faucet',
        each_turn [;
            if (random(3) == 1)
                "^Drip... drip... drip...";
        ];
```

If `each_turn` is a string, the string is printed (with a newline)
each turn.

### 24.5.2 Scope Dependency

The crucial distinction between `each_turn` and daemons is scope
dependency. A daemon runs every turn regardless of where the object is
in the game world. An `each_turn` routine runs only when the object is
in the player's scope — typically, when the object is in the same room
as the player (or in a transparent or open container accessible from
the player's location).

This makes `each_turn` ideal for ambient effects: sounds, smells,
visual descriptions that should only occur when the player is present.
Use daemons for processes that must continue regardless of the
player's location.

### 24.5.3 Execution Order

The `each_turn` routines are called by `RunEachTurnProperties()`, which
uses the scope mechanism with `EACH_TURN_REASON` to iterate over all
in-scope objects. The order is determined by the scope-scanning
algorithm (see §25.1), which traverses the object tree depth-first
starting from the player's location and the player's inventory. Within
a single container, children are processed in object-number order.

### 24.5.4 `each_turn` on Rooms

The player's current location (i.e., the room object) can also have an
`each_turn` property. The library calls the location's `each_turn`
routine each turn, just as it would for any other in-scope object.
This provides a convenient way to implement per-room ambient effects:

```inform6
Object  forest "Dark Forest"
  with  description "Tall pines block most of the light.",
        each_turn [;
            switch (random(4)) {
                1: "^An owl hoots in the distance.";
                2: "^A branch snaps somewhere nearby.";
                3: "^The wind whispers through the trees.";
            }
        ],
  has   light;
```

## 24.6 The End-of-Turn Sequence

After the player's action has been fully processed (including `before`,
the action routine, and `after` — see §20), the library executes the
**end-of-turn sequence**. This is defined as a property of the
`InformLibrary` object in `parser.h`:

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
],
```

The six steps, in order:

### 24.6.1 Step 1: `AdvanceWorldClock()`

Increments the turn counter and, for time-of-day games, advances the
game clock (`the_time`) by the configured rate. See §24.7 for details
on the time-of-day system.

### 24.6.2 Step 2: `RunTimersAndDaemons()`

Scans the `the_timers` array from slot 0 to `active_timers - 1`.
For each non-zero entry:

- If the entry has `WORD_HIGHBIT` set, it is a **daemon**. The library
  calls `RunRoutines(obj, daemon)`, which invokes the object's
  `daemon` property routine.

- Otherwise, it is a **timer**. The library decrements `obj.time_left`.
  If `time_left` reaches zero, the library calls
  `RunRoutines(obj, time_out)` and clears the slot (effectively
  calling `StopTimer` automatically).

If any daemon or timer sets `deadflag` to a non-zero value, the
sequence aborts immediately and the death/victory sequence begins.

**Processing order:** Timers and daemons are processed in the order
they appear in the `the_timers` array, which corresponds to the order
in which they were started (subject to slot reuse when earlier slots
are freed by stopping).

### 24.6.3 Step 3: `RunEachTurnProperties()`

Calls the `each_turn` property of every object currently in scope,
using the scope mechanism with `EACH_TURN_REASON`. This routine
invokes `DoScopeAction` for each in-scope object, which in turn calls
`RunRoutines(obj, each_turn)` if the property exists.

If any `each_turn` routine sets `deadflag`, the sequence aborts.

### 24.6.4 Step 4: `TimePasses()`

Calls the game author's `TimePasses()` entry point, if defined (see
§26.16). If `TimePasses()` returns `false` (or is not defined), the
library also calls any registered library extension routines. This
entry point is the last opportunity for per-turn processing before
the library adjusts lighting.

### 24.6.5 Step 5: `AdjustLight()`

Recalculates whether the player's location is lit or dark, and
transitions between light and darkness as necessary. See §25.4 for
details on the light-propagation algorithm.

### 24.6.6 Step 6: `NoteObjectAcquisitions()`

Sets the `moved` attribute on any objects in the player's possession
that have not yet been marked as moved. For each such object that
also carries the `scored` attribute, `OBJECT_SCORE` is added to both
`score` and `things_score` — this is how the library awards points
the first time a treasure is picked up.

### 24.6.7 `deadflag` Checks

Each step is followed by a check of the global variable `deadflag`. If
`deadflag` is non-zero (indicating the player has died, won, or
reached another terminal state), the remaining steps are skipped and
the library proceeds to the death/victory sequence.

## 24.7 The Time-of-Day System

The library includes a time-of-day clock for games that track the
passage of hours and minutes rather than (or in addition to) a simple
turn counter. This is primarily used for displaying the time on the
status line.

### 24.7.1 Enabling the Clock

To enable the time-of-day display, the source file must include:

```inform6
Statusline time;
```

This causes the status line to display the current time instead of the
score and turn count. The clock is controlled by three global
variables:

| Variable    | Purpose                                              |
|------------ |----------------------------------------------------- |
| `the_time`  | Current time in minutes since midnight (0–1439).     |
| `time_rate` | Minutes advanced per turn (positive or negative).    |
| `time_step` | Internal fractional accumulator (negative rates).    |

### 24.7.2 `SetTime(time, rate)`

Initializes the game clock. `time` is the starting time in minutes
since midnight (e.g., `540` for 9:00 AM). `rate` is the number of
minutes that pass per turn.

```inform6
[ Initialise;
    location = bedroom;
    SetTime(9*60, 1);   ! Start at 9:00 AM, 1 minute per turn
];
```

If `rate` is positive, the clock advances by `rate` minutes every
turn. If `rate` is negative, the clock advances by 1 minute every
`|rate|` turns — useful for slower-paced games.

The implementation in `parser.h`:

```inform6
[ SetTime t s;
    the_time = t; time_rate = s; time_step = 0;
    if (s < 0) time_step = 0-s;
];
```

### 24.7.3 Clock Advancement

Each turn, `AdvanceWorldClock()` advances the clock according to
`time_rate`:

- If `time_rate >= 0`: `the_time` is incremented by `time_rate`
  minutes. If `the_time` reaches or exceeds 1440 (midnight), it wraps
  around. (When `time_rate` is zero, the clock does not advance
  automatically, but the game can still modify `the_time` directly or
  call `SetTime`.)

- If `time_rate < 0`: the internal `time_step` counter is decremented.
  When `time_step` reaches zero, `the_time` is incremented by 1 minute
  and `time_step` is reset to `|time_rate|`. This effectively advances
  the clock by one minute every `|time_rate|` turns.

- If `time_rate == 0`: this falls into the `>= 0` branch above, so
  zero is added and the clock does not advance. The game can still
  modify `the_time` directly or call `SetTime`.

### 24.7.4 Displaying the Time

**[Z-machine]** When `Statusline time` is in effect, the Z-machine
interpreter displays the time on the status line using the values in
`sline1` (hours) and `sline2` (minutes). The library updates these
values in `DisplayStatus()`:

```inform6
sline1 = the_time / 60;
sline2 = the_time % 60;
```

The interpreter itself handles formatting (12-hour vs. 24-hour display
is interpreter-dependent).

**[Glulx]** The time is formatted by `DrawStatusLine()` in the library.
The library prints the hours and minutes directly.

### 24.7.5 Real-Time Events

**[Z-machine]** The Z-machine specification (versions 4+) supports a
**real-time input** feature. The interpreter can be instructed to
interrupt keyboard input after a specified number of tenths of a second
and call a callback routine. This is implemented with the timed variant
of the `read` (or `aread`) opcode:

```
@aread buffer parse time routine -> result;
```

Here `time` is the timeout in tenths of a second and `routine` is a
packed address of a routine to call when the timeout expires. If the
routine returns `true`, input is terminated as if the player had pressed
Enter.

The standard library does not provide a high-level interface for
real-time input; using it requires direct assembly language. A game
that wants a timer to fire every 5 seconds during input would embed
a `@aread` call with `time` set to 50 (50 tenths of a second = 5
seconds).

**[Glulx]** Glulx provides timer events through the Glk timer
mechanism. The game calls `glk_request_timer_events(milliseconds)` to
start receiving `evtype_Timer` events at the specified interval, and
`glk_request_timer_events(0)` to stop. Timer events are received
through `glk_select()` and can be handled in the game's event loop.

## 24.8 Interactions Between Mechanisms

### 24.8.1 Daemons vs. Timers

A single object can have both a daemon and a timer active
simultaneously. They occupy separate slots in the `the_timers` array
(distinguished by `WORD_HIGHBIT`). The daemon runs every turn; the
timer counts down independently. When the timer fires, the daemon
continues unless explicitly stopped.

### 24.8.2 Daemons vs. `each_turn`

An object can have both a `daemon` property and an `each_turn`
property. The daemon runs every turn regardless of scope; `each_turn`
runs only when the object is in scope. Both can be active at the same
time. The daemon is processed in step 2 of the end-of-turn sequence;
`each_turn` is processed in step 3.

A common pattern is to use a daemon for global bookkeeping and
`each_turn` for player-visible effects:

```inform6
Object  volcano "Mount Doom" landscape
  with  daemon [;
            self.pressure = self.pressure + random(3);
            if (self.pressure > 100) {
                deadflag = 1;
                "^The volcano erupts catastrophically!";
            }
        ],
        each_turn [;
            if (self.pressure > 80)
                "^The ground trembles beneath your feet.";
            if (self.pressure > 50)
                "^You hear a distant rumbling from the volcano.";
        ],
        pressure 0;
```

### 24.8.3 Stopping a Timer from Its Own `time_out`

It is safe for a `time_out` routine to call `StartTimer` on itself to
create a repeating timer:

```inform6
time_out [;
    print "^The church bells chime the hour.^";
    StartTimer(self, 60);   ! Chime again in 60 turns
],
```

The library clears the timer's slot *before* calling `time_out`, so
`StartTimer` will find the slot empty and reuse it.

### 24.8.4 Stopping a Daemon from Its Own `daemon` Routine

It is safe for a daemon routine to call `StopDaemon(self)`. The library
does not re-process the cleared slot in the same turn.

## 24.9 Debugging Timers and Daemons

### 24.9.1 `TRACE` and the `timers` Option

When the game is compiled in debug mode (`-D`), the player can type
`TRACE` to enable diagnostic output. With timers enabled, the library
prints a message each turn showing which timers and daemons were
processed:

```
[Running timers: 3 active]
[Daemon on old_clock]
[Timer on bomb: 7 turns left]
```

### 24.9.2 Inspecting the Pool

In debug mode, the `the_timers` array can be inspected directly. Each
slot contains either `0` (empty), a bare object number (timer), or
`WORD_HIGHBIT + object_number` (daemon).

### 24.9.3 Common Pitfalls

- **Forgetting `time_left`:** If an object is used with `StartTimer`
  but does not provide a `time_left` property, the library raises
  runtime error 5. Always include `time_left 0` in the object
  definition.

- **Exceeding `MAX_TIMERS`:** If more than `MAX_TIMERS` daemons and
  timers are active simultaneously, `StartDaemon` or `StartTimer`
  raises runtime error 4. Increase `MAX_TIMERS` before including the
  library if needed.

- **Pool fragmentation:** Stopping and starting many timers/daemons
  can leave empty slots in the middle of the array. The library handles
  this correctly (it scans for empty slots when starting a new entry),
  but `active_timers` only grows; it never shrinks. This means the
  scan range may be larger than the number of active entries. In
  practice this is harmless, as the cost of scanning 32 slots is
  negligible.

- **Order dependence:** If two daemons interact (e.g., one sets a
  variable that another reads), their execution order depends on their
  order in the `the_timers` array, which is the order in which they
  were started. Relying on this order is fragile; use explicit state
  management instead.

## 24.10 Summary of Routines and Properties

| Routine / Property       | Purpose                                     |
|------------------------- |-------------------------------------------- |
| `StartDaemon(obj)`       | Activate daemon for `obj`                   |
| `StopDaemon(obj)`        | Deactivate daemon for `obj`                 |
| `StartTimer(obj, turns)` | Start countdown timer on `obj`              |
| `StopTimer(obj)`         | Cancel countdown timer on `obj`             |
| `SetTime(time, rate)`    | Set game clock and advance rate             |
| `daemon` property        | Routine called every turn while active      |
| `time_out` property      | Routine called when timer reaches zero      |
| `time_left` property     | Turns remaining on active timer             |
| `each_turn` property     | Routine called every turn while in scope    |
| `the_timers` array       | Pool of active timers and daemons           |
| `active_timers` global   | High-water mark of pool usage               |
| `MAX_TIMERS` constant    | Maximum pool size (default 32)              |
| `the_time` global        | Game clock in minutes since midnight        |
| `time_rate` global       | Clock advance rate (minutes/turn)           |
