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

# Chapter 9: The Dictionary

This chapter describes the dictionary: a compile-time data structure that
maps textual words to compact binary entries used by the parser at runtime.
Every single-quoted word that appears in source code creates or references
an entry in the dictionary, and the parser consults this table to recognize
words typed by the player.

## 9.1 Overview

The dictionary is a sorted table of word entries built by the compiler and
embedded in the output story file. Its primary purpose is to allow the
runtime parser to look up words from the player's input and determine what
they mean — whether a word is a verb, a noun, a preposition, or some
combination of these.

Every single-quoted string in source code (such as `'lamp'` or `'take'`)
causes the compiler to create or locate an entry in the dictionary. The
compiler assigns flag bits to each entry based on how the word is used:
words appearing in `Verb` directives receive verb and preposition flags,
words in `name` properties receive noun flags, and so on. At runtime, the
parser tokenizes the player's input, looks up each token in the dictionary,
and uses the flags and associated data to interpret the command.

Dictionary entries are stored in alphabetical order and encoded in a
compact binary format that differs between the Z-machine and Glulx virtual
machines. The compiler handles all encoding automatically; the programmer
interacts with the dictionary through single-quoted word literals, the
`Dictionary` directive, and a small set of compiler constants for
low-level access.

```inform6
Object lamp "brass lamp"
  with name 'brass' 'lamp' 'lantern',
       description "A well-polished brass lamp.";

[ Main;
    ! Each word in 'name' creates a dictionary entry.
    print "'lamp' is at address ", 'lamp', "^";
];
```

## 9.2 Dictionary Word Literals

A **dictionary word literal** is a word enclosed in single quotes. It
evaluates at runtime to the address of the corresponding entry in the
dictionary table. If the word does not yet exist in the dictionary, the
compiler creates a new entry for it.

```inform6
if (w == 'take') print "You said take!^";
if (w == 'get')  print "You said get!^";
```

Dictionary words appear most commonly in three contexts:

1. **`name` properties** — listing the words that the parser should
   recognize as referring to an object:

   ```inform6
   Object -> sword "rusty sword"
     with name 'rusty' 'sword' 'blade' 'weapon';
   ```

2. **`Verb` directives** — defining the grammar that the parser uses to
   interpret player commands:

   ```inform6
   Verb 'take' 'get' 'grab'
       * noun -> Take
       * 'up' noun -> Take;
   ```

3. **Explicit comparisons** — testing a parsed word against a known
   dictionary entry in game code:

   ```inform6
   [ TryDirection w;
       if (w == 'north' or 'south' or 'east' or 'west')
           print "That's a compass direction.^";
   ];
   ```

### 9.2.1 Case Insensitivity

Dictionary lookups are case-insensitive. The words `'Take'`, `'take'`, and
`'TAKE'` all refer to the same dictionary entry. This matches the
behavior of the runtime parser, which converts player input to lowercase
before performing dictionary lookups.

### 9.2.2 Double-Quoted Dictionary Words

In most contexts only single-quoted tokens create dictionary entries.
However, the `Dictionary` directive (§9.6) also accepts double-quoted
strings as dictionary words, treating the entire string as a single
dictionary entry.

## 9.3 Word Length Limits

Dictionary words have a maximum number of significant characters. Any
characters beyond this limit are silently truncated, which means that two
words differing only after the cutoff point will map to the same dictionary
entry.

### 9.3.1 Z-Machine Limits

> **[Z-machine]** In Z-machine version 3, only the first **6 characters**
> of a dictionary word are significant (though internally only 4 bytes of
> Z-encoded text are stored). In versions 4 and above, the first **9
> characters** are significant (6 bytes of Z-encoded text).

This means that in a version 5 game (the most common Z-machine target),
the words `'pineapple'` (9 characters) and `'pineapples'` (10 characters)
are encoded identically — the trailing `'s'` is lost:

```inform6
! In Z-machine v5, these refer to the SAME dictionary entry:
Object -> pineapple "pineapple"
  with name 'pineapple' 'pineapples';
  ! The compiler may warn that 'pineapples' is truncated.
```

The Z-machine word size is fixed by the virtual machine specification and
cannot be changed by the programmer.

### 9.3.2 Glulx Limits

> **[Glulx]** On Glulx, the number of significant characters is controlled
> by the `$DICT_WORD_SIZE` setting (default: **9**). The byte size of each
> character is controlled by `$DICT_CHAR_SIZE` (default: **1**; set to
> **4** for full Unicode support).

Because Glulx allows configuration, the programmer can increase the
dictionary word size to avoid truncation problems. Setting `$DICT_CHAR_SIZE`
to 4 enables full Unicode dictionary words:

```inform6
!% $DICT_WORD_SIZE=12
!% $DICT_CHAR_SIZE=4
! First 12 characters significant, 4 bytes per character (Unicode).
```

The total byte size of the encoded word portion of each dictionary entry is
`DICT_WORD_SIZE × DICT_CHAR_SIZE`.

### 9.3.3 The Truncation Flag

When a dictionary word is longer than the significant character limit, the
compiler can optionally set the `TRUNC_DFLAG` (bit 6) on the entry to
indicate that truncation occurred. This behavior is controlled by the
`$DICT_TRUNCATE_FLAG` setting:

```inform6
!% $DICT_TRUNCATE_FLAG=1
! Truncated dictionary entries will now have bit 6 set in their flags.
```

When `$DICT_TRUNCATE_FLAG` is 0 (the default), bit 6 is instead used to
mark verb words (legacy behavior). Enabling the truncation flag can be
useful for games that want to warn players when their input was truncated.

## 9.4 Dictionary Flags

Each dictionary entry has a set of associated flag bits stored in the first
data field (`#dict_par1`). These flags describe how the word is used in the
game's grammar and object definitions. The compiler sets most flags
automatically based on where the word appears in source code.

| Flag | Value | Meaning |
|------|-------|---------|
| `VERB_DFLAG` | 1 | Word is used as a verb in grammar |
| `META_DFLAG` | 2 | Word is a meta-verb (always paired with `VERB_DFLAG`) |
| `PLURAL_DFLAG` | 4 | Word is a plural form (set by `//p` suffix) |
| `PREP_DFLAG` | 8 | Word is used as a preposition in grammar |
| `SING_DFLAG` | 16 | Word is a singular form (set by `//s` suffix) |
| `TRUNC_DFLAG` | 64 | Word was truncated (if `$DICT_TRUNCATE_FLAG` enabled) |
| `NOUN_DFLAG` | 128 | Word is used as a noun (set by `//n` suffix) |

The flags are bit values and can be combined. For example, a word that is
both a verb and a preposition will have both `VERB_DFLAG` and `PREP_DFLAG`
set (value 9).

### 9.4.1 How Flags Are Set

The compiler sets flags automatically based on context:

- **`VERB_DFLAG`** — set on the first words of a `Verb` directive (the
  verb synonyms).
- **`META_DFLAG`** — set (together with `VERB_DFLAG`) on words defined
  with `Verb meta`.
- **`PREP_DFLAG`** — set on single-quoted words appearing within grammar
  lines (preposition tokens).
- **`NOUN_DFLAG`** — set on words appearing in `name` properties,
  on any single-quoted dictionary word literal that appears as a value
  in an expression (such as in `if (w == 'lamp')`), or explicitly via
  the `//n` suffix.
- **`PLURAL_DFLAG`** and **`SING_DFLAG`** — set via the `//p` and `//s`
  suffixes respectively (see §9.5).
- **`TRUNC_DFLAG`** — set by the compiler when a word is truncated and
  `$DICT_TRUNCATE_FLAG` is enabled.

When the same word appears in multiple contexts, the flags are combined
with bitwise OR. A word that is both a verb and a preposition will have
both `VERB_DFLAG` and `PREP_DFLAG` set.

### 9.4.2 The Second and Third Data Fields

In addition to the flag byte (`#dict_par1`), each dictionary entry has two
more data fields:

- **`#dict_par2`** — reserved for the verb number. When a word has
  `VERB_DFLAG` set, this field contains the index into the grammar table
  for the verb's grammar. The programmer should not modify this field
  directly.

- **`#dict_par3`** — used for preposition numbers in grammar version 2.
  In many configurations this field is unused and zero. It may be omitted
  entirely in Z-code when the `$ZCODE_LESS_DICT_DATA` option is set.

## 9.5 Source Suffix Flags

Dictionary word literals can carry **suffix flags** that explicitly set or
clear specific dictionary flags on the entry. Suffixes are written after
the closing single quote, separated by `//`:

```inform6
Object -> dogs "pack of dogs"
  with name 'dogs'//p 'pack';
  ! 'dogs' has PLURAL_DFLAG set
```

### 9.5.1 Available Suffixes

The following suffixes are recognized:

| Suffix | Effect |
|--------|--------|
| `//p` | Set `PLURAL_DFLAG` (4) — marks word as plural |
| `//s` | Set `SING_DFLAG` (16) — marks word as singular |
| `//n` | Set `NOUN_DFLAG` (128) — marks word as a noun |
| `//~p` | Clear `PLURAL_DFLAG` |
| `//~s` | Clear `SING_DFLAG` |
| `//~n` | Clear `NOUN_DFLAG` |

The tilde (`~`) prefix negates the flag, explicitly clearing it even if it
would otherwise be set by context.

### 9.5.2 Combining Suffixes

Multiple suffix flags can be chained on a single word by writing one `//`
followed by the concatenated flag letters. The `~` character toggles the
negation state for the *next* flag letter, so it can be repeated to mix
set and clear operations within one suffix group:

```inform6
Object -> sheep "sheep"
  with name 'sheep'//pn;
  ! 'sheep' has both PLURAL_DFLAG and NOUN_DFLAG set
```

```inform6
Object -> geese "flock of geese"
  with name 'geese'//p 'goose'//s 'flock';
  ! 'geese' is plural, 'goose' is singular
```

A second `//` is *not* allowed inside a single word literal — the
compiler reports an error if any character other than `~`, `p`, `s`, or
`n` appears after the opening `//`.

### 9.5.3 Implicit Singular (`$DICT_IMPLICIT_SINGULAR`)

When the `$DICT_IMPLICIT_SINGULAR` setting is enabled (set to 1), the
compiler automatically adds `SING_DFLAG` to any word that has `NOUN_DFLAG`
set but does *not* have `PLURAL_DFLAG` set, and where `//~s` has not been
used to explicitly clear the singular flag.

```inform6
!% $DICT_IMPLICIT_SINGULAR=1

Object -> lamp "lamp"
  with name 'lamp';
  ! 'lamp' gets NOUN_DFLAG from the name property.
  ! Because NOUN_DFLAG is set and PLURAL_DFLAG is not,
  ! the compiler also sets SING_DFLAG automatically.
```

This feature is disabled by default (`$DICT_IMPLICIT_SINGULAR=0`). When
enabled, it provides the parser with more information about noun number,
which can be useful for games that distinguish singular and plural nouns in
their responses.

## 9.6 The `Dictionary` Directive

The `Dictionary` directive forces a word into the dictionary and
optionally sets flag values on it. It has three forms:

```
Dictionary 'word';
Dictionary 'word' val1;
Dictionary 'word' val1 val3;
```

### 9.6.1 Basic Usage

The simplest form ensures a word exists in the dictionary without
associating it with any grammar or object:

```inform6
Dictionary 'xyzzy';
! The word 'xyzzy' is now in the dictionary, even though
! it does not appear in any Verb or name property.
```

### 9.6.2 Setting Flag Values

The second form sets the first data field (`#dict_par1`) of the entry:

```inform6
Dictionary 'magic' 128;
! Creates (or updates) the entry for 'magic' with
! NOUN_DFLAG (128) set in #dict_par1.
```

The third form sets both the first and third data fields:

```inform6
Dictionary 'spell' 128 42;
! Sets #dict_par1 to 128 and #dict_par3 to 42.
```

If the word already exists in the dictionary, the flag values are combined
with the existing values using bitwise OR. There is no way to set
`#dict_par2` via the `Dictionary` directive, because that field is reserved
for verb numbers and modifying it directly would corrupt the grammar table.

### 9.6.3 Value Limits

> **[Z-machine]** In Z-code, each flag value must fit in a single byte
> (0–255, i.e. `$00`–`$FF`). The compiler issues a warning if a value
> exceeds this range.

> **[Glulx]** In Glulx, each flag value must fit in two bytes (0–65535,
> i.e. `$0000`–`$FFFF`). The compiler issues a warning if a value exceeds
> this range.

## 9.7 Dictionary Entry Format

The internal format of a dictionary entry differs between the Z-machine
and Glulx. In both cases, each entry consists of an encoded word followed
by data fields. All entries have the same fixed size, and the dictionary
table stores them in sorted order for binary search at runtime.

### 9.7.1 Z-Machine Entry Format

> **[Z-machine]** Each dictionary entry on the Z-machine contains:
>
> - **Encoded word**: 4 bytes (version 3) or 6 bytes (versions 4+) of
>   Z-character encoded text.
> - **Data fields**: The flag/data bytes that follow. In the default
>   configuration, there are 3 data bytes:
>   - `#dict_par1` (1 byte) — dictionary flags (§9.4)
>   - `#dict_par2` (1 byte) — verb number
>   - `#dict_par3` (1 byte) — preposition/adjective data
>
> In version 3, the total entry size is 7 bytes (4 + 3). In versions 4+,
> the total is 9 bytes (6 + 3). When `$ZCODE_LESS_DICT_DATA` is set, the
> third data byte is omitted, reducing the entry size by 1 byte.

The Z-character encoding compresses words using a 5-bit-per-character
scheme defined by the Z-machine specification. The compiler handles
encoding transparently.

### 9.7.2 Glulx Entry Format

> **[Glulx]** Each dictionary entry on Glulx contains:
>
> - **Type tag**: `DICT_CHAR_SIZE` bytes (1 byte with the default char
>   size). The first byte contains `$60`, identifying this as a dictionary
>   word.
> - **Encoded word**: `DICT_WORD_SIZE × DICT_CHAR_SIZE` bytes. With the
>   defaults (word size 9, char size 1), this is 9 bytes. With Unicode
>   support (char size 4), it is 36 bytes.
> - **Data fields**: 3 fields of 2 bytes (total 6 bytes):
>   - `#dict_par1` (2 bytes) — dictionary flags (§9.4)
>   - `#dict_par2` (2 bytes) — verb number
>   - `#dict_par3` (2 bytes) — preposition/adjective data
>
> With defaults, the total entry size is 16 bytes (1 + 9 + 6).

Glulx stores dictionary characters without compression: each character
occupies either 1 byte (Latin) or 4 bytes (Unicode), depending on
`$DICT_CHAR_SIZE`.

### 9.7.3 Dictionary Table Header

The dictionary table begins with a short header describing its structure:

> **[Z-machine]** The header contains: a count of word-separator
> characters, the separator characters themselves, the byte length of each
> entry, and the number of entries.

> **[Glulx]** The header contains: the number of entries (4 bytes). The
> entries follow immediately after the header, each with the same fixed
> byte length.

The `#dictionary_table` compiler constant (§9.8) gives the address of this
header.

## 9.8 Runtime Dictionary Access

The compiler provides several special constants for accessing the
dictionary at runtime. These constants are written with a `#` prefix and
are resolved at compile time.

### 9.8.1 Dictionary Address of a Word

The simplest way to reference a dictionary entry is through a word literal.
At runtime, `'word'` evaluates to the byte address of that word's entry in
the dictionary table:

```inform6
[ ShowAddress;
    print "The entry for 'lamp' is at address ", 'lamp', ".^";
    print "The entry for 'take' is at address ", 'take', ".^";
];
```

If a parsed word does not match any dictionary entry, the parser stores 0
(not a valid dictionary address) as the result.

### 9.8.2 The `#dictionary_table` Constant

The `#dictionary_table` constant gives the byte address of the start of
the dictionary table in memory. This is useful for low-level code that
needs to iterate over all dictionary entries or inspect the table
structure directly.

### 9.8.3 The `#dict_par1`, `#dict_par2`, `#dict_par3` Constants

These constants give the byte offset from the start of a dictionary entry
to each of the three data fields. They allow runtime code to read or
inspect dictionary flags and data:

```inform6
[ IsVerb w;
    ! Check whether a dictionary word has VERB_DFLAG set.
    ! #dict_par1 gives the offset to the flags field.
    if (w && (w->#dict_par1) & 1)
        rtrue;
    rfalse;
];
```

> **[Z-machine]** In version 3, the offsets are `#dict_par1` = 4,
> `#dict_par2` = 5, `#dict_par3` = 6. In versions 4+, the offsets are
> `#dict_par1` = 6, `#dict_par2` = 7, `#dict_par3` = 8.

> **[Glulx]** The offsets depend on `DICT_WORD_SIZE` and `DICT_CHAR_SIZE`.
> The `#dict_par1`, `#dict_par2`, and `#dict_par3` constants refer to the
> low bytes of the two-byte fields. For example, with defaults (word
> size 9, char size 1): `#dict_par1` = 11, `#dict_par2` = 13,
> `#dict_par3` = 15. The high bytes are at offsets one less than these
> values.

### 9.8.4 Reading Dictionary Flags at Runtime

Using the constants above, game code can inspect dictionary flags at
runtime. This is occasionally useful for debugging or for special parser
extensions:

```inform6
[ PrintWordInfo w  flags;
    if (w == 0) { print "Unknown word.^"; return; }
    flags = w->#dict_par1;
    print "Dictionary flags for word at ", w, ": ", flags, "^";
    if (flags & 1)   print "  VERB^";
    if (flags & 2)   print "  META^";
    if (flags & 4)   print "  PLURAL^";
    if (flags & 8)   print "  PREPOSITION^";
    if (flags & 16)  print "  SINGULAR^";
    if (flags & 128) print "  NOUN^";
];
```

> **[Glulx]** On Glulx, the data fields are two bytes wide. Because
> `#dict_par1`, `#dict_par2`, and `#dict_par3` point at the *low* byte
> of each field, a single-byte read (`w->#dict_par1`) returns the low 8
> bits — which is sufficient for every flag described in §9.4, since
> all standard `*_DFLAG` values fit in one byte. To read the full
> 16-bit value (only relevant when custom values larger than 255 have
> been stored via the `Dictionary` directive), use the Glulx assembly
> opcode `@aloads`, which loads a 16-bit short:
> `@aloads w (#dict_par1 - 1) flags;`. Note that the `-->` operator is
> *not* suitable here, because on Glulx it reads a 4-byte word rather
> than a 2-byte field.

## 9.9 The Dictionary and the Parser

The dictionary exists primarily to serve the parser. Understanding the
relationship between them helps explain why dictionary entries are
structured the way they are.

### 9.9.1 Input Tokenization

When the player types a command, the parser breaks the input into tokens
(words) and looks up each token in the dictionary using a binary search.
If a token matches a dictionary entry, the parser stores the address of
that entry. If no match is found, the parser stores 0 for that token.

```inform6
! In a game, if the player types: "take the lamp"
! The parser tokenizes this into: "take", "the", "lamp"
! and looks up each word in the dictionary.
! 'take' -> address of 'take' entry (has VERB_DFLAG)
! 'the'  -> address of 'the' entry (a noise word)
! 'lamp' -> address of 'lamp' entry (has NOUN_DFLAG)
```

### 9.9.2 Object Name Matching

The `name` property of an object contains a list of dictionary word
addresses. When the parser is trying to identify which object the player
is referring to, it compares the dictionary addresses of the player's
input words against the `name` lists of objects in scope.

### 9.9.3 Grammar and Prepositions

`Verb` directives define grammar patterns that the parser uses to
interpret commands. Dictionary words within grammar lines (as opposed to
the verb synonyms at the start) are treated as prepositions and receive
`PREP_DFLAG`:

```inform6
Verb 'put'
    * noun 'in' noun    -> Insert
    * noun 'on' noun    -> PutOn
    * 'down' noun       -> Drop;

! 'put' gets VERB_DFLAG (it's the verb itself)
! 'in', 'on', 'down' get PREP_DFLAG (prepositions)
```

The parser uses these flags during command interpretation: it first
identifies the verb (checking `VERB_DFLAG`), then matches the remaining
input against the grammar patterns, recognizing prepositions by their
`PREP_DFLAG`.

### 9.9.4 Unrecognized Words

When a word typed by the player does not appear in the dictionary, the
parser assigns it a dictionary address of 0. The standard Inform library
handles this with messages like "I don't understand that word."

## 9.10 Practical Considerations

### 9.10.1 Dictionary Size

There is no hard-coded limit on the number of dictionary entries in typical
configurations, but each entry consumes space in the story file. In
Z-machine games, the story file size limits can become a constraint. Glulx
games have much larger address spaces and are rarely limited by dictionary
size. The compiler reports the number of dictionary entries in its
compilation statistics.

### 9.10.2 Avoiding Truncation Pitfalls

Because dictionary words are silently truncated, it is good practice to
keep important vocabulary within the significant character limit. When two
distinct game concepts have names that differ only beyond the truncation
point, the parser cannot distinguish them:

```inform6
! In Z-machine v5 (9 significant characters), these are
! the SAME dictionary entry:
Object -> container1 "container"
  with name 'container';
Object -> containers "containers"
  with name 'containers';
! Both truncate to 'container' (9 chars).
! Solutions: use shorter synonyms, target Glulx with
! larger $DICT_WORD_SIZE, or use parse_name.
```

### 9.10.3 Dictionary Words vs. Strings

Dictionary words (single-quoted) and strings (double-quoted) are
fundamentally different things:

- `'lamp'` is a dictionary word literal — it evaluates to a dictionary
  entry address and is used for parser matching.
- `"lamp"` is a string literal — it evaluates to a packed string address
  and is used for printing text.

These two values are *not* interchangeable. Comparing a dictionary address
to a string address will never match, even if the text content is
identical:

```inform6
[ Main;
    ! These are completely different values:
    print 'lamp', "^";    ! prints the numeric dictionary address
    print "lamp", "^";    ! prints the text "lamp"

    ! This comparison is always false:
    if ('lamp' == "lamp") print "match^";   ! never prints
];
```
