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
