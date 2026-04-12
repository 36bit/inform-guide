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

This appendix provides a complete listing of every customizable library message
in the Inform 6 standard library (version 6.12.8). Each entry is verified
against the source code in `english.h`, `verblib.h`, and `parser.h`. For a
full explanation of the action processing pipeline, see Chapter 20 (Actions).
For details on customizing messages, see Chapter 32 (Replacing and Extending
the Library).

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
