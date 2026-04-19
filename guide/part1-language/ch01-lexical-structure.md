# Chapter 1: Lexical Structure

This chapter describes the lexical structure of the language: how source text
is broken into the tokens that form the input to the parser. Understanding
the lexical level is essential for writing correct programs and for
interpreting compiler error messages that refer to unexpected tokens or
illegal characters.

## 1.1 Source File Encoding and Character Sets

### 1.1.1 Source Encoding

Source files are streams of 8-bit bytes. The compiler interprets
these bytes according to a **character set setting** that can be selected
with the `-C` compiler switch:

| Setting | Encoding |
| ------- | -------- |
| 0 | Plain ASCII (0x20–0x7E) |
| 1 (default) | ISO 8859-1 (Latin-1) |
| 2 | ISO 8859-2 (Latin-2) |
| 3 | ISO 8859-3 (Latin-3) |
| 4 | ISO 8859-4 (Latin-4) |
| 5 | ISO 8859-5 (Cyrillic) |
| 6 | ISO 8859-6 (Arabic) |
| 7 | ISO 8859-7 (Greek) |
| 8 | ISO 8859-8 (Hebrew) |
| 9 | ISO 8859-9 (Latin-5) |

When a non-zero character set is selected, bytes in the range 0xA1–0xFF are
treated as valid characters and are mapped through the `source_to_iso_grid`
conversion table. Regardless of setting, bytes in the ASCII range
0x20–0x7E always have their standard ASCII meanings. The compiler also
supports reading UTF-8 encoded source when Unicode mode is enabled with
the `-Cu` switch.

The compiler internally uses six character representations:

1. **ASCII** — the 7-bit subset (0x20–0x7E) common to all sets.
2. **Source** — raw bytes from the source file.
3. **ISO** — bytes interpreted according to the selected ISO 8859 character
   set.
4. **ZSCII** — the Z-machine's 10-bit character set (values 0–1023).
5. **Textual** — escape notation such as `@'e` for e-acute or `@{03a3}`
   for Unicode U+03A3.
6. **Unicode** — a unifying character set holding all representable
   characters (16-bit values).

Conversion can always proceed downward through this list but generally not
upward.

There is a seventh form: sequences of 5-bit "Z-chars" which encode ZSCII
into the story file in compressed form. Conversion of ZSCII to and from
Z-char sequences is handled separately.

### 1.1.2 ZSCII (Z-Machine Character Set)

When compiling for the **Z-machine**, characters in strings and dictionary
words are encoded in ZSCII, a 10-bit character set defined by the
*Z-Machine Standards Document*. ZSCII values 32–126 correspond to standard
ASCII. Values 155–251 encode accented and special characters through a
configurable mapping.

The compiler maintains an **alphabet table** of three 26-letter alphabets
(A0, A1, A2) used to compress ZSCII into 5-bit Z-characters:

- **Alphabet 0 (A0):** lowercase letters and common characters.
- **Alphabet 1 (A1):** uppercase letters and additional characters.
- **Alphabet 2 (A2):** digits, punctuation, and special characters.

Two positions in A2 have special meaning: position 1 encodes newline
(which is why `^` in strings produces a newline), and position 19 holds
the tilde, which Inform maps to the double-quote character (which is why
`~` in strings produces `"`).

### 1.1.3 Unicode Support (Glulx)

When compiling for **Glulx**, the full Unicode character set is available.
Strings are stored as sequences of Unicode code points, and dictionary
words can use either 1-byte or 4-byte characters depending on the
`DICT_CHAR_SIZE` setting (1 for Latin characters, 4 for full Unicode).

Unicode characters can be included in any string using the `@{hex}`
escape notation, regardless of target platform:

```inform6
print "Greek sigma: @{03A3}^";
print "Em dash: @{2014}^";
```

### 1.1.4 The `Zcharacter` Directive

The `Zcharacter` directive customizes the Z-machine's character encoding.
It is only meaningful when compiling for the Z-machine; in Glulx mode the
compiler issues an error. There are several forms:

**Redefine all three alphabets:**

```inform6
Zcharacter "abcdefghijklmnopqrstuvwxyz"
           "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
           "0123456789.,!?_#'/\-:()";
```

The A0 and A1 strings must each contain exactly 26 characters and replace
alphabets A0 and A1. The A2 string must contain exactly 23 characters and
fills positions 3–25 of A2. Positions 0 and 1 of A2 are fixed by the Z-machine
standard: position 0 is an escape code, and position 1 is a newline
(which is why `^` in strings produces a newline). Position 2 is fixed by
Inform to hold the tilde character (which is why `~` in strings is
encoded as the double-quote character).

**Map a single Unicode character into the ZSCII table:**

```inform6
Zcharacter '@{00E9}';
```

This attempts to place the given character (here, e-acute U+00E9) into an
unused position in the alphabet table for efficient encoding.

**Define a ZSCII-to-Unicode mapping table:**

```inform6
Zcharacter table
    '@{00E0}' '@{00E1}' '@{00E2}' '@{00E3}' '@{00E4}';
```

This replaces the default ZSCII-to-Unicode mapping for characters 155
onward. Each entry can be a single-quoted character or a numeric Unicode
code point. Use `table +` to extend the existing table rather than
replacing it:

```inform6
Zcharacter table + '@{0100}' '@{0101}';
```

**Define terminating characters:**

```inform6
Zcharacter terminating 255;
```

This declares additional ZSCII codes (beyond the default 13 for Enter)
that terminate keyboard input. The value 255 is commonly used to mean
"any function key." You can specify up to 32 terminating characters.

### 1.1.5 Line Endings

The compiler accepts all common line-ending conventions:

- Unix: LF (`\n`, 0x0A)
- Windows: CR+LF (`\r\n`, 0x0D 0x0A)
- Classic Mac: CR (`\r`, 0x0D)

Form feed (0x0C) is converted to a line break. All line-ending styles are
normalized internally; the compiler counts lines for error reporting
regardless of the convention used.

## 1.2 Comments

Inform 6 has only **single-line comments**, introduced by the exclamation
mark (`!`). Everything from `!` to the end of the line is discarded by
the lexer:

```inform6
! This entire line is a comment.
x = 42;  ! This comment follows a statement.
```

There is no block comment syntax. To comment out multiple lines, each line
must begin with `!` (or the lines can be removed using conditional
compilation directives such as `#Ifdef`/`#Endif`):

```inform6
! The following lines are all commented out:
! x = 1;
! y = 2;
! z = 3;
```

Comments inside quoted strings are not recognized — `!` is an ordinary
character in string context:

```inform6
print "This is not a comment! It prints.^";
```

## 1.3 Tokens and Tokenization

### 1.3.1 How Tokenization Works

The lexer reads source text one character at a time, with three
characters of lookahead (the current character plus three characters
ahead, in the variables `lookahead`, `lookahead2`, and `lookahead3`).
It classifies each character using a precomputed 256-entry grid
called the **tokenizer grid**, which maps each possible byte value to one
of the following codes:

| Grid Code | Meaning | Characters |
| --------- | ------- | ---------- |
| `WHITESPACE_CODE` | Whitespace (skipped) | space, `\n`, `\r` |
| `COMMENT_CODE` | Comment start | `!` |
| `DIGIT_CODE` | Decimal digit | `0`–`9` |
| `RADIX_CODE` | Radix prefix | `$` |
| `IDENTIFIER_CODE` | Identifier character | `a`–`z`, `A`–`Z`, `_` |
| `QUOTE_CODE` | Single-quote delimiter | `'` |
| `DQUOTE_CODE` | Double-quote delimiter | `"` |
| `EOF_CODE` | End of file | NUL (0x00) |
| *separator range* | Separator token | all punctuation and operators |

When the lexer encounters a character, it consults the grid and dispatches
to the appropriate handler: whitespace is consumed silently, comments skip
to end-of-line, digits start number parsing, and so on.

### 1.3.2 Token Types

Every token produced by the lexer has a **type** and a **value**. The
primary lexer token types are:

| Type | Numeric Code | Description |
| ---- | ------------ | ----------- |
| `SYMBOL_TT` | 0 | An identifier entered in the symbol table |
| `NUMBER_TT` | 1 | A numeric literal (decimal, hexadecimal, binary, or float) |
| `DQ_TT` | 2 | A double-quoted string literal |
| `SQ_TT` | 3 | A single-quoted literal (character or dictionary word) |
| `UQ_TT` | 4 | An unquoted identifier (when symbol-table entry is suppressed) |
| `SEP_TT` | 5 | A separator or operator |
| `EOF_TT` | 6 | End of file |

When an identifier matches a keyword in the current lexical context, it is
returned as one of the higher-numbered keyword token types instead of
`SYMBOL_TT`:

| Type | Code | Description |
| ---- | ---- | ----------- |
| `STATEMENT_TT` | 100 | Statement keyword (`if`, `while`, `for`, etc.) |
| `SEGMENT_MARKER_TT` | 101 | Object segment marker (`with`, `has`, `class`, `private`) |
| `DIRECTIVE_TT` | 102 | Directive keyword (`Array`, `Global`, `Object`, etc.) |
| `CND_TT` | 103 | Condition keyword (`has`, `hasnt`, `in`, `notin`, `ofclass`, `or`, `provides`) |
| `SYSFUN_TT` | 105 | System function (`parent`, `child`, `sibling`, `random`, etc.) |
| `LOCAL_VARIABLE_TT` | 106 | A local variable name |
| `OPCODE_NAME_TT` | 107 | An assembly opcode name |
| `MISC_KEYWORD_TT` | 108 | Miscellaneous keyword (`bold`, `roman`, `name`, etc.) |
| `DIR_KEYWORD_TT` | 109 | Directive keyword (`alias`, `long`, `additive`, `meta`, etc.) |
| `TRACE_KEYWORD_TT` | 110 | Trace keyword (`dictionary`, `symbols`, `on`, `off`, etc.) |
| `SYSTEM_CONSTANT_TT` | 111 | System constant (`#dictionary_table`, `#code_offset`, etc.) |
| `OPCODE_MACRO_TT` | 112 | Pseudo-opcode macro (Glulx-only: `push`, `pull`, `dload`, `dstore`) |

### 1.3.3 Context-Sensitive Lexing

Unlike many compilers, the lexer is **not context-free**. It has
8,192 possible lexical states, determined by 13 independent flags (one
for each of the 11 keyword groups — `opcode_names`, `directives`,
`trace_keywords`, `segment_markers`, `directive_keywords`,
`misc_keywords`, `statements`, `conditions`, `system_functions`,
`system_constants`, `local_variables` — plus the `return_sp_as_variable`
and `dont_enter_into_symbol_table` flags). The same identifier text can
produce different token types depending on context. For example,
`default` may be:

- A `DIRECTIVE_TT` when parsed as a top-level directive (`Default`).
- A `STATEMENT_TT` when parsed inside a `switch` block (`default:`).

The higher-level parser controls which keyword groups are enabled before
each call to the tokenizer, so that identifiers are correctly classified.

The keyword groups are stored in a single hash table built at compiler
startup. When an identifier is looked up, the lexer walks the hash
collision chain and returns the first match whose group is currently
enabled. Because the parser typically enables only the keyword group(s)
relevant to the current syntactic context, collisions between groups
rarely matter in practice. If no enabled keyword matches, the identifier
is looked up in the symbol table and returned as `SYMBOL_TT`.

### 1.3.4 Whitespace Handling

Whitespace characters (spaces, tabs, newlines, carriage returns) serve
only to separate tokens. The lexer consumes all consecutive whitespace
and resumes tokenizing at the next non-whitespace character. Whitespace
within quoted strings is, of course, significant and is handled by the
string-scanning code (see §1.7).

> Note: tab characters are translated to spaces during the
> source-to-ISO preprocessing stage (in the `source_to_iso_grid`
> conversion table) before the tokenizer sees them; the tokenizer's
> whitespace classification proper recognises only space (0x20),
> linefeed (0x0A), and carriage return (0x0D).

### 1.3.5 Token Lookahead

The lexer maintains a circular buffer of 6 token positions, allowing up
to 5 tokens to be "put back" for re-reading. This supports the limited
lookahead that the parser requires.

## 1.4 Identifiers

### 1.4.1 Valid Identifier Characters

An identifier begins with a **letter** (`a`–`z`, `A`–`Z`) or an
**underscore** (`_`), and continues with any combination of letters,
digits (`0`–`9`), and underscores:

```inform6
x
count
_private
myVariable2
MAX_SCORE
player_obj
```

Digits may not begin an identifier — `2ndPlace` is not a valid identifier
(the lexer would read `2` as a number token and `ndPlace` as a separate
identifier).

### 1.4.2 Case Sensitivity

Inform 6 is **case-insensitive** for identifiers (user-defined symbols).
The symbol table uses case-insensitive comparison, so `MyVar`, `myvar`,
and `MYVAR` all refer to the same symbol. The `Directive` keyword group
is also matched case-insensitively, so directive names such as `Ifdef`,
`IFDEF`, and `ifdef` are equivalent.

Other keyword groups — statement keywords, condition operators, system
functions, segment markers, miscellaneous keywords, etc. — are matched
**case-sensitively**. The lexer recognizes `if` as the statement keyword
but treats `If` or `IF` as ordinary identifier text. (For this reason,
the `misc_keywords` group includes both `the`/`The` and `a`/`A` as
distinct entries.)

```inform6
Global score;
! All of these refer to the same variable:
score = 10;
Score = 10;
SCORE = 10;
```

The conventional style is to use lowercase for keywords, mixed case or
lowercase for identifiers, and uppercase for named constants.

### 1.4.3 Identifier Length

Identifiers may be of **any length**, and the entire identifier is
significant for symbol lookup. The lexer reads identifier characters into
a dynamically grown buffer with no fixed length cap, so two identifiers
that differ at any position — even far past character 32 — are treated
as distinct symbols. (Earlier versions of Inform 6 enforced a 32-character
limit; that limit was removed in Inform 6.34.)

### 1.4.4 Reserved Words (Keywords)

The following identifiers are reserved as keywords in some or all contexts.
They cannot be used as user-defined symbol names when the relevant keyword
group is active.

**Directives** (top-level commands):

`Abbreviate`, `Array`, `Attribute`, `Class`, `Constant`, `Default`,
`Dictionary`, `End`, `Endif`, `Extend`, `Fake_action`, `Global`,
`Ifdef`, `Ifndef`, `Ifnot`, `Ifv3`, `Ifv5`, `Iftrue`, `Iffalse`,
`Import`, `Include`, `Link`, `Lowstring`, `Message`, `Nearby`,
`Object`, `Origsource`, `Property`, `Release`, `Replace`, `Serial`,
`Switches`, `Statusline`, `Stub`, `System_file`, `Trace`, `Undef`,
`Verb`, `Version`, `Zcharacter`.

**Statement keywords** (inside routines):

`box`, `break`, `continue`, `default`, `do`, `else`, `font`, `for`,
`give`, `if`, `inversion`, `jump`, `move`, `new_line`, `objectloop`,
`print`, `print_ret`, `quit`, `read`, `remove`, `restore`, `return`,
`rfalse`, `rtrue`, `save`, `spaces`, `string`, `style`, `switch`,
`until`, `while`.

**Condition operators** (in expressions):

`has`, `hasnt`, `in`, `notin`, `ofclass`, `or`, `provides`.

**System functions:**

`child`, `children`, `elder`, `eldest`, `indirect`, `parent`, `random`,
`sibling`, `younger`, `youngest`, `metaclass`, `glk`.

**Object segment markers** (in object/class definitions):

`class`, `has`, `private`, `with`.

**Miscellaneous keywords** (in `print` statements, grammar, etc.):

`char`, `name`, `the`, `a`, `an`, `The`, `number`, `roman`, `reverse`,
`bold`, `underline`, `fixed`, `on`, `off`, `to`, `address`, `string`,
`object`, `near`, `from`, `property`, `A`.

**Directive keywords** (arguments to directives):

`alias`, `long`, `additive`, `score`, `time`, `noun`, `held`, `multi`,
`multiheld`, `multiexcept`, `multiinside`, `creature`, `special`,
`number`, `scope`, `topic`, `reverse`, `meta`, `only`, `replace`,
`first`, `last`, `string`, `table`, `buffer`, `data`, `initial`,
`initstr`, `with`, `private`, `has`, `class`, `error`, `fatalerror`,
`warning`, `terminating`, `static`, `individual`.

**Trace keywords:**

`dictionary`, `symbols`, `objects`, `verbs`, `assembly`, `expressions`,
`lines`, `tokens`, `linker`, `on`, `off`.

Because keyword recognition is context-dependent, some words appear in
multiple lists (for example, `has` is both a condition operator and a
segment marker). The parser enables the appropriate keyword group for
each syntactic context.

## 1.5 Numeric Literals

### 1.5.1 Decimal Integers

A sequence of one or more decimal digits (`0`–`9`) forms a decimal integer
literal:

```inform6
0
42
1000
65535
```

### 1.5.2 Hexadecimal Integers

A `$` prefix followed by one or more hexadecimal digits (`0`–`9`,
`a`–`f`, `A`–`F`) forms a hexadecimal literal:

```inform6
$ff       ! 255
$0A       ! 10
$FFFF     ! 65535
$DeadBeef ! 3735928559 (Glulx only — too large for Z-machine)
```

If no valid hexadecimal digit follows `$`, the compiler reports an error:
"Hexadecimal number expected after '$'".

### 1.5.3 Binary Integers

A `$$` prefix followed by one or more binary digits (`0` or `1`) forms a
binary literal:

```inform6
$$11001010   ! 202
$$10000000   ! 128
$$1          ! 1
$$0          ! 0
```

If no valid binary digit follows `$$`, the compiler reports an error:
"Binary number expected after '$$'".

### 1.5.4 Floating-Point Literals (Glulx Only)

When compiling for Glulx, the `$` prefix can also introduce floating-point
literals. A sign character (`+` or `-`) after `$` signals a 32-bit float:

```inform6
$+1.0       ! 1.0 as a 32-bit IEEE 754 float
$-3.14      ! -3.14
$+2.5e10    ! 2.5 × 10^10
$+1.0e-3    ! 0.001
```

For 64-bit doubles, use `$>` for the high 32 bits and `$<` for the low
32 bits:

```inform6
$>+1.0      ! High word of 1.0 as a 64-bit double
$<+1.0      ! Low word of 1.0 as a 64-bit double
```

Floating-point literals are not available in Z-code and produce a
compile-time error if used.

### 1.5.5 Negative Numbers

Negative numbers are **not** single tokens. The expression `-5` is
tokenized as two separate tokens: the minus separator (`-`) followed by
the number `5`. The parser handles unary negation at the expression level.

### 1.5.6 Value Ranges

The range of valid integer values depends on the target platform:

| Target | Word Size | Range |
| ------ | --------- | ----- |
| Z-machine | 16-bit signed | −32768 to 32767 |
| Glulx | 32-bit signed | −2,147,483,648 to 2,147,483,647 |

Numeric literals are stored internally as 32-bit integers regardless of
target. Values that exceed the target word size are silently truncated.

## 1.6 String Literals (Double-Quoted Strings)

### 1.6.1 Basic Strings

A double-quoted string is delimited by `"` characters:

```inform6
print "Hello, world!^";
print "She said, ~Hello!~^";
```

The string is scanned from the opening `"` to the closing `"`. Within
a string, most characters represent themselves, but several escape
sequences are recognized.

### 1.6.2 Escape Sequences

Inform 6 strings use a set of escape sequences that differ from the C
convention. The backslash is used only for line continuation, not for
character escapes:

| Sequence | Meaning |
| -------- | ------- |
| `^` | Newline |
| `~` | Double-quote character (`"`) |
| `@@`*decimal* | ZSCII character by decimal code (0–1023) |
| `@{`*hex*`}` | Unicode character by hexadecimal code point (up to 6 digits) |
| `@'`*X* | Accented character (e.g., `@'e` for e-acute) |
| `@``*X* | Accented character with grave accent |
| `@^`*X* | Accented character with circumflex |
| `@:`*X* | Accented character with diaeresis/umlaut |
| `@~`*X* | Accented character with tilde |
| `@/`*X* | Accented character with slash (`@/o`, `@/O`) |
| `@o`*X* | Accented character with ring (`@oa`, `@oA`) |
| `@oe` | oe ligature |
| `@ae` | ae ligature |
| `@cc` | c-cedilla |
| `@ss` | German sharp-s (eszett) |
| `@th` | Icelandic thorn (lowercase) |
| `@et` | Icelandic eth (lowercase) |
| `@Th` | Icelandic thorn (uppercase) |
| `@Et` | Icelandic eth (uppercase) |
| `@LL` | L-stroke (uppercase) |
| `@!!` | Inverted exclamation mark |
| `@??` | Inverted question mark |
| `@>>` | Right guillemet |
| `@<<` | Left guillemet |
| `@OE` | OE ligature (uppercase) |
| `@AE` | AE ligature (uppercase) |
| `@cC` | C-cedilla (uppercase) |
| `@`*dd* | Dynamic string (abbreviation) number *dd* (two digits, 00–99) |
| `@(`*sym*`)` | Dynamic string by symbol name or decimal number |

Examples:

```inform6
print "caf@'e^";                  ! prints: café
print "na@:ive^";                 ! prints: naïve
print "Price: @@163100^";         ! prints: Price: £100 (ZSCII 163 = £)
print "@{00C9}lan^";              ! prints: Élan
print "She said, ~Hello.~^";      ! prints: She said, "Hello."
print "@<<Bonjour@>>^";           ! prints: «Bonjour»
```

The `@@` escape is particularly useful for characters that have special
meaning in Inform strings. For example, `@@94` produces a literal
circumflex (ASCII 94) without it being interpreted as a newline, and
`@@126` produces a literal tilde without it being interpreted as a
double-quote.

### 1.6.3 Line Continuation in Strings

Strings may span multiple source lines. When a newline is encountered
inside a double-quoted string, the compiler:

1. Strips trailing spaces from the current line.
2. If the last non-space character before the newline is `^` (the newline
   escape), no additional space is inserted.
3. Otherwise, a single space is inserted in place of the line break.
4. Leading whitespace on the continuation line is consumed.

```inform6
print "This is a long string that
       continues on the next line.^";
! Prints: This is a long string that continues on the next line.
```

A backslash (`\`) at the end of a line provides explicit line
continuation: the backslash, any trailing whitespace, and the newline are
all consumed, and the string continues with the first character of the
next line (after skipping leading whitespace):

```inform6
print "This is also a long \
       string.^";
! Prints: This is also a long string.
```

If `\` appears in a string but is not followed by whitespace and a
newline, the compiler reports an error: "empty rest of line after '\\'
in string."

### 1.6.4 String Length Limits

There is no fixed compile-time limit on string length. Strings are
processed into Z-encoded text (for Z-machine) or Unicode sequences
(for Glulx) and stored in the story file. The practical limit is the
available story file memory.

## 1.7 Character Literals (Single-Quoted Characters)

A single character enclosed in single quotes produces a numeric value
equal to the character's ZSCII (or Unicode) code:

```inform6
x = 'A';    ! x = 65
x = '0';    ! x = 48
x = ' ';    ! x = 32
```

Single-character literals are tokenized as `SQ_TT` tokens. The parser
determines whether a single-quoted token is a character constant or a
dictionary word based on its length: a single character between the
quotes is a character constant (producing a number), while multiple
characters form a dictionary word (see §1.8).

Empty single quotes (`''`) are an error: "No text between quotation
marks."

## 1.8 Dictionary Word Literals

### 1.8.1 Basic Dictionary Words

When a single-quoted token contains more than one character, it is a
**dictionary word literal** — a reference to an entry in the game's
dictionary:

```inform6
if (noun == 'lamp') print "It's a lamp.^";
if (action == ##Take && noun == 'key') print "You take the key.^";
```

The parser compiles dictionary words into addresses in the dictionary
table. Two dictionary words are equal if and only if they refer to the
same dictionary entry.

### 1.8.2 Dictionary Word Length Limits

Dictionary words are truncated to a maximum length that depends on the
target and version:

| Target | Max Characters |
| ------ | -------------- |
| Z-machine v3 | 6 |
| Z-machine v4+ | 9 |
| Glulx | `DICT_WORD_SIZE` (default 9, configurable) |

Characters beyond the limit are silently discarded. In Z-machine v3
each dictionary word is compressed into 4 bytes of Z-encoded text
holding up to 6 Z-characters; in v4+ it is 6 bytes holding
up to 9 Z-characters. Neither figure is user-configurable: the Z-code
compiler forces `DICT_WORD_SIZE` to 6 and rejects any attempt to change
it. For Glulx, the memory setting `$DICT_WORD_SIZE` can be set to any
value and defaults to 9.

```inform6
! In Z-machine v5: 'pineapples' is truncated to 'pineapple' (9 chars)
! and matches 'pineapple' (also 9 chars, so not truncated)
```

### 1.8.3 Suffix Flags

Dictionary words can carry **suffix flags** that annotate how the word is
used in the game's grammar. The suffix `//` followed by flag characters
is appended after the word:

| Suffix | Flag | Meaning |
| ------ | ---- | ------- |
| `//p` | `PLURAL_DFLAG` (4) | Word is a plural |
| `//s` | `SING_DFLAG` (16) | Word is singular |
| `//n` | `NOUN_DFLAG` (128) | Word is a noun |

Flags can be combined and negated with `~`:

```inform6
'coins//p'        ! Mark 'coins' as plural
'gold//pn'        ! Mark 'gold' as both plural and noun
'knife//~p'       ! Clear the plural flag on 'knife'
```

The `~` prefix toggles negation for the immediately following flag
character. Positive flags are OR'd into the dictionary entry; negative
flags are used to clear previously set flags.

Additional flags are set automatically by the compiler: `VERB_DFLAG` (1)
for words used as verbs in grammar definitions, `META_DFLAG` (2) for meta
verbs, `PREP_DFLAG` (8) for prepositions, and `TRUNC_DFLAG` (64) for
words that were truncated to fit the dictionary word size.

### 1.8.4 Special Quoting with `@`

Single-quoted dictionary words may contain `@` escape sequences (the same
ones used in double-quoted strings) for accented and special characters:

```inform6
'caf@'e'       ! The dictionary word "café"
'na@:ive'      ! The dictionary word "naïve"
```

A `@` at the end of a single-quoted literal (before the closing `'`) is
treated specially: the closing `'` does not terminate the literal if it
immediately follows `@`. This allows the escape sequence to span across
what would otherwise be the end of the literal.

## 1.9 Separators and Operators

### 1.9.1 Complete List of Separator Tokens

The lexer recognizes exactly **49 separator tokens**. They are
matched using longest-match semantics — the lexer always tries to match
the longest possible separator before falling back to shorter ones.

The complete list, grouped by function:

**Arithmetic operators:**

| Separator | Name | Description |
| --------- | ---- | ----------- |
| `+` | `PLUS_SEP` | Addition |
| `-` | `MINUS_SEP` | Subtraction / unary negation |
| `*` | `TIMES_SEP` | Multiplication |
| `/` | `DIVIDE_SEP` | Division |
| `%` | `REMAINDER_SEP` | Modulo (remainder) |
| `++` | `INC_SEP` | Increment |
| `--` | `DEC_SEP` | Decrement |

**Comparison operators:**

| Separator | Name | Description |
| --------- | ---- | ----------- |
| `==` | `CONDEQUALS_SEP` | Equality test |
| `~=` | `NOTEQUAL_SEP` | Inequality test |
| `>` | `GREATER_SEP` | Greater than |
| `<` | `LESS_SEP` | Less than |
| `>=` | `GE_SEP` | Greater than or equal |
| `<=` | `LE_SEP` | Less than or equal |

**Logical and bitwise operators:**

| Separator | Name | Description |
| --------- | ---- | ----------- |
| `&&` | `LOGAND_SEP` | Logical AND |
| <code>&#124;&#124;</code> | `LOGOR_SEP` | Logical OR |
| `~~` | `LOGNOT_SEP` | Logical NOT |
| `&` | `ARTAND_SEP` | Bitwise AND |
| <code>&#124;</code> | `ARTOR_SEP` | Bitwise OR |
| `~` | `ARTNOT_SEP` | Bitwise NOT |

**Assignment:**

| Separator | Name | Description |
| --------- | ---- | ----------- |
| `=` | `SETEQUALS_SEP` | Assignment |

**Property and message access:**

| Separator | Name | Description |
| --------- | ---- | ----------- |
| `.` | `PROPERTY_SEP` | Property access |
| `..` | `MESSAGE_SEP` | Message send |
| `.&` | `PROPADD_SEP` | Property data address |
| `.#` | `PROPNUM_SEP` | Property data length |
| `..&` | `MPROPADD_SEP` | Individual property data address |
| `..#` | `MPROPNUM_SEP` | Individual property data length |

**Array access:**

| Separator | Name | Description |
| --------- | ---- | ----------- |
| `->` | `ARROW_SEP` | Byte array access |
| `-->` | `DARROW_SEP` | Word array access |

**Delimiters and punctuation:**

| Separator | Name | Description |
| --------- | ---- | ----------- |
| `(` | `OPENB_SEP` | Open parenthesis |
| `)` | `CLOSEB_SEP` | Close parenthesis |
| `[` | `OPEN_SQUARE_SEP` | Open square bracket |
| `]` | `CLOSE_SQUARE_SEP` | Close square bracket |
| `{` | `OPEN_BRACE_SEP` | Open brace |
| `}` | `CLOSE_BRACE_SEP` | Close brace |
| `,` | `COMMA_SEP` | Comma |
| `:` | `COLON_SEP` | Colon |
| `::` | `SUPERCLASS_SEP` | Superclass operator |
| `;` | `SEMICOLON_SEP` | Semicolon |

**Assembly and special:**

| Separator | Name | Description |
| --------- | ---- | ----------- |
| `@` | `AT_SEP` | Assembly opcode prefix |
| `?` | `BRANCH_SEP` | Branch label (assembly) |
| `?~` | `NBRANCH_SEP` | Branch-not label (assembly) |
| `$` | `DOLLAR_SEP` | Dollar (hex prefix / separator) |

**Hash-prefixed separators:**

| Separator | Name | Description |
| --------- | ---- | ----------- |
| `#` | `HASH_SEP` | Directive prefix |
| `##` | `HASHHASH_SEP` | Action name prefix |
| `#a$` | `HASHADOLLAR_SEP` | System constant (array) |
| `#g$` | `HASHGDOLLAR_SEP` | System constant (global) |
| `#n$` | `HASHNDOLLAR_SEP` | System constant (number/entry) |
| `#r$` | `HASHRDOLLAR_SEP` | System constant (routine) |
| `#w$` | `HASHWDOLLAR_SEP` | System constant (word/dictionary) |

### 1.9.2 Multi-Character Separator Matching

Multi-character separators are matched greedily. Given the source text
`-->`, the lexer produces the single token `DARROW_SEP` rather than
`MINUS_SEP` followed by `ARROW_SEP`. The matching algorithm:

1. The first character is looked up in the tokenizer grid to find which
   separators begin with that character and how many.
2. The lexer checks each candidate separator in order, comparing against
   lookahead characters. The first match wins.
3. Candidates are ordered so that if one separator is an initial substring
   of another, the longer one appears first. This ensures greedy matching:
   a shorter separator cannot match prematurely when a longer one would
   also match.

For example, the separators beginning with `.` are ordered: `.&`, `.#`,
`..&`, `..#`, `..`, `.` — the two-character `.&` and `.#` are not
prefixes of the three-character `..&` and `..#`, so their relative order
does not matter. But `..` is a prefix of `..&` and `..#`, so it comes
after them; and `.` is a prefix of all the others, so it comes last.

## 1.10 The Hash Character and Directives

### 1.10.1 `#` as Directive Prefix

At the top level of source code, the `#` character introduces a compiler
directive. Although Inform 6 directives can be written without `#` in most
cases, using `#` makes the directive nature explicit:

```inform6
#Ifdef DEBUG;
    print "Debug mode enabled.^";
#Endif;
```

When `#` appears in the token stream, it is returned as `HASH_SEP`. The
parser then reads the following identifier as a directive name.

### 1.10.2 `##` for Action Constants

The `##` prefix before an action name produces the numeric value of that
action:

```inform6
if (action == ##Take) print "You're taking something.^";
if (action == ##Drop) print "You're dropping something.^";
```

When the lexer sees `##`, it reads the following identifier as part of the
same token. The combined `##ActionName` is returned as a `SEP_TT` with
value `HASHHASH_SEP`, and the identifier text is included in the token
text. The parser then resolves the action name to its numeric constant
value.

The identifier following `##` must begin with a letter — whitespace
between `##` and the name is not permitted.

### 1.10.3 System Constant Access: `#a$`, `#g$`, `#n$`, `#r$`, `#w$`

These five compound separators provide low-level access to compiler-
generated tables and addresses:

| Prefix | Meaning | Example |
| ------ | ------- | ------- |
| `#a$` | Array address | `#a$MyArray` — address of array `MyArray` |
| `#g$` | Global variable address | `#g$score` — address of global `score` |
| `#n$` | Entry number | `#n$MyAttr` — number assigned to attribute `MyAttr` |
| `#r$` | Routine address | `#r$MyFunc` — packed address of routine `MyFunc` |
| `#w$` | Dictionary word address | `#w$lamp` — address of dictionary entry for "lamp" |

Like `##`, these read the following identifier characters into the token
text. Whitespace between the prefix and the identifier is not permitted.

```inform6
x = #r$Main;       ! Packed address of the Main routine
x = #g$score;      ! Address of the global variable 'score'
x = #w$lamp;       ! Dictionary address of the word 'lamp'
```

For `#a$`, `#g$`, `#r$`, and `##`, the character following the prefix
must be a letter or underscore (the start of an identifier); a digit in
this position is rejected with the error "Alphabetic character expected
after ...". For `#n$` and `#w$`, the following character may be any
non-whitespace character; only an immediately-following whitespace
character is rejected, with the error "Character expected after ...".
After the first character, all six prefixes accept any combination of
letters, digits, and underscores.

### 1.10.4 System Constants

The `SYSTEM_CONSTANT_TT` keyword group provides named access to
compiler-internal values. These are typically used with the `#` prefix
or evaluated at compile time:

```inform6
x = #dictionary_table;      ! Address of the dictionary
x = #highest_object_number;  ! Number of the last object
x = #grammar_table;          ! Address of the grammar table
```

The complete list of all 63 system constants is:

`adjectives_table`, `actions_table`, `classes_table`,
`identifiers_table`, `preactions_table`, `version_number`,
`largest_object`, `strings_offset`, `code_offset`, `dict_par1`,
`dict_par2`, `dict_par3`, `actual_largest_object`,
`static_memory_offset`, `array_names_offset`,
`readable_memory_offset`, `cpv__start`, `cpv__end`, `ipv__start`,
`ipv__end`, `array__start`, `array__end`,
`lowest_attribute_number`, `highest_attribute_number`,
`attribute_names_array`, `lowest_property_number`,
`highest_property_number`, `property_names_array`,
`lowest_action_number`, `highest_action_number`,
`action_names_array`, `lowest_fake_action_number`,
`highest_fake_action_number`, `fake_action_names_array`,
`lowest_routine_number`, `highest_routine_number`, `routines_array`,
`routine_names_array`, `routine_flags_array`,
`lowest_global_number`, `highest_global_number`, `globals_array`,
`global_names_array`, `global_flags_array`, `lowest_array_number`,
`highest_array_number`, `arrays_array`, `array_names_array`,
`array_flags_array`, `lowest_constant_number`,
`highest_constant_number`, `constants_array`, `constant_names_array`,
`lowest_class_number`, `highest_class_number`, `class_objects_array`,
`lowest_object_number`, `highest_object_number`, `oddeven_packing`,
`grammar_table`, `dictionary_table`, `dynam_string_table`,
`highest_meta_action_number`.

For detailed descriptions, VM availability, and usage examples for each
of these constants, see Appendix G (System Constants Reference).

---

*Copyright © 2026 Software Freedom Conservancy, Inc.*
*Licensed under the GNU General Public License, version 3 or later.*
