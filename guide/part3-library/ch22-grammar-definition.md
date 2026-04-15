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

# Chapter 22: Grammar Definition

This chapter provides a detailed reference for defining and extending
grammar. Where Chapter 21 introduced the parser, verb directives, and
token types at a practical level, this chapter examines the underlying
mechanisms: formal syntax, grammar table encoding, token type internals,
custom parsing routines, action linkage, and a complete catalog of the
standard library's grammar entries.

## 22.1 The Verb Directive in Detail

As introduced in §21.2, the `Verb` directive declares one or more
synonym words and associates them with grammar lines. This section
presents the full formal syntax and the finer points of its semantics.

### Formal Syntax

```
verb-definition →
    'Verb' 'meta'? word-list grammar-line+ ';'

word-list → verb-word+
verb-word → SINGLE-QUOTED-WORD

grammar-line →
    '*' token* '->' action-name 'reverse'? 'meta'?

token →
    SINGLE-QUOTED-WORD              ! literal preposition
  | SINGLE-QUOTED-WORD '/' ...     ! alternative prepositions
  | elementary-token                ! noun, held, multi, etc.
  | elementary-token '=' routine   ! filter token
  | 'scope' '=' routine            ! scope token
  | attribute-name                  ! attribute filter
  | routine                        ! general parsing routine

action-name → IDENTIFIER           ! must have a corresponding Sub routine
```

### Synonym Sharing

All words listed in the *word-list* share the same grammar table entry.
The compiler creates a single grammar table and points every synonym's
dictionary entry at it. This means `Verb 'take' 'grab' 'seize'` does
not create three separate grammars — it creates one grammar with three
dictionary words leading to it. Changing the grammar of any one synonym
affects all of them; to split one synonym away, use `Extend only`
(§22.2).

### The `meta` Modifier

When the word `meta` appears immediately after `Verb`, every grammar
line in that directive produces a meta action — one that operates on the
game system rather than the game world (see §20.5). Meta actions are not
intercepted by `before`/`after` handlers and are not affected by
`Darkness` or other world-state checks. In the standard library, all
system commands (`save`, `restore`, `quit`, `score`, etc.) are declared
with `Verb meta`.

### Per-Line Meta (6.43+)

Compiler version 6.43 introduces the `$GRAMMAR_META_FLAG` setting. When
enabled (set to 1 on the command line or by defining
`Grammar_Meta__Value` before the first grammar declaration), individual
grammar lines can be marked `meta` independently, allowing a single verb
to mix meta and non-meta actions:

```inform6
! Compile with $GRAMMAR_META_FLAG=1
Verb 'load'
    * noun                          -> Push
    * 'game'                        -> Restore meta;
```

In this mode the compiler sorts action numbers so that all meta actions
are assigned lower values than non-meta actions. The system constant
`#highest_meta_action_number` returns the highest action number that is
flagged as meta, or −1 if no meta actions exist. Game code can test
this at runtime:

```inform6
if (action <= #highest_meta_action_number) {
    print "This is a meta action.^";
}
```

To test whether the feature is available at compile time, use
`#ifdef GRAMMAR_META_FLAG` (note: no dollar sign in the `#ifdef` test).

## 22.2 The Extend Directive

As introduced in §21.4, the `Extend` directive adds grammar lines to an
existing verb or modifies its grammar. This section details the full
syntax and the precise effect of each placement modifier.

### Formal Syntax

```
extend-definition →
    'Extend' 'only'? word-list modifier? grammar-line+ ';'

modifier → 'first' | 'last' | 'replace'
```

When no modifier is given, `last` is the default — new lines are
appended after existing ones.

### Placement Modifiers

**`last`** (default): Appends the new grammar lines after all existing
lines for the verb. The parser tries original lines first, then the new
ones.

**`first`**: Inserts the new grammar lines before all existing lines.
The parser tries the new lines first, which is useful when overriding
default behavior without removing it:

```inform6
Extend 'look' first
    * 'through' noun              -> PeekThrough;
```

**`replace`**: Removes all existing grammar lines for the verb and
substitutes the new ones. The original grammar is discarded entirely:

```inform6
Extend 'score' replace
    *                               -> Score
    * 'full'                        -> FullScore;
```

**`only`**: Splits one or more synonym words away from their shared
grammar table, giving them a new independent grammar. This is the
mechanism for making a single synonym behave differently from its
siblings:

```inform6
! 'x' normally shares grammar with 'examine', 'check', etc.
! Split it away and give it additional grammar:
Extend only 'x' first
    * noun 'with' held              -> ExamineWith;
```

After this directive, `x` has its own grammar table — initially a copy
of the original — with the new line prepended. The other synonyms
(`examine`, `check`, etc.) are unaffected.

### Extend with `meta` (6.43+)

When `$GRAMMAR_META_FLAG=1` is active, `Extend` supports the same
per-line `meta` keyword as `Verb`:

```inform6
Extend 'load'
    * 'game'                        -> Restore meta;
```

### Common Extension Idioms

Adding a preposition variant to an existing verb:

```inform6
Extend 'cut'
    * noun 'with' held              -> Cut;
```

Overriding a verb completely with `replace`:

```inform6
Extend 'jump' replace
    *                               -> Jump
    * 'over' noun                   -> JumpOver
    * 'across' noun                 -> JumpOver;
```

## 22.3 Grammar Lines and Token Types (Internal Details)

§21.3 describes grammar tokens from the game author's perspective. This
section documents the internal constants and data structures that the
compiler and parser use to represent them.

### Token Type Constants

The parser defines six token type categories in `parser.h`. Every token
in a grammar line is classified into one of these:

| Constant             | Value | Meaning                                     |
|----------------------|-------|---------------------------------------------|
| `ELEMENTARY_TT`      | 1     | A built-in noun-phrase token (see below)    |
| `PREPOSITION_TT`     | 2     | A literal dictionary word, e.g. `'into'`    |
| `ROUTINE_FILTER_TT`  | 3     | A filter routine, e.g. `noun=CagedCreature` |
| `ATTR_FILTER_TT`     | 4     | An attribute filter, e.g. `edible`          |
| `SCOPE_TT`           | 5     | A scope routine, e.g. `scope=Spells`        |
| `GPR_TT`             | 6     | A general parsing routine (GPR)             |

### Elementary Token Constants

Within the `ELEMENTARY_TT` category, individual tokens are identified by
these constants:

| Constant            | Value | Grammar keyword  | Meaning                           |
|---------------------|-------|------------------|-----------------------------------|
| `NOUN_TOKEN`        | 0     | `noun`           | Any single object in scope        |
| `HELD_TOKEN`        | 1     | `held`           | Object held by the player         |
| `MULTI_TOKEN`       | 2     | `multi`          | One or more objects in scope      |
| `MULTIHELD_TOKEN`   | 3     | `multiheld`      | One or more held objects          |
| `MULTIEXCEPT_TOKEN` | 4     | `multiexcept`    | Multiple objects except the other |
| `MULTIINSIDE_TOKEN` | 5     | `multiinside`    | Multiple objects inside the other |
| `CREATURE_TOKEN`    | 6     | `creature`       | An animate object                 |
| `SPECIAL_TOKEN`     | 7     | `special`        | Reads any single word from input  |
| `NUMBER_TOKEN`      | 8     | `number`         | A decimal number                  |
| `TOPIC_TOKEN`       | 9     | `topic`          | Remaining words as a topic        |
| `ENDIT_TOKEN`       | 15    | *(internal)*     | Marks end of token list (GV2)     |

### Grammar Table Encoding

The compiler writes grammar into a binary table whose format depends on
the grammar version. The version is set by the `$GRAMMAR_VERSION`
compiler option or by defining the constant `Grammar__Version` early in
the source. The standard library sets `Grammar__Version` to 2.

The grammar table is a word array starting at address `#grammar_table`.
It contains addresses of individual verb grammar tables. Each verb table
begins with a byte giving the number of grammar lines, followed by the
lines themselves.

**[Z-machine]** GV1 (grammar version 1) is the historical format.
Each grammar line is exactly 8 bytes: one byte for the parameter count,
six bytes for tokens (padded with zeros if fewer than six), and one byte
for the action number. GV1 limits verbs to six tokens per line and 256
actions.

**[Z-machine]** GV2 (grammar version 2) is the standard format used by
the library. Each grammar line begins with a two-byte word encoding the
action number in the bottom 10 bits and the `reverse` flag in bit 10.
Tokens follow as three-byte entries (one byte for token type, two bytes
for token data), terminated by an `ENDIT_TOKEN` (15) byte. Lines may
have a variable number of tokens.

**[Z-machine]** GV3 (grammar version 3, new in 6.43) is a compact
variation of GV2 designed to save bytes. The header word encodes the
action number in the bottom 10 bits, the `reverse` flag in bit 10, and
the token count in the top 5 bits. Since the count is in the header,
no `ENDIT_TOKEN` terminator is needed. Token data values are single
bytes rather than words; dictionary and routine addresses are looked up
in auxiliary tables. PunyInform uses GV3.

**[Glulx]** Only GV2 is supported. Each grammar line has a two-byte
action number, a one-byte flags field (bit 0 is the `reverse` flag),
then five-byte token entries (one byte for token type, four bytes for
token data), terminated by `ENDIT_TOKEN`.

### Unpacking Grammar Lines at Runtime

The parser unpacks each grammar line into three parallel arrays before
attempting to match it:

| Array          | Contents                                              |
|----------------|-------------------------------------------------------|
| `line_token`   | Raw token values (addresses or byte values)           |
| `line_ttype`   | Token type for each position (`ELEMENTARY_TT`, etc.)  |
| `line_tdata`   | Token data (dict address, routine address, etc.)      |

These are filled by `UnpackGrammarLine(address)`, which reads the
binary grammar data according to the active grammar version. It also
sets `action_to_be`, `action_reversed`, and `params_wanted`. The parser
then walks `line_ttype`/`line_tdata` to dispatch each token to the
appropriate matching logic.

## 22.4 Custom Token Routines (General Parsing Routines)

The grammar system allows game authors to write custom parsing logic
by placing routines directly into grammar lines. There are three
mechanisms: general parsing routines (GPRs), scope routines, and
attribute filters.

### General Parsing Routines (GPR_TT)

A GPR is a routine whose name appears directly as a token in a grammar
line (token type `GPR_TT`, value 6). When the parser reaches this token,
it calls the routine. The GPR reads words from the player's input using
`NextWord()` or `NextWordStopped()` and returns one of these values:

| Constant          | Value          | Meaning                                  |
|-------------------|----------------|------------------------------------------|
| `GPR_FAIL`        | −1             | Token did not match; try next line       |
| `GPR_PREPOSITION` | 0              | Matched, no object result (like a preposition) |
| `GPR_NUMBER`      | 1              | Matched; result is in `parsed_number`    |
| `GPR_MULTIPLE`    | 2              | Matched multiple objects                 |
| `GPR_REPARSE`     | `REPARSE_CODE` | Restart parsing from scratch             |
| *(object number)* | *(positive)*   | Matched that specific object             |

The value of `REPARSE_CODE` is 10000 on **[Z-machine]** and `$40000000`
on **[Glulx]**.

A GPR should advance `wn` past the words it consumes. If it returns
`GPR_FAIL`, the parser restores `wn` and tries the next grammar line.

### Example: Parsing a Time Expression

```inform6
[ TimeParsing x hr mn;
    x = NextWord();
    if (x == 'midnight') { parsed_number = 0; return GPR_NUMBER; }
    if (x == 'noon' or 'midday') { parsed_number = 720; return GPR_NUMBER; }
    ! Try to parse "H:MM" or "H"
    x = TryNumber(wn - 1);
    if (x == -1000) return GPR_FAIL;
    hr = x;
    if (NextWord() == ':') {
        mn = TryNumber(wn);
        if (mn == -1000) return GPR_FAIL;
        wn++;
    } else { wn--; mn = 0; }
    if (hr < 0 || hr > 23 || mn < 0 || mn > 59) return GPR_FAIL;
    parsed_number = hr * 60 + mn;
    return GPR_NUMBER;
];

Verb 'set'
    * noun 'to' TimeParsing         -> SetClock;
```

### Scope Routines (SCOPE_TT)

A scope token (`scope=Routine`) invokes a custom routine to define which
objects the parser considers as possible matches. As introduced in §21.3,
the routine is called at up to three stages via `scope_stage`:

**Stage 1** — Called before searching. Return `true` to print a custom
prompt, `false` for the default.

**Stage 2** — Add objects to scope by calling `PlaceInScope(obj)` for
individual objects or `ScopeWithin(domain)` for all children of a
container. Return `true` when finished.

**Stage 3** — Called when no match was found. Print an error and return
`true`, or return `false` for the parser's default error.

```inform6
[ MagicScope;
    switch (scope_stage) {
        1: rfalse;  ! use default prompt
        2: PlaceInScope(crystal_ball);
           PlaceInScope(magic_mirror);
           ScopeWithin(enchanted_cabinet);
           rtrue;
        3: "You sense no magical object like that.";
    }
];

Verb 'scry'
    * scope=MagicScope              -> Scry;
```

**`PlaceInScope(thing)`** adds a single object to the parser's candidate
list regardless of its actual location.

**`ScopeWithin(domain)`** adds every child of `domain` to the candidate
list.

### Attribute Filter Tokens (ATTR_FILTER_TT)

An attribute name used directly as a grammar token restricts matching
to objects that have the given attribute:

```inform6
Verb 'eat'
    * edible                        -> Eat;
```

This is equivalent to `noun` with a filter checking `(noun has edible)`,
but more concise. The compiler encodes it as `ATTR_FILTER_TT` (value 4)
with the attribute number as data.

### Routine Filter Tokens (ROUTINE_FILTER_TT)

The `noun=Routine` syntax (token type `ROUTINE_FILTER_TT`, value 3)
restricts matching to objects for which the named routine returns true.
The standard library uses this for directional filtering:

```inform6
[ ADirection; if (noun in Compass) rtrue; rfalse; ];

Verb 'go'
    * noun=ADirection               -> Go;
```

## 22.5 Action Routines

Every grammar line's `-> ActionName` clause links to an action routine
by naming convention: action `Foo` requires a routine named `FooSub`.
The compiler appends `Sub` to the action name; if `PhotographSub` does
not exist for `-> Photograph`, compilation fails.

### The `##ActionName` Constant

For every action declared (via grammar or `Fake_action`), the compiler
creates a constant `##ActionName` whose value is the action number.
These constants are used in `before`/`after` handlers and in direct
comparisons:

```inform6
before [;
    ##Take: "You can't take that!";
];

if (action == ##Examine) print "You look closely.^";
```

### Fake Actions

The `Fake_action` directive creates an action number constant without
requiring either a `Sub` routine or grammar. This is used for internal
signaling between `before`/`after` handlers — the action number exists
solely for use with `<< >>` or `action == ##Name` tests:

```inform6
Fake_action LetGo;
Fake_action Receive;
Fake_action ThrownAt;
```

See §20.6 for a full treatment of fake actions.

### The Dispatch Path

When the parser successfully matches a grammar line:

1. It sets `action`, `noun`, and `second` from the matched tokens.
2. If `action_reversed` is true with two parameters, `noun` and `second`
   are swapped (§22.6).
3. The library calls `ActionPrimitive()`, which runs the `before` chain.
4. If no handler intercepts, it calls the action's `Sub` routine.
5. The `after` chain runs.

See §20.1 for action variables and §20.7 for `<< >>` syntax.

### Writing a Custom Action

Define the `Sub` routine, then the `Verb` directive (placed after
`Include "Grammar"`):

```inform6
[ WaveAtSub;
    print "You wave at ", (the) second, ".^";
];

Verb 'wave' * 'at' creature -> WaveAt;
```

Custom grammar should be placed after `Include "Grammar";`.

## 22.6 Reverse Mode

The `reverse` keyword may appear after the action name in a grammar
line, instructing the parser to swap `noun` and `second` after parsing
but before the action fires.

When two parameters were parsed and `reverse` is active, the parser
swaps them:

```inform6
! In parser.h, after matching:
if (action_reversed && parameters == 2) {
    i = results-->2;
    results-->2 = results-->3;
    results-->3 = i;
}
```

The global `action_reversed` is set to `true` when a reversed line
matches. The standard library uses this for `Give` and `Show`:

```inform6
Verb 'give' 'feed' 'offer' 'pay'
    * creature held                 -> Give reverse
    * held 'to' creature            -> Give
    * 'over' held 'to' creature     -> Give;

Verb 'show' 'display' 'present'
    * creature held                 -> Show reverse
    * held 'to' creature            -> Show;
```

Consider GIVE TROLL SWORD: `* creature held` matches creature=troll,
held=sword. But `GiveSub` expects `noun`=object, `second`=recipient.
The `reverse` keyword swaps them. Without it, you would need separate
actions for different word orders.

### Encoding

In GV2 (Z-machine), the `reverse` flag is stored in bit 10 of the
action word in the grammar table. In Glulx, it is bit 0 of the flags
byte at offset +2 of the grammar line.

## 22.7 Standard Grammar Entries

The standard library defines its verbs in `grammar.h` (version 6.12.8).
The following tables provide a complete categorized reference.

### Meta Verbs (System Commands)

These are declared with `Verb meta` and produce meta actions that bypass
the world model (see §20.5).

| Verb Words               | Action(s)                    | Token Pattern(s)                  |
|--------------------------|------------------------------|-----------------------------------|
| `brief`                  | LMode1                       | `*`                               |
| `verbose` `long`         | LMode2                       | `*`                               |
| `superbrief` `short`     | LMode3                       | `*`                               |
| `normal`                 | LModeNormal                  | `*`                               |
| `notify`                 | NotifyOn, NotifyOff          | `*`, `* 'on'`, `* 'off'`         |
| `pronouns` `nouns`       | Pronouns                     | `*`                               |
| `quit` `q` `die`         | Quit                         | `*`                               |
| `recording`              | CommandsOn, CommandsOff      | `*`, `* 'on'`, `* 'off'`         |
| `replay`                 | CommandsRead                 | `*`                               |
| `restart`                | Restart                      | `*`                               |
| `restore`                | Restore                      | `*`                               |
| `save`                   | Save                         | `*`                               |
| `score`                  | Score                        | `*`                               |
| `fullscore` `full`       | FullScore                    | `*`, `* 'score'`                  |
| `script` `transcript`    | ScriptOn, ScriptOff          | `*`, `* 'on'`, `* 'off'`         |
| `noscript` `unscript`    | ScriptOff                    | `*`                               |
| `verify`                 | Verify                       | `*`                               |
| `version`                | Version                      | `*`                               |
| `objects`†               | Objects, ObjectsTall, ObjectsWide | `*`, `* 'tall'`, `* 'wide'`  |
| `places`†                | Places, PlacesTall, PlacesWide | `*`, `* 'tall'`, `* 'wide'`    |

† Conditional on `NO_PLACES` not being defined.

### Debug Verbs (Conditional on `DEBUG`)

All debug verbs are meta. These are only available when the game is
compiled with the `DEBUG` flag.

| Verb Words               | Action(s)                    | Token Pattern(s)                     |
|--------------------------|------------------------------|--------------------------------------|
| `actions`                | ActionsOn, ActionsOff        | `*`, `* 'on'`, `* 'off'`            |
| `changes`                | ChangesOn, ChangesOff        | `*`, `* 'on'`, `* 'off'`            |
| `gonear`                 | GoNear                       | `* anynumber`, `* noun`              |
| `goto`                   | Goto                         | `* anynumber`                        |
| `random`                 | Predictable                  | `*`                                  |
| `routines` `messages`    | RoutinesOn, RoutinesVerbose, RoutinesOff | `*`, `* 'on'`, `* 'verbose'`, `* 'off'` |
| `scope`                  | Scope                        | `*`, `* anynumber`, `* noun`         |
| `showdict` `dict`        | ShowDict                     | `*`, `* topic`                       |
| `showobj`                | ShowObj                      | `*`, `* anynumber`, `* multi`        |
| `showverb`               | ShowVerb                     | `* special`                          |
| `timers` `daemons`       | TimersOn, TimersOff          | `*`, `* 'on'`, `* 'off'`            |
| `trace`                  | TraceOn, TraceLevel, TraceOff| `*`, `* number`, `* 'on'`, `* 'off'` |
| `abstract`               | XAbstract                    | `* anynumber 'to' anynumber`, `* noun 'to' noun` |
| `purloin`                | XPurloin                     | `* anynumber`, `* multi`             |
| `tree`                   | XTree                        | `*`, `* anynumber`, `* noun`         |
| `glklist`‡               | Glklist                      | `*`                                  |

‡ **[Glulx]** only.

### Movement Verbs

| Verb Words               | Action(s)                    | Key Token Patterns                             |
|--------------------------|------------------------------|------------------------------------------------|
| `enter` `cross`          | GoIn, Enter                  | `*`, `* noun`                                  |
| `exit` `out` `outside`   | Exit                         | `*`, `* noun`                                  |
| `go` `run` `walk`        | VagueGo, Go, Enter, Exit, GoIn | `*`, `* noun=ADirection`, `* noun`, etc.    |
| `in` `inside`            | GoIn                         | `*`                                            |
| `leave`                  | VagueGo, Go, Exit, Enter     | `*`, `* noun=ADirection`, `* noun`, etc.       |

### Manipulation Verbs

| Verb Words               | Action(s)                    | Key Token Patterns                             |
|--------------------------|------------------------------|------------------------------------------------|
| `take`                   | Take, Disrobe, Remove, Inv   | `* multi`, `* 'off' multiheld`, etc.           |
| `drop` `discard`         | Drop, Insert, PutOn          | `* multiheld`, `* multiexcept 'in' noun`, etc. |
| `get`                    | Exit, Take, Enter, GetOff, Remove | various compound patterns                |
| `put`                    | Insert, PutOn, Wear, Drop    | `* multiexcept 'in' noun`, etc.                |
| `insert`                 | Insert                       | `* multiexcept 'in'/'into' noun`               |
| `remove`                 | Disrobe, Remove              | `* worn`, `* multiinside 'from' noun`          |
| `carry` `hold`           | Take, Remove, Inv            | `* multi`, `* multiinside 'from' noun`, etc.   |
| `pick`                   | Take                         | `* 'up' multi`, `* multi 'up'`                 |
| `peel`                   | Take                         | `* noun`, `* 'off' noun`                       |
| `give` `feed` `offer` `pay` | Give (reverse), Give      | `* creature held -> Give reverse`, etc.        |
| `show` `display` `present` | Show (reverse), Show       | `* creature held -> Show reverse`, etc.        |
| `transfer`               | Transfer, Take               | `* noun 'to' noun=NotPlayer/IsPlayer`          |
| `empty`                  | Empty, EmptyT                | `* noun`, `* noun 'to' noun`, etc.             |

### Examination Verbs

| Verb Words               | Action(s)                    | Key Token Patterns                             |
|--------------------------|------------------------------|------------------------------------------------|
| `examine` `x` `check` `describe` `watch` | Examine       | `* noun`                                       |
| `look` `l`               | Look, Examine, Search, LookUnder, Consult | `*`, `* 'at' noun`, `* 'in' noun`, etc. |
| `read`                   | Examine, Consult             | `* noun`, `* 'about' topic 'in' noun`, etc.    |
| `search`                 | Search                       | `* noun`                                       |
| `consult`                | Consult                      | `* noun 'about' topic`, `* noun 'on' topic`    |
| `inventory` `inv` `i`    | Inv, InvTall, InvWide        | `*`, `* 'tall'`, `* 'wide'`                    |

### Communication Verbs

| Verb Words               | Action(s)                    | Key Token Patterns                             |
|--------------------------|------------------------------|------------------------------------------------|
| `answer` `say` `shout` `speak` | Answer                 | `* topic 'to' creature`                        |
| `ask`                    | Ask, AskFor, AskTo           | `* creature 'about' topic`, `* creature 'for' noun`, etc. |
| `tell`                   | Tell, AskTo                  | `* creature 'about' topic`, `* creature 'to' topic` |

### Door/Container Verbs

| Verb Words               | Action(s)                    | Key Token Patterns                             |
|--------------------------|------------------------------|------------------------------------------------|
| `open` `uncover` `undo` `unwrap` | Open, Unlock           | `* noun`, `* noun 'with' held`                 |
| `close` `cover` `shut`   | Close, SwitchOff             | `* noun`, `* 'off' noun`                       |
| `lock`                   | Lock                         | `* noun 'with' held`                           |
| `unlock`                 | Unlock                       | `* noun 'with' held`                           |
| `pry` `prise` ... `force`| Unlock                       | `* noun 'with' held`, etc.                     |

### Physical Action Verbs

| Verb Words               | Action(s)                    | Key Token Patterns                             |
|--------------------------|------------------------------|------------------------------------------------|
| `push` `clear` `move` `press` `shift` | Push, PushDir, Transfer | `* noun`, `* noun noun`, `* noun 'to' noun` |
| `pull` `drag`            | Pull                         | `* noun`                                       |
| `turn` `rotate` `screw` `twist` `unscrew` | Turn, SwitchOn/Off | `* noun`, `* noun 'on'/'off'`, etc.       |
| `touch` `feel` `fondle` `grope` | Touch                 | `* noun`                                       |
| `squeeze` `squash`       | Squeeze                      | `* noun`                                       |
| `rub` `clean` ... `wipe` | Rub                          | `* noun`                                       |
| `cut` `chop` `prune` `slice` | Cut                      | `* noun`                                       |
| `tie` `attach` `connect` `fasten` `fix` | Tie            | `* noun`, `* noun 'to' noun`                   |
| `wave`                   | WaveHands, Wave              | `*`, `* noun`, `* noun 'at' noun`              |
| `blow`                   | Blow                         | `* held`                                       |
| `burn` `light`           | Burn                         | `* noun`, `* noun 'with' held`                 |
| `fill`                   | Fill                         | `* noun`, `* noun 'from' noun`                 |
| `dig`                    | Dig                          | `* noun`, `* noun 'with' held`, etc.           |
| `set` `adjust`           | Set, SetTo                   | `* noun`, `* noun 'to' special`                |
| `switch`                 | SwitchOn, SwitchOff          | `* noun`, `* noun 'on'/'off'`, etc.            |
| `swing`                  | Swing                        | `* noun`, `* 'on' noun`                        |
| `climb` `scale`          | Climb                        | `* noun`, `* 'up'/'over' noun`                 |

### Personal Action Verbs

| Verb Words               | Action(s)                    | Key Token Patterns                             |
|--------------------------|------------------------------|------------------------------------------------|
| `eat`                    | Eat                          | `* held`                                       |
| `drink` `sip` `swallow`  | Drink                        | `* noun`                                       |
| `taste`                  | Taste                        | `* noun`                                       |
| `smell` `sniff`          | Smell                        | `*`, `* noun`                                  |
| `listen` `hear`          | Listen                       | `*`, `* noun`, `* 'to' noun`                   |
| `wear` `don`             | Wear                         | `* multiheld`                                  |
| `disrobe` `doff` `shed`  | Disrobe                      | `* multi`                                      |
| `buy` `purchase`         | Buy                          | `* noun`                                       |
| `sleep` `nap`            | Sleep                        | `*`                                            |
| `wake` `awake` `awaken`  | Wake, WakeOther              | `*`, `* creature`, etc.                        |
| `sit` `lie`              | Enter                        | `* 'on' noun`, `* 'in' noun`                   |
| `stand`                  | Exit, Enter                  | `*`, `* 'on' noun`                             |
| `swim` `dive`            | Swim                         | `*`                                            |
| `jump` `hop` `skip`      | Jump, JumpIn, JumpOn, JumpOver | `*`, `* 'in' noun`, `* 'over' noun`          |
| `kiss` `embrace` `hug`   | Kiss                         | `* creature`                                   |
| `sing`                   | Sing                         | `*`                                            |
| `think`                  | Think                        | `*`                                            |
| `pray`                   | Pray                         | `*`                                            |
| `wait` `z`               | Wait                         | `*`                                            |
| `yes` `y`                | Yes                          | `*`                                            |
| `no`                     | No                           | `*`                                            |
| `sorry`                  | Sorry                        | `*`                                            |
| `attack` ... `wreck`     | Attack                       | `* noun`                                       |
| `throw`                  | ThrowAt                      | `* noun`, `* held 'at' noun`                   |

### Profanity Verbs

| Verb Words                   | Action  | Token Patterns     |
|------------------------------|---------|--------------------|
| `bother` `curses` `darn` `drat` | Mild | `*`, `* topic`     |
| `shit` `damn` `fuck` `sod`  | Strong  | `*`, `* topic`     |

### Helper Routines

Three small filter routines are provided for use in
grammar tokens:

```inform6
[ ADirection; if (noun in Compass) rtrue; rfalse; ];
[ NotPlayer; if (noun ~= player) rtrue; rfalse; ];
[ IsPlayer; if (noun == player) rtrue; rfalse; ];
```

**`ADirection`** — Returns true if `noun` is a compass direction object.
Used in `go`, `leave`, and `look` to match directional input.

**`NotPlayer`** — Returns true if `noun` is not the player. Used in
`transfer` to route `TRANSFER X TO NPC` to the `Transfer` action.

**`IsPlayer`** — Returns true if `noun` is the player. Used in
`transfer` to route `TRANSFER X TO ME` to the `Take` action.

## 22.8 Creating New Verbs and Grammar Lines

This section provides a step-by-step guide to creating custom verbs
and extending existing grammar.

### Step-by-Step Process

1. **Define the action routine** — write the `Sub` routine:

```inform6
[ PhotographSub;
    if (noun has animate)
        "The flash goes off. ", (The) noun, " blinks.";
    "You snap a photo of ", (the) noun, ".";
];
```

2. **Define the Verb directive** — list synonyms, then grammar lines:

```inform6
Verb 'photograph' 'photo' 'snap'
    * noun                          -> Photograph
    * noun 'with' held              -> Photograph;
```

3. **Placement** — custom grammar goes after all three library includes:

```inform6
Include "Parser";
Include "VerbLib";
Include "Grammar";

! Custom grammar goes here
[ PhotographSub; ... ];
Verb 'photograph' ...;
```

4. **Extending existing verbs** — use `Extend` after `Include "Grammar"`:

```inform6
Extend 'cut'
    * noun 'with' held              -> Cut;
```

### Complete Worked Example

A custom `PLAY` verb with an extension to `LISTEN`:

```inform6
[ PlaySub;
    print_ret "You play ", (the) noun, " beautifully.";
];

Verb 'play' 'perform'
    * held                          -> Play;

Extend 'listen'
    * 'to' 'performance'            -> ListenToPerformance;
```

### The ConTopic Helper Routine

The library provides `ConTopic` in `grammar.h` as a helper parsing
routine for topic-based input. Although no longer used by the library's
own grammar, it remains available for games. It consumes words from
the input until end-of-line or the word `'to'` (for `Answer`), setting
`consult_from` and `consult_words`:

```inform6
[ ConTopic w;
    consult_from = wn;
    do w = NextWordStopped();
    until (w == -1 || (w == 'to' && action_to_be == ##Answer));
    wn--;
    consult_words = wn - consult_from;
    if (consult_words == 0) return -1;
    if (action_to_be == ##Answer or ##Ask or ##Tell) {
        w = wn; wn = consult_from; parsed_number = NextWord();
        if (parsed_number == 'the' && consult_words > 1)
            parsed_number = NextWord();
        wn = w;
        return 1;
    }
    return 0;
];
```

This routine returns −1 if no words were consumed, 1 for
`Answer`/`Ask`/`Tell` (setting `parsed_number` to the first significant
word), and 0 otherwise.

### Stub Declarations

After all grammar, `grammar.h` provides `Stub` declarations for entry
point routines the library calls but the game need not define, including
`AfterLife`, `BeforeParsing`, `ChooseObjects`, `GamePreRoutine`,
`GamePostRoutine`, `InScope`, `ParseNumber`, `ParserError`, `PrintVerb`,
and `UnknownVerb`. If the game omits one, the stub provides a no-op.
