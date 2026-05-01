# Part 1 Review Changelog

This file records corrections applied to `part1-language/ch01..ch10` during the
source-code validation review against Inform 6.44
(`/home/user/inform-guide/Inform6-6.44/`).

Each entry follows the format:

- `§section` — short description of the issue
  - **Before:** old text (truncated)
  - **After:** new text (truncated)
  - **Source:** `file.c:line` citation

Citations refer to files in `Inform6-6.44/`.

## Chapter 1 — Lexical Structure (2026-04-18)

- **§1.1.4 (Zcharacter alphabet sizes)** — Described all three Zcharacter
  alphabet strings as 26 chars, with first two of A2 "ignored" or
  interpreted as newline.
  - **Before:** "Each double-quoted string must contain exactly 26
    characters… The first character of the A2 string is ignored by the
    Z-machine (it is reserved as an escape code), and the second
    character is always interpreted as newline."
  - **After:** A0 and A1 need 26 chars; A2 needs exactly 23 chars and
    fills positions 3–25. Positions 0 (escape), 1 (newline), and 2
    (tilde) are fixed by Inform and not taken from the user string.
    Example A2 string updated to 23 chars.
  - **Source:** `chars.c:241-264` (`new_alphabet` requires 23 chars
    for A2, forces `alphabet[2][2] = '~'`, starts copying at index
    `i=3`); error message `chars.c:262`.

- **§1.3.1 (Lookahead)** — Called the buffer "three-character
  lookahead (current char plus two ahead)".
  - **Before:** "three-character lookahead buffer (the current
    character plus two characters ahead)"
  - **After:** "three characters of lookahead (current char plus
    three ahead, in `lookahead`/`lookahead2`/`lookahead3`)"
  - **Source:** `lexer.c:272-285` ("LR(3) grammar"; variables
    `current, lookahead, lookahead2, lookahead3`).

- **§1.3.2 (Keyword token types table)** — Omitted
  `OPCODE_MACRO_TT`.
  - **Added:** `OPCODE_MACRO_TT` (112), pseudo-opcode macros for
    Glulx (`push`, `pull`, `dload`, `dstore`).
  - **Source:** `header.h:1274` `#define OPCODE_MACRO_TT 112`;
    `lexer.c:475-485` (opcode_macros keyword group).

- **§1.8.2 (Z-machine dictionary word sizes)** — Wrote that
  `DICT_WORD_SIZE` is "fixed at 6" and "only 4 characters are used
  in v3 games", confusing bytes with characters and contradicting
  the preceding table.
  - **After:** v3 uses 4 bytes of Z-encoded text (up to 6
    characters); v4+ uses 6 bytes (up to 9 characters). The Z-code
    compiler rejects any attempt to change `DICT_WORD_SIZE`. Glulx
    `$DICT_WORD_SIZE` defaults to 9 and is configurable.
  - **Source:** `inform.c:152-155` (Z-code forces
    `DICT_WORD_SIZE = 6`); `text.c:1928` (`dictsize =
    (version_number==3) ? 6 : 9`); `symbols.c:900`
    (user-visible `DICT_WORD_SIZE` is 4 for v3 / 6 for v4+).

## Chapter 1 — Lexical Structure (2026-04-18, pass 2)

- **§1.3.3 (Lexical states count)** — Claimed "12,288 possible
  lexical states"; the `lexical_context()` function actually combines
  13 independent one-bit flags, giving 2^13 = 8,192 states.
  - **Before:** "It has 12,288 possible lexical states, determined by
    which keyword groups are currently enabled."
  - **After:** "It has 8,192 possible lexical states, determined by
    13 independent flags (11 keyword groups plus
    `return_sp_as_variable` and `dont_enter_into_symbol_table`)."
  - **Source:** `lexer.c:597-624` (`lexical_context()` ORs 13 flag
    bits 0–12 into the returned context value).

- **§1.6.2 (String escape table)** — Omitted the `@o`*X* ring-accent
  prefix used for a-with-ring (`@oa` → å) and A-with-ring
  (`@oA` → Å).
  - **Added:** a new row `@o`*X* "Accented character with ring
    (`@oa`, `@oA`)" and annotated the existing `@/`*X* row with the
    only two forms supported (`@/o`, `@/O`).
  - **Source:** `chars.c:85-88` (`accents` string contains the pairs
    `oa`, `oA`, `/o`, `/O`, matching the 69 accent escape codes
    documented in Appendix J).

## Chapter 1 — Lexical Structure (2026-05-01, pass 3)

- **§1.3.1 (Lookahead phrasing)** — The text said the lookahead window
  consists of "the current character plus three characters ahead, in
  the variables `lookahead`, `lookahead2`, and `lookahead3`", which
  conflates the four-position window with its three lookahead slots.
  - **After:** "The character currently being processed is held in
    `current`, and the three characters ahead of it are held in
    `lookahead`, `lookahead2`, and `lookahead3`."
  - **Source:** `lexer.c:284-285` (`current` is "The latest character
    read"; `lookahead`, `lookahead2`, `lookahead3` are "the three
    characters following it").

## Chapter 10 — Compiler Directives (2026-05-01)

- **§10.5.4 (`IFV3` equivalent expression)** — The chapter expanded
  `IFV3` to `#Iftrue (#version_number == 3)`, but the directive's
  predicate in the compiler is `version_number <= 3`.
  - **After:** `#Iftrue (#version_number <= 3);`
  - **Source:** `directs.c:448-451` (`IFV3_CODE`: `if (!glulx_mode &&
    version_number <= 3) flag = TRUE;`).
