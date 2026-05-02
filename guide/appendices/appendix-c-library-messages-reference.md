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

# Appendix C: Library Messages Reference

This appendix provides a complete listing of every customizable library
message in the standard library. For a full explanation of the action
processing pipeline, see Chapter 20 (Actions). For details on customizing
messages, see Chapter 32 (Replacing and Extending the Library).

Every message listed here can be intercepted and replaced by game code using
the `LibraryMessages` object or the `LibraryExtensions` mechanism. The
**Default Text** column shows the English-language default.
Messages marked **Tense-aware** adapt their wording based on whether the game
uses present or past tense narration.

---

## §C.1 How Library Messages Work

The standard library produces all of its player-visible text through a
centralized message dispatch system. Rather than printing text directly, action
handler routines call `L__M` (Library Message), which routes the request
through a chain of message handlers before falling back to the default text
in the `LanguageLM` routine.

### §C.1.1 The L__M Dispatch Mechanism

The `L__M` routine dispatches library messages:

```inform6
[ L__M act n x1 x2 s;
    if (keep_silent == 2) return;
    s = sw__var;
    sw__var = act;
    if (n == 0) n = 1;
    L___M(n, x1, x2);
    sw__var = s;
];
```

Parameters:
- **`act`** — The action identifier (e.g., `##Take`, `##Miscellany`)
- **`n`** — The message number within that action (defaults to 1 if 0)
- **`x1`** — First parameter (typically the primary object involved)
- **`x2`** — Second parameter (typically a secondary object)

Key behaviors:
- If `keep_silent == 2`, the message is suppressed entirely. This is used
  during implicit actions to prevent unwanted output.
- The `sw__var` global is temporarily set to the action value so that the
  message handler chain can inspect it via `action`.
- If `n` is 0, it defaults to 1. This means `L__M(##Pray)` is equivalent
  to `L__M(##Pray, 1)`.

The inner `L___M` routine handles the actual
dispatch chain:

```inform6
[ L___M n x1 x2 s;
    s = action;
    lm_n = n;
    lm_o = x1;
    lm_s = x2;
    action = sw__var;
    if (RunRoutines(LibraryMessages, before))             { action = s; rfalse; }
    if (LibraryExtensions.RunWhile(ext_messages, false )) { action = s; rfalse; }
    action = s;
    LanguageLM(n, x1, x2);
];
```

The dispatch chain tries three handlers in order:

1. **`LibraryMessages.before()`** — The game's custom message object
2. **`LibraryExtensions.RunWhile(ext_messages)`** — Extension message handlers
3. **`LanguageLM(n, x1, x2)`** — The default language-specific messages

If either of the first two handlers returns `true`, the default message is
suppressed. The `action` global is temporarily set to the action being
reported, then restored afterward.

### §C.1.2 The LibraryMessages Object

The `LibraryMessages` object provides the primary customization point. It is
declared in `verblib.h` (lines 41–43):

```inform6
#Ifndef LibraryMessages;
Object LibraryMessages;
#Endif;
```

The `#Ifndef` guard means that if the game defines its own `LibraryMessages`
object *before* including `VerbLib`, the library's empty default is skipped.
The game's version is used instead.

### §C.1.3 The LibraryExtensions Hook

The `LibraryExtensions` mechanism provides an alternative customization point,
primarily intended for library extensions that need to modify messages without
conflicting with the game's `LibraryMessages` object. Extensions register
themselves with `LibraryExtensions` and provide an `ext_messages` property
that works like `LibraryMessages.before()`.

### §C.1.4 Message Parameters (lm_n, lm_o, lm_s)

When `L___M` is called, it copies the message parameters into three global
variables before invoking the handler chain (`parser.h`, lines 336–338):

```inform6
Global lm_n;    ! Message number (n)
Global lm_o;    ! First parameter (x1) — typically noun
Global lm_s;    ! Second parameter (x2) — typically second
```

Inside a `LibraryMessages.before()` routine, the game can inspect:
- **`action`** — Which action's message is being produced
- **`lm_n`** — The message number within that action
- **`lm_o`** — The first object parameter
- **`lm_s`** — The second object parameter

---

## §C.2 Customizing Library Messages

### §C.2.1 Basic Customization Pattern

To customize library messages, define a `LibraryMessages` object with a
`before` property *before* including `VerbLib`:

```inform6
Object LibraryMessages
  with before [;
    Take:
      if (lm_n == 1) { "Snagged!"; }
    Drop:
      if (lm_n == 3) { "Tossed aside."; }
    Miscellany:
      if (lm_n == 38) { "I have no idea what you mean."; }
  ];

Include "VerbLib";
```

The `before` routine is called with `action` set to the action being reported.
Use `lm_n` to check the specific message number. Return `true` (by printing
a string with `;` or using `rtrue`) to suppress the default message. Return
`false` (or fall through) to allow the default.

### §C.2.2 Using LibraryExtensions

Library extensions can provide message overrides without conflicting with the
game's `LibraryMessages` object:

```inform6
Object MyExtMessages
  with ext_messages [;
    Look:
      if (lm_n == 7) { "Nothing catches your eye."; }
  ];
```

The extension object is registered with `LibraryExtensions` during
initialization. Its `ext_messages` property works identically to
`LibraryMessages.before()`.

---

## §C.3 Action Messages (A–D)

This section documents all library messages for actions whose names begin with
A through D, in alphabetical order.

### §C.3.1 Answer / Ask (`##Answer`, `##Ask`)

**Messages:** 1

These two actions share a single message handler via the `Answer,Ask:` case
label.

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "There is no reply." | — | — | Tense-aware |

### §C.3.2 Attack (`##Attack`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Violence isn't the answer to this one." | — | — | Tense-aware |

### §C.3.3 Blow (`##Blow`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] can't usefully blow [x1]." | `x1` = noun | — | Tense-aware |

### §C.3.4 Burn (`##Burn`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "This dangerous act would achieve little." | — | — | Tense-aware |
| 2 | `DecideAgainst()` | — | — | When noun is animate |

### §C.3.5 Buy (`##Buy`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Nothing is on sale." | — | — | Tense-aware |

### §C.3.6 Climb (`##Climb`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Climbing [x1] would achieve little." | `x1` = noun | — | Tense-aware |
| 2 | `DecideAgainst()` | — | — | When noun is animate |

### §C.3.7 Close (`##Close`)

**Messages:** 4

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] is not something [actor] can close." | `x1` = noun | — | Tense-aware |
| 2 | "[x1] is already closed." | `x1` = noun | — | |
| 3 | "[Actor] close[s] [x1]." | `x1` = noun | — | Success message; tense-aware |
| 4 | "(first closing [x1])" | `x1` = noun | — | Implicit action |

### §C.3.8 CommandsOff (`##CommandsOff`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Command recording off.]" | — | — | |
| 2 | "[Command recording already off.]" | — | — | [Glulx only] |

### §C.3.9 CommandsOn (`##CommandsOn`)

**Messages:** 4

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Command recording on.]" | — | — | |
| 2 | "[Commands are currently replaying.]" | — | — | [Glulx only] |
| 3 | "[Command recording already on.]" | — | — | [Glulx only] |
| 4 | "[Command recording failed.]" | — | — | [Glulx only] |

### §C.3.10 CommandsRead (`##CommandsRead`)

**Messages:** 5

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Replaying commands.]" | — | — | |
| 2 | "[Commands are already replaying.]" | — | — | [Glulx only] |
| 3 | "[Command replay failed. Command recording is on.]" | — | — | [Glulx only] |
| 4 | "[Command replay failed.]" | — | — | [Glulx only] |
| 5 | "[Command replay complete.]" | — | — | [Glulx only] |

### §C.3.11 Consult (`##Consult`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] discover[s] nothing of interest in [x1]." | `x1` = noun | — | Tense-aware |

### §C.3.12 Cut (`##Cut`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Cutting [x1] up would achieve little." | `x1` = noun | — | Tense-aware |
| 2 | `DecideAgainst()` | — | — | When noun is animate |

### §C.3.13 Dig (`##Dig`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Digging would achieve nothing here." | — | — | Tense-aware |

### §C.3.14 Disrobe (`##Disrobe`)

**Messages:** 4

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] isn't wearing [x1]." | `x1` = noun | — | Tense-aware |
| 2 | "[Actor] take[s] off [x1]." | `x1` = noun | — | Success message; tense-aware |
| 3 | "(first taking [x1] off)" | `x1` = obj | — | Implicit action |
| 4 | "[Actor] will need to remove [noun] first." | — | — | When `no_implicit_actions` is set; tense-aware |

### §C.3.15 Drink (`##Drink`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "There's nothing suitable to drink here." | — | — | Tense-aware |

### §C.3.16 Drop (`##Drop`)

**Messages:** 4

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] is already here." | `x1` = noun | — | |
| 2 | "[Actor] haven't/hasn't got [x1]." | `x1` = noun | — | Tense-aware |
| 3 | "Dropped." | — | — | Success message |
| 4 | "[Actor] need[s] to take [x1] off/out of [x2] before dropping it." | `x1` = noun, `x2` = parent | — | When `no_implicit_actions` is set; tense-aware |

---

## §C.4 Action Messages (E–L)

### §C.4.1 Eat (`##Eat`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] is plainly inedible." | `x1` = noun | — | |
| 2 | "[Actor] eat[s] [x1]. Not bad." | `x1` = noun | — | Success; "Not bad" only for player actor; tense-aware |

### §C.4.2 EmptyT (`##EmptyT`)

**Messages:** 4

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] can't contain things." | `x1` = noun or second | — | |
| 2 | "[x1] is closed." | `x1` = noun or second | — | |
| 3 | "[x1] is empty already." | `x1` = noun | — | |
| 4 | "That wouldn't empty anything." | — | — | Tense-aware |

### §C.4.3 Enter (`##Enter`)

**Messages:** 7

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "But [actor] is already on/in [x1]." | `x1` = noun | — | |
| 2 | "[x1] is not something [actor] can stand on/sit on/lie on/enter." | `x1` = noun, `x2` = verb_word | — | `x2` selects phrasing |
| 3 | "[Actor] can't get into the closed [x1]." | `x1` = noun | — | |
| 4 | "[Actor] can only get into something free-standing." | — | — | |
| 5 | "[Actor] get[s] onto/into [x1]." | `x1` = noun | — | Success; tense-aware |
| 6 | "(getting off/out of [x1])" | `x1` = intermediate object | — | Implicit action |
| 7 | "(getting onto/into [x1])" | `x1` = intermediate object | — | Implicit action |

### §C.4.4 Examine (`##Examine`)

**Messages:** 3

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Darkness, noun. An absence of light to see by." | — | — | In darkness |
| 2 | "[Actor] see[s] nothing special about [x1]." | `x1` = noun | — | When no description; tense-aware |
| 3 | "[x1] is currently switched on/off." | `x1` = noun | — | For switchable objects; tense-aware |

### §C.4.5 Exit (`##Exit`)

**Messages:** 6

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "But [actor] isn't in anything at the moment." | — | — | |
| 2 | "[Actor] can't get out of the closed [x1]." | `x1` = parent | — | |
| 3 | "[Actor] get[s] off/out of [x1]." | `x1` = parent | — | Success; tense-aware |
| 4 | "[Actor] isn't on/in [x1]." | `x1` = noun | — | |
| 5 | "(first getting off/out of [x1])" | `x1` = obj | — | Implicit action |
| 6 | "[Actor] stand[s] up." | — | — | Tense-aware |

### §C.4.6 Fill (`##Fill`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "There isn't anything obvious with which to fill [x1]." | `x1` = noun | — | Tense-aware |
| 2 | "Filling [x1] from [x2] wouldn't make sense." | `x1` = noun, `x2` = second | — | Tense-aware |

### §C.4.7 FullScore (`##FullScore`)

**Messages:** 4

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "The score is/was made up as follows:" | — | — | Adapts to `deadflag` |
| 2 | "finding sundry items" | — | — | Task score line |
| 3 | "visiting various places" | — | — | Places score line |
| 4 | "total (out of [MAX_SCORE])" | — | — | Final score line |

### §C.4.8 GetOff (`##GetOff`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "But [actor] isn't on [x1] at the moment." | `x1` = noun | — | Tense-aware |

### §C.4.9 Give (`##Give`)

**Messages:** 4

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] isn't holding [x1]." | `x1` = noun | — | Tense-aware |
| 2 | "[Actor] juggle[s] [x1] for a while, but don't achieve much." | `x1` = noun | — | When giving to self; tense-aware |
| 3 | "[x1] don't seem interested." | `x1` = second | — | Default NPC refusal |
| 4 | "[Actor] hand[s] over [x1]." | `x1` = noun | — | Success; tense-aware |

### §C.4.10 Go (`##Go`)

**Messages:** 7

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] will have to get off/out of [x1] first." | `x1` = container/supporter | — | Tense-aware |
| 2 | "[Actor] can't go that way." | — | — | Tense-aware |
| 3 | "[Actor] is unable to climb [x1]." | `x1` = door object | — | Going up through a door |
| 4 | "[Actor] is unable to descend by [x1]." | `x1` = door object | — | Going down through a door |
| 5 | "[Actor] can't, since [x1] is in the way." | `x1` = door object | — | Door blocking other direction |
| 6 | "[Actor] can't, since [x1] lead[s] nowhere." | `x1` = door object | — | Door with no destination; tense-aware |
| 7 | "[Actor] depart[s]." | — | — | NPC going; tense-aware |

### §C.4.11 Insert (`##Insert`)

**Messages:** 9

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] need[s] to be holding [x1] before [actor] can put it into something else." | `x1` = noun | — | Tense-aware |
| 2 | "[x1] can't contain things." | `x1` = second (or noun in parser) | — | |
| 3 | "[x1] is closed." | `x1` = second | — | |
| 4 | "[Actor] will need to take [x1] off first." | `x1` = noun | — | For worn items; tense-aware |
| 5 | "[Actor] can't put something inside itself." | — | — | |
| 6 | "(first taking [x1] off)" | `x1` = noun | — | Implicit action |
| 7 | "There is no more room in [x1]." | `x1` = second | — | Tense-aware |
| 8 | "Done." | — | — | Multi-object success |
| 9 | "[Actor] put[s] [x1] into [x2]." | `x1` = noun, `x2` = second | — | Success; tense-aware |

### §C.4.12 Inv (`##Inv`)

**Messages:** 4

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] is carrying nothing." | — | — | |
| 2 | "[Actor] is carrying" | — | — | Inventory header (no newline) |
| 3 | ":" | — | — | After header when `NEWLINE_BIT` set |
| 4 | "." | — | — | After inventory list when `ENGLISH_BIT` set |

### §C.4.13 Jump (`##Jump`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] jump[s] on the spot, fruitlessly." | — | — | Tense-aware |

### §C.4.14 JumpIn (`##JumpIn`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Jumping in [x1] would achieve nothing here." | `x1` = noun | — | Tense-aware |
| 2 | `DecideAgainst()` | — | — | When noun is animate |

### §C.4.15 JumpOn (`##JumpOn`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Jumping upon [x1] would achieve nothing here." | `x1` = noun | — | Tense-aware |
| 2 | `DecideAgainst()` | — | — | When noun is animate |

### §C.4.16 JumpOver (`##JumpOver`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Jumping over [x1] would achieve nothing here." | `x1` = noun | — | Tense-aware |
| 2 | `DecideAgainst()` | — | — | When noun is animate |

### §C.4.17 Kiss (`##Kiss`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Keep your mind on the game." | — | — | |

### §C.4.18 Listen (`##Listen`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] hear[s] nothing unexpected." | — | — | Tense-aware |

### §C.4.19 ListMiscellany (`##ListMiscellany`)

**Messages:** 22

These messages are used internally by the object listing system
(`WriteListFrom` and related routines in `verblib.h`) to format parenthetical
annotations and container/supporter contents in room descriptions and
inventory listings.

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | " (providing light)" | — | — | Light source annotation |
| 2 | " (which is/are closed)" | `x1` = obj | — | Closed container |
| 3 | " (closed and providing light)" | — | — | Combined state |
| 4 | " (which is/are empty)" | `x1` = obj | — | Empty container |
| 5 | " (empty and providing light)" | — | — | Combined state |
| 6 | " (which is/are closed and empty)" | `x1` = obj | — | Combined state |
| 7 | " (closed, empty and providing light)" | — | — | Combined state |
| 8 | " (providing light and being worn" | `x1` = obj | — | Light + worn |
| 9 | " (providing light" | `x1` = obj | — | Light only |
| 10 | " (being worn" | `x1` = obj | — | Worn only |
| 11 | " (which is/are " | `x1` = obj | — | Open container prefix |
| 12 | "open" | — | — | Open + has children |
| 13 | "open but empty" | — | — | Open + no children |
| 14 | "closed" | — | — | Closed, not locked |
| 15 | "closed and locked" | — | — | Closed + locked |
| 16 | " and empty" | — | — | Appended when empty with parenth |
| 17 | " (which is/are empty)" | `x1` = obj | — | Empty without parenth |
| 18 | " containing " | — | — | Before listing contents |
| 19 | " (on " | — | — | Terse supporter prefix |
| 20 | ", on top of " | — | — | Verbose supporter prefix |
| 21 | " (in " | — | — | Terse container prefix |
| 22 | ", inside " | — | — | Verbose container prefix |

### §C.4.20 LMode1 (`##LMode1`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Story] is now in its ~brief~ printing mode..." | — | — | Brief mode |

### §C.4.21 LMode2 (`##LMode2`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Story] is now in its ~verbose~ mode..." | — | — | Verbose mode |

### §C.4.22 LMode3 (`##LMode3`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Story] is now in its ~superbrief~ mode..." | — | — | Superbrief mode |

### §C.4.23 Lock (`##Lock`)

**Messages:** 5

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] don't seem to be something [actor] can lock." | `x1` = noun | — | Tense-aware |
| 2 | "[x1] is locked at the moment." | `x1` = noun | — | |
| 3 | "[Actor] will first have to close [x1]." | `x1` = noun | — | |
| 4 | "[x1] don't seem to fit the lock." | `x1` = second | — | Wrong key |
| 5 | "[Actor] lock[s] [x1]." | `x1` = noun | — | Success; tense-aware |

### §C.4.24 Look (`##Look`)

**Messages:** 7

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | " (on [x1])" | `x1` = supporter | — | Room header annotation |
| 2 | " (in [x1])" | `x1` = container | — | Room header annotation |
| 3 | " (as [x1])" | `x1` = player | — | When `print_player_flag` is 1 |
| 4 | "On [x1] [list of items]." | `x1` = supporter | — | Objects on supporter in room |
| 5 | "[Actor] can also see [list]." | `x1` = location/container | — | "also see" variant |
| 6 | "[Actor] can see [list]." | `x1` = location/container | — | "can see" variant |
| 7 | "[Actor] see[s] nothing unexpected in that direction." | `x1` = compass direction | — | Tense-aware; also used by `CompassDirection.description` |

### §C.4.25 LookUnder (`##LookUnder`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "But it's dark." | — | — | In darkness; tense-aware |
| 2 | "[Actor] find[s] nothing of interest." | — | — | Tense-aware |

---

## §C.5 Action Messages (M–R)

### §C.5.1 Mild (`##Mild`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Quite." | — | — | Response to mild language |

### §C.5.2 Miscellany (`##Miscellany`)

**Messages:** 60

The `##Miscellany` pseudo-action is a catch-all for parser messages, system
messages, and other text that does not belong to any specific game action.
These messages are called from throughout `parser.h` and `verblib.h`. For a
thematic breakdown, see §C.7.

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "(considering the first sixteen objects only)" | — | — | Multi-object limit |
| 2 | "Nothing to do." | — | — | Empty "all" list |
| 3 | " [Player] have/has died " | — | — | Death message fragment |
| 4 | " [Player] have/has won " | — | — | Victory message fragment |
| 5 | "Would you like to RESTART, RESTORE..., or QUIT?" | — | — | End-of-game menu; conditional `UNDO`, `FULL`, `AMUSING` |
| 6 | "[Your interpreter does not provide ~undo~. Sorry!]" | — | — | |
| 7 | "~Undo~ failed. [Not all interpreters provide it.]" | — | — | [Z-machine] |
| 7 | "[You cannot ~undo~ any further.]" | — | — | [Glulx] — same number, different text |
| 8 | "Please give one of the answers above." | — | — | Invalid end-of-game input |
| 9 | "It is now pitch dark in here." | — | — | Darkness notification; tense-aware |
| 10 | "I beg your pardon?" | — | — | Empty input |
| 11 | "[You can't ~undo~ what hasn't been done!]" | — | — | Undo at start of game |
| 12 | "[Can't ~undo~ twice in succession. Sorry!]" | — | — | Consecutive undo attempt |
| 13 | "[Previous turn undone.]" | — | — | Successful undo |
| 14 | "Sorry, that can't be corrected." | — | — | Failed OOPS |
| 15 | "Think nothing of it." | — | — | OOPS with nothing to correct |
| 16 | "~Oops~ can only correct a single word." | — | — | Multiple-word OOPS |
| 17 | "It is pitch dark, and [actor] can't see a thing." | — | — | Darkness room description; tense-aware |
| 18 | "yourself" | — | — | Pronoun text for self-reference |
| 19 | "As good-looking as ever." | — | — | Examining self |
| 20 | "To repeat a command like ~frog, jump~, just say ~again~..." | — | — | AGAIN misuse |
| 21 | "[Actor] can hardly repeat that." | — | — | Can't repeat; tense-aware |
| 22 | "[Actor] can't begin with a comma." | — | — | Leading comma; tense-aware |
| 23 | "[Actor] seem[s] to want to talk to someone, but I can't see whom." | — | — | Tense-aware |
| 24 | "[Actor] can't talk to [x1]." | `x1` = inanimate object | — | Tense-aware |
| 25 | "To talk to someone, try ~someone, hello~ or some such." | — | — | |
| 26 | "(first taking [x1])" | `x1` = object | — | Implicit take |
| 27 | "I didn't understand that sentence." | — | — | `STUCK_PE` parser error |
| 28 | "I only understood you as far as wanting to " | — | — | `UPTO_PE` — partial parse; followed by msg 56 |
| 29 | "I didn't understand that number." | — | — | `NUMBER_PE` |
| 30 | "[Actor] can't see any such thing." | — | — | `CANTSEE_PE`; tense-aware |
| 31 | "[Actor] seem[s] to have said too little." | — | — | `TOOLIT_PE`; tense-aware |
| 32 | "[Actor] isn't holding that." | `x1` = not_holding | — | `NOTHELD_PE` |
| 33 | "You can't use multiple objects with that verb." | — | — | `MULTI_PE` |
| 34 | "You can only use multiple objects once on a line." | — | — | `MMULTI_PE` |
| 35 | "I'm not sure what ~[x1]~ refers to." | `x1` = pronoun_word (address) | — | `VAGUE_PE` |
| 36 | "You excepted something not included anyway." | — | — | `EXCEPT_PE` |
| 37 | "[Actor] can only do that to something animate." | — | — | `ANIMA_PE`; tense-aware |
| 38 | "That's not a verb I recognise." | — | — | `VERB_PE`; US variant: "recognize" (`DIALECT_US`) |
| 39 | "That's not something you need to refer to in the course of this game." | — | — | `SCENERY_PE` |
| 40 | "[Actor] can't see ~[x1]~ ([x2]) at the moment." | `x1` = pronoun_word (address), `x2` = pronoun_obj | — | `ITGONE_PE`; tense-aware |
| 41 | "I didn't understand the way that finished." | — | — | `JUNKAFTER_PE` |
| 42 | "None/Only [x1] of those is/are available." | `x1` = multi_had (count) | — | `TOOFEW_PE` |
| 43 | "Nothing to do." | — | — | `NOTHING_PE` with `multi_wanted == 100` |
| 44 | "There is nothing to [x1]." | `x1` = verb_word (address) | — | `NOTHING_PE`; tense-aware |
| 45 | "Who do you mean, " | — | — | Disambiguation (creature) |
| 46 | "Which do you mean, " | — | — | Disambiguation (thing) |
| 47 | "Sorry, you can only have one item here. Which exactly?" | — | — | Single-object disambiguation |
| 48 | "Whom do you want [x1] to [command]?" | `x1` = actor | — | Incomplete creature token |
| 49 | "What do you want [x1] to [command]?" | `x1` = actor | — | Incomplete object token |
| 50 | "The score has just gone up/down by [x1] point[s]." | `x1` = score change | — | Score notification |
| 51 | "(Since something dramatic has happened, your list of commands has been cut short.)" | — | — | Multi-action interruption |
| 52 | "Type a number from 1 to [x1], 0 to redisplay or press ENTER." | `x1` = line count | — | Menu prompt |
| 53 | "[Please press SPACE.]" | — | — | Menu/page pause |
| 54 | "[Comment recorded.]" | — | — | When transcript/commands active |
| 55 | "[Comment NOT recorded.]" | — | — | When no transcript active |
| 56 | "." | — | — | Sentence terminator after msg 28 |
| 57 | "?" | — | — | Question mark for disambiguation |
| 58 | "(first taking [x1] off/out of [x2])" | `x1` = object, `x2` = container/supporter | — | Implicit take from container/supporter |
| 59 | "You'll have to be more specific." | — | — | Ambiguous "take all" |
| 60 | "[x1] observes that " | `x1` = NPC | — | NPC speech prefix |

### §C.5.3 No / Yes (`##No`, `##Yes`)

**Messages:** 1

These two actions share a single message handler via the `No,Yes:` case label.

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "That was a rhetorical question." | — | (No),  (Yes) | |

### §C.5.4 NotifyOff (`##NotifyOff`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Score notification off." | — | — | |

### §C.5.5 NotifyOn (`##NotifyOn`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Score notification on." | — | — | |

### §C.5.6 Objects (`##Objects`)

**Messages:** 10

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] have/has handled" | — | — | List header |
| 2 | "[Actor] have/has handled nothing." | — | — | Empty list |
| 3 | " (worn)" | `x1` = location, `x2` = object | — | Object is worn |
| 4 | " (held)" | `x1` = location, `x2` = object | — | Object is held |
| 5 | " (given away)" | `x1` = location, `x2` = object | — | Object is with an animate |
| 6 | " (in [x1])" | `x1` = location (name), `x2` = object | — | Object is in a visited room |
| 7 | " (in [x1])" | `x1` = location (the), `x2` = object | — | Object is in an enterable |
| 8 | " (inside [x1])" | `x1` = location (the), `x2` = object | — | Object is in a container |
| 9 | " (on [x1])" | `x1` = location (the), `x2` = object | — | Object is on a supporter |
| 10 | " (lost)" | `x1` = location, `x2` = object | — | Object location unknown |

### §C.5.7 Open (`##Open`)

**Messages:** 6

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] is not something [actor] can open." | `x1` = noun | — | Not openable; tense-aware |
| 2 | "[x1] seem[s] to be locked." | `x1` = noun | — | Tense-aware |
| 3 | "[x1] is already open." | `x1` = noun | — | |
| 4 | "[Actor] open[s] [x1], revealing [contents/nothing]." | `x1` = noun | — | Success with contents; tense-aware |
| 5 | "[Actor] open[s] [x1]." | `x1` = noun | — | Success without contents; tense-aware |
| 6 | "(first opening [x1])" | `x1` = noun | — | Implicit action |

### §C.5.8 Order (`##Order`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] has better things to do." | `x1` = noun (NPC) | — | Default NPC refusal |

### §C.5.9 Places (`##Places`)

**Messages:** 3

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] have/has visited" | — | — | List header |
| 2 | "." | — | — | List terminator |
| 3 | "[Actor] have/has visited nothing." | — | — | Empty list |

### §C.5.10 Pray (`##Pray`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Nothing practical results from [actor's] prayer." | — | — | Tense-aware |

### §C.5.11 Prompt (`##Prompt`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "^>" | — | — | Command prompt; `^` is a newline |

### §C.5.12 Pronouns (`##Pronouns`)

**Messages:** 5

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "At the moment, " | — | — | Pronoun list header |
| 2 | "means " | — | — | After pronoun word |
| 3 | "is unset" | — | — | Unassigned pronoun |
| 4 | "no pronouns are known to the game." | — | — | No pronouns defined |
| 5 | "." | — | — | Pronoun list terminator |

### §C.5.13 Pull / Push / Turn (`##Pull`, `##Push`, `##Turn`)

**Messages:** 6

These three actions share a single message handler via the `Pull,Push,Turn:`
case label.

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Punishing [oneself] that way isn't likely to help matters." | — | (Pull),  (Push),  (Turn) | When noun is player; tense-aware |
| 2 | "[x1] is fixed in place." | `x1` = noun | (Pull),  (Push),  (Turn) | Static object |
| 3 | "[Actor] is unable to." | — | (Pull),  (Push),  (Turn) | Scenery object |
| 4 | "Nothing obvious happens." | — | (Pull),  (Push),  (Turn) | Default success; tense-aware |
| 5 | "That would be less than courteous." | — | (Pull),  (Push),  (Turn) | Animate object; tense-aware |
| 6 | `DecideAgainst()` | — | (Pull) | When noun is actor (not player) |

### §C.5.14 PushDir (`##PushDir`)

**Messages:** 3

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "That really wouldn't serve any purpose." | — | — | Tense-aware |
| 2 | "That's not a direction." | — | — | Not a compass direction; tense-aware |
| 3 | "Not that way [actor] can't." | — | — | Up/down not allowed; tense-aware |

### §C.5.15 PutOn (`##PutOn`)

**Messages:** 8

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] need[s] to be holding [x1] before [actor] can put it on top of something else." | `x1` = noun | — | Tense-aware |
| 2 | "[Actor] can't put something on top of itself." | — | — | Tense-aware |
| 3 | "Putting things on [x1] would achieve nothing." | `x1` = second | — | Not a supporter; tense-aware |
| 4 | "[Actor] lack[s] the dexterity." | — | — | When noun is actor; tense-aware |
| 5 | "(first taking [x1] off)" | `x1` = noun | — | Implicit action |
| 6 | "There is no more room on [x1]." | `x1` = second | — | Full supporter; tense-aware |
| 7 | "Done." | — | — | Multi-object success |
| 8 | "[Actor] put[s] [x1] on [x2]." | `x1` = noun, `x2` = second | — | Success; tense-aware |

### §C.5.16 Quit (`##Quit`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Please answer yes or no." | — | — | Yes/no prompt |
| 2 | "Are you sure you want to quit? " | — | — | Confirmation |

### §C.5.17 Remove (`##Remove`)

**Messages:** 4

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] is unfortunately closed." | `x1` = container | — | |
| 2 | "But [x1] isn't there now." | `x1` = noun | — | Tense-aware |
| 3 | "Removed." | — | — | Success |
| 4 | "But [x1] isn't in or on anything." | `x1` = noun | — | Not inside container/supporter; tense-aware |

### §C.5.18 Restart (`##Restart`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Are you sure you want to restart? " | — | — | Confirmation |
| 2 | "Failed." | — | — | Restart failed |

### §C.5.19 Restore (`##Restore`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Restore failed." | — | — | |
| 2 | "Ok." | — | — | Restore succeeded |

### §C.5.20 Rub (`##Rub`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] achieve[s] nothing by this." | — | — | Tense-aware |
| 2 | `DecideAgainst()` | — | — | When noun is animate |

### §C.5.21 RunTimeError (`##RunTimeError`)

**Messages:** 16

These are internal runtime error diagnostics, printed with `** ... **`
delimiters. They indicate programming errors or exceeded limits. For a
complete discussion, see §C.9.

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Preposition not found (this should not occur)" | — | — | Internal error |
| 2 | "Property value not routine or string: ~[x2]~ of ~[x1]~" | `x1` = object, `x2` = property | — | |
| 3 | "Entry in property list not routine or string: ~[x2]~ list of ~[x1]~" | `x1` = object, `x2` = property | — | |
| 4 | "Too many timers/daemons are active simultaneously. The limit is MAX_TIMERS (currently [MAX_TIMERS])" | — | — | |
| 5 | "Object ~[x1]~ has no ~[x2]~ property" | `x1` = object, `x2` = property | — | |
| 7 | "Object ~[x1]~ can only be used as a player object if it has the ~number~ property" | `x1` = object | — | Note: number 6 is skipped |
| 8 | "Attempt to take random entry from an empty table array" | — | — | |
| 9 | "[x1] is not a valid direction property number" | `x1` = number | — | |
| 10 | "The player-object is outside the object tree" | — | — | |
| 11 | "The room ~[x1]~ has no ~[x2]~ property" | `x1` = room, `x2` = property | — | |
| 12 | "Tried to set a non-existent pronoun using SetPronoun" | — | — | |
| 13 | "A 'topic' token can only be followed by a preposition" | — | — | |
| 14 | "Overflowed buffer limit of [x1] using '@@64output_stream 3' [x2]" | `x1` = limit, `x2` = context string | — | |
| 15 | "LoopWithinObject broken because [x1] was moved while the loop passed through it." | `x1` = object | — | |
| 16 | "Attempt to use illegal narrative_voice of [x1]." | `x1` = voice value | — | |

---

## §C.6 Action Messages (S–Z)

### §C.6.1 Save (`##Save`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Save failed." | — | — | |
| 2 | "Ok." | — | — | Save succeeded |

### §C.6.2 Score (`##Score`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "You have so far scored [score] out of a possible [MAX_SCORE], in [turns] turn[s]." | — | — | Adapts to `deadflag` ("In that game you scored...") |
| 2 | "There is no score in this story." | — | — | When `NO_SCORE` is defined |

### §C.6.3 ScriptOff (`##ScriptOff`)

**Messages:** 3

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Transcripting is already off." | — | — | |
| 2 | "End of transcript." | — | — | |
| 3 | "Attempt to end transcript failed." | — | — | [Z-machine only] |

### §C.6.4 ScriptOn (`##ScriptOn`)

**Messages:** 3

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Transcripting is already on." | — | — | |
| 2 | "Start of a transcript of [version info]" | — | — | Calls `VersionSub()` |
| 3 | "Attempt to begin transcript failed." | — | — | |

### §C.6.5 Search (`##Search`)

**Messages:** 7

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "But it's dark." | — | — | In darkness; tense-aware |
| 2 | "There is nothing on [x1]." | `x1` = noun | — | Empty supporter; tense-aware |
| 3 | "On [x1] [list of items]." | `x1` = noun | — | Supporter with contents |
| 4 | "[Actor] find[s] nothing of interest." | — | — | Not a container; tense-aware |
| 5 | "[Actor] can't see inside, since [x1] is/are closed." | `x1` = noun | — | Closed container; tense-aware |
| 6 | "[x1] is/are empty." | `x1` = noun | — | Empty container |
| 7 | "In [x1] [list of items]." | `x1` = noun | — | Container with contents |

### §C.6.6 Set (`##Set`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] can't set [x1]." | `x1` = noun | — | Tense-aware |

### §C.6.7 SetTo (`##SetTo`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] can't set [x1] to anything." | `x1` = noun | — | Tense-aware |

### §C.6.8 Show (`##Show`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] isn't holding [x1]." | `x1` = noun | — | Tense-aware |
| 2 | "[x1] is unimpressed." | `x1` = second | — | |

### §C.6.9 Sing (`##Sing`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor's] singing is abominable." | — | — | Tense-aware |

### §C.6.10 Sleep (`##Sleep`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] isn't feeling especially drowsy." | — | — | Tense-aware |

### §C.6.11 Smell (`##Smell`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] smell[s] nothing unexpected." | — | — | Tense-aware |
| 2 | `DecideAgainst()` | — | — | When noun is animate |

### §C.6.12 Sorry (`##Sorry`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Oh, don't apologise." | — | — | US variant: "apologize" (`DIALECT_US`) |

### §C.6.13 Squeeze (`##Squeeze`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | `DecideAgainst()` | — | — | When noun is animate (and not player) |
| 2 | "[Actor] achieve[s] nothing by this." | — | — | Tense-aware |

### §C.6.14 Strong (`##Strong`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Real adventurers do not use such language." | — | — | Tense-aware |

### §C.6.15 Swim (`##Swim`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "There's not enough water to swim in." | — | — | Tense-aware |

### §C.6.16 Swing (`##Swing`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "There's nothing sensible to swing here." | — | — | Tense-aware |

### §C.6.17 SwitchOff (`##SwitchOff`)

**Messages:** 3

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] is not something [actor] can switch." | `x1` = noun | — | Not switchable; tense-aware |
| 2 | "[x1] is already off." | `x1` = noun | — | |
| 3 | "[Actor] switch[es] [x1] off." | `x1` = noun | — | Success; tense-aware |

### §C.6.18 SwitchOn (`##SwitchOn`)

**Messages:** 3

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] is not something [actor] can switch." | `x1` = noun | — | Not switchable; tense-aware |
| 2 | "[x1] is already on." | `x1` = noun | — | |
| 3 | "[Actor] switch[es] [x1] on." | `x1` = noun | — | Success; tense-aware |

### §C.6.19 Take (`##Take`)

**Messages:** 13

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Taken." | — | — | Success |
| 2 | "[Actor] is always self-possessed." | — | — | Taking self |
| 3 | "I don't suppose [x1] would care for that." | `x1` = noun (animate) | — | Taking animate; tense-aware |
| 4 | "[Actor] will have to get off/out of [x1] first." | `x1` = ancestor | — | Noun contains actor; tense-aware |
| 5 | "[Actor] already have/has [x1]." | `x1` = noun | — | Already held; tense-aware |
| 6 | "[x2] seem[s] to belong to [x1]." | `x1` = owner, `x2` = noun | — | Belongs to animate; tense-aware |
| 7 | "[x2] seem[s] to be a part of [x1]." | `x1` = parent, `x2` = noun | — | Part of another object; tense-aware |
| 8 | "[x1] is not available." | `x1` = noun | — | In unreachable place |
| 9 | "[x1] is not open." | `x1` = container | — | Closed container |
| 10 | "[x1] is hardly portable." | `x1` = noun | — | Scenery object |
| 11 | "[x1] is fixed in place." | `x1` = noun | — | Static object |
| 12 | "[Actor] is carrying too many things already." | — | — | Capacity exceeded |
| 13 | "(putting [x1] into [x2] to make room)" | `x1` = displaced object, `x2` = SACK_OBJECT | — | Auto-stash into sack |

### §C.6.20 Taste (`##Taste`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] taste[s] nothing unexpected." | — | — | Tense-aware |
| 2 | `DecideAgainst()` | — | — | When noun is animate |

### §C.6.21 Tell (`##Tell`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] talk[s] to [oneself] for a while." | — | — | Telling self; tense-aware |
| 2 | "This provokes no reaction." | — | — | Default NPC response; tense-aware |

### §C.6.22 Think (`##Think`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "What a good idea." | — | — | |

### §C.6.23 ThrowAt (`##ThrowAt`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Futile." | — | — | When no second or second is inanimate |
| 2 | "[Actor] lack[s] the nerve when it comes to the crucial moment." | — | — | Second is animate; tense-aware |

### §C.6.24 Tie (`##Tie`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] would achieve nothing by this." | — | — | Tense-aware |
| 2 | `DecideAgainst()` | — | — | When noun is animate |

### §C.6.25 Touch (`##Touch`)

**Messages:** 3

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | `DecideAgainst()` | — | — | When noun is animate |
| 2 | "[Actor] feel[s] nothing unexpected." | — | — | Tense-aware |
| 3 | "That really wouldn't serve any purpose." | — | — | Touching self; tense-aware |

### §C.6.26 Transfer (`##Transfer`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] is still holding [x1]." | `x1` = noun | — | |

### §C.6.27 Unlock (`##Unlock`)

**Messages:** 5

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] don't seem to be something [actor] can unlock." | `x1` = noun | — | Not lockable; tense-aware |
| 2 | "[x1] is unlocked at the moment." | `x1` = noun | — | Already unlocked |
| 3 | "[x1] don't seem to fit the lock." | `x1` = second | — | Wrong key |
| 4 | "[Actor] unlock[s] [x1]." | `x1` = noun | — | Success; tense-aware |
| 5 | "(first unlocking [x1])" | `x1` = noun | — | Implicit action |

### §C.6.28 VagueGo (`##VagueGo`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] will have to say which compass direction to go in." | — | — | Tense-aware |

### §C.6.29 Verify (`##Verify`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "The game file has verified as intact." | — | — | |
| 2 | "The game file did not verify as intact, and may be corrupt." | — | — | |

### §C.6.30 Wait (`##Wait`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Time passes." | — | — | Tense-aware |

### §C.6.31 Wake (`##Wake`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "The dreadful truth is, this is not a dream." | — | — | Tense-aware |

### §C.6.32 WakeOther (`##WakeOther`)

**Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "That seems unnecessary." | — | — | Tense-aware |

### §C.6.33 Wave (`##Wave`)

**Messages:** 3

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "But [actor] isn't holding [x1]." | `x1` = noun | — | Tense-aware |
| 2 | "[Actor] look[s] ridiculous waving [x1] [at x2]." | `x1` = noun, `x2` = second (optional) | — | Tense-aware; `x2` only shown if nonzero |
| 3 | `DecideAgainst()` | — | — | When noun is actor |

### §C.6.34 WaveHands (`##WaveHands`)

**Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] wave[s], feeling foolish." | — | — | Tense-aware; no target |
| 2 | "[Actor] wave[s] at [x1], feeling foolish." | `x1` = noun | — | Tense-aware; waving at something |

### §C.6.35 Wear (`##Wear`)

**Messages:** 5

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] can't wear [x1]." | `x1` = noun | — | Not clothing; tense-aware |
| 2 | "[Actor] is not holding [x1]." | `x1` = noun | — | Not held; tense-aware |
| 3 | "[Actor] is already wearing [x1]." | `x1` = noun | — | Already worn |
| 4 | "[Actor] put[s] on [x1]." | `x1` = noun | — | Success; tense-aware |
| 5 | "[Actor] need[s] to take [x1] out/off of [x2] before wearing it." | `x1` = noun, `x2` = parent | — | When `no_implicit_actions` is set; tense-aware |

---

## §C.7 Parser and System Messages (`##Miscellany`)

The `##Miscellany` messages documented in §C.5.2 can be organized into
thematic groups. This section provides that thematic breakdown for easier
reference when customizing specific categories of parser output.

### §C.7.1 Parser Error Messages (27–42)

These messages are produced when the parser fails to understand player input.
Each corresponds to a parser error constant defined in `parser.h`
(lines 475–492). The mapping is:

| Parser Error Constant | Value | Miscellany # | Triggered When |
|-----------------------|------:|-------------:|----------------|
| `STUCK_PE` | 1 | 27 | Parser completely failed to parse the input |
| `UPTO_PE` | 2 | 28 (+ 56) | Parser understood part of the input |
| `NUMBER_PE` | 3 | 29 | Expected a number but didn't get one |
| `CANTSEE_PE` | 4 | 30 | Referenced object not in scope |
| `TOOLIT_PE` | 5 | 31 | Input seems too short / incomplete |
| `NOTHELD_PE` | 6 | 32 | Required held object not held |
| `MULTI_PE` | 7 | 33 | Multiple objects not allowed |
| `MMULTI_PE` | 8 | 34 | Multiple objects used twice |
| `VAGUE_PE` | 9 | 35 | Pronoun has no referent |
| `EXCEPT_PE` | 10 | 36 | Excluded something not included |
| `ANIMA_PE` | 11 | 37 | Required animate object |
| `VERB_PE` | 12 | 38 | Unrecognized verb |
| `SCENERY_PE` | 13 | 39 | Scenery not meant to be interacted with |
| `ITGONE_PE` | 14 | 40 | Pronoun referent no longer visible |
| `JUNKAFTER_PE` | 15 | 41 | Unexpected words after valid input |
| `TOOFEW_PE` | 16 | 42 | Fewer objects available than requested |
| `NOTHING_PE` | 17 | 43, 44 | No objects matched "all" or verb |
| `ASKSCOPE_PE` | 18 | — | Handled by scope routine, no message |

### §C.7.2 Disambiguation Messages (45–49, 57)

These messages are produced when the parser needs clarification about which
object the player means.

| # | Context | Description |
|--:|---------|-------------|
| 45 | Creature disambiguation | "Who do you mean, " — followed by a list of options |
| 46 | Object disambiguation | "Which do you mean, " — followed by a list of options |
| 47 | Single-item clarification | "Sorry, you can only have one item here. Which exactly?" |
| 48 | Incomplete creature reference | "Whom do you want [actor] to [command]?" |
| 49 | Incomplete object reference | "What do you want [actor] to [command]?" |
| 57 | Question mark | "?" — appended after disambiguation list |

### §C.7.3 Implicit Action Messages (26, 58)

These messages announce automatic actions the library performs on the player's
behalf before the requested action.

| # | Default Text | Used When |
|--:|-------------|-----------|
| 26 | "(first taking [x1])" | Implicit take of an unheld object |
| 58 | "(first taking [x1] off/out of [x2])" | Implicit take from container/supporter |

### §C.7.4 Score Notification (50)

| # | Default Text | Parameters |
|--:|-------------|-----------|
| 50 | "The score has just gone up/down by [x1] point[s]." | `x1` = score change (positive = up, negative = down) |

This message is displayed automatically when `notify_mode` is on and the score
changes during a turn.

### §C.7.5 System Messages (1–25, 43–44, 51–56, 59–60)

The remaining Miscellany messages cover:

| Group | Numbers | Purpose |
|-------|---------|---------|
| Multi-object limits | 1, 2 | "considering the first sixteen objects only", "Nothing to do" |
| Death/victory | 3, 4, 5, 8 | End-of-game messages and menu |
| Undo | 6, 7, 11, 12, 13 | Undo status messages |
| OOPS | 14, 15, 16 | Correction system messages |
| Darkness | 9, 17 | Dark room descriptions |
| Empty input | 10 | "I beg your pardon?" |
| Self-reference | 18, 19 | "yourself", "As good-looking as ever" |
| AGAIN | 20, 21 | Repeat command messages |
| Command syntax | 22, 23, 24, 25 | Comma/conversation syntax errors |
| Empty "all" | 43, 44 | No objects to act upon |
| Dramatic interrupt | 51 | Command list cut short |
| Menu system | 52, 53 | Menu navigation prompts |
| Comments | 54, 55 | Comment recording feedback |
| Sentence punctuation | 56, 57 | "." and "?" for parser output |
| Ambiguity | 59 | "You'll have to be more specific" |
| NPC speech | 60 | "[NPC] observes that " prefix |

---

## §C.8 Object Listing Messages (`##ListMiscellany`)

The `##ListMiscellany` messages (documented in §C.4.19) are used by the
internal object listing routines (`WriteListFrom` and supporting code in
`verblib.h`, lines 580–700). These messages format parenthetical annotations
that appear after object names in room descriptions and inventory listings.

The messages fall into several categories:

### §C.8.1 State Combination Messages (1–7)

Messages 1–7 handle objects that are both containers and light sources. They
encode combinations of three binary states: providing light, closed, and
empty. The library selects the appropriate message by computing a combined
value (`combo`) from these states.

| Combo | Light | Closed | Empty | Message |
|------:|:-----:|:------:|:-----:|---------|
| 1 | ✓ | — | — | " (providing light)" |
| 2 | — | ✓ | — | " (which is closed)" |
| 3 | ✓ | ✓ | — | " (closed and providing light)" |
| 4 | — | — | ✓ | " (which is empty)" |
| 5 | ✓ | — | ✓ | " (empty and providing light)" |
| 6 | — | ✓ | ✓ | " (which is closed and empty)" |
| 7 | ✓ | ✓ | ✓ | " (closed, empty and providing light)" |

### §C.8.2 Worn and Light Annotations (8–10)

| # | Annotation | Conditions |
|--:|-----------|------------|
| 8 | " (providing light and being worn" | `light` + `worn` |
| 9 | " (providing light" | `light` only |
| 10 | " (being worn" | `worn` only |

These messages begin a parenthetical that may be continued by messages 11–17.

### §C.8.3 Container State Details (11–17)

| # | Text | Usage |
|--:|------|-------|
| 11 | " (which is/are " | Opens a state description |
| 12 | "open" | Container is open with children |
| 13 | "open but empty" | Container is open but empty |
| 14 | "closed" | Closed, not locked |
| 15 | "closed and locked" | Closed and locked |
| 16 | " and empty" | Appended when parenth_flag set |
| 17 | " (which is/are empty)" | Empty without prior parenthetical |

### §C.8.4 Contents Listing Prefixes (18–22)

| # | Text | Style | Object Type |
|--:|------|-------|-------------|
| 18 | " containing " | `ENGLISH_BIT` | General contents |
| 19 | " (on " | `TERSE_BIT` | Supporter (terse) |
| 20 | ", on top of " | verbose | Supporter (verbose) |
| 21 | " (in " | `TERSE_BIT` | Container (terse) |
| 22 | ", inside " | verbose | Container (verbose) |

---

## §C.9 Runtime Error Messages (`##RunTimeError`)

The runtime error messages (documented in §C.5.21) indicate programming
errors or exceeded resource limits. They are printed with `** ... **`
delimiters and are dispatched through `L__M(##RunTimeError, n, p1, p2)` from
the `RunTimeError` routine in `verblib.h` (line 158).

These messages cannot be meaningfully customized via `LibraryMessages` since
they indicate error conditions, but they pass through the same `L__M` dispatch
mechanism for consistency.

Note that message number 6 is skipped in the source code — the numbering goes
directly from 5 to 7.

### §C.9.1 Property and Routine Errors (1–3, 5)

| # | Error | Meaning |
|--:|-------|---------|
| 1 | Preposition not found | Internal parser error; should not occur |
| 2 | Property value not routine or string | A property expected to hold a routine or string holds a plain value |
| 3 | Entry in property list not routine or string | An entry in a property's additive list is not callable/printable |
| 5 | Object has no property | Attempted to access a property not provided by the object |

### §C.9.2 Resource Limit Errors (4, 8, 14)

| # | Error | Meaning |
|--:|-------|---------|
| 4 | Too many timers/daemons | Exceeded `MAX_TIMERS`; increase the constant |
| 8 | Random entry from empty table | Called random table selection on array with zero entries |
| 14 | Output stream 3 buffer overflow | Exceeded the buffer limit for `@output_stream 3` |

### §C.9.3 Object Tree Errors (7, 9, 10, 11, 15)

| # | Error | Meaning |
|--:|-------|---------|
| 7 | Player object lacks `number` property | Cannot switch player to an object without `number` |
| 9 | Invalid direction property number | A direction property number is out of range |
| 10 | Player outside object tree | The player object has no parent (orphaned) |
| 11 | Room has no property | A room is missing a required property |
| 15 | Object moved during LoopWithinObject | An object was moved while iterating over its siblings |

### §C.9.4 Miscellaneous Errors (12, 13, 16)

| # | Error | Meaning |
|--:|-------|---------|
| 12 | Non-existent pronoun in SetPronoun | Attempted to set a pronoun that doesn't exist |
| 13 | Topic token not followed by preposition | Grammar error: a `topic` token must be followed by a preposition |
| 16 | Illegal narrative_voice | The `narrative_voice` property has an unsupported value |

---

## §C.10 Parser Error Constant Cross-Reference

This table provides a quick cross-reference between the parser error constants
defined in `parser.h` (lines 475–492), the `##Miscellany` message number
that each triggers, and the parser line where the error is reported
(`parser.h`, lines 2375–2415).

| Constant | Value | Miscellany # | Default Message | Parser Line |
|----------|------:|-------------:|-----------------|-------------|
| `STUCK_PE` | 1 | 27 | "I didn't understand that sentence." | 2375 |
| `UPTO_PE` | 2 | 28, 56 | "I only understood you as far as wanting to " + "." | 2376–2378 |
| `NUMBER_PE` | 3 | 29 | "I didn't understand that number." | 2381 |
| `CANTSEE_PE` | 4 | 30 | "[Actor] can't see any such thing." | 2382 |
| `TOOLIT_PE` | 5 | 31 | "[Actor] seem[s] to have said too little." | 2383 |
| `NOTHELD_PE` | 6 | 32 | "[Actor] isn't holding that." | 2384 |
| `MULTI_PE` | 7 | 33 | "You can't use multiple objects with that verb." | 2385 |
| `MMULTI_PE` | 8 | 34 | "You can only use multiple objects once on a line." | 2386 |
| `VAGUE_PE` | 9 | 35 | "I'm not sure what ~[pronoun]~ refers to." | 2387 |
| `EXCEPT_PE` | 10 | 36 | "You excepted something not included anyway." | 2388 |
| `ANIMA_PE` | 11 | 37 | "[Actor] can only do that to something animate." | 2389 |
| `VERB_PE` | 12 | 38 | "That's not a verb I recognise." | 2390 |
| `SCENERY_PE` | 13 | 39 | "That's not something you need to refer to..." | 2391 |
| `ITGONE_PE` | 14 | 35 or 40 | Pronoun referent gone (35 if no object, 40 if visible) | 2394–2395 |
| `JUNKAFTER_PE` | 15 | 41 | "I didn't understand the way that finished." | 2397 |
| `TOOFEW_PE` | 16 | 42 | "None/Only [n] of those is/are available." | 2398 |
| `NOTHING_PE` | 17 | 43 or 44 | "Nothing to do" (43) or "There is nothing to [verb]" (44) | 2402–2415 |
| `ASKSCOPE_PE` | 18 | — | No message; handled by `scope=` routine | — |
