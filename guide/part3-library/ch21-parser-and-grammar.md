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

# Chapter 21: The Parser and Grammar

The parser is the largest and most complex component of the standard
library. It reads the player's typed input, matches it against grammar
definitions, resolves noun phrases to objects, and produces an action with
its arguments. This chapter explains how the parser works, how grammar is
defined, and how game authors can customize parsing behavior.

## 21.1 How the Parser Works

When the player types a command and presses Enter, the parser performs
four major phases:

### Phase 1: Tokenization

The library calls `Keyboard()` to read a line of input into the
**buffer** array. It then calls `Tokenise__()` to split the input into
individual words and look each one up in the dictionary. The results
are stored in the **parse** table, which records the number of words,
and for each word its dictionary address (or zero if not found), its
length, and its position in the buffer.

```inform6
! Key global variables set during tokenisation:
!   num_words    — total number of words typed
!   wn           — current word number (1-based index into parse)
!   verb_word    — dictionary address of the verb word
!   verb_wordnum — position of the verb in the input (1-based)
```

Before grammar matching begins, the library calls the
`BeforeParsing()` entry point (if the game defines one), giving the
game a chance to manipulate the input buffer or parse table before the
parser examines it.

### Phase 2: Grammar Matching

The parser identifies the verb word and looks up its grammar table —
the set of grammar lines defined by `Verb` and `Extend` directives.
It tries each grammar line in order, attempting to match the remaining
input words against the line's token pattern.

For each token in a grammar line, the parser calls internal matching
routines that consume words from the input. If a token fails to match,
the parser backtracks and tries the next grammar line. If all lines
fail, a parser error is generated.

When a grammar line matches completely, the parser records the action
and sets up `noun` and `second` from the matched objects.

### Phase 3: Noun Resolution

When the parser encounters a noun-type token (`noun`, `held`, `multi`,
etc.), it searches the current **scope** for objects whose names match
the words in the input. This involves:

1. Calling `parse_name` on each object in scope (if defined).
2. Matching individual `name` property words against input words.
3. Building a list of candidate objects in `match_list`.
4. If multiple candidates remain, running **disambiguation** (§21.7).

### Phase 4: Action Dispatch

Once parsing succeeds, the parser sets the global variables `action`,
`noun`, `second`, `inp1`, `inp2`, and `actor`, then returns control to
the main game loop. The action processing pipeline (Chapter 20) takes
over from here.

### Special Input Handling

Before grammar matching, the parser handles several special cases:

- **"oops"** — replaces a word in the previous command.
- **"again"** (or **"g"**) — re-executes the previous command.
- **Implicit "go"** — a bare direction word like `NORTH` is treated as
  `GO NORTH`.
- **Actor commands** — input of the form `NPC, COMMAND` sets `actor`
  to the NPC and parses `COMMAND` as an order.

## 21.2 The `Verb` Directive

The `Verb` directive defines grammar for one or more verb words.  It
appears in `grammar.h` (or in game source after `Include "Grammar"`).

### Basic Syntax

```inform6
Verb 'word' 'synonym1' 'synonym2' ...
    * token1 token2 ...    -> ActionName
    * token1 token2 ...    -> OtherAction;
```

Each `Verb` directive begins with one or more dictionary words that the
player can type as the verb. The body contains one or more **grammar
lines**, each starting with `*` and ending with `-> ActionName`. The
directive ends with a semicolon after the last grammar line.

```inform6
Verb 'take'
    * multi                             -> Take
    * 'off' multiheld                   -> Disrobe
    * multiinside 'from'/'off' noun     -> Remove
    * 'inventory'                       -> Inv;
```

All synonym words in a single `Verb` directive share the same grammar
lines. In the example above, `TAKE`, and any synonyms listed, all use
the same four grammar lines.

### Single-Letter Abbreviations

Dictionary words ending in `//` are treated as single-letter
abbreviations:

```inform6
Verb 'examine' 'x//' 'check' 'describe' 'watch'
    * noun                              -> Examine;
```

Here `x` is a single-letter synonym for `examine`.

### Meta Verbs

Adding `meta` after `Verb` declares all grammar lines as meta actions:

```inform6
Verb meta 'score'
    *                                   -> Score;

Verb meta 'quit' 'q//' 'die'
    *                                   -> Quit;
```

Meta actions do not consume a game turn and bypass before/after
processing (see §20.5).

### The `reverse` Modifier

Adding `reverse` after the action name swaps `noun` and `second` before
the action fires:

```inform6
Verb 'give' 'feed' 'offer' 'pay'
    * creature held                     -> Give reverse
    * held 'to' creature               -> Give;
```

The first line matches `GIVE TROLL SWORD` — the creature comes first in
the input, but `reverse` swaps them so `noun` = sword and `second` =
troll, matching the `Give` action's expected argument order.

## 21.3 Grammar Lines and Tokens

Each grammar line is a pattern of **tokens** that the parser matches
against the player's input. Tokens are listed between `*` and `->`.

### Token Types

| Token | Matches | `noun`/`second` receives |
|-------|---------|--------------------------|
| `noun` | Any single object in scope | The object |
| `held` | An object held by the actor | The object |
| `multi` | One or more objects in scope | Each object (action runs per object) |
| `multiheld` | One or more held objects | Each object |
| `multiexcept` | Multiple objects, excluding the other noun | Each object |
| `multiinside` | Multiple objects inside a container | Each object |
| `creature` | An animate object in scope | The object |
| `number` | A numeric value | The number (as integer) |
| `topic` | Free-form text (any words) | Consult-style word range |
| `special` | A single word or number | The parsed value |
| `'word'` | A specific dictionary word (literal) | (consumed, not stored) |
| `noun=Routine` | An object filtered by Routine | The object |
| `scope=Routine` | An object from a custom scope | The object |
| `attribute` | An object with a given attribute | The object |

### Literal Words

Literal words are enclosed in single quotes and must appear exactly in
the input:

```inform6
Verb 'look'
    *                                   -> Look
    * 'at' noun                         -> Examine
    * 'under' noun                      -> LookUnder
    * 'inside'/'in'/'into' noun         -> Search;
```

The `/` operator provides alternatives — `'inside'/'in'/'into'` matches
any one of those three words.

### The `multi` Family

The `multi` tokens cause the action to fire **once per matched object**.
If the player types `TAKE ALL`, the parser matches every takeable object
in scope, and `TakeSub` runs separately for each one.

- `multi` — any objects in scope.
- `multiheld` — only objects carried by the actor.
- `multiexcept` — objects in scope except the other noun argument.
  Used in patterns like `PUT ALL IN BOX` where the box is excluded.
- `multiinside` — objects inside a specified container.
  Used in patterns like `TAKE ALL FROM BOX`.

```inform6
Verb 'drop' 'discard'
    * multiheld                          -> Drop
    * multiexcept 'in'/'into' noun      -> Insert
    * multiexcept 'on'/'onto' noun      -> PutOn;
```

### Filter Tokens: `noun=Routine`

A `noun=Routine` token restricts matches to objects for which `Routine`
returns true. The library uses this for direction filtering:

```inform6
Verb 'go' 'run' 'walk'
    * noun=ADirection                   -> Go
    * noun                              -> Enter;
```

The `ADirection` routine returns true only for direction objects, so
`GO NORTH` matches the first line, while `GO CAVE` matches the second.

### Scope Tokens: `scope=Routine`

A `scope=Routine` token replaces the normal scope with objects provided
by a custom routine. The routine is called with `scope_stage` set to
indicate what phase of scoping is occurring:

```inform6
[ SearchDeadScope x;
    switch (scope_stage) {
        1: rfalse;         ! No special prompts
        2: objectloop (x ofclass Thing)
               if (x has dead) PlaceInScope(x);
           rtrue;          ! Scope provided
        3: "You think of the dead.";  ! Error message
    }
];

Verb 'remember'
    * scope=SearchDeadScope             -> Remember;
```

### The `topic` Token

The `topic` token matches free-form text — any sequence of words. It is
used for conversational actions and consulting:

```inform6
Verb 'ask'
    * creature 'about' topic           -> Ask;

Verb 'consult'
    * noun 'about' topic              -> Consult;
```

When `topic` matches, the library sets `consult_from` and `consult_words`
to indicate which words were matched:

```inform6
[ ConsultSub;
    ! consult_from = word number of first topic word
    ! consult_words = number of topic words
    ...
];
```

## 21.4 The `Extend` Directive

The `Extend` directive adds grammar lines to an existing verb without
replacing its original grammar. This is the primary mechanism for
customizing the parser from game source.

### Basic Syntax

```inform6
Extend 'verb' * tokens -> ActionName;
```

By default, new grammar lines are appended **after** the existing lines.
The parser tries original lines first, then extended ones.

### Placement Modifiers

| Modifier | Effect |
|----------|--------|
| `first` | New lines are inserted **before** existing lines. |
| `last` | New lines are appended **after** existing lines (default). |
| `replace` | All existing lines are removed; only the new lines remain. |
| `only` | Extends only the named synonym, not all words sharing the verb's grammar. |

```inform6
! Add a new grammar line tried before existing ones:
Extend 'look' first
    * 'through' noun                   -> Search;

! Replace all grammar for 'listen':
Extend 'listen' replace
    * 'to' noun                        -> Listen
    * 'carefully'                      -> Listen;

! Extend only 'x', not 'examine':
Extend only 'x//'
    * 'me'                             -> Inv;
```

### Practical Example

Adding a new `SEARCH UNDER` command:

```inform6
Extend 'search'
    * 'under' noun                     -> LookUnder
    * noun 'for' topic                 -> Consult;
```

Now `SEARCH UNDER TABLE` works alongside the original `SEARCH TABLE`.

## 21.5 The `parse_name` Property

The `parse_name` property provides custom parsing logic for an object.
When the parser tries to match input words against an object, it calls
`parse_name` (if defined) before falling back to matching individual
`name` words.

### How It Works

The `parse_name` routine is called with `wn` set to the current word
position. It should examine words starting at `wn` (using `NextWord()`
or `WordAddress()`) and return the number of consecutive words that
match the object.

```inform6
Object  "red brick wall" room
  with  parse_name [ w n;
            while (true) {
                w = NextWord();
                if (w == 'red' or 'brick' or 'wall') n++;
                else break;
            }
            return n;  ! Number of words consumed
        ],
        name 'red' 'brick' 'wall';
```

### Return Values

| Return value | Meaning |
|--------------|---------|
| `n > 0` | Matched `n` words. |
| `0` | No match — the parser will try `name` words instead. |
| `-1` | Used during `##TheSame` disambiguation: the two objects are identical. |
| `-2` | Used during `##TheSame` disambiguation: the two objects are different. |

### Disambiguation with `parse_name`

When the parser needs to determine whether two objects are identical
(for purposes like `TAKE A BALL` vs `TAKE THE BALL`), it calls
`parse_name` with `parser_action` set to `##TheSame`, `parser_one` set
to the first object, and `parser_two` set to the second. The routine
should return `-1` (identical) or `-2` (different).

```inform6
Object  -> "red ball" room
  with  parse_name [ w n;
            if (parser_action == ##TheSame) {
                if (parser_two ofclass RedBall) return -1;
                return -2;
            }
            while (true) {
                w = NextWord();
                if (w == 'red' or 'ball') n++;
                else return n;
            }
        ];
```

### Signaling Plurals

If `parse_name` matches a plural word, it should set `parser_action` to
`##PluralFound` so the parser knows to consider multiple objects:

```inform6
parse_name [ w n;
    while (true) {
        w = NextWord();
        if (w == 'coin') { n++; continue; }
        if (w == 'coins') {
            n++;
            parser_action = ##PluralFound;
            continue;
        }
        return n;
    }
],
```

## 21.6 The `ParseNoun` Entry Point

The game can define a `ParseNoun` entry-point routine that is called
during noun resolution for every object the parser considers. It
receives the object being tested and should return the number of words
matched, or `0` to decline (letting the parser use its own matching).

```inform6
[ ParseNoun obj  w;
    ! Allow "shiny" as an adjective for anything with 'shiny' in name
    w = NextWord();
    if (w == 'shiny' && obj has general) return 1;
    return 0;  ! Don't interfere
];
```

`ParseNoun` is called **after** `parse_name` but before the standard
`name` word matching. If it returns a positive value, that many words
are consumed as a match. If it returns `0`, the object is treated as
not matching (zero words matched). If it returns `-1`, the routine is
declining to handle this object, and the parser falls through to the
standard name-word matching instead.

## 21.7 Disambiguation

When multiple objects match a noun phrase, the parser must determine
which object the player intended. This process is called
**disambiguation**.

### The Disambiguation Process

1. The parser builds `match_list`, an array of candidate objects.
2. Objects are grouped into **equivalence classes** — groups of
   identical objects (same name words, same `parse_name` behavior).
3. If only one class exists, the parser picks any member.
4. If multiple classes exist, the parser asks the player:
   `"Which do you mean, the red ball or the blue ball?"`
5. The player's response is parsed to narrow the selection.

### The `ChooseObjects` Entry Point

The game can define `ChooseObjects` to influence which objects the
parser prefers. It is called for each candidate object during noun
resolution.

```inform6
[ ChooseObjects obj flag;
    ! flag == 2: called for ALL objects, return score 0-9
    ! flag == 1: called individually, return 0/1/2
    !   0 = let parser decide
    !   1 = force acceptance
    !   2 = force rejection

    if (flag == 2) {
        ! Prefer lit objects for "examine"
        if (action_to_be == ##Examine && obj has light)
            return 5;
        return 3;  ! neutral
    }
    return 0;  ! let parser decide
];
```

When `flag` is `2`, the routine returns a score from 0 to 9; the parser
prefers higher-scoring objects. When `flag` is `1`, the routine returns
`0` (parser decides), `1` (accept), or `2` (reject).

### Practical Example

Preferring a key that fits the lock being unlocked:

```inform6
[ ChooseObjects obj flag;
    if (flag < 2) return 0;
    if (action_to_be == ##Unlock && obj == brass_key
        && noun == brass_door)
        return 9;  ! Strongly prefer
    return 3;
];
```

## 21.8 Parser Error Handling

When the parser cannot understand the player's input, it generates a
parser error. The library defines 18 standard error types as constants:

| Constant | Value | Meaning |
|----------|-------|---------|
| `STUCK_PE` | 1 | Couldn't make sense of that |
| `UPTO_PE` | 2 | Understood only up to ... |
| `NUMBER_PE` | 3 | Expected a number |
| `CANTSEE_PE` | 4 | Can't see any such thing |
| `TOOLIT_PE` | 5 | Incomplete reference |
| `NOTHELD_PE` | 6 | Not holding that |
| `MULTI_PE` | 7 | Can't use multiple objects there |
| `MMULTI_PE` | 8 | Can only use multiple objects once |
| `VAGUE_PE` | 9 | Not sure what "it" refers to |
| `EXCEPT_PE` | 10 | Exception error |
| `ANIMA_PE` | 11 | Can only do that to animate things |
| `VERB_PE` | 12 | Not a verb I recognize |
| `SCENERY_PE` | 13 | Not something to interact with |
| `ITGONE_PE` | 14 | "It" is no longer available |
| `JUNKAFTER_PE` | 15 | Junk after the command |
| `TOOFEW_PE` | 16 | Not enough of those available |
| `NOTHING_PE` | 17 | Nothing to do |
| `ASKSCOPE_PE` | 18 | Scope error (for `scope=` tokens) |

### The `ParserError` Entry Point

The game can define a `ParserError` routine to customize error messages.
It receives the error type as its argument. Return `true` to suppress
the default message, or `false` to let the library print its standard
error.

```inform6
[ ParserError etype;
    switch (etype) {
        CANTSEE_PE:
            "You don't see anything like that here.";
        NOTHELD_PE:
            "You're not carrying that!";
        VERB_PE:
            "I beg your pardon?";
    }
    rfalse;  ! Use library default for other errors
];
```

### The `UnknownVerb` Entry Point

When the parser encounters a word in verb position that has no grammar
definition, it calls `UnknownVerb` (if defined). The routine receives
the dictionary address of the unknown word and can return a replacement
dictionary word (redirecting to a known verb) or `false` to let the
standard error occur.

```inform6
[ UnknownVerb word;
    if (word == 'xyzzy') return 'cast';
    rfalse;
];
```

### The `PrintVerb` Entry Point

When the library needs to print a verb word (e.g. in error messages),
it calls `PrintVerb` after trying `LanguageVerb`. Return `true` if
the routine handled the printing, or `false` for the default.

```inform6
[ PrintVerb verb;
    if (verb == 'swim') { print "swim"; rtrue; }
    rfalse;
];
```

## 21.9 The Input Buffer and Parse Table

The parser uses two main data structures to hold input: the **buffer**
(raw character input) and the **parse** table (tokenized word list).

### Z-Machine Layout

On the Z-machine, `buffer` is a byte array of 122 characters (including
a length prefix). The `parse` table holds up to 15 words, each recorded
as a 4-byte entry:

```inform6
! buffer->0  : maximum characters allowed
! buffer->1  : number of characters typed (filled by VM)
! buffer->2+ : the typed characters
!
! parse->0   : maximum words allowed (15)
! parse->1   : number of words found
! For word k (1-based), the 4 bytes starting at offset 4k-2 hold:
!   parse-->(2k-1) : dictionary address (0 if not found)
!   parse->(4k)    : length of word
!   parse->(4k+1)  : position in buffer
```

### Glulx Layout

On Glulx, `buffer` is 260 bytes and the parse table holds up to 20
words. Each parse entry occupies three machine words (12 bytes): the
32-bit dictionary address, the word length, and the buffer position.
`parse-->0` holds the number of words found.

### Key Variables

| Variable | Purpose |
|----------|---------|
| `wn` | Current word index into the parse table (1-based). The parser advances this as it consumes words. |
| `num_words` | Total number of words in the current input. |
| `verb_word` | Dictionary address of the recognized verb word. |
| `verb_wordnum` | Position of the verb word in the input (1-based). |

### Word Access Routines

The library provides routines for accessing words in the parse table:

```inform6
NextWord()          ! Returns dictionary address of word at wn,
                    ! then advances wn. Returns 0 if not in dictionary.

NextWordStopped()   ! Like NextWord() but returns -1 past end of input.

WordAddress(n)      ! Returns the buffer address of word n.

WordLength(n)       ! Returns the length of word n.

NumberWord(n)       ! Parses word n as a number; returns the value
                    ! or 0 if not found.
```

### Manipulating Input

Games can modify the buffer and re-tokenize to alter what the parser
sees. This is typically done in `BeforeParsing()`:

```inform6
[ BeforeParsing  i;
    ! Replace "please" at start of input with nothing
    if (parse-->1 == 'please') {
        for (i = 1 : i <= parse->1 - 1 : i++)
            parse-->(i) = parse-->(i + 1);
        parse->1 = parse->1 - 1;
        num_words = num_words - 1;
    }
];
```

The `Tokenise__()` function can be called to re-tokenize the buffer
after modifications:

```inform6
! Z-machine:
Tokenise__(buffer, parse);

! After modifying buffer contents, re-tokenise to update the
! parse table with new word boundaries and dictionary lookups.
```

### The `Keyboard()` Function

`Keyboard()` is the top-level input routine. It:

1. Prints the command prompt (after sending `##Prompt`).
2. Reads a line of input from the player into `buffer`.
3. Calls `Tokenise__()` to fill the parse table.
4. Handles `UNDO` if the first word is `undo`.
5. Handles `OOPS` if the first word is `oops`.

Game code rarely needs to call `Keyboard()` directly, but understanding
its role helps when debugging parser issues or writing advanced input
manipulation.
