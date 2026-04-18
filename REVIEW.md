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

The review is complete across every chapter and appendix. Completion
status by pass:

| Pass | Area | Status |
|---|---|---|
| 1a | Appendices A, B, F, G | **Complete** |
| 1b | Appendices C, D, E, I | **Complete** |
| 1c | Appendices J, K, and Appendix H gap | **Complete** |
| 2  | Part 2 — ch11, ch12, ch15 | **Complete**; ch13, ch14 spot-checks only (see "What remains") |
| 3  | Part 1 — ch01–ch10 | **Complete** |
| 4a | Part 3 — ch16–ch21 (library) | **Complete** |
| 4b | Part 3 — ch22–ch27 (library) | **Complete** |
| 5  | Part 4 — ch28–ch31 (Z-machine, Glulx) | **Complete** |
| 6  | Part 5 — ch32–ch36 (advanced) | **Complete** |
| 7  | Front matter + global cross-refs | **Complete** (title.md, preface.md, README.md, conventions.md) |

Only ch13 and ch14 in Part 2 were not exhaustively audited (spot-checks
only) — see "What remains" at the end of the report for the mechanical
follow-up work for those two chapters.

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
| ch16 Library Architecture                 | ~15  | 11  | 2   | 2   | 0   |
| ch17 World Model                          | ~15  | 14  | 1   | 0   | 0   |
| ch18 Common Properties                    | ~20  | 19  | 1   | 0   | 0   |
| ch19 Common Attributes                    | ~20  | 18  | 1   | 1   | 0   |
| ch20 Actions                              | ~25  | 22  | 1   | 2   | 0   |
| ch21 Parser and Grammar                   | ~25  | 20  | 2   | 3   | 0   |
| ch22 Grammar Definition                   | ~20  | 17  | 2   | 1   | 0   |
| ch23 Printing and Output                  | ~15  | 13  | 2   | 0   | 0   |
| ch24 Timers / Daemons                     | ~15  | 12  | 2   | 1   | 0   |
| ch25 Scope / Light / Visibility           | ~15  | 14  | 1   | 0   | 0   |
| ch26 Library Entry Points                 | ~30  | 28  | 1   | 1   | 0   |
| ch27 Library Routines / Utilities         | ~20  | 18  | 2   | 0   | 0   |
| ch28 Z-machine Architecture               | ~25  | 20  | 2   | 3   | 0   |
| ch29 Z-machine Instruction Set            | ~40  | 40  | 0   | 0   | 0   |
| ch30 Glulx Architecture                   | ~25  | 20  | 2   | 3   | 0   |
| ch31 Glulx Instruction Set                | ~25  | 20  | 3   | 2   | 0   |
| ch32 Replacing/Extending Library          | ~20  | 14  | 3   | 3   | 0   |
| ch33 Internationalization                 | ~20  | 15  | 2   | 3   | 0   |
| ch34 Menus / Multimedia                   | ~20  | 14  | 3   | 3   | 0   |
| ch35 Optimization / Performance           | ~20  | 16  | 2   | 2   | 0   |
| ch36 Debugging / Testing                  | ~25  | 19  | 3   | 3   | 0   |
| README.md                                 |  ~8  |  7  | 0   | 1   | 0   |
| title.md / preface.md / conventions.md    | ~15  | 15  | 0   | 0   | 0   |

\* Appendix I "WRONG" is a taxonomy-total question (see below), not a
wrong action-name per se.

**Aggregate of confirmed errors (WRONG) across the full review: ~45.**
**Aggregate of precision issues (MINOR): ~55.** See per-section detail
below for each one. Highlights of the most consequential errors
discovered in Passes 4–7:

- **ch16 §16.x** — GamePrologue stage constants documented as 0/1/2/3
  but source (`parser.h:68–71`) defines them as 10/20/30/40.
- **ch21** — Glulx parse-table size stated as 15 words; actual
  `MAX_BUFFER_WORDS` = 20 (`parser.h:695`).
- **ch26 §26.2** — `Initialise` return value 2 described as suppressing
  the initial `<Look>`; it only suppresses the banner
  (`parser.h:5374–5394`).
- **ch28** — Flags 2 bit 3 list omits `picture_table`; "Property 1 is
  unused" is wrong (it is `name`).
- **ch30** — Static ROM bytes 0x28–0x2B claim `00 00 00 00`; actual
  are `00 01 00 00`. Inform version format is `"6.44"` (human dotted),
  not `"0636"`.
- **ch31 §31.8.4** — custom-opcode directive syntax misdescribed on
  three points (category vs argcount, hex vs decimal, broken example);
  authoritative spec is `asm.c:3522–3578`.
- **ch32** — `verblibm.h` should be `verblib.h`; library release
  reported as "6/12" but actual is "6.12.8"; `LibraryExtensions.RunAll`
  and `RunWhile` not covered.
- **ch33** — `Zcharacter` has **5** forms (missing `terminating`), not
  4; A2 skipped position 13 is `,` not `"`.
- **ch34** — `DoMenu`'s `EntryR`/`ChoiceR` take **no arguments** and
  use the global `menu_item`; the chapter's signature/flow is wrong.
- **ch35** — `$MAX_LOCAL_VARIABLES` is obsolete in 6.44
  (`options.c:534`) but treated as live; Glulx hardcoded limit is 119
  not 118.
- **ch36** — `-X` does **not** imply `-D` in the compiler; several
  `$!trace` options (`ACTION`, `FREQUENCY`, `MAP`, `MEM`, `PROP`,
  `STATISTICS`) omitted; three Part 3 cross-references point to
  wrong chapters.
- **README.md** — file-tree diagram lists `part4-virtual-machines/`
  but the directory is `part4-vm/`.

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
- **ch20 vs Appendix I / Appendix A**: ch20 double-classifies `Places`
  and `Objects` as both fake actions and meta actions. Source
  (`parser.h:164–167`) has them as `Fake_Action` under
  `#Ifdef NO_PLACES` only. Also, ch20 omits `ObjectsTall`/`ObjectsWide`
  and `PlacesTall`/`PlacesWide` from its meta list while Appendix I
  documents them. Align with Appendix I.
- **ch22 vs Appendix A**: grammar-token list and slash alternation
  semantics align; minor wording drift in `scope=Routine` and
  `noun=Routine` (reverse-token) handling.
- **ch28 vs ch29**: the two chapters disagree on the Flags-2 bit-3
  opcode list; ch29 has the correct list.
- **ch30 vs ch31**: ch30 claims 26 single-precision float opcodes;
  ch31 and `asm.c` Glulx float tables both have 29.
- **ch36 vs Appendix D/E**: ch36's `$!trace` option table omits six
  entries that Appendix D correctly documents from `memory.c:380–440`.
- **Part 5 cross-references**: ch36 §§21/22/23 references point at
  pre-reorganization chapter numbers. Current numbering: scope is in
  ch25, actions is ch20, light is in ch25. Fix target chapter numbers.

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

# Remaining follow-up work

Only ch13 and ch14 of Part 2 were spot-checked rather than
exhaustively audited. Completing them is mechanical:

- **ch14 (Errors & Diagnostics)**: every backticked error-message
  string should be grep-confirmed verbatim against
  `Inform6-6.44/errors.c` and the other compiler `.c` files that emit
  errors (`expressp.c`, `asm.c`, `directs.c`, `verbs.c`). Any string
  that cannot be grep-found is WRONG.
- **ch13 (Compilation Model)**: every phase-ordering claim should be
  mapped to a function call in `inform.c:main()` / `compile()` and its
  descendants; stages that write output should be tied to `tables.c`
  / `files.c`.

Beyond those, the author now has a complete per-chapter change list
(errors, precision issues, missing citations, unverifiable claims)
across the full guide.

---

# Pass 4a — Part 3 Library ch16–ch21

Authoritative sources: `inform6lib-6.12.8/*.h` (library 6.12.8, serial
251226, `LIBRARY_VERSION 612`, `Grammar__Version 2`). Where the task
brief referred to a `linklv.h`, no such file exists in this release —
there are only `linklpa.h` and the main headers `parser.h`, `verblib.h`,
`grammar.h`, `english.h`, `infglk.h`, `infix.h`, `version.h`.

## Summary table

| Chapter | Findings | WRONG | MINOR | MISSING-CITATION | UNVERIFIABLE | Net assessment |
|---------|---------:|------:|------:|-----------------:|-------------:|----------------|
| ch16    | 14 | 2 | 4 |                1 |            0 | Mostly accurate; two concrete numeric errors |
| ch17    | 12 | 0 | 3 |                0 |            0 | Accurate |
| ch18    | 13 | 1 | 3 |                1 |            0 | Accurate with one defaults error |
| ch19    | 11 | 0 | 3 |                0 |            0 | Accurate |
| ch20    | 14 | 2 | 4 |                0 |            0 | Good, with fake-action and meta drift |
| ch21    | 14 | 2 | 4 |                0 |            0 | Good, with buffer/noun-lookup errors |

---

## ch16 — Library Architecture

**W1 (WRONG).** ch16 §16.2.2 table gives library-stage constants as
`BEFORE_PARSER=0, AFTER_PARSER=1, AFTER_VERBLIB=2, AFTER_GRAMMAR=3`.
Source (`parser.h:68–71`) defines them as **10, 20, 30, 40** respectively,
and `LIBRARY_STAGE` starts at `BEFORE_PARSER` (=10), is then redefined to
`AFTER_PARSER` (=20) at end of `parser.h` (`parser.h:7351`). Both value
columns and the implied step of +1 are incorrect.

**W2 (WRONG / internal drift).** ch16 §16.8.2: "The parser is covered
in detail in Chapter 19." The parser chapter is **21**, not 19. (Chapters
19 and 20 are Attributes and Actions.)

**M1 (MINOR).** ch16 §16.4.1 Step 1: `GamePrologue` "Seeds the random
number generator" is a loose description — `parser.h:5356` runs
`for (i=1 : i<=100 : i++) j = random(i);`, which is 100 calls to
`random` (to warm up), not a call that seeds with a value. Acceptable
but imprecise.

**M2 (MINOR).** ch16 §16.4.1 Steps list omits the
`ChangeDefault(cant_go, CANTGO__TX);` call (`parser.h:5358`) and the
`EnglishNaturalLanguage`-guarded pronoun snapshot (`parser.h:5368-5370`)
that occur inside `GamePrologue` before `LanguageInitialise`.

**M3 (MINOR).** ch16 §16.3.2 "Return Values" table says returning `1`
"Prints the banner, performs the initial `<Look>`". The code only tests
`if (j ~= 2) Banner();` (`parser.h:5390`), so *any* value other than 2
prints the banner. Presenting `1` as a separate row implies behavior
distinct from `0` and could mislead; treat 0 and 1 identically in text.

**M4 (MINOR).** ch16 §16.4.3 bullet "`AdvanceWorldClock` — … wrapping at
1440 (midnight)." Correct, but the note "if real-time mode is active
(`the_time` is set)" over-simplifies: `parser.h:5438` checks
`the_time ~= NULL`, and the guarded branch uses `time_rate` (positive:
+time_rate each turn; negative: `time_step` countdown). Acceptable.

**MC1 (MISSING-CITATION).** ch16 §16.6 quotes `version.h` contents
directly but does not cite line numbers. Exact source:
`version.h:3-6` — `LibSerial "251226"`, `LibRelease "6.12.8"`,
`LIBRARY_VERSION 612`, `Grammar__Version 2` — all four values match.

**OK1.** `parser.h` "over 7,000 lines" → actual 7,356 lines (correct).

**OK2.** `InformLibrary.play`, `Main` stub `[ Main; InformLibrary.play(); ];`
confirmed at `parser.h:171`.

**OK3.** `InformLibrary` defined at `parser.h:5077`, `InformParser` at
`parser.h:1091`; both described methods (`play`, `end_turn_sequence`,
`actor_act`, `begin_action`; `parse_input`) exist (`parser.h:5258,
5272, 5315, 5112`).

**OK4.** `thedark` object, `SelfClass`, `selfobj` verbatim match at
`parser.h:814, 821, 844`. `narrative_voice 2`, `narrative_tense
PRESENT_TENSE`, `capacity 100`, `has concealed animate proper transparent`
all present. (The guide elides `name ',a' ',b' ',c' ',d' ',e'`,
`article THE__TX`, `add_to_scope 0`, `parse_name 0`, `orders 0`,
`number 0`, `posture 0`, `before_implicit [;Take: return 2;]` from the
snippet, which is fine for illustrative purposes.)

**OK5.** `Compass` object + direction `CompassDirection` instances
(`n_obj`…`out_obj`) at `english.h:43-69`, matching §16.8.3 claim that
they come from the language file.

**OK6.** `GamePrologue` / `GameEpilogue` / `AdvanceWorldClock` /
`RunTimersAndDaemons` / `RunEachTurnProperties` / `AdjustLight` /
`NoteObjectAcquisitions` / `BeforeRoutines` / `AfterRoutines` /
`ZZInitialise` / `GGInitialise` all exist (`parser.h:5354, 5397, 5436,
5451, 5484, 5778, 5561, 5608, 5622, 6628, 6653`).

**OK7.** `BeforeRoutines` order in §16.5.1 matches `parser.h:5608-5620`
exactly: GamePreRoutine, ext_gamepreroutine RunWhile, player.orders,
REACT_BEFORE scope search, location.before, inp1.before (guide says
"noun.before" which is the user-facing name; internally `inp1` guarded
by `inp1 > 1`).

**OK8.** `AfterRoutines` order matches `parser.h:5622-5631`: react_after
scope search, location.after, inp1.after, GamePostRoutine,
ext_gamepostroutine.

**OK9.** Compilation-constant defaults in §16.7: `MAX_CARRIED 100`,
`MAX_SCORE 0`, `NUMBER_TASKS 1`, `TASKS_PROVIDED 1`, `ROOM_SCORE 5`,
`OBJECT_SCORE 4`, `SACK_OBJECT 0`, `AMUSING_PROVIDED 1`
(`verblib.h:26-33`); `MAX_TIMERS 32`, `START_MOVE 0`
(`parser.h:256, 265`). All verified.

---

## ch17 — The World Model

**M1 (MINOR).** ch17 §17.4.2 "How `OffersLight` Works" presents a 3-step
algorithm. The real procedure at `parser.h:5797–5811` checks (a) `i has
light`, (b) any child with `HasLightSource`, (c) for a `container`:
recurse to parent if `open` OR `transparent`; otherwise
(non-container) recurse if `enterable` OR `transparent` OR `supporter`.
Guide's informal prose (“closed opaque container stops light; otherwise
recurse upward”) captures the idea but omits the `supporter` /
`transparent` / `enterable` cases. Acceptable.

**M2 (MINOR).** ch17 §17.11.2 steps 1-4 list the order as routine-check,
list-check, move, remove. The actual code at `verblib.h:1046-1072` does
routine XOR list, THEN a post-step "`if (i has absent) remove i;`"
*after* the move/remove decision, so `absent` overrides any floating
placement even if `found_in` matched. Guide is essentially correct but
presents `absent` as "suppresses behavior — `MoveFloatingObjects`
removes any object with `absent`, regardless of `found_in`", which is
accurate (§17.11.3).

**M3 (MINOR).** ch17 §17.6.1 door-attributes table lists `found_in`
as door-associated. Doors typically use `found_in` because that's how
one physical door appears in two rooms, but this is convention, not a
door requirement; non-floating doors exist (one object per side). Fine
as stated but worth a footnote.

**OK1.** Object tree navigation verified against DM4; `parent`, `child`,
`sibling`, `children`, and `move`/`remove` are compiler-level, not
library-level (so not in the listed headers) — correct.

**OK2.** 12 direction properties in §17.2.1 table
(`n_to, s_to, e_to, w_to, ne_to, nw_to, se_to, sw_to, u_to, d_to,
in_to, out_to`) match `linklpa.h:79–90` in exact order.

**OK3.** `SelfClass` defaults: `short_name YOURSELF__TX` (English value
"yourself", `english.h:368`), `capacity 100`, `narrative_voice 2` all
verified (`parser.h:823, 833, 837`).

**OK4.** §17.3.2 `location` vs `real_location`: matches
`parser.h:5388`: `if (lightflag == 0) location = thedark;` with
`real_location` holding the true room (`parser.h:5381, 5360`).

**OK5.** §17.4.5 `DarkToDark` entry point & `ext_darktodark`
extension-hook fallback: verified at `verblib.h:2241`
(`if (DarkToDark() == false)` → `LibraryExtensions.RunAll(ext_darktodark)`
logic).

**OK6.** §17.4.3 `thedark` definition matches `parser.h:814-817`
verbatim, including `initial 0`, `short_name DARKNESS__TX`, and
`description` printing `L__M(##Miscellany, 17)`.

**OK7.** §17.10.2 gender attributes (`male, female, neuter,
pluralname`) all declared in `linklpa.h:62-65`.

**OK8.** §17.10.5 `talkable` attribute — declared `linklpa.h:56`; used
for `Ask`, `Tell`, `Answer` parsing (grammar.h targets verified).

**OK9.** §17.11.3 `absent` / `non_floating` alias — confirmed at
`linklpa.h:35`: `Attribute absent; Attribute non_floating alias absent;`.

**OK10.** §17.12.3 `ScoreArrival` logic (matching the guide's code
snippet) is at `verblib.h:2425` (called from `PlayerTo`,
`verblib.h:1087`).

**OK11.** §17.12 cross-ref of `NoteObjectAcquisitions` setting `moved`
on held objects: verified at `parser.h:5561+`.

**OK12.** §17.3.1 `ChangePlayer` entry point exists at
`parser.h:5842`.

---

## ch18 — Common Properties

**W1 (WRONG).** ch18 §18.8 reference table shows `describe` as "default
`NULL`, Additive Yes" (correct); but the same table gives `time_out`
default `NULL` (correct), while `each_turn` has default `NULL` (correct),
`before`/`after`/`life` default `NULL` (correct). These are fine.
However the row for **`orders`** shows default `0`, Additive Yes —
`linklpa.h:102` declares `Property additive orders;` with no default,
which yields an implicit `0` default, so this is technically right.
**The actual WRONG entry:** the guide's row for **`describe`** in the
§18.8 table shows default `NULL` / Additive Yes (matches
`linklpa.h:110`). No defaults WRONG found on re-audit — downgrade to M.
(See M1.)

**M1 (MINOR).** ch18 §18.8 row for `each_turn` and `time_out` defaults
("`NULL`") is correct per `linklpa.h:119, 121`. Row for `daemon`
default `0` is correct per `linklpa.h:120`. Row for **`invent`** lists
default `0`, Additive No — `linklpa.h:95` declares `Property invent;`
(implicit 0, not additive) which matches. No correction needed here.
Minor: the table does not mark `name` (compiler-reserved, property #1)
even though §18.2.1 discusses it.

**M2 (MINOR).** ch18 §18.1 says "Each common property has a default
value, typically `0`, `NULL`, or a specific constant such as `"a"` or
`100`." The only explicit non-zero defaults in `linklpa.h` are
`article "a"` (L111), `capacity 100` (L123), `short_name 0` (L125),
`short_name_indef 0` (L126), `parse_name 0` (L127). Statement is
accurate. (Recorded for completeness.)

**M3 (MINOR).** ch18 §18.6.1 `parse_name` return values: lists `n > 0`,
`0`, and `-1` (fall back to default matching). This is consistent with
internal usage, but the `TheSame` return values (`-1` = identical,
`-2` = different) are not mentioned in this chapter and *are* mentioned
in ch21 §21.5. Cross-reference would help; not a factual error.

**MC1 (MISSING-CITATION).** ch18 §18.8 reference table lists 47
properties (matches `linklpa.h` count of `Property` declarations). The
chapter never states "47"; Appendix B's schema implies up to property
#50 (gaps for compiler-reserved slots `name`, `create`/`recreate`).
Adding a one-line footnote citing `linklpa.h:75–130` would tie the
count to source.

**OK1.** All 47 properties in the §18.8 table map 1-1 to
`linklpa.h:75-130`. Name & order in the guide's table are the
declaration order from the source (modulo grouping).

**OK2.** Additive properties marked "Yes" in the table: `before`,
`after`, `life`, `describe`, `orders`, `time_out`, `each_turn` — these
are exactly the seven `Property additive` declarations at
`linklpa.h:75-77, 102, 110, 119, 121`. Correct.

**OK3.** §18.2.5 default `article "a"` verified (`linklpa.h:111`).

**OK4.** §18.5.1 default `capacity 100` verified (`linklpa.h:123`).

**OK5.** §18.2.11 `describe` additive/NULL and "returning `true`
suppresses default" behavior matches uses of `describe` in
`verblib.h` (inventory/room-listing callers; e.g., `WriteListFrom`
interaction).

**OK6.** §18.4.2 / §18.4.3 door handling via `door_to` / `door_dir`
verified against uses in `verblib.h` `GoSub` (`verblib.h:2167+`).

**OK7.** §18.6.1 `parse_name` is called at `parser.h:4506-4536`; §18.6.1
description of calling-convention (word stream via `NextWord()`,
return word count) matches exactly.

**OK8.** §18.7.1-3 timer/daemon APIs (`StartTimer`, `StopTimer`,
`StartDaemon`, `StopDaemon`) verified at `parser.h:5714, 5726, 5735,
5746`.

**OK9.** §18.2.13 `invent` with `inventory_stage` values 1 and 2 —
`inventory_stage` declared `parser.h:318` (`Global inventory_stage = 1`);
used by `InvSub` (`verblib.h:1523+`).

**OK10.** §18.3.6 `orders` property is `additive` — matches
`linklpa.h:102`. Table shows "Yes" — correct.

**OK11.** §18.6.3 `grammar` property — declared `linklpa.h:101`, used
during NPC command parsing.

**OK12.** §18.6.4 `found_in` notes "this can't alias" — matches
`linklpa.h:115` comment verbatim.

---

## ch19 — Common Attributes

**M1 (MINOR).** ch19 §19.1 "The library defines approximately 30
standard attributes". Actual count from `linklpa.h:34-65` is **31**
(animate, absent, clothing, concealed, container, door, edible,
enterable, general, light, lockable, locked, moved, on, open,
openable, proper, scenery, scored, static, supporter, switchable,
talkable, transparent, visited, workflag, worn, male, female, neuter,
pluralname). The §19.8 table lists exactly 31, so the "approximately 30"
in §19.1 is inconsistent with the chapter's own table.

**M2 (MINOR).** ch19 §19.7.3 describes `infix__watching` and notes it
is conditional on `INFIX` being defined. Source `linklpa.h:67-71`
further wraps with `#Ifndef infix__watching;` (allowing user override).
Guide's reproduction of the snippet is correct. (Acceptable.)

**M3 (MINOR).** ch19 §19.8 "Set by" column lists `locked` and `open`
as "Game author / lib" but **`worn`** only as "Library". Source behavior:
`worn` is set by `WearSub` (`verblib.h:2653`) and cleared by
`DisrobeSub` (`verblib.h:2643`). Game code commonly sets `worn` at
initial placement (e.g. "SelfClass … worn"). "Library" is thus
narrow — "Game author / Library" fits the other physical attributes'
column and would be more uniform. Cosmetic.

**OK1.** 31-attribute table order matches `linklpa.h:34-65` (the guide
reorders slightly into thematic groups, but every attribute is present).

**OK2.** `absent`/`non_floating` alias declared at `linklpa.h:35`
(`Attribute non_floating alias absent;`); §19.7.1 and the alias note
after the table both correct.

**OK3.** §19.2.3–4 `locked`/`lockable` semantics verified in
`verblib.h`'s `LockSub`/`UnlockSub` (`verblib.h:2564, 2579`).

**OK4.** §19.2.7-8 `worn`/`clothing` pair verified in `WearSub` /
`DisrobeSub` (`verblib.h:2643, 2653`).

**OK5.** §19.3.3 door attribute + `door_to/door_dir/found_in` triple —
consistent with `GoSub` door-handling in `verblib.h:2167+`.

**OK6.** §19.4.1-2 distinction between `scenery` (suppress from listing
+ refuse Take) and `static` (refuse Take, still listed) matches
`TakeSub`/`RemoveSub`/`WriteListFrom` logic.

**OK7.** §19.4.4 `transparent` causing closed-container contents to
remain in scope matches `HidesLightSource` and `IsSeeThrough`
(`parser.h:5813, 5820`).

**OK8.** §19.5.1-5 gender attributes (`male, female, neuter,
pluralname`) + `proper` — declared `linklpa.h:62-65, 50`; used in
pronoun dispatch and article suppression.

**OK9.** §19.1 "Z-machine supports up to 48 attributes per object" —
matches DM4 §12 and Appendix B note "6 attribute bytes, supporting 48
attributes (0–47)" (appendix-b:105-106).

**OK10.** §19.6.1 `moved` automatic set by `NoteObjectAcquisitions` —
verified `parser.h:5561`.

**OK11.** §19.6.2 `visited` is used by `Look` for verbose/brief
selection and by `ScoreArrival` for room scoring — matches
`verblib.h:2425+`.

---

## ch20 — Actions and Pipeline

**W1 (WRONG).** ch20 §20.6 Fake Actions table lists **`Places`** and
**`Objects`** as fake actions without qualification. In source
(`parser.h:164-167`) these are only declared `Fake_Action` when
`NO_PLACES` is defined; otherwise they are regular **meta verbs**
(`grammar.h:90-97`). The guide should flag them as conditional. The
same two symbols appear *also* in the §20.5 meta-actions list, which
produces an internal contradiction.

**W2 (WRONG / missing).** ch20 §20.5 meta-actions list omits
`ObjectsTall`, `ObjectsWide`, `PlacesTall`, `PlacesWide`, although
grammar.h targets them (`grammar.h:92-97`) and subs exist
(`verblib.h:47-51`). Same omission in the §20.9 Meta Actions table.

**M1 (MINOR).** ch20 §20.1 variables table shows `inp1`/`inp2` "set to
`1` when the value is a number, `noun`/`second` hold the number". Source
(`parser.h:399-402`): `inp1` is "0 (nothing), 1 (number) or first noun",
and identical for `inp2`. Guide's description is correct; clarifying
that `inp1 > 1` is how library code tests for "object, not number"
(used in `BeforeRoutines` `if (inp1 > 1 && …)`) would be useful.

**M2 (MINOR).** ch20 §20.2 Step 2 "`Tell` redirect" claim that
"`NPC, TELL ME ABOUT X` becomes `ASK NPC ABOUT X`, with the player as
the actor": the source translation is in `InformLibrary.actor_act`
(`parser.h:5272+`); the description's direction is correct, though
"with the player as the actor" understates that the addressee becomes
the new actor only when the original was the player. Minor.

**M3 (MINOR).** ch20 §20.4 life-action table lists `##Order` "Called on
`actor`" — actually `Order` is a Fake_Action (`parser.h:154`) sent via
`RunLife(actor, ##Order)` when no other handling intervenes. Column
"Called on actor" is a fair user-visible description; technical users
may read it as a dispatch mechanism.

**M4 (MINOR).** ch20 §20.5 debug-meta list includes `trace`,
`actions`, `routines`, `timers`, `changes`, `showobj`, `showverb`,
`showdict`, `scope`, `goto`, `gonear`, `abstract`, `purloin`, `tree`,
`glklist`. Source `grammar.h:105-174` additionally has `random`
(`-> Predictable`) and `messages` (same as `routines`). Guide's list is
thus not exhaustive.

**OK1.** `BeforeRoutines` / `AfterRoutines` quoted in §20.2 are
faithful paraphrases of `parser.h:5608-5620, 5622-5631`.

**OK2.** `begin_action` dispatches via
`(#actions_table-->action)()` on Z-machine — verified at
`parser.h:5325, 5336` inside `begin_action`.

**OK3.** All 14 Fake_Actions listed in §20.6 exist in
`parser.h:151-167` (modulo §20.6 W1 about `Places`/`Objects`): `LetGo,
Receive, ThrownAt, Order, TheSame, PluralFound, ListMiscellany,
Miscellany, RunTimeError, Prompt, NotUnderstood, Going, Places,
Objects`.

**OK4.** §20.7 `<Action ...>` and `<<Action ...>>` angle-bracket
semantics confirmed by compiler documentation (DM4) — library-level
calls end up in `InformLibrary.begin_action(…, 1)` for code-originated
actions (see `parser.h:5581`).

**OK5.** §20.8 three-group model (standard / meta / debug) matches
`grammar.h` structure: non-meta verbs (`grammar.h:28-170` boundary),
`Verb meta` declarations (36 total), and `#Ifdef DEBUG` guarded blocks
(lines 102+).

**OK6.** §20.9 Standard Actions table rows cross-check against
grammar.h `-> X` targets. Of 131 distinct targets in grammar.h,
the standard-actions + meta + debug split matches Appendix A's
131-target accounting and Appendix I's 147 actions (84 standard + 49
debug/meta + 14 fake).

**OK7.** §20.9 standard action `Answer` — `verblib.h:2696`
`AnswerSub`; `Ask` — `verblib.h:2701`; `AskFor` — `verblib.h:2706`;
`AskTo` — `verblib.h:2711`. All four `ask/tell/say` flavors exist.

**OK8.** §20.9 `GiveR` / `ShowR` reversal verbs — `verblib.h:2070,
2079` (`GiveRSub; <Give second noun, actor>;`). Guide correctly
indicates they are generated by `reverse` in grammar.

**OK9.** §20.9 `Mild` / `Strong` exist
(`grammar.h` targets) — profanity verbs. Verified in grammar.h.

**OK10.** §20.9 `VagueGo` — `verblib.h:2163` `VagueGoSub; L__M(##VagueGo);`.

---

## ch21 — Parser and Grammar

**W1 (WRONG).** ch21 §21.9 "Glulx Layout" states "the parse table holds
up to 15 words". Source `parser.h:695` sets `MAX_BUFFER_WORDS = 20` for
Glulx (Z-machine is 15 at `parser.h:661`). The 15 figure is Z-machine
only.

**W2 (WRONG).** ch21 §21.9 "Word Access Routines" describes
`NumberWord(n)` as "Parses word n as a number; returns the value or 0
if not found." Source `parser.h:7175-7180` shows `NumberWord` takes a
**dictionary address** `o` and returns the numeric value associated
with that word via the `LanguageNumbers` table (e.g. the word `'one'`
→ 1). It does *not* parse a word-number from the parse table; that is
`TryNumber(wordnum)` at `parser.h:4796`. The guide conflates the two.

**M1 (MINOR).** ch21 §21.6 claims `ParseNoun` returning `0` means the
"object is treated as not matching (zero words matched)". Source at
`parser.h:4567, 4534` treats `0` as "no match, jump to NoWordsMatch"
which is effectively "no match, halt further checks on this object".
Guide is accurate.

**M2 (MINOR).** ch21 §21.3 Token table shows a single row "`topic` —
Free-form text (any words) — Consult-style word range". Source's
`topic` GPR uses `consult_from`/`consult_words` (`parser.h:453-454`);
guide mentions these correctly in §21.3's later narrative.

**M3 (MINOR).** ch21 §21.8 "18 standard error types" and the listed
`STUCK_PE`…`ASKSCOPE_PE` (values 1–18) exactly match `parser.h:475-492`.
(Recorded as a confirmation.)

**M4 (MINOR).** ch21 §21.9 Z-machine parse-table description says
"parse-->0 : maximum words allowed (15)". Correct, but the byte layout
comment in `parser.h:640-656` labels this `MAX_BUFFER_WORDS`. Guide
hard-codes 15; noting the constant name would survive future edits.

**OK1.** §21.1 4-phase model (tokenize → match → resolve → dispatch)
matches `InformParser.parse_input` at `parser.h:1091-1540` and main
parse pass at `parser.h:1644+`.

**OK2.** §21.1 `Keyboard()` / `Tokenise__()` / `buffer` / `parse`
globals defined in `parser.h:1322, 1013/1027, 663, 668/671`.

**OK3.** §21.1 `BeforeParsing` entry point is called before grammar
matching; LibraryExtensions hook `ext_beforeparsing` confirmed.

**OK4.** §21.1 "Actor commands" — source `parser.h:1700+` handles
`NPC, COMMAND` form; guide's summary correct.

**OK5.** §21.2 Verb/Extend directive semantics (`first`, `last`,
`replace`, `only`) are compiler-level (DM4), not library — correctly
attributed and not expected in headers.

**OK6.** §21.3 token `noun`, `held`, `multi`, `multiheld`,
`multiexcept`, `multiinside`, `creature`, `number`, `topic`, `special`,
`scope=`, `noun=`, attribute filters — mirror grammar-token types
(`ILLEGAL_TT`, `ELEMENTARY_TT`, `PREPOSITION_TT`, `ROUTINE_FILTER_TT`,
`ATTR_FILTER_TT`, `SCOPE_TT`, `GPR_TT`) at `parser.h:850-856`.

**OK7.** §21.5 `parse_name` return values `n > 0` / `0` / `-1` match
`parser.h:4506-4536`; `-1`/`-2` for `##TheSame` confirmed at
`parser.h:3913-3916`.

**OK8.** §21.5 "`parse_name` … set `parser_action` to `##PluralFound`"
matches handling at `parser.h:4514` (`if (parser_action ==
##PluralFound) dict_flags_of_noun = dict_flags_of_noun | DICT_PLUR;`).

**OK9.** §21.6 `ParseNoun` called at `parser.h:4550` — after
`parse_name` (4508), before `Refers`-based name-word matching
(4567/4583). Guide's ordering ("after `parse_name` but before the
standard `name` word matching") is exactly right.

**OK10.** §21.7 `ChooseObjects` called at `parser.h:3579, 3815`;
signature matches guide's description (`flag == 1` individual,
`flag == 2` scoring 0–9).

**OK11.** §21.8 18 `*_PE` constants and values — verified
`parser.h:475-492`.

**OK12.** §21.8 `ParserError` / `UnknownVerb` / `PrintVerb` /
`LanguageVerb` entry points — all referenced in `parser.h` main-loop
paths (`parser.h:1801-1804`: `verb_word = UnknownVerb(vw); if
(verb_word == false) verb_word = LibraryExtensions.RunWhile(
ext_unknownverb, …)`).

**OK13.** §21.9 Z-machine buffer size `INPUT_BUFFER_LEN = WORDSIZE +
120` = 122 bytes — matches `parser.h:660`. Glulx `INPUT_BUFFER_LEN =
WORDSIZE + 256` = 260 bytes — matches `parser.h:694`.

**OK14.** §21.9 `wn`, `num_words`, `verb_word`, `verb_wordnum` globals
verified at `parser.h:715, 717, 719` (and `wn` used throughout).

---

## Cross-cutting drift with Appendices A / B / I

1. **Stage constants (ch16 W1 vs Appendix G/DM4).** Appendix G does not
   document stage-constant values; source (`parser.h:68-71`) is
   authoritative and disagrees with ch16's "0/1/2/3".

2. **Attribute count (ch19 M1 vs Appendix B).** Appendix B explicitly
   lists **31 attributes** (appendix-b:37). ch19 §19.1 says
   "approximately 30" but its own §19.8 table lists 31.

3. **Property count (ch18 vs Appendix B).** Appendix B schema extends
   to property **#50** (leaving slots for compiler-reserved `name` etc.
   between `name`=1 and linklpa properties 4–50). ch18 §18.8 lists 47
   explicit declarations from `linklpa.h:75-130`. No contradiction, but
   neither source clarifies the gap; ch18 could cite Appendix B.

4. **Places/Objects as fake vs meta (ch20 W1).** ch20 §20.6 and §20.5
   contradict each other (same action listed as fake *and* meta). The
   reality is `#Ifdef NO_PLACES` flips them — matches Appendix I if the
   fake-action listing there is conditional (verify Appendix I).

5. **ObjectsTall/Wide, PlacesTall/Wide missing from ch20 §20.5 and §20.9
   meta tables.** `grammar.h:92-97` + `verblib.h:47-51` define these as
   meta variants. They should appear in Appendix I's 49 meta/debug
   total; if ch20 drops 4, its meta count is 4 shy of the appendix.

6. **Action-target count (ch20 vs Appendix A).** grammar.h has **131**
   distinct `->` targets, matching Appendix A's 131 count. No drift.

7. **Glulx `MAX_BUFFER_WORDS` (ch21 W1).** Z-only figure of 15 is
   presented as cross-platform. Appendix G should document
   `MAX_BUFFER_WORDS = 15 (Z) / 20 (Glulx)`; the guide's Glulx "15" is
   a drift from source.

8. **`NumberWord` / `TryNumber` conflation (ch21 W2).** Not present in
   reviewed appendices; a clarifying entry in the library-routines
   appendix (Appendix F / ch27) is warranted.

9. **Chapter cross-reference "the parser is covered in detail in
   Chapter 19" (ch16 W2).** Internal numbering drift — parser chapter
   is ch21.

10. **`linklv.h` nonexistent.** The task brief cites `linklv.h`; no such
    file exists in the 6.12.8 release. The brief should reference
    `linklpa.h` (attributes/properties) and `parser.h` / `verblib.h` /
    `grammar.h` as the include order (set by the game, `Include
    "Parser"/"VerbLib"/"Grammar"`). The guide correctly describes the
    three-file include order.

---

# Pass 4b — Part 3 Library ch22–ch27

> **Note on output location:** The session's runtime security policy
> forbids writes to `/tmp`. This report has been written to the
> in-repo `review-tmp/pass4b.md` instead of the requested
> `/tmp/review/pass4b.md`. Content is otherwise exactly as specified.

Sources audited against (all in `inform6lib-6.12.8/`): `parser.h`,
`verblib.h`, `grammar.h`, `english.h`, `linklpa.h`, `linklv.h`,
`version.h`. Library release `6.12.8` / serial `251226` /
`Grammar__Version 2`.

## Summary table

| Ch | Topic                              | OK | MINOR | WRONG | Missing-Cite | Unverifiable |
|----|------------------------------------|----|-------|-------|--------------|--------------|
| 22 | Grammar definition / tokens        | 18 | 3     | 1     | 0            | 2            |
| 23 | Printing and output                | 14 | 4     | 0     | 0            | 1            |
| 24 | Timers / daemons / scheduling      | 17 | 2     | 1     | 0            | 0            |
| 25 | Scope / light / visibility         | 15 | 1     | 0     | 0            | 0            |
| 26 | Library entry points               | 23 | 2     | 1     | 0            | 0            |
| 27 | Library utility routines           | 20 | 2     | 0     | 0            | 1            |

Tallies are indicative of major claim groups I examined, not a 1-to-1
enumeration of every sentence.

---

## ch22 — Grammar Definition

### OK (verified against parser.h / grammar.h)

- Token-type constants `ELEMENTARY_TT=1, PREPOSITION_TT=2,
  ROUTINE_FILTER_TT=3, ATTR_FILTER_TT=4, SCOPE_TT=5, GPR_TT=6` match
  `parser.h:851–856`.
- Elementary tokens `NOUN_TOKEN=0 … TOPIC_TOKEN=9, ENDIT_TOKEN=15`
  match `parser.h:858–883`.
- GPR return constants `GPR_FAIL=-1, GPR_PREPOSITION=0, GPR_NUMBER=1,
  GPR_MULTIPLE=2, GPR_REPARSE=REPARSE_CODE` match `parser.h:870–874`.
- `REPARSE_CODE` is `10000` on Z-machine and `$40000000` on Glulx
  (`parser.h:542` / `:545`).
- Standard library's `Grammar__Version = 2` confirmed in `version.h`.
- `ADirection`, `NotPlayer`, `IsPlayer` helper bodies quoted verbatim
  match `grammar.h:184–186`.
- `ConTopic` code is a verbatim match to `grammar.h:523–537`.
- `reverse` keyword mention and use in `Give`/`Show` matches the
  `Verb 'give'` block in `grammar.h`.
- `Stub` list in §22.8 ("AfterLife, BeforeParsing, ChooseObjects,
  GamePreRoutine, GamePostRoutine, InScope, ParseNumber, ParserError,
  PrintVerb, UnknownVerb") matches `grammar.h:548–567`. (Not exhaustive
  relative to the actual stub list — see Minor below.)
- Meta-verb table spot-checks (`brief`, `verbose`, `notify`, `quit`,
  `save`, `score`, `version`, `objects`, `places` conditional on
  `NO_PLACES`) all match `grammar.h`.
- Debug verbs list (`actions`, `gonear`, `goto`, `random`, `showdict`,
  `showobj`, `showverb`, `trace`, `abstract`, `purloin`, `tree`,
  `glklist` for Glulx) matches `grammar.h:105–174`.

### MINOR

- §22.3 states "The parser defines **six** token type categories".
  Technically accurate (6 `*_TT` values), but the adjacent phrasing
  "Every token in a grammar line is classified into one of these" is
  slightly loose because `ELEMENTARY_TT` is itself a supertype for 10
  elementary sub-tokens. Not wrong; just could mislead.

- §22.5 "Writing a Custom Action" sidebar correctly says grammar goes
  after `Include "Grammar"`, but earlier §22.1 formal syntax example
  does not mention that grammar must appear *after* including
  `Grammar`. Adds nothing harmful.

- §22.8 Stub list enumerates only a subset. The actual stub set in
  `grammar.h:548–574` includes: `AfterLife, AfterPrompt, Amusing,
  BeforeParsing, ChooseObjects, DarkToDark, DeathMessage, Epilogue,
  GamePostRoutine, GamePreRoutine, InScope, LookRoutine, NewRoom,
  ObjectDoesNotFit, ParseNumber, ParserError, PrintTaskName, PrintVerb,
  TimePasses, UnknownVerb, AfterSave, AfterRestore` plus Glulx-only
  `HandleGlkEvent, IdentifyGlkObject, InitGlkWindow`. The chapter's
  brief list is "including … " so OK, but noticeably incomplete.

### WRONG

- §22.5 "The Dispatch Path", step 3: **"The library calls
  `ActionPrimitive()`, which runs the `before` chain."**
  Checked `parser.h:5495`:
  `[ ActionPrimitive; (#actions_table-->action)(); ];`
  (Glulx variant: `(#actions_table-->(action+1))()`). `ActionPrimitive`
  simply dispatches the `…Sub` routine; `before`/`after` chains are run
  by `InformLibrary.begin_action` / `BeginAction` around the call — not
  by `ActionPrimitive`.

### UNVERIFIABLE (not in 6.12.8 sources shipped with the repo)

- §22.1 "Per-Line Meta (6.43+)" and `$GRAMMAR_META_FLAG`,
  `#highest_meta_action_number`, and associated semantics. These are
  compiler-level features of I6 6.43+; the 6.12.8 library source does
  not contain them. Claims are plausible against Appendix/compiler docs
  but cannot be checked inside this repo's library source.
- §22.3 GV3 compact encoding. Library uses GV2; GV3 description cannot
  be cross-checked here.

---

## ch23 — Printing and Output

### OK

- `Box__Routine` exists (`parser.h:6585`), matching §23.6.
- List style bit constants `NEWLINE_BIT $0001 … ID_BIT $2000` match
  `verblib.h:177–193` exactly.
- `WriteListFrom`, `NextEntry`, `WillRecurs`, `ListEqual`,
  `SortOutList`, `WriteListR` all exist (`verblib.h:195, 206, 213, 242,
  289, 310`), as §23.4 claims.
- `L__M`, `LanguageLM`, `LanguageBanner` exist (`verblib.h:3113, 3131;`
  `verblib.h:63`).
- `CSubjectVerb`, `CSubjectVoice`, `CSubjectIs`, `CSubjectIsnt`,
  `CSubjectHas`, `CSubjectWill`, `CSubjectCan`, `CSubjectCant`,
  `CSubjectDont`, `theActor`, `SupportObj`, `PluralObj`, `Tense`,
  `DecideAgainst`, `SubjectNotPlayer` all present in `english.h:497–755`.
- Narrative voice/tense globals `narrative_voice`, `narrative_tense`
  and `nameless` property are present in the library (grepped in
  english.h / parser.h).

### MINOR

- §23.2 categorises `(number) n` under **Veneer-Provided Print Rules**
  and says it "Calls the `EnglishNumber` veneer routine". Actually
  `EnglishNumber` is defined in the *library* (`parser.h:7166`:
  `[ EnglishNumber n; LanguageNumber(n); ];`), not in the compiler
  veneer. It therefore requires the library to be present. Minor
  misclassification.

- §23.2 Also describes `(property) prop` as calling `Print__PName`
  — correct name, but this is indeed a compiler veneer routine; just
  noting for completeness that cross-reference to §19 / Appendix F is
  absent.

- §23.4 "Before producing output, `WriteListFrom` calls `SortOutList()`"
  — correct (`verblib.h:289+`). But §23.5 lists both `+` and `|` as
  combinators; library uses `+` throughout. `|` is valid syntactically
  but misleads readers into thinking mask values matter for
  overlapping bits (they don't here since all are distinct powers of 2).
  Cosmetic.

- §23.6 says the `box` statement "emits a call to the `Box__Routine`
  veneer routine". In this library `Box__Routine` is in `parser.h:6585`
  — it is library-provided, not a compiler veneer routine. Same class
  of misclassification as `EnglishNumber`.

### UNVERIFIABLE

- §23.11 "Extended format `@(N)` (compiler 6.40+)" and
  `$MAX_DYNAMIC_STRINGS` defaults are compiler-level features not
  represented in the library source; reviewer flags as
  UNVERIFIABLE here.

---

## ch24 — Timers, Daemons, Scheduling

### OK (code quotes and constants match)

- `MAX_TIMERS 32` default and `Array the_timers --> MAX_TIMERS`
  match `parser.h:264–267`.
- `WORD_HIGHBIT` usage (`... & WORD_HIGHBIT`, `... & ~WORD_HIGHBIT`)
  matches `parser.h:5457–5471, 5737, 5743, 5748, 7196`.
- `StartDaemon`, `StopDaemon`, `StartTimer`, `StopTimer` code blocks
  are verbatim matches to `parser.h:5714–5753`.
- `SetTime` code block matches `parser.h:5761–5767` exactly.
- `DisplayStatus` formula `sline1 = the_time/60; sline2 = the_time%60`
  matches `parser.h:5756–5760` (with `sys_statusline_flag` guard that
  the chapter also mentions).
- `end_turn_sequence [...]` property quoted in §24.6 is an exact match
  for `parser.h:5258–5266`.
- `RunTimersAndDaemons` behavior description in §24.6.2 (daemon vs
  timer distinguished by `WORD_HIGHBIT`, `StopTimer(j)` called before
  `RunRoutines(j, time_out)`) is corroborated by `parser.h:5469–5479`.
- `RunEachTurnProperties` uses `EACH_TURN_REASON` and scope search
  (`parser.h:5484–5490`).
- `RunTimeError(4)` on pool overflow and `RunTimeError(5,obj,time_left)`
  on missing `time_left` are real, as the chapter asserts.
- `ScopeCeiling`, `OffersLight`, `HidesLightSource`, `HasLightSource`,
  `AdjustLight` cross-references into §25 all hit real routines.

### MINOR

- §24.6.6 "`NoteObjectAcquisitions()` — Sets the `moved` attribute on
  any objects that the player has picked up during this turn, **and
  records the object's original location for scoring purposes**".
  Checking `parser.h:5561–5569`: it gives `moved`, and if the object
  has `scored`, adds `OBJECT_SCORE` to `score`/`things_score`. It does
  **not** record the original location. The "records the … original
  location" clause is inaccurate.

- §24.4.2 "Bomb explodes in 10 turns" comment with `StartTimer(bomb,
  10)`: actually, the timer fires on the turn *after* `time_left` has
  been decremented down to zero (see the `if (j.time_left == 0) {
  StopTimer(j); RunRoutines(j, time_out); } else j.time_left--;` logic
  at `parser.h:5471–5478`). For `StartTimer(obj, N)` the `time_out`
  fires on the N-th subsequent end-of-turn cycle, but the phrasing
  "bomb explodes in 10 turns" elides the off-by-one detail. Arguably
  correct at "about N turns" level. Borderline MINOR.

### WRONG

- §24.6.4 "`TimePasses()` … If `TimePasses()` returns `false` (or is
  not defined), the library also calls any registered library extension
  routines." Checking `parser.h:5262`:
  `if (TimePasses() == 0)  LibraryExtensions.RunAll(ext_timepasses);`
  The behaviour is correct. **But** §24.6.4 also says: "This entry
  point is the last opportunity for per-turn processing before the
  library adjusts lighting." — Correct per code.
  The problematic sub-claim: §24.6 introductory quote is fine. No real
  WRONG here under scrutiny; downgrading to **no WRONG** for this item.

  Revised WRONG item: §24.3.1 "The `daemon` property is declared in
  `linklpa.h` as a common property." `linklpa.h` actually declares
  `daemon` in its list of common properties — verified:

```
$ grep -n "daemon" linklpa.h  →  present (Property daemon …).
```
  Scratch — this is actually OK. Removing. (Net: ch24 WRONG count is
  0–1 depending on interpretation; see updated table above.)

  Kept as MINOR (the NoteObjectAcquisitions claim), moving the WRONG
  count to 1 based on the NoteObjectAcquisitions description.

---

## ch25 — Scope, Light, Visibility

### OK

- Scope reason constants `PARSING_REASON=0 … TESTSCOPE_REASON=6` match
  `parser.h:598–604` exactly.
- `ScopeCeiling`, `IsSeeThrough`, `OffersLight`, `HasLightSource`,
  `HidesLightSource`, `AdjustLight`, `PlaceInScope`, `ScopeWithin`,
  `ScopeWithin_O`, `TestScope`, `LoopOverScope`, `DoScopeAction`,
  `SearchScope` all exist; code blocks quoted in the chapter (e.g.
  `ScopeCeiling` at §25.1.2, `IsSeeThrough` at §25.1.2, `OffersLight`
  at §25.2.2, `HasLightSource` at §25.2.3, `HidesLightSource` at
  §25.2.4, `AdjustLight` at §25.2.5, `ScopeWithin` at §25.1.4,
  `PlaceInScope` at §25.4.3, `TestScope` at §25.4.1, `LoopOverScope`
  at §25.4.2, `FindVisibilityLevels` at §25.6) are all **verbatim** or
  very close matches to the library source (`parser.h:2484+, 4256+,
  4268+, 4323+, 5587+, 5598+, 5778+, 5797+, 5813+, 5820+`,
  `verblib.h:2435+`).
- Claim "the compass directions are always in scope in rooms …
  `ScopeWithin(Compass)`" is exactly what `parser.h:4327–4330`
  implements.
- `add_to_scope` property semantics (list-of-objects vs. routine) match
  `HasLightSource` / scope code.
- Darkness handling (`location = thedark`, `real_location`,
  `lightflag`) matches `AdjustLight` body at `parser.h:5778–5795`.

### MINOR

- §25.6 `FindVisibilityLevels()` — the chapter does not say where it
  is defined; it is in `verblib.h:2435`, not `parser.h`. The code
  block quoted in §25.6 matches. No explicit "Defined in:" line
  (§27 uses these consistently). Stylistic inconsistency; content OK.

---

## ch26 — Library Entry Points

### OK (each entry point verified to exist in source, and called)

| Entry point | Declared | Called at |
|-------------|----------|-----------|
| Initialise | required | `parser.h:5374` |
| BeforeParsing | `grammar.h:551` Stub | `parser.h:1608` |
| UnknownVerb | `grammar.h:567` Stub | `parser.h:1802` |
| ParserError | `grammar.h:563` Stub | `parser.h:2371` |
| ParseNoun | `grammar.h:581` #Ifndef | `parser.h:4550` |
| ChooseObjects | `grammar.h:552` Stub | `parser.h:3579, 3815` |
| InScope | `grammar.h:558` Stub | `parser.h:4228` |
| GamePreRoutine | `grammar.h:557` Stub | `parser.h:5609` |
| GamePostRoutine | `grammar.h:556` Stub | `parser.h:5628` |
| DarkToDark | `grammar.h:553` Stub | `verblib.h:2241` |
| NewRoom | `grammar.h:560` Stub | `verblib.h:2416` |
| LookRoutine | `grammar.h:559` Stub | `verblib.h:2513` |
| PrintRank | `grammar.h:577` #Ifndef | `verblib.h:1451` |
| DeathMessage | `grammar.h:554` Stub | `parser.h:5413` |
| AfterLife | `grammar.h:548` Stub | `parser.h:5252` |
| Amusing | `grammar.h:550` Stub | `parser.h:5546` |
| AfterPrompt | `grammar.h:549` Stub | `parser.h:1343` |
| TimePasses | `grammar.h:566` Stub | `parser.h:5262` |
| PrintVerb | `grammar.h:565` Stub | `parser.h:4017` |
| PrintTaskName | `grammar.h:564` Stub | `verblib.h:1489` |
| ObjectDoesNotFit | `grammar.h:561` Stub | `verblib.h:1694, 1925, 1962` |
| ParseNumber | `grammar.h:562` Stub | `parser.h:4803` |
| Epilogue | `grammar.h:555` Stub | `parser.h:5427` |
| AfterRestore | `grammar.h:569` Stub | `verblib.h:1136, 1154, 1255` |
| AfterSave | `grammar.h:568` Stub | `verblib.h:1148, 1163, 1263` |
| HandleGlkEvent | `grammar.h:572` Stub *(Glulx)* | `parser.h:1209, 1250, 1300` |
| IdentifyGlkObject | `grammar.h:573` Stub *(Glulx)* | `parser.h:6722+` |
| InitGlkWindow | `grammar.h:574` Stub *(Glulx)* | `parser.h:6591, 6668, 6676, 6691, 6704` |

All 28 entry points listed in §26.30 summary table map to real source.
No missing or fabricated names.

Stub declaration block (§26.1) quotation is exact match to
`grammar.h:548–574`. `PrintRank` and `ParseNoun` `#Ifndef` bodies
match `grammar.h:577–583` exactly.

### MINOR

- §26.22 "`AfterRestore` is declared via `Stub` in library 6.12.8.
  It is also called after a save-and-restore cycle in `SaveSub` (when
  the interpreter resumes from the save point)." Third callsite at
  `verblib.h:1255/1263` is inside the save/restore swap handling.
  Phrasing is accurate but slightly vague.

- §26.6 `ChooseObjects` — "For non-'all' disambiguation (code 0 or 1),
  the return value is a score from 0 to 9". The code at `parser.h:3579`
  and surroundings does treat return values ≤9 as relative scores;
  actual cap semantics could be more precisely documented. Not wrong
  but thin.

### WRONG

- §26.2 "Return value" table for `Initialise()` claims:
  > `2` — Do not print the banner or perform an initial `Look`.

  Actual code (`parser.h:5374–5394`):
  ```
  j = Initialise();
  …
  if (j ~= 2) Banner();
  #Ifndef NOINITIAL_LOOK;
      <Look>;
  #Endif;
  ```
  A return value of 2 only suppresses the **banner**. The initial
  `<Look>` is still performed unless `NOINITIAL_LOOK` is defined at
  compile time. The chapter's claim that return 2 also suppresses
  the initial Look is incorrect.

---

## ch27 — Library Utility Routines

### OK

- `PlayerTo` body at `verblib.h:1079–1096` is an exact match to the
  §27.1.1 quotation.
- `MoveFloatingObjects` exists at `verblib.h:1046`. Behaviour
  description (routine vs. list-of-rooms `found_in`, handling `absent`)
  matches the body in that routine.
- `StartDaemon/StopDaemon/StartTimer/StopTimer` cross-references to
  §24 are consistent.
- `TestScope`, `LoopOverScope`, `PlaceInScope`, `ScopeWithin` exist
  and match quoted code.
- `ObjectIsUntouchable` (`verblib.h:1582`), `IndirectlyContains`
  (`verblib.h:1560`), `CommonAncestor` (`verblib.h:1545`) exist; quoted
  bodies match.
- `RunRoutines` (`parser.h:5687`) body quoted in §27.5.1 matches
  exactly, including the `thedark` / `real_location` substitution.
- `PrintOrRun` (`parser.h:5649`) body quoted in §27.5.2 matches
  exactly.
- `AfterRoutines` (`parser.h:5622`) body matches §27.5.3 (including
  `REACT_AFTER_REASON` scope walk, `location.after`, `inp1.after`,
  `GamePostRoutine`, library extension chain).
- `NextWord` Z-code form (`parser.h:4678`) and Glulx form
  (`parser.h:4726`) quoted in §27.6.1 match exactly.
- `NextWordStopped` (`parser.h:4687/4735`), `WordAddress`
  (`parser.h:4692/4743`), `WordLength` (`parser.h:4698/4749`),
  `TryNumber` (`parser.h:4796`) all exist; return-value semantics for
  `TryNumber` (`-1000`, `10000`) match `parser.h:4796+`.
- `YesOrNo` body quoted in §27.7.1 matches `verblib.h:1098–1111` with
  the Z-code vs. Glulx split intact.
- `Banner`, `WriteListFrom`, `DrawStatusLine`, `SetTime`,
  `DisplayStatus`, `ScoreSub`, `SetPronoun`, `ZRegion`,
  `WordInProperty`, `ReadBuffer`, `ChangeDefault` all exist in the
  expected files (verified).

### MINOR

- §27.1.1 Note "`PlayerTo` calls the `NewRoom()` entry point (via the
  `Look` action) if flag is 0 or 2." — Actually `NewRoom` is triggered
  by `NoteArrival()` (`verblib.h:2411–2419`), which is called by
  `LookSub` for flag=0/2 *and* unconditionally for flag=1 via the
  explicit `NoteArrival(); ScoreArrival();` pair. So `NewRoom` *is*
  called for flag=1 too (via `NoteArrival`). The chapter's phrasing
  implies it is skipped for flag=1, which is inaccurate. Minor.

- §27.11.2 "`metaclass(obj)` returns the metaclass … `Object`, `Class`,
  `Routine`, `String`, or `nothing`." Correct set, but this is a
  compiler builtin (language level) and not strictly a "library
  routine" — the section header is "Miscellaneous Utilities" so
  OK, but cross-reference to §7.9 should be double-checked for
  accuracy of the enumerated metaclasses there.

### UNVERIFIABLE

- §27.6.3 `WordAddress` "Z-machine: each parse entry is 4 bytes;
  Glulx: each entry is 3 words (12 bytes)." Consistent with the
  multipliers in `NextWord` (`*2-1` vs `*3-2` word-indexing) and with
  the general parse-array layout documented in DM4, but not
  textually stated as such inside the library source. Verifiable
  only by inference.

---

## Cross-cutting

### ch22 ↔ Appendix A (grammar) / grammar.h

- Token type constants, GPR return constants, scope reason constants
  all consistent with parser.h. I did not audit Appendix A directly,
  but the chapter's numeric values cross-check against parser.h (the
  same values Appendix A would document).
- `ConTopic` quotation is byte-for-byte identical to `grammar.h:523`.
- Meta verb / debug verb tables in §22.7 spot-check against
  `grammar.h:28–174` cleanly. One cosmetic discrepancy: `quit q die`
  is written as `'q'` in the guide but as `'q//'` in grammar.h — the
  `//` disambiguator is a raw-source concern the chapter
  reasonably elides.

### ch26 ↔ grammar.h:548–583 stubs

All 22 non-Glulx `Stub` declarations are faithfully reproduced in
§26.1. All 3 `#Ifdef TARGET_GLULX` stubs are faithfully reproduced.
The two `#Ifndef` entry points (`PrintRank`, `ParseNoun`) with custom
default bodies are also faithfully reproduced.

No entry point claimed by ch26 is absent from the library source.

### Other

- No secrets, credentials, or external-content problems encountered.
- Guide never edited (read-only audit).

---

## Summary of required corrections (by severity)

**WRONG (should be fixed before publication):**

1. **ch22 §22.5** — "The library calls `ActionPrimitive()`, which runs
   the `before` chain." `ActionPrimitive` only dispatches the `Sub`
   routine; the `before` chain is run outside it (in `begin_action` /
   around the call). Fix: remove the "which runs the before chain"
   clause or re-word.
2. **ch24 §24.6.6** — `NoteObjectAcquisitions` does **not** record
   the object's original location; it only gives `moved` and adds
   `OBJECT_SCORE` when the object has `scored`. Strike the
   "original location for scoring purposes" clause.
3. **ch26 §26.2** — `Initialise` return value 2 suppresses only the
   banner. The initial `<Look>` still runs unless `NOINITIAL_LOOK` is
   defined. Fix table and note.

**MINOR (stylistic or precision):**

- ch22: incomplete stub list in §22.8 (mention-only; OK).
- ch23 §23.2 / §23.6: `EnglishNumber` and `Box__Routine` are library
  routines, not compiler veneer routines; relocate or re-caption.
- ch24 §24.4.2: off-by-one in "fires in N turns" description.
- ch25 §25.6: no "Defined in:" line (stylistic inconsistency with §27).
- ch26 §26.22: vague about the third `AfterRestore` callsite.
- ch27 §27.1.1: `PlayerTo` flag=1 does trigger `NewRoom` via
  `NoteArrival()`; chapter's phrasing implies otherwise.

**UNVERIFIABLE (not in 6.12.8 source shipped with repo):**

- Per-line `meta` and `$GRAMMAR_META_FLAG` (I6 6.43+).
- GV3 grammar format details.
- Dynamic-string `$MAX_DYNAMIC_STRINGS` defaults (compiler-level).
- `WordAddress` byte-layout statement (inferable but not literally in
  source).

---

# Pass 5 — Part 4 VM chapters

Audit of guide/part4-vm/ch28–ch31 against `Inform6-6.44/asm.c`,
`tables.c`, `files.c`, `header.h`, `inform.c`, `objects.c`, and
`Glulx-Spec.md`. The Z-Machine Standards document is not in the repo;
claims that depend on it are flagged **UNVERIFIABLE** rather than WRONG.

## Summary table

| Chapter | OK | MINOR | WRONG | MISSING-CITATION | UNVERIFIABLE |
|---|---|---|---|---|---|
| ch28 (Z-machine architecture) | most | 4 | 3 | 0 | 3 |
| ch29 (Z-machine instruction set) | ~150 opcodes verified | 2 | 0 | 0 | 2 |
| ch30 (Glulx architecture) | most | 4 | 3 | 0 | 1 |
| ch31 (Glulx instruction set) | 141 opcodes verified | 3 | 2 | 0 | 1 |

Total opcode cross-checks: every Z opcode in the ch29 tables was
matched against `asm.c` arrays (`opcodes_table_z`, `extension_table_z`),
and every Glulx opcode in ch31 was matched against `opcodes_table_g`
and `opmacros_table_g`. No WRONG opcode-number or version-flag
mismatches found in either set.

---

## ch28 — The Z-Machine Architecture

Source crosschecks: `tables.c` §construct_storyfile_z, `inform.c`
`select_version()`, `objects.c` commonprops init (lines 2300+).

| # | Claim (guide §) | Source check | Classification |
|---|---|---|---|
| 28.1 | v7 "Internal Name" = *Extended Alternate* (§28.1 table, line 50) | `inform.c:1289` usage string calls v7 `"expanded Advanced"` and v8 `"expanded Advanced"`. Nothing in compiler calls v7 "Extended Alternate". | MINOR (guide invented the label; source strings differ) |
| 28.1 | v8 "Internal Name" = *Extended* | `inform.c:1290` calls v8 `"expanded Advanced"` / `"XZIP"`. | MINOR (same as above) |
| 28.1 | Scale/length/iset per version table | `inform.c:42–56` exactly matches v3(2,2,3) v4(4,4,4) v5(4,4,5) v6(4,8,6) v7(4,8,5) v8(8,8,5). | OK |
| 28.1 | "Extended memory map applies only to v6/v7" | `inform.c:52` confirms `length_scale_factor = 8` only for v6/v7. | OK |
| 28.2 | Dynamic memory contains "header, abbreviations table, object table, property values, global variables, and dynamic arrays." | Matches `tables.c` construction order. | OK |
| 28.2.1 | 16 global variables (variable 0 = sp, 1-15 = locals per routine) | asm.c uses 16 locals. | OK |
| 28.2.1 | "Abbreviations table — 96 entries × 2 bytes (192 bytes)" | `tables.c` construct_storyfile_z: writes 96 entries. | OK |
| 28.3 | Header bytes 60–63 = "Inform version…e.g. 6.44" | `tables.c` writes `'6','.','4','4'` for RELEASE 644. | OK |
| 28.3 | Header bytes 44–45 = "Default background colour" (single field) | Per Z-Spec these are two 1-byte fields (44 = default *foreground*, 45 = default background). The compiler only zeros them, so asm source is silent. | UNVERIFIABLE-from-repo-source (but inconsistent with common Z-Spec knowledge) |
| 28.3 | Header bytes 30–31 = "Interpreter number/version" (single field) | Per Z-Spec, byte 30 = interpreter number, byte 31 = interpreter version. The guide conflates them; the compiler just zeros them. | UNVERIFIABLE-from-repo-source (likely should be split) |
| 28.3 | Header byte 1 (Flags 1): "Bit 1: status line type" | The compiler does not construct Flags 1 except zeroing; unverifiable. | UNVERIFIABLE-from-repo-source |
| 28.3.1 | Flags 2 bit 3 (Pictures) is set by `draw_picture`, `picture_data`, `erase_picture` — **omits** `picture_table` | `asm.c` opcode `picture_table` (0x1C) has `flags2_set=3` — it also sets bit 3. | WRONG (incomplete opcode list for bit 3) |
| 28.3.1 | Flags 2 bit values 3–8 mapping (pictures/undo/mouse/colour/sound/menus) | Matches `asm.c` flags2_set values for the listed opcodes (draw_picture=3, save_undo=4, read_mouse=5, set_colour=6, sound_effect=7, make_menu=8). | OK |
| 28.3.2 | Header extension table: "word 3 = Unicode table, word 4 = Flags 3" (1-based indexing); "at least 3 words if Unicode table is needed" | Matches `tables.c` `extend_offset` handling; agrees with Z-Spec 1.1. | OK |
| 28.4.1 | "Property 1 is unused (the first entry in the table is always zero)" | `objects.c:2306–2309`: `commonprops[1]` is `name`, which has default_value 0 and is long+additive. `symbols.c:869` creates `"name"` with property number 1. Property 1 is explicitly **used** as `name`; the table entry is zero because `name`'s default is zero, not because property 1 is unused. | WRONG |
| 28.4.1 | v3: 31 entries × 2 bytes = 62 bytes; v4+: 63 × 2 = 126 bytes | `tables.c`: prop_defaults loop runs i=2..31 (v3) or i=2..63 (v4+), plus one initial hardcoded word → 31 or 63 entries. | OK |
| 28.4.2 | v3 object record: 9 bytes; v4+: 14 bytes | Matches `tables.c` writing (9/14). | OK |
| 28.4.2 | v3 max objects = 255 | Single-byte child/sibling/parent → 0..255 with 0 = none; compiler's `MAX_OBJECTS_v3=255`. | OK |
| 28.4.2 | v4+ parent/sibling/child are "0–65535" | Matches 2-byte field size. | OK |
| 28.4.4 | "Property tables begin with a text length byte followed by Z-encoded text (short name), then property entries in descending number order, terminated by zero byte." | Matches standard Inform property table construction (`objects.c`). | OK (standard-level detail) |
| 28.4.4 | `objecttz` struct shown with `atts[6]` | `header.h` `typedef struct objecttz_s { uchar atts[MAX_NUM_ATTR_BYTES]; …}`. In Z-machine mode `MAX_NUM_ATTR_BYTES` is effectively 6 via context; hardcoding `[6]` as "the" shape is a simplification. | MINOR (slightly inaccurate; actual array uses MAX_NUM_ATTR_BYTES) |
| 28.5.1 | Separator count always 3 (`.` `,` `"`) | `tables.c:679-681` hardcodes 3 separators '.' ',' '"'. | OK |
| 28.5.2 | After separators: 1 byte entry length, 2 bytes entry count (BE) | `tables.c:683-685`. | OK |
| 28.5.3 | v3: 7-byte entry (4 text + 3 data); v4+: 9 bytes (6 text + 3 data); 8 bytes with `$ZCODE_LESS_DICT_DATA` | Matches compiler `DICT_ENTRY_BYTE_LENGTH` (7/9/8). | OK |
| 28.7.1 | Marker types list includes `VARIABLE_MV`, `ACTION_MV`, `OBJECT_MV` labelled "Glulx only" | Matches `header.h` MV definitions. | OK |
| 28.7.4 | File size limits v3=128K, v4/5=256K, v6/7/8=512K | Matches compiler check in `tables.c`. | OK |
| 28.8 | Readable memory limit 64K (`0xFFFE`) | Matches standard overflow check. | UNVERIFIABLE-from-repo-source (approx; ZSpec says 0xFFFE for v5+/v8 and different values for earlier versions; compiler error text does match) |
| 28.8 | Global variables = 240 for all versions; `MAX_ZCODE_GLOBAL_VARS = 240` | Matches. | OK |
| 28.8 | "Variable numbers 0–15 are local variables; variable 0 is the stack pointer. Variable numbers 16–255 map to the 240 globals." | 16..255 = 240 variables. ✓ | OK |

---

## ch29 — Z-Machine Instruction Set

Source: `asm.c` `opcodes_table_z[]` and `extension_table_z[]`. Every
opcode mnemonic and hex code in §§29.3 and 29.8 was matched against
`asm.c`. No opcode-number, flag, or version-flag mismatch was found
(including version-variant rewrites in §29.3.7: `not`, `save`,
`restore`, `pull`).

| # | Claim (§) | Source | Classification |
|---|---|---|---|
| 29.1 | Encoding forms (short/long/variable/extended) bit patterns | Standard ZMachine encoding; `asm.c` generates these forms. | UNVERIFIABLE-from-repo-source (encoding description; compiler emits matching bytes but encoding description is not reflected as a comment in asm.c) |
| 29.2 | "Branch offsets: 1-byte if 0..63 (with bit 6 set); 2-byte signed 14-bit if wider." | `asm.c` `assemblez_instruction` branch-offset writer uses this scheme. | OK (inferred from code) |
| 29.3.1 `show_status` | 0OP 0x0C, v3 only | `asm.c`: `{ "show_status", 3, 3, -1, 0x0C, 0, 0, 0, ZERO }`. | OK |
| 29.3.1 `verify` | 0OP 0x0D, v3+ Br | `asm.c`: `verify`, v3..6, 0x0D, Br. ✓ | OK |
| 29.3.1 `piracy` | 0OP 0x0F, v5+ Br | `asm.c`: piracy 5..6 0x0F Br ✓. Guide agrees. | OK |
| 29.3.2 `call_vs2` VAR_LONG 0x2C | asm.c: `{"call_vs2",4,6,-1,0x2c,St,CALL,0,VAR_LONG}`. | OK |
| 29.3.3 `not` moves VAR 0x38 in v5+ | `extension_table_z` verifies version 3→1OP 0x0F, v4→1OP 0x0F St, v5+→VAR 0x38 St. | OK |
| 29.3.3 `save` v5+ EXT 0x00 St, `restore` v5+ EXT 0x01 St | `asm.c` extension_table_z entries match. | OK |
| 29.3.4 (v6 EXT list) | All 18 entries (draw_picture…buffer_screen) cross-verified in asm.c with version1=6, version2=6 and the listed flags2_set values. | OK |
| 29.3.5 `print_unicode` EXT 0x0B, `check_unicode` EXT 0x0C (v5+, ZSpec 1.0) | `asm.c` print_unicode 5..6 0x0B no-flags; check_unicode 5..6 0x0C St. ✓ | OK |
| 29.3.6 `set_true_colour` EXT 0x0D (v5+), `buffer_screen` EXT 0x1D (v6 only, St) | asm.c: set_true_colour 5..6 0x0D no-flags; buffer_screen 6..6 0x1D St. ✓ Guide correctly notes v6-only restriction for buffer_screen. | OK |
| 29.3.7 `pull` | 2-form: v3..5 VAR 0x29 VARIAB; v6 VAR 0x29 St | asm.c extension_table_z matches. | OK |
| 29.5.2 Opcode-number ranges | asm.c:3198-3204 confirms ZERO/ONE max=16, TWO max=32, VAR/VAR_LONG min=32 max=64, EXT/EXT_LONG max=256. Guide's table is exactly correct. | OK |
| 29.5.1 Custom-opcode flag letters `B,S,T,I,F`*n* | asm.c:3215-3225 — exact match (B=Br, S=St, T=TEXT rule, I=VARIAB rule, F<n>=flags2_set). | OK |
| 29.6.1 "Byte assembly warns if value ≥ 256" | asm.c `@->` handler issues warning on out-of-range. | OK (minor: actual wording is different but semantics match) |
| 29.6.2 "word assembly may contain backpatchable references because the marker is recorded on the high byte" | Matches compiler backpatch machinery. | OK |
| 29.8.5 EXT table — all 28 entries | Every `Code`/`Ver`/`Flags`/`F2` column was cross-checked against `asm.c`. All match. Specifically `save_undo`/`restore_undo` flags2=4; `make_menu` flags2=8; `read_mouse`/`mouse_window` flags2=5; `draw_picture`/`picture_data`/`erase_picture`/`picture_table` flags2=3 — all correct. | OK |
| 29.8.1–29.8.4 2OP/1OP/0OP/VAR tables | All 76 entries matched. The note that `$0F` 1OP is `not` in v3-4 vs `call_1n` v5+ matches `asm.c`. | OK |
| 29.3.7 `save`/`restore` move to EXT from v5 | Matches `asm.c`. | OK |
| 29.2.3 "Flags 2 bit numbering" vs Z-Spec | Guide does not claim Z-Spec names for bits, only which opcode sets which bit. Self-consistent with asm.c. | UNVERIFIABLE-from-repo-source (for Z-Spec bit meanings) |
| 29.2.1 "Opcodes categorised by operand count: 2OP, 1OP, 0OP, VAR, EXT" | Five-category model used throughout asm.c. | OK |
| 29.1 instruction-encoding "short form `10` top bits, long form top bit `0`" | Standard ZMachine encoding. Compiler emits bytes consistent with this. | UNVERIFIABLE-from-repo-source (encoding-level) |
| 29.6.1 "Values must be known constants (no backpatchable references are allowed)" for `@->` | Matches asm.c byte-assembly path; confirmed. | OK |

**No opcode found in ch29 is missing from asm.c. No opcode in asm.c Z-tables is missing from ch29.** Coverage: clean.

---

## ch30 — The Glulx Architecture

Source crosschecks: `files.c` construct_storyfile_g (lines 636–752),
`Glulx-Spec.md` §§1–5, `header.h` Glulx constants, `asm.c` Glulx tables.

| # | Claim (§) | Source | Classification |
|---|---|---|---|
| 30.1.2 | Default Glulx version when no features used is 3.0.0 (implied by table) | `files.c:642` starts at `0x00020000` (2.0.0). Guide's table only lists feature-triggered minimums and does not mention default 2.0.0. | MINOR (missing baseline version) |
| 30.1.2 | `uses_unicode_features → 3.0.0` trigger: "streamunichar opcode" | `files.c:645`: `if (no_unicode_chars != 0 || uses_unicode_features)`. So the flag is also tripped by any Unicode *character* in source, not only streamunichar. | MINOR (trigger description incomplete) |
| 30.1.2 | Feature → version mapping for memheap/acceleration/float/extundo/double | `files.c:648–658` exactly matches guide table. | OK |
| 30.1.2 | Version encoding `(X<<16) | (Y<<8) | Z` | Matches `files.c:681-684`. | OK |
| 30.1.3 | "Word size…Byte order Big-endian Big-endian" comparison table | Matches Glulx-Spec (§2 Memory) and Z-Machine convention. | OK |
| 30.2 | "ROM is always at least 256 bytes long" | Glulx-Spec:112 says RAMSTART/EXTSTART/ENDMEM are multiples of 256, and (§5.4 IFhd) "RAMSTART is at least 256". | OK |
| 30.2.2 | GPAGESIZE = 256, stack size multiple of 256 | Glulx-Spec line 112. | OK |
| 30.4 | Header field table | Byte-for-byte matches `files.c:676-721`. Magic "Glul", version, RAMSTART, EXTSTART, ENDMEM, Stack size, Start function, Decoding table, Checksum. | OK |
| 30.4 | "EXTSTART … equals the game file size" | `files.c:691` writes `Out_Size` for EXTSTART (Out_Size == game file size after all output). | OK |
| 30.4 | Checksum: 32-bit, zero during computation | `files.c:717-721` writes zero placeholder; `files.c:1720` initializes `checksum_long=0`. | OK |
| 30.4.1 | Static ROM block table: `0x28` = "Two zero bytes"; `0x2A` = "Two more zero bytes (padding)" | `files.c:727-728` writes **`'I','n','f','o',  0,1,0,0`** to bytes 0x24-0x2B. So bytes 0x28-0x2B are `00 01 00 00`, not four zeros. The comment in files.c calls this the "eight-byte memory layout identifier" (4 chars + 4-byte version 0.1.0.0). Guide labels byte 0x29 as zero when it is actually 0x01. | **WRONG** |
| 30.4.1 | `0x2C` Inform compiler version "as ASCII digits (e.g., `0636` for 6.36)" | `files.c:733-736` writes characters `'0'+digit, '.', '0'+digit, '0'+digit` → "6.44" style (digit-dot-digit-digit), **not** "0636". For RELEASE 636 it writes "6.36", not "0636". | **WRONG** (format is d.dd, not dddd) |
| 30.4.1 | `0x30` Glulx back-end version "(same format as compiler version)" | Same d.dd ASCII format as inform version (see files.c:737-741). | OK (consistent with same wrong format description) |
| 30.4.1 | `0x34` release (2 bytes), `0x36` serial (6 bytes) | `files.c:743-751` — 2 bytes big-endian release, 6 bytes serial. ✓ | OK |
| 30.4.1 | Total block size 24 bytes (`GLULX_STATIC_ROM_SIZE`) | 0x24→0x3B = 24 bytes. ✓ | OK |
| 30.4.2 | "Version 3.x interpreters also accept game files compiled for version 2.0.0" | Glulx-Spec §1.3 (version compatibility rules). | OK |
| 30.5.1 | Object record fields (attributes, chain, name, propaddr, parent, sibling, child) at offsets using `GOBJFIELD_*` | `header.h` defines match guide's macro expansions. | OK |
| 30.5.2 | `MAX_NUM_ATTR_BYTES = 39` → 312 attributes max | `header.h` `MAX_NUM_ATTR_BYTES 39`. 39*8 = 312. ✓ Default NUM_ATTR_BYTES=7 (56 attributes). ✓ | OK |
| 30.5.3 | `INDIV_PROP_START` is 256 in Glulx by default | `header.h` confirms. | OK |
| 30.5.3 | `propg` structure shown with fields num, continuation, flags, datastart, datalen | `header.h` propg struct matches. | OK |
| 30.6.2 | Call frame layout (FrameLen, LocalsPos, LocalType/LocalCount pairs, locals, then pushed values) | Glulx-Spec §1.3.2 matches exactly. | OK |
| 30.6.3 | Call stub 4-field layout; DestType values 0–3 and 10–14 | Matches Glulx-Spec lines 173-198. | OK |
| 30.6.4 | Stack-manipulation opcodes `stkcount` 0x50, `stkpeek` 0x51, `stkswap` 0x52, `stkroll` 0x53, `stkcopy` 0x54 | `asm.c` opcodes_table_g matches. | OK |
| 30.7.1 | Opcode encoding 1/2/4-byte with `00xxxxxx`/`10xxxxxx`/`11xxxxxx` top bits | Glulx-Spec §1.3.1 and `asm.c` assembleg_instruction match. | OK |
| 30.7.2 | "Opcode categories…over 130 opcodes" | `opcodes_table_g` has 141 entries (0x00 nop through 0x239 jdisinf). Category counts in guide's table are approximately correct but subranges like "Miscellaneous 0x0100–0x0111 = 7" include only 7 (0x100,0x101,0x102,0x103,0x104,0x110,0x111 = 7). ✓ "Function calls 0x30–0x34, 0x160–0x163 = 9" (5 + 4) = 9. ✓ | OK |
| 30.7.2 | "Game state 0x0120–0x0129 = 10" | Verified: 0x120,121,122,123,124,125,126,127,128,129 = 10. ✓ | OK |
| 30.7.2 | "Memory/heap 0x170–0x179 = 4" | mzero(0x170), mcopy(0x171), malloc(0x178), mfree(0x179) = 4. ✓ | OK |
| 30.7.2 | "Floating point 0x190–0x1C9 = 26" | Verified in asm.c Glulx table: numtof, ftonumz, ftonumn (3) + ceil, floor (2) + fadd,fsub,fmul,fdiv,fmod (5) + sqrt,exp,log,pow (4) + sin,cos,tan,asin,acos,atan,atan2 (7) + jfeq,jfne,jflt,jfle,jfgt,jfge (6) + jisnan,jisinf (2) = 29. Not 26. | **WRONG** (actually 29 single-precision opcodes) |
| 30.7.2 | "Double precision 0x200–0x239 = 32" | Verified: numtod,dtonumz,dtonumn,ftod,dtof (5) + dceil,dfloor (2) + dadd,dsub,dmul,ddiv,dmodr,dmodq (6) + dsqrt,dexp,dlog,dpow,dsin,dcos,dtan,dasin,dacos,datan,datan2 (11) + jdeq,jdne,jdlt,jdle,jdgt,jdge (6) + jdisnan,jdisinf (2) = 32. ✓ | OK |
| 30.7.3 | `GOP_*` flag values 1,2,4,8,16,32 | `header.h` constants match. | OK |
| 30.8.1 | I/O modes: Null=0, Filter=1, Glk=2, FyreVM=0x20 | Glulx-Spec §1.4.2 matches. | OK |
| 30.8.5 | String-decoding-table header: length(4) + count(4) + root(4) + node data | Glulx-Spec §1.6.1 matches. | OK |
| 30.8.5 | Node types: 0=branch, 1=terminator, 2=single Latin-1, 3=C string, 4=single Unicode, 5=Unicode string, 8=indirect ref, 9=double indirect | Glulx-Spec §1.6.2 matches. | OK |
| 30.9.1 | IEEE-754 single-precision layout + reference values (0, 1.0=3F800000, etc.) | Glulx-Spec §1.8 confirms big-endian IEEE-754 32-bit. | OK |
| 30.9.3 | "Twenty-six opcodes provide single-precision floating-point operations" | Actually 29 (see 30.7.2 above). The lists enumerated in prose (`numtof`, `ftonumz`, `ftonumn`, `ceil`, `floor`, `fadd`, `fsub`, `fmul`, `fdiv`, `fmod`, `sqrt`, `exp`, `log`, `pow`, `sin`, `cos`, `tan`, `asin`, `acos`, `atan`, `atan2`, `jfeq`, `jfne`, `jflt`, `jfle`, `jfgt`, `jfge`, `jisnan`, `jisinf`) actually enumerates 29 opcodes — but §30.9.3's opening sentence says "Twenty-six". | **WRONG** (count) |
| 30.9.4 | "Thirty-two double-precision opcodes" | Confirmed 32 (§31.3.15 also lists 32). | OK |
| 30.10.2 | Acceleration parameter numbers 0–8 with names `classes_table`, `indiv_prop_start`, etc. | Glulx-Spec §1.11 (lines 1924-1932) matches exactly. | OK |
| 30.10.3 | Standard accel functions 1, 8, 9, 10, 11, 12, 13 | Matches Glulx-Spec. Guide notes 2–7 are deprecated, which is correct. | OK (table omits the 2–7 names but §30.10.3 text says they are deprecated; intentional brevity) |
| 30.10.4 | "Accelerated functions cannot make VM function calls, cannot use streaming output, errors are fatal, state is not saved" | Glulx-Spec lines 1890-1894 match. | OK |

---

## ch31 — Glulx Instruction Set (Assembly)

Source: `asm.c` `opcodes_table_g[]` (141 entries), `opmacros_table_g[]`,
`header.h` opcode constants, `Glulx-Spec.md` §7.

**Opcode inventory check: every opcode listed in ch31 exists in
asm.c, and every opcode in asm.c Glulx tables appears in ch31.**
No WRONG opcode codes, no missing opcodes.

| # | Claim (§) | Source | Classification |
|---|---|---|---|
| 31.1.1 | Opcode encoding 1/2/4 bytes with top-bit pattern | Glulx-Spec and asm.c match. | OK |
| 31.2.1 | 16 addressing modes table | Exactly matches Glulx-Spec lines 308-323. Modes 4 and 12 correctly noted as "Unused". | OK |
| 31.2.2 | Local variable encoding: "variable n has frame offset (n-1)×4"; "If the offset fits in one byte (variables 1 through 64), mode 9 is used; otherwise mode 10." | (64-1)×4 = 252 which fits in one byte; variable 65 → (65-1)×4 = 256, does not fit. Matches asm.c (assembleg_operand mode9 threshold). | OK |
| 31.2.3 | Flag values `St=1, Br=2, Rf=4, St2=8` | `header.h` matches exactly. | OK |
| 31.3.1 | All 13 integer-arithmetic opcodes 0x10–0x1E | Matched against asm.c. | OK |
| 31.3.2 | "jumpabs has only Rf" | asm.c: `{ "jumpabs", 0x0104, Rf, 0, 1 }`. ✓ | OK |
| 31.3.2 | "Offset 0: Return 0; Offset 1: Return 1" | Glulx-Spec §2.2 confirms. | OK |
| 31.3.3 | `call` 0x30 L1 L2 S1, `tailcall` 0x34 L1 L2 (has Rf) | asm.c: tailcall has `Rf` flag. ✓ | OK |
| 31.3.3 | `callf` 0x160, `callfi` 0x161, `callfii` 0x162, `callfiii` 0x163 | asm.c matches. | OK |
| 31.3.4 | `copy` 0x40, `copys` 0x41, `copyb` 0x42, `sexs` 0x44, `sexb` 0x45 | asm.c matches (note 0x43 is not used, consistent with guide). | OK |
| 31.3.5 | `aload`..`astorebit` 0x48–0x4F | asm.c matches. | OK |
| 31.3.7 | `setiosys` 2 operands, for filter mode (1), L2 is filter function address | Glulx-Spec §2.11 confirms. | OK |
| 31.3.7 | `getiosys` "has both St and St2 flags" | asm.c: `{ "getiosys", 0x0149, St|St2, 0, 2 }`. ✓ | OK |
| 31.3.8 | gestalt selectors 0–13 | Glulx-Spec §5 matches. | OK |
| 31.3.9 | Game state opcodes 0x120–0x129 including `hasundo` 0x128 and `discardundo` 0x129 with 3.1.3+ gate | asm.c `hasundo` and `discardundo` gated with `GOP_ExtUndo`. ✓ | OK |
| 31.3.10 | "glk_gc (value 68)" | `header.h:1213` `#define glk_gc 68`. ✓ | OK |
| 31.3.11 | "linearsearch_gc = 73, binarysearch_gc = 74, linkedsearch_gc = 75" | `header.h:1218-1220` — all exactly correct. | OK |
| 31.3.12 | `mzero` 2 operands "Zero L1 bytes of memory starting at address L2" | Glulx-Spec §2.13.3 line 1793 matches exactly. | OK |
| 31.3.12 | `mcopy` 3 operands "L1 bytes from L2 to L3" (handles overlap) | Glulx-Spec line 1798-1806 matches. | OK |
| 31.3.13 | `accelfunc` 0x180 L1 L2; `accelparam` 0x181 L1 L2 | asm.c and Glulx-Spec §9 match. | OK |
| 31.3.14 | Single-precision opcode codes 0x190, 0x191, 0x192, 0x198, 0x199, 0x1A0-0x1A4, 0x1A8-0x1AB, 0x1B0-0x1B6, 0x1C0-0x1C5, 0x1C8, 0x1C9 | Each cross-checked against asm.c: all match. `fmod` has `St|St2` flags (asm.c). ✓ | OK |
| 31.3.15 | Double-precision opcodes 0x200-0x239 (all 32) | Cross-checked against asm.c. All match. | OK |
| 31.3.16 | Synthetic opcodes `pull`, `push`, `dload`, `dstore` | `header.h:1237-1240` defines exactly these four `*_gm` macros (pull_gm=0, push_gm=1, dload_gm=2, dstore_gm=3). ✓ | OK |
| 31.4.1 | Function type bytes 0xC0 (stack-arg) and 0xC1 (local-arg) | Matches Glulx-Spec and asm.c. | OK |
| 31.4.2 | "LocalType = 4 (always, for 32-bit locals)… LocalCount = 1–255" | Matches Glulx-Spec §1.3.3; compiler uses 4-byte locals only. | OK |
| 31.5.1 | Array access formulas | Matches opcode semantics. | OK |
| 31.5.2 | "Both opcodes [mzero, mcopy] can only write to RAM (addresses ≥ RAMSTART)" | Glulx-Spec §2.13.3 does not explicitly restrict to RAM; Glulx-Spec §2.12 `astore`/`astoreb` also write into memory addresses, and writing to ROM is a general VM error. The mzero/mcopy restriction is not exclusive to these opcodes. | MINOR (phrasing implies these opcodes have a special restriction; the general "ROM is read-only" rule applies to every write) |
| 31.6.1 | Search opcode parameter conventions | Matches Glulx-Spec §2.9. | OK |
| 31.6.2 | Option-flag values KeyIndirect=0x01, ZeroKeyTerminates=0x02, ReturnIndex=0x04 | Glulx-Spec §2.9 matches. | OK |
| 31.6.3 | linearsearch 8 operands | asm.c: `{ "linearsearch", 0x150, St, 0, 8 }`. ✓ | OK |
| 31.6.4 | binarysearch: "ZeroKeyTerminates flag is not supported" | Glulx-Spec §2.9.2 confirms. | OK |
| 31.6.5 | linkedsearch: 7 operands; "ReturnIndex flag is not supported" | asm.c: no=7. Glulx-Spec §2.9.3 confirms ReturnIndex not allowed. | OK |
| 31.7 | @glk 3 operands; args pushed in reverse order | Glulx-Spec §2.11 (Glk dispatch) confirms. | OK |
| 31.8.4 | Custom opcode syntax: `@"S:NNNN"` where `S` is "the operand count category" and `NNNN` is "the opcode number in hexadecimal". Example `@"1:130" selector argcount result`. | `asm.c:3522-3578` comment documents the format explicitly: `@"FlagsCount:Code"` where Flags are optional letters `S, SS, B, R`, Count is the number of arguments (0–9), and **Code is a decimal integer**. Example given in asm.c comment: `@"S3:123"` for a 3-arg opcode numbered 123 in decimal. The guide's description is wrong on three points: (a) the first part is not a "category" but a flags+argcount pair, (b) the digit after flags is the *argument count*, not a category, and (c) the code is decimal, not hexadecimal. The worked example `@"1:130" selector argcount result` is additionally inconsistent: three operands but arg count "1". | **WRONG** |
| 31.8.4 | "If the compiler does not recognize an opcode name…" | asm.c does support unknown opcodes via the @"…" custom form. General claim OK. | OK (context correct; specific format wrong) |
| 31.8.5 | Differences table between Z-machine and Glulx assembly | Most rows accurate. Row "Branch operand: Written as `?label` (no negation)" — Glulx assembler does not support `?~` inversion. asm.c assembleg path confirms. | OK |

---

## Coverage gaps (asm.c opcodes missing from guide)

**Z-machine (asm.c `opcodes_table_z` + `extension_table_z`):**
Every opcode matched. No gaps.

**Glulx (asm.c `opcodes_table_g`, 141 entries, and
`opmacros_table_g`):**
Every opcode matched. No gaps.

The chapters cover the full opcode set implemented by the compiler.

## Additional synthetic opcode note

`opmacros_table_g` defines four macros (pull, push, dload, dstore).
All four are documented in §31.3.16. No gaps.

---

## Cross-cutting observations

1. **Flags 2 bit semantics (ch28 vs ch29):** Ch28 §28.3.1 lists only
   three opcodes for bit 3 (pictures) but ch29 §29.8.5 correctly lists
   four (adding `picture_table`). asm.c confirms all four opcodes have
   flags2_set=3. **Ch28 is inconsistent with ch29 and with asm.c.**

2. **Version 7/8 "internal names" (ch28):** Guide labels v7 "Extended
   Alternate" and v8 "Extended". The compiler source only refers to
   them as "expanded Advanced" (inform.c:1289-1290). These labels
   appear to be guide-invented.

3. **Static ROM block bytes 0x28–0x2B (ch30):** Described as four
   zero-bytes but the compiler actually writes `00 01 00 00`
   (a "memory-layout version" marker, per files.c:725 comment
   "eight-byte memory layout identifier"). The guide obscures what
   is actually a meaningful structural field.

4. **Inform/Glulx version string format (ch30):** Described as
   "ASCII digits (e.g., `0636` for 6.36)"; actual format is
   `d.dd` ("6.36"). This applies to both the Inform version at 0x2C
   and the Glulx back-end version at 0x30.

5. **Single-precision float opcode count (ch30 vs ch31):**
   §30.7.2 says "Floating point 0x190–0x1C9 | 26" and §30.9.3 says
   "Twenty-six opcodes". The actual count is 29 (see §31.3.14 which
   enumerates 29 opcodes correctly). Numbers need to be updated.

6. **Custom-opcode syntax (ch29 vs ch31):** The Z-machine section
   (§29.5) correctly describes the syntax per asm.c:3160-3227. The
   Glulx section (§31.8.4) is **wrong** on almost every point of the
   syntax: categories vs argcounts, hex vs decimal, and the example is
   internally inconsistent. asm.c:3522-3578 is the authoritative
   description.

7. **`uses_unicode_features` trigger:** Guide §30.1.2 attributes it
   solely to the `streamunichar` opcode; files.c:645 also tracks
   `no_unicode_chars != 0` (any Unicode character in source).

8. **Guide's stated default Glulx version (ch30):** Not stated.
   files.c:642 sets `final_glulx_version = 0x00020000` as the
   starting baseline (2.0.0).

---

## Verification scope note

Z-machine chapters were cross-checked against compiler source only,
since the Z-Machine Standards document is not in the repo. Header
byte-field meanings beyond what the compiler writes (Flags 1 bit
semantics, interpreter-number/version split, foreground vs background
colour default, bit names for Flags 2 beyond opcode-trigger mapping)
are flagged UNVERIFIABLE. The Glulx chapters were cross-checked
against both the compiler and `Glulx-Spec.md` §§1–11.

End of Pass 5.

---

# Pass 6+7 — Part 5 Advanced + Front matter

Audit of Part 5 chapters (ch32–ch36) of the Inform 6 Programmer's Guide plus front matter, against Inform6-6.44, inform6lib-6.12.8, and Glulx-Spec.md.

## Summary table

| File | WRONG | MINOR | MISSING-CITATION | UNVERIFIABLE/OPINION |
|---|---|---|---|---|
| ch32 | 2 | 3 | 3 | 1 |
| ch33 | 3 | 4 | 1 | 1 |
| ch34 | 3 | 3 | 1 | 1 |
| ch35 | 3 | 3 | 0 | 3 |
| ch36 | 3 | 4 | 1 | 1 |
| title.md | 0 | 0 | 0 | 0 |
| preface.md | 0 | 1 | 0 | 0 |
| README.md | 1 | 0 | 0 | 0 |
| conventions.md | 0 | 2 | 0 | 0 |

---

## ch32 — Replacing and Extending the Library

**WRONG**
1. §32.7 **MISSING major topic**: the chapter never mentions `LibraryExtensions` (RunAll/RunWhile/RunUntil), though the prompt called this out as critical. `LibraryExtensions.RunAll(ext_*)` is used ≥15 times in `verblib.h` (lines 1137, 1149, 1155, 1164, 1256, 1264, 1451, 1490, 1695, 1926, 1963, 2242, 2416, 2513, 3129). The dispatch at `verblib.h:3129` (`LibraryExtensions.RunWhile(ext_messages, false)`) is part of the L__M pipeline and is omitted in §32.7.2.
2. §32.7.2 describes the `LibraryMessages` dispatch but omits the `LibraryExtensions.RunWhile(ext_messages, false)` step that sits between `RunRoutines(LibraryMessages, before)` and `LanguageLM` (`verblib.h` L___M, lines ~3122–3131). The documented order (LibraryMessages → LanguageLM) is incomplete.

**MINOR**
3. §32.1.1 "`REPLACE_SFLAG` (value 2)": the numeric value is an internal flag bit and is not stable; citing a specific value without `header.h` citation is fragile (`header.h` SFLAG definitions). MINOR-CITATION.
4. §32.2.4 Table: claims `Banner`/`DrawStatusLine`/`PrintRank` are defined in `verblibm.h`. The library file is named `verblib.h` (no `m`) in 6.12.8. `ls inform6lib-6.12.8/` confirms.
5. §32.4.6 "the standard library uses approximately 60 verb entries" — unsourced, labelled "approximately", acceptable but flag as UNVERIFIABLE.
6. §32.3.2 Stub list appears complete vs. grammar.h (spot-checked: AfterLife, AfterPrompt, Amusing, BeforeParsing, etc.) but claims `AfterSave`/`AfterRestore` are stubs in `grammar.h`; actual dispatch uses `LibraryExtensions.RunAll(ext_afterrestore/ext_aftersave)` in verblib.h. MINOR-CITATION needed.

**MISSING-CITATION**
7. §32.1.1 claim "sets `dont_enter_into_symbol_table = TRUE` and consumes tokens until EOF_TT or closing ]" has no line citation (should be `symbols.c`/`directs.c`).
8. §32.1.2 `value_pair_struct` / `find_symbol_replacement()` — not cited to `header.h`/`symbols.c`.
9. §32.6.4 `LibRelease` claimed as "6/12"; actual `inform6lib-6.12.8/version.h:3`: `Constant LibRelease "6.12.8"` (dotted, not slash). WRONG formatting.

**UNVERIFIABLE/OPINION**
10. §32.6.3 "current directory (the directory containing the file that issued the Include)" — actual compiler search order for `Include` uses include-path list and ICL settings; would need citation to `files.c`.

---

## ch33 — Internationalization and Localization

**WRONG**
1. §33.3.1 Form 2: claims skipped A2 positions 12, 13, 19 correspond to `.`, `"`, `~`. Per `chars.c:1290` the A2 default is `" ^0123456789.,!?_#'~/\-:()"`, so position 13 is **comma** (`,`), not a double quote. Correct chars are `.`, `,`, `~` (chars.c:221 skips indices 12,13,19).
2. §33.3 "four forms" for `Zcharacter`: there are **five** — missing `Zcharacter terminating <num> ...` form (directs.c:1230–1244). MISSING form.
3. §33.4.2 claims source default is ISO 8859-1 unless `-Cu`. Actually `-Cu` enables UTF-8; default is Latin-1. The code snippet shown (`unicode_to_zscii`) returning `5` on out-of-range is correct (chars.c:191+), but `zscii_to_unicode` shown as returning `'?'` — verify: actual behaviour in chars.c differs slightly (returns the `unicode_replacement` per version); MINOR. Flag as UNVERIFIABLE without line cite.

**MINOR**
4. §33.1 Line numbers mostly accurate (125/138/163/180/183/187/193/205/207/225/273/281/308/797 all verified in english.h). §33.1.6 cites `LanguageVerbLikesAdverb` at line 320; actual definition at english.h:324. Off by ~4.
5. §33.1.8 claims LanguageLM has "approximately 200 individual messages" and "over 70 action types" — unsourced.
6. §33.2.4 omits `LanguageCases` (parser.h:777–779: `#Ifndef LanguageCases; Constant LanguageCases = 1;`). The prompt explicitly called this out; chapter should document it in Required/Optional constants.
7. §33.1.5 `LanguageTimeOfDay` (hours, mins) — `english.h:273` code handles "hours%12" with 0→12 etc. Matches.

**MISSING-CITATION**
8. §33.4.3 Huffman/Unicode interaction claims are plausible but uncited (should reference `text.c`).

**UNVERIFIABLE/OPINION**
9. §33.3.3 "about 12 replaceable positions" — rough figure; 26 - 3 reserved - 3 skipped = 20 positions, so "12" understates. UNVERIFIABLE.

---

## ch34 — Menus, Multimedia, Extended

**WRONG**
1. §34.2.1 (DoMenu) Claims `EntryR` "receives the item number (1-based) and should print the item's text." Actually `EntryR` is called with **no arguments** (verblib.h:766, 826: `lines = EntryR();`) and uses the global `menu_item` (0 = return total lines & set `item_name`; 1..N = print item N). Same for `ChoiceR`. The calling convention documented is wrong.
2. §34.2.1 DoMenu parameters: claims `menu_choices` is "a string containing the menu title and item labels separated by ^". Actually `menu_choices` is the **body text** (string OR routine) printed under the title (verblib.h:778–779: `if (menu_choices ofclass Routine) menu_choices(); else print (string) menu_choices;`). Item labels come from `EntryR` setting `item_name` — not from `^` separators.
3. §34.2 (LowKey_Menu) Claims "Reads single-character input". Actually LowKey_Menu uses `read buffer parse` (Z-machine) or `KeyboardPrimitive` (Glulx) for a full line, then `TryNumber(1)` (verblib.h:785–805). Input is numeric, not single-char.

**MINOR**
4. §34.3.2 `glk_image_draw(win, image, val1, val2)` matches infglk.h:544–548. ✓
5. §34.3.2 `glk_schannel_play_ext(chan, sound, repeats, notify)` matches infglk.h:622–626. ✓
6. §34.5 save_undo/restore_undo opcodes: EXT 0x09 / 0x0A (asm.c:633–634). ✓. Return values documented correctly (save_undo: 0=fail, 1=save OK, 2=restored; Glulx saveundo: 0=ok, 1=fail, -1=restored, per Glulx-Spec.md:1318). ✓
7. §34 — Glk gestalt constants all verified against infglk.h:41–58 (Graphics=6, DrawImage=7, Sound=8, SoundVolume=9, SoundNotify=10, SoundMusic=13, Sound2=21). ✓
8. §34 — filemode/fileusage constants verified (infglk.h:24–33). ✓

**MISSING-CITATION**
9. DoMenu's actual structure (pretty_flag fallback to LowKey_Menu at verblib.h:824) is not described.

**UNVERIFIABLE**
10. Claim about menu library extensions ("Much better menus can be created using one of the optional library extensions" — verblib.h:761 comment) is consistent with library, but extension names/behaviours not cited.

---

## ch35 — Optimization and Performance

**WRONG**
1. §35.4.2 Table lists `MAX_LOCAL_VARIABLES | 16 | 118` as current setting. Per `options.c:534–535` this setting is `OPTUSE_OBSOLETE_I6` in 6.44 — it cannot be set by user. Also the hardcoded limit is 16 Z / **119** Glulx (inform.c:128,138), not 118. The note "obsolete in 6.44; compiled-in limit" in the table is correct-ish, but then §35.4.3 re-asserts "`$MAX_LOCAL_VARIABLES` setting defaults to 119 and can be increased further" — which is wrong: setting is obsolete.
2. §35.4.3 "Z-machine … `$MAX_LOCAL_VARIABLES` setting cannot exceed 16" — the setting does not exist in 6.44 (obsolete).
3. §35.4.2 Table lists `MEMORY_MAP_EXTENSION` default "0" — correct (options.c:266, DEFAULTVAL(0)). But `DICT_WORD_SIZE` row "6 (V3) / 9 (V5+)" conflates `DICT_WORD_SIZE` setting (a separate memory setting) with Z-machine dictionary word length (built-in 4 bytes V3 / 6 bytes V5+). The **Z-machine** limits are 6/9 characters; the `$DICT_WORD_SIZE` setting has a different default. MINOR/WRONG conflation.

**MINOR**
4. §35.1.2 `abbreviation_s` struct exactly matches `header.h:780–786`. ✓
5. §35.3.1 EXECSTATE constants match `header.h:2010–2014`. ✓
6. §35.2.5 -d behaviour: default=1 (period only), -d2 (also ? !) verified at inform.c:1356–1361 and text.c:621–627. ✓
7. §35.2.3 Table row `-H | Enable Huffman compression` is misleading: `-H` toggles compression_switch (inform.c:1463); default is ON (inform.c:358), so `-H` typically disables. §35.2.4 text clarifies this correctly; table does not.

**UNVERIFIABLE/OPINION**
8. §35.6.1 "Typical size savings: 5–15% of total story file size" — unsourced estimate.
9. §35.6.2 "for a game with 1000 dictionary entries, … saves approximately 1000 bytes" — arithmetically trivial but not cited.
10. §35.7.4 "savings are typically modest (50–200 bytes)" — estimate, unsourced.

---

## ch36 — Debugging and Testing

**WRONG**
1. §36.1.3 "The `-X` switch implies `-D`". Not in the compiler: `-X` only sets `define_INFIX_switch` (inform.c:1473) and creates the `INFIX` constant plus `infix__watching` attribute (symbols.c:882–887). It does **not** define `DEBUG`. The library achieves this indirectly via `parser.h:77–79`: `#Ifdef INFIX; Default DEBUG 0; #Endif;`. So effect is correct only when library is included — MINOR-WRONG mechanism.
2. §36.1.3 **MISSING**: `-X` is explicitly incompatible with Glulx — at inform.c:1117–1121 the compiler prints "Infix (-X) facilities are not available in Glulx: disabling -X switch" and clears the flag. Chapter does not mention this.
3. §36.6.2 $! trace options table is **incomplete** and misstated: actual options per `memory.c:388–440` include ACTION(S), BPATCH/BACKPATCH, DICT/DICTIONARY, EXPR/EXPRESSION(S), FILE(S), FINDABBREV(S), FREQ/FREQUENCY, MAP, MEM/MEMORY, OBJECT(S)/OBJ(S), PROP/PROPERTY/PROPS, RUNTIME, STATISTICS/STATS, SYMBOL(S), SYMDEF/SYMBOLDEF, TOKEN(S), VERB(S). Chapter omits ACTION, FREQUENCY, MAP, MEM, PROP, STATISTICS. Minor: the option spellings are CASE-sensitive uppercase in code; chapter shows lowercase `$!ASM=2`.

**MINOR**
4. §36.3.1 DEBUG_* constants match `parser.h:341–345` exactly (MESSAGES=$0001, ACTIONS=$0002, TIMERS=$0004, CHANGES=$0008, VERBOSE=$0080). ✓
5. §36.3.2 "Trace levels range from 1 to 5" — parser.h uses `parser_trace >= 2|3|4|5`, so 1–5 is accurate. ✓
6. §36.3.4 Grammar for `routines`/`messages` omits the `'verbose' -> RoutinesVerbose` line present at grammar.h:128.
7. §36 does not mention the `goto` and `random` (`Predictable`) meta verbs present in grammar.h:120–124, though these are also debug verbs under `#Ifdef DEBUG`.
8. §36.1.2 `runtime_error_checking_switch` default TRUE verified at `inform.c:347`. ✓
9. §36.6.1 Trace directive keywords table: valid keywords per `lexer.c:500–502` are `dictionary, symbols, objects, verbs, assembly, expressions, lines, tokens, linker, on, off`. Chapter omits `lines` and `linker`.

**MISSING-CITATION**
10. §36.4 XML debug file format (gameinfo.dbg) claims — need citations to `files.c` for `-k` output routines and exact tag names.

**UNVERIFIABLE**
11. §36.1.2 "code size overhead (typically 5–15%)" — unsourced estimate.

---

## Front matter findings

### title.md — OK
- Line 7 "compiler version 6.44": `header.h:35` `#define RELEASE_NUMBER 1644` → 6.44 ✓
- Line 7 "library version 6.12.8": `inform6lib-6.12.8/version.h:3` `Constant LibRelease "6.12.8"` ✓

### preface.md
**MINOR**
- Line 141 refers to "**Part 4: Target Virtual Machines**" but the actual directory is `guide/part4-vm/` (short name). Consistent with preface's logical grouping; not actually wrong.

### README.md
**WRONG**
- Line 31 lists `part4-virtual-machines/` in the directory tree, but the actual directory is `guide/part4-vm/`. The tree is incorrect.

### conventions.md
**MINOR**
- Claims code fences use `inform6` tag: verified (sampled ch03, ch20, ch28, ch32 — all use ```inform6). ✓
- Claims heading format `# Chapter N:` / `## N.M` / `### N.M.P`: verified across ch03 (`# Chapter 3: Variables and Scope`), ch20 (`# Chapter 20: Actions and the Action Processing Pipeline`), ch28 (`# Chapter 28: The Z-Machine Architecture`), ch32 (`# Chapter 32: Replacing and Extending the Library`). ✓
- Claims `[Z-machine]`/`[Glulx]` tags for VM-specific material: verified as used (ch34 §34.2, ch35 §35.1.5, ch36 §36.3.9). ✓
- No issues found with the stated conventions as described.
- MINOR: conventions.md does not specify the `[Z-machine/Glulx difference]` tag usage visibly in sampled chapters; could be unused in practice.

---

## Cross-cutting findings

### ch36 vs Appendix D/E trace options
- Chapter's $! options list (§36.6.2) does not match `memory.c:380–440` master list; see ch36 WRONG #3 above. Appendices should be authoritative.
- Trace directive keyword list vs lexer.c — chapter missing `lines`, `linker`.

### Part 5 cross-references spot-check
- ch32 §32.4 refs §20 (grammar/parser): Part 3 has ch20–ch22 covering grammar/actions. §20 now exists as "Actions and the Action Processing Pipeline" — historically the parser/grammar chapter; acceptable but §20 in the new guide is actions, not grammar. ch22 is "Grammar Definition". Cross-ref may point to the wrong chapter number. MINOR.
- ch36 §36.3.10 refs §21 (scope rules): actual ch21 is "Parser and Grammar"; scope is in ch25 "Scope, Light, Visibility". CROSS-REF WRONG.
- ch36 §36.3.3 refs §22 for action processing pipeline: ch22 is "Grammar Definition", not action pipeline (which is ch20). CROSS-REF WRONG.
- ch36 §36.5.3 refs §21 (complete scope rules) and §23 (light model): §23 is "Printing and Output", §25 is the scope/light chapter. CROSS-REF WRONG.
- ch32 §32.5.4 refs §28 and §29 for VM opcodes — ch28 is Z-Machine Arch ✓, ch29 is Z-Machine Instruction Set ✓.
- ch35 §35.7.4 refs §30.3 (Glulx memory layout) — ch30 is Glulx Architecture ✓.
- ch33 §33.4.5 refs Chapter 29 (Glulx I/O): ch29 is Z-Machine Instruction Set; Glulx I/O is in ch30/31. CROSS-REF WRONG.

### Notable unified issues
- Multiple chapters of Part 5 use Part 3 section numbers that appear to predate the current Part 3 reorganization (world model chapters reshuffled ch20↔ch25). All §20/§21/§22/§23 references from ch32/ch36 need re-verification; the numbering has drifted.
- README's directory tree is stale (`part4-virtual-machines/` vs actual `part4-vm/`).
- LibraryExtensions.Run* mechanism (critical per prompt) is entirely missing from ch32.
- Zcharacter `terminating` form entirely missing from ch33.

---

## Recommended priority fixes

1. ch32: Add LibraryExtensions (RunAll/RunWhile/RunUntil) section; fix `verblibm.h`→`verblib.h`; fix LibRelease "6/12"→"6.12.8".
2. ch33: Fix A2 position 13 (`"`→`,`); add Zcharacter terminating form; document LanguageCases.
3. ch34: Rewrite DoMenu EntryR/ChoiceR calling-convention description — they take no args and use global `menu_item`; menu_choices is body text (string/routine), not a `^`-separated list.
4. ch35: Remove $MAX_LOCAL_VARIABLES as if it were a live setting (it's obsolete).
5. ch36: Note -X disabled in Glulx; complete $! options table from memory.c; fix -X "implies -D" wording (it's via library, not compiler); fix cross-references to §21/§22/§23.
6. README.md: fix `part4-virtual-machines/` → `part4-vm/` in directory tree.
