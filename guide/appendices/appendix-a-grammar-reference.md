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

# Appendix A: Complete Grammar Reference

This appendix provides a complete listing of all standard library grammar
lines and actions. Each entry is taken verbatim from the library source.
For a full explanation of grammar syntax, token types, and action
processing, see Chapter 20 (Actions), Chapter 21 (Parser and Grammar),
and Chapter 22 (Grammar Definition).

**How to read a grammar entry:**

- `Verb meta` — declares a meta-command (a game-control command rather than an
  in-game action). Meta-verbs do not consume a game turn.
- `Verb 'word1' 'word2'` — declares synonyms: all listed words share the same
  set of grammar lines.
- `* token1 token2 -> Action` — a grammar line that maps an input pattern to an
  action. The `*` marks the start of a pattern; tokens are matched left to
  right.
- `reverse` — after the `->` action, this keyword swaps the `noun` and `second`
  arguments before calling the action routine.
- `noun=Routine` — applies a filter routine to the `noun` token. The parser
  calls the routine for each candidate object and only accepts those for which
  it returns true.

---

## §A.1 Grammar Token Types

The following tokens may appear in grammar lines. Each token constrains what the
parser will accept in that position.

| Token | Description | Notes |
|-------|-------------|-------|
| `noun` | Matches any single object in scope | The most common token |
| `held` | Matches any object held by the player | Triggers implicit take if possible |
| `multi` | Matches one or more objects in scope | Used with "all" / "everything" |
| `multiheld` | Matches one or more objects held by the player | Triggers implicit take |
| `multiexcept` | Matches one or more objects except the other noun | Prevents "put X in X" |
| `multiinside` | Matches one or more objects inside another object | For removing from containers |
| `creature` | Matches an animate object in scope | Requires `animate` or `talkable` attribute |
| `topic` | Matches any sequence of words | Used for free-text input (ASK ABOUT, etc.) |
| `special` | Matches a single word without parsing | Returns raw dictionary word |
| `number` | Matches a number typed by the player | Parsed into `parsed_number` |
| `anynumber` | Matches a number or object number | Used in debugging verbs |
| `worn` | Matches an object currently worn by the player | Must have `worn` attribute |

---

## §A.2 Filter Routines

Three inline filter routines are defined at the top of the game verb section in
`grammar.h`. These are used with the `noun=Routine` syntax to restrict which
objects a `noun` token will accept.

```inform6
[ ADirection; if (noun in Compass) rtrue; rfalse; ];
[ NotPlayer; if (noun ~= player) rtrue; rfalse; ];
[ IsPlayer; if (noun == player) rtrue; rfalse; ];
```

- `noun=ADirection` — accepts only compass direction objects (children of the
  `Compass` object). Used by `go`, `leave`, and `look` to distinguish
  directional movement from entering a named object.
- `noun=NotPlayer` — accepts any object except the player. Used by `transfer`
  to route `TRANSFER X TO NPC` to the `Transfer` action.
- `noun=IsPlayer` — accepts only the player object. Used by `transfer` to route
  `TRANSFER X TO ME` to the `Take` action instead.

---

## §A.3 Meta-Verb Grammar

Meta-verbs are game-control commands rather than in-world actions. They do not
consume a game turn and are declared with `Verb meta`. The parser sets the
`meta` flag when one of these verbs is matched, which prevents `before` and
`after` rules from intercepting the action.

### §A.3.1 Display Mode Verbs

```inform6
Verb meta 'brief'
    *                                           -> LMode1;

Verb meta 'verbose' 'long'
    *                                           -> LMode2;

Verb meta 'superbrief' 'short'
    *                                           -> LMode3;

Verb meta 'normal'
    *                                           -> LModeNormal;
```

### §A.3.2 Notification and Score

```inform6
Verb meta 'notify'
    *                                           -> NotifyOn
    * 'on'                                      -> NotifyOn
    * 'off'                                     -> NotifyOff;

Verb meta 'score'
    *                                           -> Score;

Verb meta 'fullscore' 'full'
    *                                           -> FullScore
    * 'score'                                   -> FullScore;
```

### §A.3.3 Transcript and Recording

```inform6
Verb meta 'script' 'transcript'
    *                                           -> ScriptOn
    * 'on'                                      -> ScriptOn
    * 'off'                                     -> ScriptOff;

Verb meta 'noscript' 'unscript'
    *                                           -> ScriptOff;

Verb meta 'recording'
    *                                           -> CommandsOn
    * 'on'                                      -> CommandsOn
    * 'off'                                     -> CommandsOff;

Verb meta 'replay'
    *                                           -> CommandsRead;
```

### §A.3.4 Game Control

```inform6
Verb meta 'quit' 'q//' 'die'
    *                                           -> Quit;

Verb meta 'restart'
    *                                           -> Restart;

Verb meta 'restore'
    *                                           -> Restore;

Verb meta 'save'
    *                                           -> Save;

Verb meta 'verify'
    *                                           -> Verify;

Verb meta 'version'
    *                                           -> Version;
```

### §A.3.5 Pronoun Tracking

```inform6
Verb meta 'pronouns' 'nouns'
    *                                           -> Pronouns;
```

### §A.3.6 Places and Objects [conditional on NO_PLACES]

These verbs are included only when `NO_PLACES` is not defined (the default). If
`NO_PLACES` is defined, `Places` and `Objects` become fake actions instead (see
§A.6).

```inform6
#Ifndef NO_PLACES;
Verb meta 'objects'
    *                                           -> Objects
    * 'tall'                                    -> ObjectsTall
    * 'wide'                                    -> ObjectsWide;
Verb meta 'places'
    *                                           -> Places
    * 'tall'                                    -> PlacesTall
    * 'wide'                                    -> PlacesWide;
#Endif; ! NO_PLACES
```

### §A.3.7 Debugging Verbs [DEBUG only]

These verbs are only available when the game is compiled with the `DEBUG` flag
(compiler switch `-D`). They provide tools for inspecting and manipulating the
game state during testing.

```inform6
#Ifdef DEBUG;
Verb meta 'actions'
    *                                           -> ActionsOn
    * 'on'                                      -> ActionsOn
    * 'off'                                     -> ActionsOff;

Verb meta 'changes'
    *                                           -> ChangesOn
    * 'on'                                      -> ChangesOn
    * 'off'                                     -> ChangesOff;

Verb meta 'gonear'
    * anynumber                                 -> GoNear
    * noun                                      -> GoNear;

Verb meta 'goto'
    * anynumber                                 -> Goto;

Verb meta 'random'
    *                                           -> Predictable;

Verb meta 'routines' 'messages'
    *                                           -> RoutinesOn
    * 'on'                                      -> RoutinesOn
    * 'verbose'                                 -> RoutinesVerbose
    * 'off'                                     -> RoutinesOff;

Verb meta 'scope'
    *                                           -> Scope
    * anynumber                                 -> Scope
    * noun                                      -> Scope;

Verb meta 'showdict' 'dict'
    *                                           -> ShowDict
    * topic                                     -> ShowDict;

Verb meta 'showobj'
    *                                           -> ShowObj
    * anynumber                                 -> ShowObj
    * multi                                     -> ShowObj;

Verb meta 'showverb'
    * special                                   -> ShowVerb;

Verb meta 'timers' 'daemons'
    *                                           -> TimersOn
    * 'on'                                      -> TimersOn
    * 'off'                                     -> TimersOff;

Verb meta 'trace'
    *                                           -> TraceOn
    * number                                    -> TraceLevel
    * 'on'                                      -> TraceOn
    * 'off'                                     -> TraceOff;

Verb meta 'abstract'
    * anynumber 'to' anynumber                  -> XAbstract
    * noun 'to' noun                            -> XAbstract;

Verb meta 'purloin'
    * anynumber                                 -> XPurloin
    * multi                                     -> XPurloin;

Verb meta 'tree'
    *                                           -> XTree
    * anynumber                                 -> XTree
    * noun                                      -> XTree;
```

The following debugging verb is available only on the Glulx platform:

```inform6
#Ifdef TARGET_GLULX;
Verb meta 'glklist'
    *                                           -> Glklist;
#Endif; ! TARGET_GLULX
```

```inform6
#Endif; ! DEBUG
```

---

## §A.4 Game Verb Grammar

The following are in-game verbs — actions performed by the player character
within the game world. Each entry shows the `Verb` declaration exactly as it
appears in the library (6.12.8), preserving synonym lists, grammar
lines, and all tokens. Entries are listed in alphabetical order by primary verb
word.

**answer** (synonyms: say, shout, speak):

```inform6
Verb 'answer' 'say' 'shout' 'speak'
    * topic 'to' creature                       -> Answer;
```

**ask**:

```inform6
Verb 'ask'
    * creature 'about' topic                    -> Ask
    * creature 'for' noun                       -> AskFor
    * creature 'to' topic                       -> AskTo
    * 'that' creature topic                     -> AskTo;
```

**attack** (synonyms: break, crack, destroy, fight, hit, kill, murder, punch,
smash, thump, torture, wreck):

```inform6
Verb 'attack' 'break' 'crack' 'destroy'
     'fight' 'hit' 'kill' 'murder' 'punch'
     'smash' 'thump' 'torture' 'wreck'
    * noun                                      -> Attack;
```

**blow**:

```inform6
Verb 'blow'
    * held                                      -> Blow;
```

**bother** (synonyms: curses, darn, drat):

```inform6
Verb 'bother' 'curses' 'darn' 'drat'
    *                                           -> Mild
    * topic                                     -> Mild;
```

**burn** (synonyms: light):

```inform6
Verb 'burn' 'light'
    * noun                                      -> Burn
    * noun 'with' held                          -> Burn;
```

**buy** (synonyms: purchase):

```inform6
Verb 'buy' 'purchase'
    * noun                                      -> Buy;
```

**carry**:

```inform6
Verb 'carry'
    * multi                                     -> Take
    * multiinside 'from'/'off' noun             -> Remove
    * 'inventory'                               -> Inv;
```

**climb** (synonyms: scale):

```inform6
Verb 'climb' 'scale'
    * noun                                      -> Climb
    * 'up'/'over' noun                          -> Climb;
```

**close** (synonyms: cover, shut):

```inform6
Verb 'close' 'cover' 'shut'
    * noun                                      -> Close
    * 'up' noun                                 -> Close
    * 'off' noun                                -> SwitchOff;
```

**consult**:

```inform6
Verb 'consult'
    * noun 'about' topic                        -> Consult
    * noun 'on' topic                           -> Consult;
```

**cut** (synonyms: chop, prune, slice):

```inform6
Verb 'cut' 'chop' 'prune' 'slice'
    * noun                                      -> Cut;
```

**dig**:

```inform6
Verb 'dig'
    * noun                                      -> Dig
    * noun 'with' held                          -> Dig
    * 'in' noun                                 -> Dig
    * 'in' noun 'with' held                     -> Dig;
```

**disrobe** (synonyms: doff, shed):

```inform6
Verb 'disrobe' 'doff' 'shed'
    * multi                                     -> Disrobe;
```

**drink** (synonyms: sip, swallow):

```inform6
Verb 'drink' 'sip' 'swallow'
    * noun                                      -> Drink;
```

**drop** (synonyms: discard):

```inform6
Verb 'drop' 'discard'
    * multiheld                                 -> Drop
    * multiexcept 'in'/'into'/'down' noun       -> Insert
    * multiexcept 'on'/'onto' noun              -> PutOn;
```

**eat**:

```inform6
Verb 'eat'
    * held                                      -> Eat;
```

**empty**:

```inform6
Verb 'empty'
    * noun                                      -> Empty
    * 'out' noun                                -> Empty
    * noun 'out'                                -> Empty
    * noun 'to'/'into'/'on'/'onto' noun         -> EmptyT;
```

**enter** (synonyms: cross):

```inform6
Verb 'enter' 'cross'
    *                                           -> GoIn
    * noun                                      -> Enter;
```

**examine** (synonyms: x, check, describe, watch):

```inform6
Verb 'examine' 'x//' 'check' 'describe' 'watch'
    * noun                                      -> Examine;
```

**exit** (synonyms: out, outside):

```inform6
Verb 'exit' 'out' 'outside'
    *                                           -> Exit
    * noun                                      -> Exit;
```

**fill**:

```inform6
Verb 'fill'
    * noun                                      -> Fill
    * noun 'from' noun                          -> Fill;
```

**get**:

```inform6
Verb 'get'
    * 'out'/'off'/'up' 'of'/'from' noun         -> Exit
    * multi                                     -> Take
    * 'in'/'into'/'on'/'onto' noun              -> Enter
    * 'off' noun                                -> GetOff
    * multiinside 'from'/'off' noun             -> Remove;
```

**give** (synonyms: feed, offer, pay) — the first grammar line uses `reverse`
to swap noun and second so that `GIVE TROLL SWORD` is parsed as giving the
sword to the troll:

```inform6
Verb 'give' 'feed' 'offer' 'pay'
    * creature held                             -> Give reverse
    * held 'to' creature                        -> Give
    * 'over' held 'to' creature                 -> Give;
```

**go** (synonyms: run, walk) — uses `noun=ADirection` filter to distinguish
compass directions from enterable objects:

```inform6
Verb 'go' 'run' 'walk'
    *                                           -> VagueGo
    * noun=ADirection                           -> Go
    * noun                                      -> Enter
    * 'out'/'outside'                           -> Exit
    * 'in'/'inside'                             -> GoIn
    * 'into'/'in'/'inside'/'through' noun       -> Enter;
```

**hold**:

```inform6
Verb 'hold'
    * multi                                     -> Take
    * multiinside 'from'/'off' noun             -> Remove
    * 'inventory'                               -> Inv;
```

**in** (synonyms: inside):

```inform6
Verb 'in' 'inside'
    *                                           -> GoIn;
```

**insert**:

```inform6
Verb 'insert'
    * multiexcept 'in'/'into' noun              -> Insert;
```

**inventory** (synonyms: inv, i):

```inform6
Verb 'inventory' 'inv' 'i//'
    *                                           -> Inv
    * 'tall'                                    -> InvTall
    * 'wide'                                    -> InvWide;
```

**jump** (synonyms: hop, skip):

```inform6
Verb 'jump' 'hop' 'skip'
    *                                           -> Jump
    * 'in' noun                                 -> JumpIn
    * 'into' noun                               -> JumpIn
    * 'on' noun                                 -> JumpOn
    * 'upon' noun                               -> JumpOn
    * 'over' noun                               -> JumpOver;
```

**kiss** (synonyms: embrace, hug):

```inform6
Verb 'kiss' 'embrace' 'hug'
    * creature                                  -> Kiss;
```

**leave** — uses `noun=ADirection` filter:

```inform6
Verb 'leave'
    *                                           -> VagueGo
    * noun=ADirection                           -> Go
    * noun                                      -> Exit
    * 'into'/'in'/'inside'/'through' noun       -> Enter;
```

**listen** (synonyms: hear):

```inform6
Verb 'listen' 'hear'
    *                                           -> Listen
    * noun                                      -> Listen
    * 'to' noun                                 -> Listen;
```

**lock**:

```inform6
Verb 'lock'
    * noun 'with' held                          -> Lock;
```

**look** (synonyms: l) — uses `noun=ADirection` filter for directional
examining:

```inform6
Verb 'look' 'l//'
    *                                           -> Look
    * 'at' noun                                 -> Examine
    * 'inside'/'in'/'into'/'through'/'on' noun  -> Search
    * 'under' noun                              -> LookUnder
    * 'up' topic 'in' noun                      -> Consult
    * noun=ADirection                           -> Examine
    * 'to' noun=ADirection                      -> Examine;
```

**no**:

```inform6
Verb 'no'
    *                                           -> No;
```

**open** (synonyms: uncover, undo, unwrap):

```inform6
Verb 'open' 'uncover' 'undo' 'unwrap'
    * noun                                      -> Open
    * noun 'with' held                          -> Unlock;
```

**peel**:

```inform6
Verb 'peel'
    * noun                                      -> Take
    * 'off' noun                                -> Take;
```

**pick**:

```inform6
Verb 'pick'
    * 'up' multi                                -> Take
    * multi 'up'                                -> Take;
```

**pray**:

```inform6
Verb 'pray'
    *                                           -> Pray;
```

**pry** (synonyms: prise, prize, lever, jemmy, force):

```inform6
Verb 'pry' 'prise' 'prize' 'lever' 'jemmy' 'force'
    * noun 'with' held                          -> Unlock
    * 'apart'/'open' noun 'with' held           -> Unlock
    * noun 'apart'/'open' 'with' held           -> Unlock;
```

**pull** (synonyms: drag):

```inform6
Verb 'pull' 'drag'
    * noun                                      -> Pull;
```

**push** (synonyms: clear, move, press, shift):

```inform6
Verb 'push' 'clear' 'move' 'press' 'shift'
    * noun                                      -> Push
    * noun noun                                 -> PushDir
    * noun 'to' noun                            -> Transfer;
```

**put**:

```inform6
Verb 'put'
    * multiexcept 'in'/'inside'/'into' noun     -> Insert
    * multiexcept 'on'/'onto' noun              -> PutOn
    * 'on' multiheld                            -> Wear
    * 'down' multiheld                          -> Drop
    * multiheld 'down'                          -> Drop;
```

**read**:

```inform6
Verb 'read'
    * noun                                      -> Examine
    * 'about' topic 'in' noun                   -> Consult
    * topic 'in' noun                           -> Consult;
```

**remove**:

```inform6
Verb 'remove'
    * worn                                      -> Disrobe
    * multi                                     -> Remove
    * multiinside 'from' noun                   -> Remove;
```

**rub** (synonyms: clean, dust, polish, scrub, shine, sweep, wipe):

```inform6
Verb 'rub' 'clean' 'dust' 'polish' 'scrub'
     'shine' 'sweep' 'wipe'
    * noun                                      -> Rub;
```

**search**:

```inform6
Verb 'search'
    * noun                                      -> Search;
```

**set** (synonyms: adjust):

```inform6
Verb 'set' 'adjust'
    * noun                                      -> Set
    * noun 'to' special                         -> SetTo;
```

**shit** (synonyms: damn, fuck, sod):

```inform6
Verb 'shit' 'damn' 'fuck' 'sod'
    *                                           -> Strong
    * topic                                     -> Strong;
```

**show** (synonyms: display, present) — the first grammar line uses `reverse`:

```inform6
Verb 'show' 'display' 'present'
    * creature held                             -> Show reverse
    * held 'to' creature                        -> Show;
```

**sing**:

```inform6
Verb 'sing'
    *                                           -> Sing;
```

**sit** (synonyms: lie):

```inform6
Verb 'sit' 'lie'
    * 'on' 'top' 'of' noun                      -> Enter
    * 'on'/'in'/'inside' noun                   -> Enter;
```

**sleep** (synonyms: nap):

```inform6
Verb 'sleep' 'nap'
    *                                           -> Sleep;
```

**smell** (synonyms: sniff):

```inform6
Verb 'smell' 'sniff'
    *                                           -> Smell
    * noun                                      -> Smell;
```

**sorry**:

```inform6
Verb 'sorry'
    *                                           -> Sorry;
```

**squeeze** (synonyms: squash):

```inform6
Verb 'squeeze' 'squash'
    * noun                                      -> Squeeze;
```

**stand**:

```inform6
Verb 'stand'
    *                                           -> Exit
    * 'up'                                      -> Exit
    * 'on' noun                                 -> Enter;
```

**swim** (synonyms: dive):

```inform6
Verb 'swim' 'dive'
    *                                           -> Swim;
```

**swing**:

```inform6
Verb 'swing'
    * noun                                      -> Swing
    * 'on' noun                                 -> Swing;
```

**switch**:

```inform6
Verb 'switch'
    * noun                                      -> SwitchOn
    * noun 'on'                                 -> SwitchOn
    * noun 'off'                                -> SwitchOff
    * 'on' noun                                 -> SwitchOn
    * 'off' noun                                -> SwitchOff;
```

**take**:

```inform6
Verb 'take'
    * multi                                     -> Take
    * 'off' multiheld                           -> Disrobe
    * multiinside 'from'/'off' noun             -> Remove
    * 'inventory'                               -> Inv;
```

**taste**:

```inform6
Verb 'taste'
    * noun                                      -> Taste;
```

**tell**:

```inform6
Verb 'tell'
    * creature 'about' topic                    -> Tell
    * creature 'to' topic                       -> AskTo;
```

**think**:

```inform6
Verb 'think'
    *                                           -> Think;
```

**throw**:

```inform6
Verb 'throw'
    * noun                                      -> ThrowAt
    * held 'at'/'against'/'on'/'onto' noun      -> ThrowAt;
```

**tie** (synonyms: attach, connect, fasten, fix):

```inform6
Verb 'tie' 'attach' 'connect' 'fasten' 'fix'
    * noun                                      -> Tie
    * noun 'to' noun                            -> Tie;
```

**touch** (synonyms: feel, fondle, grope):

```inform6
Verb 'touch' 'feel' 'fondle' 'grope'
    * noun                                      -> Touch;
```

**transfer** — uses `noun=NotPlayer` and `noun=IsPlayer` filter routines to
distinguish between transferring to an NPC and taking for yourself:

```inform6
Verb 'transfer'
    * noun 'to' noun=NotPlayer                  -> Transfer
    * noun 'to' noun=IsPlayer                  -> Take;
```

**turn** (synonyms: rotate, screw, twist, unscrew):

```inform6
Verb 'turn' 'rotate' 'screw' 'twist' 'unscrew'
    * noun                                      -> Turn
    * noun 'on'                                 -> SwitchOn
    * noun 'off'                                -> SwitchOff
    * 'on' noun                                 -> SwitchOn
    * 'off' noun                                -> SwitchOff;
```

**unlock**:

```inform6
Verb 'unlock'
    * noun 'with' held                          -> Unlock;
```

**wait** (synonyms: z):

```inform6
Verb 'wait' 'z//'
    *                                           -> Wait;
```

**wake** (synonyms: awake, awaken):

```inform6
Verb 'wake' 'awake' 'awaken'
    *                                           -> Wake
    * 'up'                                      -> Wake
    * creature                                  -> WakeOther
    * creature 'up'                             -> WakeOther
    * 'up' creature                             -> WakeOther;
```

**wave**:

```inform6
Verb 'wave'
    *                                           -> WaveHands
    * noun                                      -> Wave
    * noun 'at' noun                            -> Wave
    * 'at' noun                                 -> WaveHands;
```

**wear** (synonyms: don):

```inform6
Verb 'wear' 'don'
    * multiheld                                 -> Wear;
```

**yes** (synonyms: y):

```inform6
Verb 'yes' 'y//'
    *                                           -> Yes;
```

---

## §A.5 Complete Action Index

The following table lists every action that appears as a target (`-> Action`) in
`grammar.h`, sorted alphabetically. The **Meta?** column indicates whether the
action is a meta-action (game control, not an in-world action). The **Sub
Routine** column gives the name of the routine the library calls to handle the
action. The **Source** column indicates which library file defines the sub
routine.

| Action | Meta? | Verb(s) | Sub Routine | Source |
|--------|-------|---------|-------------|--------|
| `ActionsOff` | Yes | actions | `ActionsOffSub` | verblib.h |
| `ActionsOn` | Yes | actions | `ActionsOnSub` | verblib.h |
| `Answer` | No | answer, say, shout, speak | `AnswerSub` | verblib.h |
| `Ask` | No | ask | `AskSub` | verblib.h |
| `AskFor` | No | ask | `AskForSub` | verblib.h |
| `AskTo` | No | ask, tell | `AskToSub` | verblib.h |
| `Attack` | No | attack, break, crack, destroy, fight, hit, kill, murder, punch, smash, thump, torture, wreck | `AttackSub` | verblib.h |
| `Blow` | No | blow | `BlowSub` | verblib.h |
| `Burn` | No | burn, light | `BurnSub` | verblib.h |
| `Buy` | No | buy, purchase | `BuySub` | verblib.h |
| `ChangesOff` | Yes | changes | `ChangesOffSub` | verblib.h |
| `ChangesOn` | Yes | changes | `ChangesOnSub` | verblib.h |
| `Climb` | No | climb, scale | `ClimbSub` | verblib.h |
| `Close` | No | close, cover, shut | `CloseSub` | verblib.h |
| `CommandsOff` | Yes | recording | `CommandsOffSub` | verblib.h |
| `CommandsOn` | Yes | recording | `CommandsOnSub` | verblib.h |
| `CommandsRead` | Yes | replay | `CommandsReadSub` | verblib.h |
| `Consult` | No | consult, look, l, read | `ConsultSub` | verblib.h |
| `Cut` | No | cut, chop, prune, slice | `CutSub` | verblib.h |
| `Dig` | No | dig | `DigSub` | verblib.h |
| `Disrobe` | No | disrobe, doff, shed, remove, take | `DisrobeSub` | verblib.h |
| `Drink` | No | drink, sip, swallow | `DrinkSub` | verblib.h |
| `Drop` | No | drop, discard, put | `DropSub` | verblib.h |
| `Eat` | No | eat | `EatSub` | verblib.h |
| `Empty` | No | empty | `EmptySub` | verblib.h |
| `EmptyT` | No | empty | `EmptyTSub` | verblib.h |
| `Enter` | No | enter, cross, get, go, run, walk, leave, sit, lie, stand | `EnterSub` | verblib.h |
| `Examine` | No | examine, x, check, describe, watch, look, l, read | `ExamineSub` | verblib.h |
| `Exit` | No | exit, out, outside, get, go, run, walk, leave, stand | `ExitSub` | verblib.h |
| `Fill` | No | fill | `FillSub` | verblib.h |
| `FullScore` | Yes | fullscore, full | `FullScoreSub` | verblib.h |
| `GetOff` | No | get | `GetOffSub` | verblib.h |
| `Give` | No | give, feed, offer, pay | `GiveSub` | verblib.h |
| `Glklist` | Yes | glklist | `GlkListSub` | verblib.h |
| `Go` | No | go, run, walk, leave | `GoSub` | verblib.h |
| `GoIn` | No | enter, cross, go, run, walk, in, inside | `GoInSub` | verblib.h |
| `GoNear` | Yes | gonear | `GoNearSub` | verblib.h |
| `Goto` | Yes | goto | `GotoSub` | verblib.h |
| `Insert` | No | insert, drop, discard, put | `InsertSub` | verblib.h |
| `Inv` | No | inventory, inv, i, carry, hold, take | `InvSub` | verblib.h |
| `InvTall` | No | inventory, inv, i | `InvTallSub` | verblib.h |
| `InvWide` | No | inventory, inv, i | `InvWideSub` | verblib.h |
| `Jump` | No | jump, hop, skip | `JumpSub` | verblib.h |
| `JumpIn` | No | jump, hop, skip | `JumpInSub` | verblib.h |
| `JumpOn` | No | jump, hop, skip | `JumpOnSub` | verblib.h |
| `JumpOver` | No | jump, hop, skip | `JumpOverSub` | verblib.h |
| `Kiss` | No | kiss, embrace, hug | `KissSub` | verblib.h |
| `Listen` | No | listen, hear | `ListenSub` | verblib.h |
| `LMode1` | Yes | brief | `LMode1Sub` | verblib.h |
| `LMode2` | Yes | verbose, long | `LMode2Sub` | verblib.h |
| `LMode3` | Yes | superbrief, short | `LMode3Sub` | verblib.h |
| `LModeNormal` | Yes | normal | `LModeNormalSub` | verblib.h |
| `Lock` | No | lock | `LockSub` | verblib.h |
| `Look` | No | look, l | `LookSub` | verblib.h |
| `LookUnder` | No | look, l | `LookUnderSub` | verblib.h |
| `Mild` | No | bother, curses, darn, drat | `MildSub` | verblib.h |
| `No` | No | no | `NoSub` | verblib.h |
| `NotifyOff` | Yes | notify | `NotifyOffSub` | verblib.h |
| `NotifyOn` | Yes | notify | `NotifyOnSub` | verblib.h |
| `Objects` | Yes | objects | `ObjectsSub` | verblib.h |
| `ObjectsTall` | Yes | objects | `ObjectsTallSub` | verblib.h |
| `ObjectsWide` | Yes | objects | `ObjectsWideSub` | verblib.h |
| `Open` | No | open, uncover, undo, unwrap | `OpenSub` | verblib.h |
| `Places` | Yes | places | `PlacesSub` | verblib.h |
| `PlacesTall` | Yes | places | `PlacesTallSub` | verblib.h |
| `PlacesWide` | Yes | places | `PlacesWideSub` | verblib.h |
| `Pray` | No | pray | `PraySub` | verblib.h |
| `Predictable` | Yes | random | `PredictableSub` | verblib.h |
| `Pronouns` | Yes | pronouns, nouns | `PronounsSub` | parser.h |
| `Pull` | No | pull, drag | `PullSub` | verblib.h |
| `Push` | No | push, clear, move, press, shift | `PushSub` | verblib.h |
| `PushDir` | No | push, clear, move, press, shift | `PushDirSub` | verblib.h |
| `PutOn` | No | put, drop, discard | `PutOnSub` | verblib.h |
| `Quit` | Yes | quit, q, die | `QuitSub` | verblib.h |
| `Remove` | No | remove, carry, get, hold, take | `RemoveSub` | verblib.h |
| `Restart` | Yes | restart | `RestartSub` | verblib.h |
| `Restore` | Yes | restore | `RestoreSub` | verblib.h |
| `RoutinesOff` | Yes | routines, messages | `RoutinesOffSub` | verblib.h |
| `RoutinesOn` | Yes | routines, messages | `RoutinesOnSub` | verblib.h |
| `RoutinesVerbose` | Yes | routines, messages | `RoutinesVerboseSub` | verblib.h |
| `Rub` | No | rub, clean, dust, polish, scrub, shine, sweep, wipe | `RubSub` | verblib.h |
| `Save` | Yes | save | `SaveSub` | verblib.h |
| `Scope` | Yes | scope | `ScopeSub` | verblib.h |
| `Score` | Yes | score | `ScoreSub` | verblib.h |
| `ScriptOff` | Yes | script, transcript, noscript, unscript | `ScriptOffSub` | verblib.h |
| `ScriptOn` | Yes | script, transcript | `ScriptOnSub` | verblib.h |
| `Search` | No | search, look, l | `SearchSub` | verblib.h |
| `Set` | No | set, adjust | `SetSub` | verblib.h |
| `SetTo` | No | set, adjust | `SetToSub` | verblib.h |
| `Show` | No | show, display, present | `ShowSub` | verblib.h |
| `ShowDict` | Yes | showdict, dict | `ShowDictSub` | parser.h |
| `ShowObj` | Yes | showobj | `ShowObjSub` | parser.h |
| `ShowVerb` | Yes | showverb | `ShowVerbSub` | parser.h |
| `Sing` | No | sing | `SingSub` | verblib.h |
| `Sleep` | No | sleep, nap | `SleepSub` | verblib.h |
| `Smell` | No | smell, sniff | `SmellSub` | verblib.h |
| `Sorry` | No | sorry | `SorrySub` | verblib.h |
| `Squeeze` | No | squeeze, squash | `SqueezeSub` | verblib.h |
| `Strong` | No | shit, damn, fuck, sod | `StrongSub` | verblib.h |
| `Swim` | No | swim, dive | `SwimSub` | verblib.h |
| `Swing` | No | swing | `SwingSub` | verblib.h |
| `SwitchOff` | No | switch, close, cover, shut, turn, rotate, screw, twist, unscrew | `SwitchOffSub` | verblib.h |
| `SwitchOn` | No | switch, turn, rotate, screw, twist, unscrew | `SwitchOnSub` | verblib.h |
| `Take` | No | take, carry, get, hold, peel, pick, transfer | `TakeSub` | verblib.h |
| `Taste` | No | taste | `TasteSub` | verblib.h |
| `Tell` | No | tell | `TellSub` | verblib.h |
| `Think` | No | think | `ThinkSub` | verblib.h |
| `ThrowAt` | No | throw | `ThrowAtSub` | verblib.h |
| `Tie` | No | tie, attach, connect, fasten, fix | `TieSub` | verblib.h |
| `TimersOff` | Yes | timers, daemons | `TimersOffSub` | verblib.h |
| `TimersOn` | Yes | timers, daemons | `TimersOnSub` | verblib.h |
| `Touch` | No | touch, feel, fondle, grope | `TouchSub` | verblib.h |
| `TraceLevel` | Yes | trace | `TraceLevelSub` | verblib.h |
| `TraceOff` | Yes | trace | `TraceOffSub` | verblib.h |
| `TraceOn` | Yes | trace | `TraceOnSub` | verblib.h |
| `Transfer` | No | transfer, push, clear, move, press, shift | `TransferSub` | verblib.h |
| `Turn` | No | turn, rotate, screw, twist, unscrew | `TurnSub` | verblib.h |
| `Unlock` | No | unlock, open, uncover, undo, unwrap, pry, prise, prize, lever, jemmy, force | `UnlockSub` | verblib.h |
| `VagueGo` | No | go, run, walk, leave | `VagueGoSub` | verblib.h |
| `Verify` | Yes | verify | `VerifySub` | verblib.h |
| `Version` | Yes | version | `VersionSub` | verblib.h |
| `Wait` | No | wait, z | `WaitSub` | verblib.h |
| `Wake` | No | wake, awake, awaken | `WakeSub` | verblib.h |
| `WakeOther` | No | wake, awake, awaken | `WakeOtherSub` | verblib.h |
| `Wave` | No | wave | `WaveSub` | verblib.h |
| `WaveHands` | No | wave | `WaveHandsSub` | verblib.h |
| `Wear` | No | wear, don, put | `WearSub` | verblib.h |
| `XAbstract` | Yes | abstract | `XAbstractSub` | verblib.h |
| `XPurloin` | Yes | purloin | `XPurloinSub` | verblib.h |
| `XTree` | Yes | tree | `XTreeSub` | verblib.h |
| `Yes` | No | yes, y | `YesSub` | verblib.h |

---

## §A.6 Fake Actions

Fake actions are action constants that have no grammar lines and no `Verb`
declarations. They are used internally by the parser and library to send
messages to `life` routines, handle disambiguation, and print library messages.
They are declared with `Fake_Action` in `parser.h`.

| Fake Action | Defined In | Purpose |
|-------------|-----------|---------|
| `LetGo` | parser.h | Sent to `life` when an NPC is asked to release an object |
| `Receive` | parser.h | Sent to `life` when an object is given or shown to an NPC |
| `ThrownAt` | parser.h | Sent to `life` when an object is thrown at an NPC |
| `Order` | parser.h | Sent to `life` when the player issues a command to an NPC |
| `TheSame` | parser.h | Used by the parser's disambiguation to test whether two objects are identical |
| `PluralFound` | parser.h | Used internally by the parser when a plural word matches multiple objects |
| `ListMiscellany` | parser.h | Used by `WriteListFrom` for list-printing messages |
| `Miscellany` | parser.h | Used for general parser and library messages |
| `RunTimeError` | parser.h | Used to print runtime error messages |
| `Prompt` | parser.h | Used to print the command prompt |
| `NotUnderstood` | parser.h | Generated when the parser cannot match input to any grammar |
| `Going` | parser.h | Used in `before`/`after` rules during movement to provide direction information |
| `Places` | parser.h | Defined as fake action only when `NO_PLACES` is defined |
| `Objects` | parser.h | Defined as fake action only when `NO_PLACES` is defined |

> **Note:** `Places` and `Objects` are conditionally fake. When `NO_PLACES` is
> *not* defined (the default), they are real actions with grammar lines and sub
> routines (see §A.3.6). When `NO_PLACES` *is* defined, they become fake
> actions so that `L__M` messages referencing them still compile without errors.

---

## §A.7 Library Stubs and Defaults

At the end of `grammar.h`, the library provides stub implementations for entry
point routines and default values for constants. The `Stub` directive creates an
empty routine (one that simply returns false) if the game has not already
defined one. The `Default` directive creates a constant with the given value if
the game has not already defined it.

**Default constants:**

```inform6
Default Story          0;
Default Headline       0;
Default d_obj          NULL;
Default u_obj          NULL;
```

`Story` and `Headline` default to zero (which suppresses printing). `d_obj` and
`u_obj` are the objects representing the "down" and "up" directions; they
default to `NULL` because not all games define them.

**Stub routines (always available):**

| Routine | Parameters | Purpose |
|---------|-----------|---------|
| `AfterLife` | 0 | Called after the player dies, before the end-game menu |
| `AfterPrompt` | 0 | Called after the command prompt is printed |
| `Amusing` | 0 | Called when the player wins and selects "amusing" from the menu |
| `BeforeParsing` | 0 | Called before the parser processes each command |
| `ChooseObjects` | 2 | Called to disambiguate or rank objects during parsing |
| `DarkToDark` | 0 | Called when the player moves from one dark room to another |
| `DeathMessage` | 0 | Called to print a custom death message |
| `Epilogue` | 0 | Called after the game ends, before the final prompt |
| `GamePostRoutine` | 0 | Called after each action's `after` processing |
| `GamePreRoutine` | 0 | Called before each action's `before` processing |
| `InScope` | 1 | Called to add extra objects to scope for a given actor |
| `LookRoutine` | 0 | Called at the end of every `Look` action |
| `NewRoom` | 0 | Called when the player enters a new room |
| `ObjectDoesNotFit` | 2 | Called when an object is too large for a container |
| `ParseNumber` | 2 | Called to parse non-standard number formats |
| `ParserError` | 1 | Called when the parser encounters an error |
| `PrintTaskName` | 1 | Called to print the name of a scored task |
| `PrintVerb` | 1 | Called to print an unrecognized verb for disambiguation |
| `TimePasses` | 0 | Called at the end of each turn |
| `UnknownVerb` | 1 | Called when the parser encounters an unrecognized verb |
| `AfterSave` | 1 | Called after a save operation (parameter: success flag) |
| `AfterRestore` | 1 | Called after a restore operation (parameter: success flag) |

**Stub routines (Glulx only)** [Glulx only]:

| Routine | Parameters | Purpose |
|---------|-----------|---------|
| `HandleGlkEvent` | 2 | Called to handle Glk events in the main event loop |
| `IdentifyGlkObject` | 4 | Called during startup to identify Glk objects (windows, streams, etc.) |
| `InitGlkWindow` | 1 | Called during startup to initialize Glk windows |

**Non-trivial defaults:**

`PrintRank` and `ParseNoun` use `#Ifndef` instead of `Stub` because they have
non-trivial default implementations:

```inform6
#Ifndef PrintRank;
[ PrintRank; "."; ];
#Endif;

#Ifndef ParseNoun;
[ ParseNoun obj; obj = obj; return -1; ];
#Endif;
```

`PrintRank` prints a period by default (appended after the score rank text).
`ParseNoun` returns −1 by default, indicating that the parser should use its
standard name-matching algorithm rather than a custom one. The `obj = obj`
assignment suppresses the compiler warning about an unused parameter.
