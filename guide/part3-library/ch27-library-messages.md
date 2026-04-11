<!--
  Copyright (C) 2026 Software Freedom Conservancy, Inc.
  This file is part of The Inform 6 Programmer's Guide.

  The Inform 6 Programmer's Guide is free software: you can redistribute it
  and/or modify it under the terms of the GNU General Public License as
  published by the Free Software Foundation, either version 3 of the License,
  or (at your option) any later version.

  The Inform 6 Programmer's Guide is distributed in the hope that it will be
  useful, but WITHOUT ANY WARRANTY; without even the implied warranty of
  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General
  Public License for more details.

  You should have received a copy of the GNU General Public License along with
  The Inform 6 Programmer's Guide. If not, see <https://www.gnu.org/licenses/>.
-->

# Chapter 27: Library Messages and Customization

## 27.1 Overview

Every action the Inform 6 library handles produces text output — messages like
"Taken.", "You can't see any such thing.", or "It is pitch dark." These default
messages are functional but generic. Most games benefit from customizing at
least some of them to match the tone, setting, and narrative voice of the work.

The library provides a clean mechanism for this: the `LibraryMessages` object.
By defining this object in your game source, you can intercept any message the
library would print and substitute your own text. You can customize a single
message, dozens of them, or all of them.

This chapter covers how the message system works, how to customize it, and
provides reference tables for the most commonly changed messages.

## 27.2 The LibraryMessages Object

The primary customization mechanism is an object named `LibraryMessages` with a
`before` property. The library checks for this object before printing any
default message, giving your code the first opportunity to respond.

### Definition and Placement

The `LibraryMessages` object must be defined **before** `Include "Parser"` is
processed. This is because the parser sets up the message system during
compilation, and it needs to know whether a `LibraryMessages` object exists.

```inform6
Object LibraryMessages
    with before [;
        Take:
            switch (lm_n) {
                1: print_ret "You already have ", (the) lm_o, ".";
                2: print_ret "Hmm, ", (the) lm_o,
                             " seems to be part of the scenery.";
            };
        Drop:
            switch (lm_n) {
                1: print_ret "You're not even holding ", (the) lm_o, ".";
            };
    ];

Include "Parser";
Include "VerbLib";
```

### How It Works

When the library needs to print a message, it sets several variables and then
calls the `before` property of `LibraryMessages` (if the object exists):

| Variable | Contains                                          |
|----------|---------------------------------------------------|
| `action` | The current action (e.g., `##Take`, `##Drop`)     |
| `lm_n`   | The message number within that action              |
| `lm_o`   | The relevant object (if any)                       |
| `lm_s`   | The relevant string (if any)                       |

Your `before` routine should check the current `action` (using action labels
like `Take:`, `Drop:`, etc.), then switch on `lm_n` to identify the specific
message. If you print your own message and return `true`, the library's default
message is suppressed.

If your routine returns `false` (or falls through without matching), the
library prints its default message as usual.

### Selective Customization

You do not need to handle every message. Handle only the ones you want to
change; everything else falls through to the defaults:

```inform6
Object LibraryMessages
    with before [;
        Miscellany:
            switch (lm_n) {
                17: "It's pitch dark and you can't see a damn thing.";
                19: "That's not something you can pick up.";
            };
        Examine:
            if (lm_n == 1)
                "You don't notice anything remarkable about ",
                (the) lm_o, ".";
    ];
```

## 27.3 The L__M Routine

Under the hood, the library prints all its messages by calling an internal
routine named `L__M`. Understanding this routine helps when debugging message
customization and when you want to invoke standard library messages from your
own code.

### Signature

```inform6
L__M(action, message_number, obj);
```

- `action` — the action constant (e.g., `##Take`)
- `message_number` — which message to print (1, 2, 3, etc.)
- `obj` — an optional object reference passed as `lm_o`

### Calling L__M from Game Code

You can call `L__M` directly to produce a standard library message:

```inform6
Object strange_box "strange box"
    with name 'strange' 'box',
         before [;
            Open:
                L__M(##Open, 1, self);
                print "^Something rattles inside.^";
                rtrue;
         ],
    has  openable container;
```

This prints the standard "You open the strange box." message, followed by your
custom text. It is a useful technique when you want to augment rather than
replace the default message.

### How L__M Interacts with LibraryMessages

When `L__M` is called, it first checks whether a `LibraryMessages` object
exists and gives it a chance to handle the message. Only if `LibraryMessages`
does not handle the message does `L__M` print the default text.

This means that calling `L__M` from your game code also respects any
`LibraryMessages` customizations you have defined.

## 27.4 Message Numbers by Action

Each action has a set of numbered messages. Below are the most commonly
customized actions and their message numbers. The default text shown is
representative; consult the library source (`english.h`) for exact wording.

### Take (action `##Take`)

| # | Default Text                                               |
|---|------------------------------------------------------------|
| 1 | "You already have (the object)."                           |
| 2 | "(The object) is not something you can pick up."           |
| 3 | "You don't think (the NPC) would appreciate that."         |
| 4 | "You'd have to get off (the object) first."                |
| 5 | "You already have that."                                   |
| 6 | "(The container) seems to belong to (the NPC)."            |
| 7 | "(The container) is inside (the object) which is closed."  |
| 8 | "(The object) is not obviously takeable."                  |
| 9 | "(The object) is hardly portable."                         |
| 10| "(The object) is fixed in place."                          |
| 11| "You're carrying too many things already."                 |
| 12| "(putting X into the sack to make room)"                   |
| 13| "Taken."                                                   |

### Drop (action `##Drop`)

| # | Default Text                                               |
|---|------------------------------------------------------------|
| 1 | "You aren't holding (the object)."                         |
| 2 | "You can't drop something you're wearing."                 |
| 3 | "(first taking (the object) off)"                          |
| 4 | "Dropped."                                                 |

### Look (action `##Look`)

| # | Default Text                                               |
|---|------------------------------------------------------------|
| 1 | " (on the object)"                                         |
| 2 | " (in the object)"                                         |
| 3 | " (as a light source)"                                     |
| 4 | "On (the object) " + list of contents                      |
| 5 | "You can also see " + list of items                        |
| 6 | "You can see " + list of items                             |
| 7 | "You can also see " + list of items (long form)            |
| 8 | "On (the supporter) you can see " + list                   |

### Go (action `##Go`)

| # | Default Text                                               |
|---|------------------------------------------------------------|
| 1 | "You'll have to get off/out of (the object) first."        |
| 2 | "You can't go that way."                                   |
| 3 | "You are unable to climb (the object)."                    |
| 4 | "You are unable to descend by (the object)."               |
| 5 | "You can't, since (the door) is in the way."               |
| 6 | "You can't, since (the door) leads nowhere."               |

### Open (action `##Open`)

| # | Default Text                                               |
|---|------------------------------------------------------------|
| 1 | "You open (the object)."                                   |
| 2 | "It's locked."                                             |
| 3 | "That's not something you can open."                       |
| 4 | "(The object) is already open."                            |
| 5 | "You open (the object), revealing ..."                     |

### Close (action `##Close`)

| # | Default Text                                               |
|---|------------------------------------------------------------|
| 1 | "That's not something you can close."                      |
| 2 | "(The object) is already closed."                          |
| 3 | "You close (the object)."                                  |

### Lock (action `##Lock`)

| # | Default Text                                               |
|---|------------------------------------------------------------|
| 1 | "That's not something you can lock."                       |
| 2 | "(The object) is already locked."                          |
| 3 | "You'll need to close (the object) first."                 |
| 4 | "That doesn't seem to fit the lock."                       |
| 5 | "You lock (the object)."                                   |

### Unlock (action `##Unlock`)

| # | Default Text                                               |
|---|------------------------------------------------------------|
| 1 | "That's not something you can unlock."                     |
| 2 | "(The object) is already unlocked."                        |
| 3 | "That doesn't seem to fit the lock."                       |
| 4 | "You unlock (the object)."                                 |

### Examine (action `##Examine`)

| # | Default Text                                               |
|---|------------------------------------------------------------|
| 1 | "You see nothing special about (the object)."              |
| 2 | "(The object) is currently switched on/off."               |
| 3 | "(The object) is currently open/closed."                   |

### Miscellany

The `Miscellany` pseudo-action covers parser and system messages that are not
tied to a specific action:

| #  | Default Text                                              |
|----|-----------------------------------------------------------|
| 1  | "(considering the first sixteen objects only)"             |
| 2  | "Nothing to do!"                                          |
| 3  | "You have died."                                          |
| 4  | "You have won."                                           |
| 5  | "Would you like to RESTART, RESTORE, or QUIT?"            |
| 6  | "Would you like to RESTART, RESTORE, UNDO, or QUIT?"     |
| 10 | "I beg your pardon?"                                      |
| 17 | "It is pitch dark, and you can't see a thing."            |
| 19 | "That's not a verb I recognise."                          |
| 27 | "I didn't understand that sentence."                      |
| 28 | "I only understood you as far as..."                      |
| 30 | "You can't see any such thing."                           |
| 38 | "That's not something you need to refer to..."            |
| 51 | "> "  (the command prompt)                                |

## 27.5 String Constants

The library defines several text constants used in descriptions and messages.
These can be redefined in your game to alter how the library refers to common
concepts.

### Key String Constants

| Constant        | Default Value    | Used For                          |
|-----------------|------------------|-----------------------------------|
| `YOURSELF__TX`  | "yourself"       | How the player object is named    |
| `DARKNESS__TX`  | "Darkness"       | Room name when in the dark        |
| `THOSET__TX`    | "those things"   | Referring to multiple objects      |
| `THAT__TX`      | "that"           | Generic pronoun for objects        |
| `OR__TX`        | " or "           | Joining alternatives in lists      |
| `AND__TX`       | " and "          | Joining items in lists             |
| `NOTHING__TX`   | "nothing"        | Empty containers/supporters        |
| `IS__TX`        | " is"            | Copula for singular objects        |
| `ARE__TX`       | " are"           | Copula for plural objects          |

### Redefining String Constants

To redefine a string constant, use a `Constant` directive before including the
library:

```inform6
Constant YOURSELF__TX = "your magnificent self";
Constant DARKNESS__TX = "Total Darkness";

Include "Parser";
Include "VerbLib";
```

Now when the player examines themselves, the game will say "your magnificent
self" instead of "yourself", and dark rooms will be titled "Total Darkness"
instead of "Darkness".

## 27.6 Language Customization

The Inform 6 library separates language-specific text from its core logic. All
English-specific text lives in `english.h`, which is included by the parser.
This design makes it possible to create games in other languages by replacing
`english.h` with an equivalent file.

### The Language Definition File

The language file provides:

- **LanguageVerb** — a routine that recognizes language-specific verb words not
  covered by the standard grammar.
- **LanguageDirection** — maps direction words to direction properties.
- **LanguageNumber** — prints a number as words.
- **LanguageContents** — describes the contents of containers/supporters.
- **LanguageLM** — the master routine containing all default library message
  text.

### Customizing LanguageVerb

The `LanguageVerb` routine is called by the parser when it encounters a verb
word it doesn't recognize in the grammar tables. You can provide a replacement
to handle game-specific abbreviations:

```inform6
Replace LanguageVerb;

[ LanguageVerb word;
    if (word == 'x')    { verb_wordnum = 1; return 'examine'; }
    if (word == 'z')    { verb_wordnum = 1; return 'wait'; }
    if (word == 'l')    { verb_wordnum = 1; return 'look'; }
    rfalse;
];
```

### Creating Non-English Games

To create a game in another language, you need to:

1. Write a replacement for `english.h` (e.g., `french.h`, `german.h`).
2. Provide translations for all messages in `LanguageLM`.
3. Define the grammar with vocabulary in the target language.
4. Handle grammatical features specific to the language (gender, case, etc.).

Several community-maintained language definition files exist for languages
including French, German, Spanish, Italian, and Swedish.

### Partial Language Customization

You don't need to replace the entire language file to make smaller changes. For
example, to change how directions are printed:

```inform6
Replace LanguageDirection;

[ LanguageDirection d;
    switch (d) {
        n_to:  print "to the north";
        s_to:  print "to the south";
        e_to:  print "to the east";
        w_to:  print "to the west";
        ne_to: print "to the northeast";
        se_to: print "to the southeast";
        nw_to: print "to the northwest";
        sw_to: print "to the southwest";
        u_to:  print "upward";
        d_to:  print "downward";
        in_to: print "inside";
        out_to: print "outside";
    }
];
```

## 27.7 Common Customization Examples

This section presents practical examples of common customizations that
demonstrate the techniques covered in this chapter.

### Changing the "Taken" Message

The single most commonly changed message is the confirmation printed after
successfully picking up an object:

```inform6
Object LibraryMessages
    with before [;
        Take:
            if (lm_n == 13)
                "Got it.";
    ];
```

### Adding Personality to Parser Messages

You can give your game's parser a distinctive voice by customizing the
`Miscellany` messages:

```inform6
Object LibraryMessages
    with before [;
        Miscellany:
            switch (lm_n) {
                10: "Come again? I didn't catch that.";
                17: "It's so dark you can barely see your own hands.";
                19: "That's not a word I know. Try something else.";
                27: "I understood some of that but not all. Could you
                     rephrase?";
                30: "You don't see anything like that here.";
            };
    ];
```

### Customizing the Room Description Format

By customizing `Look` messages, you can change how room contents are listed:

```inform6
Object LibraryMessages
    with before [;
        Look:
            switch (lm_n) {
                5: print "Nearby you notice ";
                   WriteListFrom(child(location), ENGLISH_BIT);
                   ".";
                6: print "Here you see ";
                   WriteListFrom(child(location), ENGLISH_BIT);
                   ".";
            };
    ];
```

### Changing the Inventory Format

The library's default inventory is a simple list. You can customize it by
intercepting the `Inv` messages:

```inform6
Object LibraryMessages
    with before [;
        Inv:
            if (lm_n == 1)
                "You aren't carrying a thing.";
            if (lm_n == 2) {
                print "You are carrying:^";
                rtrue;
            }
    ];
```

### Comprehensive Example: A Terse Game

This example shows a `LibraryMessages` object that creates a terse, clipped
narrative style:

```inform6
Object LibraryMessages
    with before [;
        Take:
            switch (lm_n) {
                1:  "Already have it.";
                5:  "Already have it.";
                11: "Hands full.";
                13: "Taken.";
            };
        Drop:
            switch (lm_n) {
                1:  "Not holding it.";
                4:  "Dropped.";
            };
        Go:
            switch (lm_n) {
                1:  "Get up first.";
                2:  "No exit that way.";
            };
        Open:
            switch (lm_n) {
                1:  "Opened.";
                2:  "Locked.";
                4:  "Already open.";
            };
        Close:
            switch (lm_n) {
                2:  "Already closed.";
                3:  "Closed.";
            };
        Examine:
            if (lm_n == 1)
                "Nothing special.";
        Miscellany:
            switch (lm_n) {
                10: "What?";
                17: "Too dark.";
                19: "Unknown verb.";
                27: "Unclear.";
                30: "Not here.";
            };
    ];
```

### Example: Themed Messages for a Horror Game

```inform6
Object LibraryMessages
    with before [;
        Miscellany:
            switch (lm_n) {
                17: "The darkness presses in around you, thick and
                     suffocating. You can see nothing.";
                30: "Your eyes dart around frantically, but there's
                     nothing like that here.";
            };
        Take:
            switch (lm_n) {
                11: "Your trembling hands are already full.";
                13: "You snatch up ", (the) lm_o,
                    " with shaking fingers.";
            };
        Drop:
            if (lm_n == 4)
                "You let ", (the) lm_o,
                " fall from your nerveless fingers.";
        Go:
            if (lm_n == 2)
                "Something stops you. There's no way through.";
        Examine:
            if (lm_n == 1)
                "You stare at ", (the) lm_o,
                " but your mind refuses to focus.";
    ];
```

### Combining LibraryMessages with Other Techniques

`LibraryMessages` handles the text the library generates. For changing the
library's *behavior* rather than its text, combine `LibraryMessages` with
the mechanisms described in Chapter 26:

```inform6
Object LibraryMessages
    with before [;
        Take:
            if (lm_n == 13)
                "Acquired.";
    ];

[ GamePreRoutine;
    if (action == ##Eat && noun hasnt edible) {
        "That's definitely not food.";
    }
    rfalse;
];

[ GamePostRoutine;
    if (action == ##Take)
        UpdateWeight();
    rfalse;
];
```

By using `LibraryMessages` for text changes, `GamePreRoutine` for behavior
interception, and `GamePostRoutine` for side effects, you keep each concern
cleanly separated.
