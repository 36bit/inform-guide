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

# Chapter 24: NPC Behavior, Conversations, and Orders

Non-player characters (NPCs) are objects that the player can interact with as
people or creatures: talking to them, giving them orders, and receiving
responses. The standard library provides a framework for NPC behavior derived
from routines in `parser.h`, `verblib.h`, and `grammar.h` (library 6.12.8).
This framework handles the parsing of commands directed at NPCs, routes
conversation actions through the `life` property, and provides hooks for
autonomous NPC behavior through daemons and timers. This chapter covers the
full NPC pipeline: defining characters, processing orders, handling
conversation, and implementing autonomous behavior.

## §24.1 Defining NPCs

An NPC is, at its simplest, an object with the `animate` attribute. This
single attribute tells the parser that the object represents a living being and
enables the full range of personal interaction actions.

### §24.1.1 The `animate` Attribute

The `animate` attribute marks an object as a person or creature. When an object
has `animate`, the parser allows it to be the target of conversation actions
(`##Ask`, `##Tell`, `##Answer`), personal actions (`##Give`, `##Show`,
`##Attack`, `##Kiss`, `##WakeOther`, `##ThrowAt`), and orders
(`"NPC, do something"`). The library also uses `animate` when selecting
pronouns: an `animate` object defaults to "him" or "her" depending on gender
attributes, rather than "it".

```inform6
Object  troll "cave troll" dark_cave
  with  name 'cave' 'troll',
        description "A massive troll blocks the passage.",
  has   animate;
```

### §24.1.2 The `talkable` Attribute

The `talkable` attribute allows an object to be the target of conversation
commands without being treated as a living creature. This is useful for objects
like telephones, intercoms, or magical communication devices. An object with
`talkable` but without `animate` can receive `##Ask`, `##Tell`, `##Answer`,
and `##AskFor` actions, but not personal actions like `##Attack` or `##Kiss`.

The `CreatureTest` routine in `parser.h` implements this distinction:

```inform6
[ CreatureTest obj;
    if (actor ~= player) rtrue;
    if (obj has animate) rtrue;
    if (obj hasnt talkable) rfalse;
    if (action_to_be == ##Ask or ##Answer or ##Tell
                      or ##AskFor or ##AskTo) rtrue;
    rfalse;
];
```

When the parser encounters a `creature` grammar token, it calls
`CreatureTest`. The routine returns true for any `animate` object. For objects
that are only `talkable`, it returns true only when the current action is a
conversation action. This means a `talkable` telephone can be asked about
things but not attacked as a person.

```inform6
Object  red_phone "red telephone" office
  with  name 'red' 'telephone' 'phone',
        description "A red telephone connected to headquarters.",
        life [;
            Ask:
                if (second == 'orders' or 'mission')
                    "A voice on the line says, ~Proceed to the vault.~";
                "There is no reply.";
        ],
  has   talkable;
```

### §24.1.3 Required Properties for NPCs

A well-defined NPC typically provides the following properties:

| Property      | Purpose                                              |
|---------------|------------------------------------------------------|
| `name`        | Dictionary words the parser uses to match the NPC    |
| `description` | Printed when the player examines the NPC             |
| `life`        | Handles personal interaction actions                 |
| `orders`      | Handles commands directed at the NPC                 |
| `each_turn`   | Per-turn behavior when the NPC is in scope           |

None of these properties are strictly required by the compiler — an object with
only `animate` and `name` is a valid NPC — but omitting `life` or `orders`
means the NPC will produce only default library refusal messages.

### §24.1.4 Gender Attributes

The library uses four attributes to determine how pronouns refer to an NPC:

| Attribute    | Pronoun         | Usage                               |
|--------------|-----------------|-------------------------------------|
| `male`       | "him"           | Male characters                     |
| `female`     | "her"           | Female characters                   |
| `neuter`     | "it"            | Creatures, genderless beings        |
| `pluralname` | "them"          | Groups of characters                |

If no gender attribute is set on an `animate` object, the library defaults to
`neuter`. An object may have both `male` and `female` if appropriate (though
this is unusual). The `pluralname` attribute causes the library to use plural
verb forms ("The guards are here" rather than "The guards is here").

```inform6
Object  queen "Queen Helena" throne_room
  with  name 'queen' 'helena',
        description "Queen Helena regards you with cool authority.",
        life [;
            Ask: "~I shall consider your question,~ she says.";
        ],
  has   animate female proper;

Object  guards "palace guards" throne_room
  with  name 'palace' 'guards',
        description "A pair of heavily armed guards.",
        life [;
            Ask: "The guards exchange a glance but say nothing.";
        ],
  has   animate pluralname;
```

### §24.1.5 The `creature` Grammar Token

In grammar definitions, the `creature` token matches only objects that pass
`CreatureTest`. This token is used throughout the standard grammar for
conversation and personal interaction verbs. For example, the grammar for
`'ask'` in `grammar.h` is:

```inform6
Verb 'ask'
    * creature 'about' topic    -> Ask
    * creature 'for' noun       -> AskFor
    * creature 'to' topic       -> AskTo
    * 'that' creature topic     -> AskTo;
```

The `creature` token has the internal value `CREATURE_TOKEN` (6). When the
parser processes this token, it calls `NounDomain` with `CREATURE_TOKEN` as
the context, which in turn calls `CreatureTest` on each candidate object. Only
objects that pass `CreatureTest` can match a `creature` token.

## §24.2 Orders: Directing Commands at NPCs

The player can direct commands at NPCs using the comma-separated syntax:
`"NPC, do something"`. For example, `"troll, go north"` or
`"guard, drop the sword"`. The parser and library handle this through a
multi-stage pipeline.

### §24.2.1 How the Parser Handles NPC Orders

When the parser encounters a comma in the input, it treats everything before
the comma as the name of an NPC and everything after the comma as a command
directed at that NPC. The processing in `parser.h` proceeds as follows:

1. The parser detects the comma and enters the conversation handler.
2. It calls `NounDomain` with `CREATURE_TOKEN` and `TALKING_REASON` as the
   scope reason to identify the NPC before the comma.
3. If the NPC is not found, the parser prints an error
   ("You seem to want to talk to someone, but I can't see whom.") and
   requests new input.
4. If the matched object lacks both `animate` and `talkable`, the parser
   prints a refusal ("You can't talk to that.") and requests new input.
5. If the NPC is valid, the parser calls `PronounNotice` on the NPC (updating
   pronoun references), sets the global variable `actor` to the NPC, advances
   past the comma, and parses the rest of the input as a normal command.

```inform6
! Simplified flow from parser.h:
wn = 1; lookahead = HELD_TOKEN;
scope_reason = TALKING_REASON;
l = NounDomain(player, actors_location, CREATURE_TOKEN);
scope_reason = PARSING_REASON;
if (l == 0) {
    L__M(##Miscellany, 23);
    jump ReType;
}
if (l hasnt animate && l hasnt talkable) {
    L__M(##Miscellany, 24, l);
    jump ReType;
}
PronounNotice(l);
verb_wordnum = j + 1;
```

### §24.2.2 The `actor` Global

The global variable `actor` holds a reference to the object that is the
subject of the current command. During normal player commands, `actor` is set
to `player`. When the player issues an order to an NPC (e.g., `"troll, go
north"`), `actor` is set to that NPC.

Code can test whether an action is a direct player action or an NPC order by
comparing `actor` to `player`:

```inform6
before [;
    Take:
        if (actor ~= player)
            "You're not going to let anyone else take that.";
];
```

The related global `actors_location` holds the location of the current actor,
analogous to `location` for the player.

### §24.2.3 The Orders Processing Pipeline

When the parser has successfully parsed an NPC order, the library processes it
through the `actor_act` routine in `parser.h`. The pipeline has four stages:

**Stage 1: `player.orders`**

The library first calls `RunRoutines(player, orders)`. This gives the player
object's `orders` property a chance to intercept any order before it reaches
the NPC. If `player.orders` returns true, the order is consumed and processing
stops.

**Stage 2: `actor.orders` (the NPC's `orders` property)**

If `player.orders` returned false, the library calls
`RunRoutines(actor, orders)` — that is, the `orders` property of the NPC being
addressed. If this returns true, the NPC has handled the order and processing
stops.

**Stage 3: `##NotUnderstood` conversion**

If both `orders` properties returned false and the parser could not parse the
text after the comma into a valid action (the action is `##NotUnderstood`),
the library converts the action to `##Answer`. The NPC is placed in
`inputobjs-->3`, the actor is reset to `player`, and the action becomes
`##Answer`. This means that gibberish directed at an NPC (e.g.,
`"troll, xyzzy"`) is treated as speech: "say xyzzy to troll".

**Stage 4: `life` property with `##Order`**

If the action was a valid parsed command and both `orders` properties returned
false, the library calls `RunLife(actor, ##Order)`. This invokes the NPC's
`life` property with `reason_code` set to `##Order`. The original `action`,
`noun`, and `second` values are available inside the `life` routine. If `life`
returns false, the library prints a default refusal message ("The troll has
better things to do.").

The complete pipeline from `parser.h`:

```inform6
! Inside actor_act():
sa = action; sn = noun; ss = second;
action = a; noun = n; second = s;
j = RunRoutines(player, orders);
if (j == 0) {
    j = RunRoutines(actor, orders);
    if (j == 0) {
        if (action == ##NotUnderstood) {
            inputobjs-->3 = actor;
            actor = player;
            action = ##Answer;
            return ACTOR_ACT_ABORT_NOTUNDERSTOOD;
        }
        if (RunLife(actor, ##Order) == 0) {
            L__M(##Order, 1, actor);
            return ACTOR_ACT_ABORT_ORDER;
        }
    }
}
action = sa; noun = sn; second = ss;
```

### §24.2.4 Action Variables During NPC Orders

When an NPC order is being processed, the standard action variables hold the
parsed command directed at the NPC:

| Variable  | Value                                              |
|-----------|----------------------------------------------------|
| `actor`   | The NPC being commanded                            |
| `action`  | The action the player wants the NPC to perform     |
| `noun`    | The first noun of the command (if any)             |
| `second`  | The second noun of the command (if any)            |

Inside an `orders` property, these variables can be tested to decide whether
the NPC should obey:

```inform6
Object  servant "obedient servant" kitchen
  with  name 'servant' 'obedient',
        orders [;
            Go:
                print_ret (The) self, " bows and leaves.";
            Take:
                if (noun has scenery)
                    "~I cannot carry that, my lord.~";
                print (The) self, " picks up ", (the) noun, ".";
                move noun to self;
                rtrue;
            Drop:
                if (noun notin self)
                    "~I don't have that, my lord.~";
                print (The) self, " puts down ", (the) noun, ".";
                move noun to parent(self);
                rtrue;
        ],
        life [;
            Order:
                "~I don't understand that command, my lord.~";
        ],
  has   animate male;
```

### §24.2.5 `##NotUnderstood` and `##Answer` Conversion

When the parser fails to parse the text after the comma into any valid
grammar, it sets the action to `##NotUnderstood`. The global variables are
configured as follows:

| Variable        | Value                                           |
|-----------------|-------------------------------------------------|
| `action`        | `##NotUnderstood`                               |
| `noun`          | 1 (a flag value)                                |
| `second`        | The NPC addressed                               |
| `consult_from`  | Word number where the unparsed text begins      |
| `consult_words` | Number of words in the unparsed text            |

If the `orders` property does not handle this, `actor_act` converts the action
to `##Answer` and sets the actor back to `player`. The unparsed text is
accessible through `consult_from` and `consult_words`, allowing the `life`
property (or other handlers) to examine what the player tried to say.

An `orders` property can intercept `##NotUnderstood` before the conversion:

```inform6
Object  parrot "green parrot" deck
  with  name 'green' 'parrot' 'polly',
        orders [;
            NotUnderstood:
                print "The parrot squawks, ~";
                PrintCommand(verb_wordnum);
                "!~";
        ],
  has   animate;
```

## §24.3 The `life` Property

The `life` property is a routine (or chain of routines, since `life` is
additive) that handles personal interaction actions directed at an NPC. When
the library determines that an action is a personal interaction, it calls
`RunLife(object, action)`, which sets the global `reason_code` to the action
and then calls `RunRoutines(object, life)`.

### §24.3.1 Actions Routed to `life`

The following actions are routed to the `life` property rather than to
`before`. The table shows how `noun` and `second` are set for each:

| Action        | Grammar example              | `noun`          | `second`        |
|---------------|------------------------------|-----------------|-----------------|
| `##Give`      | "give sword to troll"        | the sword       | the troll       |
| `##Show`      | "show map to wizard"         | the map         | the wizard      |
| `##Ask`       | "ask wizard about dragon"    | the wizard      | `'dragon'`      |
| `##Tell`      | "tell wizard about cave"     | the wizard      | `'cave'`        |
| `##Answer`    | "say password to guard"      | `'password'`    | the guard       |
| `##Order`     | "troll, go north" (fallback) | *(from order)*  | *(from order)*  |
| `##Attack`    | "attack troll"               | the troll       | —               |
| `##Kiss`      | "kiss queen"                 | the queen       | —               |
| `##ThrowAt`   | "throw rock at troll"        | the rock        | the troll       |
| `##WakeOther` | "wake wizard"                | the wizard      | —               |

For `##Give` and `##Show`, the `life` property is called on `second` (the
NPC receiving or being shown the object). For `##Ask`, `##Tell`, `##Attack`,
`##Kiss`, and `##WakeOther`, `life` is called on `noun` (the NPC). For
`##Answer` and `##ThrowAt`, `life` is called on `second`.

### §24.3.2 Return Value Conventions

Inside a `life` property routine, the action being handled is available in
`reason_code` (or equivalently, the `action` global if not during an order).
The routine should:

- Return **true** (or print a string, which implies true) to indicate the
  action has been handled. The library takes no further default action.
- Return **false** (0) to decline handling. The library prints its default
  message for that action (e.g., "The troll isn't interested." for `##Give`).

### §24.3.3 Comprehensive `life` Property Example

```inform6
Object  wizard "old wizard" tower
  with  name 'old' 'wizard' 'mage',
        description "The wizard peers at you over half-moon spectacles.",
        life [;
            Ask:
                switch (second) {
                    'dragon', 'beast':
                        "~The dragon nests in the eastern caverns,~
                         the wizard says. ~Take the silver sword.~";
                    'sword', 'silver':
                        "~The silver sword was forged by the
                         dwarves. It is the only weapon that can
                         pierce the dragon's hide.~";
                    'spell', 'magic':
                        "~I have one spell remaining,~ he says
                         sadly.";
                    default:
                        "The wizard strokes his beard thoughtfully
                         but says nothing useful.";
                }
            Tell:
                switch (second) {
                    'dragon', 'beast':
                        "~Yes, yes, I know about the dragon,~
                         he says impatiently.";
                    default:
                        "The wizard listens politely but seems
                         uninterested.";
                }
            Give:
                if (noun == spellbook) {
                    move spellbook to self;
                    "~My spellbook! At last!~ The wizard clutches
                     the book to his chest.";
                }
                if (noun == silver_sword)
                    "~No, you'll need that more than I will.~";
                "~I have no use for that,~ he says.";
            Show:
                if (noun == ancient_map)
                    "The wizard examines the map with great
                     interest. ~This changes everything!~";
                "The wizard glances at ", (the) noun,
                " without interest.";
            Answer:
                "~I'm afraid I don't follow,~ says the wizard.";
            Attack:
                remove self;
                "The wizard vanishes in a flash of light!";
            Kiss:
                "~I'm flattered, but no,~ says the wizard.";
            ThrowAt:
                print_ret (The) noun,
                    " bounces off the wizard's shield spell.";
            WakeOther:
                "~I'm quite awake, thank you!~";
        ],
  has   animate male;
```

## §24.4 Conversation: Ask/Tell Model

The standard library implements an **Ask/Tell** conversation model. The player
types `"ask NPC about topic"` or `"tell NPC about topic"`, and the library
routes the action through the NPC's `life` property.

### §24.4.1 How Topics Are Resolved

For `##Ask` and `##Tell`, the grammar uses a `topic` token for the subject
of conversation. The parser processes the topic as follows:

1. The parser looks up each word of the topic in the dictionary.
2. `second` is set to the dictionary address of the first recognized word in
   the topic text. If no word is recognized, `second` is set to 0.
3. The globals `consult_from` and `consult_words` are set to the starting
   word number and word count of the topic text, allowing examination of
   multi-word topics.

For example, if the player types `"ask wizard about the ancient dragon"`:

| Variable        | Value                                     |
|-----------------|-------------------------------------------|
| `action`        | `##Ask`                                   |
| `noun`          | the wizard object                         |
| `second`        | dictionary address of `'ancient'` or `'dragon'` (first recognized word) |
| `consult_from`  | word number of "the" (or "ancient")       |
| `consult_words` | number of words in "the ancient dragon"   |

### §24.4.2 Testing for Specific Topics

The simplest approach is to test `second` against dictionary words:

```inform6
life [;
    Ask:
        if (second == 'dragon')
            "~The dragon is very dangerous,~ she says.";
        if (second == 'sword' or 'weapon')
            "~You'll need the silver sword.~";
        "She shrugs.";
];
```

For multi-word topics, use `consult_from` and `consult_words` to scan the
entire topic text with `NextWord` or `WordAddress`/`WordLength`:

```inform6
[ TopicContains w   i j;
    j = wn;
    wn = consult_from;
    for (i = 0 : i < consult_words : i++) {
        if (NextWord() == w) { wn = j; rtrue; }
    }
    wn = j;
    rfalse;
];
```

This utility routine checks whether a given dictionary word appears anywhere
in the topic text. It can be used in a `life` property:

```inform6
life [;
    Ask:
        if (TopicContains('dragon') && TopicContains('cave'))
            "~The dragon's cave is to the east.~";
        if (TopicContains('dragon'))
            "~The dragon is fearsome.~";
        "She has nothing to say about that.";
];
```

### §24.4.3 Full Ask/Tell Conversation Example

```inform6
Object  historian "court historian" library
  with  name 'court' 'historian',
        description
            "The historian sits surrounded by ancient texts.",
        life [;
            Ask:
                switch (second) {
                    'king', 'majesty':
                        "~King Dorian rules wisely, though his
                         temper is legendary.~";
                    'queen':
                        "~Queen Elara vanished ten years ago.
                         Some say she fled to the northern
                         mountains.~";
                    'war', 'battle':
                        "~The Last War ended a century ago with
                         the Pact of Stones. We have been at peace
                         since.~";
                    'pact', 'stones':
                        "~The Pact of Stones bound the five
                         kingdoms to eternal peace. It is kept in
                         the cathedral vault.~";
                    'vault', 'cathedral':
                        "~The cathedral vault lies beneath the
                         high altar. Only the archbishop has the
                         key.~";
                    default:
                        "The historian flips through several books
                         but finds nothing relevant.";
                }
            Tell:
                switch (second) {
                    'queen', 'elara':
                        "~You've seen the queen?~ The historian
                         nearly drops his quill in excitement.";
                    'pact':
                        "~I know all about the Pact,~ he says
                         dismissively.";
                    default:
                        "The historian nods politely but continues
                         reading.";
                }
            Give:
                if (noun ofclass Book) {
                    move noun to self;
                    "~A new volume for the collection!~ The
                     historian carefully shelves it.";
                }
                "~I only collect books,~ he says.";
        ],
  has   animate male;

Class   Book
  with  short_name "book",
  has   ;
```

## §24.5 Conversation: Alternative Models

The standard Ask/Tell model is adequate for many games, but the library
provides hooks for alternative conversation systems.

### §24.5.1 Using `before` on the NPC

Instead of (or in addition to) handling `##Ask` and `##Tell` in `life`, an
NPC can intercept these actions in its `before` property. This is possible
because the library calls `before` on the NPC object before routing to `life`
for some actions. However, the standard library routing for conversation
actions calls `life` directly via `RunLife`, bypassing `before`. To use
`before` for conversation handling, add grammar that routes actions to the NPC
as `noun`:

```inform6
Object  oracle "oracle" shrine
  with  name 'oracle' 'seer',
        before [;
            Consult:
                if (second == 'fate' or 'future')
                    "The oracle's eyes glow. ~Your fate is
                     sealed.~";
                "The oracle is silent.";
        ],
  has   animate female;
```

### §24.5.2 Using `orders` for Custom Conversation

The `orders` property receives all commands directed at an NPC, including
those the parser cannot understand. This makes it a natural place to implement
free-form conversation where any text after the NPC's name is treated as
dialogue:

```inform6
Object  bartender "bartender" tavern
  with  name 'bartender' 'barkeep',
        orders [;
            NotUnderstood:
                if (TopicContains('beer') || TopicContains('ale'))
                    "~Coming right up!~ The bartender pours
                     you a pint.";
                if (TopicContains('rumor') ||
                    TopicContains('gossip') ||
                    TopicContains('news'))
                    "~I hear the old mine has reopened,~ the
                     bartender says, lowering his voice.";
                "The bartender nods noncommittally.";
        ],
  has   animate male;
```

With this approach, the player can type `"bartender, tell me about the beer"`
or `"bartender, any gossip?"` — the parser will fail to match a standard
grammar line, set the action to `##NotUnderstood`, and the `orders` property
handles it by scanning the unparsed text.

### §24.5.3 The `grammar` Property

The `grammar` property provides the most powerful mechanism for NPC-specific
command parsing. When a command is directed at an NPC, the library calls the
NPC's `grammar` property before attempting to parse the command with the
standard grammar. The `grammar` routine can:

- Return **0**: continue with normal parsing.
- Return **1**: the grammar property has fully handled the command. It must
  set `action`, `noun`, and `second` before returning.
- Return a **negative dictionary word**: the library uses this word as the
  verb and continues parsing the rest of the input against standard grammar
  lines for that verb.
- **[Z-machine]** Return a **positive dictionary word** (or a non-dictionary
  value that is also not 0 or 1): treated as a dictionary word to use as the
  verb.
- **[Glulx]** Return a **negative value**: its absolute value is treated as a
  dictionary word to use as the verb.

```inform6
Object  robot "repair robot" lab
  with  name 'repair' 'robot',
        grammar [;
            if (NextWord() == 'fix') {
                action = ##Repair;
                noun = NextWord();
                if (noun) noun = NounDomain(player, location, 0);
                second = nothing;
                return 1;
            }
            return 0;
        ],
        orders [;
            Repair:
                if (noun == nothing)
                    "~Fix what?~ asks the robot.";
                print "The robot extends a manipulator arm and
                    repairs ", (the) noun, ".^";
                rtrue;
        ],
  has   animate;
```

### §24.5.4 Extending Grammar for Conversation

New verbs can be added specifically for conversation:

```inform6
Verb 'greet' 'salute'
    * creature                -> Greet;

[ GreetSub;
    if (RunLife(noun, ##Greet)) return;
    "There is no reaction.";
];
```

The NPC's `life` property can then handle `##Greet`:

```inform6
life [;
    Greet:
        "~Well met, traveller!~ says the merchant.";
];
```

### §24.5.5 The `##Answer` Action

The `##Answer` action handles `"say X to NPC"` and `"answer X to NPC"`
patterns. In the standard grammar:

```inform6
Verb 'answer' 'say' 'shout' 'speak'
    * topic 'to' creature               -> Answer;
```

For `##Answer`, `noun` holds the dictionary word from the topic and `second`
holds the creature addressed. The `life` property on the NPC (as `second`)
receives the action:

```inform6
Object  sphinx "sphinx" desert
  with  name 'sphinx',
        life [;
            Answer:
                if (noun == 'riddle')
                    "The sphinx smiles. ~Correct.~";
                "The sphinx shakes its head slowly.";
        ],
  has   animate neuter;
```

## §24.6 NPC Actions and Autonomous Behavior

NPCs are more compelling when they act on their own, moving around the world,
performing actions, and reacting to events. The library provides several
mechanisms for autonomous NPC behavior.

### §24.6.1 Making NPCs Perform Actions

To make an NPC perform an action, use `<action noun second>` with the `actor`
variable temporarily set, or use the action-processing routine with the NPC
as the actor. The simplest approach is to directly manipulate the game state:

```inform6
! Move an object into the NPC's possession
move sword to guard;
print "The guard picks up the sword.^";

! Move an NPC to a new location
move guard to courtyard;
```

For actions that should pass through the full action-processing pipeline
(triggering `before`, `after`, and other hooks), use the library's
`<action noun second, actor>` syntax, which is the short form for setting
`actor` and calling the action:

```inform6
! Make the NPC take the sword through normal action processing
<Take sword, guard>;
```

### §24.6.2 Moving NPCs

Moving an NPC is typically done with `move`:

```inform6
move guard to courtyard;
```

After moving an NPC, call `MoveFloatingObjects()` if the player might be in
a location with floating objects that could be affected. The `found_in`
property of floating objects is re-evaluated by `MoveFloatingObjects`.

For NPCs that move between rooms, a common pattern is to print arrival and
departure messages only when the player can see them:

```inform6
[ MoveNPC npc destination   old_loc;
    old_loc = parent(npc);
    move npc to destination;
    if (old_loc == location)
        print_ret (The) npc, " leaves the room.";
    if (destination == location)
        print_ret (The) npc, " enters the room.";
];
```

### §24.6.3 NPC Daemons

A daemon is a routine that runs every turn, regardless of the player's action.
Daemons are started with `StartDaemon(object)` and stopped with
`StopDaemon(object)`. The daemon routine is the object's `daemon` property:

```inform6
Object  cat "black cat" garden
  with  name 'black' 'cat',
        description "A sleek black cat.",
        daemon [ r;
            if (random(3) ~= 1) return;
            r = random(3);
            if (parent(self) == location) {
                switch (r) {
                    1: "^The cat licks its paw.";
                    2: "^The cat stretches lazily.";
                    3: "^The cat purrs softly.";
                }
            }
        ],
  has   animate;
```

Start the daemon in `Initialise` or when the NPC is first encountered:

```inform6
[ Initialise;
    location = garden;
    StartDaemon(cat);
];
```

A patrolling NPC can use a daemon to move along a predefined route:

```inform6
Object  guard "patrolling guard" guardhouse
  with  name 'guard' 'patrolling',
        description "A guard in chain mail.",
        patrol_route 0,
        daemon [;
            self.patrol_route = self.patrol_route + 1;
            if (self.patrol_route > 3)
                self.patrol_route = 1;
            switch (self.patrol_route) {
                1: MoveNPC(self, guardhouse);
                2: MoveNPC(self, corridor);
                3: MoveNPC(self, throne_room);
            }
        ],
  has   animate male;

[ MoveNPC npc destination;
    if (parent(npc) == location)
        print "^", (The) npc, " marches away.^";
    move npc to destination;
    if (destination == location)
        print "^", (The) npc, " marches in.^";
];
```

### §24.6.4 NPC Timers

A timer is similar to a daemon but counts down from a starting value to zero,
then fires. Timers use the object's `time_left` property. Start a timer with
`StartTimer(object, count)` and stop it with `StopTimer(object)`:

```inform6
Object  bomb "ticking bomb" vault
  with  name 'ticking' 'bomb',
        description [;
            print "A bomb with a digital readout showing ",
                self.time_left, ".^";
        ],
        time_out [;
            deadflag = 1;
            "^The bomb explodes with a deafening roar!";
        ],
        time_left,
  has   ;
```

Start the timer when the bomb is armed:

```inform6
StartTimer(bomb, 10);  ! Explodes in 10 turns
```

Timers can be used for NPCs that perform delayed actions:

```inform6
Object  messenger "royal messenger" courtyard
  with  name 'royal' 'messenger',
        time_out [;
            if (parent(self) == location)
                "^The messenger says, ~Your audience with the
                 king is now.~";
            else {
                move self to location;
                "^A royal messenger arrives. ~Your audience
                 with the king is now,~ he announces.";
            }
        ],
        time_left,
  has   animate male;
```

### §24.6.5 The `each_turn` Property

The `each_turn` property is called every turn for objects that are in scope
(typically in the player's location). Unlike daemons, `each_turn` only fires
when the NPC is present:

```inform6
Object  beggar "old beggar" market
  with  name 'old' 'beggar',
        description "A ragged old beggar.",
        each_turn [;
            if (random(4) == 1)
                "^~Spare a coin?~ wheezes the beggar.";
        ],
  has   animate male;
```

The difference between `daemon` and `each_turn`:

| Feature       | `daemon`                     | `each_turn`                   |
|---------------|------------------------------|-------------------------------|
| Activation    | `StartDaemon(obj)`           | Automatic when in scope       |
| Scope         | Runs regardless of location  | Only when object is in scope  |
| Deactivation  | `StopDaemon(obj)`            | Automatic when out of scope   |
| Property used | `daemon`                     | `each_turn`                   |

Both can be used together on the same object: the daemon handles global
behavior (movement, state changes) while `each_turn` handles local
atmospheric messages.

## §24.7 Player Object and Player-as-NPC

The player character is itself an object in the world model, and the library
treats it in many ways like any other NPC.

### §24.7.1 The `player` Variable and `selfobj`

The global variable `player` holds a reference to the object representing the
player character. By default, this is `selfobj`, a library-defined object in
`parser.h`:

```inform6
Object  selfobj "(self object)"
  with  short_name [;
            return L__M(##Miscellany, 18);
        ],
        description [;
            return L__M(##Miscellany, 19);
        ],
        before NULL,
        after NULL,
        life NULL,
        each_turn NULL,
        time_out NULL,
        describe NULL,
        capacity MAX_CARRIED,
        parse_name 0,
        orders 0,
        number 0,
  has   concealed animate proper transparent;
```

The `selfobj` object has all the standard NPC properties (`life`, `orders`,
`each_turn`) set to `NULL`, meaning they do nothing by default. A game can
add behavior to the player object by modifying these properties.

### §24.7.2 Changing the Player Object

The `ChangePlayer` routine switches the player character to a different
object. This is used for games where the player controls multiple characters
or transforms into a different form:

```inform6
ChangePlayer(new_object, flag);
```

- `new_object`: the object to become the new player character.
- `flag`: if provided and true, the library prints a message announcing the
  change and calls `<Look>` to describe the new location.

The new player object should have `animate` and typically needs appropriate
properties:

```inform6
Object  dwarf_body "Dorin the dwarf" mine
  with  name 'dorin' 'dwarf',
        description "You are Dorin, a sturdy dwarf.",
        capacity 10,
  has   animate male proper;

! Switch to the dwarf:
ChangePlayer(dwarf_body, 1);
```

After `ChangePlayer`:

- `player` is set to `new_object`.
- `location` is updated to the new player's parent.
- Floating objects are moved as needed.
- Pronouns are updated.
- The `short_name` of the old player object becomes visible to the new player
  (it is no longer `concealed`).

### §24.7.3 `player.orders`

As described in §24.2.3, `player.orders` is called before any NPC's `orders`
property when the player directs a command at an NPC. This allows the player
object to intercept orders globally:

```inform6
Object  selfobj "(self object)"
  with  ...
        orders [;
            if (actor == guard && action == ##Attack)
                "You would never order the guard to attack!";
        ],
  has   concealed animate proper transparent;
```

This is useful for scenarios where the player character has personality traits
that prevent certain commands from being issued.

## §24.8 Advanced NPC Techniques

### §24.8.1 NPCs with Inventories

NPCs can hold objects just like the player. Objects given to an NPC via the
`##Give` action (handled in `life`) should be moved into the NPC with `move`:

```inform6
Object  porter "porter" hotel_lobby
  with  name 'porter',
        life [;
            Give:
                if (children(self) >= 3)
                    "~I'm carrying too much already, sir.~";
                move noun to self;
                print_ret "The porter takes ", (the) noun,
                    ". ~I'll put this in your room, sir.~";
        ],
  has   animate male;
```

To let the player see what an NPC is carrying, provide a custom `describe` or
override the `##Examine` action in `before`:

```inform6
Object  merchant "travelling merchant" market
  with  name 'merchant' 'travelling',
        description [;
            print "A merchant with a heavy pack.";
            if (child(self) ~= nothing) {
                print " He is carrying ";
                WriteListFrom(child(self),
                    ENGLISH_BIT + RECURSE_BIT + PARTINV_BIT);
                print ".";
            }
            new_line; rtrue;
        ],
  has   animate male;
```

### §24.8.2 NPCs That Follow the Player

A common pattern is an NPC companion who follows the player from room to
room. This is typically implemented with a daemon:

```inform6
Object  dog "loyal dog" garden
  with  name 'loyal' 'dog' 'hound',
        description "A loyal brown dog.",
        daemon [;
            if (parent(self) == location) return;
            move self to location;
            "^Your dog bounds in, wagging its tail.";
        ],
        before [;
            Take:
                "The dog is too big to carry.";
        ],
  has   animate;
```

For NPCs that follow with a one-turn delay (arriving in the room the player
just left), track the player's previous location:

```inform6
Object  stalker "shadowy figure" alley
  with  name 'shadowy' 'figure' 'shadow',
        description "A figure in a dark cloak.",
        last_player_loc 0,
        daemon [;
            if (self.last_player_loc ~= 0 &&
                parent(self) ~= self.last_player_loc) {
                if (parent(self) == location)
                    "^The shadowy figure slips away.";
                move self to self.last_player_loc;
                if (self.last_player_loc == location)
                    "^A shadowy figure emerges from the
                     darkness behind you.";
            }
            self.last_player_loc = location;
        ],
  has   animate;
```

### §24.8.3 NPCs with Their Own Scope

By default, NPCs do not have their own scope — the library does not track what
an NPC can see. For simple games, this is sufficient: orders are processed
using the player's scope. For more sophisticated NPC behavior, scope can be
customized using `InScope` or by manually checking object locations.

To give an NPC awareness of its surroundings, check the NPC's location in
`orders` or `life`:

```inform6
Object  guard "castle guard" gatehouse
  with  name 'castle' 'guard',
        orders [;
            Take:
                if (noun notin parent(self))
                    "~I can't see that from here,~ says
                     the guard.";
                move noun to self;
                print_ret "The guard picks up ", (the) noun, ".";
        ],
  has   animate male;
```

### §24.8.4 `react_before` and `react_after` for NPC-Aware Rooms

The `react_before` and `react_after` properties fire for any action that
occurs while the NPC is in scope. This allows an NPC to react to the player's
actions without being directly involved:

```inform6
Object  bodyguard "bodyguard" throne_room
  with  name 'bodyguard',
        react_before [;
            Take:
                if (noun == crown)
                    "The bodyguard steps in front of you.
                     ~Don't touch that.~";
            Attack:
                if (noun == queen)
                    "The bodyguard tackles you to the ground!";
        ],
        react_after [;
            Drop:
                if (noun == gold_coin)
                    "The bodyguard eyes the dropped coin
                     suspiciously.";
        ],
  has   animate male;
```

Note that `react_before` and `react_after` apply to all objects in scope, not
just NPCs. They are additive properties, so class-based NPCs can inherit
reactive behavior.

### §24.8.5 Multiple NPCs Interacting

When multiple NPCs are present, their `react_before` and `react_after`
properties can create the appearance of inter-NPC awareness. Daemons can also
coordinate NPC behavior:

```inform6
Object  thief "thief" market
  with  name 'thief',
        daemon [;
            if (parent(self) ~= parent(guard_npc)) return;
            if (guard_npc has general) return;
            give guard_npc general;
            if (parent(self) == location) {
                remove self;
                "^The guard spots the thief! After a brief
                 chase, the guard drags the thief away.";
            }
            else
                remove self;
        ],
  has   animate male concealed;

Object  guard_npc "town guard" town_square
  with  name 'town' 'guard',
        description [;
            if (self has general)
                "The guard looks pleased with himself.";
            "The guard patrols alertly.";
        ],
        daemon [;
            switch (random(3)) {
                1: MoveNPC(self, market);
                2: MoveNPC(self, town_square);
                3: MoveNPC(self, tavern);
            }
        ],
  has   animate male;
```

To have NPCs converse with each other, use daemons that print dialogue when
both NPCs are in the player's location:

```inform6
Object  scholar "scholar" library
  with  name 'scholar',
        daemon [;
            if (parent(self) ~= location) return;
            if (parent(librarian) ~= location) return;
            switch (random(3)) {
                1: "^~Have you read the new treatise?~
                    the scholar asks the librarian.";
                2: "^The librarian and the scholar argue
                    about ancient history.";
                3: return;
            }
        ],
  has   animate male;
```

### §24.8.6 NPC State Machines

Complex NPC behavior can be modeled as a state machine using a property to
track the current state:

```inform6
Constant NPC_IDLE     0;
Constant NPC_ALERT    1;
Constant NPC_HOSTILE  2;
Constant NPC_FRIENDLY 3;

Object  sentry "sentry" bridge
  with  name 'sentry',
        npc_state NPC_IDLE,
        description [;
            switch (self.npc_state) {
                NPC_IDLE:
                    "The sentry leans against the wall, bored.";
                NPC_ALERT:
                    "The sentry watches you warily.";
                NPC_HOSTILE:
                    "The sentry brandishes a spear at you!";
                NPC_FRIENDLY:
                    "The sentry nods at you in a friendly way.";
            }
        ],
        life [;
            Ask:
                if (self.npc_state == NPC_HOSTILE)
                    "The sentry snarls wordlessly.";
                if (second == 'password') {
                    self.npc_state = NPC_FRIENDLY;
                    "~Ah, you know the password. Pass freely.~";
                }
                "~I have nothing to say to strangers.~";
            Give:
                if (noun == gold_coin) {
                    remove gold_coin;
                    self.npc_state = NPC_FRIENDLY;
                    "The sentry pockets the coin and winks.
                     ~You may pass.~";
                }
                "~I don't want that.~";
            Attack:
                self.npc_state = NPC_HOSTILE;
                "The sentry blocks your attack and readies
                 a counter-strike!";
        ],
        each_turn [;
            if (self.npc_state == NPC_IDLE)
                self.npc_state = NPC_ALERT;
            if (self.npc_state == NPC_HOSTILE) {
                deadflag = 1;
                "^The sentry lunges forward with the spear!";
            }
        ],
        orders [;
            if (self.npc_state ~= NPC_FRIENDLY)
                "~I don't take orders from you!~";
            Go:
                "~I can't leave my post.~";
            Drop:
                "~I'll hold onto my things, thanks.~";
        ],
  has   animate male;
```

This pattern separates the NPC's behavior into clear states with defined
transitions, making complex NPC logic easier to maintain and extend. The
`each_turn` property drives automatic state transitions, `life` handles
interactions that may change the state, and `orders` consults the current
state to determine whether to obey commands.
