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

# Chapter 25: Scoring, Endgame, and Game State

This chapter documents the library's scoring system, endgame sequence,
and game-state operations: save, restore, restart, undo, transcription,
and verification. These facilities are implemented across `verblib.h`,
`parser.h`, and `english.h` in library version 6.12.8, with grammar
definitions in `grammar.h` and entry-point stubs in `grammar.h`.

## §25.1 The Score System

The library maintains a numeric score in the global variable `score`,
declared in `parser.h` (line 270):

```inform6
Global score;
```

The initial value is `0`. Games modify `score` directly or through the
library's automatic scoring mechanisms described in §25.2 and §25.3.

### §25.1.1 The `MAX_SCORE` Constant

The constant `MAX_SCORE` represents the maximum achievable score. It is
declared with a default value in `verblib.h` (line 28):

```inform6
Default MAX_SCORE 0;
```

To set a game's maximum score, define `MAX_SCORE` before including
`VerbLib`:

```inform6
Constant MAX_SCORE 350;

Include "Parser";
Include "VerbLib";
Include "Grammar";
```

`MAX_SCORE` is used by the library's score-display message. When the
player types `SCORE`, the library prints a message of the form "You have
so far scored X out of a possible Y" where Y is `MAX_SCORE`.

### §25.1.2 Score Notification

The library automatically notifies the player when the score changes.
This is controlled by two globals declared in `parser.h` (lines 271–272):

```inform6
Global last_score;          ! Score last turn (for testing for changes)
Global notify_mode = true;  ! Score notification
```

At the start of each turn, before the parser runs, the main game loop
in `parser.h` (line 5098) checks whether the score has changed:

```inform6
if (score ~= last_score) {
    if (notify_mode == 1) NotifyTheScore();
    last_score = score;
}
```

The `NotifyTheScore` routine (parser.h, line 5766) prints the
notification message:

```inform6
[ NotifyTheScore;
    #Ifdef TARGET_GLULX;
    glk_set_style(style_Note);
    #Endif;
    print "^[";
    L__M(##Miscellany, 50, score-last_score);
    print ".]^";
    #Ifdef TARGET_GLULX;
    glk_set_style(style_Normal);
    #Endif;
];
```

The `L__M(##Miscellany, 50)` message produces text such as "Your score
has just gone up by two points" or "Your score has just gone down by
one point", depending on the sign and magnitude of `score - last_score`.

**[Glulx]** On Glulx, the notification is displayed using the
`style_Note` Glk style, which interpreters typically render as italic or
otherwise visually distinct.

### §25.1.3 The `NOTIFY` Command

The player can toggle score notifications on and off with the `NOTIFY`
verb, defined in `grammar.h` (line 40):

```inform6
Verb meta 'notify'
    *                                           -> NotifyOn
    * 'on'                                      -> NotifyOn
    * 'off'                                     -> NotifyOff;
```

The action routines are defined in `verblib.h` (line 1339):

```inform6
[ NotifyOnSub;  notify_mode = true; L__M(##NotifyOn);  ];
[ NotifyOffSub; notify_mode = false; L__M(##NotifyOff); ];
```

Both `NOTIFY` and `NOTIFY ON` enable notifications; `NOTIFY OFF`
disables them.

### §25.1.4 The `SCORE` Action

The `SCORE` verb is defined in `grammar.h` (line 68):

```inform6
Verb meta 'score'
    *                                           -> Score;
```

The `ScoreSub` routine (verblib.h, line 1445) handles the action:

```inform6
[ ScoreSub;
    #Ifdef NO_SCORE;
    if (deadflag == 0) L__M(##Score, 2);
    #Ifnot;
    if (deadflag) new_line;
    L__M(##Score, 1);
    if (PrintRank() == false)
        LibraryExtensions.RunAll(ext_printrank);
    #Endif; ! NO_SCORE
];
```

When `NO_SCORE` is not defined, `ScoreSub` prints the current score via
`L__M(##Score, 1)` and then calls the `PrintRank` entry point. When
`NO_SCORE` is defined, the routine prints a message indicating that the
game does not use scoring.

After displaying the score, `ScoreSub` is also called during the
endgame sequence (see §25.5).

### §25.1.5 Disabling Scoring: `NO_SCORE`

To disable scoring entirely, define the `NO_SCORE` constant before
including the library:

```inform6
Constant NO_SCORE;

Include "Parser";
Include "VerbLib";
Include "Grammar";
```

When `NO_SCORE` is defined:

- The `SCORE` command prints a message saying the game has no score.
- The status line omits the score display (see §25.10).
- The `DrawStatusLine` routine skips the `SCORE__TX` label.

## §25.2 Automatic Scoring

The library provides two built-in automatic scoring mechanisms: room
scoring and object scoring. Both use the `scored` attribute.

### §25.2.1 Room Scoring

When the player visits a room that has the `scored` attribute for the
first time, the library awards points automatically. The number of
points per scored room is controlled by the `ROOM_SCORE` constant,
defaulting to 5 (verblib.h, line 31):

```inform6
Default ROOM_SCORE 5;
```

The `ScoreArrival` routine (verblib.h, line 2425) handles room scoring:

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

`ScoreArrival` is called from two places:

1. During `PlayerTo` when `flag` is `1` (verblib.h, line 1087):
   `NoteArrival(); ScoreArrival();`
2. At the end of `LookSub` (verblib.h, line 2514).

The `places_score` global (parser.h, line 273) tracks the running total
of points earned from visiting scored rooms. It is used by the
`FULLSCORE` display (see §25.3.3).

### §25.2.2 Object Scoring

When the player acquires an object that has the `scored` attribute for
the first time, the library awards points. The number of points per
scored object is controlled by the `OBJECT_SCORE` constant (verblib.h,
line 30):

```inform6
Default OBJECT_SCORE 4;
```

Object scoring is handled by `NoteObjectAcquisitions` (parser.h,
line 5561):

```inform6
[ NoteObjectAcquisitions i;
    objectloop (i in player)
        if (i hasnt moved) {
            give i moved;
            if (i has scored) {
                score = score + OBJECT_SCORE;
                things_score = things_score + OBJECT_SCORE;
            }
        }
];
```

This routine runs at the end of every turn as part of the end-of-turn
sequence (see §23.2). It iterates over all objects directly in the
player's inventory. For each object that has not yet been `moved`, it
gives the object the `moved` attribute. If the object also has `scored`,
points are added to both `score` and the `things_score` global
(parser.h, line 274).

### §25.2.3 Overriding Automatic Score Values

To change the points awarded per room or per object, define the
constants before including `VerbLib`:

```inform6
Constant ROOM_SCORE   10;
Constant OBJECT_SCORE  5;

Include "Parser";
Include "VerbLib";
Include "Grammar";
```

### §25.2.4 Example: Rooms and Objects with `scored`

```inform6
Object  treasure_room "Treasure Room"
  with  description "Gold glitters on every surface.",
  has   light scored;

Object  gold_crown "gold crown" treasure_room
  with  name 'gold' 'crown',
        description "A crown encrusted with jewels.",
  has   scored;
```

In this example, the player earns `ROOM_SCORE` points when first
entering the Treasure Room and `OBJECT_SCORE` points when first picking
up the gold crown.

## §25.3 Task-Based Scoring

For more fine-grained scoring, the library supports a task-based system
where each task has its own point value and completion state.

### §25.3.1 Enabling Task Scoring

Task scoring is controlled by two constants defined with defaults in
`verblib.h` (lines 29 and 33):

```inform6
Default NUMBER_TASKS    1;
Default TASKS_PROVIDED  1;
```

To use task-based scoring, set `TASKS_PROVIDED` to `0` (indicating that
the game provides tasks) and `NUMBER_TASKS` to the number of tasks:

```inform6
Constant TASKS_PROVIDED 0;
Constant NUMBER_TASKS   5;
```

Note that `TASKS_PROVIDED` uses inverted logic: a value of `0` means
tasks **are** provided; the default value of `1` means they are not.

### §25.3.2 The `task_scores` and `task_done` Arrays

The library declares two byte arrays in `verblib.h` (lines 35–39):

```inform6
#Ifndef task_scores;
Array  task_scores -> 0 0 0 0;
#Endif;

Array  task_done -> NUMBER_TASKS;
```

The `task_scores` array holds the point value for each task. The default
declaration provides a placeholder array of four zero bytes. Games must
define their own `task_scores` before including `VerbLib`:

```inform6
Constant TASKS_PROVIDED 0;
Constant NUMBER_TASKS   5;
Array task_scores -> 5 10 15 5 20;

Include "Parser";
Include "VerbLib";
Include "Grammar";
```

The `task_done` array is a byte array of `NUMBER_TASKS` elements, all
initialized to `0`. Each element tracks whether the corresponding task
has been completed (`1`) or not (`0`).

### §25.3.3 The `Achieved` Routine

The `Achieved` routine (verblib.h, line 1461) marks a task as complete
and awards its points:

```inform6
[ Achieved num;
    if (task_done->num == 0) {
        task_done->num = 1;
        score = score + TaskScore(num);
    }
];
```

Call `Achieved(N)` from game code to mark task N as complete. If the
task has already been completed, the call has no effect — points are
awarded only once per task. Task numbers are zero-based.

The `TaskScore` routine (verblib.h, line 1456) retrieves the point
value for a task:

```inform6
#Ifndef TaskScore;
[ TaskScore i;
    return task_scores->i;
];
#Endif;
```

Because `TaskScore` is guarded by `#Ifndef`, a game can provide its own
replacement `TaskScore` routine if task scores need to be computed
dynamically rather than read from the array.

### §25.3.4 Example: Using Task-Based Scoring

```inform6
Constant TASKS_PROVIDED 0;
Constant NUMBER_TASKS   3;
Array task_scores -> 10 25 15;

Constant MAX_SCORE 50;

Include "Parser";
Include "VerbLib";

[ PrintTaskName task_number;
    switch (task_number) {
        0: "finding the hidden key";
        1: "opening the sealed vault";
        2: "escaping the labyrinth";
    }
];

Include "Grammar";
```

In the game code, call `Achieved` when the player completes a task:

```inform6
[ UnlockVaultSub;
    print "The vault door swings open with a satisfying click.^";
    Achieved(1);
];
```

### §25.3.5 The `FULLSCORE` Action

The `FULLSCORE` verb is defined in `grammar.h` (line 71):

```inform6
Verb meta 'fullscore' 'full'
    *                                           -> FullScore
    * 'score'                                   -> FullScore;
```

The `FullScoreSub` routine (verblib.h, line 1481) displays a detailed
score breakdown:

```inform6
[ FullScoreSub i;
    ScoreSub();
    if (score == 0 || TASKS_PROVIDED == 1) rfalse;
    new_line;
    L__M(##FullScore, 1);
    for (i=0 : i<NUMBER_TASKS : i++)
        if (task_done->i == 1) {
            PANum(TaskScore(i));
            if (PrintTaskName(i) == false)
                LibraryExtensions.RunAll(ext_printtaskname, i);
        }
    if (things_score) {
        PANum(things_score);
        L__M(##FullScore, 2);
    }
    if (places_score) {
        PANum(places_score);
        L__M(##FullScore, 3);
    }
    new_line;
    PANum(score);
    L__M(##FullScore, 4);
];
```

The display includes:

| Component | Source | Message |
|-----------|--------|---------|
| Completed tasks | `task_done` / `PrintTaskName` | Game-defined task descriptions |
| Object acquisition | `things_score` | `L__M(##FullScore, 2)` — "finding sundry items" |
| Room exploration | `places_score` | `L__M(##FullScore, 3)` — "visiting various places" |
| Total | `score` | `L__M(##FullScore, 4)` — "total (out of MAX_SCORE)" |

If `TASKS_PROVIDED` is `1` (the default, meaning no tasks), `FullScoreSub`
returns after printing the basic score. If the total `score` is `0`, it
also returns immediately.

### §25.3.6 The `PrintTaskName` Entry Point

The `PrintTaskName` entry point is called by `FullScoreSub` to print
a description for each completed task. It is declared as a stub in
`grammar.h` (line 564):

```inform6
Stub PrintTaskName 1;
```

The stub generates a do-nothing routine if the game does not define
`PrintTaskName`. When defined, the routine receives the task number
(zero-based) as its argument and should print the task description.
If `PrintTaskName` returns `false`, the library calls
`LibraryExtensions.RunAll(ext_printtaskname, i)` as a fallback.

## §25.4 The Endgame: `deadflag`

The `deadflag` global variable controls whether the game is still in
progress and, if not, what kind of ending has occurred. It is declared
in `parser.h` (line 281):

```inform6
Global deadflag;            ! Normally 0, or false; 1 for dead
                            ! 2 for victorious, and higher numbers
                            ! represent exotic forms of death
```

### §25.4.1 Values of `deadflag`

| Value | Meaning |
|-------|---------|
| `0` | Game continues normally |
| `1` | Player has died |
| `2` | Player has won (victory) |
| `3+` | Custom ending (game-defined) |

To end the game, set `deadflag` to the appropriate value:

```inform6
! Player death
deadflag = 1;

! Victory
deadflag = 2;

! Custom ending
deadflag = 3;
```

### §25.4.2 How `deadflag` Stops the Game

The main game loop in `parser.h` (line 5087) runs while `deadflag` is
zero:

```inform6
while (~~deadflag) {
    ! ... parser, action processing, end-of-turn sequence ...
}
```

When `deadflag` becomes non-zero during action processing, the current
action completes. If the action was not meta and `deadflag` was set
during action processing, the end-of-turn sequence still runs, but each
step checks `deadflag` before proceeding (parser.h, lines 5259–5265):

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

This means that if `deadflag` is set during a daemon or timer, the
remaining end-of-turn steps are skipped.

### §25.4.3 The `DeathMessage` Entry Point

When `deadflag` is `3` or higher, the library calls the `DeathMessage`
entry point to print a custom ending message. The stub is declared in
`grammar.h` (line 554):

```inform6
Stub DeathMessage 0;
```

The entry point is called from `GameEpilogue` (parser.h, line 5413):

```inform6
switch (deadflag) {
    1:        L__M(##Miscellany, 3);
    2:        L__M(##Miscellany, 4);
    default:  print " ";
              if (DeathMessage() == false)
                  LibraryExtensions.RunAll(ext_deathmessage);
              print " ";
}
```

For `deadflag == 1`, the library prints "You have died" (or the
localized equivalent). For `deadflag == 2`, it prints "You have won".
For any other value, `DeathMessage` is responsible for printing the
ending text.

```inform6
[ DeathMessage;
    if (deadflag == 3) "You have been turned to stone";
    if (deadflag == 4) "You have been banished to another dimension";
];
```

### §25.4.4 The `AfterLife` Entry Point

After the main loop exits due to a non-zero `deadflag`, the library
calls `AfterLife` (parser.h, line 5252):

```inform6
if (deadflag ~= 2 && AfterLife() == false)
    LibraryExtensions.RunAll(ext_afterlife);
if (deadflag == 0) jump very__late__error;
```

`AfterLife` is called only when the player has not won (`deadflag ~= 2`).
The entry point can:

- **Resume play** by setting `deadflag` back to `0`. Execution jumps
  back to the main loop.
- **Change the ending** by setting `deadflag` to a different value.
- **Do nothing** by returning `false`, allowing the endgame to proceed.

The stub is declared in `grammar.h` (line 548):

```inform6
Stub AfterLife 0;
```

Example — giving the player a second chance:

```inform6
[ AfterLife;
    if (deadflag == 1 && ruby_amulet in player) {
        deadflag = 0;
        remove ruby_amulet;
        print "The ruby amulet flares with light, and you feel
            life flowing back into your body!^";
        rtrue;
    }
    rfalse;
];
```

### §25.4.5 The `Epilogue` Entry Point

The `Epilogue` entry point is called from `GameEpilogue` (parser.h,
line 5427) after the death/victory message has been printed but before
the score is displayed:

```inform6
Epilogue();
ScoreSub();
DisplayStatus();
AfterGameOver();
```

The stub is declared in `grammar.h` (line 555):

```inform6
Stub Epilogue 0;
```

`Epilogue` can print additional text after the ending message, such as
credits or statistics.

## §25.5 The End-of-Game Sequence

When `deadflag` becomes non-zero, the library executes the following
sequence:

1. **The current turn completes.** Action processing finishes. If the
   action was non-meta, the end-of-turn sequence runs (but each step
   checks `deadflag` and may exit early).

2. **`AfterLife()` is called** (only if `deadflag ~= 2`). If `AfterLife`
   or a library extension clears `deadflag` to `0`, play resumes from
   the top of the main loop.

3. **`GameEpilogue()` is called** (parser.h, line 5397). This routine:
   - Checks for any final score change and notifies the player.
   - Prints the ending banner: `*** You have died ***`,
     `*** You have won ***`, or a custom message from `DeathMessage`.
   - Calls `Epilogue()`.
   - Calls `ScoreSub()` to display the final score.
   - Calls `DisplayStatus()` to update `sline1` and `sline2`.
   - Calls `AfterGameOver()`.

4. **`AfterGameOver()` offers post-game options** (parser.h, line 5503).
   The player is prompted with options to choose from. The available
   options depend on the game's configuration:

   | Command | Condition | Action |
   |---------|-----------|--------|
   | `RESTART` | Always | Executes `@restart` |
   | `RESTORE` | Always | Calls `RestoreSub()` |
   | `QUIT` | Always | Executes `quit` |
   | `FULLSCORE` | `TASKS_PROVIDED == 0` | Calls `FullScoreSub()` |
   | `AMUSING` | `deadflag == 2` and `AMUSING_PROVIDED == 0` | Calls `Amusing()` |
   | `UNDO` | **[Z-machine]** Version 5+ only | Calls `PerformUndo()` |

### §25.5.1 The `AMUSING_PROVIDED` Constant and `Amusing` Entry Point

The `AMUSING_PROVIDED` constant (verblib.h, line 26) controls whether
the `AMUSING` option is offered after victory:

```inform6
Default AMUSING_PROVIDED 1;
```

Like `TASKS_PROVIDED`, the logic is inverted: set `AMUSING_PROVIDED` to
`0` to enable the amusing option:

```inform6
Constant AMUSING_PROVIDED 0;
```

The `Amusing` entry point (stub in grammar.h, line 550) is then called
when the player types `AMUSING` after winning:

```inform6
Stub Amusing 0;
```

Example:

```inform6
Constant AMUSING_PROVIDED 0;

[ Amusing;
    print "Have you tried...^^";
    print "  ...giving the fish to the cat?^";
    print "  ...wearing the colander as a hat?^";
    print "  ...typing XYZZY in the dark?^";
];
```

The relevant code in `AfterGameOver` (parser.h, line 5542):

```inform6
if (deadflag == 2 && i == AMUSING__WD) {
    amus_ret = false;
    if (AMUSING_PROVIDED == 0) {
        new_line;
        amus_ret = Amusing();
    }
    if (amus_ret == false)
        LibraryExtensions.RunAll(ext_amusing);
    jump RRQPL;
}
```

### §25.5.2 The `GameEpilogue` Routine

The full `GameEpilogue` routine (parser.h, line 5397):

```inform6
[ GameEpilogue;
    if (score ~= last_score) {
        if (notify_mode == 1) NotifyTheScore();
        last_score = score;
    }
    print "^^    ";
    #Ifdef TARGET_ZCODE;
    #IfV5; style bold; #Endif;
    #Ifnot; ! TARGET_GLULX
    glk_set_style(style_Alert);
    #Endif;
    print "***";
    switch (deadflag) {
        1:        L__M(##Miscellany, 3);
        2:        L__M(##Miscellany, 4);
        default:  print " ";
                  if (DeathMessage() == false)
                      LibraryExtensions.RunAll(ext_deathmessage);
                  print " ";
    }
    print "***";
    #Ifdef TARGET_ZCODE;
    #IfV5; style roman; #Endif;
    #Ifnot; ! TARGET_GLULX
    glk_set_style(style_Normal);
    #Endif;
    print "^^";
    #Ifndef NO_SCORE;
    print "^";
    #Endif;
    Epilogue();
    ScoreSub();
    DisplayStatus();
    AfterGameOver();
];
```

**[Z-machine]** On Z-machine version 5+, the ending banner is printed
in bold. On version 3–4, `style bold` is not available.

**[Glulx]** On Glulx, the banner uses `glk_set_style(style_Alert)`,
which interpreters typically render in bold or a similarly prominent
style.

## §25.6 Save and Restore

The library provides `SAVE` and `RESTORE` commands to write and read
game state to and from files. These are meta actions — they do not
consume a game turn. The implementations differ significantly between
Z-machine and Glulx.

### §25.6.1 The `SAVE` Action

The `SAVE` verb is defined in `grammar.h` (line 65):

```inform6
Verb meta 'save'
    *                                           -> Save;
```

**[Z-machine]** The Z-machine `SaveSub` (verblib.h, line 1142) uses
the `@save` opcode. The behavior depends on the Z-machine version:

```inform6
[ SaveSub flag;
    #IfV5;
    @save -> flag;
    switch (flag) {
        0: L__M(##Save, 1);
        1:
            if (AfterSave() == 1)
                LibraryExtensions.RunAll(ext_aftersave);
            else
                L__M(##Save, 2);
        2:
            RestoreColours();
            if (AfterRestore() == 1)
                LibraryExtensions.RunAll(ext_afterrestore);
            else
                L__M(##Restore, 2);
    }
    #Ifnot;
    save Smaybe;
    return L__M(##Save, 1);
  .SMaybe;
    if (AfterSave() == 1)
        LibraryExtensions.RunAll(ext_aftersave);
    else
        L__M(##Save, 2);
    #Endif; ! V5
];
```

In version 5+, `@save` stores a result value: `0` indicates failure,
`1` indicates a successful save, and `2` indicates that a restore has
just occurred (execution resumes from the save point). In versions 3–4,
`@save` uses branch-on-success semantics, jumping to `.SMaybe` on
success.

The result value of `2` is significant: when a player restores a saved
game, execution continues from within `SaveSub` at the point where
`@save` was called, but with a return value of `2`. This is why
`SaveSub` handles the restore case (calling `AfterRestore` and printing
the restore-success message).

**[Glulx]** The Glulx `SaveSub` (verblib.h, line 1240) uses Glk file
dialogs:

```inform6
[ SaveSub res fref;
    fref = glk_fileref_create_by_prompt($01, $01, 0);
    if (fref == 0) jump SFailed;
    gg_savestr = glk_stream_open_file(fref, $01, GG_SAVESTR_ROCK);
    glk_fileref_destroy(fref);
    if (gg_savestr == 0) jump SFailed;
    @save gg_savestr res;
    if (res == -1) {
        GGRecoverObjects();
        glk_stream_close(gg_savestr, 0);
        gg_savestr = 0;
        if (AfterRestore() == 1)
            return LibraryExtensions.RunAll(ext_afterrestore);
        else
            return L__M(##Restore, 2);
    }
    glk_stream_close(gg_savestr, 0);
    gg_savestr = 0;
    if (res == 0) {
        if (AfterSave() == 1)
            return LibraryExtensions.RunAll(ext_aftersave);
        else
            return L__M(##Save, 2);
    }
  .SFailed;
    L__M(##Save, 1);
];
```

The Glulx version prompts the user for a filename via
`glk_fileref_create_by_prompt`, opens a stream with
`glk_stream_open_file`, and saves the game state with the `@save`
opcode. On the Glulx `@save`, a result of `0` means success and `-1`
means a restore has just occurred. After a restore, `GGRecoverObjects`
must be called to rebuild Glk object references.

### §25.6.2 The `RESTORE` Action

The `RESTORE` verb is defined in `grammar.h` (line 62):

```inform6
Verb meta 'restore'
    *                                           -> Restore;
```

**[Z-machine]** The `RestoreSub` (verblib.h, line 1132):

```inform6
[ RestoreSub;
    restore Rmaybe;
    return L__M(##Restore, 1);
  .RMaybe;
    if (AfterRestore() == 1)
        LibraryExtensions.RunAll(ext_afterrestore);
    else
        L__M(##Restore, 2);
];
```

On the Z-machine, the `restore` statement invokes the `@restore`
opcode. If restoration succeeds, execution continues from within the
`SaveSub` that originally saved the game (in version 5+) or jumps to
`.RMaybe` (in versions 3–4). If restoration fails, `L__M(##Restore, 1)`
prints a failure message.

**[Glulx]** The `RestoreSub` (verblib.h, line 1227):

```inform6
[ RestoreSub res fref;
    fref = glk_fileref_create_by_prompt($01, $02, 0);
    if (fref == 0) jump RFailed;
    gg_savestr = glk_stream_open_file(fref, $02, GG_SAVESTR_ROCK);
    glk_fileref_destroy(fref);
    if (gg_savestr == 0) jump RFailed;
    @restore gg_savestr res;
    glk_stream_close(gg_savestr, 0);
    gg_savestr = 0;
  .RFailed;
    L__M(##Restore, 1);
];
```

On Glulx, `@restore` does not return on success — execution resumes
from the `@save` point in `SaveSub`. If `@restore` does return, the
restore failed.

### §25.6.3 The `AfterSave` and `AfterRestore` Entry Points

Both entry points are declared as stubs in `grammar.h` (lines 568–569):

```inform6
Stub AfterSave    1;
Stub AfterRestore 1;
```

These are called after a successful save or restore, respectively. If
the entry point returns `1`, the library calls the corresponding
`LibraryExtensions.RunAll` method instead of printing the default
success message. If it returns any other value, the default message is
printed.

```inform6
[ AfterRestore;
    ! Re-sync any cached state after restoring
    player.description = saved_player_desc;
    rfalse;  ! Print the default "Ok, restored" message
];
```

### §25.6.4 The `RESTART` Action

The `RESTART` verb is defined in `grammar.h` (line 59):

```inform6
Verb meta 'restart'
    *                                           -> Restart;
```

**[Z-machine]** The `RestartSub` (verblib.h, line 1127):

```inform6
[ RestartSub;
    L__M(##Restart, 1);
    if (YesOrNo()) { @restart; L__M(##Restart, 2); }
];
```

The routine prompts for confirmation using the `YesOrNo()` utility
function. If the player confirms, `@restart` reinitializes the game.
The `L__M(##Restart, 2)` message is printed only if `@restart` fails
(which is rare).

**[Glulx]** The Glulx `RestartSub` (verblib.h, line 1222) is
identical in structure:

```inform6
[ RestartSub;
    L__M(##Restart, 1);
    if (YesOrNo()) { @restart; L__M(##Restart, 2); }
];
```

Both platforms use the same `@restart` opcode syntax in Inform source,
though the underlying VM operation differs.

### §25.6.5 The `YesOrNo` Utility

The `YesOrNo` routine (verblib.h, line 1098) reads a line of input and
checks for affirmative or negative responses:

```inform6
[ YesOrNo noStatusRedraw i j;
    for (::) {
        #Ifdef TARGET_ZCODE;
        if (location == nothing || parent(player) == nothing
            || noStatusRedraw)
            read buffer parse;
        else
            read buffer parse DrawStatusLine;
        j = parse->1;
        #Ifnot; ! TARGET_GLULX
        noStatusRedraw = 0;
        KeyboardPrimitive(buffer, parse);
        j = parse-->0;
        #Endif;
        if (j) {
            i = parse-->1;
            if (i == YES1__WD or YES2__WD or YES3__WD) rtrue;
            if (i == NO1__WD or NO2__WD or NO3__WD) rfalse;
        }
        L__M(##Quit, 1);
        print "> ";
    }
];
```

The routine loops until the player types a recognized yes or no word.
The word constants (`YES1__WD`, `NO1__WD`, etc.) are defined in the
language definition file (`english.h`).

## §25.7 Undo

The library supports a single-level undo mechanism that allows the
player to reverse the most recent turn.

### §25.7.1 Undo State Management

Undo state is managed through two globals in `parser.h` (lines 209–210):

```inform6
Global undo_flag;           ! Can the interpreter provide "undo"?
Global just_undone;         ! Can't have two successive UNDOs
```

The `undo_flag` variable tracks the interpreter's undo capability:

| Value | Meaning |
|-------|---------|
| `0` | Interpreter does not support undo |
| `1` | Undo save failed (insufficient memory) |
| `2` | Undo state was saved successfully |

### §25.7.2 Saving Undo State

At the start of each turn, after checking for an undo command, the
`Keyboard` routine (parser.h, line 1383) saves the current game state:

```inform6
#Ifdef TARGET_ZCODE;
@save_undo i;
#Ifnot; ! TARGET_GLULX
@saveundo i;
if (i == -1) {
    GGRecoverObjects();
    i = 2;
}
else i = (~~i);
#Endif;
just_undone = 0;
undo_flag = 2;
if (i == -1) undo_flag = 0;
if (i == 0) undo_flag = 1;
```

**[Z-machine]** The `@save_undo` opcode saves the entire machine state.
The result is `-1` if the interpreter does not support undo, `0` if the
save failed, or `1` if the save succeeded. A result of `2` indicates
that an undo has just been performed and execution is resuming from
the save point.

**[Glulx]** The `@saveundo` opcode works similarly. After a successful
`@restoreundo`, execution resumes from the `@saveundo` point with a
result of `-1`. The library inverts the result and calls
`GGRecoverObjects` to rebuild Glk references.

### §25.7.3 Performing Undo

When the player types `UNDO`, the `PerformUndo` routine (parser.h,
line 1499) is called:

```inform6
[ PerformUndo i;
    if (turns == START_MOVE) {
        L__M(##Miscellany, 11);
        return 0;
    }
    if (undo_flag == 0) {
        L__M(##Miscellany, 6);
        return 0;
    }
    if (undo_flag == 1) {
        L__M(##Miscellany, 7);
        return 0;
    }
    #Ifdef TARGET_ZCODE;
    @restore_undo i;
    #Ifnot; ! TARGET_GLULX
    @restoreundo i;
    i = (~~i);
    #Endif;
    if (i == 0) {
        L__M(##Miscellany, 7);
        return 0;
    }
    L__M(##Miscellany, 6);
    return 1;
];
```

The routine checks three conditions before attempting undo:

1. If `turns == START_MOVE`, the game has just started — there is
   nothing to undo (`L__M(##Miscellany, 11)`: "You can't ~undo~ what
   hasn't been done!").
2. If `undo_flag == 0`, the interpreter does not support undo
   (`L__M(##Miscellany, 6)`).
3. If `undo_flag == 1`, the last undo save failed
   (`L__M(##Miscellany, 7)`).

If undo is available, `@restore_undo` (Z-machine) or `@restoreundo`
(Glulx) restores the previous state. On success, execution resumes
from the `@save_undo`/`@saveundo` point in `Keyboard`, and the
library prints the location name and a confirmation message
(`L__M(##Miscellany, 13)`).

### §25.7.4 Undo After Restoration

After a successful undo, the `Keyboard` routine (parser.h, line 1397)
handles the resumed state:

```inform6
if (i == 2) {
    RestoreColours();
    #Ifdef TARGET_ZCODE;
    style bold;
    #Ifnot; ! TARGET_GLULX
    glk_set_style(style_Subheader);
    #Endif;
    print (name) location, "^";
    #Ifdef TARGET_ZCODE;
    style roman;
    #Ifnot; ! TARGET_GLULX
    glk_set_style(style_Normal);
    #Endif;
    L__M(##Miscellany, 13);
    just_undone = 1;
    jump FreshInput;
}
```

The location name is reprinted in bold, the undo-success message is
displayed, and `just_undone` is set to `1` to prevent consecutive
undos. The parser then jumps to `FreshInput` to read a new command.

### §25.7.5 Undo Limitations

- **[Z-machine]** Undo requires version 5 or later. The code is
  wrapped in `#IfV5` / `#Endif` directives. Standard Z-machine
  interpreters provide a single level of undo.
- **[Glulx]** Multiple undo levels may be available depending on the
  interpreter, though the library's `PerformUndo` routine restores
  only one level at a time.
- The `just_undone` flag prevents two consecutive undos in the same
  input cycle, though this only affects undo performed during the
  parser's input handling — undo at the end-of-game prompt is not
  restricted this way.

## §25.8 Transcription

The library provides commands for recording game output to a transcript
file and for recording or replaying commands.

### §25.8.1 The `SCRIPT` / `TRANSCRIPT` Action

The `SCRIPT` verb is defined in `grammar.h` (line 75):

```inform6
Verb meta 'script' 'transcript'
    *                                           -> ScriptOn
    * 'on'                                      -> ScriptOn
    * 'off'                                     -> ScriptOff;

Verb meta 'noscript' 'unscript'
    *                                           -> ScriptOff;
```

The `transcript_mode` global (parser.h, line 212) tracks whether a
transcript is active:

```inform6
Global transcript_mode;     ! true when game scripting is on
```

**[Z-machine]** The `ScriptOnSub` routine (verblib.h, line 1179)
activates transcription via the `@output_stream 2` opcode:

```inform6
[ ScriptOnSub;
    transcript_mode = ((HDR_GAMEFLAGS-->0) & 1);
    if (transcript_mode) return L__M(##ScriptOn, 1);
    @output_stream 2;
    if (((HDR_GAMEFLAGS-->0) & 1) == 0)
        return L__M(##ScriptOn, 3);
    L__M(##ScriptOn, 2);
    transcript_mode = true;
];
```

The routine first checks the header flags to determine if a transcript
is already active. If not, it issues `@output_stream 2` to direct
output to the transcript file. On failure (the header flag is not set
after the opcode), it reports an error.

`ScriptOffSub` (verblib.h, line 1188) deactivates transcription:

```inform6
[ ScriptOffSub;
    transcript_mode = ((HDR_GAMEFLAGS-->0) & 1);
    if (transcript_mode == false) return L__M(##ScriptOff, 1);
    L__M(##ScriptOff, 2);
    @output_stream -2;
    if ((HDR_GAMEFLAGS-->0) & 1)
        return L__M(##ScriptOff, 3);
    transcript_mode = false;
];
```

The `@output_stream -2` opcode deactivates the transcript stream.

**[Glulx]** On Glulx, transcription uses Glk file and stream objects.
The relevant globals are declared in `parser.h` (lines 236–237):

```inform6
Global gg_scriptfref = 0;
Global gg_scriptstr = 0;
```

The `ScriptOnSub` routine (verblib.h, line 1278):

```inform6
[ ScriptOnSub;
    if (gg_scriptstr) return L__M(##ScriptOn, 1);
    if (gg_scriptfref == 0) {
        gg_scriptfref = glk_fileref_create_by_prompt(
            $102, $05, GG_SCRIPTFREF_ROCK);
        if (gg_scriptfref == 0) jump S1Failed;
    }
    gg_scriptstr = glk_stream_open_file(
        gg_scriptfref, $05, GG_SCRIPTSTR_ROCK);
    if (gg_scriptstr == 0) jump S1Failed;
    glk_window_set_echo_stream(gg_mainwin, gg_scriptstr);
    L__M(##ScriptOn, 2);
    return;
  .S1Failed;
    L__M(##ScriptOn, 3);
];
```

The Glulx version prompts for a transcript file via
`glk_fileref_create_by_prompt`, opens it as a Glk stream, and sets it
as the echo stream for the main window using
`glk_window_set_echo_stream`. All output to the main window is then
automatically echoed to the transcript file.

`ScriptOffSub` (verblib.h, line 1293) closes the stream:

```inform6
[ ScriptOffSub;
    if (gg_scriptstr == 0) return L__M(##ScriptOff, 1);
    L__M(##ScriptOff, 2);
    glk_stream_close(gg_scriptstr, 0);
    gg_scriptstr = 0;
];
```

### §25.8.2 Command Recording and Replay

The library supports recording commands to a file and replaying them.
The `RECORDING` verb is defined in `grammar.h` (lines 51–54):

```inform6
Verb meta 'recording'
    *                                           -> CommandsOn
    * 'on'                                      -> CommandsOn
    * 'off'                                     -> CommandsOff;

Verb meta 'replay'
    *                                           -> CommandsRead;
```

The `xcommsdir` global (parser.h, line 215) tracks the command
recording state:

```inform6
Global xcommsdir;           ! true if command recording is on
```

**[Z-machine]** The Z-machine implementations use output and input
stream opcodes:

```inform6
[ CommandsOnSub;
    @output_stream 4;
    xcommsdir = 1;
    L__M(##CommandsOn, 1);
];

[ CommandsOffSub;
    if (xcommsdir == 1) @output_stream -4;
    xcommsdir = 0;
    L__M(##CommandsOff, 1);
];

[ CommandsReadSub;
    @input_stream 1;
    xcommsdir = 2;
    L__M(##CommandsRead, 1);
];
```

`@output_stream 4` directs command input to a recording file.
`@output_stream -4` stops recording. `@input_stream 1` switches the
input source to the command file for replay.

**[Glulx]** The Glulx implementations use Glk file and stream
operations. The relevant globals (parser.h, lines 239–240):

```inform6
Global gg_commandstr = 0;
Global gg_command_reading = 0;  ! true if gg_commandstr is being replayed
```

The `CommandsOnSub` routine (verblib.h, line 1300):

```inform6
[ CommandsOnSub fref;
    if (gg_commandstr) {
        if (gg_command_reading)
            return L__M(##CommandsOn, 2);
        else
            return L__M(##CommandsOn, 3);
    }
    fref = glk_fileref_create_by_prompt($103, $01, 0);
    if (fref == 0) return L__M(##CommandsOn, 4);
    gg_command_reading = false;
    gg_commandstr = glk_stream_open_file(
        fref, $01, GG_COMMANDWSTR_ROCK);
    glk_fileref_destroy(fref);
    if (gg_commandstr == 0) return L__M(##CommandsOn, 4);
    L__M(##CommandsOn, 1);
];
```

`CommandsOffSub` (verblib.h, line 1314) closes the command stream:

```inform6
[ CommandsOffSub;
    if (gg_commandstr == 0) return L__M(##CommandsOff, 2);
    if (gg_command_reading) return L__M(##CommandsRead, 5);
    glk_stream_close(gg_commandstr, 0);
    gg_commandstr = 0;
    gg_command_reading = false;
    L__M(##CommandsOff, 1);
];
```

`CommandsReadSub` (verblib.h, line 1323) opens a file for replay:

```inform6
[ CommandsReadSub fref;
    if (gg_commandstr) {
        if (gg_command_reading)
            return L__M(##CommandsRead, 2);
        else
            return L__M(##CommandsRead, 3);
    }
    fref = glk_fileref_create_by_prompt($103, $02, 0);
    if (fref == 0) return L__M(##CommandsRead, 4);
    gg_command_reading = true;
    gg_commandstr = glk_stream_open_file(
        fref, $02, GG_COMMANDRSTR_ROCK);
    glk_fileref_destroy(fref);
    if (gg_commandstr == 0) return L__M(##CommandsRead, 4);
    L__M(##CommandsRead, 1);
];
```

### §25.8.3 Glk Rock Constants

The Glk rock constants for stream identification are declared in
`parser.h` (lines 222–226):

| Constant | Value | Purpose |
|----------|-------|---------|
| `GG_SAVESTR_ROCK` | 301 | Save/restore file stream |
| `GG_SCRIPTSTR_ROCK` | 302 | Transcript stream |
| `GG_COMMANDWSTR_ROCK` | 303 | Command recording stream |
| `GG_COMMANDRSTR_ROCK` | 304 | Command replay stream |
| `GG_SCRIPTFREF_ROCK` | 401 | Transcript file reference |

These constants allow the library to identify its streams during
`GGRecoverObjects` calls after an undo or restore.

## §25.9 Verify

The `VERIFY` command checks the integrity of the story file to detect
corruption.

### §25.9.1 The `VERIFY` Action

The `VERIFY` verb is defined in `grammar.h` (line 83):

```inform6
Verb meta 'verify'
    *                                           -> Verify;
```

**[Z-machine]** The `VerifySub` routine (verblib.h, line 1170) uses the
`@verify` opcode:

```inform6
[ VerifySub;
    @verify ?Vmaybe;
    jump Vwrong;
  .Vmaybe;
    return L__M(##Verify, 1);
  .Vwrong;
    L__M(##Verify, 2);
];
```

The `@verify` opcode computes a checksum of the story file and compares
it to the expected value stored in the file header. If the checksum
matches, execution branches to `.Vmaybe` and a success message is
printed. If it does not match, execution falls through to `.Vwrong` and
a failure message is printed.

**[Glulx]** The Glulx `VerifySub` (verblib.h, line 1272) uses the
Glulx `@verify` opcode:

```inform6
[ VerifySub res;
    @verify res;
    if (res == 0) return L__M(##Verify, 1);
    L__M(##Verify, 2);
];
```

Unlike the Z-machine version, the Glulx `@verify` stores a result
value: `0` indicates success, and any other value indicates failure.

### §25.9.2 Purpose

Story file verification is useful for detecting files that have been
corrupted during transfer or storage. It compares the checksum computed
at runtime against the checksum embedded in the file header during
compilation.

## §25.10 The Status Line

The status line is a single line at the top of the screen that displays
the current location and either score/turns or the time of day. The
library manages it through a set of globals and the `DrawStatusLine`
routine.

### §25.10.1 Status Line Globals

The key globals are declared in `parser.h`:

```inform6
Global sline1;                  ! line 190 — must be second
Global sline2;                  ! line 191 — must be third
```

```inform6
#Ifndef sys_statusline_flag;
Global sys_statusline_flag = 0; ! line 251–252 — non-zero for time display
#Endif;
```

The `sline1` and `sline2` globals hold the values displayed on the
right side of the status line. Their meaning depends on
`sys_statusline_flag`:

| `sys_statusline_flag` | `sline1` | `sline2` | Display |
|-----------------------|----------|----------|---------|
| `0` (default) | `score` | `turns` | Score and turn count |
| Non-zero | `the_time / 60` | `the_time % 60` | Hours and minutes |

The `DisplayStatus` routine (parser.h, line 5756) sets these values
each turn:

```inform6
[ DisplayStatus;
    if (sys_statusline_flag == 0) {
        sline1 = score;
        sline2 = turns;
    }
    else {
        sline1 = the_time / 60;
        sline2 = the_time % 60;
    }
];
```

### §25.10.2 The `DrawStatusLine` Routine

The `DrawStatusLine` routine renders the status line on screen. The
library provides a default implementation that can be replaced using the
`Replace` directive.

The default implementation (parser.h, line 6498) handles both Z-machine
and Glulx:

```inform6
#Ifndef DrawStatusLine;
[ DrawStatusLine width posa posb posc posd pose;
    #Ifdef TARGET_GLULX;
    if (gg_statuswin == 0)
        return;
    #Endif;

    if (location == nothing || parent(player) == nothing)
        return;

    StatusLineHeight(gg_statuswin_size);
    MoveCursor(1, 1);

    width = ScreenWidth();
    posa = width - 26;
    posb = width - 13;
    posc = width - 5;
    posd = width - 19;
    pose = width - 14;

    spaces width;
    MoveCursor(1, 2);
    if (location == thedark) {
        print (name) location;
    }
    else {
        FindVisibilityLevels();
        if (visibility_ceiling == location)
            print (name) location;
        else
            print (The) visibility_ceiling;
    }
    ! ... score/turns or time display ...
];
#Endif;
```

The routine:

1. Sets the status window height via `StatusLineHeight`.
2. Moves the cursor to position (1, 1) and clears the line with spaces.
3. Prints the current location name (or the visibility ceiling if the
   player is inside a container or on a supporter).
4. On the right side, prints either score/turns or the time of day
   depending on `sys_statusline_flag`.

The score/turns display section adapts to the available screen width:

- **Width > 66 characters**: Displays `Score: N` and `Moves: N`
  separately.
- **Width 54–66 characters**: Displays `N/N` (score/turns) in a
  compact format.
- **Width ≤ 53 characters**: Displays `N/N` in minimal format.

When `NO_SCORE` is defined, the score portion is suppressed and only
the moves count is shown.

### §25.10.3 Switching Between Score and Time Display

The `sys_statusline_flag` global determines the display mode. For
score/turns mode (the default), leave it at `0`. For time mode, the
library sets it automatically based on the time system.

Games using timed play (see §23.6) set the time via `SetTime` in
`Initialise`, which causes the status line to show time instead of
score/turns. The `sys_statusline_flag` can also be set directly:

```inform6
[ Initialise;
    sys_statusline_flag = 1;   ! Force time display
    ! ...
];
```

### §25.10.4 Replacing `DrawStatusLine`

Because the default `DrawStatusLine` is guarded by `#Ifndef`, a game
can provide its own implementation:

```inform6
Replace DrawStatusLine;

[ DrawStatusLine width;
    StatusLineHeight(1);
    MoveCursor(1, 1);
    width = ScreenWidth();
    spaces width;
    MoveCursor(1, 2);
    print (name) location;
    MoveCursor(1, width - 20);
    print "Chapter ", chapter_number;
    MainWindow();
];
```

This replaces the standard score/turns display with a custom "Chapter"
indicator.

**[Z-machine]** On Z-machine version 3, the status line is drawn in
hardware by the interpreter, not by `DrawStatusLine`. The
`DrawStatusLine` routine is used only in version 5+ and version 6.

**[Z-machine]** On Z-machine version 6, the library provides a
separate `DrawStatusLine` implementation (verblib.h, line 6439) that
uses version 6–specific features like `@set_font`, `@set_cursor`, and
`@get_wind_prop` for precise graphical layout.

**[Glulx]** On Glulx, the status window is managed through Glk window
operations. The `gg_statuswin` global holds the Glk window reference.
If `gg_statuswin` is `0` (no status window), `DrawStatusLine` returns
immediately.

### §25.10.5 The `PrintRank` Entry Point

The `PrintRank` entry point is called by `ScoreSub` after displaying
the score. It allows games to print a rank or title based on the
current score. The default definition in `grammar.h` (line 577):

```inform6
#Ifndef PrintRank;
[ PrintRank; "."; ];
#Endif;
```

The default simply prints a period to end the score sentence. A game
can override this:

```inform6
[ PrintRank;
    print ", earning you the rank of ";
    if (score >= 350) "Grandmaster Adventurer.";
    if (score >= 250) "Master Adventurer.";
    if (score >= 150) "Seasoned Adventurer.";
    if (score >= 50)  "Novice Adventurer.";
    "Beginner.";
];
```

Because `PrintRank` is guarded by `#Ifndef` in `grammar.h`, defining
it in the game source before including `Grammar` replaces the default.
If `PrintRank` returns `false`, the library additionally calls
`LibraryExtensions.RunAll(ext_printrank)`.

### §25.10.6 The `StatusLineHeight` Routine

The `StatusLineHeight` routine (parser.h, line 6271 for Glulx,
line 6289 for Z-machine) manages the height of the status window:

**[Z-machine]**

```inform6
[ StatusLineHeight height;
    ! Sets the status window to the given height in lines.
];
```

**[Glulx]**

```inform6
[ StatusLineHeight height wx wy x y charh;
    ! Uses Glk window size operations to resize the status window.
];
```

The default status line height is `1` line, stored in the
`gg_statuswin_size` variable. Games can call `StatusLineHeight` with
a larger value to create a multi-line status area, though this requires
a custom `DrawStatusLine` to populate the additional lines.

### §25.10.7 Summary of Status Line–Related Identifiers

| Identifier | Type | Source | Purpose |
|-----------|------|--------|---------|
| `sline1` | Global | parser.h | Left value (score or hours) |
| `sline2` | Global | parser.h | Right value (turns or minutes) |
| `sys_statusline_flag` | Global | parser.h | `0` for score/turns, non-zero for time |
| `DrawStatusLine` | Routine | parser.h | Renders the status line |
| `StatusLineHeight` | Routine | parser.h | Sets the status window height |
| `DisplayStatus` | Routine | parser.h | Updates `sline1` and `sline2` |
| `PrintRank` | Entry point | grammar.h | Prints rank after score display |
| `NO_SCORE` | Constant | Game-defined | Suppresses score on status line |
| `gg_statuswin` | Global | parser.h | **[Glulx]** Glk status window reference |
| `gg_statuswin_size` | Global | parser.h | **[Glulx]** Status window height |
| `SCORE__TX` | Constant | english.h | "Score: " label text |
| `MOVES__TX` | Constant | english.h | "Moves: " label text |
| `TIME__TX` | Constant | english.h | "Time: " label text |
