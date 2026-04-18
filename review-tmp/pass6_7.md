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
