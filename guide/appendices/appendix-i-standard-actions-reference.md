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

# Appendix I: Standard Actions Reference

This appendix is a comprehensive reference for every action defined by the
Inform 6 standard library (version 6.12.8). It covers the full action
processing pipeline, all action-related variables, and detailed entries for
each individual action.

For broader discussion of how actions work, see Chapter 20 (Actions). For the
grammar lines that trigger actions, see Appendix A (Grammar). For the library
messages that actions produce, see Appendix C (Library Messages). For the
attributes and properties that actions check, see Appendix B (Attributes and
Properties).

---

## §I.1 Introduction and Conventions

The standard library defines two categories of actions:

- **Player actions** — triggered by parsed player input and defined in
  `grammar.h` with corresponding handler routines in `verblib.h` and
  `parser.h`. Standard game actions are listed alphabetically in §I.3;
  meta actions (game-control and debug commands) are in §I.4.
- **Fake actions** — generated internally by the library to communicate
  between objects. They have no grammar and no `*Sub` routine. These are
  listed in §I.5.

Every action, whether real or fake, is assigned a unique numeric value by the
compiler. Real actions may be either **meta** (game-control commands that do
not consume a game turn) or **non-meta** (in-world actions that advance the
turn counter).

### §I.1.1 How to Read Action Entries

Each action entry in §I.3 and §I.4 uses the following format:

> **ActionName**
>
> - **Routine:** `ActionNameSub` — the default handler routine called when
>   the action fires.
> - **Meta:** Yes/No — whether the action is a meta-command.
> - **Grammar:** The `Verb` lines from `grammar.h` that can trigger this
>   action, or "None (Fake_Action)" for internally generated actions.
> - **Default Behavior:** What the handler routine does when no `before`
>   rule intercepts the action.
> - **Checks:** Conditions the handler tests before performing the action
>   (e.g., checking attributes like `open`, `container`, `static`).
> - **Effects:** State changes the handler makes (e.g., setting attributes,
>   moving objects).
> - **Messages:** Library messages printed, referenced as
>   `L__M(##ActionName, N)`.
> - **Related Actions:** Other actions commonly associated with this one,
>   including any fake actions it generates.
> - **Notes:** Additional details, edge cases, or VM-specific behavior.

### §I.1.2 Notation Key

The following notation is used throughout this appendix:

| Notation | Meaning |
|----------|---------|
| `##Action` | Action value constant, usable in `before`/`after` rules and `switch` statements |
| `ActionSub` | Default handler routine for the action, called by `ActionPrimitive()` |
| `L__M(##Action, N)` | Library message number N for the action; see Appendix C |
| `meta` | Declared with `Verb meta`; does not consume a game turn |
| `Fake_Action` | No grammar line; generated internally by the library (e.g., `##Receive`) |
| `[Z-machine]` | Behavior specific to the Z-machine virtual machine |
| `[Glulx]` | Behavior specific to the Glulx virtual machine |

For example, to intercept the `Take` action in a `before` property:

```inform6
before [;
    Take: "You don't want to pick that up.";
],
```

And to test whether a fake action triggered a `life` routine:

```inform6
life [;
    Receive: "You accept the gift graciously.";
],
```

---

## §I.2 Action Processing Pipeline Summary

This section documents the full action processing pipeline as implemented in
`parser.h` of library 6.12.8. Understanding this pipeline is essential for
writing correct `before`, `after`, `react_before`, `react_after`, and `life`
rules.

### §I.2.1 Execution Sequence

When an action fires, the library executes the following sequence (from
`begin_action` in `parser.h`, line 5315):

1. **Save state** — The current values of `action`, `noun`, and `second` are
   saved onto temporary variables (line 5316).
2. **Set new values** — The new action, noun, and second are installed into
   the globals (line 5317).
3. **Debug trace** — If `debug_flag` is set, the action and its arguments are
   printed to aid debugging (lines 5318–5322).
4. **Before routines** — For non-meta actions, `BeforeRoutines()` is called
   (line 5325/5336). If any before routine returns true, the action is
   intercepted and the handler is never called.
5. **Action handler** — If `BeforeRoutines()` returns false (action not
   intercepted), `ActionPrimitive()` is called to execute the handler
   (line 5332/5343).
6. **Restore state** — The saved action, noun, and second are restored
   (line 5347).

**BeforeRoutines** (`parser.h`, line 5608) runs the following checks in
order. If any routine returns true, processing stops and the action is
considered handled:

1. `GamePreRoutine()` — game-wide pre-action hook (line 5609).
2. `LibraryExtensions.RunWhile(ext_gamepreroutine)` — extension hook
   (line 5610).
3. `RunRoutines(player, orders)` — the player object's `orders` property
   (line 5612).
4. `SearchScope` with `REACT_BEFORE_REASON` — runs the `react_before`
   property on every object currently in scope (lines 5613–5616).
5. `RunRoutines(location, before)` — the current location's `before`
   property (line 5617).
6. `RunRoutines(inp1, before)` — the first noun's `before` property
   (line 5618).

**ActionPrimitive** (`parser.h`, line 5493) looks up and calls the action's
handler routine from the actions table:

- [Z-machine]: `(#actions_table-->action)()` — direct lookup by action number.
- [Glulx]: `(#actions_table-->(action+1))()` — Glulx uses an offset of 1
  because entry 0 stores the table length.

**AfterRoutines** (`parser.h`, line 5622) are called by the action handler
itself (typically at the end of a successful action). The sequence is:

1. `SearchScope` with `REACT_AFTER_REASON` — runs the `react_after` property
   on every object currently in scope (lines 5623–5625).
2. `RunRoutines(location, after)` — the current location's `after` property
   (line 5626).
3. `RunRoutines(inp1, after)` — the first noun's `after` property
   (line 5627).
4. `GamePostRoutine()` — game-wide post-action hook (line 5628).
5. `LibraryExtensions.RunWhile(ext_gamepostroutine)` — extension hook
   (line 5629).

**R_Process** (`parser.h`, line 5577) is the entry point used by the `<>`
action-invocation syntax:

- `<action noun second>` — calls `begin_action` directly.
- `<action noun second, actor>` — saves `inp1`/`inp2`/`actor`, sets the new
  actor, calls `begin_action`, then restores the saved state.

### §I.2.2 Action Variables

The following global variables are relevant to action processing (declared in
`parser.h`):

| Variable | Type | Declared | Description |
|----------|------|----------|-------------|
| `action` | Global | parser.h:398 | Current action being performed |
| `inp1` | Global | parser.h:399 | First input: 0 (nothing), 1 (number), or object |
| `inp2` | Global | parser.h:400 | Second input: 0 (nothing), 1 (number), or object |
| `noun` | Global | parser.h:401 | First noun or numerical value |
| `second` | Global | parser.h:402 | Second noun or numerical value |
| `actor` | Global | parser.h:436 | Person asked to perform action |
| `meta` | Global | parser.h:438 | True if action is a meta-command |
| `keep_silent` | Global | parser.h:404 | If true, suppress default messages |
| `reason_code` | Global | parser.h:408 | Reason for calling a `life` rule |
| `receive_action` | Global | parser.h:411 | `##PutOn` or `##Insert` during Receive fake action |
| `action_to_be` | Global | parser.h:527 | Action from the grammar line being tested |
| `action_reversed` | Global | parser.h:528 | True if grammar line uses `reverse` |
| `multiflag` | Global | parser.h:445 | True during multi-object action processing |
| `no_implicit_actions` | Global | parser.h:415 | If true, don't perform implicit takes/opens |
| `consult_from` | Global | parser.h:453 | Starting word of a `topic` token |
| `consult_words` | Global | parser.h:454 | Number of words in the `topic` |

### §I.2.3 Scope Reason Constants

When `SearchScope` is called, it passes a reason code that determines which
property is invoked on in-scope objects. The constants are defined in
`parser.h` (lines 598–606):

| Constant | Value | Purpose |
|----------|-------|---------|
| `PARSING_REASON` | 0 | Normal scope search during parsing |
| `TALKING_REASON` | 1 | Scope search for conversation target |
| `EACH_TURN_REASON` | 2 | Scope search for `each_turn` processing |
| `REACT_BEFORE_REASON` | 3 | Scope search to run `react_before` properties |
| `REACT_AFTER_REASON` | 4 | Scope search to run `react_after` properties |
| `LOOPOVERSCOPE_REASON` | 5 | User-initiated `LoopOverScope()` call |
| `TESTSCOPE_REASON` | 6 | User-initiated `TestScope()` call |

### §I.2.4 RunLife and the life Property

The `RunLife` routine (`parser.h`, line 5633) is the library's mechanism for
dispatching actions to an NPC's `life` property. It is used by communication
actions (`Ask`, `Tell`, `Answer`) and physical interaction actions (`Kiss`,
`Attack`, `ThrowAt`, `Give`, `Show`).

The calling sequence is:

1. Set `reason_code` to the action or fake action passed as an argument.
2. Call `RunRoutines(object, life)` on the target NPC.
3. If the `life` routine returns true, the action is considered handled.
4. If it returns false, the action handler prints a default refusal message.

Inside a `life` property routine, the game can check `reason_code` (or
equivalently `action`) to determine which action triggered the call:

```inform6
life [;
    Give:
        if (noun == gold_coin) "The merchant accepts your payment.";
        "~I don't want that,~ says the merchant.";
    Attack:
        "The merchant ducks behind the counter!";
],
```

Note that `Receive` is a `Fake_Action` generated by `GiveSub` and `ShowSub`.
When an NPC is given or shown an object, the library first tries `Receive` on
the NPC's `life` property (with `noun` set to the object being given or
shown). The variable `receive_action` is set to `##Give` or `##Show` so the
`life` routine can distinguish between the two.
---

## §I.3 Standard Game Actions

This section documents all non-meta game actions in alphabetical order. Each
action is triggered by player input matching a grammar line in `grammar.h` and
handled by a corresponding `*Sub` routine in `verblib.h`.

For meta actions (game-control and debug commands), see §I.4.

---

### Answer

- **Routine:** `AnswerSub` (verblib.h:2696)
- **Meta:** No

**Grammar** (grammar.h:189–190):

```inform6
Verb 'answer' 'say' 'shout' 'speak'
    * topic 'to' creature                       -> Answer;
```

**Default behavior:**

1. Calls `RunLife(second, ##Answer)` to invoke the `life` property of the
   creature addressed by the player.
2. If `RunLife` returns false (the creature's `life` property did not handle the
   action), prints `L__M(##Answer, 1, noun)` — the default "there is no
   reply" message.

**Checks performed:** None beyond `RunLife` delegation to the `life` property.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Answer, 1, noun)` | `RunLife` returned false — no `life` handler consumed the action |

**Related actions:** Ask, AskFor, AskTo, Tell

---

### Ask

- **Routine:** `AskSub` (verblib.h:2701)
- **Meta:** No

**Grammar** (grammar.h:192–193):

```inform6
Verb 'ask'
    * creature 'about' topic                    -> Ask;
```

**Default behavior:**

1. Calls `RunLife(noun, ##Ask)` to invoke the `life` property of the creature
   being asked.
2. If `RunLife` returns false, prints `L__M(##Ask, 1, noun)` — the default
   "there is no reply" message.

**Checks performed:** None beyond `RunLife` delegation to the `life` property.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Ask, 1, noun)` | `RunLife` returned false — no `life` handler consumed the action |

**Related actions:** AskFor, AskTo, Answer, Tell

---

### AskFor

- **Routine:** `AskForSub` (verblib.h:2706)
- **Meta:** No

**Grammar** (grammar.h:192, 194):

```inform6
Verb 'ask'
    * creature 'for' noun                       -> AskFor;
```

**Default behavior:**

1. If `noun == player`, the action is redirected to `<<Inv, actor>>` — the
   player is effectively asking for their own inventory.
2. Otherwise, prints `L__M(##Order, 1, noun)` — the default refusal message
   (shared with the Order mechanism).

**Checks performed:** Compares `noun` against `player`.

**World-model effects:** None (may redirect to Inv).

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Order, 1, noun)` | `noun` is not the player — creature has no specific response |

**Related actions:** Ask, Give

---

### AskTo

- **Routine:** `AskToSub` (verblib.h:2711)
- **Meta:** No

**Grammar** (grammar.h:192, 195–196):

```inform6
Verb 'ask'
    * creature 'to' topic                       -> AskTo
    * 'that' creature topic                     -> AskTo;
```

**Default behavior:**

1. Prints `L__M(##Order, 1, noun)` — the default refusal message (shared with
   the Order mechanism).

**Checks performed:** None.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Order, 1, noun)` | Always (default refusal) |

**Related actions:** Ask, Tell, Order (fake action)

---

### Attack

- **Routine:** `AttackSub` (verblib.h:2713)
- **Meta:** No

**Grammar** (grammar.h:198–201):

```inform6
Verb 'attack' 'break' 'crack' 'destroy' 'fight' 'hit' 'kill'
     'murder' 'punch' 'smash' 'thump' 'torture' 'wreck'
    * noun                                      -> Attack;
```

**Default behavior:**

1. Calls `ObjectIsUntouchable(noun)` — if the noun cannot be touched the action
   stops early.
2. If `noun` has the `animate` attribute, calls `RunLife(noun, ##Attack)` to
   let the creature's `life` property respond.
3. If `RunLife` returns false (or the noun is not animate), prints
   `L__M(##Attack, 1, noun)`.

**Checks performed:**

| Check | Detail |
|-------|--------|
| Touchability | `ObjectIsUntouchable(noun)` |
| `animate` | Determines whether `RunLife` is called |

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Attack, 1, noun)` | `RunLife` returned false, or noun is not `animate` |

**Related actions:** None directly; creature responses are handled via `life`.

---

### Blow

- **Routine:** `BlowSub` (verblib.h:2719)
- **Meta:** No

**Grammar** (grammar.h:203–204):

```inform6
Verb 'blow'
    * held                                      -> Blow;
```

**Default behavior:**

1. Prints `L__M(##Blow, 1, noun)`.

This is a pure stub — it performs no checks and changes no state. Override it
in a `before` rule or the object's `before` property to implement custom
behavior (e.g., blowing a whistle).

**Checks performed:** None.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Blow, 1, noun)` | Always |

**Related actions:** None.

---

### Burn

- **Routine:** `BurnSub` (verblib.h:2721)
- **Meta:** No

**Grammar** (grammar.h:210–212):

```inform6
Verb 'burn' 'light'
    * noun                                      -> Burn
    * noun 'with' held                          -> Burn;
```

**Default behavior:**

1. If `noun` has the `animate` attribute, prints `L__M(##Burn, 2, noun)` (you
   can't set fire to a living creature) and returns.
2. Otherwise prints `L__M(##Burn, 1, noun)` (a generic refusal).

This is a stub — it does not actually burn anything. Override to implement
combustion.

**Checks performed:**

| Check | Detail |
|-------|--------|
| `animate` | Selects which refusal message to print |

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Burn, 1, noun)` | `noun` is not `animate` |
| `L__M(##Burn, 2, noun)` | `noun` has `animate` |

**Related actions:** None.

---

### Buy

- **Routine:** `BuySub` (verblib.h:2726)
- **Meta:** No

**Grammar** (grammar.h:214–215):

```inform6
Verb 'buy' 'purchase'
    * noun                                      -> Buy;
```

**Default behavior:**

1. Prints `L__M(##Buy, 1, noun)`.

This is a pure stub. Override to implement commerce.

**Checks performed:** None.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Buy, 1, noun)` | Always |

**Related actions:** None.

---

### Climb

- **Routine:** `ClimbSub` (verblib.h:2728)
- **Meta:** No

**Grammar** (grammar.h:222–224):

```inform6
Verb 'climb' 'scale'
    * noun                                      -> Climb
    * 'up'/'over' noun                          -> Climb;
```

**Default behavior:**

1. If `noun` has the `animate` attribute, prints `L__M(##Climb, 2, noun)` and
   returns.
2. Otherwise prints `L__M(##Climb, 1, noun)`.

This is a stub. Override to let the player climb specific objects.

**Checks performed:**

| Check | Detail |
|-------|--------|
| `animate` | Selects which refusal message to print |

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Climb, 1, noun)` | `noun` is not `animate` |
| `L__M(##Climb, 2, noun)` | `noun` has `animate` |

**Related actions:** None.

---

### Close

- **Routine:** `CloseSub` (verblib.h:2633)
- **Meta:** No

**Grammar** (grammar.h:226–228):

```inform6
Verb 'close' 'cover' 'shut'
    * noun                                      -> Close
    * 'up' noun                                 -> Close;
```

**Default behavior:**

1. Calls `ObjectIsUntouchable(noun)` — aborts if the noun cannot be touched.
2. If `noun` does not have the `openable` attribute, prints
   `L__M(##Close, 1, noun)` (can't be closed).
3. If `noun` does not have the `open` attribute (already closed), prints
   `L__M(##Close, 2, noun)`.
4. Otherwise, gives `noun` `~open` (clears the `open` attribute).
5. Calls `AfterRoutines()`. If not intercepted and not `keep_silent`, prints
   `L__M(##Close, 3, noun)` (success confirmation).

**Checks performed:**

| Check | Detail |
|-------|--------|
| Touchability | `ObjectIsUntouchable(noun)` |
| `openable` | Must be present to attempt closing |
| `open` | Must be set — object must currently be open |

**World-model effects:**

| Effect | Detail |
|--------|--------|
| `give noun ~open` | Clears the `open` attribute on `noun` |

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Close, 1, noun)` | `noun` is not `openable` |
| `L__M(##Close, 2, noun)` | `noun` is already closed (`hasnt open`) |
| `L__M(##Close, 3, noun)` | Success — `noun` is now closed |

**Related actions:** Open

---

### Consult

- **Routine:** `ConsultSub` (verblib.h:2733)
- **Meta:** No

**Grammar** (grammar.h:231–233):

```inform6
Verb 'consult'
    * noun 'about' topic                        -> Consult
    * noun 'on' topic                           -> Consult;
```

**Default behavior:**

1. Prints `L__M(##Consult, 1, noun)`.

This is a pure stub. The topic text typed by the player is available via the
globals `consult_from` (word number where the topic begins) and `consult_words`
(number of words in the topic). Override the object's `before` property to
implement look-up behavior (e.g., consulting a book about a subject).

**Checks performed:** None.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Consult, 1, noun)` | Always |

**Related actions:** None.

---

### Cut

- **Routine:** `CutSub` (verblib.h:2735)
- **Meta:** No

**Grammar** (grammar.h:235–236):

```inform6
Verb 'cut' 'chop' 'prune' 'slice'
    * noun                                      -> Cut;
```

**Default behavior:**

1. If `noun` has the `animate` attribute, prints `L__M(##Cut, 2, noun)` and
   returns.
2. Otherwise prints `L__M(##Cut, 1, noun)`.

This is a stub. Override to implement cutting.

**Checks performed:**

| Check | Detail |
|-------|--------|
| `animate` | Selects which refusal message to print |

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Cut, 1, noun)` | `noun` is not `animate` |
| `L__M(##Cut, 2, noun)` | `noun` has `animate` |

**Related actions:** None.

---

### Dig

- **Routine:** `DigSub` (verblib.h:2740)
- **Meta:** No

**Grammar** (grammar.h:238–242):

```inform6
Verb 'dig'
    * noun                                      -> Dig
    * noun 'with' held                          -> Dig
    * 'in' noun                                 -> Dig
    * 'in' noun 'with' held                     -> Dig;
```

**Default behavior:**

1. Prints `L__M(##Dig, 1, noun)`.

This is a pure stub. Override to implement digging.

**Checks performed:** None.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Dig, 1, noun)` | Always |

**Related actions:** None.

---

### Disrobe

- **Routine:** `DisrobeSub` (verblib.h:2643)
- **Meta:** No

**Grammar** (grammar.h:245–246, 393–394):

```inform6
Verb 'disrobe' 'doff' 'shed'
    * multi                                     -> Disrobe;

Verb 'remove'
    * worn                                      -> Disrobe;
```

**Default behavior:**

1. Calls `ObjectIsUntouchable(noun)` — aborts if the noun cannot be touched.
2. If `noun` does not have the `clothing` attribute, prints
   `L__M(##Disrobe, 1, noun)` (not something you can take off).
3. If `noun` does not have the `worn` attribute (not currently worn), prints
   `L__M(##Disrobe, 2, noun)`.
4. Otherwise, gives `noun` `~worn` (clears the `worn` attribute).
5. Calls `AfterRoutines()`.

**Checks performed:**

| Check | Detail |
|-------|--------|
| Touchability | `ObjectIsUntouchable(noun)` |
| `clothing` | Must be present — object must be a garment |
| `worn` | Must be set — object must currently be worn |

**World-model effects:**

| Effect | Detail |
|--------|--------|
| `give noun ~worn` | Clears the `worn` attribute on `noun` |

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Disrobe, 1, noun)` | `noun` is not `clothing` |
| `L__M(##Disrobe, 2, noun)` | `noun` is not currently worn (`hasnt worn`) |

**Related actions:** Wear

---

### Drink

- **Routine:** `DrinkSub` (verblib.h:2742)
- **Meta:** No

**Grammar** (grammar.h:248–249):

```inform6
Verb 'drink' 'sip' 'swallow'
    * noun                                      -> Drink;
```

**Default behavior:**

1. Prints `L__M(##Drink, 1, noun)`.

This is a pure stub. Override to implement drinking.

**Checks performed:** None.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Drink, 1, noun)` | Always |

**Related actions:** None.

---

### Drop

- **Routine:** `DropSub` (verblib.h:1890)
- **Meta:** No

**Grammar** (grammar.h:251–252, 381, 385–386):

```inform6
Verb 'drop' 'discard'
    * multiheld                                 -> Drop;

Verb 'put'
    * 'down' multiheld                          -> Drop
    * multiheld 'down'                          -> Drop;
```

**Default behavior:**

1. If `noun == actor`, prints `L__M(##PutOn, 4, noun)` (you can't drop
   yourself) and returns.
2. If `noun` is already in `parent(actor)` (already on the floor / in the
   current location), prints `L__M(##Drop, 1, noun)`.
3. If `noun` has the `worn` attribute, calls `ImplicitDisrobe()` to silently
   remove the garment first.
4. If `noun` is not in the actor after implicit operations, prints
   `L__M(##Drop, 4, noun, parent(noun))` or `L__M(##Miscellany, 58)`.
5. Moves `noun` to `parent(actor)` (drops it into the actor's immediate
   environment).
6. Calls `AfterRoutines()`. If not intercepted and not `keep_silent`, prints
   `L__M(##Drop, 3, noun)` (success confirmation).

**Checks performed:**

| Check | Detail |
|-------|--------|
| Self-reference | `noun == actor` |
| Already dropped | `noun in parent(actor)` |
| `worn` | If set, triggers implicit disrobing |
| Possession | `noun` must still be in `actor` after implicit actions |

**World-model effects:**

| Effect | Detail |
|--------|--------|
| `move noun to parent(actor)` | Object is placed in the actor's containing room or object |
| `give noun ~worn` | (via `ImplicitDisrobe`, if applicable) |

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##PutOn, 4, noun)` | Trying to drop yourself |
| `L__M(##Drop, 1, noun)` | `noun` is already in `parent(actor)` |
| `L__M(##Drop, 4, noun, parent(noun))` | `noun` is not held after implicit operations |
| `L__M(##Miscellany, 58)` | Alternative "not held" message |
| `L__M(##Drop, 3, noun)` | Success — object dropped |

**Related actions:** Take, PutOn, Insert, Transfer

---

### Eat

- **Routine:** `EatSub` (verblib.h:2672)
- **Meta:** No

**Grammar** (grammar.h:256–257):

```inform6
Verb 'eat'
    * held                                      -> Eat;
```

**Default behavior:**

1. Calls `ObjectIsUntouchable(noun)` — aborts if the noun cannot be touched.
2. If `noun` does not have the `edible` attribute, prints
   `L__M(##Eat, 1, noun)` (not edible).
3. If `noun` has the `worn` attribute, calls `ImplicitDisrobe()` to silently
   remove the garment first.
4. Removes `noun` from the game entirely (`remove noun` — the object is
   destroyed / consumed).
5. Calls `AfterRoutines()`. If not intercepted, prints `L__M(##Eat, 2, noun)`
   (success confirmation).

**Checks performed:**

| Check | Detail |
|-------|--------|
| Touchability | `ObjectIsUntouchable(noun)` |
| `edible` | Must be present — object must be edible |
| `worn` | If set, triggers implicit disrobing |

**World-model effects:**

| Effect | Detail |
|--------|--------|
| `remove noun` | Object is removed from the object tree (destroyed) |
| `give noun ~worn` | (via `ImplicitDisrobe`, if applicable) |

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Eat, 1, noun)` | `noun` is not `edible` |
| `L__M(##Eat, 2, noun)` | Success — object has been eaten |

**Related actions:** None.

---

### Empty

- **Routine:** `EmptySub` (verblib.h:2017)
- **Meta:** No

**Grammar** (grammar.h:259–262):

```inform6
Verb 'empty'
    * noun                                      -> Empty
    * 'out' noun                                -> Empty
    * noun 'out'                                -> Empty;
```

**Default behavior:**

1. Sets `second` to `d_obj` (the default object representing the ground /
   floor).
2. Falls through to `EmptyTSub` to perform the actual emptying.

This routine is a thin wrapper that supplies a default destination (the ground)
before delegating to `EmptyT`.

**Checks performed:** None (delegated to `EmptyTSub`).

**World-model effects:** None directly (delegated to `EmptyTSub`).

**Library messages:** None directly (all messages are produced by `EmptyTSub`).

**Related actions:** EmptyT

---

### EmptyT

- **Routine:** `EmptyTSub` (verblib.h:2019)
- **Meta:** No

**Grammar** (grammar.h:259, 263):

```inform6
Verb 'empty'
    * noun 'to'/'into'/'on'/'onto' noun         -> EmptyT;
```

**Default behavior:**

1. If `noun == second`, prints `L__M(##EmptyT, 4)` (can't empty something into
   itself).
2. If `noun` does not have the `container` attribute, prints
   `L__M(##EmptyT, 1, noun)` (not a container).
3. If `noun` does not have the `open` attribute, prints
   `L__M(##EmptyT, 2, noun)` (container is closed).
4. Validates `second` as a legal destination. Calls `ObjectIsUntouchable()` and
   `ImplicitOpen()` as needed on the destination.
5. If `noun` has no children (is already empty), prints
   `L__M(##EmptyT, 3, noun)`.
6. Otherwise, loops through all children of `noun` and issues
   `<<Transfer child second, actor>>` for each, moving every item to the
   destination.

**Checks performed:**

| Check | Detail |
|-------|--------|
| Self-reference | `noun == second` |
| `container` | `noun` must be a container |
| `open` | `noun` must be open |
| `supporter` / `container` | `second` must be a valid destination |
| Touchability | `ObjectIsUntouchable()` on destination |
| Children | `noun` must have at least one child |

**World-model effects:**

| Effect | Detail |
|--------|--------|
| Transfer all children | Each child of `noun` is moved to `second` via the `Transfer` action |

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##EmptyT, 1, noun)` | `noun` is not a `container` |
| `L__M(##EmptyT, 2, noun)` | `noun` is closed (`hasnt open`) |
| `L__M(##EmptyT, 3, noun)` | `noun` is already empty (no children) |
| `L__M(##EmptyT, 4)` | `noun == second` (emptying into itself) |

**Related actions:** Empty, Transfer

---

### Enter

- **Routine:** `EnterSub` (verblib.h:2085)
- **Meta:** No

**Grammar** (grammar.h:265, 267, 281, 283, 420–422, 440):

```inform6
Verb 'enter' 'cross'
    * noun                                      -> Enter;

Verb 'get'
    * 'in'/'into'/'on' noun                     -> Enter;

Verb 'sit' 'lie'
    * 'on' 'top' 'of' noun                      -> Enter
    * 'on'/'in'/'inside' noun                   -> Enter;

Verb 'stand'
    * 'on' noun                                 -> Enter;
```

**Default behavior:**

1. If `noun` has the `door` attribute, redirects to `<<Go noun>>`.
2. If `noun` is in the `Compass` (a direction object), redirects to
   `<<Go noun>>`.
3. If the actor is already inside `noun`, prints `L__M(##Enter, 1, noun)`.
4. If `noun` does not have the `enterable` attribute, prints
   `L__M(##Enter, 2, noun)` or `L__M(##Enter, 3, noun)` depending on context.
5. If `noun` has `container` but not `open`, calls `ImplicitOpen(noun)` to
   silently open it first; aborts if this fails.
6. Navigates nested containment: finds the common ancestor of the actor and
   `noun` via `CommonAncestor()`, then calls `ImplicitExit()` as needed to
   move the actor up the object tree to a shared parent.
7. Moves the actor into `noun` (`move actor to noun`).
8. Calls `Locale()` to describe the new local environment and
   `AfterRoutines()` to allow after-rules to fire.
9. On success, prints `L__M(##Enter, 5, noun)` or `L__M(##Enter, 7, noun)`.

**Checks performed:**

| Check | Detail |
|-------|--------|
| `door` | Redirects to `Go` if present |
| `Compass` membership | Redirects to `Go` if noun is a direction |
| Already inside | Actor's parent is `noun` |
| `enterable` | Must be present on `noun` |
| `container` | If present, `open` must also be set (or implicitly opened) |
| Ancestry | `CommonAncestor()` navigates nested containment |

**World-model effects:**

| Effect | Detail |
|--------|--------|
| `move actor to noun` | Actor is placed inside or on top of `noun` |

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Enter, 1, noun)` | Actor is already in/on `noun` |
| `L__M(##Enter, 2, noun)` | `noun` is not `enterable` (general) |
| `L__M(##Enter, 3, noun)` | `noun` is not `enterable` (alternative phrasing) |
| `L__M(##Enter, 4, noun)` | Cannot reach `noun` — implicit exit failed |
| `L__M(##Enter, 5, noun)` | Success — actor enters `noun` (container) |
| `L__M(##Enter, 6, noun)` | Implicit exit message during nested entry |
| `L__M(##Enter, 7, noun)` | Success — actor gets on `noun` (supporter) |

**Related actions:** Exit, GetOff, Go

---

### Examine

- **Routine:** `ExamineSub` (verblib.h:2519)
- **Meta:** No

**Grammar** (grammar.h:269–270, 388–389):

```inform6
Verb 'examine' 'x//' 'check' 'describe' 'watch'
    * noun                                      -> Examine;

Verb 'read'
    * noun                                      -> Examine;
```

**Default behavior:**

1. If `location == thedark`, prints `L__M(##Examine, 1, noun)` (too dark to
   see) and returns.
2. If `noun` provides a `description` property, calls
   `PrintOrRun(noun, description)` to display it, then calls
   `AfterRoutines()` and returns.
3. If `noun` has the `container` attribute and is either `open` or
   `transparent`, redirects to `<<Search noun>>` to list the container's
   contents.
4. If `noun` has the `switchable` attribute, prints `L__M(##Examine, 3, noun)`
   reporting whether the device is switched on or off.
5. Otherwise prints `L__M(##Examine, 2, noun)` (nothing special about the
   noun).

**Checks performed:**

| Check | Detail |
|-------|--------|
| Darkness | `location == thedark` |
| `description` property | Checked first — if provided, it is printed |
| `container` + (`open` or `transparent`) | Triggers `<<Search noun>>` to list contents |
| `switchable` | Reports on/off state if no description is provided |

**World-model effects:** None (purely informational).

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Examine, 1, noun)` | Too dark to examine anything |
| `L__M(##Examine, 2, noun)` | No `description` property and no special attributes |
| `L__M(##Examine, 3, noun)` | `noun` has `switchable` — reports on/off state |

**Related actions:** Search, LookUnder

---

### Exit

- **Routine:** `ExitSub` (verblib.h:2135)
- **Meta:** No

**Grammar** (grammar.h:272–274, 437–439):

```inform6
Verb 'exit' 'out' 'outside'
    *                                           -> Exit
    * noun                                      -> Exit;

Verb 'stand'
    *                                           -> Exit
    * 'up'                                      -> Exit;
```

**Default behavior:**

1. If `noun` is specified (not `nothing`), redirects to the noun's handler.
2. Determines the actor's immediate container `p = parent(actor)`.
3. If `p == location` (actor is directly in the room, not inside any object):
   - If `real_location` provides an `out_to` property, redirects to
     `<<Go out_obj>>`.
   - Otherwise prints `L__M(##Exit, 1)` (no obvious exit).
4. If `p` has `container` but not `open`, calls `ImplicitOpen(p)` to silently
   open the container.
5. If the actor or player provides a `posture` property, resets it (clears any
   sitting / lying state).
6. Calls `RunRoutines(p, before)` with the action set to `##Exit` to let the
   container run before-rules.
7. Moves the actor to `parent(p)` (one level up in the object tree).
8. Checks for darkness and adjusts `location` if needed.
9. Calls `AfterRoutines()`. If the location has changed, calls `LookSub()` to
   redisplay the room description.

**Checks performed:**

| Check | Detail |
|-------|--------|
| `noun` specified | If so, delegates to noun's handler |
| `p == location` | Determines room-exit vs. object-exit behavior |
| `out_to` property | On `real_location`, used for room exits |
| `container` + `open` | If `p` is a closed container, attempts implicit open |
| `posture` property | Reset on exit |
| Darkness | Rechecked after moving |

**World-model effects:**

| Effect | Detail |
|--------|--------|
| `move actor to parent(p)` | Actor moves out of the current container/supporter |
| Clear `posture` | Posture property reset if present |

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Exit, 1)` | In a room with no `out_to` — no obvious exit |
| `L__M(##Exit, 2)` | Cannot exit closed container (implicit open failed) |
| `L__M(##Exit, 3)` | Leaving a container — narration |
| `L__M(##Exit, 4)` | Leaving a supporter — narration |
| `L__M(##Exit, 6)` | Alternative exit message |

**Related actions:** Enter, GetOff, Go
### Fill

- **Routine:** `FillSub` (verblib.h:2744)
- **Meta:** No

**Grammar** (grammar.h:276–278):

```inform6
Verb 'fill'
    * noun                         -> Fill
    * noun 'from' noun             -> Fill;
```

**Default behavior:**

1. If `second == nothing`, print `L__M(##Fill, 1, noun)`.
2. Otherwise print `L__M(##Fill, 2, noun, second)`.

This is a pure stub action — it performs no world-model changes and exists only to provide a default refusal message that the author can override with `before` or `react_before` rules.

**Checks performed:** none

**World-model effects:** none

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Fill, 1)` | No second noun specified — default refusal |
| `L__M(##Fill, 2)` | A second noun was specified — default refusal |

**Related actions:** —

---

### GetOff

- **Routine:** `GetOffSub` (verblib.h:2130)
- **Meta:** No

**Grammar** (grammar.h:280, 284):

```inform6
Verb 'get'
    * 'off' noun                   -> GetOff;
```

**Default behavior:**

1. If `parent(actor) == noun` (the actor is on/in the noun), redirect to `<<Exit, actor>>`.
2. Otherwise print `L__M(##GetOff, 1, noun)` ("But you aren't on that.").

**Checks performed:** `parent(actor)` relationship

**World-model effects:** none (delegates to `Exit`)

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##GetOff, 1)` | Actor is not on/in the noun |

**Related actions:** Exit, Enter

---

### Give

- **Routine:** `GiveSub` (verblib.h:2057)
- **Meta:** No

**Grammar** (grammar.h:287–290):

```inform6
Verb 'give' 'feed' 'offer' 'pay'
    * creature held                -> Give reverse
    * held 'to' creature           -> Give
    * 'over' held 'to' creature    -> Give;
```

The `reverse` token on the first grammar line swaps `noun` and `second` so that internally the held object is always `noun` and the creature is always `second`.

**Default behavior:**

1. If `noun` is not in `actor`, call `ImplicitTake(noun)`.
2. If `noun has worn`, call `ImplicitDisrobe(noun)`.
3. If `second == actor`, print `L__M(##Give, 1, noun)` ("You can't give something to yourself.").
4. If `second == player` and `actor ~= player` (an NPC giving to the player), move `noun` to `player`, call `AfterRoutines()`, and print `L__M(##Give, 4, noun)`.
5. Otherwise call `RunLife(second, ##Give)`.
6. If `RunLife` returns false, print `L__M(##Give, 3, second)` ("The recipient doesn't seem interested.").

**Checks performed:** `parent` relationships, `worn` attribute

**World-model effects:** `move noun to player` (when an NPC gives to the player)

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Give, 1)` | Trying to give something to yourself |
| `L__M(##Give, 2)` | (reserved) |
| `L__M(##Give, 3)` | Recipient's `life` routine did not handle the action |
| `L__M(##Give, 4)` | NPC successfully gives object to the player |

**Related actions:** GiveR, Show, AskFor

---

### GiveR

- **Routine:** `GiveRSub` (verblib.h:2070)
- **Meta:** No

**Grammar:** (no direct grammar — reached via internal routing when the `reverse` token swaps arguments)

**Default behavior:**

1. Redirect to `<Give second noun, actor>` (swap the two arguments back and re-enter `GiveSub`).

**Checks performed:** none

**World-model effects:** none (delegates to Give)

**Library messages:** none

**Related actions:** Give

---

### Go

- **Routine:** `GoSub` (verblib.h:2167)
- **Meta:** No

**Grammar** (grammar.h:292, 294):

```inform6
Verb 'go' 'run' 'walk'
    * noun=ADirection              -> Go;
```

**Default behavior:**

1. If `second` is provided and is not in `Compass`, reject the command.
2. Look up the direction property (e.g. `n_to`) on the current location. If it yields `nothing`, check the location's `cant_go` property; if set, print it via `PrintOrRun`, otherwise print `L__M(##Go, 1)`.
3. If the result is a **door** object:
   - If the door `has concealed`, print `L__M(##Go, 2)`.
   - If the door is not `open`, attempt `ImplicitOpen(door)`.
   - Evaluate the door's `door_to` property to obtain the destination.
4. Call `ImplicitExit()` to extract the actor from any nested containers/supporters.
5. Handle the `##Going` fake action: temporarily set `action` to `##Going` and call `RunRoutines` on the departure location's `before`. If intercepted, stop.
6. Move the actor to the destination.
7. Call `MoveFloatingObjects()` to relocate "floating" objects that follow the player.
8. Handle darkness transitions via `AdjustLight()` and, if applicable, `DarkToDark()`.
9. Call `LookSub()` to print the new room description.
10. Call `AfterRoutines()`.

**Checks performed:** `door`, `concealed`, `open` attributes; direction properties (`n_to`, `s_to`, etc.); `cant_go`; `door_to`; `door_dir`; `parent` relationships

**World-model effects:** moves `actor` to `next_loc`; updates `location`/`real_location`; may trigger darkness

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Go, 1)` | No exit in that direction and no `cant_go` message |
| `L__M(##Go, 2)` | Door is concealed |
| `L__M(##Go, 3)` | Door is locked and implicit-open failed |
| `L__M(##Go, 4)` | Actor must first leave a nested container/supporter |
| `L__M(##Go, 5)` | Actor can't leave current object |
| `L__M(##Go, 6)` | Direction has no property (shouldn't reach player) |
| `L__M(##Go, 7)` | Arrival in darkness |

**Related actions:** GoIn, VagueGo, Enter, Exit

---

### GoIn

- **Routine:** `GoInSub` (verblib.h:2165)
- **Meta:** No

**Grammar** (grammar.h:265–266, 292/297, 305–306):

```inform6
Verb 'enter' 'cross'
    *                              -> GoIn;
Verb 'go'
    * 'in'/'inside'               -> GoIn;
Verb 'in' 'inside'
    *                              -> GoIn;
```

**Default behavior:**

1. Redirect to `<<Go in_obj, actor>>`.

**Checks performed:** none

**World-model effects:** none (delegates to Go)

**Library messages:** none

**Related actions:** Go, Enter

---

### Insert

- **Routine:** `InsertSub` (verblib.h:1943)
- **Meta:** No

**Grammar** (grammar.h:251/253, 308–309, 381–382):

```inform6
Verb 'drop' 'discard'
    * multiexcept 'in'/'into'/'down' noun  -> Insert;
Verb 'insert'
    * multiexcept 'in'/'into' noun          -> Insert;
Verb 'put'
    * multiexcept 'in'/'inside'/'into' noun -> Insert;
```

**Default behavior:**

1. If `second == d_obj` (the "down" direction object), redirect to `<<Drop noun>>`.
2. If `actor` is in `second`, print `L__M(##Insert, 1)` ("You'd need to get out first.").
3. If `noun` is already in `second`, print `L__M(##Drop, 1)` ("Already there.").
4. Check via `CommonAncestor()` that moving `noun` into `second` would not create a recursive containment loop.
5. If `second hasnt container`, print `L__M(##Insert, 3)`.
6. If `noun has worn`, call `ImplicitDisrobe(noun)`.
7. Check `multiflag` for multiple-object operations and capacity constraints.
8. If `second has container` but `second hasnt open`, call `ImplicitOpen(second)`.
9. Check `ObjectDoesNotFit(noun, second)` and `AtFullCapacity(second)`.
10. Temporarily set `action` to `##Receive` and `receive_action` to `##Insert`; call `RunRoutines(second, before)`. If intercepted, stop.
11. Move `noun` to `second`.
12. Call `AfterRoutines()`.
13. If not intercepted, print `L__M(##Insert, 9, noun, second)`.

**Checks performed:** `container`, `open`, `worn` attributes; `parent` relationships; capacity

**World-model effects:** `move noun to second`; sets `receive_action = ##Insert`

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Insert, 1)` | Actor is inside the target container |
| `L__M(##Insert, 2)` | (reserved/used internally) |
| `L__M(##Insert, 3)` | Second noun is not a container |
| `L__M(##Insert, 5)` | Recursive containment would result |
| `L__M(##Insert, 7)` | Implicit take or disrobe failed |
| `L__M(##Insert, 8)` | Container is full / object does not fit |
| `L__M(##Insert, 9)` | Success — object placed in container |
| `L__M(##Drop, 1)` | Object is already in the container |
| `L__M(##Disrobe, 4)` | Implicit disrobe refusal |

**Related actions:** PutOn, Drop, Transfer, Remove, Receive (fake)

---

### Inv

- **Routine:** `InvSub` (verblib.h:1523)
- **Meta:** No

**Grammar** (grammar.h:311–312; also 217/220, 300/303, 456/460):

```inform6
Verb 'inventory' 'inv' 'i//'
    *                              -> Inv;
Verb 'carry'
    * 'inventory'                  -> Inv;
Verb 'hold'
    * 'inventory'                  -> Inv;
Verb 'take'
    * 'inventory'                  -> Inv;
```

**Default behavior:**

1. If `child(actor) == 0` (nothing carried), print `L__M(##Inv, 1)` and return.
2. If `inventory_style == 0`, set it to a default combination of style bits.
3. If `actor == player`, print `L__M(##Inv, 2)` as the inventory header.
4. Call `WriteListFrom(child(actor), inventory_style)` to list the actor's possessions.
5. Print `L__M(##Inv, 4)` as the inventory footer.
6. Update pronouns via `PronounNotice()`.
7. Call `AfterRoutines()`.
8. If `actor ~= player`, print `L__M(##Inv, 3)` instead of the player-specific messages.

**Checks performed:** `child(actor)`, `inventory_style`, `actor == player`

**World-model effects:** none (informational only)

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Inv, 1)` | Actor is carrying nothing |
| `L__M(##Inv, 2)` | Inventory header (player only) |
| `L__M(##Inv, 3)` | NPC inventory listing |
| `L__M(##Inv, 4)` | Inventory footer |

**Related actions:** InvTall, InvWide

---

### InvTall

- **Routine:** `InvTallSub` (verblib.h:1515)
- **Meta:** No

**Grammar** (grammar.h:311, 313):

```inform6
Verb 'inventory' 'inv' 'i//'
    * 'tall'                       -> InvTall;
```

**Default behavior:**

1. Set `inventory_style` to `NEWLINE_BIT + INDENT_BIT + FULLINV_BIT + RECURSE_BIT` (or a variant if `actor ~= player`).
2. Redirect to `<Inv, actor>`.

**Checks performed:** none

**World-model effects:** sets `inventory_style`

**Library messages:** none (delegates to Inv)

**Related actions:** Inv, InvWide

---

### InvWide

- **Routine:** `InvWideSub` (verblib.h:1507)
- **Meta:** No

**Grammar** (grammar.h:311, 314):

```inform6
Verb 'inventory' 'inv' 'i//'
    * 'wide'                       -> InvWide;
```

**Default behavior:**

1. Set `inventory_style` to `ENGLISH_BIT + FULLINV_BIT + RECURSE_BIT` (or a variant if `actor ~= player`).
2. Redirect to `<Inv, actor>`.

**Checks performed:** none

**World-model effects:** sets `inventory_style`

**Library messages:** none (delegates to Inv)

**Related actions:** Inv, InvTall

---

### Jump

- **Routine:** `JumpSub` (verblib.h:2749)
- **Meta:** No

**Grammar** (grammar.h:316–317):

```inform6
Verb 'jump' 'hop' 'skip'
    *                              -> Jump;
```

**Default behavior:**

1. Print `L__M(##Jump, 1, noun)`.

This is a pure stub action — it performs no world-model changes and exists only to provide a default refusal message.

**Checks performed:** none

**World-model effects:** none

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Jump, 1)` | Default refusal |

**Related actions:** JumpIn, JumpOn, JumpOver

---

### JumpIn

- **Routine:** `JumpInSub` (verblib.h:2751)
- **Meta:** No

**Grammar** (grammar.h:316, 318–319):

```inform6
Verb 'jump' 'hop' 'skip'
    * 'in' noun                    -> JumpIn
    * 'into' noun                  -> JumpIn;
```

**Default behavior:**

1. If `noun has animate`, print `L__M(##JumpIn, 2, noun)`.
2. If `noun has enterable`, redirect to `<<Enter noun>>`.
3. Otherwise print `L__M(##JumpIn, 1, noun)`.

**Checks performed:** `animate`, `enterable` attributes

**World-model effects:** none (may delegate to Enter)

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##JumpIn, 1)` | Noun is not enterable |
| `L__M(##JumpIn, 2)` | Noun is animate |

**Related actions:** JumpOn, JumpOver, Enter

---

### JumpOn

- **Routine:** `JumpOnSub` (verblib.h:2757)
- **Meta:** No

**Grammar** (grammar.h:316, 320–321):

```inform6
Verb 'jump' 'hop' 'skip'
    * 'on' noun                    -> JumpOn
    * 'upon' noun                  -> JumpOn;
```

**Default behavior:**

1. If `noun has animate`, print `L__M(##JumpOn, 2, noun)`.
2. If `noun has enterable` and `noun has supporter`, redirect to `<<Enter noun>>`.
3. Otherwise print `L__M(##JumpOn, 1, noun)`.

**Checks performed:** `animate`, `enterable`, `supporter` attributes

**World-model effects:** none (may delegate to Enter)

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##JumpOn, 1)` | Noun is not an enterable supporter |
| `L__M(##JumpOn, 2)` | Noun is animate |

**Related actions:** JumpIn, JumpOver, Enter

---

### JumpOver

- **Routine:** `JumpOverSub` (verblib.h:2763)
- **Meta:** No

**Grammar** (grammar.h:316, 322):

```inform6
Verb 'jump' 'hop' 'skip'
    * 'over' noun                  -> JumpOver;
```

**Default behavior:**

1. If `noun has animate`, print `L__M(##JumpOver, 2, noun)`.
2. Otherwise print `L__M(##JumpOver, 1, noun)`.

This is a pure stub action — it provides default refusal messages and performs no world-model changes.

**Checks performed:** `animate` attribute

**World-model effects:** none

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##JumpOver, 1)` | Default refusal |
| `L__M(##JumpOver, 2)` | Noun is animate |

**Related actions:** JumpIn, JumpOn

---

### Kiss

- **Routine:** `KissSub` (verblib.h:2768)
- **Meta:** No

**Grammar** (grammar.h:324–325):

```inform6
Verb 'kiss' 'embrace' 'hug'
    * creature                     -> Kiss;
```

**Default behavior:**

1. If `noun == actor`, print `L__M(##Touch, 3, noun)` (self-interaction message shared with Touch).
2. Call `ObjectIsUntouchable(noun)` — if untouchable, stop.
3. Call `RunLife(noun, ##Kiss)`.
4. If `RunLife` returns false, print `L__M(##Kiss, 1, noun)`.

**Checks performed:** self-reference (`noun == actor`)

**World-model effects:** none

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Touch, 3)` | Kissing yourself |
| `L__M(##Kiss, 1)` | Creature's `life` routine did not handle the action |

**Related actions:** Touch

---

### Listen

- **Routine:** `ListenSub` (verblib.h:2775)
- **Meta:** No

**Grammar** (grammar.h:333–336):

```inform6
Verb 'listen' 'hear'
    *                              -> Listen
    * noun                         -> Listen
    * 'to' noun                    -> Listen;
```

**Default behavior:**

1. Print `L__M(##Listen, 1, noun)`.

This is a pure stub action — it performs no world-model changes and exists only to provide a default refusal message.

**Checks performed:** none

**World-model effects:** none

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Listen, 1)` | Default refusal |

**Related actions:** —

---

### Lock

- **Routine:** `LockSub` (verblib.h:2579)
- **Meta:** No

**Grammar** (grammar.h:338–339):

```inform6
Verb 'lock'
    * noun 'with' held             -> Lock;
```

**Default behavior:**

1. Call `ObjectIsUntouchable(noun)` — if untouchable, stop.
2. If `noun hasnt lockable`, print `L__M(##Lock, 1, noun)`.
3. If `noun has locked`, print `L__M(##Lock, 2, noun)` ("It's already locked.").
4. If `noun has open`, call `ImplicitClose(noun)`.
5. Check the `with_key` property of `noun`:
   - If it is an Object, it must match `second`; otherwise print `L__M(##Lock, 4, second)` ("That doesn't seem to fit.").
   - If it is a Routine, call it to validate the key.
6. Give `noun` the `locked` attribute.
7. Call `AfterRoutines()`.
8. If not intercepted, print `L__M(##Lock, 5, noun)` ("You lock …").

**Checks performed:** `lockable`, `locked`, `open` attributes; `with_key` property

**World-model effects:** gives `noun` the `locked` attribute; may call `ImplicitClose()`

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Lock, 1)` | Noun is not lockable |
| `L__M(##Lock, 2)` | Noun is already locked |
| `L__M(##Lock, 3)` | Implicit close failed |
| `L__M(##Lock, 4)` | Key does not match |
| `L__M(##Lock, 5)` | Success — noun is now locked |

**Related actions:** Unlock, Open, Close

---

### Look

- **Routine:** `LookSub` (verblib.h:2446)
- **Meta:** No

**Grammar** (grammar.h:341–342):

```inform6
Verb 'look' 'l//'
    *                              -> Look;
```

**Default behavior:**

1. If `parent(player) == 0`, signal a runtime error and stop.
2. Call `NoteArrival()`.
3. Call `FindVisibilityLevels()` to determine the `visibility_ceiling`.
4. If in darkness, print `L__M(##Look, 1)` (the darkness description).
5. Otherwise print the room name via `L__M(##Look, 2)`.
6. If the actor is nested inside an intermediate object (on a supporter or in a container), print `L__M(##Look, 3)` describing the nesting.
7. Determine whether to print the long room description based on `lookmode` and the `visited` attribute.
8. Print the location's `description` property via `PrintOrRun()`.
9. Call `LookRoutine()` for each visibility level.
10. Call `Locale()` to list visible objects in scope.
11. Call `ScoreArrival()` to award any first-visit points.
12. Call `AfterRoutines()`.
13. Give `location` the `visited` attribute.

**Checks performed:** visibility levels; `container`, `open`, `transparent`, `supporter`, `visited` attributes; `lookmode`; darkness

**World-model effects:** gives `location` the `visited` attribute (if not already set); may award score via `ScoreArrival()`

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Look, 1)` | In darkness |
| `L__M(##Look, 2)` | Room name header |
| `L__M(##Look, 3)` | Actor is on/in a nested object |
| `L__M(##Look, 4)` | "On the …" (supporter nesting detail) |
| `L__M(##Look, 5)` | Room description style indicator |
| `L__M(##Look, 6)` | "You can also see …" listing |

**Related actions:** Examine, Search, LMode1, LMode2, LMode3

---

### LookUnder

- **Routine:** `LookUnderSub` (verblib.h:2534)
- **Meta:** No

**Grammar** (grammar.h:341, 345):

```inform6
Verb 'look' 'l//'
    * 'under' noun                 -> LookUnder;
```

**Default behavior:**

1. If `location == thedark`, print `L__M(##LookUnder, 1, noun)` ("It is too dark to see.").
2. Otherwise print `L__M(##LookUnder, 2)` ("You find nothing of interest.").

This is a pure stub action — it performs no world-model changes and exists only to provide a default refusal message that the author can override.

**Checks performed:** darkness (`location == thedark`)

**World-model effects:** none

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##LookUnder, 1)` | Too dark to look under anything |
| `L__M(##LookUnder, 2)` | Default refusal — nothing found |

**Related actions:** Look, Examine, Search
### Mild

- **Routine:** `MildSub` (verblib.h:2777)
- **Meta:** No

**Grammar** (grammar.h:206–208):

```inform6
Verb 'bother' 'curses' 'darn' 'drat'
    * -> Mild
    * topic -> Mild;
```

**Default behavior:**

1. Prints `L__M(##Mild, 1, noun)`.

**Checks performed:** None.

**World-model effects:** None (pure stub).

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Mild, 1)` | Always |

**Related actions:** Strong

---

### No

- **Routine:** `NoSub` (verblib.h:2779)
- **Meta:** No

**Grammar** (grammar.h:350–351):

```inform6
Verb 'no'
    * -> No;
```

**Default behavior:**

1. Prints `L__M(##No)`.

**Checks performed:** None.

**World-model effects:** None (pure stub).

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##No)` | Always |

**Related actions:** Yes

---

### Open

- **Routine:** `OpenSub` (verblib.h:2614)
- **Meta:** No

**Grammar** (grammar.h:353–354):

```inform6
Verb 'open' 'uncover' 'undo' 'unwrap'
    * noun -> Open;
```

**Default behavior:**

1. Calls `ObjectIsUntouchable(noun)`.
2. If `noun` hasnt `openable`, prints `L__M(##Open, 1, noun)`.
3. If `noun` has `locked`, calls `ImplicitUnlock()`.
4. If `noun` has `open`, prints `L__M(##Open, 2, noun)`.
5. Gives `noun` `open`.
6. Calls `AfterRoutines()`.
7. If not intercepted: checks if `noun` is a `container` (not `transparent`, not a `door`, not `IndirectlyContains(noun, player)`) and has visible contents — prints `L__M(##Open, 4, noun)` with a contents listing. Otherwise prints `L__M(##Open, 5, noun)`.

**Checks performed:** `openable`, `locked`, `open`, `container`, `transparent`, `door` attributes; visible contents.

**World-model effects:** Gives `noun` `open`; may call `ImplicitUnlock()`.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Open, 1)` | `noun` is not openable |
| `L__M(##Open, 2)` | `noun` is already open |
| `L__M(##Open, 3)` | `noun` is locked (after failed implicit unlock) |
| `L__M(##Open, 4)` | Opened successfully, revealing contents |
| `L__M(##Open, 5)` | Opened successfully, nothing special to reveal |

**Related actions:** Close, Lock, Unlock

---

### Pray

- **Routine:** `PraySub` (verblib.h:2781)
- **Meta:** No

**Grammar** (grammar.h:365–366):

```inform6
Verb 'pray'
    * -> Pray;
```

**Default behavior:**

1. Prints `L__M(##Pray, 1, noun)`.

**Checks performed:** None.

**World-model effects:** None (pure stub).

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Pray, 1)` | Always |

---

### Pull

- **Routine:** `PullSub` (verblib.h:2783)
- **Meta:** No

**Grammar** (grammar.h:373–374):

```inform6
Verb 'pull' 'drag'
    * noun -> Pull;
```

**Default behavior:**

1. Calls `ObjectIsUntouchable(noun)`.
2. If `noun == player` or `noun == actor`, prints `L__M(##Pull, 1, noun)`.
3. If `noun` has `static`, prints `L__M(##Pull, 6, noun)`.
4. If `noun` has `scenery`, prints `L__M(##Pull, 2, noun)`.
5. If `noun` has `animate`, prints `L__M(##Pull, 5, noun)`.
6. Otherwise prints `L__M(##Pull, 4, noun)`.

**Checks performed:** `static`, `scenery`, `animate` attributes; self-reference.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Pull, 1)` | Pulling yourself |
| `L__M(##Pull, 2)` | `noun` is scenery |
| `L__M(##Pull, 4)` | Default refusal |
| `L__M(##Pull, 5)` | `noun` is animate |
| `L__M(##Pull, 6)` | `noun` is static |

**Related actions:** Push, Turn

---

### Push

- **Routine:** `PushSub` (verblib.h:2793)
- **Meta:** No

**Grammar** (grammar.h:376–377):

```inform6
Verb 'push' 'clear' 'move' 'press' 'shift'
    * noun -> Push;
```

**Default behavior:**

1. Calls `ObjectIsUntouchable(noun)`.
2. If `noun == player` or `noun == actor`, prints `L__M(##Push, 1, noun)`.
3. If `noun` has `static`, prints `L__M(##Push, 5, noun)`.
4. If `noun` has `scenery`, prints `L__M(##Push, 2, noun)`.
5. If `noun` has `animate`, prints `L__M(##Push, 5, noun)`.
6. Otherwise prints `L__M(##Push, 4, noun)`.

**Checks performed:** `static`, `scenery`, `animate` attributes; self-reference.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Push, 1)` | Pushing yourself |
| `L__M(##Push, 2)` | `noun` is scenery |
| `L__M(##Push, 4)` | Default refusal |
| `L__M(##Push, 5)` | `noun` is static or animate |

**Related actions:** Pull, Turn, PushDir

---

### PushDir

- **Routine:** `PushDirSub` (verblib.h:2803)
- **Meta:** No

**Grammar** (grammar.h:376, 378):

```inform6
Verb 'push' 'clear' 'move' 'press' 'shift'
    * noun noun -> PushDir;
```

**Default behavior:**

1. Prints `L__M(##PushDir, 1, noun)`.

This is always the default response. The game must provide a `before` rule calling `AllowPushDir()` (verblib.h:2687) for pushing objects between rooms to work.

**Checks performed:** None.

**World-model effects:** None (unless the game provides a `before` rule using `AllowPushDir()`).

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##PushDir, 1)` | Always |

**Related actions:** Push, Go

---

### PutOn

- **Routine:** `PutOnSub` (verblib.h:1905)
- **Meta:** No

**Grammar** (grammar.h:251, 254 and 381, 383):

```inform6
Verb 'drop' 'discard'
    * multiexcept 'on'/'onto' noun -> PutOn;
Verb 'put'
    * multiexcept 'on'/'onto' noun -> PutOn;
```

**Default behavior:**

1. If `second == d_obj`, redirects to `<<Drop noun>>`.
2. If actor is in `second`, prints `L__M(##PutOn, 1)`.
3. If `noun` is already on `second`, prints `L__M(##PutOn, 2)`.
4. Checks `CommonAncestor()` to prevent recursive placement.
5. If `second` hasnt `supporter`, prints `L__M(##PutOn, 3)`.
6. If `noun` has `worn`, calls `ImplicitDisrobe()`.
7. Checks multiflag for multiple-object capacity.
8. Checks `ObjectDoesNotFit()` and `AtFullCapacity()`.
9. Temporarily sets `action` to `##Receive` and `receive_action` to `##PutOn`, calls `RunRoutines(second, before)`.
10. Moves `noun` to `second`.
11. Calls `AfterRoutines()`.
12. If not intercepted, prints `L__M(##PutOn, 8, noun, second)`.

**Checks performed:** `supporter`, `worn` attributes; parent relationships; capacity.

**World-model effects:** Moves `noun` to `second`; sets `receive_action` to `##PutOn`.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##PutOn, 1)` | Actor is already on the supporter |
| `L__M(##PutOn, 2)` | `noun` is already on `second` |
| `L__M(##PutOn, 3)` | `second` is not a supporter |
| `L__M(##PutOn, 6)` | Capacity exceeded |
| `L__M(##PutOn, 7)` | Multi-object capacity exceeded |
| `L__M(##PutOn, 8)` | Success |
| `L__M(##Disrobe, 4)` | Implicit disrobe message |
| `L__M(##Drop, 1)` | Redirected drop |

**Related actions:** Insert, Drop, Transfer, Receive (fake)

---

### Remove

- **Routine:** `RemoveSub` (verblib.h:1868)
- **Meta:** No

**Grammar** (grammar.h:217, 219 and 280, 285 and 393, 395–396 and 456, 459):

```inform6
Verb 'carry'
    * multiinside 'from'/'off' noun -> Remove;
Verb 'get'
    * multiinside 'from'/'off' noun -> Remove;
Verb 'remove'
    * multi -> Remove
    * multiinside 'from' noun -> Remove;
Verb 'take'
    * multiinside 'from' noun -> Remove;
```

**Default behavior:**

1. Gets the parent of `noun` (stored as `i`).
2. If `i` has `container` and `i` hasnt `open`, calls `ImplicitOpen()`.
3. If `noun` has `clothing` and `noun` has `worn`, redirects to `<<Disrobe noun>>`.
4. If `i == actor`, prints `L__M(##Remove, 4)`.
5. If `i` hasnt `container` and `i` hasnt `supporter`, prints `L__M(##Remove, 1)` for animate parents or an error otherwise.
6. Sets `action` to `##Take` and calls `AttemptToTakeObject()`.
7. Calls `AfterRoutines()`.
8. If not intercepted, prints `L__M(##Remove, 3, noun)`.

**Checks performed:** `container`, `open`, `clothing`, `worn`, `supporter`, `animate` attributes; parent relationships.

**World-model effects:** Takes `noun` from its container/supporter into the actor's inventory.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Remove, 1)` | Parent is animate but not a container or supporter |
| `L__M(##Remove, 2)` | Parent is not a container or supporter |
| `L__M(##Remove, 3)` | Success |
| `L__M(##Remove, 4)` | Already held by actor |
| `L__M(##Disrobe, 1)` | Item is worn (redirected) |
| `L__M(##Take, 6)` | Take attempt failed |

**Related actions:** Take, Insert, PutOn

---

### Rub

- **Routine:** `RubSub` (verblib.h:2805)
- **Meta:** No

**Grammar** (grammar.h:398–400):

```inform6
Verb 'rub' 'clean' 'dust' 'polish' 'scrub' 'shine' 'sweep' 'wipe'
    * noun -> Rub;
```

**Default behavior:**

1. Calls `ObjectIsUntouchable(noun)`.
2. If `noun` has `animate`, prints `L__M(##Rub, 2, noun)`.
3. Otherwise prints `L__M(##Rub, 1, noun)`.

**Checks performed:** `animate` attribute.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Rub, 1)` | Default (inanimate object) |
| `L__M(##Rub, 2)` | `noun` is animate |

---

### Search

- **Routine:** `SearchSub` (verblib.h:2544)
- **Meta:** No

**Grammar** (grammar.h:341, 344 and 402–403):

```inform6
Verb 'look' 'l//'
    * 'inside'/'in'/'into'/'through'/'on' noun -> Search;
Verb 'search'
    * noun -> Search;
```

**Default behavior:**

1. If `location == thedark`, prints `L__M(##Search, 1, noun)`.
2. If `noun` has `supporter`: calls `VisibleContents()`. If empty, prints `L__M(##Search, 2, noun)`; otherwise prints `L__M(##Search, 3, noun)` with contents listing.
3. If `noun` hasnt `container`, prints `L__M(##Search, 4, noun)`.
4. If `noun` hasnt `transparent` and `noun` hasnt `open`, calls `ImplicitOpen()`.
5. If `noun` hasnt `open` after implicit attempt, prints `L__M(##Search, 6, noun)`.
6. Calls `VisibleContents()`: if empty, prints `L__M(##Search, 5, noun)`; otherwise prints `L__M(##Search, 7, noun)` with contents listing.
7. Calls `AfterRoutines()`.

**Checks performed:** `supporter`, `container`, `transparent`, `open` attributes; darkness.

**World-model effects:** May call `ImplicitOpen()`.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Search, 1)` | In darkness |
| `L__M(##Search, 2)` | Supporter is empty |
| `L__M(##Search, 3)` | Supporter has visible contents |
| `L__M(##Search, 4)` | `noun` is not a container |
| `L__M(##Search, 5)` | Container is empty |
| `L__M(##Search, 6)` | Container is closed and cannot be opened |
| `L__M(##Search, 7)` | Container has visible contents |

**Related actions:** Examine, Look, LookUnder

---

### Set

- **Routine:** `SetSub` (verblib.h:2811)
- **Meta:** No

**Grammar** (grammar.h:405–406):

```inform6
Verb 'set' 'adjust'
    * noun -> Set;
```

**Default behavior:**

1. Prints `L__M(##Set, 1, noun)`.

**Checks performed:** None.

**World-model effects:** None (pure stub).

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Set, 1)` | Always |

**Related actions:** SetTo

---

### SetTo

- **Routine:** `SetToSub` (verblib.h:2813)
- **Meta:** No

**Grammar** (grammar.h:405, 407):

```inform6
Verb 'set' 'adjust'
    * noun 'to' special -> SetTo;
```

**Default behavior:**

1. Prints `L__M(##SetTo, 1, noun)`.

The `special_word` and `special_number` globals contain the parsed value.

**Checks performed:** None.

**World-model effects:** None (pure stub).

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##SetTo, 1)` | Always |

**Related actions:** Set

---

### Show

- **Routine:** `ShowSub` (verblib.h:2072)
- **Meta:** No

**Grammar** (grammar.h:413–415):

```inform6
Verb 'show' 'display' 'present'
    * creature held -> Show reverse
    * held 'to' creature -> Show;
```

**Default behavior:**

1. If `noun` is not in actor, calls `ImplicitTake()`.
2. If `second == player`, redirects to `<<Examine noun, actor>>`.
3. Calls `RunLife(second, ##Show)`.
4. If `RunLife` returns false, prints `L__M(##Show, 2, second)`.

The grammar uses `reverse` so that "show creature held" swaps `noun` and `second`.

**Checks performed:** Parent relationships.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Show, 1)` | `noun` not held (implicit take failed) |
| `L__M(##Show, 2)` | `second` is uninterested |

**Related actions:** ShowR, Give, Examine

---

### ShowR

- **Routine:** `ShowRSub` (verblib.h:2079)
- **Meta:** No

**Grammar:** (No direct grammar — reached via internal routing.)

**Default behavior:**

1. Redirects to `<<Show second noun, actor>>` (reverses arguments).

**Checks performed:** None.

**World-model effects:** None.

**Library messages:** None.

**Related actions:** Show

---

### Sing

- **Routine:** `SingSub` (verblib.h:2815)
- **Meta:** No

**Grammar** (grammar.h:417–418):

```inform6
Verb 'sing'
    * -> Sing;
```

**Default behavior:**

1. Prints `L__M(##Sing, 1, noun)`.

**Checks performed:** None.

**World-model effects:** None (pure stub).

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Sing, 1)` | Always |

---

### Sleep

- **Routine:** `SleepSub` (verblib.h:2817)
- **Meta:** No

**Grammar** (grammar.h:424–425):

```inform6
Verb 'sleep' 'nap'
    * -> Sleep;
```

**Default behavior:**

1. Prints `L__M(##Sleep, 1, noun)`.

**Checks performed:** None.

**World-model effects:** None (pure stub).

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Sleep, 1)` | Always |

---

### Smell

- **Routine:** `SmellSub` (verblib.h:2819)
- **Meta:** No

**Grammar** (grammar.h:427–429):

```inform6
Verb 'smell' 'sniff'
    * -> Smell
    * noun -> Smell;
```

**Default behavior:**

1. If `noun` is not `nothing` and `noun` has `animate`, prints `L__M(##Smell, 2, noun)`.
2. Otherwise prints `L__M(##Smell, 1, noun)`.

**Checks performed:** `animate` attribute (only when `noun` is specified).

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Smell, 1)` | Default (no noun, or inanimate noun) |
| `L__M(##Smell, 2)` | `noun` is animate |

---

### Sorry

- **Routine:** `SorrySub` (verblib.h:2824)
- **Meta:** No

**Grammar** (grammar.h:431–432):

```inform6
Verb 'sorry'
    * -> Sorry;
```

**Default behavior:**

1. Prints `L__M(##Sorry, 1, noun)`.

**Checks performed:** None.

**World-model effects:** None (pure stub).

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Sorry, 1)` | Always |

---

### Squeeze

- **Routine:** `SqueezeSub` (verblib.h:2826)
- **Meta:** No

**Grammar** (grammar.h:434–435):

```inform6
Verb 'squeeze' 'squash'
    * noun -> Squeeze;
```

**Default behavior:**

1. Calls `ObjectIsUntouchable(noun)`.
2. If `noun` has `animate` and `noun ~= player`, prints `L__M(##Squeeze, 1, noun)`.
3. Otherwise prints `L__M(##Squeeze, 2, noun)`.

**Checks performed:** `animate` attribute; self-reference (allows self-squeezing).

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Squeeze, 1)` | `noun` is animate (and not the player) |
| `L__M(##Squeeze, 2)` | Default (inanimate, or squeezing yourself) |

---

### Strong

- **Routine:** `StrongSub` (verblib.h:2832)
- **Meta:** No

**Grammar** (grammar.h:409–411):

```inform6
Verb 'shit' 'damn' 'fuck' 'sod'
    * -> Strong
    * topic -> Strong;
```

**Default behavior:**

1. Prints `L__M(##Strong, 1, noun)`.

**Checks performed:** None.

**World-model effects:** None (pure stub).

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Strong, 1)` | Always |

**Related actions:** Mild

---

### Swim

- **Routine:** `SwimSub` (verblib.h:2834)
- **Meta:** No

**Grammar** (grammar.h:442–443):

```inform6
Verb 'swim' 'dive'
    * -> Swim;
```

**Default behavior:**

1. Prints `L__M(##Swim, 1, noun)`.

**Checks performed:** None.

**World-model effects:** None (pure stub).

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Swim, 1)` | Always |

---

### Swing

- **Routine:** `SwingSub` (verblib.h:2836)
- **Meta:** No

**Grammar** (grammar.h:445–447):

```inform6
Verb 'swing'
    * noun -> Swing
    * 'on' noun -> Swing;
```

**Default behavior:**

1. Prints `L__M(##Swing, 1, noun)`.

**Checks performed:** None.

**World-model effects:** None (pure stub).

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Swing, 1)` | Always |

---
### SwitchOff

- **Routine:** `SwitchOffSub` (verblib.h:2604)
- **Meta:** No

**Grammar** (grammar.h:226,229,449,452,454,487,490,492):

```inform6
Verb 'close' 'cover' 'shut'
    * 'off' noun                        -> SwitchOff;

Verb 'switch'
    * noun 'off'                        -> SwitchOff
    * 'off' noun                        -> SwitchOff;

Verb 'turn' 'rotate' 'screw' 'twist' 'unscrew'
    * noun 'off'                        -> SwitchOff
    * 'off' noun                        -> SwitchOff;
```

**Default behavior:**

1. Calls `ObjectIsUntouchable(noun)` — if untouchable, stops.
2. If `noun` hasnt `switchable`, prints `L__M(##SwitchOff, 1, noun)` and stops.
3. If `noun` hasnt `on`, prints `L__M(##SwitchOff, 2, noun)` and stops.
4. Gives `noun` `~on`.
5. Calls `AfterRoutines()`. If not intercepted, prints `L__M(##SwitchOff, 3, noun)`.

**Checks performed:** `switchable`, `on` attributes.

**World-model effects:** Gives `noun` `~on`.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##SwitchOff, 1)` | `noun` is not switchable |
| `L__M(##SwitchOff, 2)` | `noun` is already off |
| `L__M(##SwitchOff, 3)` | Successfully switched off |

**Related actions:** SwitchOn

---

### SwitchOn

- **Routine:** `SwitchOnSub` (verblib.h:2594)
- **Meta:** No

**Grammar** (grammar.h:449–451,453,487,489,491):

```inform6
Verb 'switch'
    * noun                              -> SwitchOn
    * noun 'on'                         -> SwitchOn
    * 'on' noun                         -> SwitchOn;

Verb 'turn' 'rotate' 'screw' 'twist' 'unscrew'
    * noun 'on'                         -> SwitchOn
    * 'on' noun                         -> SwitchOn;
```

**Default behavior:**

1. Calls `ObjectIsUntouchable(noun)` — if untouchable, stops.
2. If `noun` hasnt `switchable`, prints `L__M(##SwitchOn, 1, noun)` and stops.
3. If `noun` has `on`, prints `L__M(##SwitchOn, 2, noun)` and stops.
4. Gives `noun` `on`.
5. Calls `AfterRoutines()`. If not intercepted, prints `L__M(##SwitchOn, 3, noun)`.

**Checks performed:** `switchable`, `on` attributes.

**World-model effects:** Gives `noun` `on`.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##SwitchOn, 1)` | `noun` is not switchable |
| `L__M(##SwitchOn, 2)` | `noun` is already on |
| `L__M(##SwitchOn, 3)` | Successfully switched on |

**Related actions:** SwitchOff

---

### Take

- **Routine:** `TakeSub` (verblib.h:1858)
- **Meta:** No

**Grammar** (grammar.h:217–218,280,282,357–359,361–363,456–457):

```inform6
Verb 'carry'
    * multi                             -> Take;

Verb 'get'
    * multi                             -> Take;

Verb 'peel'
    * noun                              -> Take
    * 'off' noun                        -> Take;

Verb 'pick'
    * 'up' multi                        -> Take
    * multi 'up'                        -> Take;

Verb 'take'
    * multi                             -> Take;
```

**Default behavior:**

1. If `noun` is already in `actor`:
   - If `noun` has `worn`, redirects to `<<Disrobe noun, actor>>`.
   - If `onotheld_mode` is set, sets `notheld_mode = onotheld_mode` and returns.
   - Otherwise prints `L__M(##Take, 1, noun)` and stops.
2. Calls `AttemptToTakeObject()`. If unsuccessful, stops.
3. Calls `AfterRoutines()`. If not intercepted, prints a success message (suppressed if `keep_silent` is set).

**Checks performed:** `worn` attribute; parent relationships; `notheld_mode`/`onotheld_mode`.

**World-model effects:** Moves `noun` to `actor` (via `AttemptToTakeObject`); gives `noun` `moved`; gives `noun` `~concealed`.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Take, 1)` | `noun` is already held by `actor` |

> **Note:** `AttemptToTakeObject()` (verblib.h:1811) handles the complex taking logic including capacity checks, the `Receive` fake action, and most Take messages (`L__M(##Take, 2–13)`).

**Related actions:** Drop, Remove, PutOn, Insert, Give

---

### Taste

- **Routine:** `TasteSub` (verblib.h:2838)
- **Meta:** No

**Grammar** (grammar.h:462–463):

```inform6
Verb 'taste'
    * noun                              -> Taste;
```

**Default behavior:**

1. Calls `ObjectIsUntouchable(noun)` — if untouchable, stops.
2. If `noun` has `animate`, prints `L__M(##Taste, 2, noun)` and stops.
3. Otherwise prints `L__M(##Taste, 1, noun)`.

**Checks performed:** `animate` attribute.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Taste, 1)` | `noun` is inanimate |
| `L__M(##Taste, 2)` | `noun` is animate |

**Related actions:** —

---

### Tell

- **Routine:** `TellSub` (verblib.h:2844)
- **Meta:** No

**Grammar** (grammar.h:465–466):

```inform6
Verb 'tell'
    * creature 'about' topic           -> Tell;
```

**Default behavior:**

1. If `noun == actor`, prints `L__M(##Tell, 1, noun)` and stops.
2. Calls `RunLife(noun, ##Tell)`. If `RunLife` returns false, prints `L__M(##Tell, 2, noun)`.

**Checks performed:** Self-reference.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Tell, 1)` | Talking to yourself |
| `L__M(##Tell, 2)` | `noun`'s `life` routine did not handle the action |

**Related actions:** Ask, Answer

---

### Think

- **Routine:** `ThinkSub` (verblib.h:2850)
- **Meta:** No

**Grammar** (grammar.h:469–470):

```inform6
Verb 'think'
    *                                   -> Think;
```

**Default behavior:**

1. Prints `L__M(##Think, 1, noun)`.

**Checks performed:** None.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Think, 1)` | Always (stub response) |

**Related actions:** —

---

### ThrowAt

- **Routine:** `ThrowAtSub` (verblib.h:2852)
- **Meta:** No

**Grammar** (grammar.h:472–474):

```inform6
Verb 'throw'
    * noun                              -> ThrowAt
    * held 'at'/'against'/'on'/'onto' noun -> ThrowAt;
```

**Default behavior:**

1. If `second == nothing` or `second <= 1`, prints `L__M(##ThrowAt, 1, noun)` and stops.
2. Calls `ObjectIsUntouchable(second)` — if untouchable, stops.
3. If `second` hasnt `animate`:
   - Temporarily sets `action` to `##ThrownAt` and calls `RunRoutines(second, before)`.
   - If intercepted, restores `action` to `##ThrowAt` and returns.
   - Restores `action` to `##ThrowAt`.
4. If `noun` has `worn`, calls `ImplicitDisrobe()`.
5. Calls `RunLife(second, ##ThrowAt)`. If `RunLife` returns false, prints `L__M(##ThrowAt, 2, noun)`.

**Checks performed:** `animate` attribute on `second`; `worn` attribute on `noun`; validity of `second`.

**World-model effects:** May call `ImplicitDisrobe()` on `noun`.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##ThrowAt, 1)` | No valid target (`second` is `nothing` or ≤ 1) |
| `L__M(##ThrowAt, 2)` | Target's `life` routine did not handle the action |

> **Note:** Uses the `ThrownAt` fake action to let inanimate targets intercept the throw via their `before` property.

**Related actions:** ThrownAt (fake), Give

---

### Tie

- **Routine:** `TieSub` (verblib.h:2866)
- **Meta:** No

**Grammar** (grammar.h:476–478):

```inform6
Verb 'tie' 'attach' 'connect' 'fasten' 'fix'
    * noun                              -> Tie
    * noun 'to' noun                    -> Tie;
```

**Default behavior:**

1. If `noun` has `animate`, prints `L__M(##Tie, 2, noun)` and stops.
2. Otherwise prints `L__M(##Tie, 1, noun)`.

**Checks performed:** `animate` attribute.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Tie, 1)` | `noun` is inanimate (stub: impractical) |
| `L__M(##Tie, 2)` | `noun` is animate (stub: impractical) |

**Related actions:** —

---

### Touch

- **Routine:** `TouchSub` (verblib.h:2871)
- **Meta:** No

**Grammar** (grammar.h:480–481):

```inform6
Verb 'touch' 'feel' 'fondle' 'grope'
    * noun                              -> Touch;
```

**Default behavior:**

1. If `noun == actor`, prints `L__M(##Touch, 3, noun)` and stops.
2. Calls `ObjectIsUntouchable(noun)` — if untouchable, stops.
3. If `noun` has `animate`, prints `L__M(##Touch, 1, noun)` and stops.
4. Otherwise prints `L__M(##Touch, 2, noun)`.

**Checks performed:** `animate` attribute; self-reference.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Touch, 1)` | `noun` is animate |
| `L__M(##Touch, 2)` | `noun` is inanimate |
| `L__M(##Touch, 3)` | Touching yourself |

**Related actions:** Kiss, Squeeze

---

### Transfer

- **Routine:** `TransferSub` (verblib.h:1984)
- **Meta:** No

**Grammar** (grammar.h:376,379,483–484):

```inform6
Verb 'push' 'clear' 'move' 'press' 'shift'
    * noun 'to' noun                    -> Transfer;

Verb 'transfer'
    * noun 'to' noun=NotPlayer          -> Transfer;
```

**Default behavior:**

1. If `noun` is not in `actor`, calls `AttemptToTakeObject()`. If it fails, prints `L__M(##Take, 5, noun)` and stops.
2. If `second == actor`, redirects to `<<Drop noun, actor>>`.
3. If `second` has `supporter`, redirects to `<<PutOn noun second, actor>>`.
4. Otherwise redirects to `<<Insert noun second, actor>>`.

**Checks performed:** Parent relationships; `supporter` attribute on `second`.

**World-model effects:** Delegates to Drop, PutOn, or Insert (effects depend on the redirected action).

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Transfer, 1)` | Transfer not possible |
| `L__M(##Take, 5)` | Failed to take `noun` first |

**Related actions:** PutOn, Insert, Drop, Give

---

### Turn

- **Routine:** `TurnSub` (verblib.h:2878)
- **Meta:** No

**Grammar** (grammar.h:487–488):

```inform6
Verb 'turn' 'rotate' 'screw' 'twist' 'unscrew'
    * noun                              -> Turn;
```

**Default behavior:**

1. Calls `ObjectIsUntouchable(noun)` — if untouchable, stops.
2. If `noun == player` or `noun == actor`, prints `L__M(##Turn, 1, noun)` and stops.
3. If `noun` has `static`, prints `L__M(##Turn, 5, noun)` and stops.
4. If `noun` has `scenery`, prints `L__M(##Turn, 2, noun)` and stops.
5. If `noun` has `animate`, prints `L__M(##Turn, 5, noun)` and stops.
6. Otherwise prints `L__M(##Turn, 4, noun)`.

**Checks performed:** `static`, `scenery`, `animate` attributes; self-reference.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Turn, 1)` | Trying to turn yourself |
| `L__M(##Turn, 2)` | `noun` is scenery |
| `L__M(##Turn, 4)` | Default stub response |
| `L__M(##Turn, 5)` | `noun` is static or animate |

**Related actions:** Pull, Push

---

### Unlock

- **Routine:** `UnlockSub` (verblib.h:2564)
- **Meta:** No

**Grammar** (grammar.h:353,355,368–371,494–495):

```inform6
Verb 'open' 'uncover' 'undo' 'unwrap'
    * noun 'with' held                  -> Unlock;

Verb 'pry' 'prise' 'prize' 'lever' 'jemmy' 'force'
    * noun 'with' held                  -> Unlock
    * 'apart'/'open' noun 'with' held   -> Unlock
    * noun 'apart'/'open' 'with' held   -> Unlock;

Verb 'unlock'
    * noun 'with' held                  -> Unlock;
```

**Default behavior:**

1. Calls `ObjectIsUntouchable(noun)` — if untouchable, stops.
2. If `noun` hasnt `lockable`, prints `L__M(##Unlock, 1, noun)` and stops.
3. If `noun` hasnt `locked`, prints `L__M(##Unlock, 2, noun)` and stops.
4. Checks the `with_key` property:
   - If it is an Object, `second` must match it; otherwise prints `L__M(##Unlock, 3, second)` and stops.
   - If it is a Routine, calls it to validate the key.
5. Gives `noun` `~locked`.
6. Calls `AfterRoutines()`. If not intercepted, prints a success message.

**Checks performed:** `lockable`, `locked` attributes; `with_key` property.

**World-model effects:** Gives `noun` `~locked`.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Unlock, 1)` | `noun` is not lockable |
| `L__M(##Unlock, 2)` | `noun` is already unlocked |
| `L__M(##Unlock, 3)` | Wrong key |

**Related actions:** Lock, Open, Close

---

### VagueGo

- **Routine:** `VagueGoSub` (verblib.h:2163)
- **Meta:** No

**Grammar** (grammar.h:292–293,327–328):

```inform6
Verb 'go' 'run' 'walk'
    *                                   -> VagueGo;

Verb 'leave'
    *                                   -> VagueGo;
```

**Default behavior:**

1. Prints `L__M(##VagueGo)`.

**Checks performed:** None.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##VagueGo)` | Always (no direction specified) |

> **Note:** Triggered when the player types "go" without specifying a direction.

**Related actions:** Go, GoIn

---

### Wait

- **Routine:** `WaitSub` (verblib.h:2888)
- **Meta:** No

**Grammar** (grammar.h:497–498):

```inform6
Verb 'wait' 'z//'
    *                                   -> Wait;
```

**Default behavior:**

1. Calls `AfterRoutines()` first (allowing `each_turn` processing to be intercepted).
2. If not intercepted, prints `L__M(##Wait, 1, noun)`.

**Checks performed:** None.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Wait, 1)` | Default "Time passes" message |

> **Note:** Calling `AfterRoutines()` before the default message is unusual — it allows `location.after` or other after handlers to replace the "Time passes" message.

**Related actions:** —

---

### Wake

- **Routine:** `WakeSub` (verblib.h:2893)
- **Meta:** No

**Grammar** (grammar.h:500–502):

```inform6
Verb 'wake' 'awake' 'awaken'
    *                                   -> Wake
    * 'up'                              -> Wake;
```

**Default behavior:**

1. Prints `L__M(##Wake, 1, noun)`.

**Checks performed:** None.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Wake, 1)` | Always (stub: waking self) |

**Related actions:** WakeOther

---

### WakeOther

- **Routine:** `WakeOtherSub` (verblib.h:2895)
- **Meta:** No

**Grammar** (grammar.h:500,503–505):

```inform6
Verb 'wake' 'awake' 'awaken'
    * creature                          -> WakeOther
    * creature 'up'                     -> WakeOther
    * 'up' creature                     -> WakeOther;
```

**Default behavior:**

1. Calls `ObjectIsUntouchable(noun)` — if untouchable, stops.
2. Calls `RunLife(noun, ##WakeOther)`. If `RunLife` returns false, prints `L__M(##WakeOther, 1, noun)`.

**Checks performed:** None (beyond touchability).

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##WakeOther, 1)` | `noun`'s `life` routine did not handle the action |

**Related actions:** Wake

---

### Wave

- **Routine:** `WaveSub` (verblib.h:2901)
- **Meta:** No

**Grammar** (grammar.h:507,509–510):

```inform6
Verb 'wave'
    * noun                              -> Wave
    * noun 'at' noun                    -> Wave;
```

**Default behavior:**

1. If `noun == player` or `noun == actor`, prints `L__M(##Wave, 2, noun, second)` and stops.
2. If `noun` is not in `actor`, calls `ImplicitTake()`.
3. If `second` exists, prints `L__M(##Wave, 3, noun, second)`.
4. Otherwise prints `L__M(##Wave, 1, noun)`.

**Checks performed:** Self-reference; parent relationships.

**World-model effects:** None (may call `ImplicitTake()`).

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Wave, 1)` | Waving `noun` (no target) |
| `L__M(##Wave, 2)` | Waving yourself |
| `L__M(##Wave, 3)` | Waving `noun` at `second` |

**Related actions:** WaveHands

---

### WaveHands

- **Routine:** `WaveHandsSub` (verblib.h:2908)
- **Meta:** No

**Grammar** (grammar.h:507–508,511):

```inform6
Verb 'wave'
    *                                   -> WaveHands
    * 'at' noun                         -> WaveHands;
```

**Default behavior:**

1. If `noun` is specified (non-zero), prints `L__M(##WaveHands, 2, noun)`.
2. Otherwise prints `L__M(##WaveHands, 1, noun)`.

**Checks performed:** None.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##WaveHands, 1)` | Waving hands (no target) |
| `L__M(##WaveHands, 2)` | Waving hands at `noun` |

**Related actions:** Wave

---

### Wear

- **Routine:** `WearSub` (verblib.h:2653)
- **Meta:** No

**Grammar** (grammar.h:381,384,513–514):

```inform6
Verb 'put'
    * 'on' multiheld                    -> Wear;

Verb 'wear' 'don'
    * multiheld                         -> Wear;
```

**Default behavior:**

1. Calls `ObjectIsUntouchable(noun)` — if untouchable, stops.
2. If `noun` hasnt `clothing`, prints `L__M(##Wear, 1, noun)` and stops.
3. If `noun` is not in `actor`:
   - If `IndirectlyContains(actor, noun)` (e.g., in a bag), checks whether the parent is a `container`/`supporter`.
   - If `no_implicit_actions` is set, prints an appropriate message and stops.
   - Otherwise calls `AttemptToTakeObject()`.
   - If `noun` is still not in `actor`, prints `L__M(##Wear, 5)` or `L__M(##Miscellany, 58)` and stops.
4. If `noun` is not in `actor` at all, prints `L__M(##Wear, 2)` and stops.
5. If `noun` has `worn`, prints `L__M(##Wear, 3, noun)` and stops.
6. Gives `noun` `worn`.
7. Calls `AfterRoutines()`. If not intercepted, prints `L__M(##Wear, 4, noun)`.

**Checks performed:** `clothing`, `worn` attributes; parent relationships; `no_implicit_actions`.

**World-model effects:** Gives `noun` `worn`; may call `AttemptToTakeObject()`.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Wear, 1)` | `noun` is not clothing |
| `L__M(##Wear, 2)` | `noun` is not held |
| `L__M(##Wear, 3)` | `noun` is already worn |
| `L__M(##Wear, 4)` | Successfully put on |
| `L__M(##Wear, 5)` | Failed to implicitly take `noun` |
| `L__M(##Miscellany, 58)` | Implicit take failure (alternate) |

**Related actions:** Disrobe, Take

---

### Yes

- **Routine:** `YesSub` (verblib.h:2912)
- **Meta:** No

**Grammar** (grammar.h:516–517):

```inform6
Verb 'yes' 'y//'
    *                                   -> Yes;
```

**Default behavior:**

1. Prints `L__M(##Yes)`.

**Checks performed:** None.

**World-model effects:** None.

**Library messages:**

| Message | Condition |
|---------|-----------|
| `L__M(##Yes)` | Always (stub response) |

**Related actions:** No
---

## §I.4 Meta Actions

Meta actions are game-control commands that do not consume a game turn. They are
declared with `Verb meta` in `grammar.h`. The action processing pipeline skips
`BeforeRoutines()` for meta actions — the `*Sub` routine is called directly.

This section covers standard meta actions (§I.4.1) and debug-only meta actions
(§I.4.2).

### §I.4.1 Standard Meta Actions

---

#### CommandsOff

- **Routine:** `CommandsOffSub` (`verblib.h`:1203 [Z] / 1314 [Glulx])
- **Meta:** Yes
- **Grammar:** `Verb meta 'recording', * 'off' -> CommandsOff`
  (`grammar.h`:51,54)
- **Default Behavior:** [Z-machine] Calls `@output_stream -4`. Sets
  `xcommsdir = 0`. Prints `L__M(##CommandsOff, 1)`. [Glulx] If
  `gg_commandstr` is active, closes stream. If `gg_command_reading`, clears
  it. Prints `L__M(##CommandsOff, 1)` or `L__M(##CommandsOff, 2)`.
- **Messages:** `L__M(##CommandsOff, 1)`, `L__M(##CommandsOff, 2)`
- **Related Actions:** `CommandsOn`, `CommandsRead`

---

#### CommandsOn

- **Routine:** `CommandsOnSub` (`verblib.h`:1197 [Z] / 1300 [Glulx])
- **Meta:** Yes
- **Grammar:** `Verb meta 'recording', * -> CommandsOn, * 'on' -> CommandsOn`
  (`grammar.h`:51–53)
- **Default Behavior:** [Z-machine] Calls `@output_stream 4`. Sets
  `xcommsdir = 1`. Prints `L__M(##CommandsOn, 1)`. [Glulx] Uses Glk file
  dialogs to open command recording file. Prints `L__M(##CommandsOn, 1–4)`.
- **Messages:** `L__M(##CommandsOn, 1–4)`
- **Related Actions:** `CommandsOff`, `CommandsRead`

---

#### CommandsRead

- **Routine:** `CommandsReadSub` (`verblib.h`:1209 [Z] / 1323 [Glulx])
- **Meta:** Yes
- **Grammar:** `Verb meta 'replay', * -> CommandsRead` (`grammar.h`:56–57)
- **Default Behavior:** [Z-machine] Calls `@input_stream 1`. Sets
  `xcommsdir = 2`. Prints `L__M(##CommandsRead, 1)`. [Glulx] Uses Glk file
  dialogs to open command replay file. Sets `gg_command_reading = true`.
  Prints `L__M(##CommandsRead, 1–5)`.
- **Messages:** `L__M(##CommandsRead, 1–5)`
- **Related Actions:** `CommandsOn`, `CommandsOff`

---

#### FullScore

- **Routine:** `FullScoreSub` (`verblib.h`:1481)
- **Meta:** Yes
- **Grammar:** `Verb meta 'fullscore' 'full', * -> FullScore, * 'score' -> FullScore`
  (`grammar.h`:71–73)
- **Default Behavior:** Prints `L__M(##FullScore, 1)`. If `TASKS_PROVIDED`
  is defined, iterates through tasks and prints completed ones via
  `TaskScore()`/`PrintTaskName()`. Prints `things_score` and
  `places_score`. Prints total via `ScoreSub()`. `L__M(##FullScore, 2)`
  header for tasks, `L__M(##FullScore, 3)` for things/places,
  `L__M(##FullScore, 4)` for total.
- **Messages:** `L__M(##FullScore, 1–4)`
- **Related Actions:** `Score`

---

#### LMode1

- **Routine:** `LMode1Sub` (`verblib.h`:2397)
- **Meta:** Yes
- **Grammar:** `Verb meta 'brief', * -> LMode1` (`grammar.h`:28–29)
- **Default Behavior:** Sets `lookmode = 1`. Prints `L__M(##LMode1)`.
- **Effects:** `lookmode = 1` (brief — full descriptions only on first visit)
- **Messages:** `L__M(##LMode1)`
- **Related Actions:** `LMode2`, `LMode3`, `LModeNormal`

---

#### LMode2

- **Routine:** `LMode2Sub` (`verblib.h`:2399)
- **Meta:** Yes
- **Grammar:** `Verb meta 'verbose' 'long', * -> LMode2` (`grammar.h`:31–32)
- **Default Behavior:** Sets `lookmode = 2`. Prints `L__M(##LMode2)`.
- **Effects:** `lookmode = 2` (verbose — always print full descriptions)
- **Messages:** `L__M(##LMode2)`
- **Related Actions:** `LMode1`, `LMode3`, `LModeNormal`

---

#### LMode3

- **Routine:** `LMode3Sub` (`verblib.h`:2401)
- **Meta:** Yes
- **Grammar:** `Verb meta 'superbrief' 'short', * -> LMode3`
  (`grammar.h`:34–35)
- **Default Behavior:** Sets `lookmode = 3`. Prints `L__M(##LMode3)`.
- **Effects:** `lookmode = 3` (superbrief — never print full descriptions
  automatically)
- **Messages:** `L__M(##LMode3)`
- **Related Actions:** `LMode1`, `LMode2`, `LModeNormal`

---

#### LModeNormal

- **Routine:** `LModeNormalSub` (`verblib.h`:2403)
- **Meta:** Yes
- **Grammar:** `Verb meta 'normal', * -> LModeNormal` (`grammar.h`:37–38)
- **Default Behavior:** Checks `initial_lookmode`. If 1, redirects to
  `<<LMode1>>`. If 3, redirects to `<<LMode3>>`. Otherwise redirects to
  `<<LMode2>>`.
- **Effects:** Resets `lookmode` to `initial_lookmode` value
- **Related Actions:** `LMode1`, `LMode2`, `LMode3`

---

#### NotifyOff

- **Routine:** `NotifyOffSub` (`verblib.h`:1340)
- **Meta:** Yes
- **Grammar:** `Verb meta 'notify', * 'off' -> NotifyOff`
  (`grammar.h`:40,43)
- **Default Behavior:** Sets `notify_mode = false`. Prints
  `L__M(##NotifyOff)`.
- **Effects:** `notify_mode = false`
- **Messages:** `L__M(##NotifyOff)`
- **Related Actions:** `NotifyOn`

---

#### NotifyOn

- **Routine:** `NotifyOnSub` (`verblib.h`:1339)
- **Meta:** Yes
- **Grammar:** `Verb meta 'notify', * -> NotifyOn, * 'on' -> NotifyOn`
  (`grammar.h`:40–42)
- **Default Behavior:** Sets `notify_mode = true`. Prints
  `L__M(##NotifyOn)`.
- **Effects:** `notify_mode = true`
- **Messages:** `L__M(##NotifyOn)`
- **Related Actions:** `NotifyOff`

---

#### Objects

- **Routine:** `ObjectsSub` (`verblib.h`:46); `Objects1Sub` (`verblib.h`:1388)
- **Meta:** Yes
- **Grammar:** `Verb meta 'objects', * -> Objects` (`grammar.h`:90–91)
- **Default Behavior:** `ObjectsSub` delegates to `Objects1Sub`. Lists all
  objects that have been moved, showing current locations. For each moved
  object, shows where it is (carried, worn, in/on something, in a room,
  etc.).
- **Checks:** `moved`, `worn`, `animate`, `visited`, `container`,
  `supporter`, `enterable` attributes
- **Messages:** `L__M(##Objects, 1–10)`
- **Related Actions:** `ObjectsTall`, `ObjectsWide`, `Places`

---

#### ObjectsTall

- **Routine:** `ObjectsTallSub` (`verblib.h`:47); `Objects1TallSub`
  (`verblib.h`:1378)
- **Meta:** Yes
- **Grammar:** `Verb meta 'objects', * 'tall' -> ObjectsTall`
  (`grammar.h`:90,92)
- **Default Behavior:** Sets
  `objects_style = NEWLINE_BIT+INDENT_BIT+FULLINV_BIT`. Calls `<Objects>`.
- **Related Actions:** `Objects`, `ObjectsWide`

---

#### ObjectsWide

- **Routine:** `ObjectsWideSub` (`verblib.h`:48); `Objects1WideSub`
  (`verblib.h`:1383)
- **Meta:** Yes
- **Grammar:** `Verb meta 'objects', * 'wide' -> ObjectsWide`
  (`grammar.h`:90,93)
- **Default Behavior:** Sets `objects_style = ENGLISH_BIT+FULLINV_BIT`. Calls
  `<Objects>`.
- **Related Actions:** `Objects`, `ObjectsTall`

---

#### Places

- **Routine:** `PlacesSub` (`verblib.h`:49); `Places1Sub` (`verblib.h`:1352)
- **Meta:** Yes
- **Grammar:** `Verb meta 'places', * -> Places` (`grammar.h`:94–95)
- **Default Behavior:** `PlacesSub` delegates to `Places1Sub`. Lists all
  rooms with `visited` attribute. Shows count at end.
- **Checks:** `visited` attribute
- **Messages:** `L__M(##Places, 1–3)`
- **Related Actions:** `PlacesTall`, `PlacesWide`, `Objects`

---

#### PlacesTall

- **Routine:** `PlacesTallSub` (`verblib.h`:50); `Places1TallSub`
  (`verblib.h`:1342)
- **Meta:** Yes
- **Grammar:** `Verb meta 'places', * 'tall' -> PlacesTall`
  (`grammar.h`:94,96)
- **Default Behavior:** Sets
  `places_style = NEWLINE_BIT+INDENT_BIT+FULLINV_BIT`. Calls `<Places>`.
- **Related Actions:** `Places`, `PlacesWide`

---

#### PlacesWide

- **Routine:** `PlacesWideSub` (`verblib.h`:51); `Places1WideSub`
  (`verblib.h`:1347)
- **Meta:** Yes
- **Grammar:** `Verb meta 'places', * 'wide' -> PlacesWide`
  (`grammar.h`:94,97)
- **Default Behavior:** Sets `places_style = ENGLISH_BIT+FULLINV_BIT`. Calls
  `<Places>`.
- **Related Actions:** `Places`, `PlacesTall`

---

#### Pronouns

- **Routine:** `PronounsSub` (`parser.h`:4997)
- **Meta:** Yes
- **Grammar:** `Verb meta 'pronouns' 'nouns', * -> Pronouns`
  (`grammar.h`:45–46)
- **Default Behavior:** Iterates through `LanguagePronouns` table and prints
  current pronoun bindings. Shows what "it", "him", "her", "them" refer to.
  Also shows "me" → `player`.
- **Notes:** Routine is in `parser.h`, not `verblib.h`.

---

#### Quit

- **Routine:** `QuitSub` (`verblib.h`:1122 [Z] / 1217 [Glulx])
- **Meta:** Yes
- **Grammar:** `Verb meta 'quit' 'q//' 'die', * -> Quit`
  (`grammar.h`:48–49)
- **Default Behavior:** Prints `L__M(##Quit, 2)` (confirmation prompt).
  Calls `YesOrNo()`. If yes, calls `quit` opcode.
- **Messages:** `L__M(##Quit, 2)`

---

#### Restart

- **Routine:** `RestartSub` (`verblib.h`:1127 [Z] / 1222 [Glulx])
- **Meta:** Yes
- **Grammar:** `Verb meta 'restart', * -> Restart` (`grammar.h`:59–60)
- **Default Behavior:** Prints `L__M(##Restart, 1)` (confirmation). If
  `YesOrNo()`, calls `@restart`. If restart fails, prints
  `L__M(##Restart, 2)`.
- **Messages:** `L__M(##Restart, 1)`, `L__M(##Restart, 2)`

---

#### Restore

- **Routine:** `RestoreSub` (`verblib.h`:1132 [Z] / 1227 [Glulx])
- **Meta:** Yes
- **Grammar:** `Verb meta 'restore', * -> Restore` (`grammar.h`:62–63)
- **Default Behavior:** [Z-machine] Calls `@restore` opcode. On failure
  prints `L__M(##Restore, 1)`. On success, calls `AfterRestore()`.
  [Glulx] Uses Glk file dialogs for file selection and `@restore` opcode.
- **Messages:** `L__M(##Restore, 1)`, `L__M(##Restore, 2)`
- **Notes:** Calls `AfterRestore()`,
  `LibraryExtensions.RunAll(ext_afterrestore)`.

---

#### Save

- **Routine:** `SaveSub` (`verblib.h`:1142 [Z] / 1240 [Glulx])
- **Meta:** Yes
- **Grammar:** `Verb meta 'save', * -> Save` (`grammar.h`:65–66)
- **Default Behavior:** [Z-machine] Calls `@save` opcode. On failure prints
  `L__M(##Save, 1)`. On success prints `L__M(##Save, 2)`. Also detects
  restore-after-save (prints `L__M(##Restore, 2)`). [Glulx] Uses Glk file
  dialogs. Similar save/restore detection.
- **Messages:** `L__M(##Save, 1)`, `L__M(##Save, 2)`,
  `L__M(##Restore, 2)`
- **Notes:** Calls `AfterSave()`, `AfterRestore()`, `RestoreColours()`,
  `LibraryExtensions.RunAll(ext_aftersave/ext_afterrestore)`.

---

#### Score

- **Routine:** `ScoreSub` (`verblib.h`:1445)
- **Meta:** Yes
- **Grammar:** `Verb meta 'score', * -> Score` (`grammar.h`:68–69)
- **Default Behavior:** If `NO_SCORE` defined, prints `L__M(##Score, 2)`.
  Otherwise prints `L__M(##Score, 1)`. Calls `PrintRank()` and
  `LibraryExtensions.RunAll(ext_printrank)`.
- **Messages:** `L__M(##Score, 1)`, `L__M(##Score, 2)`
- **Related Actions:** `FullScore`

---

#### ScriptOff

- **Routine:** `ScriptOffSub` (`verblib.h`:1188 [Z] / 1293 [Glulx])
- **Meta:** Yes
- **Grammar:** `Verb meta 'script' 'transcript' * 'off' -> ScriptOff`
  (`grammar.h`:75,78); `Verb meta 'noscript' 'unscript' * -> ScriptOff`
  (`grammar.h`:80–81)
- **Default Behavior:** [Z-machine] Checks `transcript_mode`. If not on,
  prints `L__M(##ScriptOff, 1)`. Otherwise calls `@output_stream -2`, sets
  `transcript_mode = false`, prints `L__M(##ScriptOff, 2)`. [Glulx] Checks
  `gg_scriptstr`. Closes stream. Prints `L__M(##ScriptOff, 1)` or
  `L__M(##ScriptOff, 2)`.
- **Messages:** `L__M(##ScriptOff, 1–3)`
- **Related Actions:** `ScriptOn`

---

#### ScriptOn

- **Routine:** `ScriptOnSub` (`verblib.h`:1179 [Z] / 1278 [Glulx])
- **Meta:** Yes
- **Grammar:** `Verb meta 'script' 'transcript' * -> ScriptOn, * 'on' -> ScriptOn`
  (`grammar.h`:75–77)
- **Default Behavior:** [Z-machine] Checks `transcript_mode` and
  `HDR_GAMEFLAGS`. If already on, prints `L__M(##ScriptOn, 1)`. Otherwise
  calls `@output_stream 2`, sets `transcript_mode = true`, prints
  `L__M(##ScriptOn, 2)` or `L__M(##ScriptOn, 3)`. [Glulx] Uses Glk file
  dialogs.
- **Messages:** `L__M(##ScriptOn, 1–3)`
- **Related Actions:** `ScriptOff`

---

#### Verify

- **Routine:** `VerifySub` (`verblib.h`:1170 [Z] / 1272 [Glulx])
- **Meta:** Yes
- **Grammar:** `Verb meta 'verify', * -> Verify` (`grammar.h`:83–84)
- **Default Behavior:** [Z-machine] Calls `@verify` opcode. Prints
  `L__M(##Verify, 1)` on success or `L__M(##Verify, 2)` on failure.
  [Glulx] Calls `@verify` opcode. Same messages.
- **Messages:** `L__M(##Verify, 1)`, `L__M(##Verify, 2)`

---

#### Version

- **Routine:** `VersionSub` (`verblib.h`:109)
- **Meta:** Yes
- **Grammar:** `Verb meta 'version', * -> Version` (`grammar.h`:86–87)
- **Default Behavior:** Calls `Banner()`. Prints interpreter and VM
  information. Prints library serial number and language version. Has
  Z-machine and Glulx specific output sections.
- **Notes:** Does not use `L__M()` — prints directly using internal text
  constants.

---

### §I.4.2 Debug Meta Actions

The following actions are only available when the game is compiled with the
`DEBUG` flag set. They provide interactive debugging tools.

---

#### ActionsOff

- **Routine:** `ActionsOffSub` (`verblib.h`:2949)
- **Meta:** Yes
- **Grammar:** `Verb meta 'actions' * 'off' -> ActionsOff`
  (`grammar.h`:105,108)
- **Default Behavior:** Clears `DEBUG_ACTIONS` from `debug_flag`. Prints
  `"[Actions listing off.]"`

---

#### ActionsOn

- **Routine:** `ActionsOnSub` (`verblib.h`:2944)
- **Meta:** Yes
- **Grammar:** `Verb meta 'actions' * -> ActionsOn, * 'on' -> ActionsOn`
  (`grammar.h`:105–107)
- **Default Behavior:** Sets `DEBUG_ACTIONS` in `debug_flag`. Prints
  `"[Actions listing on.]"`

---

#### ChangesOff

- **Routine:** `ChangesOffSub` (`verblib.h`:2967)
- **Meta:** Yes
- **Grammar:** `Verb meta 'changes' * 'off' -> ChangesOff`
  (`grammar.h`:110,113)
- **Default Behavior:** Clears `DEBUG_CHANGES` from `debug_flag`. Prints
  `"[Changes listing off.]"`
- **Notes:** Requires compiler >= 6.2 (`VN_1610`).

---

#### ChangesOn

- **Routine:** `ChangesOnSub` (`verblib.h`:2966)
- **Meta:** Yes
- **Grammar:** `Verb meta 'changes' * -> ChangesOn, * 'on' -> ChangesOn`
  (`grammar.h`:110–112)
- **Default Behavior:** Sets `DEBUG_CHANGES` in `debug_flag`. Prints
  `"[Changes listing on.]"`
- **Notes:** Requires compiler >= 6.2 (`VN_1610`).

---

#### GlkList

- **Routine:** `GlkListSub` (`verblib.h`:3062)
- **Meta:** Yes
- **Grammar:** `Verb meta 'glklist' * -> Glklist` (`grammar.h`:174–175)
- **Default Behavior:** Lists all Glk windows, streams, file references, and
  sound channels using Glk iteration functions.
- **Notes:** Only compiled for Glulx target.

---

#### GoNear

- **Routine:** `GoNearSub` (`verblib.h`:3044)
- **Meta:** Yes
- **Grammar:** `Verb meta 'gonear' * anynumber -> GoNear, * noun -> GoNear`
  (`grammar.h`:115–117)
- **Default Behavior:** Finds the room containing `noun` by walking up parent
  chain. Teleports player to that room via `PlayerTo()`.

---

#### Goto

- **Routine:** `GotoSub` (`verblib.h`:3039)
- **Meta:** Yes
- **Grammar:** `Verb meta 'goto' * anynumber -> Goto` (`grammar.h`:120–121)
- **Default Behavior:** Teleports player directly to `noun` via
  `PlayerTo(noun)`. Validates `noun` is an Object.

---

#### Predictable

- **Routine:** `PredictableSub` (`verblib.h`:2979 [Z] / 2986 [Glulx])
- **Meta:** Yes
- **Grammar:** `Verb meta 'random' * -> Predictable` (`grammar.h`:123–124)
- **Default Behavior:** Seeds random number generator to produce predictable
  sequence. [Z-machine] Uses `@random 0`. [Glulx] Uses `@setrandom 0`.

---

#### RoutinesOff

- **Routine:** `RoutinesOffSub` (`verblib.h`:2934)
- **Meta:** Yes
- **Grammar:** `Verb meta 'routines' 'messages' * 'off' -> RoutinesOff`
  (`grammar.h`:126,130)
- **Default Behavior:** Clears `DEBUG_MESSAGES` from `debug_flag`. Prints
  `"[Message listing off.]"`

---

#### RoutinesOn

- **Routine:** `RoutinesOnSub` (`verblib.h`:2929)
- **Meta:** Yes
- **Grammar:** `Verb meta 'routines' 'messages' * -> RoutinesOn, * 'on' -> RoutinesOn`
  (`grammar.h`:126–128)
- **Default Behavior:** Sets `DEBUG_MESSAGES` in `debug_flag`. Prints
  `"[Message listing on.]"`

---

#### RoutinesVerbose

- **Routine:** `RoutinesVerboseSub` (`verblib.h`:2939)
- **Meta:** Yes
- **Grammar:** `Verb meta 'routines' 'messages' * 'verbose' -> RoutinesVerbose`
  (`grammar.h`:126,129)
- **Default Behavior:** Sets `DEBUG_VERBOSE|DEBUG_MESSAGES` in `debug_flag`.
  Prints `"[Message listing on, verbose mode.]"`

---

#### Scope

- **Routine:** `ScopeSub` (`verblib.h`:3053)
- **Meta:** Yes
- **Grammar:** `Verb meta 'scope' * -> Scope, * anynumber -> Scope, * noun -> Scope`
  (`grammar.h`:132–135)
- **Default Behavior:** Calls `LoopOverScope()` to list all objects in scope
  of `noun` (or `player` if no noun). Prints count.

---

#### ShowDict

- **Routine:** `ShowDictSub` (`parser.h`:6133)
- **Meta:** Yes
- **Grammar:** `Verb meta 'showdict' 'dict' * -> ShowDict, * topic -> ShowDict`
  (`grammar.h`:137–139)
- **Default Behavior:** Dumps dictionary entries with flag information
  (noun/verb/meta/plural/etc.). Can show all or filter by topic.
- **Notes:** Routine is in `parser.h`.

---

#### ShowObj

- **Routine:** `ShowObjSub` (`parser.h`:6065)
- **Meta:** Yes
- **Grammar:** `Verb meta 'showobj' * -> ShowObj, * anynumber -> ShowObj, * multi -> ShowObj`
  (`grammar.h`:141–144)
- **Default Behavior:** Displays detailed object info: name, location, class,
  attributes, and property values.
- **Notes:** Routine is in `parser.h`.

---

#### ShowVerb

- **Routine:** `ShowVerbSub` (`parser.h`:6018 [Z] / 6039 [Glulx])
- **Meta:** Yes
- **Grammar:** `Verb meta 'showverb' * special -> ShowVerb`
  (`grammar.h`:146–147)
- **Default Behavior:** Looks up verb in grammar table and displays all
  grammar lines. Shows meta flag and all verb aliases.
- **Notes:** Routine is in `parser.h`. Has separate Z-machine and Glulx
  implementations.

---

#### TimersOff

- **Routine:** `TimersOffSub` (`verblib.h`:2959)
- **Meta:** Yes
- **Grammar:** `Verb meta 'timers' 'daemons' * 'off' -> TimersOff`
  (`grammar.h`:149,152)
- **Default Behavior:** Clears `DEBUG_TIMERS` from `debug_flag`. Prints
  `"[Timers listing off.]"`

---

#### TimersOn

- **Routine:** `TimersOnSub` (`verblib.h`:2954)
- **Meta:** Yes
- **Grammar:** `Verb meta 'timers' 'daemons' * -> TimersOn, * 'on' -> TimersOn`
  (`grammar.h`:149–151)
- **Default Behavior:** Sets `DEBUG_TIMERS` in `debug_flag`. Prints
  `"[Timers listing on.]"`

---

#### TraceLevel

- **Routine:** `TraceLevelSub` (`verblib.h`:2922)
- **Meta:** Yes
- **Grammar:** `Verb meta 'trace' * number -> TraceLevel`
  (`grammar.h`:154,156)
- **Default Behavior:** Sets `parser_trace = noun`. Prints
  `"[Parser tracing set to level N.]"`

---

#### TraceOff

- **Routine:** `TraceOffSub` (`verblib.h`:2927)
- **Meta:** Yes
- **Grammar:** `Verb meta 'trace' * 'off' -> TraceOff` (`grammar.h`:154,158)
- **Default Behavior:** Sets `parser_trace = 0`. Prints `"Trace off."`

---

#### TraceOn

- **Routine:** `TraceOnSub` (`verblib.h`:2920)
- **Meta:** Yes
- **Grammar:** `Verb meta 'trace' * -> TraceOn, * 'on' -> TraceOn`
  (`grammar.h`:154–155,157)
- **Default Behavior:** Sets `parser_trace = 1`. Prints `"[Trace on.]"`

---

#### XAbstract

- **Routine:** `XAbstractSub` (`verblib.h`:3011)
- **Meta:** Yes
- **Grammar:** `Verb meta 'abstract' * anynumber 'to' anynumber -> XAbstract, * noun 'to' noun -> XAbstract`
  (`grammar.h`:160–162)
- **Default Behavior:** Validates `noun` is an Object. Calls
  `XTestMove(noun, second)`. Moves `noun` to `second`.

---

#### XPurloin

- **Routine:** `XPurloinSub` (`verblib.h`:3005)
- **Meta:** Yes
- **Grammar:** `Verb meta 'purloin' * anynumber -> XPurloin, * multi -> XPurloin`
  (`grammar.h`:164–166)
- **Default Behavior:** Validates `noun` is an Object. Calls
  `XTestMove(noun, player)`. Moves `noun` to `player`. Gives `noun`
  `moved`, gives `noun` `~concealed`.
- **Effects:** `move noun to player`; give `moved`; give `~concealed`

---

#### XTree

- **Routine:** `XTreeSub` (`verblib.h`:3030)
- **Meta:** Yes
- **Grammar:** `Verb meta 'tree' * -> XTree, * anynumber -> XTree, * noun -> XTree`
  (`grammar.h`:168–171)
- **Default Behavior:** If no `noun`, prints full object tree from all
  parentless objects. If `noun` specified, prints subtree from `noun`.
---

## §I.5 Fake Actions

Fake actions are declared with the `Fake_Action` directive and have no grammar
lines, no `*Sub` handler routine, and no entry in the actions table. They exist
solely as constants that can be checked in `before`, `after`, `life`, and
`orders` property routines. The library generates fake actions internally to
notify objects of events that are not directly triggered by player input.

All fake actions are declared in `parser.h` (lines 151–166).

---

### LetGo

- **Declaration:** `Fake_Action LetGo;` (parser.h:151)

**Purpose:** Notifies a container or supporter that an object is being removed
from it.

**Generated by:** `AttemptToTakeObject()` in `verblib.h` (~line 1811) when
taking an object from a container or supporter. The container's `before`
property is called with `action` temporarily set to `##LetGo`.

**Where intercepted:** `before` property of the container/supporter object.

**Semantics:** When the player takes an object from inside a container or off a
supporter, the library temporarily sets `action` to `##LetGo` and calls
`RunRoutines(parent, before)`. If the parent's `before` routine returns true,
the take is blocked. `noun` is the object being removed. The `receive_action`
global is not relevant here.

---

### Receive

- **Declaration:** `Fake_Action Receive;` (parser.h:152)

**Purpose:** Notifies a container or supporter that an object is being placed
into or onto it.

**Generated by:** `InsertSub` and `PutOnSub` in `verblib.h` when inserting or
placing objects. The container/supporter's `before` property is called with
`action` temporarily set to `##Receive`.

**Where intercepted:** `before` property of the container/supporter object.

**Semantics:** When the player puts an object into a container (`Insert`) or
onto a supporter (`PutOn`), the library temporarily sets `action` to
`##Receive` and calls `RunRoutines(second, before)`. The global
`receive_action` is set to `##Insert` or `##PutOn` so the `before` routine can
distinguish which operation triggered it. If the `before` routine returns true,
the placement is blocked.

---

### ThrownAt

- **Declaration:** `Fake_Action ThrownAt;` (parser.h:153)

**Purpose:** Notifies an inanimate target that something is being thrown at it.

**Generated by:** `ThrowAtSub` in `verblib.h` (line 2852) when the target
(`second`) does not have the `animate` attribute.

**Where intercepted:** `before` property of the target (`second`) object.

**Semantics:** For animate targets, `ThrowAtSub` uses
`RunLife(second, ##ThrowAt)` instead. For inanimate targets, the library
temporarily sets `action` to `##ThrownAt` and calls
`RunRoutines(second, before)`. If not intercepted, prints
`L__M(##ThrowAt, 2)`.

---

### Order

- **Declaration:** `Fake_Action Order;` (parser.h:154)

**Purpose:** Represents a command given to an NPC (e.g., "Fred, take the
lamp").

**Generated by:** The parser when it detects `creature, command` syntax. The
NPC's `orders` property and `life` property are checked.

**Where intercepted:** `orders` property of the NPC, then `life` property with
`reason_code` set to `##Order`.

**Semantics:** When the player addresses an NPC with a command, the parser sets
`actor` to the NPC. The library calls `RunRoutines(actor, orders)` first. If
`orders` returns false, it calls `RunLife(actor, ##Order)`. The `action`,
`noun`, and `second` globals contain the parsed command. If neither handles it,
prints `L__M(##Order, 1)`.

---

### TheSame

- **Declaration:** `Fake_Action TheSame;` (parser.h:155)

**Purpose:** Allows objects to declare themselves identical for disambiguation
purposes.

**Generated by:** The parser during disambiguation when two objects have the
same name words.

**Where intercepted:** `before` property of one of the identical objects.

**Semantics:** When the parser finds two objects with identical name words and
cannot distinguish them, it temporarily sets `action` to `##TheSame`, `noun` to
the first object, and `second` to the second object, then calls
`RunRoutines(noun, before)`. If the routine returns `-1`, the objects are
treated as genuinely indistinguishable. If it returns `-2`, they are treated as
distinguishable. Any other return value is ignored.

---

### PluralFound

- **Declaration:** `Fake_Action PluralFound;` (parser.h:156)

**Purpose:** Signals to a `parse_name` routine that a plural word has been
matched.

**Generated by:** The parser when it encounters a word with the `plural`
dictionary flag.

**Where intercepted:** `parse_name` property routines.

**Semantics:** During parsing, if a dictionary word has the plural flag
(`//p`), the parser sets `parser_action` to `##PluralFound` before calling the
object's `parse_name` routine. The `parse_name` routine can check
`parser_action` to determine whether the match included a plural word.

---

### ListMiscellany

- **Declaration:** `Fake_Action ListMiscellany;` (parser.h:157)

**Purpose:** Used by the list-writing module to generate miscellaneous list
formatting messages.

**Generated by:** `WriteListFrom()` and related list-writing routines in
`verblib.h`.

**Where intercepted:** Via `L__M(##ListMiscellany, N)` calls — not directly
intercepted by game objects.

**Semantics:** This fake action serves as a message namespace for the
list-writing subsystem. `L__M(##ListMiscellany, 1–22)` provide various list
formatting phrases like "providing light", "which is open", "which is closed
and locked", etc.

---

### Miscellany

- **Declaration:** `Fake_Action Miscellany;` (parser.h:158)

**Purpose:** General-purpose message namespace for miscellaneous library
messages.

**Generated by:** Various library routines.

**Where intercepted:** Via `L__M(##Miscellany, N)` calls — not directly
intercepted by game objects.

**Semantics:** Acts as a catch-all message namespace. `L__M(##Miscellany,
1–76)` cover a wide range of messages including parser errors, disambiguation
prompts, implicit action reports, game-over messages, and other library
communications.

---

### RunTimeError

- **Declaration:** `Fake_Action RunTimeError;` (parser.h:159)

**Purpose:** Message namespace for runtime error diagnostics.

**Generated by:** `RT__Err()` veneer routine and various library error-checking
code.

**Where intercepted:** Via `L__M(##RunTimeError, N)` calls — can be customized
via `LibraryMessages`.

**Semantics:** `L__M(##RunTimeError, 1–20+)` provide runtime error messages
such as property access errors, object tree violations, and other diagnostic
messages. These are generated by the `RT__Err` veneer routine.

---

### Prompt

- **Declaration:** `Fake_Action Prompt;` (parser.h:160)

**Purpose:** Allows customization of the command prompt.

**Generated by:** The parser's main loop, just before reading player input.

**Where intercepted:** Via `L__M(##Prompt, 1)` call — customizable via
`LibraryMessages`.

**Semantics:** Each turn, before the parser reads input, it calls
`L__M(##Prompt, 1)` to print the command prompt (default: `>`). Games can
customize this by intercepting `##Prompt` in a `LibraryMessages` object.

---

### NotUnderstood

- **Declaration:** `Fake_Action NotUnderstood;` (parser.h:161)

**Purpose:** Notifies an NPC's `orders`/`life` routines that a command directed
at them was not understood.

**Generated by:** The parser when an NPC command cannot be parsed.

**Where intercepted:** `orders` and `life` properties of the addressed NPC.

**Semantics:** When the player gives an NPC a command that the parser cannot
match to any grammar line, the library generates `##NotUnderstood`. The NPC's
`orders` property is called first; if that returns false,
`RunLife(actor, ##NotUnderstood)` is called. The `consult_from` and
`consult_words` globals contain the unparsed text.

---

### Going

- **Declaration:** `Fake_Action Going;` (parser.h:162)

**Purpose:** Notifies objects that the player is about to leave a location.

**Generated by:** `GoSub` in `verblib.h` (line 2167) during the `Go` action
processing.

**Where intercepted:** `before` property of objects in the current location
(via `react_before`), `before` property of the location, `before` property of
door objects.

**Semantics:** During the `Go` action, after determining the destination but
before actually moving the player, the library temporarily sets `action` to
`##Going` and calls `RunRoutines` with the current location and relevant
objects. If intercepted (returns true), the movement is blocked. The `noun`
variable contains the direction object, and `second` contains the door (if
any). The `Going` fake action allows `before` rules to examine the pending
movement without having to handle the full complexity of the `Go` action's
direction lookup.

---

### Places

- **Declaration:** `Fake_Action Places;` (parser.h:165) — **conditional**

**Purpose:** When the game defines `NO_PLACES`, the `Places` action and its
grammar are not compiled. The fake action declaration prevents compiler errors
if game code references `##Places`.

**Generated by:** Not generated at runtime; exists only as a compile-time
safeguard.

**Where intercepted:** N/A under `NO_PLACES`. Under normal compilation
(without `NO_PLACES`), `Places` is a real meta action handled by `PlacesSub`.

**Semantics:** This declaration is conditional. It appears inside an
`#Ifdef NO_PLACES` block (parser.h:164). When `NO_PLACES` is defined, the
normal `Places` verb grammar and `PlacesSub` routine are omitted. Declaring
`Places` as a fake action ensures that any `##Places` references in game code
still compile, even though the action has no effect.

---

### Objects

- **Declaration:** `Fake_Action Objects;` (parser.h:166) — **conditional**

**Purpose:** Same as `Places` — prevents compiler errors when `NO_PLACES`
removes the real `Objects` action.

**Generated by:** Not generated at runtime; exists only as a compile-time
safeguard.

**Where intercepted:** N/A under `NO_PLACES`. Under normal compilation
(without `NO_PLACES`), `Objects` is a real meta action handled by `ObjectsSub`.

**Semantics:** This declaration is conditional, appearing inside the same
`#Ifdef NO_PLACES` block as `Places` (parser.h:164). When `NO_PLACES` is
defined, the normal `Objects` verb grammar and `ObjectsSub` routine are
omitted. Declaring `Objects` as a fake action ensures that any `##Objects`
references in game code still compile, even though the action has no effect.
---

## §I.6 Verb Synonym Quick-Reference Table

This table lists every verb word defined in `grammar.h` alphabetically, showing
which action(s) it maps to. Verb words marked with `//` are abbreviations (e.g.,
`'i//'` matches just the letter "i").

| Verb Word | Action(s) | Meta | Source |
|-----------|-----------|------|--------|
| `'abstract'` | `XAbstract` | ✓ debug | `grammar.h:160` |
| `'actions'` | `ActionsOn`, `ActionsOff` | ✓ debug | `grammar.h:105` |
| `'answer'` | `Answer` | | `grammar.h:189` |
| `'ask'` | `Ask`, `AskFor`, `AskTo` | | `grammar.h:192` |
| `'attach'` | `Tie` | | `grammar.h:476` |
| `'attack'` | `Attack` | | `grammar.h:198` |
| `'awake'` | `Wake`, `WakeOther` | | `grammar.h:500` |
| `'awaken'` | `Wake`, `WakeOther` | | `grammar.h:500` |
| `'blow'` | `Blow` | | `grammar.h:203` |
| `'bother'` | `Mild` | | `grammar.h:206` |
| `'break'` | `Attack` | | `grammar.h:198` |
| `'brief'` | `LMode1` | ✓ | `grammar.h:28` |
| `'burn'` | `Burn` | | `grammar.h:210` |
| `'buy'` | `Buy` | | `grammar.h:214` |
| `'carry'` | `Take`, `Remove`, `Inv` | | `grammar.h:217` |
| `'changes'` | `ChangesOn`, `ChangesOff` | ✓ debug | `grammar.h:110` |
| `'check'` | `Examine` | | `grammar.h:269` |
| `'chop'` | `Cut` | | `grammar.h:235` |
| `'clean'` | `Rub` | | `grammar.h:398` |
| `'clear'` | `Push`, `PushDir`, `Transfer` | | `grammar.h:376` |
| `'climb'` | `Climb` | | `grammar.h:222` |
| `'close'` | `Close`, `SwitchOff` | | `grammar.h:226` |
| `'connect'` | `Tie` | | `grammar.h:476` |
| `'consult'` | `Consult` | | `grammar.h:231` |
| `'cover'` | `Close`, `SwitchOff` | | `grammar.h:226` |
| `'crack'` | `Attack` | | `grammar.h:198` |
| `'cross'` | `Enter`, `GoIn` | | `grammar.h:265` |
| `'curses'` | `Mild` | | `grammar.h:206` |
| `'cut'` | `Cut` | | `grammar.h:235` |
| `'daemons'` | `TimersOn`, `TimersOff` | ✓ debug | `grammar.h:149` |
| `'damn'` | `Strong` | | `grammar.h:409` |
| `'darn'` | `Mild` | | `grammar.h:206` |
| `'describe'` | `Examine` | | `grammar.h:269` |
| `'destroy'` | `Attack` | | `grammar.h:198` |
| `'dict'` | `ShowDict` | ✓ debug | `grammar.h:137` |
| `'dig'` | `Dig` | | `grammar.h:238` |
| `'discard'` | `Drop`, `Insert`, `PutOn` | | `grammar.h:251` |
| `'disrobe'` | `Disrobe` | | `grammar.h:245` |
| `'display'` | `Show` | | `grammar.h:413` |
| `'dive'` | `Swim` | | `grammar.h:442` |
| `'doff'` | `Disrobe` | | `grammar.h:245` |
| `'don'` | `Wear` | | `grammar.h:513` |
| `'drag'` | `Pull` | | `grammar.h:373` |
| `'drat'` | `Mild` | | `grammar.h:206` |
| `'drink'` | `Drink` | | `grammar.h:248` |
| `'drop'` | `Drop`, `Insert`, `PutOn` | | `grammar.h:251` |
| `'dust'` | `Rub` | | `grammar.h:398` |
| `'eat'` | `Eat` | | `grammar.h:256` |
| `'embrace'` | `Kiss` | | `grammar.h:324` |
| `'empty'` | `Empty`, `EmptyT` | | `grammar.h:259` |
| `'enter'` | `Enter`, `GoIn` | | `grammar.h:265` |
| `'examine'` | `Examine` | | `grammar.h:269` |
| `'exit'` | `Exit` | | `grammar.h:272` |
| `'fasten'` | `Tie` | | `grammar.h:476` |
| `'feed'` | `Give` | | `grammar.h:287` |
| `'feel'` | `Touch` | | `grammar.h:480` |
| `'fight'` | `Attack` | | `grammar.h:198` |
| `'fill'` | `Fill` | | `grammar.h:276` |
| `'fix'` | `Tie` | | `grammar.h:476` |
| `'fondle'` | `Touch` | | `grammar.h:480` |
| `'force'` | `Unlock` | | `grammar.h:368` |
| `'fuck'` | `Strong` | | `grammar.h:409` |
| `'full'` | `FullScore` | ✓ | `grammar.h:71` |
| `'fullscore'` | `FullScore` | ✓ | `grammar.h:71` |
| `'get'` | `Take`, `Remove`, `GetOff`, `Enter`, `Inv` | | `grammar.h:280` |
| `'give'` | `Give` | | `grammar.h:287` |
| `'glklist'` | `Glklist` | ✓ debug | `grammar.h:174` |
| `'go'` | `Go`, `GoIn`, `VagueGo` | | `grammar.h:292` |
| `'gonear'` | `GoNear` | ✓ debug | `grammar.h:115` |
| `'goto'` | `Goto` | ✓ debug | `grammar.h:120` |
| `'grope'` | `Touch` | | `grammar.h:480` |
| `'hear'` | `Listen` | | `grammar.h:333` |
| `'hit'` | `Attack` | | `grammar.h:198` |
| `'hold'` | `Take`, `Remove`, `Inv` | | `grammar.h:300` |
| `'hop'` | `Jump`, `JumpIn`, `JumpOn`, `JumpOver` | | `grammar.h:316` |
| `'hug'` | `Kiss` | | `grammar.h:324` |
| `'i//'` | `Inv`, `InvTall`, `InvWide` | | `grammar.h:311` |
| `'in'` | `GoIn` | | `grammar.h:305` |
| `'insert'` | `Insert` | | `grammar.h:308` |
| `'inside'` | `GoIn` | | `grammar.h:305` |
| `'inv'` | `Inv`, `InvTall`, `InvWide` | | `grammar.h:311` |
| `'inventory'` | `Inv`, `InvTall`, `InvWide` | | `grammar.h:311` |
| `'jemmy'` | `Unlock` | | `grammar.h:368` |
| `'jump'` | `Jump`, `JumpIn`, `JumpOn`, `JumpOver` | | `grammar.h:316` |
| `'kill'` | `Attack` | | `grammar.h:198` |
| `'kiss'` | `Kiss` | | `grammar.h:324` |
| `'l//'` | `Look`, `Search`, `LookUnder` | | `grammar.h:341` |
| `'leave'` | `VagueGo`, `Exit` | | `grammar.h:327` |
| `'lever'` | `Unlock` | | `grammar.h:368` |
| `'lie'` | `Enter` | | `grammar.h:420` |
| `'light'` | `Burn` | | `grammar.h:210` |
| `'listen'` | `Listen` | | `grammar.h:333` |
| `'lock'` | `Lock` | | `grammar.h:338` |
| `'long'` | `LMode2` | ✓ | `grammar.h:31` |
| `'look'` | `Look`, `Search`, `LookUnder`, `Examine` | | `grammar.h:341` |
| `'messages'` | `RoutinesOn`, `RoutinesOff`, `RoutinesVerbose` | ✓ debug | `grammar.h:126` |
| `'move'` | `Push`, `PushDir`, `Transfer` | | `grammar.h:376` |
| `'murder'` | `Attack` | | `grammar.h:198` |
| `'nap'` | `Sleep` | | `grammar.h:424` |
| `'no'` | `No` | | `grammar.h:350` |
| `'normal'` | `LModeNormal` | ✓ | `grammar.h:37` |
| `'noscript'` | `ScriptOff` | ✓ | `grammar.h:80` |
| `'nouns'` | `Pronouns` | ✓ | `grammar.h:45` |
| `'notify'` | `NotifyOn`, `NotifyOff` | ✓ | `grammar.h:40` |
| `'objects'` | `Objects`, `ObjectsTall`, `ObjectsWide` | ✓ | `grammar.h:90` |
| `'offer'` | `Give` | | `grammar.h:287` |
| `'open'` | `Open`, `Unlock` | | `grammar.h:353` |
| `'out'` | `Exit` | | `grammar.h:272` |
| `'outside'` | `Exit` | | `grammar.h:272` |
| `'pay'` | `Give` | | `grammar.h:287` |
| `'peel'` | `Take` | | `grammar.h:357` |
| `'pick'` | `Take` | | `grammar.h:361` |
| `'places'` | `Places`, `PlacesTall`, `PlacesWide` | ✓ | `grammar.h:94` |
| `'polish'` | `Rub` | | `grammar.h:398` |
| `'pray'` | `Pray` | | `grammar.h:365` |
| `'present'` | `Show` | | `grammar.h:413` |
| `'press'` | `Push`, `PushDir`, `Transfer` | | `grammar.h:376` |
| `'prise'` | `Unlock` | | `grammar.h:368` |
| `'prize'` | `Unlock` | | `grammar.h:368` |
| `'pronouns'` | `Pronouns` | ✓ | `grammar.h:45` |
| `'prune'` | `Cut` | | `grammar.h:235` |
| `'pry'` | `Unlock` | | `grammar.h:368` |
| `'pull'` | `Pull` | | `grammar.h:373` |
| `'punch'` | `Attack` | | `grammar.h:198` |
| `'purchase'` | `Buy` | | `grammar.h:214` |
| `'purloin'` | `XPurloin` | ✓ debug | `grammar.h:164` |
| `'push'` | `Push`, `PushDir`, `Transfer` | | `grammar.h:376` |
| `'put'` | `Insert`, `PutOn`, `Wear`, `Drop` | | `grammar.h:381` |
| `'q//'` | `Quit` | ✓ | `grammar.h:48` |
| `'quit'` | `Quit` | ✓ | `grammar.h:48` |
| `'random'` | `Predictable` | ✓ debug | `grammar.h:123` |
| `'read'` | `Examine`, `Consult` | | `grammar.h:388` |
| `'recording'` | `CommandsOn`, `CommandsOff` | ✓ | `grammar.h:51` |
| `'remove'` | `Disrobe`, `Remove` | | `grammar.h:393` |
| `'replay'` | `CommandsRead` | ✓ | `grammar.h:56` |
| `'restart'` | `Restart` | ✓ | `grammar.h:59` |
| `'restore'` | `Restore` | ✓ | `grammar.h:62` |
| `'rotate'` | `Turn`, `SwitchOn`, `SwitchOff` | | `grammar.h:487` |
| `'routines'` | `RoutinesOn`, `RoutinesOff`, `RoutinesVerbose` | ✓ debug | `grammar.h:126` |
| `'rub'` | `Rub` | | `grammar.h:398` |
| `'run'` | `Go`, `GoIn`, `VagueGo` | | `grammar.h:292` |
| `'save'` | `Save` | ✓ | `grammar.h:65` |
| `'say'` | `Answer` | | `grammar.h:189` |
| `'scale'` | `Climb` | | `grammar.h:222` |
| `'scope'` | `Scope` | ✓ debug | `grammar.h:132` |
| `'score'` | `Score` | ✓ | `grammar.h:68` |
| `'script'` | `ScriptOn`, `ScriptOff` | ✓ | `grammar.h:75` |
| `'scrub'` | `Rub` | | `grammar.h:398` |
| `'screw'` | `Turn`, `SwitchOn`, `SwitchOff` | | `grammar.h:487` |
| `'search'` | `Search` | | `grammar.h:402` |
| `'set'` | `Set`, `SetTo` | | `grammar.h:405` |
| `'shed'` | `Disrobe` | | `grammar.h:245` |
| `'shift'` | `Push`, `PushDir`, `Transfer` | | `grammar.h:376` |
| `'shine'` | `Rub` | | `grammar.h:398` |
| `'shit'` | `Strong` | | `grammar.h:409` |
| `'short'` | `LMode3` | ✓ | `grammar.h:34` |
| `'shout'` | `Answer` | | `grammar.h:189` |
| `'show'` | `Show` | | `grammar.h:413` |
| `'showdict'` | `ShowDict` | ✓ debug | `grammar.h:137` |
| `'showobj'` | `ShowObj` | ✓ debug | `grammar.h:141` |
| `'showverb'` | `ShowVerb` | ✓ debug | `grammar.h:146` |
| `'shut'` | `Close`, `SwitchOff` | | `grammar.h:226` |
| `'sing'` | `Sing` | | `grammar.h:417` |
| `'sip'` | `Drink` | | `grammar.h:248` |
| `'sit'` | `Enter` | | `grammar.h:420` |
| `'skip'` | `Jump`, `JumpIn`, `JumpOn`, `JumpOver` | | `grammar.h:316` |
| `'sleep'` | `Sleep` | | `grammar.h:424` |
| `'slice'` | `Cut` | | `grammar.h:235` |
| `'smash'` | `Attack` | | `grammar.h:198` |
| `'smell'` | `Smell` | | `grammar.h:427` |
| `'sniff'` | `Smell` | | `grammar.h:427` |
| `'sod'` | `Strong` | | `grammar.h:409` |
| `'sorry'` | `Sorry` | | `grammar.h:431` |
| `'speak'` | `Answer` | | `grammar.h:189` |
| `'squash'` | `Squeeze` | | `grammar.h:434` |
| `'squeeze'` | `Squeeze` | | `grammar.h:434` |
| `'stand'` | `Exit`, `Enter` | | `grammar.h:437` |
| `'superbrief'` | `LMode3` | ✓ | `grammar.h:34` |
| `'swallow'` | `Drink` | | `grammar.h:248` |
| `'sweep'` | `Rub` | | `grammar.h:398` |
| `'swim'` | `Swim` | | `grammar.h:442` |
| `'swing'` | `Swing` | | `grammar.h:445` |
| `'switch'` | `SwitchOn`, `SwitchOff` | | `grammar.h:449` |
| `'take'` | `Take`, `Remove`, `Inv` | | `grammar.h:456` |
| `'taste'` | `Taste` | | `grammar.h:462` |
| `'tell'` | `Tell` | | `grammar.h:465` |
| `'think'` | `Think` | | `grammar.h:469` |
| `'throw'` | `ThrowAt` | | `grammar.h:472` |
| `'thump'` | `Attack` | | `grammar.h:198` |
| `'tie'` | `Tie` | | `grammar.h:476` |
| `'timers'` | `TimersOn`, `TimersOff` | ✓ debug | `grammar.h:149` |
| `'torture'` | `Attack` | | `grammar.h:198` |
| `'touch'` | `Touch` | | `grammar.h:480` |
| `'trace'` | `TraceOn`, `TraceOff`, `TraceLevel` | ✓ debug | `grammar.h:154` |
| `'transcript'` | `ScriptOn`, `ScriptOff` | ✓ | `grammar.h:75` |
| `'transfer'` | `Transfer` | | `grammar.h:483` |
| `'tree'` | `XTree` | ✓ debug | `grammar.h:168` |
| `'turn'` | `Turn`, `SwitchOn`, `SwitchOff` | | `grammar.h:487` |
| `'twist'` | `Turn`, `SwitchOn`, `SwitchOff` | | `grammar.h:487` |
| `'uncover'` | `Open`, `Unlock` | | `grammar.h:353` |
| `'undo'` | `Open`, `Unlock` | | `grammar.h:353` |
| `'unlock'` | `Unlock` | | `grammar.h:494` |
| `'unscrew'` | `Turn`, `SwitchOn`, `SwitchOff` | | `grammar.h:487` |
| `'unscript'` | `ScriptOff` | ✓ | `grammar.h:80` |
| `'unwrap'` | `Open`, `Unlock` | | `grammar.h:353` |
| `'verbose'` | `LMode2` | ✓ | `grammar.h:31` |
| `'verify'` | `Verify` | ✓ | `grammar.h:83` |
| `'version'` | `Version` | ✓ | `grammar.h:86` |
| `'wait'` | `Wait` | | `grammar.h:497` |
| `'wake'` | `Wake`, `WakeOther` | | `grammar.h:500` |
| `'walk'` | `Go`, `GoIn`, `VagueGo` | | `grammar.h:292` |
| `'watch'` | `Examine` | | `grammar.h:269` |
| `'wave'` | `Wave`, `WaveHands` | | `grammar.h:507` |
| `'wear'` | `Wear` | | `grammar.h:513` |
| `'wipe'` | `Rub` | | `grammar.h:398` |
| `'wreck'` | `Attack` | | `grammar.h:198` |
| `'x//'` | `Examine` | | `grammar.h:269` |
| `'y//'` | `Yes` | | `grammar.h:516` |
| `'yes'` | `Yes` | | `grammar.h:516` |
| `'z//'` | `Wait` | | `grammar.h:497` |

**Total:** 185 verb words mapping to 78 distinct actions.

---

## §I.7 Actions by Category

This section groups all standard library actions by functional category for
quick reference. Each action name links to its detailed entry in §I.3, §I.4,
or §I.5.

### Movement

| Action | Description |
|--------|-------------|
| `Go` | Move the player to an adjacent room via a compass direction or door. |
| `GoIn` | Move the player inside an enterable object or through an `in_to` property. |
| `Enter` | Enter or get into/onto a container, supporter, or enterable object. |
| `Exit` | Leave the current container/supporter, or leave the room via `out_to`. |
| `GetOff` | Get off a specific supporter (as opposed to generic `Exit`). |
| `VagueGo` | Player typed "go" or "leave" without specifying a direction. |

### Object Manipulation

| Action | Description |
|--------|-------------|
| `Take` | Pick up an object and add it to the player's inventory. |
| `Drop` | Remove an object from inventory and place it in the current room. |
| `PutOn` | Place a held object on top of a supporter. |
| `Insert` | Place a held object inside a container. |
| `Remove` | Take an object out of a container. |
| `Transfer` | Move an object from one place to another (internal; rarely triggered directly). |
| `Empty` | Empty the contents of a container onto the ground. |
| `EmptyT` | Empty the contents of a container into/onto another object. |

### Giving and Showing

| Action | Description |
|--------|-------------|
| `Give` | Hand an object to an NPC. |
| `GiveR` | Reverse form: NPC gives an object to the player (internal redirect). |
| `Show` | Show an object to an NPC without giving it away. |
| `ShowR` | Reverse form of `Show` (internal redirect). |
| `ThrowAt` | Throw a held object at a target. |

### Examination and Search

| Action | Description |
|--------|-------------|
| `Look` | Display the full room description and list visible objects. |
| `Examine` | Examine an object closely, printing its `description` or `initial` text. |
| `LookUnder` | Look underneath an object. |
| `Search` | Search inside a container or on a supporter. |

### Container and Lock

| Action | Description |
|--------|-------------|
| `Open` | Open a container or door. |
| `Close` | Close a container or door. |
| `Lock` | Lock a lockable object with the correct key. |
| `Unlock` | Unlock a locked object with the correct key. |

### Switching

| Action | Description |
|--------|-------------|
| `SwitchOn` | Turn on a switchable device. |
| `SwitchOff` | Turn off a switchable device. |

### Clothing

| Action | Description |
|--------|-------------|
| `Wear` | Put on a piece of clothing from inventory. |
| `Disrobe` | Take off a worn piece of clothing. |

### Consumption

| Action | Description |
|--------|-------------|
| `Eat` | Eat an edible object, removing it from play. |
| `Drink` | Drink a drinkable object. |

### Communication

| Action | Description |
|--------|-------------|
| `Answer` | Reply to an NPC with a topic (e.g., "say hello to troll"). |
| `Ask` | Ask an NPC about a topic. |
| `AskFor` | Ask an NPC for a specific object. |
| `AskTo` | Order an NPC to perform an action (e.g., "ask bob to open door"). |
| `Tell` | Tell an NPC about a topic. |

### Physical Interaction

| Action | Description |
|--------|-------------|
| `Attack` | Attack or fight an object or NPC. |
| `Blow` | Blow on or into something. |
| `Burn` | Set fire to an object, optionally with a second held object. |
| `Buy` | Attempt to purchase something. |
| `Climb` | Climb an object such as a wall or tree. |
| `Consult` | Consult an object (e.g., a book) about a topic. |
| `Cut` | Cut or chop an object. |
| `Dig` | Dig in or at something. |
| `Fill` | Fill a container with something. |
| `Jump` | Jump in place (intransitive). |
| `JumpIn` | Jump into something. |
| `JumpOn` | Jump onto something. |
| `JumpOver` | Jump over an obstacle. |
| `Kiss` | Kiss or embrace an NPC. |
| `Listen` | Listen to ambient sounds or a specific object. |
| `Pull` | Pull or drag an object. |
| `Push` | Push or press an object. |
| `PushDir` | Push an object in a compass direction. |
| `Rub` | Rub, clean, polish, or wipe an object. |
| `Set` | Set an object to a particular state. |
| `SetTo` | Set an object to a specific named value. |
| `Sing` | Sing (intransitive). |
| `Sleep` | Go to sleep or take a nap. |
| `Smell` | Smell an object or the room in general. |
| `Squeeze` | Squeeze or squash an object. |
| `Swim` | Attempt to swim. |
| `Swing` | Swing on or at something. |
| `Taste` | Taste an object. |
| `Think` | Think (intransitive). |
| `Tie` | Tie, attach, fasten, or connect an object to something. |
| `Touch` | Touch or feel an object. |
| `Turn` | Turn, twist, rotate, screw, or unscrew an object. |
| `Wait` | Wait and let time pass (one turn). |
| `Wake` | Wake up (intransitive — the player wakes themselves). |
| `WakeOther` | Wake up an NPC. |
| `Wave` | Wave a held object. |
| `WaveHands` | Wave hands (intransitive). |

### Exclamations and Social

| Action | Description |
|--------|-------------|
| `Mild` | Player typed a mild expletive ("bother", "darn", "drat", "curses"). |
| `Strong` | Player typed a strong expletive ("damn", "shit", "fuck", "sod"). |
| `Sorry` | Player typed "sorry". |
| `Yes` | Player typed "yes" or "y". |
| `No` | Player typed "no". |
| `Pray` | Player typed "pray". |

### Inventory and Display

| Action | Description |
|--------|-------------|
| `Inv` | Display the player's inventory. |
| `InvTall` | Display the player's inventory in tall (vertical list) format. |
| `InvWide` | Display the player's inventory in wide (sentence) format. |
| `LMode1` | Set room descriptions to brief mode (short on revisit). |
| `LMode2` | Set room descriptions to verbose mode (always long). |
| `LMode3` | Set room descriptions to superbrief mode (always short). |
| `LModeNormal` | Reset room description mode to the default (brief). |

### Scoring (Meta)

| Action | Description |
|--------|-------------|
| `Score` | Display the player's current score and turn count. |
| `FullScore` | Display a detailed breakdown of how the score was earned. |
| `NotifyOn` | Enable automatic score-change notifications. |
| `NotifyOff` | Disable automatic score-change notifications. |

### Game Control (Meta)

| Action | Description |
|--------|-------------|
| `Quit` | Quit the game. |
| `Restart` | Restart the game from the beginning. |
| `Save` | Save the current game state to a file. |
| `Restore` | Restore a previously saved game state. |
| `Verify` | Verify the integrity of the story file. |
| `Version` | Display the game's version and compiler information. |
| `ScriptOn` | Begin transcripting game output to a file. |
| `ScriptOff` | Stop transcripting. |
| `CommandsOn` | Begin recording player commands to a file. |
| `CommandsOff` | Stop recording player commands. |
| `CommandsRead` | Replay commands from a previously recorded file. |
| `Pronouns` | Display the current pronoun assignments (it, him, her, them). |
| `Places` | List all rooms the player has visited. |
| `PlacesTall` | List visited rooms in tall (vertical list) format. |
| `PlacesWide` | List visited rooms in wide (sentence) format. |
| `Objects` | List all objects the player has handled. |
| `ObjectsTall` | List handled objects in tall (vertical list) format. |
| `ObjectsWide` | List handled objects in wide (sentence) format. |

### Debug (Meta)

These actions are only available when the game is compiled in debug mode.

| Action | Description |
|--------|-------------|
| `TraceOn` | Turn on parser tracing (level 1). |
| `TraceOff` | Turn off parser tracing. |
| `TraceLevel` | Set parser tracing to a specific numeric level. |
| `RoutinesOn` | Trace routine entry (print routine names as they execute). |
| `RoutinesOff` | Turn off routine tracing. |
| `RoutinesVerbose` | Enable verbose routine tracing with additional detail. |
| `ActionsOn` | Trace actions (print each action as it is processed). |
| `ActionsOff` | Turn off action tracing. |
| `TimersOn` | Trace timers and daemons each turn. |
| `TimersOff` | Turn off timer/daemon tracing. |
| `ChangesOn` | Trace object property changes. |
| `ChangesOff` | Turn off property-change tracing. |
| `Predictable` | Seed the random-number generator for repeatable behaviour. |
| `Goto` | Teleport the player to a room by internal object number. |
| `GoNear` | Teleport the player to the room containing a named object. |
| `Scope` | Display all objects currently in scope. |
| `ShowVerb` | Show the grammar lines for a given verb. |
| `ShowObj` | Display the internal details of an object. |
| `ShowDict` | Display the game dictionary. |
| `XAbstract` | Move one object into another (debug object tree manipulation). |
| `XPurloin` | Acquire any object in the game, bypassing all rules. |
| `XTree` | Display the full object tree. |
| `GlkList` | List all active Glk objects (Glulx target only). |

### Fake Actions

Fake actions are never triggered by player input. They are generated internally
by the library and exist so that `before`/`after` rules can intercept them.

| Action | Description |
|--------|-------------|
| `LetGo` | Fired on a container when an object is removed from it. |
| `Receive` | Fired on a container/supporter when an object is placed in/on it. |
| `ThrownAt` | Fired on the target of a `ThrowAt` action. |
| `Order` | Fired when the player gives a command to an NPC (e.g., "troll, go north"). |
| `TheSame` | Fired when the parser cannot distinguish two objects and needs help. |
| `PluralFound` | Fired when the parser encounters a plural word (e.g., "take all"). |
| `ListMiscellany` | Used internally when printing object lists to generate stock messages. |
| `Miscellany` | Used internally for miscellaneous library messages. |
| `RunTimeError` | Generated when a run-time error occurs (Strict mode). |
| `Prompt` | Fired each turn just before printing the command prompt. |
| `NotUnderstood` | Fired when the parser cannot understand the player's command at all. |
| `Going` | Fired during movement to allow `before` rules to intercept room changes. |
