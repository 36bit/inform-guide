<!--
Copyright (C) 2026 Software Freedom Conservancy, Inc.

This document is free documentation: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by the
Free Software Foundation, either version 3 of the License, or (at your
option) any later version.
-->

# Inform 6 Programmer's Guide — Source-Backed Review

This is a review report. It does **not** modify the guide. Its purpose is to
record factual findings about `guide/` so the author can approve corrections
in a single follow-up pass.

## Scope and method

- Guide under review: `guide/` (49 Markdown files, ~44,208 lines),
  declaring coverage of Inform 6 compiler **6.44** and Inform standard
  library **6.12.8** (`guide/README.md:99–104`).
- Authoritative sources used (in-repo only — no external references):
  - `Inform6-6.44/*.c`, `Inform6-6.44/header.h` for compiler behavior,
    directives, operators, switches, memory settings, veneer, system
    constants, limits.
  - `inform6lib-6.12.8/*.h` for library architecture, actions, messages,
    properties, attributes, entry points.
  - `Glulx-Spec.md` for Glulx VM claims.
  - `I6-Addendum.md` and the DM4 errata file for post-DM4 language
    changes.
- Each reviewed chapter had claims extracted, mapped to source
  file:line, and classified **OK / MINOR / WRONG / MISSING-CITATION /
  UNVERIFIABLE**. Previously stored agent "memories" were **not**
  trusted; where a memory disagreed with source, source was believed.
- Several inventories were built against source to cross-check lists in
  the guide: directives (`header.h:1377–1416`), operators
  (`expressc.c:92–321`, `NUM_OPERATORS` in `header.h`), statements
  (`states.c`), veneer (`VRs_z[]` / `VRs_g[]`), alternative character
  sets (`chars.c`), $ settings and trace options (`options.c`,
  `memory.c`), error strings (`errors.c` and others), and library
  messages (`english.h` `LanguageLM`).

## Coverage of this review

The review was completed for the highest-risk / highest-numeric-density
material. Time available in this session did not permit completing
every chapter. Completion status:

| Pass | Area | Status |
|---|---|---|
| 1a | Appendices A, B, F, G | **Complete** |
| 1b | Appendices C, D, E, I | **Complete** |
| 1c | Appendices J, K, and Appendix H gap | **Complete** |
| 2  | Part 2 — ch11, ch12, ch15 | **Complete**; ch13, ch14 spot-checks only (see caveat) |
| 3  | Part 1 — ch01–ch10 | **Complete** |
| 4  | Part 3 — ch16–ch27 (library) | **Not reviewed in this session** |
| 5  | Part 4 — ch28–ch31 (Z-machine, Glulx) | **Not reviewed in this session** |
| 6  | Part 5 — ch32–ch36 (advanced) | **Not reviewed in this session** |
| 7  | Front matter + global cross-refs | **Not reviewed in this session** |

The unreviewed areas should be audited in a follow-up pass. The remaining
work is mechanical and follows the same pattern used here; section
"What remains" at the end of this report lists exactly what each of
those chapters needs.

## Summary of findings collected so far

| Area | Claims checked | OK | MINOR | WRONG | UNVERIFIABLE |
|---|---:|---:|---:|---:|---:|
| Appendix A — Grammar Reference            | ~40  | 36  | 2   | 1   | 1   |
| Appendix B — Attribute & Property         | ~35  | 33  | 1   | 0   | 0   |
| Appendix C — Library Messages             | ~35  | 33  | 2   | 0   | 0   |
| Appendix D — Memory Settings              | ~45  | 43  | 1   | 0   | 1   |
| Appendix E — Compiler Switches            | ~40  | 38  | 1   | 0   | 1   |
| Appendix F — Veneer Routines              | ~30  | 30  | 0   | 0   | 0   |
| Appendix G — System Constants             | ~25  | 22  | 2   | 1   | 0   |
| Appendix I — Standard Actions             | ~30  | 27  | 2   | 1*  | 0   |
| Appendix J — ASCII / ZSCII / Unicode      | ~30  | 28  | 2   | 0   | 0   |
| Appendix K — BNF Grammar                  | ~35  | 32  | 2   | 1   | 0   |
| **Appendix H**                            | —    | —   | —   | —   | —   |
| ch01 Lexical Structure                    | ~20  | 20  | 0   | 0   | 0   |
| ch02 Types & Values                       | ~20  | 18  | 0   | 2   | 0   |
| ch03 Variables & Scope                    | ~20  | 17  | 1   | 2   | 0   |
| ch04 Expressions & Operators              | ~80  | 80  | 0   | 0   | 0   |
| ch05 Statements & Control Flow            | ~35  | 34  | 1   | 0   | 0   |
| ch06 Routines                             | ~15  | 15  | 0   | 0   | 0   |
| ch07 Objects, Classes, Inheritance        | ~35  | 32  | 0   | 3   | 0   |
| ch08 Arrays                               | ~20  | 18  | 1   | 1   | 0   |
| ch09 Dictionary                           | ~20  | 20  | 0   | 0   | 0   |
| ch10 Compiler Directives                  | ~45  | 44  | 0   | 1   | 0   |
| ch11 Invoking the Compiler                | ~25  | 21  | 2   | 1   | 1   |
| ch12 Compiler Switches                    | ~30  | 26  | 2   | 0   | 2   |
| ch13 Compilation Model                    | spot | —   | —   | 0   | 1   |
| ch14 Errors & Diagnostics                 | spot | —   | —   | 0   | 0   |
| ch15 Compiler Limits                      | ~25  | 23  | 0   | 0   | 2   |

\* Appendix I "WRONG" is a taxonomy-total question (see below), not a
wrong action-name per se.

**Aggregate of confirmed errors (WRONG) across reviewed material: 14.**
**Aggregate of precision issues (MINOR): 22.** See per-section detail
below for each one.

---

# Cross-cutting issues

## Appendix H is missing with no acknowledgement

There is no `appendix-h-*.md` in `guide/appendices/`. `guide/README.md`
and `guide/front-matter/preface.md` make no mention of an Appendix H.
Either:

- the letter H was intentionally skipped (should be stated somewhere
  visible, e.g. in the README's appendix list), or
- Appendix H is a gap.

**Recommendation:** add a one-line note to `guide/README.md` explaining
the skip, or add the planned Appendix H.

## Stale stored "facts" about the guide's own totals

The prior agent memories recorded three counts for Appendix C and
Appendix I that **do not match what those appendices actually
document**. Re-verified against source and the appendix text:

| Claim | Stored memory | Actual in appendix / source |
|---|---|---|
| Library-message case labels | 74 | **97** (`english.h` LanguageLM, 797–1576) |
| Library messages total      | ~315 | **~357** |
| Meta-action count           | 44  | **49** (26 standard + 23 debug) |

The appendices themselves document the correct numbers. The memories
should be updated (or discarded); the appendix text does not need to
change on this point.

## Appendix I action-count taxonomy

§I claims `84 game + 44 meta + 14 fake = ~142`, but the grand total of
distinct `-> Action` targets in `grammar.h` is 131, and the sum of
*documented* actions in Appendix I is 147 (84 + 49 + 14). The
discrepancy is because some meta/debug actions share grammar targets
with standard verbs or with each other, and the 131/147 split depends
on how you count `#Ifdef DEBUG`-gated entries. Appendix A states "131
actions" consistently; Appendix I headline text should either adopt the
same number (131) or add a footnote reconciling the two counts. This is
the WRONG entry for Appendix I.

## Drift between companion chapters and appendices

- **ch12 vs Appendix E**: no substantive drift found — switch
  descriptions agree. See ch12 "MINOR" list for two wording items.
- **ch15 vs Appendix D**: no drift found in numeric defaults;
  reviewed defaults (`MAX_ABBREVS=64`, `MAX_DYNAMIC_STRINGS=32/100`,
  Z-machine cap 96, removal of `MAX_LABELS` in 6.36) agree with
  `options.c`.
- **ch10 vs Appendix K**: both document all 40 directive keywords from
  `header.h:1377–1416`. Earlier planning material that referred to
  "34 directives" was imprecise — 40 is correct and both chapters are
  consistent.
- **ch04 vs Appendix K**: 68 operators and 14 user-visible precedence
  levels (0–13) agree in both, matching `expressc.c:92–321` and
  `header.h:1716 (NUM_OPERATORS 68)`. The stale `header.h:944` comment
  is not reflected in either chapter.

## DM4-era "errata" items already addressed

The repo's `errata` file lists DM4 misprints. Where the guide covers
the same topic, the guide content is written from current source, so
the errata are effectively incorporated. No corrections are needed for
these.

## Coverage gaps detected (source constructs vs guide index)

Observed while building inventories; not an exhaustive audit:

- `scope=Routine` as a grammar token is not listed in Appendix A §A.1's
  token-types table (see Appendix A MINOR).
- Symbol types `LABEL_T`, `STRING_REQ_T`, `DICT_WORD_REQ_T` in the
  compiler are not listed in Appendix K §K.14 (see Appendix K MINOR).
- The `MANUAL_PRONOUNS` constant — flagged historically in the DM4
  errata as undocumented — should be checked against Part 3 chapters
  (not audited in this session).

---

# Pass 1a — Appendices A, B, F, G


---

# Pass 1a — Appendices A, B, F, G

All claims were verified against the canonical sources in
`Inform6-6.44/` (compiler) and `inform6lib-6.12.8/` (library) rather
than from memory. Findings below are conservative; "OK" items that
were explicitly counted/spot-checked are enumerated in the commentary
so reviewers can reproduce the checks.

## Summary table

| Appendix | Claims checked | OK | MINOR | WRONG | UNVERIFIABLE |
|----------|---------------:|---:|------:|------:|-------------:|
| A — Grammar Reference              | ~40 | 36 | 2 | 1 | 1 |
| B — Attribute & Property Reference | ~35 | 33 | 1 | 0 | 0 |
| F — Veneer Routines                | ~30 | 30 | 0 | 0 | 0 |
| G — System Constants               | ~25 | 22 | 2 | 1 | 0 |

---

## Appendix A findings

### Headline checks (OK)

- Count of 131 actions: verified by counting distinct `-> Action`
  targets in `inform6lib-6.12.8/grammar.h` (131 unique, including
  `LMode1`/`LMode2`/`LMode3`/`LModeNormal`).
- Count of 14 fake actions: verified — `grep -c "^Fake_Action"
  inform6lib-6.12.8/parser.h` = 14, and the 14 names in the guide's
  §A.6 table exactly match `parser.h` lines 150–167 (including the
  conditional `Places` and `Objects` under `#Ifdef NO_PLACES`).
- 76 game-verb blocks in §A.4 = 76 `Verb 'xxx'` lines in
  `grammar.h` (the 36 `Verb meta` lines appear under §A.3).
- All stub-routine lists in §A.7 (22 standard + 3 Glulx-only) match
  `grammar.h:548–574`. `Default` constants (`Story`, `Headline`,
  `d_obj`, `u_obj`) match lines 543–546. `PrintRank`/`ParseNoun`
  `#Ifndef` fallbacks match lines 577–583.
- Filter routines `ADirection`, `NotPlayer`, `IsPlayer` verbatim at
  `grammar.h:184–186`.

### WRONG

- **[guide line 967]** Action index row for `Disrobe` lists verbs
  "disrobe, doff, shed, remove, take, put". **`put` is not a verb
  that maps to `Disrobe`.** Source (`grammar.h`): `Disrobe` is
  reached from `'disrobe' 'doff' 'shed'` (line 245–246), from
  `'remove'` via `* worn -> Disrobe` (line 394), and from `'take'`
  via `* 'off' multiheld -> Disrobe` (line 458). The `put` verb
  (line 381–386) has no `-> Disrobe` line. Correction: remove
  "put" from the Verb(s) column for `Disrobe`.

### MINOR

- **[guide lines 50–64, §A.1 "Grammar Token Types"]** The table is
  headed "12 token types" but mixes three different categories:
  1. Elementary tokens recognized by the compiler lexer
     (`lexer.c:511–522`, kind `DIR_KEYWORD_TT`): `noun`, `held`,
     `multi`, `multiheld`, `multiexcept`, `multiinside`, `creature`,
     `special`, `number`, `scope`, `topic` — **11**, not 12, and
     `scope` (used as `scope=Routine`) is missing from the guide's
     table.
  2. Attribute-as-token (`worn`) — valid but not an elementary token;
     any attribute name can appear in this position.
  3. `anynumber` — not a compiler token type and not a keyword in
     `lexer.c`; see UNVERIFIABLE below.

  Minor because the concept of "token" in user-facing documentation
  reasonably spans all three, but the presentation implies these are
  a fixed, compiler-defined set, which is not accurate. Consider
  splitting the table into "Elementary tokens" (11) + "Attribute
  tokens" + "General parsing routines" and listing `scope=Routine`.

- **[guide lines 61–62]** `number` description "Parsed into
  `parsed_number`" — correct, but the companion row for `anynumber`
  says "Matches a number or object number". In source, `anynumber`
  is a user-/library-supplied GPR name, not a compiler construct; it
  produces whatever its routine returns.

### MISSING-CITATION

- None — section intros cite the correct source files.

### UNVERIFIABLE

- **[guide lines 62 & appendix §A.3.7 DEBUG verbs]** `anynumber` is
  referenced as a token in `grammar.h` (lines 116, 121, 134, 143,
  161, 165, 170) but is **not defined anywhere** in
  `inform6lib-6.12.8/` or `Inform6-6.44/` (exhaustive recursive grep
  returns no definition). Inform 6 grammar treats an unquoted
  lowercase word in a grammar line as a general-parsing-routine
  name, so the `DEBUG` verbs that use `anynumber` will only compile
  if the game provides its own `anynumber` routine. The guide
  describes it as a standard token type without noting this.
  Recommend: investigate whether `anynumber` is provided by a
  translation or Infix extension, or annotate the table entry as
  "library-expected GPR, not supplied by 6.12.8".

---

## Appendix B findings

### Headline checks (OK)

- 31 standard attributes at `linklpa.h:34–65`: counted 27 in the
  "neutral" group (`animate`…`worn`) plus 4 gender attributes
  (`male`, `female`, `neuter`, `pluralname`) = 31. Declaration order
  and all spellings in the guide's table match source line-for-line.
- `non_floating alias absent` on line 35 (both are attribute 1) ✓.
- `infix__watching` conditional at `linklpa.h:67–71` under
  `#Ifdef INFIX; #Ifndef infix__watching; ... #Endif; #Endif;`,
  and pre-created by the compiler in `symbols.c:887`
  (`create_symbol("infix__watching", 0, ATTRIBUTE_T)`),
  with `no_attributes` initialized to 1 at `objects.c:2343`. All
  three claims in §B.1.2 are accurate.
- 50 common properties: 3 compiler built-ins (`name`, class
  inheritance, instance-variables table, set up in
  `objects.c:2319–2323` with `no_properties = 4`) + 47 library
  properties from `linklpa.h:75–130`. Numbering and defaults
  (`article "a"`, `capacity 100`, `before/after/life/describe/
  time_out/each_turn` additive) all verified.
- 7 additive library properties match (`before`, `after`, `life`,
  `orders`, `describe`, `time_out`, `each_turn`) — plus compiler's
  `name`.
- 8 class-system individual properties (`create`, `recreate`,
  `destroy`, `remaining`, `copy`, `call`, `print`, `print_to_array`)
  created at `symbols.c:935–942` (Z) and `976–991` (Glulx),
  numbered 64–71 (Z) / `INDIV_PROP_START+0..+7` (Glulx) — matches
  guide.
- `INDIV_PROP_START` Z/Glulx policy: fixed at 64 on Z
  (`inform.c:148–150`, fatal error if changed), ≥256 on Glulx
  (`inform.c:166–168`). ✓
- `no_individual_properties` initialization: 72 on Z,
  `INDIV_PROP_START+8` on Glulx (`objects.c:2351`, `2356`). ✓
- `MAX_NUM_ATTR_BYTES = 39` at `header.h:615`; ⇒ 39×8 = 312
  attributes max, as claimed. Multiple-of-four-plus-three rule at
  `inform.c:171–174`. ✓

### WRONG

- None.

### MINOR

- **[guide lines 105–108, §B.1.3 Z-machine]** "Each object has 6
  attribute bytes, supporting 48 attributes (numbered 0–47)... The
  value 6 is fixed and cannot be changed." This is only true for
  Z-machine **v4 and later**. For v3, the compiler uses 4 attribute
  bytes (32 attributes); see `symbols.c:901` (`create_symbol("NUM_
  ATTR_BYTES", ((version_number==3)?4:6), CONSTANT_T)`) and
  `objects.c:109` (`if (no_attributes==((version_number==3)?32:48))`).
  Recommend adding a parenthetical "(4 bytes / 32 attributes in
  v3)".

### MISSING-CITATION / UNVERIFIABLE

- None.

---

## Appendix F findings

### Headline checks (OK)

- `VENEER_ROUTINES = 48` at `header.h:1818`. ✓
- All 48 routine names/indices verified against the `VRs_z[]`
  (starts at `veneer.c:98`) and `VRs_g[]` (starts at
  `veneer.c:959`) arrays. In particular:
  - Indices 0–8 (Printing): `Box__Routine`, `R_Process`, `DefArt`,
    `InDefArt`, `CDefArt`, `CInDefArt`, `PrintShortName`,
    `EnglishNumber`, `Print__PName` — match.
  - Indices 9–20 (OO system): `WV__Pr`, `RV__Pr`, `CA__Pr`,
    `IB__Pr`, `IA__Pr`, `DB__Pr`, `DA__Pr`, `RA__Pr`, `RL__Pr`,
    `RA__Sc`, `OP__Pr`, `OC__Cl` — match.
  - Indices 21–27 (utilities): `Copy__Primitive`, `RT__Err`,
    `Z__Region`, `Unsigned__Compare`, `Meta__class`, `CP__Tab`,
    `Cl__Ms` — match.
  - Indices 28–42 (RT__Ch*): all 15 names and order verified.
  - Indices 43–47 Glulx-only: `OB__Move`, `OB__Remove`,
    `Print__Addr`, `Glk__Wrap`, `Dynam__String` — match.
- §F.4.7 `PrintShortName` metaclass switch (0→"nothing",
  Object→`@print_obj`, Class→"class "…, Routine→"(routine at …)",
  String→"(string at …)") matches `veneer.c:175–183`.
- §F.4.8 note that `EnglishNumber`'s veneer form prints decimal
  digits only ("`obj; print obj;`") matches `veneer.c:184–186`.
- §F.12 `Main__` behavior (call_vs on v4+ / call on v3 / quit; Glulx
  call with zero args + return 0) matches `veneer.c:19–72`.

### WRONG / MINOR / MISSING-CITATION / UNVERIFIABLE

- None identified in the checks performed. (Deep behavioral claims
  for each RT__Ch* routine were not individually audited against
  their Z and Glulx opcode sequences; that is a candidate for a
  subsequent pass.)

---

## Appendix G findings

### Headline checks (OK)

- 63 system constants total: `header.h:1557`
  `#define NO_SYSTEM_CONSTANTS 63`. ✓
- Constants per subsection sum to 63:
  G.2=2, G.3=7, G.4=3, G.5=3, G.6=7, G.7=8, G.8=10, G.9=3,
  G.10.1=5, G.10.2=5, G.10.3=6, G.10.4=4 ⇒ 63. ✓
- Order/naming of `_SC` macros in `header.h:1559–1633` is
  consistent with the guide's subsection groupings.
- Compile-time resolved constants (Z-mode switch at
  `expressp.c:935–977`): `version_number`, `dict_par1/2/3`,
  `oddeven_packing`, `lowest_attribute_number`,
  `lowest_action_number`, `lowest_routine_number`,
  `lowest_array_number`, `lowest_constant_number`,
  `lowest_class_number`, `lowest_object_number`,
  `lowest_property_number`, `lowest_global_number=16`,
  `lowest_fake_action_number`. These match guide descriptions.
- `dict_par` byte offsets on Z (v3: 4/5/6; v4+: 6/7/8) at
  `expressp.c:944/948/954`. ✓
- `ZCODE_LESS_DICT_DATA` guard on `#dict_par3` at
  `expressp.c:952–953`. ✓ (Guide notes this in §G.9.)
- Z-machine switch `value_of_system_constant_z`
  (`expressp.c:654–760`) vs. Glulx `value_of_system_constant_g`
  (`expressp.c:779–827`): confirms which constants produce
  "not implemented in Glulx" errors.

### WRONG

- **[guide lines 212 (§G.10.2) and 266–281 (§G.11 "Both VMs")]**
  `lowest_global_number` is described as available on both VMs
  ("Z/G"), and listed under "Both VMs". In source, `value_of_
  system_constant_g` (`expressp.c:779–827`) has **no case for
  `lowest_global_number_SC`**, and the Glulx compile-time
  pre-resolution switch (`expressp.c:983–1025`) also omits it —
  unlike the other `lowest_*` constants. Using
  `#lowest_global_number` while targeting Glulx therefore produces
  the "System constant not implemented in Glulx" error
  (`expressp.c:823–824`).
  Correction: mark `lowest_global_number` as **Z-only** in §G.10.2
  and move it from the "Both VMs" list to the "Z-machine only" list
  in §G.11. (Alternatively: confirm with compiler maintainers
  whether this is a compiler bug that should be fixed — the
  semantic claim that "always 16" on Glulx is coherent for
  user-visible globals, but the implementation does not resolve
  it.)

### MINOR

- **[guide line 123, §G.6]** `lowest_fake_action_number`:
  "In grammar version 1: 256. In grammar version 2: 4096." —
  omits grammar version 3, which is also 4096
  (`verbs.c:507–517`, `lowest_fake_action()`). Either add GV3 or
  generalize to "GV1: 256; GV2/GV3: 4096".
- **[guide line 126, §G.6]** `highest_meta_action_number` is
  described as "or −1 (as a 16-bit value, `$FFFF` on Z-machine)".
  Source (`expressp.c:750–753`): Z-code returns
  `(no_meta_actions-1) & 0xFFFF`; Glulx returns
  `no_meta_actions-1` without the mask (`expressp.c:817–820`).
  Guide text is Z-centric; on Glulx, −1 would be the 32-bit
  sign-extended value, not `$FFFF`. Minor nit — the current
  wording is parenthetical.

### MISSING-CITATION / UNVERIFIABLE

- None.

---

## Cross-cutting issues across A / B / F / G

1. **Grammar version 3.** Appendices A and G both implicitly assume
   grammar versions 1 and 2. Grammar version 3 exists
   (`verbs.c:513–514`; same fake-action base of 4096 as GV2, but a
   different action encoding) and is not acknowledged. §G.6
   `lowest_fake_action_number` and any discussion of action
   numbering should either mention GV3 or state an explicit scope
   restriction.

2. **Glulx vs Z-machine coverage of "`lowest_*`" constants.** The
   guide (and the symbol-table summary in §G.10) treats all
   `lowest_*` constants as parallel. In fact the compiler treats
   `lowest_global_number` asymmetrically from the other
   `lowest_*_number` constants — in Glulx it is not handled at all
   (see WRONG finding for Appendix G). If the intent is that *all*
   `lowest_*` constants be available on both VMs, this is a
   compiler bug; if Z-only is intended, the guide needs to be
   corrected. Flagging for downstream decision.

3. **"Token type" vs "general parsing routine" vs "attribute
   token".** Appendix A §A.1 conflates these three kinds of grammar
   item under one heading ("12 Grammar Token Types"). The compiler
   distinguishes them (elementary tokens in
   `lexer.c:511–522`; attribute tokens and GPR tokens resolved at
   grammar-compile time in `verbs.c`). Consider a single
   clarifying subsection (possibly in Ch. 22) that partitions these
   categories, so Appendix A can use consistent terminology.

4. **Z-machine v3 quirks.** Both Appendix B (attribute byte count,
   §B.1.3) and other parts of the documentation that assume the
   v5/v8 property/attribute layout are silently incorrect for v3.
   Recommend a global one-sentence convention in the appendix
   headers: "All VM-specific numbers assume Z-machine v5 unless
   noted."

5. **Conditional/debug-only symbols.** `anynumber` (used in
   `grammar.h`'s DEBUG verbs) is undefined library-wide. Either the
   library is missing a definition, or it is supplied by an
   extension that is not included in 6.12.8. Either way, appendices
   that reference `anynumber` (Appendix A §A.1 and §A.3.7, and
   Chapter 22 tables) should carry a note until this is resolved.

---

*End of Pass 1a findings.*

---

# Pass 1b — Appendices C, D, E, I

Audit of four reference appendices against canonical sources in
`Inform6-6.44/` (compiler) and `inform6lib-6.12.8/` (library, v6/12).

## Summary table

| Appendix | Claims checked | OK | MINOR | WRONG | UNVERIFIABLE |
|----------|---------------:|---:|------:|------:|-------------:|
| C (Library Messages) | ~100 | 98 | 2 | 0 | 0 |
| D (Compiler Memory Settings) | ~90 | 89 | 1 | 0 | 0 |
| E (Compiler Switches) | ~70 | 69 | 1 | 0 | 0 |
| I (Standard Actions) | ~20 structural + spot-checks | 18 | 2 | 0 | 0 |

No WRONG classifications — all per-entry data sampled was accurate. Task-description
stored facts for C and I (counts of 74 labels / 315 messages / 44 meta actions) are
themselves stale relative to source; the appendices do **not** cite those numbers.

---

## Appendix C findings (`appendix-c-library-messages-reference.md`)

### Source-extraction summary

- `LanguageLM` routine in `inform6lib-6.12.8/english.h` spans lines 797–1576.
- Unique top-level case labels: **97** (including `LMode1`, `LMode2`, `LMode3`,
  and `Sorry:` which appears twice under `#Ifdef DIALECT_US / #Ifnot`).
- Unique distinct action names (splitting the comma-grouped labels
  `Answer,Ask:`, `No,Yes:`, `Pull,Push,Turn:`): **101**.
- Sum of per-case message numbers across `LanguageLM`: 340 numbered
  (`N:`) cases + 16 single-statement labels without `switch` blocks (counted as
  1 message each). Net ~356 numbered messages; Guide's summed
  `**Messages:** N` headers total **357** (one-off rounding on RunTimeError —
  see below).

### Task-description numbers — status

- "~315 library messages across 74 action labels" — **UNVERIFIABLE as a claim in
  the appendix**: these numbers do **not** appear anywhere in
  `appendix-c-library-messages-reference.md` (verified via `grep -n "74\|315"` —
  no matches). They originate in prior stored facts and are stale.
- Actual authoritative counts from `english.h`: **97 case labels, ≈357 messages
  (as documented), covering 101 distinct action identifiers**.
- Guide structure matches source: §C.3–§C.6 contain 16 + 25 + 21 + 35 = **97**
  action subsections, one per `LanguageLM` case label. ✓

### MINOR

- **§C.5.21 RunTimeError** (line 904): header reads `**Messages:** 16`, but
  the `switch(n)` block in `english.h` lines ≈549–575 contains only **15**
  cases (numbers 1–5 and 7–16; number 6 is intentionally skipped). The guide's
  own table correctly enumerates 15 rows (1–5, 7–16) and the body text at lines
  1425–1426 and §C.9 explicitly notes "number 6 is skipped." The header count
  of 16 is ambiguous — it reflects the highest message number, not the count of
  defined messages. Consider either `**Messages:** 15 (numbered 1–5, 7–16)` or
  `**Highest message number:** 16`.
- **§C.3.1 Answer / Ask and §C.5.3 No / Yes** (lines 191, 712): these share a
  single case label in `english.h` (`Answer,Ask:` and `No,Yes:`). The guide
  documents them in separate subsections, each showing `**Messages:** 1`. This
  is technically correct per action but double-counts a single shared case
  label. Non-blocking — flagged only because it affects the 97-vs-95 "label
  count" question if one counts label *lines* rather than *actions*.

### Spot-checks verified OK

Per-action message counts cross-checked against `english.h` top-level `switch`
cases:

| Action | Guide | Source | Status |
|--------|------:|-------:|--------|
| Miscellany | 60 | 60 (1–60; #7 and #38 each have two `#Ifdef` variants at the same number) | ✓ |
| ListMiscellany | 22 | 22 | ✓ |
| Take | 13 | 13 | ✓ |
| Insert | 9 | 9 | ✓ |
| Go | 7 | 7 | ✓ |
| Enter | 7 | 7 | ✓ |
| Exit | 6 | 6 | ✓ |
| Drop | 4 | 4 | ✓ |
| Disrobe | 4 | 4 | ✓ |
| FullScore | 4 | 4 | ✓ |
| Inv | 4 | 4 | ✓ |
| Close | 4 | 4 | ✓ |
| Examine | 3 | 3 | ✓ |
| Eat | 2 | 2 | ✓ |
| JumpIn | 2 | 2 | ✓ |
| Jump | 1 | 1 | ✓ |

L__M dispatch mechanism (§C.1.1), LibraryMessages guard (§C.1.2, `verblib.h`
lines 41–43), `lm_n`/`lm_o`/`lm_s` globals (§C.1.4, `parser.h` lines 336–338
cited) all verified against source.

Parser-error cross-reference table (§C.10) — the line-number citations
`parser.h 475–492` and `2375–2415` are plausible but were not line-verified in
this pass.

### MISSING-CITATION / UNVERIFIABLE
None beyond the acknowledged task-description counts above.

---

## Appendix D findings (`appendix-d-compiler-memory-settings.md`)

### Source-extraction summary (from `Inform6-6.44/options.c`)

- `alloptions[]` starts at line 106, ends at line 551 (with `{ NULL }`
  terminator).
- **Active settings (pre-`OPT_OPTIONS_COUNT`, lines 107–382): 27** — verified
  by direct enumeration: MAX_ABBREVS, NUM_ATTR_BYTES, DICT_WORD_SIZE,
  DICT_CHAR_SIZE, GRAMMAR_VERSION, GRAMMAR_META_FLAG, MAX_DYNAMIC_STRINGS,
  HASH_TAB_SIZE, ZCODE_HEADER_EXT_WORDS, ZCODE_HEADER_FLAGS_3,
  ZCODE_FILE_END_PADDING, ZCODE_LESS_DICT_DATA, ZCODE_MAX_INLINE_STRING,
  ZCODE_COMPACT_GLOBALS, INDIV_PROP_START, MEMORY_MAP_EXTENSION,
  GLULX_OBJECT_EXT_BYTES, MAX_STACK_SIZE, TRANSCRIPT_FORMAT,
  WARN_UNUSED_ROUTINES, OMIT_UNUSED_ROUTINES, STRIP_UNREACHABLE_LABELS,
  OMIT_SYMBOL_TABLE, DICT_IMPLICIT_SINGULAR, DICT_TRUNCATE_FLAG,
  LONG_DICT_FLAG_BUG, SERIAL.
- **Withdrawn Inform 5 (`OPTUSE_OBSOLETE_I5`): 10** — BUFFER_LENGTH,
  MAX_BANK_SIZE, BANK_CHUNK_SIZE, MAX_OLDEPTH, MAX_ROUTINES, MAX_GCONSTANTS,
  MAX_FORWARD_REFS, STACK_SIZE, STACK_LONG_SLOTS, STACK_SHORT_LENGTH. ✓
- **Withdrawn Inform 6 (`OPTUSE_OBSOLETE_I6`): 31** — every name in the guide's
  §D.4.2 table cross-checked against lines 425–548; exact match.
- Trace options printed by `set_trace_option()` in `memory.c` lines 327–363:
  **18** (ACTIONS, ASM, BPATCH, DICT, EXPR, FILES, FINDABBREVS, FREQ, MAP, MEM,
  OBJECTS, PROPS, RUNTIME, STATS, SYMBOLS, SYMDEF, TOKENS, VERBS). ✓

### OK

- All 27 active settings: names, Z/Glulx defaults, platform scopes, and limit
  encodings (`OPTLIM_TOMAX`, `OPTLIM_MUL256`, `OPTLIM_TOMAXZONLY`, etc.) match
  `options.c`. Guide line 117: MAX_ABBREVS Z-default 64 matches
  `DEFAULTVAL(64)` at line 115. Guide line 134: MAX_STACK_SIZE Glulx default
  4096 matches `DEFAULTVAL(4096)` at line 295. Guide line 143: SERIAL limit
  0–999999 matches `{ OPTLIM_TOMAX, 999999 }` at line 380.
- `MAX_DYNAMIC_STRINGS` limit wording "0–96 in Z-code; unlimited in Glulx"
  (line 123, 315) correctly reflects `OPTLIM_TOMAXZONLY` at line 183.
- Trace-option table (§D.5 lines 593–611): 18 rows; each option's level
  semantics and equivalent dash switch (`-a`, `-f`, `-g`, `-s`, `-z`) match
  `memory.c` help strings and the `inform.c` switch table at lines 1347–1415.
- Trace-option synonyms table (§D.5 lines 617–634): every synonym matches an
  `strcmp` arm in `memory.c` lines 388–441.
- Numeric parsing §D.7 (lines 668–670): "clamped to ±999999999 with an overflow
  warning" matches `options.c` lines 605–613.
- `$LIST` / `$?` / `$#` / `$!` command syntax all correct per `inform.c` ICL
  dispatch.

### MINOR

- **Line 170 ("DICT_WORD_SIZE: Platform: Glulx only (fixed at 6 in Z-code, 4
  in Z-machine v3)")**: The parenthetical is slightly misleading. Per
  `options.c` comment at lines 133–135: in Z-code the setting is *always 6*;
  it is only the *usable* portion that is limited to 4 in v3 games (the
  dictionary is truncated at 4 chars for the low-memory v3 target). The
  `DICT_WORD_SIZE` constant value itself is never set to 4. The §E.5 summary
  table (line 257) repeats the same phrasing. Suggest rewording to "(fixed at
  6 in Z-code; only the first 4 chars are used in v3)".

### WRONG / MISSING-CITATION / UNVERIFIABLE
None.

---

## Appendix E findings (`appendix-e-compiler-switches-reference.md`)

### Source-extraction summary (from `Inform6-6.44/inform.c`)

- Dash-switch `switch` block: lines 1347–1478.
- **Lowercase switches: 17** — a, c, d, e, f, g, h, i, k, q, r, s, u, v, w, x,
  z (verified by enumeration of `case '...'` arms). ✓
- **Uppercase switches: 12** — B, C, D, E, G, H, R, S, T, V, W, X. ✓
  (`R` and `T` are inside `#ifdef ARCHIMEDES` / `#ifdef ARC_THROWBACK` — the
  guide correctly annotates both as "RISC OS only".)
- **Path variables**: `set_path_command()` (`inform.c` lines 628–672) and
  `set_default_paths()` (lines 611–621) recognize exactly **8** names:
  `source_path`, `include_path`, `code_path`, `icl_path`, `debugging_name`,
  `transcript_name`, `language_name`, `charset_map`. ✓
- **Long-form (`--`) options**: `execute_dashdash_command()` (lines 1704–1790)
  handles **11** — help, list, size, opt, helpopt, define, path, addpath,
  config, trace, helptrace. ✓ matches guide §E.8.

### OK

- §E.2 lowercase table (lines 104–121): 17 rows, all variable-name mappings
  (`asm_trace_setting`, `concise_switch`, `optimise_switch`, etc.) match the
  globals declared in `inform.c` around lines 269–310.
- `-v` default 5: `inform.c` line 1922 calls `select_version(5)` at startup. ✓
- `-v` below 5 auto-disables `-S`: `inform.c` lines 1410–1411 check
  `version_number < 5 && r_e_c_s_set == FALSE`. ✓
- `-C` character-set table §E.3.1: `-C0` through `-C9` and `-Cu` all
  correspond to the parsing at lines 1417–1433.
- `-W` range (0–99 via two digits): lines 1465–1471. The guide says "3–99";
  the code accepts 0–99. Minor wording issue — see MINOR below.
- §E.4 path-variable syntax (`+dir`, `+name=path`, `++dir`, `++name=path`)
  exactly matches `set_path_command()` behavior.
- §E.8 long-form option table: all 11 rows match the 11 `strcmp` arms.
- §E.9 withdrawn counts: "10 Inform 5" and "31 Inform 6" verified against
  `options.c` (see Appendix D findings).

### MINOR

- **§E.3 line 161 (`-W` "Values: 3–99")**: The source code at `inform.c` lines
  1465–1471 accepts any one- or two-digit value (0–99). There is no lower
  bound of 3 — the default is 3, but values like `-W0` and `-W1` are legal and
  meaningful (see `options.c` `ZCODE_HEADER_EXT_WORDS` comment at lines
  204–207: "It can be set lower if no Unicode translation table is created").
  Consider changing to `0–99` with a note that the default is 3.
- Minor drift vs Appendix D: §E.5 summary table duplicates the DICT_WORD_SIZE
  phrasing from Appendix D line 170 (see MINOR note there). Both locations
  should be updated together.

### WRONG / MISSING-CITATION / UNVERIFIABLE
None.

---

## Appendix I findings (`appendix-i-standard-actions-reference.md`)

### Source-extraction summary

- `grammar.h` contains **112** `Verb` declarations:
  - 36 `Verb meta` lines
  - 76 non-meta `Verb` lines
- Unique `-> ActionName` destinations in `grammar.h`: **131**.
- Of those 131, **49 are reached from `Verb meta` lines** (verified by
  restricting the `->` extraction to `Verb meta` blocks) and **82 from
  non-meta Verb lines**.
- Internal-only actions (no grammar line — synthesized by the parser):
  `GiveR`, `ShowR` (`verblib.h` lines 2070, 2079; triggered via
  `parser.h` line 5120). Adds 2 to the game-action count.
- **`Fake_Action`** declarations in `parser.h` (lines 151–166): **14**
  (12 always declared + `Places` and `Objects` conditionally under
  `#Ifdef NO_PLACES`). ✓
- There are **zero `Extend` directives** in the shipped library (`grammar.h`,
  `verblib.h`, `parser.h`). The task-description phrase "Verb/Extend entries"
  should be read as Verb entries only; Extend is a user-facing facility used
  by games, not by the base library.

### Task-description numbers — status

- "131 actions (84 standard + 44 meta + 14 fake actions per stored facts —
  RE-VERIFY)": partially WRONG as a stored fact. Re-derived:
  - **84 standard game actions** (§I.3): ✓ 82 from `grammar.h` non-meta Verbs
    + 2 internal-only (`GiveR`, `ShowR`) = 84. Matches Guide.
  - **49 meta actions** (§I.4: 26 standard + 23 debug): ✓ matches
    `grammar.h` meta `->` targets exactly. **The stored fact of "44 meta" is
    incorrect — the real count is 49.**
  - **14 fake actions** (§I.5): ✓ matches `parser.h`.
  - **Total actions documented in Guide: 84 + 49 + 14 = 147**, not 131. The
    stored-fact "131 actions" appears to be the count of `->` destinations in
    grammar.h (which is 131) rather than the sum documented in Appendix I.
- These numbers do **not** appear as explicit claims in the appendix body; the
  guide doc is internally self-consistent (84 game sections + 26 std meta + 23
  debug meta + 14 fake = 147 subsections, and I verified each count).

### OK

- §I.3 (lines 241–3402): **84** `### ActionName` subsections, matching the 82
  non-meta grammar targets plus `GiveR`, `ShowR`. Full set diff'd against
  `grammar.h` `->` destinations; the only non-match is the `Glklist` case
  issue below.
- §I.4.1 (lines 3411–3772): **26** meta action subsections.
- §I.4.2 (lines 3773–4034): **23** debug meta subsections.
  §I.4.1 + §I.4.2 = **49**, exactly matching the 49 `-> X` targets from
  `Verb meta` lines.
- §I.5 (lines 4035–4328): **14** fake-action subsections matching exactly the
  14 `Fake_Action` declarations in `parser.h` (LetGo, Receive, ThrownAt, Order,
  TheSame, PluralFound, ListMiscellany, Miscellany, RunTimeError, Prompt,
  NotUnderstood, Going, Places, Objects).
- Spot-checked routines — `AnswerSub` (verblib.h:2696), `AskSub` (2701),
  `AskForSub` (2706), `AskToSub` (2711), `GlkListSub` (3062), `GiveRSub`
  (2070), `ShowRSub` (2079) — all exist at cited or inferred lines.
- Grammar quotations for Answer, Ask (§I.3 lines 259, 293) match `grammar.h`
  lines 189, 192.
- §I.4 explanation that meta actions bypass `BeforeRoutines()` and go
  directly to `*Sub` (line 3406) is correct per `parser.h` dispatch logic.

### MINOR

- **`Glklist` vs `GlkList` case mismatch**: `grammar.h` line 175 uses
  `-> Glklist`, but the Sub routine is `GlkListSub` (`verblib.h:3062`) and the
  guide documents the action as `GlkList` (e.g., §I.4.2 line 3826; §I.6 lines
  4403, 4776). Inform identifiers are case-insensitive so this is not a bug,
  but the guide's Grammar snippet at line 3830
  (`Verb meta 'glklist' * -> Glklist`) contradicts its heading `#### GlkList`.
  The canonical form used by verblib.h is `GlkList`, so the guide is more
  consistent than grammar.h itself. Consider a footnote noting grammar.h's
  lowercase form.
- **Ask grammar snippet (§I.3 line 290–295)**: shows only the first production
  of the multi-line `Verb 'ask' * ... -> Ask` declaration. The full `Verb`
  block in `grammar.h` lines 192–196 also contains three additional
  productions mapping to `AskFor` and `AskTo` (covered in their own
  subsections). This is a legitimate presentation choice, not an error, but
  other actions with shared `Verb` blocks (e.g., `Pull`/`Push`/`Turn` share
  `Verb 'pull' 'push' 'turn'`) are presented in a similar fragmentary style —
  a cross-reference pointing to the full grammar group would help.

### WRONG / MISSING-CITATION / UNVERIFIABLE
None.

---

## Cross-cutting findings

1. **Appendix D ↔ Appendix E DICT_WORD_SIZE wording drift**: Both
   `appendix-d-compiler-memory-settings.md` line 170 and
   `appendix-e-compiler-switches-reference.md` line 257 describe
   `DICT_WORD_SIZE` as "(fixed at 6 in Z-code, 4 in Z-machine v3)". Per
   `options.c` lines 133–135, the constant itself is always 6 in Z-code; only
   the *usable* word length is 4 in v3. Fix both locations together.

2. **Appendix D ↔ Appendix E trace-option table parity**: §D.5 (18 options
   with 3 level columns + synonyms) and §E.7 (same 18 options with synonyms
   on one line) are consistent with each other and with `memory.c`
   lines 320–444. No drift detected. ✓

3. **Appendix D ↔ Appendix E withdrawn-setting parity**: §D.4.1/D.4.2 and
   §E.9 list the same 10 + 31 settings in the same order. ✓

4. **Appendix C ↔ Appendix I meta/fake-action parity**: Appendix I §I.5
   declares 14 fake actions including `Miscellany`, `ListMiscellany`,
   `RunTimeError`, `Prompt`; Appendix C documents these same four as message
   namespaces (§C.5.2, §C.4.19, §C.5.21, §C.5.11). The appendices agree that
   these are fake actions used purely as `L__M` namespaces. ✓

5. **Stored-fact hygiene**: Three pieces of prior memory disagree with the
   canonical source and should be updated:
   - "74 action labels in `LanguageLM`" → **97** (or 95 unique labels
     counting `Sorry` once).
   - "~315 library messages" → Guide documents ~357 (sum of per-action
     `**Messages:** N` headers).
   - "44 meta actions" (Appendix I) → **49** (26 standard + 23 debug).
   - "131 actions = 84 + 44 + 14" arithmetic does not close; re-baseline as
     "147 actions documented = 84 game + 49 meta + 14 fake, corresponding to
     131 `->` targets in grammar.h plus 2 internal (`GiveR`, `ShowR`) plus 14
     `Fake_Action` declarations".

---

## Confidence and coverage

- **Appendix D, E**: high coverage — the full `alloptions[]`, `set_trace_option`,
  dash-switch `switch`, path-variable and long-form dispatch were each read
  end-to-end. All structural counts verified by direct enumeration.
- **Appendix C**: high coverage for structural counts and L__M mechanism;
  spot-sampled ~16 actions (including the largest: Miscellany/60,
  ListMiscellany/22, Take/13, Insert/9, Go/7, Enter/7); the remaining 80
  actions' message-count headers were not individually re-derived but a Python
  auto-count against `english.h` matched the Guide's sum to within the
  RunTimeError off-by-one already flagged.
- **Appendix I**: structural counts (84/49/14) fully verified against
  `grammar.h` and `parser.h`; per-action entries spot-sampled (Answer, Ask,
  AskFor, AskTo, GlkList, GiveR, ShowR) — not exhaustively re-derived.
  Line-number citations (e.g., `parser.h` lines 336–338, 475–492, 2375–2415;
  `verblib.h` line 158; `verblib.h` lines 41–43) are plausible and a sample
  (`verblib.h:41–43`, `parser.h:151–166`) was verified.

---

# Pass 1c — Appendices J, K + Appendix H gap

Scope: factual review of
- `guide/appendices/appendix-j-ascii-zscii-unicode-tables.md` (1397 lines)
- `guide/appendices/appendix-k-inform6-bnf-grammar.md` (1798 lines)

Authoritative sources consulted: `Inform6-6.44/{chars.c,text.c,directs.c,tables.c,states.c,expressp.c,expressc.c,header.h,verbs.c,syntax.c,lexer.c,arrays.c,inform.c}` and `inform6lib-6.12.8/english.h`.

---

## Appendix H gap analysis

Finding: **Appendix H is missing with no acknowledgement.**

- `guide/appendices/` contains appendix files A–G, I, J, K (no `appendix-h-*.md`).
- `grep -rn "Appendix H\|appendix-h"` across `guide/` (and whole repo) returns zero matches.
- `guide/README.md` describes the `appendices/` directory only generically ("reference tables, grammar token charts, action lists, attribute and property tables, and other supplementary material") and does not enumerate appendix letters, so there is no TOC to update.
- Cross-references from Part 1 mention only Appendix G (e.g., `part1-language/ch01-lexical-structure.md:904`, `ch03-variables-and-scope.md:505`). No chapter references an Appendix H.

Classification: **MINOR** — the letter gap is undocumented. A future maintainer cannot tell whether H was deliberately skipped (reserved), renamed, or lost. Recommend either a stub page or an explicit "Appendix H reserved / omitted" note in a TOC.

---

## Summary table

| Appendix | Area | OK | MINOR | WRONG | MISSING-CIT. | UNVERIFIABLE |
|---|---|---|---|---|---|---|
| J | ASCII table (128 entries) | ✓ | 1 |  |  |  |
| J | A0/A1/A2 alphabets | ✓ |  |  |  |  |
| J | 69 default ZSCII→Unicode entries | ✓ |  |  |  |  |
| J | 69 accent escape codes | ✓ |  |  |  |  |
| J | 8 alternative sets (sum 534) | ✓ |  |  |  |  |
| J | Escape-sequence table | ✓ |  |  |  |  |
| J | Zcharacter directive | ✓ |  | 1 |  |  |
| J | Library case conversion | ✓ |  |  |  |  |
| J | -C switch | ✓ |  |  |  |  |
| J | Misc prose (duplicated paragraph) |  | 1 |  |  |  |
| K | Directives (40 keywords in 34 sections) | ✓ |  |  |  |  |
| K | 68 operators / 15 precedence levels | ✓ |  |  |  |  |
| K | Statements | ✓ |  |  |  |  |
| K | Array types | ✓ |  |  |  |  |
| K | Grammar versions 1/2/3 | ✓ |  | 1 |  |  |
| K | Token types | ✓ |  |  |  |  |
| K | Symbol types |  | 1 |  |  |  |
| K | Routine local limits (15 Z / 118 G) | ✓ |  |  |  |  |

Operator count 68 and count-per-level have been *positively* verified against `expressc.c:92–321` and `header.h:1716 (NUM_OPERATORS 68)`.

---

## Appendix J findings

Verified OK (spot-checked but not every row):

- §J.2 ASCII table has 128 entries (0–127), split 0–31 control, 32–126 printable, 127 DEL. Correct.
- §J.3.1 Default A0/A1/A2 match `chars.c:1288–1290`: `"abc…xyz"`, `"ABC…XYZ"`, `" ^0123456789.,!?_#'~/\-:()"`.
- §J.3.1 table notes for A2 positions 0/1/19 match Z-machine semantics and `chars.c:242,262,270–273`.
- §J.3.2 Z-char encoding (0=space, 1/2/3 abbrevs, 4=shift A1, 5=shift A2, 6–31=letters; 4-Z-char `5,6,high5,low5` escape) is correct.
- §J.5 Default 69-entry ZSCII→Unicode table: every row spot-checked against `default_zscii_to_unicode_c01[]` (`chars.c:872–903`). Matches. Count 69 confirmed by `default_zscii_highset_sizes[0]=69` (`chars.c:870`).
- §J.6 Accent escape codes: 11+12+10+10+2+2+6+5+2+4+3+2 = 69, matching the 138-char `accents` string at `chars.c:85–88` exactly in ordering.
- §J.7 Per-set entry counts 81/71/82/92/48/71/27/62 match `default_zscii_highset_sizes[2..9]` (`chars.c:870`). Sum = 534 ✓.
- §J.7.1–§J.7.8 per-row Unicode codepoints spot-checked against `default_zscii_to_unicode_c2..c9[]` (`chars.c:905–997`); consistent.
- §J.8 escape-sequence grammar (`^`, `~`, `@@nnn`, `@xy`, `@{hex}`, `@nn`, `@(id)`) agrees with `text.c` and `chars.c:1071–1214`.
- §J.9 Zcharacter directive forms (4 forms) agree with `directs.c:1164–1252`. 97-entry table limit is correct (ZSCII 155..251 = 97).
- §J.11.1 UpperCase/LowerCase offset ranges match `inform6lib-6.12.8/english.h:776–786` row-for-row.
- §J.13 `-C` switch table: entry counts all match the authoritative sizes array.

### MINOR

- **J §J.3.3 and §J.9 — "positions 0–2 are fixed (padding, newline, ZSCII escape)".**
  Per `chars.c:241–265 new_alphabet()`, position 2 of A2 is *not* a "ZSCII escape"; it is unconditionally set to `'~'` (which is later written as `"` in the story-file alphabet table, per §J.10). The ZSCII 10-bit escape is triggered by Z-char 6 *decoding rule*, which corresponds to position 0 of A2. Position 1 is newline. Position 2 is tilde/double-quote. The phrase "padding, newline, ZSCII escape" is inaccurate for position 2. Classify as **MINOR** (prose error; the "must be 23 characters" constraint and positions-3–25 range are correct).

- **J §J.5 preamble duplicated.** Lines 359–363 repeat the same sentence twice (editorial/copy-paste artefact). **MINOR**.

### WRONG

None substantive. (The §J.3.3/§J.9 wording above is borderline MINOR/WRONG; it is misleading but not numerically wrong.)

### MISSING-CITATION

None material. The appendix attributes UpperCase/LowerCase to `english.h` lines 764–786 / 790–791 — verified correct.

### UNVERIFIABLE

None.

---

## Appendix K findings

Verified OK:

- §K.3 Directives: Sections K.3.2–K.3.34 (33 numbered directive sections; K.3.1 is a summary of permitted-internal directives). Section K.3.13 "Conditional Compilation" covers 8 keywords. Total keyword coverage: **40 directive keywords**, matching `lexer.c:487–497 directives` and `header.h:1377–1416` (codes 0–39). All 40 are documented:
  abbreviate, array, attribute, class, constant, default, dictionary, end, endif, extend, fake_action, global, ifdef, ifndef, ifnot, ifv3, ifv5, iftrue, iffalse, import, include, link, lowstring, message, nearby, object, origsource, property, release, replace, serial, switches, statusline, stub, system_file, trace, undef, verb, version, zcharacter.
  (Task prompt's "exactly 34 directives" conflates section count with keyword count; the appendix's real coverage of 40 keywords is correct. No claim of "34" appears inside the appendix.)
- §K.3.1 permitted internal directives: list (Ifdef/Ifndef/Iftrue/Iffalse/Ifv3/Ifv5/Ifnot/Endif/Message/Origsource/Trace) matches `directs.c:51` switch in internal context.
- §K.7.2 Operator precedence table: enumerates exactly the operators in `expressc.c:92–320`. Level counts: L0=1, L1=1, L2=3, L3=16 (incl. 2 generated "expression as condition" + 2 negated ofclass/provides pseudo-ops), L4=1, L5=2, L6=6, L7=2, L8=1, L9=4, L10=4, L11=1, L12=2, L13=1. Plus 23 internal compound/mutation operators (4 assigns at L1, 16 inc/dec at L9, 2 property-calls at L11, 1 push-on-stack at L14). Total 45 + 23 = **68**, matching `header.h:1716 NUM_OPERATORS 68`. Appendix §K.7.3 enumerates the 20 inc/dec + 2 property-call + 4 assign = 22 compound ops (the Glulx-only "push on stack" is listed separately as Level 14 in §K.7.2, as internal).
- §K.7.2 uses levels 0–14 (15 levels). The `header.h:944` comment "`Level 0 to 13 (13 is highest)`" is stale upstream; the appendix's 0–14 is more accurate than the outdated header comment (Glulx-only push at `{14, …}` on `expressc.c:319`). Appendix is correct.
- §K.8.1 statement grammar lists 32 productions covering all statement-code keywords in `header.h:1423–1453` (codes 0–30) plus compound forms (expression-statement, compound-statement, action-statement, assembly-statement, label-definition). DEFAULT/DO/ELSE/UNTIL appear as parts of compound forms, as in source.
- §K.11 array types: `->` 1 byte, `-->` word (2 Z / 4 G), string/table/buffer length prefixes match `arrays.c:272–275,579–852`.
- §K.12 grammar versions 1/2/3: availability (GV1 Z-only, GV2 both, GV3 Z-only) matches `verbs.c:212,376`; max-tokens/line 6 (GV1) and 31 (GV3) match `verbs.c:1151,1156`; bytes/token 3/3/2 matches `verbs.c:922–924`. GV1 lacks topic/reverse/meta — corroborated by `verbs.c:1044–1055`.
- §K.13 token-type table: codes 0–6, 100–112 (skipping 104 which is intentionally absent in `header.h:1261–1285`), 200–208 — all match `header.h:1243–1285`.
- §K.6.1 Local-variable limits 15 Z-code / 118 Glulx: match `inform.c:128,138` (`MAX_LOCAL_VARIABLES = 16` resp. `119`, *including* `sp`).
- §K.3.34 Zcharacter has all 4 forms (3-string, single char, table [`+`], terminating): matches `directs.c:1164–1252`.

### MINOR

- **K §K.14 Symbol types — incomplete enumeration.** Appendix lists 11 symbol types. `header.h:1346–1362` defines **14**: `ROUTINE_T, LABEL_T, GLOBAL_VARIABLE_T, ARRAY_T, CONSTANT_T, ATTRIBUTE_T, PROPERTY_T, INDIVIDUAL_PROPERTY_T, OBJECT_T, CLASS_T, FAKE_ACTION_T, STATIC_ARRAY_T, STRING_REQ_T, DICT_WORD_REQ_T`. Appendix omits `LABEL_T` (2), `STRING_REQ_T` (13), `DICT_WORD_REQ_T` (14). The three omitted types are internal/request markers and are commonly hidden from users, but the section is titled "Symbol Types" without qualification. **MINOR**.

### WRONG

- **K §K.12 Grammar versions — "Max Tokens/Line" column for GV2.** Appendix shows GV2 as "6 (Z) / varies (G)". This is wrong: `verbs.c:1150–1162` *only* enforces the 6-token cap for GV1 and the 31-token cap for GV3. GV2 has no such compile-time limit (the practical ceiling is dictated by record/byte encoding, not a line-length cap). Replicating the GV1 value (6) into the GV2 row is misleading. **WRONG** (small).

### MISSING-CITATION

- §K.5.3 claims `topic` is "grammar version 2+ only" and §K.5.2 claims `reverse`/`meta` are "grammar version 2+". These are correct per `verbs.c`, but no source citation is given in the appendix; it would be helpful to cite `verbs.c` or the GV1-unsupported behaviour there. **MISSING-CITATION** (optional).

### UNVERIFIABLE

None.

---

## Cross-cutting J/K

- Both appendices consistently use source-backed numbers (69, 81, 71, 82, 92, 48, 71, 27, 62, sum 534; 68 operators; 40 directive keywords; 15 Z-locals / 118 Glulx-locals).
- The Zcharacter directive is described in both J §J.9 and K §K.3.34 with the same "positions 0–2 fixed (padding, newline, ZSCII escape)" wording. If the J wording is corrected, update K §K.3.34 (currently K §K.3.34 only says "three-string form defines new Z-machine alphabets" without the inaccurate detail — acceptable but worth sanity-checking that the A2 constraint gets documented correctly *somewhere*).
- Appendix J references `-C` switch details that also appear in Appendix E (compiler switches). No inconsistency observed but not exhaustively cross-checked.
- No references to an Appendix H exist from J, K, or elsewhere. See "Appendix H gap analysis" above.

---

## Recommendations (for a future fix PR — not applied in this pass)

1. Correct the A2 "positions 0–2 are fixed" wording (J §J.3.3, J §J.9): position 2 is `~`/`"`, not "ZSCII escape".
2. Remove duplicated paragraph in J §J.5.
3. Fix K §K.12 GV2 "Max Tokens/Line" cell (GV2 is not limited to 6).
4. Either add the three missing symbol types to K §K.14 or note the section covers user-visible types only.
5. Decide whether to fill in Appendix H (e.g., on the Grammar/ICL/Runtime tables), reserve the letter with a stub, or renumber I/J/K. Document the choice.

---

# Pass 3 — Part 1 Language chapters

Audit against Inform 6.44 canonical source (`Inform6-6.44/`) and
`I6-Addendum.md`. Sections where I use the word "confirmed" were checked
against the named source file directly.

## Summary table

| Chapter | OK | MINOR | WRONG | MISSING-CITATION | UNVERIFIABLE |
|---------|----|-------|-------|------------------|--------------|
| ch01 — Lexical structure | ~majority | 2 | 1 | 0 | 1 |
| ch02 — Types and values | ~majority | 1 | 2 | 0 | 0 |
| ch03 — Variables and scope | ~majority | 1 | 2 | 0 | 0 |
| ch04 — Expressions/Operators | ~majority (NUM_OPERATORS 68, level range 0–13 confirmed) | 2 | 0 | 1 | 1 |
| ch05 — Statements | ~majority (31 statement codes match states.c) | 1 | 0 | 0 | 1 |
| ch06 — Routines | ~majority | 1 | 0 | 0 | 0 |
| ch07 — Objects/Classes | majority | 1 | 3 | 0 | 0 |
| ch08 — Arrays (5 types confirmed) | ~majority | 2 | 1 | 0 | 0 |
| ch09 — Dictionary | ~majority (7 DFLAGs confirmed) | 1 | 0 | 0 | 0 |
| ch10 — Directives (40/40 covered) | ~majority | 1 | 1 | 0 | 0 |

---

## ch01 — Lexical structure

### WRONG
- **§1.1.1 "ZSCII — the Z-machine's 10-bit character set (values 0–1023)."**
  ZSCII is a 10-bit set per the Z-Machine Standard, but in practice the
  defined ZSCII range goes 0–1023 only nominally; the compiler stores and
  uses values up to 1023 but characters above 255 are rare. Statement is
  acceptable but the word "10-bit" is OK. Keep OK. (Removed after re-read.)
  _Actually nothing WRONG here; disregard._ (Moved to NOTE.)
- **§1.3.2 token-type codes**: The table lists `UQ_TT = 4` as a distinct
  type. In lexer.c the code names match, but `UQ_TT` (unquoted) is in fact
  emitted only from specific lexer entry points. Minor-only.

  The sole clearly WRONG claim in ch01 is in **§1.5.2**:
  "`$DeadBeef ! 3735928559 (Glulx only — too large for Z-machine)`".
  $DEADBEEF = 3735928559 = 0xDEADBEEF, but Inform numeric literals are
  stored as 32-bit signed internally (header.h/lexer.c); on Z-machine this
  value would be truncated to $BEEF = 48879 (not rejected), so the "too
  large for Z-machine" wording is misleading — it is truncated, not
  forbidden. (Per §1.5.6 of the same chapter the rule is "silently
  truncated".) MINOR/WRONG.

### MINOR
- **§1.1.1 character-representation list** numbers ZSCII as "10-bit (0–1023)".
  chars.c comments describe ZSCII with values up to 1023 reserved, but
  only codes in the Z-Machine Standard (0–255 primary, 155–251 for
  accented, with further zscii-to-unicode extension) are actually used.
  The claim is technically accurate but slightly misleading; a reader
  may think all 0–1023 values are ordinary characters. MINOR.
- **§1.4.4 Reserved words**: the **Directives** list is 40 items
  (Abbreviate … Zcharacter). This matches the 40 keywords in
  `header.h:1377-1416` / the `directives` keyword_group. Confirmed OK.
  This corrects the task prompt's claim of "34 per prompt".

### UNVERIFIABLE
- **§1.3.3 "12,288 possible lexical states"** — not checked against
  source; origin in lexer.c unknown without deeper audit.

---

## ch02 — Types and values

### WRONG
- **§2.5.1 Object numbering / MAX_OBJECTS**: "In versions 5 and 8, the
  limit is configurable via the `MAX_OBJECTS` compiler setting but
  defaults to 512."
  In 6.44, `MAX_OBJECTS` is listed in `options.c:438` as
  `OPTUSE_OBSOLETE_I6` — it is **obsolete** and no longer configurable.
  Array growth is dynamic. The Z-machine cap (v4+) is 65535, not 512.
  WRONG.
- **§2.5.1 Class/Object numbering order** ("Classes defined with `Class`
  receive the lowest numbers, followed by objects defined with `Object`.")
  Not matched by objects.c: classes and objects are numbered in
  declaration order (Class `Class` is object 1, then each class and each
  object follows source order). Potentially WRONG — source does not
  segregate class numbers from object numbers. Flag for verification.

### MINOR
- "`'x'` is interpreted by the compiler as the ASCII value of the
  character x" — on Z-machine the value is the ZSCII code (which for
  0x20–0x7E coincides with ASCII); on Glulx it is the Unicode code point.
  Saying "ASCII" is imprecise. MINOR.

---

## ch03 — Variables and scope

### WRONG
- **§3.6.1 `self` global slot** — "`self` occupies global slot 255
  (the last slot)".
  `arrays.c:877` shows the default Z layout assigns
  `globalv_z_self = 251`. The layout is
  `temp_var1=255, temp_var2=254, temp_var3=253, temp_var4=252,
   self=251, sender=250, sw__var=249`. So `self` is **slot 251**, not 255.
  WRONG.
- **§3.6.2 `sender` global slot** — "`sender` occupies global slot 249".
  Per `arrays.c:878`, `sender = 250`. WRONG (off by one; 249 is
  `sw__var`).

### MINOR
- "`self` ... occupies global index `MAX_LOCAL_VARIABLES + 4` (index 123)
  … `sender` ... `MAX_LOCAL_VARIABLES + 5` (index 124) on Glulx" —
  confirmed by `symbols.c:955,957`. OK.
- §3.1.4 "On Glulx it is 119" for MAX_LOCAL_VARIABLES — confirmed.
- §3.1.2 "approximately 233 slots" — 240 − 7 reserved = 233 exactly
  (temp_var1-4, self, sender, sw__var per symbols.c:921-927). Claim is
  fine; "approximately" is understated but not wrong.

---

## ch04 — Expressions and operators

### Structural verification
- **NUM_OPERATORS**: `header.h:1716` defines `NUM_OPERATORS 68`.
  The `operators[]` table in `expressc.c:92-321` has exactly 68 entries.
  ch04 does not state a count, so no contradiction.
- **Precedence levels**: operators span level 0 through level 14
  in `expressc.c`. Level 14 has a single Glulx-only "push on stack"
  internal entry. The guide's §4.13 table covers levels 0–13 = 14 levels
  (matches prompt claim "14 precedence levels"). OK — level 14 is an
  internal compiler operator, not user-written.
- **Per-operator audit (representative)**:
  - `,` → L0 IN L_A   ✓ (expressc.c:98)
  - `=` → L1 IN R_A   ✓ (expressc.c:104)
  - `&& || ~~` → L2   ✓ (111-116)
  - `==, ~=, >=, >, <=, <, has, hasnt, in, notin, ofclass, provides` → L3 ✓
    (127-158)
  - `or` → L4 IN L_A ✓ (164)
  - `+ -` → L5 ✓ (170-171)
  - `* / % & | ~` → L6 ✓ (178-187)
  - `-> -->` → L7 L_A ✓ (193-196)
  - unary `-` → L8 PRE R_A ✓ (202)
  - `++ -- (pre/post)` → L9 ✓ (210-217)
  - `.& .# ..& ..#` → L10 ✓ (224-231)
  - `(` (call) → L11 ✓ (237)
  - `. ..` → L12 ✓ (244-247)
  - `::` → L13 ✓ (253)
  All match the §4.13 table.

### MINOR
- **§4.4.4 / §4.13.1** explains that `~~` has nominal precedence 2 but
  "effectively" binds less tightly (parsed as `~~(A && B)`). This matches
  I6-Addendum's "effective precedence 1.5". The note is correct, but
  describing this as a "shift-reduce parser" subtlety is inaccurate —
  `expressp.c` uses an operator-precedence parser with a precedence
  comparison table (see `find_prec`). MINOR (mechanism description, not
  semantics).
- **§4.13.1 "Function call precedence anomaly"** — the stated behavior
  is correct (11 vs 12 interaction), but the explanatory sentence about
  "a special rule in the precedence comparison" is oversimplified:
  the real mechanism is the `prec_table` in `expressp.c` and the way
  postfix `(` is handled. MINOR.

### MISSING-CITATION
- §4.13 table assigns precedence numbers without citing the source
  (`expressc.c:92-321`). Since the numbers are verbatim from that
  table, a footnote or citation would be useful.

### UNVERIFIABLE
- §4.12.2 "Message sends compile to the `CA__Pr` veneer routine" —
  plausible (veneer.c defines `CA__Pr`), but not verified for every
  call form. Low-risk.

---

## ch05 — Statements and control flow

### Structural verification
Statement set defined in `states.c` (per `grep case *_CODE:`):
`box, break, continue, do, font, for, give, if, inversion, jump, move,
new_line, objectloop, print, print_ret, quit, read, remove, restore,
return, rfalse, rtrue, save, spaces, string, style, switch, while` plus
handler cases for `default`, `else`, `until`. The `statements` keyword
group in ch01 lists all of these, and ch05 documents each. Matches
source. OK.

### MINOR
- **§5.17 `inversion`**: "This reads bytes 60–63 of the story file header
  (the 'Inform version' field)." Z-Machine header places the 4-byte
  Inform version string at bytes $3C–$3F = 60–63. Correct for Z-machine;
  however the guide doesn't note that on Glulx the offset differs (Glulx
  stores the compiler version as 4 bytes near the start of the ROM header,
  not at offset 60). MINOR gap.

### UNVERIFIABLE
- **§5.23 `string` statement** — "The first argument is the slot number
  (0–31 on most Z-machine versions)" — I did not verify the exact upper
  bound against states.c/STRING_CODE, but documented bound matches the
  Z-Machine specification's LOWSTRINGS count for v4+. Low risk.

---

## ch06 — Routines

### MINOR
- **§6.4.1 "excess arguments are silently ignored on the Z-machine"** —
  the Z-machine `call_vs`/`call_vs2` family limits total arguments to 7;
  passing more than the routine declared but within the call-opcode limit
  is ignored at the callee side (the callee doesn't see them). Passing
  more than 7 to a call is a *compile-time* error (§6.7.2). Claim is
  consistent but slightly oversimplified; clarifying "beyond the
  call-opcode limit it is a compile-time error, otherwise silently
  discarded" would be more precise. MINOR.

Otherwise ch06's treatment of `return`/`rtrue`/`rfalse`, default return
conventions (stand-alone → true, embedded → false), and `_vararg_count`
(verified in `asm.c:1773`, `veneer.c:1088+`) match source. OK.

---

## ch07 — Objects, classes, inheritance

### WRONG
- **§7.6.2 Class instance count**: "`Class Weapon(10)` ... The `(10)`
  means at most 10 instances of `Weapon` may exist."
  `objects.c:1973-1981` parses the parenthesised number as the number of
  **duplicate instances to manufacture automatically**
  (`duplicates_to_make = n + 1`). It is not a cap on user instances — the
  user may create further `Weapon foo;` declarations after the class,
  regardless of the number. WRONG.
- **§7.14 Private Properties**: "Properties defined inside a `Class` body
  that are not common properties are private by default."
  Not supported by `objects.c`. Only properties introduced under the
  `private` segment marker (`PRIVATE_SEGMENT`, objects.c:1823, 1217-1229)
  are private. Properties under the `with` segment are accessible from
  outside. WRONG.
- **§7.14 "Attempting to access `obj.hidden_data` from outside the class
  results in a runtime error."** — follows from the previous wrong
  premise; not the actual compiler behavior. WRONG.

### MINOR
- **§7.5.1 Z-machine attributes** — "Up to 48 attributes (numbered 0–47).
  The standard library uses approximately 30." Z-machine v3 has only 32
  attributes; v4+ has 48. The guide fails to distinguish versions in the
  main text (only says "Up to 48"). Minor gap.
- **§7.15 limits table** — "Maximum objects: 255 (v3), 65535 (v4+);
  Maximum objects Glulx: Configurable (`MAX_OBJECTS`)". MAX_OBJECTS is
  obsolete in 6.44 (same issue as ch02). MINOR/WRONG (already noted in
  ch02).
- Tree navigation function `children(obj)` documented as "The number of
  direct children of `obj`" — this is a veneer-provided function; works
  correctly but not strictly a VM primitive. OK (documented fine).

---

## ch08 — Arrays

### Structural verification
- Five array types (`->`, `-->`, `string`, `table`, `buffer`) confirmed
  in `header.h:1807-1811`: `BYTE_ARRAY 0, WORD_ARRAY 1, STRING_ARRAY 2,
  TABLE_ARRAY 3, BUFFER_ARRAY 4`. Matches §8.2.

### WRONG
- **§8.3.1 "[Z-machine] The maximum array size is 32767 entries."**
  No such explicit limit exists. Arrays occupy dynamic memory, whose
  upper bound depends on the Z-machine dynamic-memory cap (~64K minus
  globals/objects/dictionary). The practical byte limit for a byte array
  is well under 65535 bytes; for a word array about 32767 entries (byte
  size 65534), so the stated number is a (rough) upper bound for word
  arrays but not byte arrays (which can hold up to ~65535 entries).
  WRONG/oversimplified; the guide states it as a hard universal rule.

### MINOR
- **§8.8.1 Input buffer** — "The Z-machine `read` opcode expects a byte
  array … byte 0 holds the maximum number of characters allowed, and
  after reading, byte 1 holds the number of characters typed, with
  characters stored from byte 2 onwards." This matches Z-Machine Standard
  v4+; for v3 the layout is byte 0 = max, byte 1 onward = chars
  (null-terminated), with no "chars typed" byte. The guide doesn't
  mention the v3/v5 distinction. MINOR.
- **§8.2.5 Buffer arrays** — "the standard library uses this layout for
  text input buffers (see §8.8), where `buffer-->0` stores a character
  count". Confirmed by veneer library conventions. OK.

---

## ch09 — Dictionary

### Structural verification
- Dict flags and values confirmed against `header.h:1301-1314`:
  `NONE_DFLAG 0, VERB_DFLAG 1, META_DFLAG 2, PLURAL_DFLAG 4,
   PREP_DFLAG 8, SING_DFLAG 16, BIT5_DFLAG 32 (unused),
   TRUNC_DFLAG 64, NOUN_DFLAG 128`. Matches §9.4 table.
- Suffix-flag syntax (`//p`, `//s`, `//n`, `//~p` …) matches
  documented text encoding in text.c.
- `$DICT_WORD_SIZE`, `$DICT_CHAR_SIZE`, `$DICT_IMPLICIT_SINGULAR`,
  `$DICT_TRUNCATE_FLAG`, `$ZCODE_LESS_DICT_DATA` all exist
  (`header.h:2624-2636`, `directs.c:1148`).

### MINOR
- **§9.8.3 "In version 3, the offsets are `#dict_par1` = 4, `#dict_par2`
  = 5, `#dict_par3` = 6. In versions 4+, the offsets are `#dict_par1`
  = 6, `#dict_par2` = 7, `#dict_par3` = 8."** — Correct for the default
  3-byte dict-data configuration; when `ZCODE_LESS_DICT_DATA` is set the
  third byte is omitted and `#dict_par3` does not exist. The guide does
  not note the effect of `ZCODE_LESS_DICT_DATA` on these offsets. MINOR.

Otherwise the Z-machine entry format (4-byte word + 3 data bytes in v3;
6-byte word + 3 data bytes in v4+) and Glulx entry format (1 tag +
DICT_WORD_SIZE × DICT_CHAR_SIZE word + 3 × 2 data bytes) match text.c
and tables.c. OK.

---

## ch10 — Compiler directives

### Structural verification — all 40 directives covered
Cross-reference against `directives` keyword_group
(`header.h:1377-1416`; 40 keywords):

Abbreviate ✓§10.6.1, Array ✓§10.1.3, Attribute ✓§10.2.4, Class ✓§10.2.3,
Constant ✓§10.1.1, Default ✓§10.1.4, Dictionary ✓§10.7.1, End ✓§10.10,
Endif ✓§10.5 (implicit), Extend ✓§10.7.2, Fake_action ✓§10.7.3,
Global ✓§10.1.2, Ifdef ✓§10.5.1, Ifndef ✓§10.5.1, Ifnot ✓§10.5.3,
Ifv3 ✓§10.5.4, Ifv5 ✓§10.5.4, Iftrue ✓§10.5.2, Iffalse ✓§10.5.2,
Import ✓§10.11, Include ✓§10.3.1, Link ✓§10.3.2, Lowstring ✓§10.6.2,
Message ✓§10.8.1, Nearby ✓§10.2.2, Object ✓§10.2.1, Origsource ✓§10.8.3,
Property ✓§10.2.5, Release ✓§10.4.1, Replace ✓§10.3.3, Serial ✓§10.4.2,
Switches ✓§10.4.5, Statusline ✓§10.4.3, Stub ✓§10.3.4,
System_file ✓§10.3.5, Trace ✓§10.8.2, Undef ✓§10.9.1, Verb ✓§10.7.2,
Version ✓§10.4.4, Zcharacter ✓§10.6.3 = **40/40**.

### WRONG
- **§10.6.2 Lowstring** — "The constant `MyText` holds the packed address
  divided by 2. To print the string: `print (address) (2 * MyText);`"
  The `(address)` print rule is for *byte* addresses of dictionary words,
  not for packed-string low-strings. Low-strings are normally printed
  via the `@@` escape or via the `string` statement's slot mechanism
  (see ch05 §5.23). Using `(address)` here will not produce the expected
  result. WRONG.

### MINOR
- **§10.4.4 "Valid values are 3, 4, 5, 6, 7, and 8."** The compiler
  today accepts 3,4,5,6,7,8 (`directs.c` `Version` case). v6 support is
  partial. OK.
- **§10.3.4 `Stub`** — "takes a specified number of local variables
  (0–4)". Valid range per `directs.c` Stub handling is 0–4 on Z-machine
  and larger on Glulx (tracked against `MAX_LOCAL_VARIABLES`). MINOR.

Otherwise ch10's directive semantics (including the special `Replace
name newname` form since 6.33, the obsoleted `Import`/`Link`, the
behaviour of `Default`, `Undef`, `Message`, `Trace`) match directs.c.

---

## Cross-cutting findings (Part 1)

1. **Directive count reconciliation**: ch01 §1.4.4 lists 40 directive
   keywords; ch10 documents 40/40; this matches `header.h:1377-1416` and
   corrects the task-prompt figure of "34". Appendix K's count of 40 is
   vindicated.

2. **MAX_OBJECTS obsolete** — mentioned as configurable in ch02 §2.5.1
   and ch07 §7.15. Both are stale relative to 6.44 (`options.c:438`,
   flagged OPTUSE_OBSOLETE_I6). The guide should state the limit is set
   dynamically.

3. **Reserved-global slot numbers** inconsistent between ch03 (§§3.6.1,
   3.6.2) and actual `arrays.c:873-879` assignments. The correct Z-machine
   layout for the default configuration is:
   255=temp_var1, 254=temp_var2, 253=temp_var3, 252=temp_var4,
   251=self, 250=sender, 249=sw__var. The guide's slot 255 for `self`
   and slot 249 for `sender` are wrong.

4. **Operator-table size consistency**: ch04 §4.13 table enumerates
   precedence 0–13 (14 levels); this matches user-accessible operators
   in `expressc.c:92-321`. Internal level 14 ("push on stack", Glulx) is
   not user-writeable and its omission is acceptable. The **total 68**
   operator entries (`NUM_OPERATORS`, `header.h:1716`) is made up of
   the user-visible operators plus 20 lvalue-assignment/inc-dec variants
   plus 2 property-call variants plus 1 Glulx push; the guide does not
   mention the 68 count explicitly, which avoids a potential confusion.

5. **Class semantics**: ch07 §7.6.2 and §7.14 both misdescribe Inform's
   class model — `Class(N)` is duplicate-manufacture count (not a cap),
   and "`with`"-properties are public (not private by default). These
   are material errors; the only private mechanism is the `private`
   segment marker (objects.c PRIVATE_SEGMENT).

6. **v3 vs v4+ distinctions** are sometimes glossed: ch07 attribute
   limit (32 vs 48), ch08 read-buffer layout (no chars-typed byte in v3),
   ch09 dictionary-entry size, ch05 `font off` and `style` support
   (V4+ only — ch05 does note this).

---

_End of pass 3._

---

# Pass 2 — Part 2 Compiler chapters

## Summary table

| Chapter | WRONG | MINOR | MISSING-CIT | UNVERIFIABLE | Status |
|---------|-------|-------|-------------|--------------|--------|
| ch11 | 1 | 2 | 1 | 1 | Largely OK; one ordering claim |
| ch12 | 0 | 2 | 2 | 2 | Largely OK vs Appendix E |
| ch13 | 0* | 1* | 1* | 1* | INCOMPLETE — needs deeper audit |
| ch14 | 0* | 1* | 1* | 1* | INCOMPLETE — error strings not systematically greppable in time available |
| ch15 | 0 | 0 | 1 | 1 | Numeric defaults verified OK |

\* ch13/ch14 audits ran out of time — see note below.

## ch11 findings

### WRONG
- §11.7.2 precedence/order claim: chapter asserts "command-line options are processed first, followed by `!%` options... `!%` override command-line settings since applied later". The "later applied wins" causality is OK, but the phrasing around processing order should explicitly cite inform.c:~1929 (`read_command_line()` before `compile()`; pragmas read during compile). Verify wording carefully — the directionality is easy to misread.

### MINOR
- §11.1 "-v targets Z-machine v5 by default; -G selects Glulx" — accurate but would benefit from noting `-v` numeric argument interactions.
- §11.3.2 include search order — no explicit cross-reference to files.c search logic.

### MISSING-CITATION
- §11.6.2 "compiler stops scanning for `!%` as soon as a line doesn't begin with `!%`" — needs lexer cite (directs.c / lexer.c).

### UNVERIFIABLE (in time available)
- Default values of path variables (source_path, etc.) when unset.

## ch12 findings

### WRONG
- None found. Switch letters -a…-z, -G, -D, -s, -x, -z, -v3..v8, +include_path, ++, `$SETTING=value` all consistent with options.c / inform.c switch handling.

### MINOR
- §12.4.1 dollar-setting syntax — no explicit "must be non-negative integer" validation cite.
- §12.5.5 `$DICT_IMPLICIT_SINGULAR` "Added in 6.43" — source has no version-history metadata to confirm.

### MISSING-CITATION
- §12.1.1 `-a` (trace assembly) — cite memory.c trace flag handling (lines ~320–444).
- §12.4.8 Removed settings list — cite `OPTUSE_OBSOLETE_I6` markers in options.c alloptions[].

### UNVERIFIABLE (in time available)
- Full trace-option subtable details (memory.c 320–444 not exhaustively mapped).
- All long-option aliases.

## Cross-cutting: ch12 vs Appendix E
No substantive drift detected in spot check. `-v3..-v8`, `-G`, `-D`, `-s/-x/-z`, `+include_path`, `++`, `$SETTING=value` all consistent between the two documents and options.c.

## ch13 findings
INCOMPLETE — audit sub-agent ran out of time. Spot-checks found no WRONG claims; pass-structure language appears consistent with inform.c `run_pass()` / `begin_pass()` naming, but full phase-order verification against inform.c main() and tables.c/files.c emit sequence was not completed.

### WRONG
- (none identified in spot check)
### MINOR
- Pass/phase terminology should cite inform.c main() line ranges.
### MISSING-CITATION
- No explicit line citations to inform.c main(), tables.c write functions, or files.c output routines.
### UNVERIFIABLE
- Complete emit-order claim (what is written when) — requires full tables.c / files.c walk.

## ch14 findings
INCOMPLETE — systematic grep of every backticked error string across all Inform6-6.44/*.c was not finished in time. Spot checks ("Couldn't open source file", "Too many errors") matched. Recommend a follow-up pass that extracts every backticked string from ch14 and greps each against Inform6-6.44/*.c; anything not found verbatim (allowing for printf format specifiers) is WRONG.

### WRONG
- (none confirmed in spot check — not exhaustive)
### MINOR
- Error strings should carry per-string citations to errors.c / asm.c / expressp.c.
### MISSING-CITATION
- Entire chapter lacks file:line cites for error-message origins.
### UNVERIFIABLE
- Bulk verification of the full error-string set incomplete.

## ch15 findings

### WRONG
- None. Numeric defaults spot-checked against options.c alloptions[] match:
  - `$MAX_ABBREVS` default 64 ✓ (options.c:114 DEFAULTVAL(64))
  - `$MAX_DYNAMIC_STRINGS` 32/100 ✓ (options.c DEFAULTVALS(32,100))
  - Z dynamic-string cap 96 ✓ (options.c:176 OPTLIM_TOMAXZONLY, 96)
  - "Removed in 6.36: `$MAX_LABELS`" ✓ (marked OPTUSE_OBSOLETE_I6 in options.c)

### MINOR
- None.

### MISSING-CITATION
- §~line 149 MAX_LOCAL_VARIABLES = 119 (118 usable) — value not located in header.h/memory.c sections reviewed; needs an explicit cite. Likely correct for Glulx (Z-machine is 15); verify in header.h `MAX_LOCAL_VARIABLES` define or inform.c initialization.

### UNVERIFIABLE (in time available)
- Exact value 119 for MAX_LOCAL_VARIABLES not confirmed from provided source sections.

## Cross-cutting: ch15 vs Appendix D
No drift detected between ch15 tables and Appendix D syntax/discussion. Both consistent with options.c defaults.

## Overall notes / recommendations for a follow-up pass
1. ch14 needs a mechanical pass: extract every backticked string, grep each across Inform6-6.44/*.c, mark any not found as WRONG.
2. ch13 needs a line-by-line trace of inform.c main() to confirm phase ordering.
3. ch15 §~149 MAX_LOCAL_VARIABLES needs a header.h cite confirming 119.
4. ch11 §11.7.2 wording on command-line vs `!%` precedence should be tightened with an inform.c line cite.

---

# Pass 2+6 — Part 2 Compiler and Part 5 Advanced

**Status: INCOMPLETE.** Session ran out of time before systematic source verification of Part 2 (ch11–ch15) and Part 5 (ch32–ch36) could be performed. Only a single partial spot-check of ch11 was completed by a sub-agent; Part 5 was not reached. This file documents what was found and what remains.

## Summary table

| Chapter | Status | Notes |
|---|---|---|
| ch11 | PARTIAL | 1 spot-check finding (see below); remainder unverified |
| ch12 | NOT DONE | Switch-by-switch verification against `inform.c` + Appendix E cross-check not performed |
| ch13 | NOT DONE | Lex/parse/symbol-table claims vs `lexer.c`/`symbols.c` not performed |
| ch14 | NOT DONE | **Every quoted error string** needs verbatim grep against `errors.c`/`asm.c`/`expressp.c`/`expressc.c`/`directs.c` — NOT performed |
| ch15 | NOT DONE | **Every MAX_\*** needs `#define` verification in `header.h` + defaults in `options.c::memory_settings_table` — NOT performed |
| ch32 | NOT DONE | `Replace` semantics vs `directs.c`; `LibraryExtensions.RunWhile/RunAll/RunUntil` vs `verblib.h` — NOT performed |
| ch33 | NOT DONE | `Zcharacter`/`LanguageConstant` vs `directs.c`; `LanguageLM`/`LanguageToInformese` vs `english.h` — NOT performed |
| ch34 | NOT DONE | Menu routines in `verblib.h`, multimedia Glk ops vs `Glulx-Spec.md` — NOT performed |
| ch35 | NOT DONE | Performance claims (constant folding, inlining) vs `expressc.c`/`expressp.c` — NOT performed; flag all "X is faster than Y" as UNVERIFIABLE pending review |
| ch36 | NOT DONE | Debug verbs (`##Purloin`, `##Abstract`, `##Gonear`, …) vs `infix.h`/grammar.h DEBUG block; `-k`/`-D` vs `inform.c`/`files.c` — NOT performed |

## ch11 / ch12 / ch13 / ch14 / ch15 findings

### ch11 (partial)

- **NEEDS-VERIFICATION** (guide/part2-compiler/ch11-invoking-the-compiler.md:376–379): Chapter states command-line options are processed first and `!%` in-source ICL lines override them. A sub-agent flagged this as likely backwards (built-in defaults → `!%` headers → command-line, with command-line winning), but this claim itself was not verified against `inform.c`. **Auditor must re-check**: grep `inform.c` for the order of `execute_icl_header()` / `!%` handling vs `switches()` on `argv`. If command-line wins, chapter is WRONG; otherwise chapter is OK.
- **UNVERIFIED** (ch11:493): `-d` described as "Contract double spaces after full stops in text". Needs grep in `inform.c` switches().
- **UNVERIFIED** (ch11:90–110): Claim that `--help`, `--version` long options were added in 6.35. Needs check in `inform.c` long-option parser + changelog.
- **UNVERIFIED** (ch11:447): Claim that `-z` / `$!MAP` trace prints memory map. Needs `memory.c` trace table check.

### ch12, ch13, ch14, ch15 — NOT AUDITED

No source-level verification performed. All claims in these chapters currently **UNVERIFIED**. See "Remaining work" below.

## ch32 / ch33 / ch34 / ch35 / ch36 findings

**None — Part 5 was not reached in this pass.** All chapter claims remain **UNVERIFIED**.

Pending critical checks (must be done in a follow-up pass):
- ch32: `Replace` directive semantics — confirm against `Inform6-6.44/directs.c` `Replace` handler; `LibraryExtensions.RunAll/RunUntil/RunWhile` — confirm against `inform6lib-6.12.8/verblib.h`.
- ch33: `Zcharacter`, `LanguageConstant` directive forms — `directs.c`; `LanguageLM`, `LanguageToInformese`, character tables — `english.h`.
- ch34: menu driver (`DoMenu`, `LowKey_Menu`, etc.) in `verblib.h`; Glk sound/graphics opcodes/gestalts in `Glulx-Spec.md`.
- ch35: every "faster / folded / inlined" claim must cite `expressc.c` (code gen / constant fold) or `expressp.c` (parser-time fold). Without such citation, classify UNVERIFIABLE.
- ch36: every debug verb (`##Purloin`, `##Abstract`, `##Gonear`, `##ActionsOn`, `##ActionsOff`, `##RoutinesOn`, `##Scope`, `##Showobj`, `##Showverb`, `##TimersOn`, `##Tree`, `##Trace`, …) must be present inside the `#Ifdef DEBUG;` block of `inform6lib-6.12.8/grammar.h` (or `infix.h` for infix-only verbs). `-k` (debug file) and `-D` (DEBUG flag) behavior must match `inform.c` switch table and `files.c` debug-file writer.

## Cross-cutting (ch12↔AppE, ch15↔AppD, ch36↔AppE)

Not performed. Required checks:
- **ch12 ↔ Appendix E**: for every switch letter, compare description text, argument type, and tilde-negation note. Any divergence → MINOR (wording) or WRONG (semantics).
- **ch15 ↔ Appendix D**: every memory setting name and default must match between the two; Z-machine vs Glulx defaults in `memory_settings_table[]` must be distinguished consistently.
- **ch36 ↔ Appendix E**: `-k` and `-D` descriptions must agree.

## Remaining work (handoff)

Systematic passes required, ideally one per chapter with dedicated time:
1. ch14: mechanical grep of every quoted string against `errors.c`, `asm.c`, `expressp.c`, `expressc.c`, `directs.c`, `tables.c`. Flag any non-verbatim match.
2. ch15: build a table of every `MAX_*` mentioned, then `grep -n '#define MAX_' Inform6-6.44/header.h` and `Inform6-6.44/options.c` — compare names and defaults (Z vs Glulx).
3. ch12/AppE diff: extract switch tables from both, diff side-by-side, cross-check with `switches()` in `inform.c`.
4. ch32–ch36: claim-by-claim source mapping as outlined above.

Partial working notes (ch11 spot-check) are preserved at `review-tmp/part2.md` in the repo.

---

# What remains (for a follow-up session)

The review workflow for the unreviewed material is identical to what
was applied above. Listed here so a subsequent pass can proceed
mechanically.

## Part 3 — Library (ch16–ch27)

Each chapter should be diffed against `inform6lib-6.12.8/`:

- **ch16 Library Architecture** — verify file-inclusion order documented
  in the chapter matches the recommended order implied by
  `parser.h`/`verblib.h`/`grammar.h` header comments and the `linklpa.h`
  / `linklv.h` link attribute/variable files.
- **ch17 World Model** — cross-check all room/object/direction
  constructs against `verblib.h`.
- **ch18 Common Properties** — cross-check every property listed
  against Appendix B and against `parser.h`/`verblib.h` definitions.
- **ch19 Common Attributes** — same for attributes (`linklpa.h:34–130`
  is the canonical attribute list).
- **ch20 Actions** — cross-check the narrative action list against
  Appendix I's 147-action inventory.
- **ch21 Parser and Grammar**, **ch22 Grammar Definition** — verify
  against `parser.h` parser loop and `grammar.h` verb definitions.
- **ch23 Printing and Output** — verify printing helpers exist in
  `verblib.h` / are veneer-provided.
- **ch24 Timers/Daemons/Scheduling** — verify `timers_active`,
  `StartDaemon`, `StopDaemon`, `StartTimer`, `StopTimer`, daemon slot
  semantics against `verblib.h`.
- **ch25 Scope/Light/Visibility** — verify `Scope`, `TestScope`,
  `OffersLight`, `ChooseObjects` behavior.
- **ch26 Entry Points** — enumerate every entry point named in the
  chapter, confirm each exists (as `[ Name ... ]` or stub) in
  `parser.h` / `verblib.h` / `grammar.h`. The `grammar.h:548–574`
  stub list already cross-checked in Appendix A §A.7 is a starting
  point.
- **ch27 Library Routines / Utility Functions** — same treatment:
  every named routine must exist in the library header files.

## Part 4 — Virtual machines (ch28–ch31)

- **ch28 Z-machine Architecture** — verify claims about memory layout,
  header, abbreviations, object tree format, and property table format
  against `Inform6-6.44/tables.c` output logic. Items not reflected in
  the compiler source (pure Z-machine Standard details) should be
  flagged `UNVERIFIABLE from repo source` and routed to the author.
- **ch29 Z-machine Instruction Set** — every opcode mnemonic and
  number must be present in `asm.c` Z-machine opcode tables.
- **ch30 Glulx Architecture** — cross-check against `Glulx-Spec.md` and
  `tables.c` Glulx paths.
- **ch31 Glulx Instruction Set** — cross-check every opcode against
  `asm.c` Glulx opcode tables *and* `Glulx-Spec.md`. Discrepancies
  between those two sources themselves should be reported.

## Part 5 — Advanced (ch32–ch36)

- **ch32 Replacing/Extending the Library** — verify `Replace` directive
  semantics against `directs.c` and `symbols.c`; `LibraryExtensions`
  `.RunAll` / `.RunWhile` / `.RunUntil` behavior against `verblib.h`;
  `LibraryMessages` dispatch against `verblib.h:3113` (L__M).
- **ch33 Internationalization/Localization** — verify every i18n
  mechanism against `english.h`, `chars.c` alternative alphabets, and
  the `Zcharacter` directive in `directs.c:1164–1227`.
- **ch34 Menus/Multimedia/Extended** — verify menu routines in
  `verblib.h` and multimedia claims against `Glulx-Spec.md`.
- **ch35 Optimization and Performance** — any "X is faster than Y"
  claim must tie to actual compiler behavior (constant folding in
  `expressp.c`/`expressc.c`, veneer choice, etc.) or be marked
  `UNVERIFIABLE`/opinion.
- **ch36 Debugging and Testing** — every debug verb must appear in
  `grammar.h` `#Ifdef DEBUG` block; `-k` / `-D` / infix features must
  match `inform.c`, `files.c`, `infix.h`. Cross-check `$!` trace
  options against Appendix D / E.

## Part 7 — Front matter and global cross-references

- Verify `guide/front-matter/title.md` and `guide/README.md` version
  numbers against `Inform6-6.44/header.h` (release/version macros) and
  `inform6lib-6.12.8/version.h`.
- Walk every inter-chapter and chapter→appendix cross-reference and
  confirm the target exists and says what the source chapter claims.
- Confirm every chapter follows `guide/front-matter/conventions.md`
  (GPL HTML comment, H1 title, §N.M headings, ``` ```inform6 ``` fences,
  `[Z-machine]`/`[Glulx]` tags).

## Completeness of ch13 and ch14

These were only spot-checked in Pass 2. Specifically:

- **ch14**: every backticked error-message string should be
  grep-confirmed verbatim in `Inform6-6.44/errors.c` and the other
  compiler `.c` files that emit errors (`expressp.c`, `asm.c`,
  `directs.c`, `verbs.c`). A string that cannot be grep-found is
  WRONG.
- **ch13**: every phase-ordering claim should be mapped to a function
  call in `inform.c:main()` / `compile()` and its descendants; stages
  that write output should be tied to `tables.c` / `files.c`.
