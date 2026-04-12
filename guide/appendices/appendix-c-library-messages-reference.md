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
**Default Text** column shows the English-language default from `english.h`.
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

The `L__M` routine is defined in `verblib.h` (lines 3113–3120):

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

The inner `L___M` routine (`verblib.h`, lines 3122–3132) handles the actual
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

**Source:** `english.h:798–801` | **Messages:** 1

These two actions share a single message handler via the `Answer,Ask:` case
label.

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "There is no reply." | — | `verblib.h:2698` (Answer), `verblib.h:2703` (Ask) | Tense-aware |

### §C.3.2 Attack (`##Attack`)

**Source:** `english.h:803–805` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Violence isn't the answer to this one." | — | `verblib.h:2716` | Tense-aware |

### §C.3.3 Blow (`##Blow`)

**Source:** `english.h:806–807` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] can't usefully blow [x1]." | `x1` = noun | `verblib.h:2719` | Tense-aware |

### §C.3.4 Burn (`##Burn`)

**Source:** `english.h:808–813` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "This dangerous act would achieve little." | — | `verblib.h:2723` | Tense-aware |
| 2 | `DecideAgainst()` | — | `verblib.h:2722` | When noun is animate |

### §C.3.5 Buy (`##Buy`)

**Source:** `english.h:814–816` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Nothing is on sale." | — | `verblib.h:2726` | Tense-aware |

### §C.3.6 Climb (`##Climb`)

**Source:** `english.h:817–822` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Climbing [x1] would achieve little." | `x1` = noun | `verblib.h:2730` | Tense-aware |
| 2 | `DecideAgainst()` | — | `verblib.h:2729` | When noun is animate |

### §C.3.7 Close (`##Close`)

**Source:** `english.h:823–832` | **Messages:** 4

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] is not something [actor] can close." | `x1` = noun | `verblib.h:2635` | Tense-aware |
| 2 | "[x1] is already closed." | `x1` = noun | `verblib.h:2636` | |
| 3 | "[Actor] close[s] [x1]." | `x1` = noun | `verblib.h:2640` | Success message; tense-aware |
| 4 | "(first closing [x1])" | `x1` = noun | `verblib.h:1809` | Implicit action |

### §C.3.8 CommandsOff (`##CommandsOff`)

**Source:** `english.h:833–837` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Command recording off.]" | — | `verblib.h:1206` [Z-machine], `verblib.h:1320` [Glulx] | |
| 2 | "[Command recording already off.]" | — | `verblib.h:1315` | [Glulx only] |

### §C.3.9 CommandsOn (`##CommandsOn`)

**Source:** `english.h:839–845` | **Messages:** 4

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Command recording on.]" | — | `verblib.h:1200` [Z-machine], `verblib.h:1311` [Glulx] | |
| 2 | "[Commands are currently replaying.]" | — | `verblib.h:1302` | [Glulx only] |
| 3 | "[Command recording already on.]" | — | `verblib.h:1303` | [Glulx only] |
| 4 | "[Command recording failed.]" | — | `verblib.h:1306`, `verblib.h:1310` | [Glulx only] |

### §C.3.10 CommandsRead (`##CommandsRead`)

**Source:** `english.h:847–854` | **Messages:** 5

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Replaying commands.]" | — | `verblib.h:1212` [Z-machine], `verblib.h:1334` [Glulx] | |
| 2 | "[Commands are already replaying.]" | — | `verblib.h:1325` | [Glulx only] |
| 3 | "[Command replay failed. Command recording is on.]" | — | `verblib.h:1326` | [Glulx only] |
| 4 | "[Command replay failed.]" | — | `verblib.h:1329`, `verblib.h:1333` | [Glulx only] |
| 5 | "[Command replay complete.]" | — | `verblib.h:1316` | [Glulx only]; see also commented-out call at `parser.h:1274` |

### §C.3.11 Consult (`##Consult`)

**Source:** `english.h:856–859` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] discover[s] nothing of interest in [x1]." | `x1` = noun | `verblib.h:2733` | Tense-aware |

### §C.3.12 Cut (`##Cut`)

**Source:** `english.h:860–865` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Cutting [x1] up would achieve little." | `x1` = noun | `verblib.h:2737` | Tense-aware |
| 2 | `DecideAgainst()` | — | `verblib.h:2736` | When noun is animate |

### §C.3.13 Dig (`##Dig`)

**Source:** `english.h:866–868` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Digging would achieve nothing here." | — | `verblib.h:2740` | Tense-aware |

### §C.3.14 Disrobe (`##Disrobe`)

**Source:** `english.h:869–876` | **Messages:** 4

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] isn't wearing [x1]." | `x1` = noun | `verblib.h:2645`, `verblib.h:2646`, `verblib.h:1876` | Tense-aware |
| 2 | "[Actor] take[s] off [x1]." | `x1` = noun | `verblib.h:2650` | Success message; tense-aware |
| 3 | "(first taking [x1] off)" | `x1` = obj | `verblib.h:1849` | Implicit action |
| 4 | "[Actor] will need to remove [noun] first." | — | `verblib.h:1893`, `verblib.h:1922`, `verblib.h:1959` | When `no_implicit_actions` is set; tense-aware |

### §C.3.15 Drink (`##Drink`)

**Source:** `english.h:877–879` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "There's nothing suitable to drink here." | — | `verblib.h:2742` | Tense-aware |

### §C.3.16 Drop (`##Drop`)

**Source:** `english.h:880–893` | **Messages:** 4

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] is already here." | `x1` = noun | `verblib.h:1892`, `verblib.h:1908` | |
| 2 | "[Actor] haven't/hasn't got [x1]." | `x1` = noun | — | Tense-aware |
| 3 | "Dropped." | — | `verblib.h:1902` | Success message |
| 4 | "[Actor] need[s] to take [x1] off/out of [x2] before dropping it." | `x1` = noun, `x2` = parent | `verblib.h:1896` | When `no_implicit_actions` is set; tense-aware |

---

## §C.4 Action Messages (E–L)

### §C.4.1 Eat (`##Eat`)

**Source:** `english.h:894–898` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] is plainly inedible." | `x1` = noun | `verblib.h:2674` | |
| 2 | "[Actor] eat[s] [x1]. Not bad." | `x1` = noun | `verblib.h:2679` | Success; "Not bad" only for player actor; tense-aware |

### §C.4.2 EmptyT (`##EmptyT`)

**Source:** `english.h:899–906` | **Messages:** 4

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] can't contain things." | `x1` = noun or second | `verblib.h:2022`, `verblib.h:2026` | |
| 2 | "[x1] is closed." | `x1` = noun or second | `verblib.h:2023`, `verblib.h:2028` | |
| 3 | "[x1] is empty already." | `x1` = noun | `verblib.h:2032` | |
| 4 | "That wouldn't empty anything." | — | `verblib.h:2020` | Tense-aware |

### §C.4.3 Enter (`##Enter`)

**Source:** `english.h:907–928` | **Messages:** 7

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "But [actor] is already on/in [x1]." | `x1` = noun | `verblib.h:2092` | |
| 2 | "[x1] is not something [actor] can stand on/sit on/lie on/enter." | `x1` = noun, `x2` = verb_word | `verblib.h:2088`, `verblib.h:2093` | `x2` selects phrasing |
| 3 | "[Actor] can't get into the closed [x1]." | `x1` = noun | `verblib.h:2122` | |
| 4 | "[Actor] can only get into something free-standing." | — | `verblib.h:2097` | |
| 5 | "[Actor] get[s] onto/into [x1]." | `x1` = noun | `verblib.h:2126` | Success; tense-aware |
| 6 | "(getting off/out of [x1])" | `x1` = intermediate object | `verblib.h:2102` | Implicit action |
| 7 | "(getting onto/into [x1])" | `x1` = intermediate object | `verblib.h:2113` | Implicit action |

### §C.4.4 Examine (`##Examine`)

**Source:** `english.h:930–938` | **Messages:** 3

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Darkness, noun. An absence of light to see by." | — | `verblib.h:2520` | In darkness |
| 2 | "[Actor] see[s] nothing special about [x1]." | `x1` = noun | `verblib.h:2527` | When no description; tense-aware |
| 3 | "[x1] is currently switched on/off." | `x1` = noun | `verblib.h:2526`, `verblib.h:2530` | For switchable objects; tense-aware |

### §C.4.5 Exit (`##Exit`)

**Source:** `english.h:939–954` | **Messages:** 6

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "But [actor] isn't in anything at the moment." | — | `verblib.h:2145` | |
| 2 | "[Actor] can't get out of the closed [x1]." | `x1` = parent | `verblib.h:2148` | |
| 3 | "[Actor] get[s] off/out of [x1]." | `x1` = parent | `verblib.h:2159` | Success; tense-aware |
| 4 | "[Actor] isn't on/in [x1]." | `x1` = noun | `verblib.h:2137` | |
| 5 | "(first getting off/out of [x1])" | `x1` = obj | `verblib.h:1794` | Implicit action |
| 6 | "[Actor] stand[s] up." | — | `verblib.h:2141` | Tense-aware |

### §C.4.6 Fill (`##Fill`)

**Source:** `english.h:955–962` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "There isn't anything obvious with which to fill [x1]." | `x1` = noun | `verblib.h:2745` | Tense-aware |
| 2 | "Filling [x1] from [x2] wouldn't make sense." | `x1` = noun, `x2` = second | `verblib.h:2746` | Tense-aware |

### §C.4.7 FullScore (`##FullScore`)

**Source:** `english.h:963–969` | **Messages:** 4

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "The score is/was made up as follows:" | — | `verblib.h:1485` | Adapts to `deadflag` |
| 2 | "finding sundry items" | — | `verblib.h:1494` | Task score line |
| 3 | "visiting various places" | — | `verblib.h:1498` | Places score line |
| 4 | "total (out of [MAX_SCORE])" | — | `verblib.h:1500` | Final score line |

### §C.4.8 GetOff (`##GetOff`)

**Source:** `english.h:970–971` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "But [actor] isn't on [x1] at the moment." | `x1` = noun | `verblib.h:2132` | Tense-aware |

### §C.4.9 Give (`##Give`)

**Source:** `english.h:972–981` | **Messages:** 4

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] isn't holding [x1]." | `x1` = noun | `verblib.h:2058` | Tense-aware |
| 2 | "[Actor] juggle[s] [x1] for a while, but don't achieve much." | `x1` = noun | `verblib.h:2059` | When giving to self; tense-aware |
| 3 | "[x1] don't seem interested." | `x1` = second | `verblib.h:2067` | Default NPC refusal |
| 4 | "[Actor] hand[s] over [x1]." | `x1` = noun | `verblib.h:2063` | Success; tense-aware |

### §C.4.10 Go (`##Go`)

**Source:** `english.h:982–992` | **Messages:** 7

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] will have to get off/out of [x1] first." | `x1` = container/supporter | `verblib.h:2183` | Tense-aware |
| 2 | "[Actor] can't go that way." | — | `verblib.h:2210`, `verblib.h:2215` | Tense-aware |
| 3 | "[Actor] is unable to climb [x1]." | `x1` = door object | `verblib.h:2217` | Going up through a door |
| 4 | "[Actor] is unable to descend by [x1]." | `x1` = door object | `verblib.h:2218` | Going down through a door |
| 5 | "[Actor] can't, since [x1] is in the way." | `x1` = door object | `verblib.h:2219` | Door blocking other direction |
| 6 | "[Actor] can't, since [x1] lead[s] nowhere." | `x1` = door object | `verblib.h:2222` | Door with no destination; tense-aware |
| 7 | "[Actor] depart[s]." | — | `verblib.h:2232` | NPC going; tense-aware |

### §C.4.11 Insert (`##Insert`)

**Source:** `english.h:993–1010` | **Messages:** 9

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] need[s] to be holding [x1] before putting it into something." | `x1` = noun | `verblib.h:1947` | Tense-aware |
| 2 | "[x1] can't contain things." | `x1` = second (or noun in parser) | `verblib.h:1958`, `parser.h:2403` | |
| 3 | "[x1] is closed." | `x1` = second | `verblib.h:1956` | |
| 4 | "[Actor] will need to take [x1] off first." | `x1` = noun | — | For worn items; tense-aware |
| 5 | "[Actor] can't put something inside itself." | — | `verblib.h:1949` | |
| 6 | "(first taking [x1] off)" | `x1` = noun | — | Implicit action |
| 7 | "There is no more room in [x1]." | `x1` = second | `verblib.h:1964` | Tense-aware |
| 8 | "Done." | — | `verblib.h:1976` | Multi-object success |
| 9 | "[Actor] put[s] [x1] into [x2]." | `x1` = noun, `x2` = second | `verblib.h:1977` | Success; tense-aware |

### §C.4.12 Inv (`##Inv`)

**Source:** `english.h:1011–1016` | **Messages:** 4

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] is carrying nothing." | — | `verblib.h:1524` | |
| 2 | "[Actor] is carrying" | — | `verblib.h:1528` | Inventory header (no newline) |
| 3 | ":" | — | `verblib.h:1529` | After header when `NEWLINE_BIT` set |
| 4 | "." | — | `verblib.h:1532` | After inventory list when `ENGLISH_BIT` set |

### §C.4.13 Jump (`##Jump`)

**Source:** `english.h:1017` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] jump[s] on the spot, fruitlessly." | — | `verblib.h:2749` | Tense-aware |

### §C.4.14 JumpIn (`##JumpIn`)

**Source:** `english.h:1018–1023` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Jumping in [x1] would achieve nothing here." | `x1` = noun | `verblib.h:2754` | Tense-aware |
| 2 | `DecideAgainst()` | — | `verblib.h:2752` | When noun is animate |

### §C.4.15 JumpOn (`##JumpOn`)

**Source:** `english.h:1024–1029` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Jumping upon [x1] would achieve nothing here." | `x1` = noun | `verblib.h:2760` | Tense-aware |
| 2 | `DecideAgainst()` | — | `verblib.h:2758` | When noun is animate |

### §C.4.16 JumpOver (`##JumpOver`)

**Source:** `english.h:1030–1035` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Jumping over [x1] would achieve nothing here." | `x1` = noun | `verblib.h:2765` | Tense-aware |
| 2 | `DecideAgainst()` | — | `verblib.h:2764` | When noun is animate |

### §C.4.17 Kiss (`##Kiss`)

**Source:** `english.h:1036` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Keep your mind on the game." | — | `verblib.h:2772` | |

### §C.4.18 Listen (`##Listen`)

**Source:** `english.h:1037` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] hear[s] nothing unexpected." | — | `verblib.h:2775` | Tense-aware |

### §C.4.19 ListMiscellany (`##ListMiscellany`)

**Source:** `english.h:1038–1061` | **Messages:** 22

These messages are used internally by the object listing system
(`WriteListFrom` and related routines in `verblib.h`) to format parenthetical
annotations and container/supporter contents in room descriptions and
inventory listings.

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | " (providing light)" | — | `verblib.h:619` | Light source annotation |
| 2 | " (which is/are closed)" | `x1` = obj | `verblib.h:619` | Closed container |
| 3 | " (closed and providing light)" | — | `verblib.h:619` | Combined state |
| 4 | " (which is/are empty)" | `x1` = obj | `verblib.h:619` | Empty container |
| 5 | " (empty and providing light)" | — | `verblib.h:619` | Combined state |
| 6 | " (which is/are closed and empty)" | `x1` = obj | `verblib.h:619` | Combined state |
| 7 | " (closed, empty and providing light)" | — | `verblib.h:619` | Combined state |
| 8 | " (providing light and being worn" | `x1` = obj | `verblib.h:626` | Light + worn |
| 9 | " (providing light" | `x1` = obj | `verblib.h:628` | Light only |
| 10 | " (being worn" | `x1` = obj | `verblib.h:629` | Worn only |
| 11 | " (which is/are " | `x1` = obj | `verblib.h:635` | Open container prefix |
| 12 | "open" | — | `verblib.h:637` | Open + has children |
| 13 | "open but empty" | — | `verblib.h:638` | Open + no children |
| 14 | "closed" | — | `verblib.h:641` | Closed, not locked |
| 15 | "closed and locked" | — | `verblib.h:640` | Closed + locked |
| 16 | " and empty" | — | `verblib.h:646` | Appended when empty with parenth |
| 17 | " (which is/are empty)" | `x1` = obj | `verblib.h:647` | Empty without parenth |
| 18 | " containing " | — | `verblib.h:660` | Before listing contents |
| 19 | " (on " | — | `verblib.h:667` | Terse supporter prefix |
| 20 | ", on top of " | — | `verblib.h:668` | Verbose supporter prefix |
| 21 | " (in " | — | `verblib.h:676` | Terse container prefix |
| 22 | ", inside " | — | `verblib.h:677` | Verbose container prefix |

### §C.4.20 LMode1 (`##LMode1`)

**Source:** `english.h:1062–1065` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Story] is now in its ~brief~ printing mode..." | — | `verblib.h:2397` | Brief mode |

### §C.4.21 LMode2 (`##LMode2`)

**Source:** `english.h:1066–1069` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Story] is now in its ~verbose~ mode..." | — | `verblib.h:2399` | Verbose mode |

### §C.4.22 LMode3 (`##LMode3`)

**Source:** `english.h:1070–1073` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Story] is now in its ~superbrief~ mode..." | — | `verblib.h:2401` | Superbrief mode |

### §C.4.23 Lock (`##Lock`)

**Source:** `english.h:1074–1083` | **Messages:** 5

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] don't seem to be something [actor] can lock." | `x1` = noun | `verblib.h:2581` | Tense-aware |
| 2 | "[x1] is locked at the moment." | `x1` = noun | `verblib.h:2582` | |
| 3 | "[Actor] will first have to close [x1]." | `x1` = noun | `verblib.h:2583` | |
| 4 | "[x1] don't seem to fit the lock." | `x1` = second | `verblib.h:2587` | Wrong key |
| 5 | "[Actor] lock[s] [x1]." | `x1` = noun | `verblib.h:2591` | Success; tense-aware |

### §C.4.24 Look (`##Look`)

**Source:** `english.h:1084–1105` | **Messages:** 7

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | " (on [x1])" | `x1` = supporter | `verblib.h:2479` | Room header annotation |
| 2 | " (in [x1])" | `x1` = container | `verblib.h:2480` | Room header annotation |
| 3 | " (as [x1])" | `x1` = player | `verblib.h:2482` | When `print_player_flag` is 1 |
| 4 | "On [x1] [list of items]." | `x1` = supporter | `verblib.h:2271` | Objects on supporter in room |
| 5 | "[Actor] can also see [list]." | `x1` = location/container | `verblib.h:2380` | "also see" variant |
| 6 | "[Actor] can see [list]." | `x1` = location/container | `verblib.h:2382` | "can see" variant |
| 7 | "[Actor] see[s] nothing unexpected in that direction." | `x1` = compass direction | `verblib.h:2527` (via `english.h:37`) | Tense-aware; also used by `CompassDirection.description` in `english.h:37` |

### §C.4.25 LookUnder (`##LookUnder`)

**Source:** `english.h:1106–1111` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "But it's dark." | — | `verblib.h:2535` | In darkness; tense-aware |
| 2 | "[Actor] find[s] nothing of interest." | — | `verblib.h:2536` | Tense-aware |

---

## §C.5 Action Messages (M–R)

### §C.5.1 Mild (`##Mild`)

**Source:** `english.h:1112` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Quite." | — | `verblib.h:2777` | Response to mild language |

### §C.5.2 Miscellany (`##Miscellany`)

**Source:** `english.h:1113–1221` | **Messages:** 60

The `##Miscellany` pseudo-action is a catch-all for parser messages, system
messages, and other text that does not belong to any specific game action.
These messages are called from throughout `parser.h` and `verblib.h`. For a
thematic breakdown, see §C.7.

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "(considering the first sixteen objects only)" | — | `parser.h:5210` | Multi-object limit |
| 2 | "Nothing to do." | — | `parser.h:5205` | Empty "all" list |
| 3 | " [Player] have/has died " | — | `parser.h:5410` | Death message fragment |
| 4 | " [Player] have/has won " | — | `parser.h:5411` | Victory message fragment |
| 5 | "Would you like to RESTART, RESTORE..., or QUIT?" | — | `parser.h:5507` | End-of-game menu; conditional `UNDO`, `FULL`, `AMUSING` |
| 6 | "[Your interpreter does not provide ~undo~. Sorry!]" | — | `parser.h:1501` [Z-machine], `parser.h:1510` [Glulx] | |
| 7 | "~Undo~ failed. [Not all interpreters provide it.]" | — | `parser.h:1502`, `parser.h:1509` | [Z-machine] |
| 7 | "[You cannot ~undo~ any further.]" | — | `parser.h:1502`, `parser.h:1509` | [Glulx] — same number, different text |
| 8 | "Please give one of the answers above." | — | `parser.h:5557` | Invalid end-of-game input |
| 9 | "It is now pitch dark in here." | — | `parser.h:5791` | Darkness notification; tense-aware |
| 10 | "I beg your pardon?" | — | `parser.h:1353` | Empty input |
| 11 | "[You can't ~undo~ what hasn't been done!]" | — | `parser.h:1500` | Undo at start of game |
| 12 | "[Can't ~undo~ twice in succession. Sorry!]" | — | — | Consecutive undo attempt |
| 13 | "[Previous turn undone.]" | — | `parser.h:1410` | Successful undo |
| 14 | "Sorry, that can't be corrected." | — | `parser.h:1420` | Failed OOPS |
| 15 | "Think nothing of it." | — | `parser.h:1424` | OOPS with nothing to correct |
| 16 | "~Oops~ can only correct a single word." | — | `parser.h:1428` | Multiple-word OOPS |
| 17 | "It is pitch dark, and [actor] can't see a thing." | — | `parser.h:817` | Darkness room description; tense-aware |
| 18 | "yourself" | — | — | Pronoun text for self-reference |
| 19 | "As good-looking as ever." | — | `parser.h:824` | Examining self |
| 20 | "To repeat a command like ~frog, jump~, just say ~again~..." | — | `parser.h:1674`, `parser.h:1865` | AGAIN misuse |
| 21 | "[Actor] can hardly repeat that." | — | `parser.h:1678` | Can't repeat; tense-aware |
| 22 | "[Actor] can't begin with a comma." | — | `parser.h:1815` | Leading comma; tense-aware |
| 23 | "[Actor] seem[s] to want to talk to someone, but I can't see whom." | — | `parser.h:1828` | Tense-aware |
| 24 | "[Actor] can't talk to [x1]." | `x1` = inanimate object | `parser.h:1839` | Tense-aware |
| 25 | "To talk to someone, try ~someone, hello~ or some such." | — | `parser.h:1847` | |
| 26 | "(first taking [x1])" | `x1` = object | `parser.h:2285`, `verblib.h:1779` | Implicit take |
| 27 | "I didn't understand that sentence." | — | `parser.h:2375` | `STUCK_PE` parser error |
| 28 | "I only understood you as far as wanting to " | — | `parser.h:2376` | `UPTO_PE` — partial parse; followed by msg 56 |
| 29 | "I didn't understand that number." | — | `parser.h:2381` | `NUMBER_PE` |
| 30 | "[Actor] can't see any such thing." | — | `parser.h:2382` | `CANTSEE_PE`; tense-aware |
| 31 | "[Actor] seem[s] to have said too little." | — | `parser.h:2383` | `TOOLIT_PE`; tense-aware |
| 32 | "[Actor] isn't holding that." | `x1` = not_holding | `parser.h:2384` | `NOTHELD_PE` |
| 33 | "You can't use multiple objects with that verb." | — | `parser.h:2385` | `MULTI_PE` |
| 34 | "You can only use multiple objects once on a line." | — | `parser.h:2386` | `MMULTI_PE` |
| 35 | "I'm not sure what ~[x1]~ refers to." | `x1` = pronoun_word (address) | `parser.h:2387`, `parser.h:2394` | `VAGUE_PE` |
| 36 | "You excepted something not included anyway." | — | `parser.h:2388` | `EXCEPT_PE` |
| 37 | "[Actor] can only do that to something animate." | — | `parser.h:2389` | `ANIMA_PE`; tense-aware |
| 38 | "That's not a verb I recognise." | — | `parser.h:2390` | `VERB_PE`; US variant: "recognize" (`DIALECT_US`) |
| 39 | "That's not something you need to refer to in the course of this game." | — | `parser.h:2391` | `SCENERY_PE` |
| 40 | "[Actor] can't see ~[x1]~ ([x2]) at the moment." | `x1` = pronoun_word (address), `x2` = pronoun_obj | `parser.h:2395` | `ITGONE_PE`; tense-aware |
| 41 | "I didn't understand the way that finished." | — | `parser.h:2397` | `JUNKAFTER_PE` |
| 42 | "None/Only [x1] of those is/are available." | `x1` = multi_had (count) | `parser.h:2398` | `TOOFEW_PE` |
| 43 | "Nothing to do." | — | `parser.h:2409` | `NOTHING_PE` with `multi_wanted == 100` |
| 44 | "There is nothing to [x1]." | `x1` = verb_word (address) | `parser.h:2402`, `parser.h:2413`, `parser.h:2415` | `NOTHING_PE`; tense-aware |
| 45 | "Who do you mean, " | — | `parser.h:3348` | Disambiguation (creature) |
| 46 | "Which do you mean, " | — | `parser.h:3349` | Disambiguation (thing) |
| 47 | "Sorry, you can only have one item here. Which exactly?" | — | `parser.h:3387` | Single-object disambiguation |
| 48 | "Whom do you want [x1] to [command]?" | `x1` = actor | `parser.h:3220` | Incomplete creature token |
| 49 | "What do you want [x1] to [command]?" | `x1` = actor | `parser.h:3221` | Incomplete object token |
| 50 | "The score has just gone up/down by [x1] point[s]." | `x1` = score change | `parser.h:5770` | Score notification |
| 51 | "(Since something dramatic has happened, your list of commands has been cut short.)" | — | `parser.h:5216` | Multi-action interruption |
| 52 | "Type a number from 1 to [x1], 0 to redisplay or press ENTER." | `x1` = line count | `verblib.h:781` | Menu prompt |
| 53 | "[Please press SPACE.]" | — | `verblib.h:913`, `verblib.h:1023` | Menu/page pause |
| 54 | "[Comment recorded.]" | — | `parser.h:1366` [Z-machine], `parser.h:1369` [Glulx] | When transcript/commands active |
| 55 | "[Comment NOT recorded.]" | — | `parser.h:1367` [Z-machine], `parser.h:1370` [Glulx] | When no transcript active |
| 56 | "." | — | `parser.h:2378` | Sentence terminator after msg 28 |
| 57 | "?" | — | `parser.h:3361` | Question mark for disambiguation |
| 58 | "(first taking [x1] off/out of [x2])" | `x1` = object, `x2` = container/supporter | `verblib.h:1778`, `verblib.h:1897`, `verblib.h:2660` | Implicit take from container/supporter |
| 59 | "You'll have to be more specific." | — | `parser.h:2412` | Ambiguous "take all" |
| 60 | "[x1] observes that " | `x1` = NPC | — | NPC speech prefix |

### §C.5.3 No / Yes (`##No`, `##Yes`)

**Source:** `english.h:1222` | **Messages:** 1

These two actions share a single message handler via the `No,Yes:` case label.

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "That was a rhetorical question." | — | `verblib.h:2779` (No), `verblib.h:2912` (Yes) | |

### §C.5.4 NotifyOff (`##NotifyOff`)

**Source:** `english.h:1223–1224` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Score notification off." | — | `verblib.h:1340` | |

### §C.5.5 NotifyOn (`##NotifyOn`)

**Source:** `english.h:1225` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Score notification on." | — | `verblib.h:1339` | |

### §C.5.6 Objects (`##Objects`)

**Source:** `english.h:1226–1237` | **Messages:** 10

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] have/has handled" | — | `verblib.h:1396` | List header |
| 2 | "[Actor] have/has handled nothing." | — | `verblib.h:1394` | Empty list |
| 3 | " (worn)" | `x1` = location, `x2` = object | `verblib.h:1407` | Object is worn |
| 4 | " (held)" | `x1` = location, `x2` = object | `verblib.h:1408` | Object is held |
| 5 | " (given away)" | `x1` = location, `x2` = object | `verblib.h:1411` | Object is with an animate |
| 6 | " (in [x1])" | `x1` = location (name), `x2` = object | `verblib.h:1412` | Object is in a visited room |
| 7 | " (in [x1])" | `x1` = location (the), `x2` = object | `verblib.h:1415` | Object is in an enterable |
| 8 | " (inside [x1])" | `x1` = location (the), `x2` = object | `verblib.h:1413` | Object is in a container |
| 9 | " (on [x1])" | `x1` = location (the), `x2` = object | `verblib.h:1414` | Object is on a supporter |
| 10 | " (lost)" | `x1` = location, `x2` = object | `verblib.h:1417` | Object location unknown |

### §C.5.7 Open (`##Open`)

**Source:** `english.h:1238–1251` | **Messages:** 6

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] is not something [actor] can open." | `x1` = noun | `verblib.h:2616` | Not openable; tense-aware |
| 2 | "[x1] seem[s] to be locked." | `x1` = noun | `verblib.h:2617` | Tense-aware |
| 3 | "[x1] is already open." | `x1` = noun | `verblib.h:2618` | |
| 4 | "[Actor] open[s] [x1], revealing [contents/nothing]." | `x1` = noun | `verblib.h:2628` | Success with contents; tense-aware |
| 5 | "[Actor] open[s] [x1]." | `x1` = noun | `verblib.h:2624`, `verblib.h:2630` | Success without contents; tense-aware |
| 6 | "(first opening [x1])" | `x1` = noun | `verblib.h:1825` | Implicit action |

### §C.5.8 Order (`##Order`)

**Source:** `english.h:1252` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] has better things to do." | `x1` = noun (NPC) | `verblib.h:2708`, `verblib.h:2711`, `parser.h:5297` | Default NPC refusal |

### §C.5.9 Places (`##Places`)

**Source:** `english.h:1253–1257` | **Messages:** 3

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] have/has visited" | — | `verblib.h:1357` | List header |
| 2 | "." | — | `verblib.h:1369` | List terminator |
| 3 | "[Actor] have/has visited nothing." | — | `verblib.h:1355` | Empty list |

### §C.5.10 Pray (`##Pray`)

**Source:** `english.h:1258–1260` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Nothing practical results from [actor's] prayer." | — | `verblib.h:2781` | Tense-aware |

### §C.5.11 Prompt (`##Prompt`)

**Source:** `english.h:1261` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "^>" | — | `parser.h:1342`, `parser.h:5511` | Command prompt; `^` is a newline |

### §C.5.12 Pronouns (`##Pronouns`)

**Source:** `english.h:1262–1268` | **Messages:** 5

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "At the moment, " | — | `parser.h:4998` | Pronoun list header |
| 2 | "means " | — | `parser.h:5010`, `parser.h:5018` | After pronoun word |
| 3 | "is unset" | — | `parser.h:5008` | Unassigned pronoun |
| 4 | "no pronouns are known to the game." | — | `parser.h:5003` | No pronouns defined |
| 5 | "." | — | `parser.h:5022` | Pronoun list terminator |

### §C.5.13 Pull / Push / Turn (`##Pull`, `##Push`, `##Turn`)

**Source:** `english.h:1269–1287` | **Messages:** 6

These three actions share a single message handler via the `Pull,Push,Turn:`
case label.

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Punishing [oneself] that way isn't likely to help matters." | — | `verblib.h:2785` (Pull), `verblib.h:2795` (Push), `verblib.h:2880` (Turn) | When noun is player; tense-aware |
| 2 | "[x1] is fixed in place." | `x1` = noun | `verblib.h:2787` (Pull), `verblib.h:2797` (Push), `verblib.h:2882` (Turn) | Static object |
| 3 | "[Actor] is unable to." | — | `verblib.h:2788` (Pull), `verblib.h:2798` (Push), `verblib.h:2883` (Turn) | Scenery object |
| 4 | "Nothing obvious happens." | — | `verblib.h:2790` (Pull), `verblib.h:2800` (Push), `verblib.h:2885` (Turn) | Default success; tense-aware |
| 5 | "That would be less than courteous." | — | `verblib.h:2789` (Pull), `verblib.h:2796`, `verblib.h:2799` (Push), `verblib.h:2881`, `verblib.h:2884` (Turn) | Animate object; tense-aware |
| 6 | `DecideAgainst()` | — | `verblib.h:2786` (Pull) | When noun is actor (not player) |

### §C.5.14 PushDir (`##PushDir`)

**Source:** `english.h:1289–1299` | **Messages:** 3

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "That really wouldn't serve any purpose." | — | `verblib.h:2803` | Tense-aware |
| 2 | "That's not a direction." | — | `verblib.h:2688` | Not a compass direction; tense-aware |
| 3 | "Not that way [actor] can't." | — | `verblib.h:2689` | Up/down not allowed; tense-aware |

### §C.5.15 PutOn (`##PutOn`)

**Source:** `english.h:1300–1316` | **Messages:** 8

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] need[s] to be holding [x1] before putting it on something." | `x1` = noun | `verblib.h:1909` | Tense-aware |
| 2 | "[Actor] can't put something on top of itself." | — | `verblib.h:1912` | Tense-aware |
| 3 | "Putting things on [x1] would achieve nothing." | `x1` = second | `verblib.h:1920` | Not a supporter; tense-aware |
| 4 | "[Actor] lack[s] the dexterity." | — | `verblib.h:1891` | When noun is actor; tense-aware |
| 5 | "(first taking [x1] off)" | `x1` = noun | — | Implicit action |
| 6 | "There is no more room on [x1]." | `x1` = second | `verblib.h:1927` | Full supporter; tense-aware |
| 7 | "Done." | — | `verblib.h:1939` | Multi-object success |
| 8 | "[Actor] put[s] [x1] on [x2]." | `x1` = noun, `x2` = second | `verblib.h:1940` | Success; tense-aware |

### §C.5.16 Quit (`##Quit`)

**Source:** `english.h:1317–1320` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Please answer yes or no." | — | `verblib.h:1116` [Z-machine], `verblib.h:1218` [Glulx] | Yes/no prompt |
| 2 | "Are you sure you want to quit? " | — | `verblib.h:1123` [Z-machine], `verblib.h:1218` [Glulx] | Confirmation |

### §C.5.17 Remove (`##Remove`)

**Source:** `english.h:1321–1328` | **Messages:** 4

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] is unfortunately closed." | `x1` = container | `verblib.h:1870` | |
| 2 | "But [x1] isn't there now." | `x1` = noun | `verblib.h:1879` | Tense-aware |
| 3 | "Removed." | — | `verblib.h:1887` | Success |
| 4 | "But [x1] isn't in or on anything." | `x1` = noun | `verblib.h:1878` | Not inside container/supporter; tense-aware |

### §C.5.18 Restart (`##Restart`)

**Source:** `english.h:1329–1332` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Are you sure you want to restart? " | — | `verblib.h:1128` [Z-machine], `verblib.h:1223` [Glulx] | Confirmation |
| 2 | "Failed." | — | `verblib.h:1129` [Z-machine], `verblib.h:1224` [Glulx] | Restart failed |

### §C.5.19 Restore (`##Restore`)

**Source:** `english.h:1333–1336` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Restore failed." | — | `verblib.h:1134` [Z-machine], `verblib.h:1237` [Glulx] | |
| 2 | "Ok." | — | `verblib.h:1139`, `verblib.h:1157` [Z-machine], `verblib.h:1258` [Glulx] | Restore succeeded |

### §C.5.20 Rub (`##Rub`)

**Source:** `english.h:1337–1341` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] achieve[s] nothing by this." | — | `verblib.h:2808` | Tense-aware |
| 2 | `DecideAgainst()` | — | `verblib.h:2807` | When noun is animate |

### §C.5.21 RunTimeError (`##RunTimeError`)

**Source:** `english.h:1342–1371` | **Messages:** 16

These are internal runtime error diagnostics, printed with `** ... **`
delimiters. They indicate programming errors or exceeded limits. For a
complete discussion, see §C.9.

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Preposition not found (this should not occur)" | — | — | Internal error |
| 2 | "Property value not routine or string: ~[x2]~ of ~[x1]~" | `x1` = object, `x2` = property | `verblib.h:158` | |
| 3 | "Entry in property list not routine or string: ~[x2]~ list of ~[x1]~" | `x1` = object, `x2` = property | `verblib.h:158` | |
| 4 | "Too many timers/daemons are active simultaneously. The limit is MAX_TIMERS (currently [MAX_TIMERS])" | — | `verblib.h:158` | |
| 5 | "Object ~[x1]~ has no ~[x2]~ property" | `x1` = object, `x2` = property | `verblib.h:158` | |
| 7 | "Object ~[x1]~ can only be used as a player object if it has the ~number~ property" | `x1` = object | `verblib.h:158` | Note: number 6 is skipped |
| 8 | "Attempt to take random entry from an empty table array" | — | `verblib.h:158` | |
| 9 | "[x1] is not a valid direction property number" | `x1` = number | `verblib.h:158` | |
| 10 | "The player-object is outside the object tree" | — | `verblib.h:158` | |
| 11 | "The room ~[x1]~ has no ~[x2]~ property" | `x1` = room, `x2` = property | `verblib.h:158` | |
| 12 | "Tried to set a non-existent pronoun using SetPronoun" | — | `verblib.h:158` | |
| 13 | "A 'topic' token can only be followed by a preposition" | — | `verblib.h:158` | |
| 14 | "Overflowed buffer limit of [x1] using '@@64output_stream 3' [x2]" | `x1` = limit, `x2` = context string | `verblib.h:158` | |
| 15 | "LoopWithinObject broken because [x1] was moved while the loop passed through it." | `x1` = object | `verblib.h:158` | |
| 16 | "Attempt to use illegal narrative_voice of [x1]." | `x1` = voice value | `verblib.h:158` | |

---

## §C.6 Action Messages (S–Z)

### §C.6.1 Save (`##Save`)

**Source:** `english.h:1372–1375` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Save failed." | — | `verblib.h:1146`, `verblib.h:1161` [Z-machine], `verblib.h:1269` [Glulx] | |
| 2 | "Ok." | — | `verblib.h:1151`, `verblib.h:1166` [Z-machine], `verblib.h:1266` [Glulx] | Save succeeded |

### §C.6.2 Score (`##Score`)

**Source:** `english.h:1376–1382` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "You have so far scored [score] out of a possible [MAX_SCORE], in [turns] turn[s]." | — | `verblib.h:1450` | Adapts to `deadflag` ("In that game you scored...") |
| 2 | "There is no score in this story." | — | `verblib.h:1447` | When `NO_SCORE` is defined |

### §C.6.3 ScriptOff (`##ScriptOff`)

**Source:** `english.h:1383–1387` | **Messages:** 3

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Transcripting is already off." | — | `verblib.h:1190` [Z-machine], `verblib.h:1294` [Glulx] | |
| 2 | "End of transcript." | — | `verblib.h:1191` [Z-machine], `verblib.h:1295` [Glulx] | |
| 3 | "Attempt to end transcript failed." | — | `verblib.h:1193` [Z-machine] | [Z-machine only] |

### §C.6.4 ScriptOn (`##ScriptOn`)

**Source:** `english.h:1388–1392` | **Messages:** 3

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Transcripting is already on." | — | `verblib.h:1181` [Z-machine], `verblib.h:1279` [Glulx] | |
| 2 | "Start of a transcript of [version info]" | — | `verblib.h:1184` [Z-machine], `verblib.h:1287` [Glulx] | Calls `VersionSub()` |
| 3 | "Attempt to begin transcript failed." | — | `verblib.h:1183` [Z-machine], `verblib.h:1290` [Glulx] | |

### §C.6.5 Search (`##Search`)

**Source:** `english.h:1393–1410` | **Messages:** 7

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "But it's dark." | — | `verblib.h:2545` | In darkness; tense-aware |
| 2 | "There is nothing on [x1]." | `x1` = noun | `verblib.h:2549` | Empty supporter; tense-aware |
| 3 | "On [x1] [list of items]." | `x1` = noun | `verblib.h:2550` | Supporter with contents |
| 4 | "[Actor] find[s] nothing of interest." | — | `verblib.h:2552` | Not a container; tense-aware |
| 5 | "[Actor] can't see inside, since [x1] is/are closed." | `x1` = noun | `verblib.h:2525`, `verblib.h:2553` | Closed container; tense-aware |
| 6 | "[x1] is/are empty." | `x1` = noun | `verblib.h:2556`, `parser.h:2405` | Empty container |
| 7 | "In [x1] [list of items]." | `x1` = noun | `verblib.h:2557` | Container with contents |

### §C.6.6 Set (`##Set`)

**Source:** `english.h:1412` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] can't set [x1]." | `x1` = noun | `verblib.h:2811` | Tense-aware |

### §C.6.7 SetTo (`##SetTo`)

**Source:** `english.h:1413` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] can't set [x1] to anything." | `x1` = noun | `verblib.h:2813` | Tense-aware |

### §C.6.8 Show (`##Show`)

**Source:** `english.h:1414–1417` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] isn't holding [x1]." | `x1` = noun | `verblib.h:2073` | Tense-aware |
| 2 | "[x1] is unimpressed." | `x1` = second | `verblib.h:2076` | |

### §C.6.9 Sing (`##Sing`)

**Source:** `english.h:1418–1420` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor's] singing is abominable." | — | `verblib.h:2815` | Tense-aware |

### §C.6.10 Sleep (`##Sleep`)

**Source:** `english.h:1421` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] isn't feeling especially drowsy." | — | `verblib.h:2817` | Tense-aware |

### §C.6.11 Smell (`##Smell`)

**Source:** `english.h:1422–1425` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] smell[s] nothing unexpected." | — | `verblib.h:2821` | Tense-aware |
| 2 | `DecideAgainst()` | — | `verblib.h:2820` | When noun is animate |

### §C.6.12 Sorry (`##Sorry`)

**Source:** `english.h:1426–1430` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Oh, don't apologise." | — | `verblib.h:2824` | US variant: "apologize" (`DIALECT_US`) |

### §C.6.13 Squeeze (`##Squeeze`)

**Source:** `english.h:1431–1434` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | `DecideAgainst()` | — | `verblib.h:2828` | When noun is animate (and not player) |
| 2 | "[Actor] achieve[s] nothing by this." | — | `verblib.h:2829` | Tense-aware |

### §C.6.14 Strong (`##Strong`)

**Source:** `english.h:1435–1437` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Real adventurers do not use such language." | — | `verblib.h:2832` | Tense-aware |

### §C.6.15 Swim (`##Swim`)

**Source:** `english.h:1438–1440` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "There's not enough water to swim in." | — | `verblib.h:2834` | Tense-aware |

### §C.6.16 Swing (`##Swing`)

**Source:** `english.h:1441–1443` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "There's nothing sensible to swing here." | — | `verblib.h:2836` | Tense-aware |

### §C.6.17 SwitchOff (`##SwitchOff`)

**Source:** `english.h:1444–1451` | **Messages:** 3

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] is not something [actor] can switch." | `x1` = noun | `verblib.h:2606` | Not switchable; tense-aware |
| 2 | "[x1] is already off." | `x1` = noun | `verblib.h:2607` | |
| 3 | "[Actor] switch[es] [x1] off." | `x1` = noun | `verblib.h:2611` | Success; tense-aware |

### §C.6.18 SwitchOn (`##SwitchOn`)

**Source:** `english.h:1452–1459` | **Messages:** 3

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] is not something [actor] can switch." | `x1` = noun | `verblib.h:2596` | Not switchable; tense-aware |
| 2 | "[x1] is already on." | `x1` = noun | `verblib.h:2597` | |
| 3 | "[Actor] switch[es] [x1] on." | `x1` = noun | `verblib.h:2601` | Success; tense-aware |

### §C.6.19 Take (`##Take`)

**Source:** `english.h:1461–1480` | **Messages:** 13

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Taken." | — | `verblib.h:1865` | Success |
| 2 | "[Actor] is always self-possessed." | — | `verblib.h:1654` | Taking self |
| 3 | "I don't suppose [x1] would care for that." | `x1` = noun (animate) | `verblib.h:1655` | Taking animate; tense-aware |
| 4 | "[Actor] will have to get off/out of [x1] first." | `x1` = ancestor | `verblib.h:1665` | Noun contains actor; tense-aware |
| 5 | "[Actor] already have/has [x1]." | `x1` = noun | `verblib.h:1668`, `verblib.h:1997` | Already held; tense-aware |
| 6 | "[x2] seem[s] to belong to [x1]." | `x1` = owner, `x2` = noun | `verblib.h:1629`, `verblib.h:1880` | Belongs to animate; tense-aware |
| 7 | "[x2] seem[s] to be a part of [x1]." | `x1` = parent, `x2` = noun | `verblib.h:1633` | Part of another object; tense-aware |
| 8 | "[x1] is not available." | `x1` = noun | `verblib.h:1636` | In unreachable place |
| 9 | "[x1] is not open." | `x1` = container | `verblib.h:1613`, `verblib.h:1640`, `parser.h:2404` | Closed container |
| 10 | "[x1] is hardly portable." | `x1` = noun | `verblib.h:1688` | Scenery object |
| 11 | "[x1] is fixed in place." | `x1` = noun | `verblib.h:1687` | Static object |
| 12 | "[Actor] is carrying too many things already." | — | `verblib.h:1696` | Capacity exceeded |
| 13 | "(putting [x1] into [x2] to make room)" | `x1` = displaced object, `x2` = SACK_OBJECT | `verblib.h:1732` | Auto-stash into sack |

### §C.6.20 Taste (`##Taste`)

**Source:** `english.h:1481–1484` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] taste[s] nothing unexpected." | — | `verblib.h:2841` | Tense-aware |
| 2 | `DecideAgainst()` | — | `verblib.h:2840` | When noun is animate |

### §C.6.21 Tell (`##Tell`)

**Source:** `english.h:1485–1491` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] talk[s] to [oneself] for a while." | — | `verblib.h:2845` | Telling self; tense-aware |
| 2 | "This provokes no reaction." | — | `verblib.h:2847` | Default NPC response; tense-aware |

### §C.6.22 Think (`##Think`)

**Source:** `english.h:1492` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "What a good idea." | — | `verblib.h:2850` | |

### §C.6.23 ThrowAt (`##ThrowAt`)

**Source:** `english.h:1493–1499` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Futile." | — | `verblib.h:2854`, `verblib.h:2861` | When no second or second is inanimate |
| 2 | "[Actor] lack[s] the nerve when it comes to the crucial moment." | — | `verblib.h:2863` | Second is animate; tense-aware |

### §C.6.24 Tie (`##Tie`)

**Source:** `english.h:1500–1505` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] would achieve nothing by this." | — | `verblib.h:2868` | Tense-aware |
| 2 | `DecideAgainst()` | — | `verblib.h:2867` | When noun is animate |

### §C.6.25 Touch (`##Touch`)

**Source:** `english.h:1506–1512` | **Messages:** 3

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | `DecideAgainst()` | — | `verblib.h:2874` | When noun is animate |
| 2 | "[Actor] feel[s] nothing unexpected." | — | `verblib.h:2875` | Tense-aware |
| 3 | "That really wouldn't serve any purpose." | — | `verblib.h:2771`, `verblib.h:2872` | Touching self; tense-aware |

### §C.6.26 Transfer (`##Transfer`)

**Source:** `english.h:1513–1515` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] is still holding [x1]." | `x1` = noun | `verblib.h:2014` | |

### §C.6.27 Unlock (`##Unlock`)

**Source:** `english.h:1518–1527` | **Messages:** 5

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[x1] don't seem to be something [actor] can unlock." | `x1` = noun | `verblib.h:2566` | Not lockable; tense-aware |
| 2 | "[x1] is unlocked at the moment." | `x1` = noun | `verblib.h:2567` | Already unlocked |
| 3 | "[x1] don't seem to fit the lock." | `x1` = second | `verblib.h:2571` | Wrong key |
| 4 | "[Actor] unlock[s] [x1]." | `x1` = noun | `verblib.h:2576` | Success; tense-aware |
| 5 | "(first unlocking [x1])" | `x1` = noun | — | Implicit action |

### §C.6.28 VagueGo (`##VagueGo`)

**Source:** `english.h:1528–1531` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] will have to say which compass direction to go in." | — | `verblib.h:2163` | Tense-aware |

### §C.6.29 Verify (`##Verify`)

**Source:** `english.h:1532–1535` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "The game file has verified as intact." | — | `verblib.h:1174` [Z-machine], `verblib.h:1274` [Glulx] | |
| 2 | "The game file did not verify as intact, and may be corrupt." | — | `verblib.h:1176` [Z-machine], `verblib.h:1275` [Glulx] | |

### §C.6.30 Wait (`##Wait`)

**Source:** `english.h:1536–1538` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "Time passes." | — | `verblib.h:2890` | Tense-aware |

### §C.6.31 Wake (`##Wake`)

**Source:** `english.h:1539–1541` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "The dreadful truth is, this is not a dream." | — | `verblib.h:2893` | Tense-aware |

### §C.6.32 WakeOther (`##WakeOther`)

**Source:** `english.h:1542–1544` | **Messages:** 1

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "That seems unnecessary." | — | `verblib.h:2898` | Tense-aware |

### §C.6.33 Wave (`##Wave`)

**Source:** `english.h:1545–1554` | **Messages:** 3

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "But [actor] isn't holding [x1]." | `x1` = noun | `verblib.h:2904` | Tense-aware |
| 2 | "[Actor] look[s] ridiculous waving [x1] [at x2]." | `x1` = noun, `x2` = second (optional) | `verblib.h:2902`, `verblib.h:2905` | Tense-aware; `x2` only shown if nonzero |
| 3 | `DecideAgainst()` | — | `verblib.h:2903` | When noun is actor |

### §C.6.34 WaveHands (`##WaveHands`)

**Source:** `english.h:1555–1561` | **Messages:** 2

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] wave[s], feeling foolish." | — | `verblib.h:2910` | Tense-aware; no target |
| 2 | "[Actor] wave[s] at [x1], feeling foolish." | `x1` = noun | `verblib.h:2909` | Tense-aware; waving at something |

### §C.6.35 Wear (`##Wear`)

**Source:** `english.h:1562–1574` | **Messages:** 5

| # | Default Text | Parameters | Called From | Notes |
|--:|-------------|-----------|------------|-------|
| 1 | "[Actor] can't wear [x1]." | `x1` = noun | `verblib.h:2655` | Not clothing; tense-aware |
| 2 | "[Actor] is not holding [x1]." | `x1` = noun | `verblib.h:2663` | Not held; tense-aware |
| 3 | "[Actor] is already wearing [x1]." | `x1` = noun | `verblib.h:2665` | Already worn |
| 4 | "[Actor] put[s] on [x1]." | `x1` = noun | `verblib.h:2669` | Success; tense-aware |
| 5 | "[Actor] need[s] to take [x1] out/off of [x2] before wearing it." | `x1` = noun, `x2` = parent | `verblib.h:2659` | When `no_implicit_actions` is set; tense-aware |

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
