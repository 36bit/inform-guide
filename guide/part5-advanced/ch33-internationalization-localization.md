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

# Chapter 33: Internationalization and Localization

The Inform 6 library separates language-dependent text and grammar from the
core parser and world model. All English-specific behaviour is isolated in
a single file, `english.h`, which defines arrays, constants, and functions
that the parser and verb library call through a well-defined interface. To
translate a game into another language, one replaces `english.h` with a new
language definition file that implements the same interface. This chapter
documents that interface in full, covers the character-set directives
available on each virtual machine, and describes the narrative voice and
tense system introduced in library 6.12.

## 33.1 The english.h Language Definition File

The file `english.h` (1583 lines in library 6.12.8) is automatically
included by `parserm.h` and provides every piece of English-specific
behaviour that the library requires. It is organised into several parts:
pronoun and descriptor tables, printing routines, text constants, pronoun
helper functions, narrative voice conjugation helpers, and the main library
message handler. This section provides a complete inventory.

### 33.1.1 Pronoun and Descriptor Arrays

The `LanguagePronouns` table array (line 125) maps English pronouns to
Gender/Number/Animacy (GNA) bit patterns. Each entry consists of a
dictionary word, a 12-bit GNA pattern, and the object the pronoun is
initially connected to (`NULL` means unset):

```inform6
Array LanguagePronouns table

  ! word        possible GNAs                   connected
  !             to follow:                      to:
  !             a     i
  !             s  p  s  p
  !             mfnmfnmfnmfn

    'it'      $$001000111000                    NULL
    'him'     $$100000100000                    NULL
    'her'     $$010000010000                    NULL
    'them'    $$000111000111                    NULL;
```

The 12-bit GNA pattern encodes which combinations of gender, number, and
animacy each pronoun can refer to. Reading left to right, the bits
correspond to: animate singular (male, female, neuter), animate plural
(male, female, neuter), inanimate singular (male, female, neuter),
inanimate plural (male, female, neuter). For example, `'it'` has pattern
`$$001000111000` — it can refer to animate singular neuter, inanimate
singular (all genders), matching objects like a dog or a table.

The `LanguageDescriptors` table array (line 138) maps words that can
precede a noun — possessives, demonstratives, and articles — to GNA
patterns, descriptor types, and connected objects:

```inform6
Array LanguageDescriptors table

  ! word        possible GNAs   descriptor      connected
  !             to follow:      type:           to:
  !             a     i
  !             s  p  s  p
  !             mfnmfnmfnmfn

    'my'      $$111111111111    POSSESS_PK      0
    'this'    $$111111111111    POSSESS_PK      0
    'these'   $$000111000111    POSSESS_PK      0
    'that'    $$111111111111    POSSESS_PK      1
    'those'   $$000111000111    POSSESS_PK      1
    'his'     $$111111111111    POSSESS_PK      'him'
    'her'     $$111111111111    POSSESS_PK      'her'
    'their'   $$111111111111    POSSESS_PK      'them'
    'its'     $$111111111111    POSSESS_PK      'it'
    'the'     $$111111111111    DEFART_PK       NULL
    'a//'     $$111000111000    INDEFART_PK     NULL
    'an'      $$111000111000    INDEFART_PK     NULL
    'some'    $$000111000111    INDEFART_PK     NULL
    'lit'     $$111111111111    light           NULL
    'lighted' $$111111111111    light           NULL
    'unlit'   $$111111111111    (-light)        NULL;
```

The descriptor types `POSSESS_PK`, `DEFART_PK`, and `INDEFART_PK` are
constants defined in the parser. The `light` and `(-light)` entries allow
the parser to understand "lit lamp" and "unlit torch" as attribute tests.

The `LanguageNumbers` table array (line 163) maps number words to their
numeric values:

```inform6
Array LanguageNumbers table
    'one' 1 'two' 2 'three' 3 'four' 4 'five' 5
    'six' 6 'seven' 7 'eight' 8 'nine' 9 'ten' 10
    'eleven' 11 'twelve' 12 'thirteen' 13 'fourteen' 14 'fifteen' 15
    'sixteen' 16 'seventeen' 17 'eighteen' 18 'nineteen' 19 'twenty' 20;
```

The parser uses this table when it encounters a word that might be a number
in the player's input, converting it to the corresponding integer.

### 33.1.2 Gender, Contraction, and Article Constants

Three constants (lines 180–183) define fundamental gender and article
properties of the language:

```inform6
Constant LanguageAnimateGender   = male;
Constant LanguageInanimateGender = neuter;

Constant LanguageContractionForms = 2;     ! English has two:
                                           ! 0 = starting with a consonant
                                           ! 1 = starting with a vowel
```

`LanguageAnimateGender` tells the library what gender to assume for animate
objects that lack explicit `male` or `female` attributes. In English, this
defaults to `male` (historically, the library defaults to "he" for
unspecified animate objects). `LanguageInanimateGender` defaults to
`neuter`.

`LanguageContractionForms` specifies how many variant forms of articles
exist. English has two: one for words beginning with a consonant ("a dog")
and one for words beginning with a vowel ("an egg"). Languages without
contraction (e.g., those with a single indefinite article form) set this
to 1.

### 33.1.3 Contraction and Article Functions

The `LanguageContraction` function (line 187) determines which contraction
form applies to a given text:

```inform6
[ LanguageContraction text;
    if (text->0 == 'a' or 'e' or 'i' or 'o' or 'u'
                or 'A' or 'E' or 'I' or 'O' or 'U') return 1;
    return 0;
];
```

It examines the first character of the object's short name. Returning 0
selects the consonant column of the article table; returning 1 selects the
vowel column.

The `LanguageArticles` array (line 193) is a two-dimensional word array
indexed by article row and contraction form. Each row contains six
entries — capitalised definite, lowercase definite, and indefinite, for
each contraction form:

```inform6
Array LanguageArticles -->

 !   Contraction form 0:     Contraction form 1:
 !   Cdef   Def    Indef     Cdef   Def    Indef

     "The " "the " "a "      "The " "the " "an "          ! Articles 0
     "The " "the " "some "   "The " "the " "some ";       ! Articles 1
```

Row 0 is used for singular objects ("a"/"an"), and row 1 for plural objects
("some"). The `LanguageGNAsToArticles` array (line 205) maps each of the
12 GNA values to an article row:

```inform6
                   !             a           i
                   !             s     p     s     p
                   !             m f n m f n m f n m f n

Array LanguageGNAsToArticles --> 0 0 0 1 1 1 0 0 0 1 1 1;
```

All singular GNA values (positions 0–2 and 6–8) map to row 0; all plural
values (positions 3–5 and 9–11) map to row 1. The library computes the
article to print by looking up an object's GNA, mapping it to an article
row via `LanguageGNAsToArticles`, and then selecting the correct column
based on the contraction form returned by `LanguageContraction`.

### 33.1.4 Direction and Number Printing

The `LanguageDirection` function (line 207) prints the English name of a
compass direction given a direction property:

```inform6
[ LanguageDirection d;
    switch (d) {
      n_to:    print "north";
      s_to:    print "south";
      ...
    }
];
```

The `LanguageNumber` function (line 225) prints an integer as English
words. It handles zero, negative numbers, thousands, hundreds, teens, and
compound forms with hyphens:

```inform6
[ LanguageNumber n f;
    if (n == 0)    { print "zero"; rfalse; }
    if (n < 0)     { print "minus "; n = -n; }
    if (n >= 1000) { print (LanguageNumber) n/1000, " thousand";
                     n = n%1000; f = 1; }
    if (n >= 100)  {
        if (f == 1) print ", ";
        print (LanguageNumber) n/100, " hundred"; n = n%100; f = 1;
    }
    if (n == 0) rfalse;
    #Ifdef DIALECT_US;
    if (f == 1) print " ";
    #Ifnot;
    if (f == 1) print " and ";
    #Endif;
    switch (n) {
      1:    print "one";
      ...
      20 to 99: switch (n/10) {
        2:  print "twenty";
        ...
        }
        if (n%10 ~= 0) print "-", (LanguageNumber) n%10;
    }
];
```

Note the `DIALECT_US` conditional: British English uses "and" between
hundreds and tens ("one hundred and twelve"), while American English omits
it ("one hundred twelve"). Define `Constant DIALECT_US;` before including
the library to select American conventions.

### 33.1.5 Time of Day

The `LanguageTimeOfDay` function (line 273) prints a time value in 12-hour
AM/PM format:

```inform6
[ LanguageTimeOfDay hours mins i;
    i = hours%12;
    if (i == 0) i = 12;
    if (i < 10) print " ";
    print i, ":", mins/10, mins%10;
    if ((hours/12) > 0) print " pm"; else print " am";
];
```

This is called by the status line printing code when the game uses
`Statusline time;`.

### 33.1.6 Verb Printing and Debugging

The `LanguageVerb` function (line 281) maps abbreviated verb dictionary
words to their full English names for use in parser messages:

```inform6
[ LanguageVerb i;
    switch (i) {
      'i//','inv','inventory':
               print "take inventory";
      'l//':   print "look";
      'x//':   print "examine";
      'z//':   print "wait";
      'n//':   print "north";
      's//':   print "south";
      'e//':   print "east";
      'w//':   print "west";
      'ne//':  print "northeast";
      'nw//':  print "northwest";
      'se//':  print "southeast";
      'sw//':  print "southwest";
      default: rfalse;
    }
    rtrue;
];
```

The `LanguageVerbIsDebugging` function (line 308) identifies debugging
verbs so the parser can give them full scope. It is wrapped in
`#Ifdef DEBUG` so it only exists in debug builds:

```inform6
#Ifdef DEBUG;
[ LanguageVerbIsDebugging w;
    if (w == 'purloin' or 'tree' or 'abstract'
                       or 'gonear' or 'scope' or 'showobj')
        rtrue;
    rfalse;
];
#Endif;
```

Two additional functions in `english.h` assist the parser with verb
handling:

- `LanguageVerbLikesAdverb(w)` (line 320): Returns true if the verb word
  `w` is intransitive and takes a direction adverb rather than a noun
  object (e.g., "go north" rather than "take lamp").
- `LanguageVerbMayBeName(w)` (line 336): Returns true if the verb word `w`
  could also be an object name (e.g., "long" in "long sword" vs. "long"
  as a `VERBOSE` synonym).

### 33.1.7 Text Constants

Lines 346–407 define string constants used throughout the library for user
interface text. These are separated into several groups:

**Menu navigation keys** (lines 346–357):

```inform6
Constant NKEY__TX       = "N = next subject";
Constant PKEY__TX       = "P = previous";
Constant QKEY1__TX      = "  Q = resume game";
Constant QKEY2__TX      = "Q = previous menu";
Constant RKEY__TX       = "RETURN = read subject";

Constant NKEY1__KY      = 'N';
Constant NKEY2__KY      = 'n';
...
```

**Status line labels** (lines 359–364):

```inform6
Constant SCORE__TX      = "Score: ";
Constant MOVES__TX      = "Moves: ";
Constant TIME__TX       = "Time: ";
Constant SCORE_S__TX    = "S: ";    ! Short forms for narrow displays
Constant MOVES_S__TX    = "M: ";
Constant TIME_S__TX     = "T: ";
```

**Core prose fragments** (lines 365–391):

```inform6
Constant CANTGO__TX     = "You can't go that way.";
Constant FORMER__TX     = "your former self";
Constant YOURSELF__TX   = "yourself";
Constant MYSELF__TX     = "myself";
Constant YOU__TX        = "You";
Constant DARKNESS__TX   = "Darkness";
Constant THOSET__TX     = "those things";
Constant THAT__TX       = "that";
Constant THE__TX        = "the";
Constant OR__TX         = " or ";
Constant NOTHING__TX    = "nothing";
Constant IS__TX         = " is";
Constant ARE__TX        = " are";
Constant WAS__TX        = " was";
Constant WERE__TX       = " were";
Constant AND__TX        = " and ";
Constant WHOM__TX       = "whom ";
Constant WHICH__TX      = "which ";
```

**Version and error reporting** (lines 397–407):

```inform6
Constant LIBERROR__TX   = "Library error ";
Constant TERP__TX       = "Interpreter ";
Constant VER__TX        = "Version ";
Constant RELEASE__TX    = "Release ";
```

A language definition file must define all of these constants with
appropriate translations, as the library references them by name.

### 33.1.8 The LanguageLM Function

The `LanguageLM` function (line 797) is the largest component of
`english.h`. It handles all library messages — the text printed in
response to player actions and parser errors. It receives three arguments:

- `n` — the message number, typically a `switch` label corresponding to an
  action name and sub-message index
- `x1` — the primary object involved (e.g., the noun)
- `x2` — the secondary object (e.g., the second noun), or other context

The function is structured as a large `switch` on action names, with
nested `switch` statements on `n` for actions that produce multiple
messages:

```inform6
[ LanguageLM n x1 x2;
  Answer,Ask:
            print "There ";
            Tense("is", "was");
            " no reply.";
  Attack:   print "Violence ";
            Tense("isn't", "wasn't");
            " the answer to this one.";
  ...
  Close: switch (n) {
        1:  CSubjectIs(x1,true);
            print " not something ", (theActor) actor;
            Tense(" can close", " could have closed");
            ".";
        2:  CSubjectIs(x1,true); " already closed.";
        3:  CSubjectVerb(actor,false,false,"close",0,"closes","closed");
            " ", (the) x1, ".";
    }
  ...
];
```

`LanguageLM` handles over 70 action types with approximately 200
individual messages. It makes extensive use of the conjugation helper
functions described in §33.5 to support narrative voice and tense
variations. Every message uses `Tense()`, `CSubjectVerb()`, and related
helpers rather than hard-coded strings, allowing all messages to adapt to
first-person, second-person, or third-person narration in either present
or past tense.

The library calls `LanguageLM` through the `L__M` wrapper function (see
Chapter 32), which first checks for a `LibraryMessages` object that the
game can use to intercept and replace individual messages.

## 33.2 Writing a New Language Definition File

To translate the Inform library into a new natural language, one creates a
replacement for `english.h` that implements the same interface. The parser
(`parser.h`) and verb library (`verblib.h`) call Language\* functions and
reference Language\* arrays and constants by name. This section documents
the complete interface contract.

### 33.2.1 Required Arrays

A language definition file must define the following arrays:

**`LanguagePronouns`** — A `table` array where each entry consists of a
dictionary word, a 12-bit GNA pattern, and a connected-object value. The
parser uses this to resolve pronouns in player input.

**`LanguageDescriptors`** — A `table` array where each entry consists of a
dictionary word, a 12-bit GNA pattern, a descriptor type constant
(`POSSESS_PK`, `DEFART_PK`, or `INDEFART_PK`), and a connected-object
value. The parser uses this to handle articles, possessives, and other
pre-noun descriptors.

**`LanguageNumbers`** — A `table` array of alternating dictionary words
and integer values. The parser uses this to convert number words in the
player's input to numeric values.

**`LanguageArticles`** — A `-->` (word) array containing article strings.
The array has dimensions `(number of article rows) × (LanguageContractionForms × 3)`.
Each group of three strings per contraction form contains the capitalised
definite article, lowercase definite article, and indefinite article.

**`LanguageGNAsToArticles`** — A `-->` (word) array of 12 entries, one per
GNA value, mapping each GNA to an article row index in `LanguageArticles`.

### 33.2.2 Required Constants

**`LanguageAnimateGender`** — The default attribute to assume for animate
objects without explicit gender attributes. Must be `male`, `female`, or
`neuter`.

**`LanguageInanimateGender`** — The default attribute to assume for
inanimate objects. Typically `neuter`.

**`LanguageContractionForms`** — An integer specifying how many contraction
forms the language uses. English uses 2 (consonant vs. vowel). A language
with no article contraction uses 1. A language with three forms (e.g.,
based on the following consonant type) uses 3.

### 33.2.3 Required Functions

The following functions are called unconditionally by the library and must
be defined:

**`LanguageContraction(text)`** — Receives the printed text of an object's
short name (as a byte array). Returns an integer from 0 to
`LanguageContractionForms - 1` identifying which contraction form to use.
The library uses this return value to select the correct column in
`LanguageArticles`.

**`LanguageDirection(d)`** — Receives a direction property value (e.g.,
`n_to`, `s_to`) and prints the corresponding direction name in the target
language.

**`LanguageNumber(n)`** — Receives an integer and prints it as words in
the target language. Called by the library when printing numbers in prose
(e.g., "There are three coins here").

**`LanguageTimeOfDay(hours, mins)`** — Receives hours (0–23) and minutes
(0–59) and prints a formatted time string. Called by the status line code
when the game uses `Statusline time;`.

**`LanguageVerb(i)`** — Receives a dictionary word and prints its
full-form verb name. Returns true if the word was recognised, false
otherwise. Used in parser messages like "I only understood you as far as
wanting to [verb]".

**`LanguageVerbIsDebugging(w)`** — Receives a dictionary word and returns
true if it is a debugging verb that needs all objects in scope. Wrap the
definition in `#Ifdef DEBUG; ... #Endif;`.

**`LanguageLM(n, x1, x2)`** — The main library message handler. Must
handle all action messages and parser error messages. This is the largest
and most complex function in the language definition file.

### 33.2.4 Optional Functions

The following functions are checked with `#Ifdef` before being called. If
they are absent, the library uses default behaviour:

**`LanguageInitialise`** — Called once at game startup during
`Initialise()` (parser.h line 5364). Can be used to set up
language-specific global state.

**`LanguageToInformese`** — Called during input tokenisation (parser.h
line 1600) to preprocess the raw input buffer before the parser analyses
it. Useful for normalising accented characters, expanding contractions, or
handling language-specific input transformations. The `english.h` file
defines this as an empty function.

**`LanguageIsVerb(w)`** — Called during verb identification (parser.h
lines 1751, 3229, 3398). If defined, gives the language file a chance to
identify a word as a verb before the standard grammar tables are checked.

**`LanguageRefers(obj, word)`** — Called during pronoun reference
resolution (parser.h line 4605). If defined, allows the language file to
provide custom logic for determining whether a given word can refer to a
given object.

**`LanguageCommand(v)`** — Called when printing a command back to the
player (parser.h line 4011). If defined, gives the language file a chance
to customise how commands are displayed.

**`LanguagePrintShortName(obj)`** — Called when printing an object's short
name (parser.h line 7067). If defined, allows the language file to
override the default short-name printing. Return true to suppress the
default print, false to let it proceed.

**`LanguageBanner`** — Called during the game banner display (verblib.h
line 62). If defined, replaces the default banner printing.

**`LanguageVersion`** — Called during the `VERSION` verb output (verblib.h
line 145). If defined, prints additional version information about the
language definition file.

**`LanguageVersionSub`** — Called during the `VERSION` verb handler
(verblib.h line 110). If defined, replaces the entire version display.

**`LanguageError`** — Called on library errors (verblib.h line 152). If
defined, allows the language file to provide custom error formatting.

### 33.2.5 The GNA System

The Gender/Number/Animacy (GNA) system encodes grammatical properties of
objects and words using a 12-bit pattern. The 12 positions correspond to:

| GNA | Animacy   | Number   | Gender | Index |
|-----|-----------|----------|--------|-------|
| 0   | Animate   | Singular | Male   | 0     |
| 1   | Animate   | Singular | Female | 1     |
| 2   | Animate   | Singular | Neuter | 2     |
| 3   | Animate   | Plural   | Male   | 3     |
| 4   | Animate   | Plural   | Female | 4     |
| 5   | Animate   | Plural   | Neuter | 5     |
| 6   | Inanimate | Singular | Male   | 6     |
| 7   | Inanimate | Singular | Female | 7     |
| 8   | Inanimate | Singular | Neuter | 8     |
| 9   | Inanimate | Plural   | Male   | 9     |
| 10  | Inanimate | Plural   | Female | 10    |
| 11  | Inanimate | Plural   | Neuter | 11    |

A GNA pattern like `$$001000111000` has bit 2 (animate singular neuter),
bits 6–8 (inanimate singular all genders) set. This matches the pronoun
"it", which can refer to a neuter animate creature or any singular
inanimate object.

The parser computes an object's GNA from its attributes: `animate` sets
the animate flag; `male`, `female`, and `neuter` set gender; `pluralname`
sets plural. It then compares the object's GNA against the pronoun or
descriptor pattern to determine compatibility.

### 33.2.6 The Article/Contraction System

The article selection mechanism works as follows:

1. The library determines the object's GNA value (0–11).
2. It looks up `LanguageGNAsToArticles-->gna` to get the article row.
3. It calls `LanguageContraction(obj.short_name_text)` to get a
   contraction form (0 to `LanguageContractionForms - 1`).
4. It computes the index into `LanguageArticles`:
   `row * LanguageContractionForms * 3 + contraction_form * 3 + column`,
   where column is 0 for capitalised definite, 1 for lowercase definite,
   or 2 for indefinite.
5. It prints the string at that index.

For a language with three genders and two numbers but no contraction
(French, for example), one would set `LanguageContractionForms = 1` and
provide article rows for each gender/number combination (le/la/les/un/une/
des), with `LanguageGNAsToArticles` mapping GNA values to the correct
rows.

### 33.2.7 Required Text Constants

The language definition file must define all of the text constants listed
in §33.1.7. The library references these constants by name throughout the
parser and verb library code. The most critical ones are:

- `YOU__TX`, `YOURSELF__TX`, `MYSELF__TX` — player reference text
- `DARKNESS__TX` — the name shown in the status line in dark rooms
- `CANTGO__TX` — the message printed for blocked exits
- `IS__TX`, `ARE__TX`, `WAS__TX`, `WERE__TX` — verb forms
- `THE__TX`, `THAT__TX`, `OR__TX`, `AND__TX` — articles and conjunctions
- `SCORE__TX`, `MOVES__TX`, `TIME__TX` — status line labels

## 33.3 Character Sets: The Zcharacter Directive

**[Z-machine]** The `Zcharacter` directive controls the character encoding
tables used by the Z-machine. It is only available when compiling for the
Z-machine; the compiler produces an error if it is used in Glulx mode.

The Z-machine encodes text using three alphabets:

- **A0** (lowercase): 26 letters, initially `a`–`z`
- **A1** (uppercase): 26 letters, initially `A`–`Z`
- **A2** (punctuation/special): 23 usable positions (positions 0–2 are
  reserved for shift characters), initially containing digits and common
  punctuation

Characters in A0 are the most compact (one Z-character each). Characters
in A1 or A2 require a shift character prefix (two Z-characters). Any
character outside these alphabets must be encoded as a four-Z-character
ZSCII escape sequence, which is expensive.

### 33.3.1 Directive Forms

The `Zcharacter` directive (compiler source: `directs.c` lines 1164–1254)
has four forms:

**Form 1: Replace all three alphabets**

```inform6
Zcharacter "abcdefghijklmnopqrstuvwxyz"
           "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
           "0123456789!@#$%^&*()_+-=";
```

Each string replaces one alphabet: A0 (26 characters), A1 (26 characters),
A2 (23 characters, since positions 0–2 are reserved). The compiler calls
`new_alphabet()` (`chars.c` line 233) for each string, which replaces the
entire alphabet array for that position.

**Form 2: Add a single character to alphabet A2**

```inform6
Zcharacter '@{00E9}';
```

This calls `map_new_zchar()` (`chars.c` line 191), which attempts to place
the given Unicode character into an unused position in alphabet A2. The
function scans A2 positions left to right, skipping positions 12, 13, and
19 (which correspond to `.`, `"`, and `~` — characters considered
essential). It replaces the first position whose character has not yet been
used in any compiled text. If no positions are available, the operation
fails silently.

**Form 3: Define a custom ZSCII extension table**

```inform6
Zcharacter table
    $00E9   ! é (e-acute)
    $00E8   ! è (e-grave)
    $00EA   ! ê (e-circumflex)
    $00EB   ! ë (e-diaeresis)
    ;
```

This replaces the default ZSCII extension table entirely. Each number is a
Unicode code point to be mapped into the ZSCII range 155–251. The compiler
calls `new_zscii_character()` (`chars.c` line 1034) for each value. The
first call (without the `+` flag) resets `zscii_high_water_mark` to 0,
clearing any previous mappings. Up to 97 custom characters can be defined
(ZSCII codes 155 through 251).

**Form 4: Extend the existing ZSCII table**

```inform6
Zcharacter table + $0100 $0101 $010C $010D;
```

The `+` sign preserves existing entries and appends new ones. The
`plus_flag` is passed to `new_zscii_character()`, which skips the
`zscii_high_water_mark` reset.

### 33.3.2 Practical Example

A game using French accented characters might include:

```inform6
! Add frequently-used accented characters to alphabet A2
! so they encode compactly (2 Z-characters instead of 4)
Zcharacter '@{00E9}';  ! é
Zcharacter '@{00E8}';  ! è
Zcharacter '@{00EA}';  ! ê
Zcharacter '@{00E0}';  ! à
Zcharacter '@{00E7}';  ! ç
Zcharacter '@{00EE}';  ! î
Zcharacter '@{00F4}';  ! ô
Zcharacter '@{00FB}';  ! û

! Define ZSCII table for all accented characters needed
Zcharacter table
    $00E9 $00E8 $00EA $00EB   ! é è ê ë
    $00E0 $00E2               ! à â
    $00E7                     ! ç
    $00EE $00EF               ! î ï
    $00F4                     ! ô
    $00F9 $00FB               ! ù û
    ;
```

Characters placed in alphabet A2 via `Zcharacter 'x'` encode in two
Z-characters instead of four, so frequently-used accented letters should
be added to A2 when possible. Characters that only need to be
representable (but not necessarily compact) should be in the ZSCII
extension table.

### 33.3.3 Limitations

- The Z-machine can support at most 97 extra characters (ZSCII 155–251)
  beyond the base Latin-1 set.
- Unicode characters beyond U+FFFF cannot be placed in the ZSCII table
  (`chars.c` line 1036 checks `u > 0xFFFF`).
- Alphabet A2 has only about 12 replaceable positions (digits and some
  punctuation), since positions 0–2 are reserved and several punctuation
  characters are protected.
- The `Zcharacter` directive is rejected with an error in Glulx mode
  (`directs.c` line 1172).

## 33.4 Unicode Support in Glulx

**[Glulx]** Glulx provides full Unicode support without the character-set
limitations of the Z-machine. There is no need for `Zcharacter` directives
or custom alphabet tables; any Unicode character can be used directly.

### 33.4.1 Unicode in Strings

The `@{NNNN}` escape notation allows arbitrary Unicode code points in
strings and character constants:

```inform6
! Unicode characters can be used directly in Glulx strings
print "Caf@{00E9} @{2014} a nice caf@{00E9}.^";
! Prints: Café — a nice café.

! Em dash (U+2014) and curly quotes
print "@{201C}Hello,@{201D} she said.^";
! Prints: "Hello," she said.
```

The `@{...}` syntax accepts 1 to 6 hexadecimal digits specifying a
Unicode code point. The compiler converts these to the target encoding
during compilation.

### 33.4.2 Character Mapping

The compiler maintains a chain of character mappings between different
encoding systems:

- **Source text** is read as ISO 8859-1 by default, or as UTF-8 if the
  `character_set_unicode` flag is set (via the `-Cu` compiler switch).
- **ISO 8859-1 ↔ Unicode**: The `iso_to_unicode_grid` array (`chars.c`
  line 81) maps ISO 8859-1 byte values to Unicode code points.
- **Unicode ↔ ZSCII**: `unicode_to_zscii()` (`chars.c` line 1049) maps
  Unicode to ZSCII codes; `zscii_to_unicode()` (`chars.c` line 1057)
  performs the reverse. For Glulx, these are used during compilation but
  the runtime operates on Unicode directly.

The conversion functions:

```c
/* chars.c line 1049 */
extern int unicode_to_zscii(int32 u)
{   int i;
    if (u < 0x7f) return u;
    for (i=0; i<zscii_high_water_mark; i++)
        if (zscii_to_unicode_grid[i] == u) return i+155;
    return 5;
}

/* chars.c line 1057 */
extern int32 zscii_to_unicode(int z)
{   if (z < 0x80) return z;
    if ((z >= 155) && (z <= 251))
        return zscii_to_unicode_grid[z-155];
    return '?';
}
```

### 33.4.3 Huffman Compression and Unicode

Glulx uses Huffman compression for strings. The compiler tracks distinct
Unicode characters beyond U+00FF with the `no_unicode_chars` counter
(`text.c` line 48). The `huff_unicode_start` variable (`text.c` line 57)
records the position in the Huffman entity table where Unicode character
entries begin, allowing the decompression code to distinguish Unicode
entries from ASCII entries and abbreviation references.

When a Unicode character beyond the Latin-1 range appears in a string, the
compiler:

1. Records it in the Unicode character table.
2. Assigns it a Huffman entity for compression.
3. In the output file, encodes it as a multi-byte Huffman sequence.

At runtime, the Glulx interpreter's string-decoding code handles these
transparently, printing the correct Unicode character to the output stream.

### 33.4.4 UTF-8 Source Files

The compiler's `-Cu` switch sets the `character_set_unicode` flag, enabling
UTF-8 decoding of source files. When active, the `text_to_unicode()`
function (`chars.c` line 1085) performs UTF-8 multi-byte decoding when it
encounters high-bit characters (bytes with bit 7 set). This allows source
files to contain accented characters and non-Latin scripts directly,
without `@{...}` escapes:

```inform6
! With -Cu, UTF-8 source files can contain characters directly
! (shown here with escape notation for portability)
print "Hélas, c'est la vie.^";
```

### 33.4.5 Glulx Unicode Opcodes

Glulx provides two I/O opcodes for Unicode support:

- **`@streamunichar` (opcode 0x73)** — Outputs a single Unicode character
  to the current output stream. The compiler uses this for printing
  individual Unicode characters that fall outside the Latin-1 range.

- **`@"1CC"` (`gestalt_Unicode`)** — Tests whether the interpreter
  supports Unicode I/O. Games that need to verify Unicode support at
  runtime can check this gestalt value.

For further details on Glulx I/O and string handling, see Chapter 29.

## 33.5 Narrative Voice and Tense Control

Library 6.12 introduced a system for varying the narrative voice and tense
of library messages. By default, the library uses second-person present
tense ("You take the lamp"), but it can be configured for first-person
("I take the lamp"), third-person ("George takes the lamp"), and past
tense ("You took the lamp") narration. This system is documented in
`voices_and_tenses.txt` in the library distribution.

### 33.5.1 Configuration Properties and Constants

Two constants are defined in `parser.h` (lines 799–800):

```inform6
Constant PRESENT_TENSE 0;
Constant PAST_TENSE    1;
```

The `SelfClass` class (parser.h lines 821–842), from which the default
`selfobj` player object is derived, defines two properties that control
narration:

```inform6
Class  SelfClass
  with ...
        narrative_voice 2,
        narrative_tense PRESENT_TENSE,
        ...
  has   concealed animate proper transparent;
```

- **`narrative_voice`** — Controls grammatical person:
  - `1` = first person ("I")
  - `2` = second person ("You") — the default
  - `3` = third person ("He/She/George")
- **`narrative_tense`** — Controls grammatical tense:
  - `PRESENT_TENSE` (0) — the default
  - `PAST_TENSE` (1)

These properties belong to the player object. Changing them at runtime
immediately affects all subsequent library messages.

### 33.5.2 Switching Voice at Runtime

To switch the narrative voice, set the properties on the player object:

```inform6
! Switch to third-person past tense
player.narrative_voice = 3;
player.short_name = "George";
player.narrative_tense = PAST_TENSE;
```

When `narrative_voice` is 3, the library uses the player object's
`short_name` property as the character name. The object should have the
`proper` attribute to prevent the library from printing "the George".
Set `male` or `female` as appropriate for pronoun selection.

For body-switching (changing which object the player controls), the
`ChangePlayer` routine transfers these properties to the new player
object. Setting up a flashback in past tense is straightforward:

```inform6
[ BeginFlashback;
    player.narrative_tense = PAST_TENSE;
    print "^[You remember...]^";
];

[ EndFlashback;
    player.narrative_tense = PRESENT_TENSE;
    print "^[The memory fades.]^";
];
```

### 33.5.3 The CSubjectVerb Function

The core conjugation function is `CSubjectVerb` (`english.h` line 554).
All library messages that print a subject-verb combination call this
function or one of its specialised wrappers:

```inform6
[ CSubjectVerb obj reportage nocaps v1 v2 v3 past;
    if (past && player provides narrative_tense
             && player.narrative_tense == PAST_TENSE) {
        v1 = past;
        v2 = past;
        v3 = past;
    } else {
        if (v2 == 0) v2 = v1;
        if (v3 == 0) v3 = v1;
    }
    if (obj == player) {
        if (player provides narrative_voice)
          switch (player.narrative_voice) {
          1:  print "I ", (string) v1; return;
          2:  ! Do nothing (fall through to default).
          3:  CDefart(player);
              print " ", (string) v3; return;
          default: RunTimeError(16, player.narrative_voice);
        }
        if (nocaps) { print "you ", (string) v2; return; }
        print "You ", (string) v2; return;
    }
    SubjectNotPlayer(obj, reportage, nocaps, v2, v3);
];
```

The parameters are:

- **`obj`** — The subject of the verb, usually `actor`.
- **`reportage`** — If true and `actor` is not the player, the function
  prepends "George observes that" (via `L__M(##Miscellany, 60, actor)`).
- **`nocaps`** — If true, prints "you" instead of "You" (for mid-sentence
  use).
- **`v1`** — First-person present tense form (e.g., `"take"`).
- **`v2`** — Second-person present tense form. Pass 0 to use `v1`.
- **`v3`** — Third-person present tense form (e.g., `"takes"`).
- **`past`** — Past tense form (e.g., `"took"`). Used for all persons when
  `narrative_tense` is `PAST_TENSE`.

Example usage showing output for each voice/tense combination:

```inform6
CSubjectVerb(actor, false, false, "close", 0, "closes", "closed");
print " ", (the) x1, ".^";
! Voice 1, present: "I close the door."
! Voice 2, present: "You close the door."
! Voice 3, present: "George closes the door."
! Voice 1, past:    "I closed the door."
! Voice 2, past:    "You closed the door."
! Voice 3, past:    "George closed the door."
```

### 33.5.4 Specialised Conjugation Helpers

`english.h` provides wrapper functions for common English verbs. Each
follows the same pattern as `CSubjectVerb` but with the verb forms
built in:

**`CSubjectIs(obj, reportage, nocaps)`** (line 577):
Prints the subject with the appropriate form of "to be":

| Voice | Present       | Past          |
|-------|---------------|---------------|
| 1     | "I'm"         | "I was"       |
| 2     | "You're"      | "You were"    |
| 3     | "George is"   | "George was"  |

**`CSubjectIsnt(obj, reportage, nocaps)`** (line 593):
Negative form of "to be":

| Voice | Present          | Past             |
|-------|------------------|------------------|
| 1     | "I'm not"        | "I wasn't"       |
| 2     | "You aren't"     | "You weren't"    |
| 3     | "George isn't"   | "George wasn't"  |

**`CSubjectHas(obj, reportage, nocaps)`** (line 609):

| Voice | Present       | Past          |
|-------|---------------|---------------|
| 1     | "I've"        | "I had"       |
| 2     | "You've"      | "You'd"       |
| 3     | "George has"  | "George had"  |

**`CSubjectWill(obj, reportage, nocaps)`** (line 625):

| Voice | Present        | Past            |
|-------|----------------|-----------------|
| 1     | "I'll"         | "I would"       |
| 2     | "You'll"       | "You'd"         |
| 3     | "George will"  | "George would"  |

**`CSubjectCan(obj, reportage, nocaps)`** (line 641):
Delegates to `CSubjectVerb` with `"can"` / `"could"`.

**`CSubjectCant(obj, reportage, nocaps)`** (line 645):
Delegates to `CSubjectVerb` with `"can't"` / `"couldn't"`.

**`CSubjectDont(obj, reportage, nocaps)`** (line 649):
Delegates to `CSubjectVerb` with `"don't"` / `"doesn't"` / `"didn't"`.

### 33.5.5 The Tense Helper

The `Tense` function (`english.h` line 746) is a low-level helper that
prints one of two strings depending on the current tense:

```inform6
[ Tense present past;
    if (player provides narrative_tense
        && player.narrative_tense == PAST_TENSE) {
        if (past == false) return;
        print (string) past;
    }
    else
        print (string) present;
];
```

If `past` is 0 (false), the function prints nothing in past tense. This
is useful for verb suffixes that exist in present tense but not in past
tense. Library messages use `Tense` extensively:

```inform6
! From LanguageLM, the Answer/Ask action:
print "There ";
Tense("is", "was");
" no reply.";
! Present: "There is no reply."
! Past:    "There was no reply."
```

### 33.5.6 Pronoun and Demonstrative Functions

Several functions handle pronoun selection based on narrative voice,
gender, and number:

**`CThatOrThose(obj)`** (line 451) — Prints the nominative subject
pronoun. For the player object, respects `narrative_voice`:

| Voice | Output       |
|-------|--------------|
| 1     | "I"          |
| 2     | "You"        |
| 3     | (player name)|

For non-player objects, prints "He", "She", "Those", or "That" based on
attributes.

**`CTheyreOrThats(obj)`** (line 469) — Prints the subject with a
contracted form of "to be", aware of both voice and tense:

| Voice | Present      | Past           |
|-------|--------------|----------------|
| 1     | "I'm"        | "I was"        |
| 2     | "You're"     | "You were"     |
| 3     | "George's"   | "George was"   |

**`ThatOrThose(obj)`** (line 417) — Accusative (object) pronoun. For
the player: "me" (voice 1), "you" (voice 2), or the definite article
name (voice 3). For non-player objects: "him", "her", "those", or "that".

**`ItOrThem(obj)`** (line 434) — Accusative reflexive pronoun. For the
player: "myself" (voice 1), "yourself" (voice 2), or the definite
article name (voice 3). For non-player objects: "him", "her", "them", or
"it".

**`IsOrAre(obj)`** (line 486) — Prints "is" or "are" (present) / "was"
or "were" (past) based on whether `obj` has `pluralname` or is the player.

**`OnesSelf(obj)`** (line 654) — Reflexive pronoun: "myself" (voice 1),
"yourself" (voice 2), "himself"/"herself" (voice 3), or appropriate
third-person form for non-player objects.

### 33.5.7 Actor Reference Functions

**`theActor(obj)`** (line 705) — Prints a lowercase nominative pronoun
for the subject. For the player object:

| Voice | Output |
|-------|--------|
| 1     | "I"    |
| 2     | "you"  |
| 3     | "he"/"she"/"it" (based on attributes) |

For non-player objects: "he", "she", "they", or "that".

**`Possessive(obj, caps)`** (line 671) — Prints the possessive pronoun
for `obj`, with optional capitalisation:

| Voice | Output (caps=false) | Output (caps=true) |
|-------|---------------------|--------------------|
| 1     | "my"                | "My"               |
| 2     | "your"              | "Your"             |
| 3     | "George's"          | "George's"         |

For non-player objects: "his", "her", "their", or "its".

**`PossessiveCaps(obj)`** (line 701) — Convenience wrapper that calls
`Possessive(obj, true)`.

### 33.5.8 Additional Helper Functions

**`SubjectNotPlayer(obj, reportage, nocaps, v2, v3, past)`** (line 497)
— Internal helper used by `CSubjectVerb` and its wrappers when `obj` is
not the player. Handles reportage mode, plural/singular verb agreement,
and tense selection for third-party NPCs.

**`CSubjectVoice(obj, v1, v2, v3, past)`** (line 533) — Prints only the
verb form (without the subject name) selected by narrative voice and
tense. Useful when the subject has already been printed separately.

**`SupportObj(obj, s1, s2)`** (line 726) — Prints `s1` if `obj` has the
`supporter` attribute, `s2` otherwise. Used in messages about
containers vs. supporters.

**`PluralObj(obj, s1, s2, past)`** (line 731) — Prints `s1` if `obj` has
`pluralname`, `s2` otherwise, but overrides with `past` if narrative
tense is past. Used for singular/plural verb agreement in action
messages.

**`DecideAgainst()`** (line 755) — Prints a complete sentence indicating
the actor decided against an action, using `CSubjectVerb` to handle
voice and tense:

```inform6
[ DecideAgainst;
    CSubjectVerb(actor, false, false, "decide",0,"decides","decided");
    print " that";
    Tense("'s not", " wasn't");
    " such a good idea.";
];
```

### 33.5.9 Writing Voice-Aware Library Messages

When writing custom library messages (via `LibraryMessages` or a
replacement `LanguageLM`), always use the conjugation helpers rather than
hard-coded strings. This ensures messages adapt correctly to voice and
tense changes:

```inform6
! WRONG: hard-coded second-person present
Eat: "You eat ", (the) x1, ". Delicious!";

! RIGHT: voice and tense aware
Eat:
    CSubjectVerb(actor, false, false, "eat", 0, "eats", "ate");
    print " ", (the) x1, ". Delicious!";
```

For messages that include auxiliary verbs, use the specialised helpers:

```inform6
! "You can't eat that." / "I couldn't eat that." / etc.
CSubjectCant(actor, true);
" eat ", (ThatOrThose) x1, ".";
```

For conditional text that varies by tense alone, use `Tense`:

```inform6
print "Nothing ";
Tense("happens", "happened");
".";
```

For further information on replacing and intercepting library messages,
see Chapter 32.
