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
