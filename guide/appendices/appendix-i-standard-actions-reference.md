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
  `grammar.h` with corresponding handler routines in `verblibm.h` and
  `parserm.h`. These are listed in §I.3 (grouped by category).
- **Fake actions** — generated internally by the library to communicate
  between objects. They have no grammar and no `*Sub` routine. These are
  listed in §I.4.

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
