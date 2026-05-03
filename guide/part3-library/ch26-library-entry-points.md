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

# Chapter 26: Library Entry Points and Hooks

A **library entry point** is a routine that the game author can define
to customize the library's behavior at a specific moment during
execution. The library checks whether each entry point exists (using
the `Stub` mechanism or `#Ifdef` tests) and calls it when appropriate.
Entry points provide the primary way to hook into the library's
processing pipeline without modifying the library source code.

This chapter provides a complete reference for every entry point
recognized by library version 6.12.8. Entry points are listed in
rough order of when they are called during a game session: startup,
parsing, action processing, display, turn end, and death/victory.

## 26.1 Entry Point Mechanism

Most entry points are declared with the `Stub` directive in
`grammar.h`, which creates a do-nothing default implementation if the
game does not define the routine:

```inform6
Stub AfterLife         0;
Stub AfterPrompt       0;
Stub Amusing           0;
Stub BeforeParsing     0;
Stub ChooseObjects     2;
Stub DarkToDark        0;
Stub DeathMessage      0;
Stub Epilogue          0;
Stub GamePostRoutine   0;
Stub GamePreRoutine    0;
Stub InScope           1;
Stub LookRoutine       0;
Stub NewRoom           0;
Stub ObjectDoesNotFit  2;
Stub ParseNumber       2;
Stub ParserError       1;
Stub PrintTaskName     1;
Stub PrintVerb         1;
Stub TimePasses        0;
Stub UnknownVerb       1;
Stub AfterSave         1;
Stub AfterRestore      1;

#Ifdef TARGET_GLULX;
Stub HandleGlkEvent    2;
Stub IdentifyGlkObject 4;
Stub InitGlkWindow     1;
#Endif; ! TARGET_GLULX
```

The number after each name is the argument count. `Stub` defines a
function that takes the specified number of arguments and returns
`false`.

Two additional entry points use `#Ifndef` with custom default
implementations rather than `Stub`:

```inform6
#Ifndef PrintRank;
[ PrintRank; "."; ];
#Endif;

#Ifndef ParseNoun;
[ ParseNoun obj; obj = obj; return -1; ];
#Endif;
```

`PrintRank` defaults to printing a full stop after the score line.
`ParseNoun` defaults to returning −1 (meaning "not handled").

The `Initialise` routine is the only **required** entry point; the
game will not compile without it.

## 26.2 `Initialise()`

Called once at the start of the game, before the first command is
read. This is where the game sets up initial conditions.

**When called:** During library startup, before the banner is printed
and before the first `Look`.

**Arguments:** None.

**Return value:**

| Value | Effect                                                    |
|------ |---------------------------------------------------------- |
| 0     | Print the banner, then perform an initial `Look`.         |
| 1     | Print the banner, then perform an initial `Look` (same as 0). |
| 2     | Do not print the banner or perform an initial `Look`.     |

**Typical usage:**

```inform6
[ Initialise;
    location = kitchen;
    move lantern to player;
    "^You wake up in the kitchen. Something smells burnt.^";
];
```

**Notes:**
- The `location` variable *must* be set to a valid room object before
  `Initialise` returns, or the game will crash.
- If `Initialise` prints any text, it appears before the banner.
- A return value of 2 is useful when the game wants to display a
  custom title screen before normal play begins.

## 26.3 `BeforeParsing()`

Called at the start of each turn, *before* the player's input line is
tokenized and parsed.

**When called:** After the prompt is printed and the input line is
read, but before the parser begins analysis.

**Arguments:** None.

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `false` | Library extensions' `ext_beforeparsing` hooks also run.|
| `true`  | Library extensions are skipped.                        |

The parser proceeds regardless of the return value; it controls only
whether `LibraryExtensions` are given a chance to run.

**Typical usage:** Modifying the raw input buffer before parsing.
This is useful for implementing command aliases, stripping
punctuation, or performing custom pre-processing:

```inform6
[ BeforeParsing i;
    ! Replace "x" with "examine" in the input buffer
    for (i=1 : i<=parse->1 : i++)
        if (parse-->(i*2-1) == 'x//')
            SetWord(i, 'examine');
];
```

**Notes:**
- The input buffer (`buffer`) and parse array (`parse`) are available
  for inspection and modification.
- Changes to the buffer or parse array affect all subsequent parsing.

## 26.4 `UnknownVerb(word)`

Called when the parser cannot find a verb matching the first word of
the player's input.

**When called:** After the parser fails to find the word in the verb
grammar tables.

**Arguments:** `word` — the dictionary value of the unrecognized word.

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `false` | The parser prints its standard "I don't know the verb" error. |
| A dictionary word | The parser retries using the returned word as the verb. |

**Typical usage:** Mapping non-standard verbs to standard ones:

```inform6
[ UnknownVerb word;
    if (word == 'steal')  return 'take';
    if (word == 'slay')   return 'attack';
    rfalse;
];
```

## 26.5 `ParserError(etype)`

Called when the parser encounters an error it would normally report
with a library message.

**When called:** After the parser has determined that the input cannot
be successfully parsed.

**Arguments:** `etype` — a constant identifying the type of error:

| Constant               | Value | Meaning                              |
|----------------------- |------ |------------------------------------- |
| `STUCK_PE`             | 1     | Input not recognized at all          |
| `UPTO_PE`              | 2     | Verb recognized, but rest is gibberish |
| `NUMBER_PE`            | 3     | Number out of range                  |
| `CANTSEE_PE`           | 4     | "You can't see any such thing"       |
| `TOOLIT_PE`            | 5     | That noun can only mean one object   |
| `NOTHELD_PE`           | 6     | "You aren't holding that"            |
| `MULTI_PE`             | 7     | Can't use multiple objects here      |
| `MMULTI_PE`            | 8     | Can only use multiple objects once   |
| `VAGUE_PE`             | 9     | "I'm not sure what you are referring to" |
| `EXCEPT_PE`            | 10    | "You excepted something not included" |
| `ANIMA_PE`             | 11    | Can only do that to something animate |
| `VERB_PE`              | 12    | "I don't know that verb"             |
| `SCENERY_PE`           | 13    | "That's not something you need to refer to" |
| `ITGONE_PE`            | 14    | "I can no longer see that"           |
| `JUNKAFTER_PE`         | 15    | Extra words after command            |
| `TOOFEW_PE`            | 16    | "Only N of those are available"      |
| `NOTHING_PE`           | 17    | "Nothing to do"                      |
| `ASKSCOPE_PE`          | 18    | Custom scope routine: error case     |

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `true`  | The error is handled; the parser requests new input.   |
| `false` | The library prints its default error message.          |

**Typical usage:**

```inform6
[ ParserError etype;
    if (etype == CANTSEE_PE) {
        "You peer around but see nothing like that here.";
    }
    rfalse;
];
```

## 26.6 `ChooseObjects(obj, code)`

Called during disambiguation and "all" processing to help the parser
decide which objects to prefer.

**When called:** When the parser is choosing between ambiguous objects
or deciding which objects to include in a "take all" set.

**Arguments:**

- `obj` — the object being considered.
- `code` — the parser's recommendation:
  - `0` = the parser is not recommending this object
  - `1` = the parser recommends this object
  - `2` = the parser is asking for an "all" decision

**Return value:**

| Value | Effect                                                    |
|------ |---------------------------------------------------------- |
| 0     | Accept the parser's recommendation.                       |
| 1     | Force the object to be included (for "all") or preferred. |
| 2     | Force the object to be excluded (for "all") or rejected.  |

For non-"all" disambiguation (code 0 or 1), the return value is a
score from 0 to 9, where higher numbers indicate higher preference.

**Typical usage:**

```inform6
[ ChooseObjects obj code;
    ! Don't include scenery in "take all"
    if (code == 2 && obj has scenery) return 2;
    ! Prefer the lit torch over the unlit one
    if (obj == torch && torch has light) return 5;
    return 0;
];
```

## 26.7 `InScope(actor)`

Called during every scope determination to allow the game to customize
what objects are in scope.

**When called:** During scope computation (see §25.1.3), after any
custom scope token but before the standard scope search.

**Arguments:** `actor` — the person whose scope is being computed
(usually `player`).

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `true`  | Skip the standard scope search entirely.               |
| `false` | Perform the standard scope search in addition.         |

**Typical usage:** See §25.5.1 for detailed examples.

## 26.8 `GamePreRoutine()`

Called before *every* action, before any `before` property is checked.
This is the highest-priority action interceptor.

**When called:** After parsing is complete and the action variables
(`action`, `noun`, `second`) are set, but before any `before` rules
fire.

**Arguments:** None.

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `true`  | The action is stopped; no further processing occurs.   |
| `false` | Processing continues normally.                         |

**Typical usage:** Global rules that apply everywhere:

```inform6
[ GamePreRoutine;
    if (action == ##Attack)
        "Violence isn't the answer.";
    rfalse;
];
```

## 26.9 `GamePostRoutine()`

Called after an action has been fully processed, including `after`
properties and `react_after` checks. This is the lowest-priority
post-action hook.

**When called:** After the action routine has run and all `after` and
`react_after` properties have been checked. Only called if no earlier
handler returned `true`.

**Arguments:** None.

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `true`  | Suppress any remaining post-action processing.         |
| `false` | Default behavior (no additional effect).               |

**Typical usage:**

```inform6
[ GamePostRoutine;
    if (action == ##Take && noun == golden_idol)
        "The temple begins to rumble ominously...";
    rfalse;
];
```

## 26.10 `DarkToDark()`

Called when the player moves from one dark location to another dark
location (or stays in darkness after an action that might have changed
the lighting).

**When called:** During location transitions when both the previous
and new locations are dark.

**Arguments:** None.

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `false` | Library extensions' `ext_darktodark` hooks also run.   |
| `true`  | Library extensions are skipped.                        |

**Typical usage:** Warning the player of danger:

```inform6
[ DarkToDark;
    "You stumble in the darkness. Be careful — there may be a pit nearby!";
];
```

**Notes:** The library calls `DarkToDark()` before printing the
"It is pitch dark" message when the player arrives at a dark
location. A classic use is to kill the player after a certain number
of moves in the dark.

## 26.11 `NewRoom()`

Called when the player enters a new room.

**When called:** After the player has been moved to a new location
and floating objects have been repositioned, but before the room
description is printed.

**Arguments:** None.

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `false` | Library extensions' `ext_newroom` hooks also run.      |
| `true`  | Library extensions are skipped.                        |

**Typical usage:** Triggering events on entering specific rooms:

```inform6
[ NewRoom;
    if (location == treasure_room && treasure_room hasnt visited)
        "You gasp at the sight of so much gold!";
];
```

**Notes:** `NewRoom()` is called by `PlayerTo()` and the `Go` action.
It is not called during the initial `Look` at game startup.

## 26.12 `DeathMessage()`

Called to print a custom death or victory message when `deadflag` is
set to a value of 3 or higher.

**When called:** During the death/victory sequence, when `deadflag`
holds a value other than 0, 1, or 2. The library handles `deadflag`
values 1 ("You have died") and 2 ("You have won") automatically. For
values 3 and above, the library calls `DeathMessage()`.

**Arguments:** None (the routine should check `deadflag` itself).

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `false` | Library extensions' `ext_deathmessage` hooks also run. |
| `true`  | Library extensions are skipped.                        |

**Typical usage:**

```inform6
Constant ASLEEP = 3;
Constant ARRESTED = 4;

[ DeathMessage;
    switch (deadflag) {
        ASLEEP:   print "You have fallen asleep";
        ARRESTED: print "You have been arrested";
    }
];
```

**Notes:** The routine should *print* the message without a trailing
newline (the library adds formatting around it). The standard form is
"You have died" / "You have won" — custom messages conventionally
follow the pattern "You have ..." for consistency.

## 26.13 `AfterLife()`

Called after the game loop exits due to `deadflag` being set, giving
the game a chance to resurrect the player or modify the outcome.

**When called:** After `deadflag` has been set (to any non-zero value
other than 2), but before the death message is printed. When
`deadflag` is 2 (victory), `AfterLife` is not called.

**Arguments:** None.

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `true`  | Library extensions are skipped. If `deadflag` has been reset to 0, the game continues. |
| `false` | Library extensions' `ext_afterlife` hooks also run. If `deadflag` is still non-zero, the death/victory sequence proceeds. |

**Typical usage:** Giving the player another chance:

```inform6
[ AfterLife;
    if (deadflag == 1 && lives_remaining > 0) {
        lives_remaining--;
        deadflag = 0;
        PlayerTo(hospital);
        "^You wake up in a hospital bed. You have ",
            lives_remaining, " lives remaining.";
    }
];
```

## 26.14 `AfterPrompt()`

Called after the command prompt has been printed, but before the
library reads the player's input.

**When called:** After the `>` prompt (or custom prompt) is displayed,
immediately before the keyboard input routine is called.

**Arguments:** None.

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `false` | Library extensions' `ext_afterprompt` hooks also run.  |
| `true`  | Library extensions are skipped.                        |

**Typical usage:** Displaying extra information at the prompt, or
modifying the prompt itself:

```inform6
[ AfterPrompt;
    print "(Health: ", player_health, ") ";
];
```

## 26.15 `PrintVerb(word)`

Called when the library needs to print a verb word (e.g., during
command replay or `OOPS` correction).

**When called:** When the library is about to print a verb and wants
to allow the game to override the default printing.

**Arguments:** `word` — the dictionary word value of the verb.

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `true`  | The game has printed the verb; the library does nothing more. |
| `false` | The library prints the verb using its default method.  |

## 26.16 `TimePasses()`

Called once per turn during the end-of-turn sequence, after daemons,
timers, and `each_turn` routines have run, but before lighting is
recalculated.

**When called:** Step 4 of the end-of-turn sequence (see §24.6.4).

**Arguments:** None.

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `false` | Library extensions' `ext_timepasses` hooks also run.   |
| `true`  | Library extensions are skipped.                        |

The main turn sequence proceeds regardless of the return value.

**Typical usage:** Per-turn global events:

```inform6
[ TimePasses;
    if (hunger > 0) {
        hunger--;
        if (hunger == 0) {
            deadflag = 1;
            "^You collapse from hunger!";
        }
        if (hunger < 10) "^Your stomach growls.";
    }
];
```

## 26.17 `LookRoutine()`

Called after the `Look` action has completed its output.

**When called:** After the room name, description, and object list
have been printed by `LookSub`.

**Arguments:** None.

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `false` | Library extensions' `ext_lookroutine` hooks also run.  |
| `true`  | Library extensions are skipped.                        |

**Typical usage:** Appending extra information to the room
description:

```inform6
[ LookRoutine;
    if (location == garden && rain_falling)
        "^Rain is falling steadily.";
];
```

## 26.18 `PrintTaskName(task_number)`

Called when the library prints the list of completed tasks for the
`FULL SCORE` command.

**When called:** During `FullScoreSub`, for each completed task.

**Arguments:** `task_number` — the index of the completed task (0-based).

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `true`  | The game has printed the task name.                    |
| `false` | The library uses its default output (if any).          |

**Typical usage:**

```inform6
[ PrintTaskName n;
    switch (n) {
        0: "finding the hidden key";
        1: "opening the ancient door";
        2: "defeating the dragon";
    }
];
```

## 26.19 `ObjectDoesNotFit(obj, container)`

Called when the player tries to put an object into a container that is
at capacity.

**When called:** During the `InsertSub` action, when `container`
already holds `capacity` items.

**Arguments:**
- `obj` — the object the player is trying to insert.
- `container` — the container that is full.

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `true`  | The game has handled the situation (and printed a message). |
| `false` | The library prints its default "There is no room" message. |

## 26.20 `ParseNumber(buffer, length)`

Called to parse a word as a number when the standard digit parser does
not recognize it.

**When called:** During `TryNumber()` (see §27.6), after `NumberWord()`
has been tried but *before* the standard digit parser. If `ParseNumber`
returns a non-zero value, that value is used and digit parsing is
skipped.

**Arguments:**
- `buffer` — byte address of the word text.
- `length` — byte length of the word.

**Return value:**

| Value | Effect                                                    |
|------ |---------------------------------------------------------- |
| 0     | The word is not a number.                                 |
| Other | The numeric value of the word.                            |

**Typical usage:** Handling number words in languages other than
English, or supporting written-out numbers:

```inform6
[ ParseNumber buf len;
    if (len == 7 && buf->0 == 'h' && buf->1 == 'u' && ...)  ! "hundred"
        return 100;
    return 0;
];
```

## 26.21 `Epilogue()`

Called after the game has ended (after the death/victory menu).

**When called:** After the `RESTART/RESTORE/QUIT` menu sequence
completes.

**Arguments:** None.

**Return value:** Not checked.

**Notes:** This entry point was added in library 6.12. It is called
unconditionally, regardless of `deadflag`.

## 26.22 `AfterRestore()`

Called after a game has been successfully restored from a saved file.

**When called:** After the `Restore` action succeeds and the game
state has been loaded.

**Arguments:** The `Stub` declaration reserves one local variable,
but no arguments are passed by the library's callers.

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `true`  | The game has handled the restore notification; library extensions' `ext_afterrestore` hooks also run. The default "OK, restored." message is suppressed. |
| `false` | The library prints its default "OK, restored." message. Library extensions are not run. |

**Typical usage:** Reinitializing state that is not saved (e.g.,
screen dimensions, Glk window handles):

```inform6
[ AfterRestore;
    DrawStatusLine();
    rtrue;
];
```

**Notes:** `AfterRestore` is declared via `Stub` in library 6.12.8.
It is also called after a save-and-restore cycle in `SaveSub` (when
the interpreter resumes from the save point).

## 26.23 `AfterSave()`

Called after a game has been successfully saved to a file.

**When called:** After the `Save` action succeeds.

**Arguments:** The `Stub` declaration reserves one local variable,
but no arguments are passed by the library's callers.

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `true`  | The game has handled the save notification; library extensions' `ext_aftersave` hooks also run. The default "OK, saved." message is suppressed. |
| `false` | The library prints its default "OK, saved." message. Library extensions are not run. |

**Typical usage:**

```inform6
[ AfterSave;
    print "Game saved successfully.^";
    rtrue;
];
```

## 26.24 `ParseNoun(obj)`

Called during noun phrase matching to allow the game to override or
supplement the standard name-matching algorithm.

**When called:** During `NounDomain()`, for each object being tested
against the player's noun phrase. Called before the standard dictionary
word matching.

**Arguments:** `obj` — the object being tested.

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| −1      | The routine has not handled this object; fall through to the standard name-matching algorithm. Library extensions' `ext_parsenoun` hooks are also tried. |
| 0       | No words matched; this object does not match the input.|
| *n* > 0 | The number of words in the input that matched this object. |

**Typical usage:** Matching objects whose names are generated
dynamically or cannot be expressed with the `name` property:

```inform6
[ ParseNoun obj w;
    if (obj == numbered_locker) {
        w = NextWord();
        if (w == 'locker') {
            w = TryNumber(wn);
            if (w == obj.locker_number) { wn++; return 2; }
            return 1;
        }
        return 0;
    }
    return -1;
];
```

**Notes:** `ParseNoun` uses `#Ifndef` rather than `Stub`. Its default
implementation returns −1 (not handled). The `wn` variable points to
the current word position; the routine may advance it.

## 26.25 `PrintRank()`

Called after the score is printed during the `Score` action, to print
a ranking or comment on the player's score.

**When called:** After `ScoreSub` prints the current score and maximum.

**Arguments:** None.

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `false` | Library extensions' `ext_printrank` hooks also run.    |
| `true`  | Library extensions are skipped.                        |

**Typical usage:**

```inform6
[ PrintRank;
    print ", earning you the rank of ";
    if (score >= 100)     "Master Adventurer.";
    if (score >= 50)      "Seasoned Explorer.";
    "Novice.";
];
```

**Notes:** `PrintRank` uses `#Ifndef` rather than `Stub`. Its default
implementation simply prints `"."` (a full stop).

## 26.26 `Amusing()`

Called when the player types `AMUSING` at the end-of-game menu after
winning.

**When called:** During the `AfterGameOver` menu, when `deadflag` is 2
(victory) and the player selects the `AMUSING` option.

**Arguments:** None.

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `false` | Library extensions' `ext_amusing` hooks also run.      |
| `true`  | Library extensions are skipped.                        |

**Typical usage:**

```inform6
Constant AMUSING_PROVIDED;

[ Amusing;
    "Did you try typing ~xyzzy~?^
     Did you find the secret passage behind the bookcase?";
];
```

**Notes:** The `AMUSING` option only appears in the end-of-game menu
if the constant `AMUSING_PROVIDED` is defined. Define it with
`Constant AMUSING_PROVIDED;` to enable the option. `Amusing` itself is
declared via `Stub`.

## 26.27 `HandleGlkEvent(event, context, buffer)` *(Glulx only)*

Called when a Glk event is received during input, allowing the game
to handle or override the event.

**When called:** Inside the Glk event loop, after a `glk_select()`
call returns an event. Called for character input, line input, and
timed-input contexts.

**Arguments:**

- `event` — pointer to the Glk event structure.
- `context` — 0 for line input, 1 for character input.

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| 0       | The event is not handled; library extensions' `ext_handleglkevent` hooks are also tried, then normal processing continues. |
| 2       | The event has been handled; the value in `gg_arguments-->0` is used as the result. |
| −1      | Force the input loop to continue waiting (cancel any pending completion). |

**Notes:** Only available when compiling for the Glulx target
(`TARGET_GLULX`). Declared via `Stub` inside `#Ifdef TARGET_GLULX`.

## 26.28 `IdentifyGlkObject(phase, type, ref, rock)` *(Glulx only)*

Called during Glk object recovery to let the game re-identify its
custom Glk objects (windows, streams, file references) after a
restore or restart.

**When called:** During `GGRecoverObjects()`, in three phases:

- Phase 0: Clear all game-held Glk references.
- Phase 1: Identify a specific Glk object. `type` is 0 for windows,
  1 for streams, 2 for file references. `ref` is the Glk object
  reference, `rock` is its rock value.
- Phase 2: Tie up loose ends after all objects have been identified.

**Arguments:**

- `phase` — 0, 1, or 2 (see above).
- `type` — object type (only meaningful in phase 1).
- `ref` — the Glk object reference (only meaningful in phase 1).
- `rock` — the rock value (only meaningful in phase 1).

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `false` | Library extensions' `ext_identifyglkobject` hooks also run. |
| `true`  | Library extensions are skipped for this call.           |

**Notes:** Only available when compiling for the Glulx target
(`TARGET_GLULX`). Declared via `Stub` with 4 arguments.

## 26.29 `InitGlkWindow(rock)` *(Glulx only)*

Called during Glk window setup to let the game create or customize
windows.

**When called:** During `GGInitialise()`, when the library is about
to create or reconfigure Glk windows.

**Arguments:** `rock` — the rock value identifying the window to be
created. The library uses `GG_MAINWIN_ROCK` for the main story
window, `GG_STATUSWIN_ROCK` for the status line window,
`GG_QUOTEWIN_ROCK` for quote box windows, 0 at the start of
initialization, and 1 at the end.

**Return value:**

| Value   | Effect                                                 |
|-------- |------------------------------------------------------- |
| `false` | Library extensions' `ext_InitGlkWindow` hooks are also tried. If still `false`, the library creates the window with its default settings. |
| `true`  | The game has handled window creation; the library skips its default window creation for this rock. When `rock` is 0 and `true` is returned, the entire default window setup is skipped. |

**Notes:** Only available when compiling for the Glulx target
(`TARGET_GLULX`). Declared via `Stub` with 1 argument.

## 26.30 Summary Table

The following table lists all library entry points, their argument
counts, when they are called, and whether they are required.

| Entry Point          | Args | When Called                           | Required? |
|--------------------- |----- |------------------------------------- |---------- |
| `Initialise`         | 0    | Game startup                         | **Yes**   |
| `BeforeParsing`      | 0    | Before each command is parsed        | No        |
| `UnknownVerb`        | 1    | Verb not found in grammar            | No        |
| `ParserError`        | 1    | Parser encounters an error           | No        |
| `ParseNoun`          | 1    | During noun phrase matching          | No        |
| `ChooseObjects`      | 2    | Disambiguation / "all" selection     | No        |
| `InScope`            | 1    | Scope computation                    | No        |
| `GamePreRoutine`     | 0    | Before every action                  | No        |
| `GamePostRoutine`    | 0    | After every action                   | No        |
| `DarkToDark`         | 0    | Moving dark-to-dark                  | No        |
| `NewRoom`            | 0    | Entering a new room                  | No        |
| `LookRoutine`        | 0    | After `Look` output                  | No        |
| `PrintRank`          | 0    | After score is printed               | No        |
| `DeathMessage`       | 0    | Death/victory (deadflag ≥ 3)         | No        |
| `AfterLife`          | 0    | After deadflag set (deadflag ≠ 2)    | No        |
| `Amusing`            | 0    | Player selects AMUSING after winning | No        |
| `AfterPrompt`        | 0    | After prompt, before input           | No        |
| `TimePasses`         | 0    | End-of-turn processing               | No        |
| `PrintVerb`          | 1    | Printing a verb word                 | No        |
| `PrintTaskName`      | 1    | Printing a task name in FULL SCORE   | No        |
| `ObjectDoesNotFit`   | 2    | Container at capacity                | No        |
| `ParseNumber`        | 2    | Parsing a non-digit number           | No        |
| `Epilogue`           | 0    | After game end menu                  | No        |
| `AfterRestore`       | 1    | After successful restore             | No        |
| `AfterSave`          | 1    | After successful save                | No        |
| `HandleGlkEvent`     | 2    | Glk event received (Glulx only)      | No        |
| `IdentifyGlkObject`  | 4    | Glk object recovery (Glulx only)     | No        |
| `InitGlkWindow`      | 1    | Glk window setup (Glulx only)        | No        |

## 26.31 Execution Order Summary

The following diagram shows the order in which entry points are called
during a single turn of the main game loop (simplified):

```
1.  AfterPrompt()              — after prompt, before input
2.  [Input is read]
3.  BeforeParsing()            — before tokenisation
4.  [Parser runs]
    4a. UnknownVerb(word)      — if verb not found
    4b. ParserError(etype)     — if parse fails
    4c. ParseNoun(obj)         — during noun matching
    4d. ChooseObjects(obj,code)— during disambiguation
    4e. InScope(actor)         — during scope computation
5.  GamePreRoutine()           — before action
6.  [before properties]
7.  [Action routine]
8.  [after properties]
9.  GamePostRoutine()          — after action
10. [End-of-turn sequence]
    10a. TimePasses()          — after daemons/timers/each_turn
    10b. NewRoom()             — if location changed
    10c. DarkToDark()          — if dark-to-dark transition
    10d. LookRoutine()         — after Look output
11. [If deadflag set]
    11a. AfterLife()           — chance to resurrect (deadflag ≠ 2)
    11b. DeathMessage()        — custom death text (deadflag ≥ 3)
    11c. Epilogue()            — after death message
    11d. PrintRank()           — after score display
    11e. [End-of-game menu]
    11f. Amusing()             — if player selects AMUSING
```

This is a simplified view; see §20.2 for the complete action pipeline
and §24.6 for the end-of-turn sequence.
