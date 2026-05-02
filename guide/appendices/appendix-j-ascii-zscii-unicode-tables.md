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

# Appendix J: ASCII, ZSCII, and Unicode Tables

This appendix provides comprehensive reference tables for the character sets
used by the compiler: ASCII, ZSCII, Unicode, and the mappings between them.

---

## §J.1 Character Representation Overview

The Inform 6.44 compiler uses six internal character representations:

1. **ASCII** — Plain ASCII characters in range 0x20 to 0x7E (unsigned 7-bit).
2. **Source** — Raw bytes from source code (unsigned 8-bit).
3. **ISO** — Plain ASCII or ISO 8859-1 to 8859-9, per `-C` switch; or
   individual UTF-8 bytes in Unicode mode (unsigned 8-bit).
4. **ZSCII** — Z-machine character set (unsigned 10-bit, 0–1023).
5. **Textual** — Escape notation such as `@'e` for e-acute or `@{03a3}` for
   Greek sigma; in Unicode mode, UTF-8 sequences.
6. **Unicode** — Unifying character set (unsigned 16-bit, 0–0xFFFF).

There is also a seventh form: sequences of 5-bit **Z-chars** which encode
ZSCII into the story file in compressed form.

### Conversion Pipeline

The compiler converts characters through the following pipeline:

```
Source → (source_to_iso_grid) → ISO → (iso_to_unicode) → Unicode
  → (unicode_to_zscii) → ZSCII → (zscii_to_alphabet_grid) → Z-chars (5-bit)
```

Each arrow represents a mapping function or lookup table that transforms one
representation into the next. The pipeline flows from raw source file bytes
through to the compressed Z-character encoding stored in the story file.

---

## §J.2 ASCII Table (0x00–0x7F)

The full 128-entry ASCII character set, with notes on how the compiler
treats each character in source input.

### §J.2.1 Control Characters (0–31)

| Dec | Hex | Name | Inform 6 Notes |
|-----|------|------|----------------|
| 0 | 0x00 | NUL | End of file marker in source |
| 1 | 0x01 | SOH | Converted to `?` by source filter |
| 2 | 0x02 | STX | Converted to `?` by source filter |
| 3 | 0x03 | ETX | Converted to `?` by source filter |
| 4 | 0x04 | EOT | Converted to `?` by source filter |
| 5 | 0x05 | ENQ | Converted to `?` by source filter |
| 6 | 0x06 | ACK | Converted to `?` by source filter |
| 7 | 0x07 | BEL | Converted to `?` by source filter |
| 8 | 0x08 | BS | Converted to `?` by source filter |
| 9 | 0x09 | TAB | Converted to space by source filter |
| 10 | 0x0A | LF | Newline; preserved in source |
| 11 | 0x0B | VT | Converted to `?` |
| 12 | 0x0C | FF | Form feed; converted to newline |
| 13 | 0x0D | CR | Carriage return; preserved |
| 14 | 0x0E | SO | Converted to `?` by source filter |
| 15 | 0x0F | SI | Converted to `?` by source filter |
| 16 | 0x10 | DLE | Converted to `?` by source filter |
| 17 | 0x11 | DC1 | Converted to `?` by source filter |
| 18 | 0x12 | DC2 | Converted to `?` by source filter |
| 19 | 0x13 | DC3 | Converted to `?` by source filter |
| 20 | 0x14 | DC4 | Converted to `?` by source filter |
| 21 | 0x15 | NAK | Converted to `?` by source filter |
| 22 | 0x16 | SYN | Converted to `?` by source filter |
| 23 | 0x17 | ETB | Converted to `?` by source filter |
| 24 | 0x18 | CAN | Converted to `?` by source filter |
| 25 | 0x19 | EM | Converted to `?` by source filter |
| 26 | 0x1A | SUB | Converted to `?` by source filter |
| 27 | 0x1B | ESC | Converted to `?` by source filter |
| 28 | 0x1C | FS | Converted to `?` by source filter |
| 29 | 0x1D | GS | Converted to `?` by source filter |
| 30 | 0x1E | RS | Converted to `?` by source filter |
| 31 | 0x1F | US | Converted to `?` by source filter |

### §J.2.2 Printable Characters (32–126)

| Dec | Hex | Char | Inform 6 Notes |
|-----|------|------|----------------|
| 32 | 0x20 | *(space)* | Z-char 0 in Z-machine encoding |
| 33 | 0x21 | `!` | Comment delimiter in source (after `!` to end of line) |
| 34 | 0x22 | `"` | String delimiter; use `~` in strings to output a double-quote |
| 35 | 0x23 | `#` | Directive prefix; also in A2 alphabet |
| 36 | 0x24 | `$` | Hex literal prefix (`$ff`), also used for system constants (`$$`) |
| 37 | 0x25 | `%` | Binary literal prefix (`%%0110`) |
| 38 | 0x26 | `&` | Bitwise AND operator; `&&` is logical AND |
| 39 | 0x27 | `'` | Dictionary word delimiter (`'word'`) |
| 40 | 0x28 | `(` | Grouping, function call; in A2 alphabet |
| 41 | 0x29 | `)` | Grouping, function call; in A2 alphabet |
| 42 | 0x2A | `*` | Multiplication; in dictionary words means any single char |
| 43 | 0x2B | `+` | Addition operator |
| 44 | 0x2C | `,` | Separator; in A2 alphabet |
| 45 | 0x2D | `-` | Subtraction / negation; in A2 alphabet |
| 46 | 0x2E | `.` | Property access operator; in A2 alphabet |
| 47 | 0x2F | `/` | Division operator; in A2 alphabet |
| 48 | 0x30 | `0` | Digit; in A2 alphabet |
| 49 | 0x31 | `1` | Digit; in A2 alphabet |
| 50 | 0x32 | `2` | Digit; in A2 alphabet |
| 51 | 0x33 | `3` | Digit; in A2 alphabet |
| 52 | 0x34 | `4` | Digit; in A2 alphabet |
| 53 | 0x35 | `5` | Digit; in A2 alphabet |
| 54 | 0x36 | `6` | Digit; in A2 alphabet |
| 55 | 0x37 | `7` | Digit; in A2 alphabet |
| 56 | 0x38 | `8` | Digit; in A2 alphabet |
| 57 | 0x39 | `9` | Digit; in A2 alphabet |
| 58 | 0x3A | `:` | Label terminator; in A2 alphabet |
| 59 | 0x3B | `;` | Statement terminator |
| 60 | 0x3C | `<` | Less-than; `<<` is action shorthand |
| 61 | 0x3D | `=` | Assignment; `==` is equality test |
| 62 | 0x3E | `>` | Greater-than |
| 63 | 0x3F | `?` | In A2 alphabet |
| 64 | 0x40 | `@` | Escape character in strings (see §J.8) |
| 65 | 0x41 | `A` | Letter; in A1 alphabet |
| 66 | 0x42 | `B` | Letter; in A1 alphabet |
| 67 | 0x43 | `C` | Letter; in A1 alphabet |
| 68 | 0x44 | `D` | Letter; in A1 alphabet |
| 69 | 0x45 | `E` | Letter; in A1 alphabet |
| 70 | 0x46 | `F` | Letter; in A1 alphabet |
| 71 | 0x47 | `G` | Letter; in A1 alphabet |
| 72 | 0x48 | `H` | Letter; in A1 alphabet |
| 73 | 0x49 | `I` | Letter; in A1 alphabet |
| 74 | 0x4A | `J` | Letter; in A1 alphabet |
| 75 | 0x4B | `K` | Letter; in A1 alphabet |
| 76 | 0x4C | `L` | Letter; in A1 alphabet |
| 77 | 0x4D | `M` | Letter; in A1 alphabet |
| 78 | 0x4E | `N` | Letter; in A1 alphabet |
| 79 | 0x4F | `O` | Letter; in A1 alphabet |
| 80 | 0x50 | `P` | Letter; in A1 alphabet |
| 81 | 0x51 | `Q` | Letter; in A1 alphabet |
| 82 | 0x52 | `R` | Letter; in A1 alphabet |
| 83 | 0x53 | `S` | Letter; in A1 alphabet |
| 84 | 0x54 | `T` | Letter; in A1 alphabet |
| 85 | 0x55 | `U` | Letter; in A1 alphabet |
| 86 | 0x56 | `V` | Letter; in A1 alphabet |
| 87 | 0x57 | `W` | Letter; in A1 alphabet |
| 88 | 0x58 | `X` | Letter; in A1 alphabet |
| 89 | 0x59 | `Y` | Letter; in A1 alphabet |
| 90 | 0x5A | `Z` | Letter; in A1 alphabet |
| 91 | 0x5B | `[` | Routine / switch block opener |
| 92 | 0x5C | `\` | Backslash; in A2 alphabet |
| 93 | 0x5D | `]` | Routine / switch block closer |
| 94 | 0x5E | `^` | In strings: newline (ZSCII 10); literal via `@@94` |
| 95 | 0x5F | `_` | Identifier character; in A2 alphabet |
| 96 | 0x60 | `` ` `` | Unused in language syntax |
| 97 | 0x61 | `a` | Letter; in A0 alphabet |
| 98 | 0x62 | `b` | Letter; in A0 alphabet |
| 99 | 0x63 | `c` | Letter; in A0 alphabet |
| 100 | 0x64 | `d` | Letter; in A0 alphabet |
| 101 | 0x65 | `e` | Letter; in A0 alphabet |
| 102 | 0x66 | `f` | Letter; in A0 alphabet |
| 103 | 0x67 | `g` | Letter; in A0 alphabet |
| 104 | 0x68 | `h` | Letter; in A0 alphabet |
| 105 | 0x69 | `i` | Letter; in A0 alphabet |
| 106 | 0x6A | `j` | Letter; in A0 alphabet |
| 107 | 0x6B | `k` | Letter; in A0 alphabet |
| 108 | 0x6C | `l` | Letter; in A0 alphabet |
| 109 | 0x6D | `m` | Letter; in A0 alphabet |
| 110 | 0x6E | `n` | Letter; in A0 alphabet |
| 111 | 0x6F | `o` | Letter; in A0 alphabet |
| 112 | 0x70 | `p` | Letter; in A0 alphabet |
| 113 | 0x71 | `q` | Letter; in A0 alphabet |
| 114 | 0x72 | `r` | Letter; in A0 alphabet |
| 115 | 0x73 | `s` | Letter; in A0 alphabet |
| 116 | 0x74 | `t` | Letter; in A0 alphabet |
| 117 | 0x75 | `u` | Letter; in A0 alphabet |
| 118 | 0x76 | `v` | Letter; in A0 alphabet |
| 119 | 0x77 | `w` | Letter; in A0 alphabet |
| 120 | 0x78 | `x` | Letter; in A0 alphabet |
| 121 | 0x79 | `y` | Letter; in A0 alphabet |
| 122 | 0x7A | `z` | Letter; in A0 alphabet |
| 123 | 0x7B | `{` | Used in `@{hex}` escape sequences |
| 124 | 0x7C | `\|` | Bitwise OR operator; `\|\|` is logical OR |
| 125 | 0x7D | `}` | Closes `@{hex}` escape sequences |
| 126 | 0x7E | `~` | In strings: double-quote `"` (ZSCII 34); literal via `@@126` |

### §J.2.3 DEL (127)

| Dec | Hex | Name | Inform 6 Notes |
|-----|------|------|----------------|
| 127 | 0x7F | DEL | Converted to `?` by source filter |

---

## §J.3 Z-Machine Alphabets (A0, A1, A2)

### §J.3.1 Default Alphabets

**[Z-machine]**

The Z-machine encodes text using three alphabets of 26 characters each. The
default alphabets are:

```inform6
! Default Z-machine alphabets (internal compiler data)
! alphabet[0] = "abcdefghijklmnopqrstuvwxyz"
! alphabet[1] = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
! alphabet[2] = " ^0123456789.,!?_#'~/\-:()"
```

The following table shows all 26 positions across all three alphabets:

| Position | Z-char | A0 | A1 | A2 |
|----------|--------|----|----|-----|
| 0 | 6 | a | A | *(padding)* |
| 1 | 7 | b | B | *(newline)* |
| 2 | 8 | c | C | 0 |
| 3 | 9 | d | D | 1 |
| 4 | 10 | e | E | 2 |
| 5 | 11 | f | F | 3 |
| 6 | 12 | g | G | 4 |
| 7 | 13 | h | H | 5 |
| 8 | 14 | i | I | 6 |
| 9 | 15 | j | J | 7 |
| 10 | 16 | k | K | 8 |
| 11 | 17 | l | L | 9 |
| 12 | 18 | m | M | . |
| 13 | 19 | n | N | , |
| 14 | 20 | o | O | ! |
| 15 | 21 | p | P | ? |
| 16 | 22 | q | Q | _ |
| 17 | 23 | r | R | # |
| 18 | 24 | s | S | ' |
| 19 | 25 | t | T | ~ → " |
| 20 | 26 | u | U | / |
| 21 | 27 | v | V | \ |
| 22 | 28 | w | W | - |
| 23 | 29 | x | X | : |
| 24 | 30 | y | Y | ( |
| 25 | 31 | z | Z | ) |

**A2 special positions:**

- **Position 0** (Z-char 6 in A2): Padding/escape marker. When Z-char 6 is
  encountered in alphabet A2, it begins a 10-bit ZSCII literal encoded as two
  subsequent 5-bit Z-chars.
- **Position 1** (Z-char 7 in A2): Newline. The source character `^` is stored
  here; the Z-machine interprets A2 position 1 as a newline (ZSCII 10).
- **Position 19** (Z-char 25 in A2): The source character `~` is stored here;
  during story file output it is written as `"` (double-quote, ZSCII 34).

### §J.3.2 Z-Character Encoding

Each Z-character is a 5-bit value (0–31). Three Z-characters are packed into
each 16-bit word. The special Z-char values are:

| Z-char | Meaning |
|--------|---------|
| 0 | Space |
| 1 | Abbreviation from set 0 (followed by abbreviation number) |
| 2 | Abbreviation from set 1 (followed by abbreviation number) |
| 3 | Abbreviation from set 2 (followed by abbreviation number) |
| 4 | Shift to Alphabet 1 (next character only) |
| 5 | Shift to Alphabet 2 (next character only) |
| 6–31 | Letter at position (Z-char − 6) in current alphabet |

For characters not present in any of the three alphabets, a 4-Z-char escape
sequence is used: `5, 6, high5, low5`, where:

- `5` shifts to alphabet A2
- `6` triggers the ZSCII escape (A2 position 0)
- `high5` = ZSCII ÷ 32 (upper 5 bits)
- `low5` = ZSCII mod 32 (lower 5 bits)

### §J.3.3 Custom Alphabets via `Zcharacter`

**[Z-machine]**

The `Zcharacter` directive can replace the default alphabets:

```inform6
Zcharacter "abcdefghijklmnopqrstuvwxyz"
           "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
           "0123456789.,!?_#'~/@\-:()>";
```

Constraints:

- A0 and A1 strings must each contain exactly **26** characters.
- The A2 string must contain exactly **23** characters (positions 3–25);
  positions 0–2 are fixed (padding, newline, ZSCII escape).
- This directive is an error in Glulx mode.

---

## §J.4 ZSCII Character Set (0–1023)

### §J.4.1 ZSCII Range Overview

**[Z-machine]**

ZSCII is the Z-machine's native character set, using 10-bit values (0–1023).

| Range | Description |
|-------|-------------|
| 0 | Null (output: prints nothing) |
| 1–7 | Unused in standard; 5 is the pad character |
| 8 | Delete (input only) |
| 9 | Tab (output only, V6+) |
| 10 | Newline |
| 11 | Sentence space (output only, V6+) |
| 12 | Paragraph space (output only, V6+) |
| 13 | Carriage return (input only) |
| 14–26 | Unused |
| 27 | Escape (input only) |
| 28 | Cursor up (input only) |
| 29 | Cursor down (input only) |
| 30 | Cursor left (input only) |
| 31 | Cursor right (input only) |
| 32–126 | Standard ASCII printable characters (identical to ASCII) |
| 127–154 | Undefined |
| 155–251 | Extra characters (Unicode translation table) |
| 252 | Mouse double-click (input only) |
| 253 | Mouse single-click (input only) |
| 254 | (Reserved) |
| 255 | Undefined |
| 256–1023 | Defined by specification but unused in practice |

### §J.4.2 ZSCII-to-Unicode Conversion Functions

The compiler provides two conversion functions:

**`unicode_to_zscii(u)`**: For Unicode values < 0x7F, returns the value
unchanged (identity mapping). For values ≥ 0x7F, searches
`zscii_to_unicode_grid[]` for a match and returns 155 + index. Returns 5
(pad character) if no match is found.

**`zscii_to_unicode(z)`**: For ZSCII values < 0x80, returns the value
unchanged. For values 155–251, returns `zscii_to_unicode_grid[z - 155]`.
Returns `?` otherwise.

---

## §J.5 Default ZSCII-to-Unicode Translation Table (Latin-1/ASCII)

This is the standard Z-Machine Standard 1.0 default table used when `-C0` or
`-C1` is active. It contains 69 entries mapping ZSCII 155–223 to Unicode code
points. This is the standard Z-Machine Standard 1.0 default table used when `-C0` or
`-C1` is active. It contains 69 entries mapping ZSCII 155–223 to Unicode code
points.

| ZSCII | Unicode | Char | Escape | Description |
|-------|---------|------|--------|-------------|
| 155 | U+00E4 | ä | `@:a` | a-diaeresis |
| 156 | U+00F6 | ö | `@:o` | o-diaeresis |
| 157 | U+00FC | ü | `@:u` | u-diaeresis |
| 158 | U+00C4 | Ä | `@:A` | A-diaeresis |
| 159 | U+00D6 | Ö | `@:O` | O-diaeresis |
| 160 | U+00DC | Ü | `@:U` | U-diaeresis |
| 161 | U+00DF | ß | `@ss` | sz-ligature |
| 162 | U+00BB | » | `@>>` | right guillemet |
| 163 | U+00AB | « | `@<<` | left guillemet |
| 164 | U+00EB | ë | `@:e` | e-diaeresis |
| 165 | U+00EF | ï | `@:i` | i-diaeresis |
| 166 | U+00FF | ÿ | `@:y` | y-diaeresis |
| 167 | U+00CB | Ë | `@:E` | E-diaeresis |
| 168 | U+00CF | Ï | `@:I` | I-diaeresis |
| 169 | U+00E1 | á | `@'a` | a-acute |
| 170 | U+00E9 | é | `@'e` | e-acute |
| 171 | U+00ED | í | `@'i` | i-acute |
| 172 | U+00F3 | ó | `@'o` | o-acute |
| 173 | U+00FA | ú | `@'u` | u-acute |
| 174 | U+00FD | ý | `@'y` | y-acute |
| 175 | U+00C1 | Á | `@'A` | A-acute |
| 176 | U+00C9 | É | `@'E` | E-acute |
| 177 | U+00CD | Í | `@'I` | I-acute |
| 178 | U+00D3 | Ó | `@'O` | O-acute |
| 179 | U+00DA | Ú | `@'U` | U-acute |
| 180 | U+00DD | Ý | `@'Y` | Y-acute |
| 181 | U+00E0 | à | `` @`a `` | a-grave |
| 182 | U+00E8 | è | `` @`e `` | e-grave |
| 183 | U+00EC | ì | `` @`i `` | i-grave |
| 184 | U+00F2 | ò | `` @`o `` | o-grave |
| 185 | U+00F9 | ù | `` @`u `` | u-grave |
| 186 | U+00C0 | À | `` @`A `` | A-grave |
| 187 | U+00C8 | È | `` @`E `` | E-grave |
| 188 | U+00CC | Ì | `` @`I `` | I-grave |
| 189 | U+00D2 | Ò | `` @`O `` | O-grave |
| 190 | U+00D9 | Ù | `` @`U `` | U-grave |
| 191 | U+00E2 | â | `@^a` | a-circumflex |
| 192 | U+00EA | ê | `@^e` | e-circumflex |
| 193 | U+00EE | î | `@^i` | i-circumflex |
| 194 | U+00F4 | ô | `@^o` | o-circumflex |
| 195 | U+00FB | û | `@^u` | u-circumflex |
| 196 | U+00C2 | Â | `@^A` | A-circumflex |
| 197 | U+00CA | Ê | `@^E` | E-circumflex |
| 198 | U+00CE | Î | `@^I` | I-circumflex |
| 199 | U+00D4 | Ô | `@^O` | O-circumflex |
| 200 | U+00DB | Û | `@^U` | U-circumflex |
| 201 | U+00E5 | å | `@oa` | a-ring |
| 202 | U+00C5 | Å | `@oA` | A-ring |
| 203 | U+00F8 | ø | `@/o` | o-slash |
| 204 | U+00D8 | Ø | `@/O` | O-slash |
| 205 | U+00E3 | ã | `@~a` | a-tilde |
| 206 | U+00F1 | ñ | `@~n` | n-tilde |
| 207 | U+00F5 | õ | `@~o` | o-tilde |
| 208 | U+00C3 | Ã | `@~A` | A-tilde |
| 209 | U+00D1 | Ñ | `@~N` | N-tilde |
| 210 | U+00D5 | Õ | `@~O` | O-tilde |
| 211 | U+00E6 | æ | `@ae` | ae-ligature |
| 212 | U+00C6 | Æ | `@AE` | AE-ligature |
| 213 | U+00E7 | ç | `@cc` | c-cedilla |
| 214 | U+00C7 | Ç | `@cC` | C-cedilla |
| 215 | U+00FE | þ | `@th` | thorn |
| 216 | U+00F0 | ð | `@et` | eth |
| 217 | U+00DE | Þ | `@Th` | Thorn |
| 218 | U+00D0 | Ð | `@Et` | Eth |
| 219 | U+00A3 | £ | `@LL` | pound sign |
| 220 | U+0153 | œ | `@oe` | oe-ligature |
| 221 | U+0152 | Œ | `@OE` | OE-ligature |
| 222 | U+00A1 | ¡ | `@!!` | inverted exclamation mark |
| 223 | U+00BF | ¿ | `@??` | inverted question mark |

---

## §J.6 Accent Escape Codes

The accent escape codes define pairs of characters that form escape codes when
prefixed with `@`. Each pair is looked up to return the corresponding Unicode
code point from the default ZSCII-to-Unicode table.

### Diaeresis (`:` prefix)

| Escape | Unicode | Char | Description |
|--------|---------|------|-------------|
| `@:a` | U+00E4 | ä | a-diaeresis |
| `@:o` | U+00F6 | ö | o-diaeresis |
| `@:u` | U+00FC | ü | u-diaeresis |
| `@:A` | U+00C4 | Ä | A-diaeresis |
| `@:O` | U+00D6 | Ö | O-diaeresis |
| `@:U` | U+00DC | Ü | U-diaeresis |
| `@:e` | U+00EB | ë | e-diaeresis |
| `@:i` | U+00EF | ï | i-diaeresis |
| `@:y` | U+00FF | ÿ | y-diaeresis |
| `@:E` | U+00CB | Ë | E-diaeresis |
| `@:I` | U+00CF | Ï | I-diaeresis |

### Acute (`'` prefix)

| Escape | Unicode | Char | Description |
|--------|---------|------|-------------|
| `@'a` | U+00E1 | á | a-acute |
| `@'e` | U+00E9 | é | e-acute |
| `@'i` | U+00ED | í | i-acute |
| `@'o` | U+00F3 | ó | o-acute |
| `@'u` | U+00FA | ú | u-acute |
| `@'y` | U+00FD | ý | y-acute |
| `@'A` | U+00C1 | Á | A-acute |
| `@'E` | U+00C9 | É | E-acute |
| `@'I` | U+00CD | Í | I-acute |
| `@'O` | U+00D3 | Ó | O-acute |
| `@'U` | U+00DA | Ú | U-acute |
| `@'Y` | U+00DD | Ý | Y-acute |

### Grave (`` ` `` prefix)

| Escape | Unicode | Char | Description |
|--------|---------|------|-------------|
| `` @`a `` | U+00E0 | à | a-grave |
| `` @`e `` | U+00E8 | è | e-grave |
| `` @`i `` | U+00EC | ì | i-grave |
| `` @`o `` | U+00F2 | ò | o-grave |
| `` @`u `` | U+00F9 | ù | u-grave |
| `` @`A `` | U+00C0 | À | A-grave |
| `` @`E `` | U+00C8 | È | E-grave |
| `` @`I `` | U+00CC | Ì | I-grave |
| `` @`O `` | U+00D2 | Ò | O-grave |
| `` @`U `` | U+00D9 | Ù | U-grave |

### Circumflex (`^` prefix)

| Escape | Unicode | Char | Description |
|--------|---------|------|-------------|
| `@^a` | U+00E2 | â | a-circumflex |
| `@^e` | U+00EA | ê | e-circumflex |
| `@^i` | U+00EE | î | i-circumflex |
| `@^o` | U+00F4 | ô | o-circumflex |
| `@^u` | U+00FB | û | u-circumflex |
| `@^A` | U+00C2 | Â | A-circumflex |
| `@^E` | U+00CA | Ê | E-circumflex |
| `@^I` | U+00CE | Î | I-circumflex |
| `@^O` | U+00D4 | Ô | O-circumflex |
| `@^U` | U+00DB | Û | U-circumflex |

### Ring (`o` prefix)

| Escape | Unicode | Char | Description |
|--------|---------|------|-------------|
| `@oa` | U+00E5 | å | a-ring |
| `@oA` | U+00C5 | Å | A-ring |

### Slash (`/` prefix)

| Escape | Unicode | Char | Description |
|--------|---------|------|-------------|
| `@/o` | U+00F8 | ø | o-slash |
| `@/O` | U+00D8 | Ø | O-slash |

### Tilde (`~` prefix)

| Escape | Unicode | Char | Description |
|--------|---------|------|-------------|
| `@~a` | U+00E3 | ã | a-tilde |
| `@~n` | U+00F1 | ñ | n-tilde |
| `@~o` | U+00F5 | õ | o-tilde |
| `@~A` | U+00C3 | Ã | A-tilde |
| `@~N` | U+00D1 | Ñ | N-tilde |
| `@~O` | U+00D5 | Õ | O-tilde |

### Ligatures

| Escape | Unicode | Char | Description |
|--------|---------|------|-------------|
| `@ae` | U+00E6 | æ | ae-ligature |
| `@AE` | U+00C6 | Æ | AE-ligature |
| `@oe` | U+0153 | œ | oe-ligature |
| `@OE` | U+0152 | Œ | OE-ligature |
| `@ss` | U+00DF | ß | sz-ligature |

### Cedilla

| Escape | Unicode | Char | Description |
|--------|---------|------|-------------|
| `@cc` | U+00E7 | ç | c-cedilla |
| `@cC` | U+00C7 | Ç | C-cedilla |

### Icelandic

| Escape | Unicode | Char | Description |
|--------|---------|------|-------------|
| `@th` | U+00FE | þ | thorn |
| `@et` | U+00F0 | ð | eth |
| `@Th` | U+00DE | Þ | Thorn |
| `@Et` | U+00D0 | Ð | Eth |

### Punctuation

| Escape | Unicode | Char | Description |
|--------|---------|------|-------------|
| `@LL` | U+00A3 | £ | pound sign |
| `@!!` | U+00A1 | ¡ | inverted exclamation mark |
| `@??` | U+00BF | ¿ | inverted question mark |

### Guillemets

| Escape | Unicode | Char | Description |
|--------|---------|------|-------------|
| `@>>` | U+00BB | » | right guillemet |
| `@<<` | U+00AB | « | left guillemet |

---

## §J.7 Alternative Character Set Tables

The `-C` switch selects different source character sets, each with its own
ZSCII-to-Unicode translation table. The following table summarises the
available character sets:

| Index | `-C` Switch | Character Set | ISO Standard | Entries |
|-------|-------------|---------------|--------------|---------|
| 0 | `-C0` | Plain ASCII | — | 69 |
| 1 | `-C1` | Latin-1 | ISO 8859-1 | 69 |
| 2 | `-C2` | Latin-2 | ISO 8859-2 | 81 |
| 3 | `-C3` | Latin-3 | ISO 8859-3 | 71 |
| 4 | `-C4` | Latin-4 | ISO 8859-4 | 82 |
| 5 | `-C5` | Cyrillic | ISO 8859-5 | 92 |
| 6 | `-C6` | Arabic | ISO 8859-6 | 48 |
| 7 | `-C7` | Greek | ISO 8859-7 | 71 |
| 8 | `-C8` | Hebrew | ISO 8859-8 | 27 |
| 9 | `-C9` | Latin-5 | ISO 8859-9 | 62 |

The `-C0` and `-C1` tables are identical and are documented in §J.5 above. The
following subsections provide the complete translation tables for each
remaining character set.

### §J.7.1 ISO 8859-2 (Latin-2) — 81 entries

| ZSCII | Unicode | Char | Name |
|-------|---------|------|------|
| 155 | U+0104 | Ą | A-ogonek |
| 156 | U+0141 | Ł | L-stroke |
| 157 | U+013D | Ľ | L-caron |
| 158 | U+015A | Ś | S-acute |
| 159 | U+0160 | Š | S-caron |
| 160 | U+015E | Ş | S-cedilla |
| 161 | U+0164 | Ť | T-caron |
| 162 | U+0179 | Ź | Z-acute |
| 163 | U+017D | Ž | Z-caron |
| 164 | U+017B | Ż | Z-dot-above |
| 165 | U+0154 | Ŕ | R-acute |
| 166 | U+00C1 | Á | A-acute |
| 167 | U+00C2 | Â | A-circumflex |
| 168 | U+0102 | Ă | A-breve |
| 169 | U+00C4 | Ä | A-diaeresis |
| 170 | U+0139 | Ĺ | L-acute |
| 171 | U+0106 | Ć | C-acute |
| 172 | U+00C7 | Ç | C-cedilla |
| 173 | U+010C | Č | C-caron |
| 174 | U+00C9 | É | E-acute |
| 175 | U+0118 | Ę | E-ogonek |
| 176 | U+00CB | Ë | E-diaeresis |
| 177 | U+011A | Ě | E-caron |
| 178 | U+00CD | Í | I-acute |
| 179 | U+00CE | Î | I-circumflex |
| 180 | U+010E | Ď | D-caron |
| 181 | U+0110 | Đ | D-stroke |
| 182 | U+0143 | Ń | N-acute |
| 183 | U+0147 | Ň | N-caron |
| 184 | U+00D3 | Ó | O-acute |
| 185 | U+00D4 | Ô | O-circumflex |
| 186 | U+0150 | Ő | O-double-acute |
| 187 | U+00D6 | Ö | O-diaeresis |
| 188 | U+0158 | Ř | R-caron |
| 189 | U+016E | Ů | U-ring |
| 190 | U+00DA | Ú | U-acute |
| 191 | U+0170 | Ű | U-double-acute |
| 192 | U+00DC | Ü | U-diaeresis |
| 193 | U+00DD | Ý | Y-acute |
| 194 | U+0162 | Ţ | T-cedilla |
| 195 | U+0105 | ą | a-ogonek |
| 196 | U+0142 | ł | l-stroke |
| 197 | U+013E | ľ | l-caron |
| 198 | U+015B | ś | s-acute |
| 199 | U+0161 | š | s-caron |
| 200 | U+015F | ş | s-cedilla |
| 201 | U+0165 | ť | t-caron |
| 202 | U+017A | ź | z-acute |
| 203 | U+017E | ž | z-caron |
| 204 | U+017C | ż | z-dot-above |
| 205 | U+00DF | ß | sz-ligature |
| 206 | U+0155 | ŕ | r-acute |
| 207 | U+00E1 | á | a-acute |
| 208 | U+00E2 | â | a-circumflex |
| 209 | U+0103 | ă | a-breve |
| 210 | U+00E4 | ä | a-diaeresis |
| 211 | U+013A | ĺ | l-acute |
| 212 | U+0107 | ć | c-acute |
| 213 | U+00E7 | ç | c-cedilla |
| 214 | U+010D | č | c-caron |
| 215 | U+00E9 | é | e-acute |
| 216 | U+0119 | ę | e-ogonek |
| 217 | U+00EB | ë | e-diaeresis |
| 218 | U+011B | ě | e-caron |
| 219 | U+00ED | í | i-acute |
| 220 | U+00EE | î | i-circumflex |
| 221 | U+010F | ď | d-caron |
| 222 | U+0111 | đ | d-stroke |
| 223 | U+0144 | ń | n-acute |
| 224 | U+0148 | ň | n-caron |
| 225 | U+00F3 | ó | o-acute |
| 226 | U+00F4 | ô | o-circumflex |
| 227 | U+0151 | ő | o-double-acute |
| 228 | U+00F6 | ö | o-diaeresis |
| 229 | U+0159 | ř | r-caron |
| 230 | U+016F | ů | u-ring |
| 231 | U+00FA | ú | u-acute |
| 232 | U+0171 | ű | u-double-acute |
| 233 | U+00FC | ü | u-diaeresis |
| 234 | U+00FD | ý | y-acute |
| 235 | U+0163 | ţ | t-cedilla |

### §J.7.2 ISO 8859-3 (Latin-3) — 71 entries

| ZSCII | Unicode | Char | Name |
|-------|---------|------|------|
| 155 | U+0126 | Ħ | H-stroke |
| 156 | U+0124 | Ĥ | H-circumflex |
| 157 | U+0130 | İ | I-dot-above |
| 158 | U+015E | Ş | S-cedilla |
| 159 | U+011E | Ğ | G-breve |
| 160 | U+0134 | Ĵ | J-circumflex |
| 161 | U+017B | Ż | Z-dot-above |
| 162 | U+0127 | ħ | h-stroke |
| 163 | U+0125 | ĥ | h-circumflex |
| 164 | U+0131 | ı | dotless-i |
| 165 | U+015F | ş | s-cedilla |
| 166 | U+011F | ğ | g-breve |
| 167 | U+0135 | ĵ | j-circumflex |
| 168 | U+017C | ż | z-dot-above |
| 169 | U+00C0 | À | A-grave |
| 170 | U+00C1 | Á | A-acute |
| 171 | U+00C2 | Â | A-circumflex |
| 172 | U+00C4 | Ä | A-diaeresis |
| 173 | U+010A | Ċ | C-dot-above |
| 174 | U+0108 | Ĉ | C-circumflex |
| 175 | U+00C7 | Ç | C-cedilla |
| 176 | U+00C8 | È | E-grave |
| 177 | U+00C9 | É | E-acute |
| 178 | U+00CA | Ê | E-circumflex |
| 179 | U+00CB | Ë | E-diaeresis |
| 180 | U+00CC | Ì | I-grave |
| 181 | U+00CD | Í | I-acute |
| 182 | U+00CE | Î | I-circumflex |
| 183 | U+00CF | Ï | I-diaeresis |
| 184 | U+00D1 | Ñ | N-tilde |
| 185 | U+00D2 | Ò | O-grave |
| 186 | U+00D3 | Ó | O-acute |
| 187 | U+00D4 | Ô | O-circumflex |
| 188 | U+0120 | Ġ | G-dot-above |
| 189 | U+00D6 | Ö | O-diaeresis |
| 190 | U+011C | Ĝ | G-circumflex |
| 191 | U+00D9 | Ù | U-grave |
| 192 | U+00DA | Ú | U-acute |
| 193 | U+00DB | Û | U-circumflex |
| 194 | U+00DC | Ü | U-diaeresis |
| 195 | U+016C | Ŭ | U-breve |
| 196 | U+015C | Ŝ | S-circumflex |
| 197 | U+00DF | ß | sz-ligature |
| 198 | U+00E0 | à | a-grave |
| 199 | U+00E1 | á | a-acute |
| 200 | U+00E2 | â | a-circumflex |
| 201 | U+00E4 | ä | a-diaeresis |
| 202 | U+010B | ċ | c-dot-above |
| 203 | U+0109 | ĉ | c-circumflex |
| 204 | U+00E7 | ç | c-cedilla |
| 205 | U+00E8 | è | e-grave |
| 206 | U+00E9 | é | e-acute |
| 207 | U+00EA | ê | e-circumflex |
| 208 | U+00EB | ë | e-diaeresis |
| 209 | U+00EC | ì | i-grave |
| 210 | U+00ED | í | i-acute |
| 211 | U+00EE | î | i-circumflex |
| 212 | U+00EF | ï | i-diaeresis |
| 213 | U+00F1 | ñ | n-tilde |
| 214 | U+00F2 | ò | o-grave |
| 215 | U+00F3 | ó | o-acute |
| 216 | U+00F4 | ô | o-circumflex |
| 217 | U+0121 | ġ | g-dot-above |
| 218 | U+00F6 | ö | o-diaeresis |
| 219 | U+011D | ĝ | g-circumflex |
| 220 | U+00F9 | ù | u-grave |
| 221 | U+00FA | ú | u-acute |
| 222 | U+00FB | û | u-circumflex |
| 223 | U+00FC | ü | u-diaeresis |
| 224 | U+016D | ŭ | u-breve |
| 225 | U+015D | ŝ | s-circumflex |

### §J.7.3 ISO 8859-4 (Latin-4) — 82 entries

| ZSCII | Unicode | Char | Name |
|-------|---------|------|------|
| 155 | U+0104 | Ą | A-ogonek |
| 156 | U+0138 | ĸ | kra |
| 157 | U+0156 | Ŗ | R-cedilla |
| 158 | U+0128 | Ĩ | I-tilde |
| 159 | U+013B | Ļ | L-cedilla |
| 160 | U+0160 | Š | S-caron |
| 161 | U+0112 | Ē | E-macron |
| 162 | U+0122 | Ģ | G-cedilla |
| 163 | U+0166 | Ŧ | T-stroke |
| 164 | U+017D | Ž | Z-caron |
| 165 | U+0105 | ą | a-ogonek |
| 166 | U+0157 | ŗ | r-cedilla |
| 167 | U+0129 | ĩ | i-tilde |
| 168 | U+013C | ļ | l-cedilla |
| 169 | U+0161 | š | s-caron |
| 170 | U+0113 | ē | e-macron |
| 171 | U+0123 | ģ | g-cedilla |
| 172 | U+0167 | ŧ | t-stroke |
| 173 | U+014A | Ŋ | Eng |
| 174 | U+017E | ž | z-caron |
| 175 | U+014B | ŋ | eng |
| 176 | U+0100 | Ā | A-macron |
| 177 | U+00C1 | Á | A-acute |
| 178 | U+00C2 | Â | A-circumflex |
| 179 | U+00C3 | Ã | A-tilde |
| 180 | U+00C4 | Ä | A-diaeresis |
| 181 | U+00C5 | Å | A-ring |
| 182 | U+00C6 | Æ | AE-ligature |
| 183 | U+012E | Į | I-ogonek |
| 184 | U+010C | Č | C-caron |
| 185 | U+00C9 | É | E-acute |
| 186 | U+0118 | Ę | E-ogonek |
| 187 | U+00CB | Ë | E-diaeresis |
| 188 | U+0116 | Ė | E-dot-above |
| 189 | U+00CD | Í | I-acute |
| 190 | U+00CE | Î | I-circumflex |
| 191 | U+012A | Ī | I-macron |
| 192 | U+0110 | Đ | D-stroke |
| 193 | U+0145 | Ņ | N-cedilla |
| 194 | U+014C | Ō | O-macron |
| 195 | U+0136 | Ķ | K-cedilla |
| 196 | U+00D4 | Ô | O-circumflex |
| 197 | U+00D5 | Õ | O-tilde |
| 198 | U+00D6 | Ö | O-diaeresis |
| 199 | U+00D8 | Ø | O-slash |
| 200 | U+0172 | Ų | U-ogonek |
| 201 | U+00DA | Ú | U-acute |
| 202 | U+00DB | Û | U-circumflex |
| 203 | U+00DC | Ü | U-diaeresis |
| 204 | U+0168 | Ũ | U-tilde |
| 205 | U+016A | Ū | U-macron |
| 206 | U+00DF | ß | sz-ligature |
| 207 | U+0101 | ā | a-macron |
| 208 | U+00E1 | á | a-acute |
| 209 | U+00E2 | â | a-circumflex |
| 210 | U+00E3 | ã | a-tilde |
| 211 | U+00E4 | ä | a-diaeresis |
| 212 | U+00E5 | å | a-ring |
| 213 | U+00E6 | æ | ae-ligature |
| 214 | U+012F | į | i-ogonek |
| 215 | U+010D | č | c-caron |
| 216 | U+00E9 | é | e-acute |
| 217 | U+0119 | ę | e-ogonek |
| 218 | U+00EB | ë | e-diaeresis |
| 219 | U+0117 | ė | e-dot-above |
| 220 | U+00ED | í | i-acute |
| 221 | U+00EE | î | i-circumflex |
| 222 | U+012B | ī | i-macron |
| 223 | U+0111 | đ | d-stroke |
| 224 | U+0146 | ņ | n-cedilla |
| 225 | U+014D | ō | o-macron |
| 226 | U+0137 | ķ | k-cedilla |
| 227 | U+00F4 | ô | o-circumflex |
| 228 | U+00F5 | õ | o-tilde |
| 229 | U+00F6 | ö | o-diaeresis |
| 230 | U+00F8 | ø | o-slash |
| 231 | U+0173 | ų | u-ogonek |
| 232 | U+00FA | ú | u-acute |
| 233 | U+00FB | û | u-circumflex |
| 234 | U+00FC | ü | u-diaeresis |
| 235 | U+0169 | ũ | u-tilde |
| 236 | U+016B | ū | u-macron |

### §J.7.4 ISO 8859-5 (Cyrillic) — 92 entries

| ZSCII | Unicode | Char | Name |
|-------|---------|------|------|
| 155 | U+0401 | Ё | Io |
| 156 | U+0402 | Ђ | Dje |
| 157 | U+0403 | Ѓ | Gje |
| 158 | U+0404 | Є | Ukrainian Ie |
| 159 | U+0405 | Ѕ | Dze |
| 160 | U+0406 | І | Byelorussian-Ukrainian I |
| 161 | U+0407 | Ї | Yi |
| 162 | U+0408 | Ј | Je |
| 163 | U+0409 | Љ | Lje |
| 164 | U+040A | Њ | Nje |
| 165 | U+040B | Ћ | Tshe |
| 166 | U+040C | Ќ | Kje |
| 167 | U+040E | Ў | Short U |
| 168 | U+040F | Џ | Dzhe |
| 169 | U+0410 | А | A |
| 170 | U+0411 | Б | Be |
| 171 | U+0412 | В | Ve |
| 172 | U+0413 | Г | Ghe |
| 173 | U+0414 | Д | De |
| 174 | U+0415 | Е | Ie |
| 175 | U+0416 | Ж | Zhe |
| 176 | U+0417 | З | Ze |
| 177 | U+0418 | И | I |
| 178 | U+0419 | Й | Short I |
| 179 | U+041A | К | Ka |
| 180 | U+041B | Л | El |
| 181 | U+041C | М | Em |
| 182 | U+041D | Н | En |
| 183 | U+041E | О | O |
| 184 | U+041F | П | Pe |
| 185 | U+0420 | Р | Er |
| 186 | U+0421 | С | Es |
| 187 | U+0422 | Т | Te |
| 188 | U+0423 | У | U |
| 189 | U+0424 | Ф | Ef |
| 190 | U+0425 | Х | Kha |
| 191 | U+0426 | Ц | Tse |
| 192 | U+0427 | Ч | Che |
| 193 | U+0428 | Ш | Sha |
| 194 | U+0429 | Щ | Shcha |
| 195 | U+042A | Ъ | Hard Sign |
| 196 | U+042B | Ы | Yeru |
| 197 | U+042C | Ь | Soft Sign |
| 198 | U+042D | Э | E |
| 199 | U+042E | Ю | Yu |
| 200 | U+042F | Я | Ya |
| 201 | U+0430 | а | a |
| 202 | U+0431 | б | be |
| 203 | U+0432 | в | ve |
| 204 | U+0433 | г | ghe |
| 205 | U+0434 | д | de |
| 206 | U+0435 | е | ie |
| 207 | U+0436 | ж | zhe |
| 208 | U+0437 | з | ze |
| 209 | U+0438 | и | i |
| 210 | U+0439 | й | short i |
| 211 | U+043A | к | ka |
| 212 | U+043B | л | el |
| 213 | U+043C | м | em |
| 214 | U+043D | н | en |
| 215 | U+043E | о | o |
| 216 | U+043F | п | pe |
| 217 | U+0440 | р | er |
| 218 | U+0441 | с | es |
| 219 | U+0442 | т | te |
| 220 | U+0443 | у | u |
| 221 | U+0444 | ф | ef |
| 222 | U+0445 | х | kha |
| 223 | U+0446 | ц | tse |
| 224 | U+0447 | ч | che |
| 225 | U+0448 | ш | sha |
| 226 | U+0449 | щ | shcha |
| 227 | U+044A | ъ | hard sign |
| 228 | U+044B | ы | yeru |
| 229 | U+044C | ь | soft sign |
| 230 | U+044D | э | e |
| 231 | U+044E | ю | yu |
| 232 | U+044F | я | ya |
| 233 | U+0451 | ё | io |
| 234 | U+0452 | ђ | dje |
| 235 | U+0453 | ѓ | gje |
| 236 | U+0454 | є | Ukrainian ie |
| 237 | U+0455 | ѕ | dze |
| 238 | U+0456 | і | Byelorussian-Ukrainian i |
| 239 | U+0457 | ї | yi |
| 240 | U+0458 | ј | je |
| 241 | U+0459 | љ | lje |
| 242 | U+045A | њ | nje |
| 243 | U+045B | ћ | tshe |
| 244 | U+045C | ќ | kje |
| 245 | U+045E | ў | short u |
| 246 | U+045F | џ | dzhe |

### §J.7.5 ISO 8859-6 (Arabic) — 48 entries

| ZSCII | Unicode | Char | Name |
|-------|---------|------|------|
| 155 | U+060C | ، | Arabic comma |
| 156 | U+061B | ؛ | Arabic semicolon |
| 157 | U+061F | ؟ | Arabic question mark |
| 158 | U+0621 | ء | Hamza |
| 159 | U+0622 | آ | Alef with madda above |
| 160 | U+0623 | أ | Alef with hamza above |
| 161 | U+0624 | ؤ | Waw with hamza above |
| 162 | U+0625 | إ | Alef with hamza below |
| 163 | U+0626 | ئ | Yeh with hamza above |
| 164 | U+0627 | ا | Alef |
| 165 | U+0628 | ب | Beh |
| 166 | U+0629 | ة | Teh marbuta |
| 167 | U+062A | ت | Teh |
| 168 | U+062B | ث | Theh |
| 169 | U+062C | ج | Jeem |
| 170 | U+062D | ح | Hah |
| 171 | U+062E | خ | Khah |
| 172 | U+062F | د | Dal |
| 173 | U+0630 | ذ | Thal |
| 174 | U+0631 | ر | Reh |
| 175 | U+0632 | ز | Zain |
| 176 | U+0633 | س | Seen |
| 177 | U+0634 | ش | Sheen |
| 178 | U+0635 | ص | Sad |
| 179 | U+0636 | ض | Dad |
| 180 | U+0637 | ط | Tah |
| 181 | U+0638 | ظ | Zah |
| 182 | U+0639 | ع | Ain |
| 183 | U+063A | غ | Ghain |
| 184 | U+0640 | ـ | Tatweel |
| 185 | U+0641 | ف | Feh |
| 186 | U+0642 | ق | Qaf |
| 187 | U+0643 | ك | Kaf |
| 188 | U+0644 | ل | Lam |
| 189 | U+0645 | م | Meem |
| 190 | U+0646 | ن | Noon |
| 191 | U+0647 | ه | Heh |
| 192 | U+0648 | و | Waw |
| 193 | U+0649 | ى | Alef maksura |
| 194 | U+064A | ي | Yeh |
| 195 | U+064B | ً | Fathatan |
| 196 | U+064C | ٌ | Dammatan |
| 197 | U+064D | ٍ | Kasratan |
| 198 | U+064E | َ | Fathah |
| 199 | U+064F | ُ | Dammah |
| 200 | U+0650 | ِ | Kasrah |
| 201 | U+0651 | ّ | Shaddah |
| 202 | U+0652 | ْ | Sukun |

### §J.7.6 ISO 8859-7 (Greek) — 71 entries

| ZSCII | Unicode | Char | Name |
|-------|---------|------|------|
| 155 | U+0384 | ΄ | Tonos |
| 156 | U+0385 | ΅ | Dialytika tonos |
| 157 | U+0386 | Ά | Alpha with tonos |
| 158 | U+0388 | Έ | Epsilon with tonos |
| 159 | U+0389 | Ή | Eta with tonos |
| 160 | U+038A | Ί | Iota with tonos |
| 161 | U+038C | Ό | Omicron with tonos |
| 162 | U+038E | Ύ | Upsilon with tonos |
| 163 | U+038F | Ώ | Omega with tonos |
| 164 | U+0390 | ΐ | Iota with dialytika and tonos |
| 165 | U+0391 | Α | Alpha |
| 166 | U+0392 | Β | Beta |
| 167 | U+0393 | Γ | Gamma |
| 168 | U+0394 | Δ | Delta |
| 169 | U+0395 | Ε | Epsilon |
| 170 | U+0396 | Ζ | Zeta |
| 171 | U+0397 | Η | Eta |
| 172 | U+0398 | Θ | Theta |
| 173 | U+0399 | Ι | Iota |
| 174 | U+039A | Κ | Kappa |
| 175 | U+039B | Λ | Lambda |
| 176 | U+039C | Μ | Mu |
| 177 | U+039D | Ν | Nu |
| 178 | U+039E | Ξ | Xi |
| 179 | U+039F | Ο | Omicron |
| 180 | U+03A0 | Π | Pi |
| 181 | U+03A1 | Ρ | Rho |
| 182 | U+03A3 | Σ | Sigma |
| 183 | U+03A4 | Τ | Tau |
| 184 | U+03A5 | Υ | Upsilon |
| 185 | U+03A6 | Φ | Phi |
| 186 | U+03A7 | Χ | Chi |
| 187 | U+03A8 | Ψ | Psi |
| 188 | U+03A9 | Ω | Omega |
| 189 | U+03AA | Ϊ | Iota with dialytika |
| 190 | U+03AB | Ϋ | Upsilon with dialytika |
| 191 | U+03AC | ά | Alpha with tonos |
| 192 | U+03AD | έ | Epsilon with tonos |
| 193 | U+03AE | ή | Eta with tonos |
| 194 | U+03AF | ί | Iota with tonos |
| 195 | U+03B0 | ΰ | Upsilon with dialytika and tonos |
| 196 | U+03B1 | α | Alpha |
| 197 | U+03B2 | β | Beta |
| 198 | U+03B3 | γ | Gamma |
| 199 | U+03B4 | δ | Delta |
| 200 | U+03B5 | ε | Epsilon |
| 201 | U+03B6 | ζ | Zeta |
| 202 | U+03B7 | η | Eta |
| 203 | U+03B8 | θ | Theta |
| 204 | U+03B9 | ι | Iota |
| 205 | U+03BA | κ | Kappa |
| 206 | U+03BB | λ | Lambda |
| 207 | U+03BC | μ | Mu |
| 208 | U+03BD | ν | Nu |
| 209 | U+03BE | ξ | Xi |
| 210 | U+03BF | ο | Omicron |
| 211 | U+03C0 | π | Pi |
| 212 | U+03C1 | ρ | Rho |
| 213 | U+03C2 | ς | Final sigma |
| 214 | U+03C3 | σ | Sigma |
| 215 | U+03C4 | τ | Tau |
| 216 | U+03C5 | υ | Upsilon |
| 217 | U+03C6 | φ | Phi |
| 218 | U+03C7 | χ | Chi |
| 219 | U+03C8 | ψ | Psi |
| 220 | U+03C9 | ω | Omega |
| 221 | U+03CA | ϊ | Iota with dialytika |
| 222 | U+03CB | ϋ | Upsilon with dialytika |
| 223 | U+03CC | ό | Omicron with tonos |
| 224 | U+03CD | ύ | Upsilon with tonos |
| 225 | U+03CE | ώ | Omega with tonos |

### §J.7.7 ISO 8859-8 (Hebrew) — 27 entries

| ZSCII | Unicode | Char | Name |
|-------|---------|------|------|
| 155 | U+05D0 | א | Alef |
| 156 | U+05D1 | ב | Bet |
| 157 | U+05D2 | ג | Gimel |
| 158 | U+05D3 | ד | Dalet |
| 159 | U+05D4 | ה | He |
| 160 | U+05D5 | ו | Vav |
| 161 | U+05D6 | ז | Zayin |
| 162 | U+05D7 | ח | Het |
| 163 | U+05D8 | ט | Tet |
| 164 | U+05D9 | י | Yod |
| 165 | U+05DA | ך | Final kaf |
| 166 | U+05DB | כ | Kaf |
| 167 | U+05DC | ל | Lamed |
| 168 | U+05DD | ם | Final mem |
| 169 | U+05DE | מ | Mem |
| 170 | U+05DF | ן | Final nun |
| 171 | U+05E0 | נ | Nun |
| 172 | U+05E1 | ס | Samekh |
| 173 | U+05E2 | ע | Ayin |
| 174 | U+05E3 | ף | Final pe |
| 175 | U+05E4 | פ | Pe |
| 176 | U+05E5 | ץ | Final tsadi |
| 177 | U+05E6 | צ | Tsadi |
| 178 | U+05E7 | ק | Qof |
| 179 | U+05E8 | ר | Resh |
| 180 | U+05E9 | ש | Shin |
| 181 | U+05EA | ת | Tav |

### §J.7.8 ISO 8859-9 (Latin-5, Turkish) — 62 entries

| ZSCII | Unicode | Char | Name |
|-------|---------|------|------|
| 155 | U+00C0 | À | A-grave |
| 156 | U+00C1 | Á | A-acute |
| 157 | U+00C2 | Â | A-circumflex |
| 158 | U+00C3 | Ã | A-tilde |
| 159 | U+00C4 | Ä | A-diaeresis |
| 160 | U+00C5 | Å | A-ring |
| 161 | U+00C6 | Æ | AE-ligature |
| 162 | U+00C7 | Ç | C-cedilla |
| 163 | U+00C8 | È | E-grave |
| 164 | U+00C9 | É | E-acute |
| 165 | U+00CA | Ê | E-circumflex |
| 166 | U+00CB | Ë | E-diaeresis |
| 167 | U+00CC | Ì | I-grave |
| 168 | U+00CD | Í | I-acute |
| 169 | U+00CE | Î | I-circumflex |
| 170 | U+00CF | Ï | I-diaeresis |
| 171 | U+011E | Ğ | G-breve |
| 172 | U+00D1 | Ñ | N-tilde |
| 173 | U+00D2 | Ò | O-grave |
| 174 | U+00D3 | Ó | O-acute |
| 175 | U+00D4 | Ô | O-circumflex |
| 176 | U+00D5 | Õ | O-tilde |
| 177 | U+00D6 | Ö | O-diaeresis |
| 178 | U+00D8 | Ø | O-slash |
| 179 | U+00D9 | Ù | U-grave |
| 180 | U+00DA | Ú | U-acute |
| 181 | U+00DB | Û | U-circumflex |
| 182 | U+00DC | Ü | U-diaeresis |
| 183 | U+0130 | İ | I-dot-above |
| 184 | U+015E | Ş | S-cedilla |
| 185 | U+00DF | ß | sz-ligature |
| 186 | U+00E0 | à | a-grave |
| 187 | U+00E1 | á | a-acute |
| 188 | U+00E2 | â | a-circumflex |
| 189 | U+00E3 | ã | a-tilde |
| 190 | U+00E4 | ä | a-diaeresis |
| 191 | U+00E5 | å | a-ring |
| 192 | U+00E6 | æ | ae-ligature |
| 193 | U+00E7 | ç | c-cedilla |
| 194 | U+00E8 | è | e-grave |
| 195 | U+00E9 | é | e-acute |
| 196 | U+00EA | ê | e-circumflex |
| 197 | U+00EB | ë | e-diaeresis |
| 198 | U+00EC | ì | i-grave |
| 199 | U+00ED | í | i-acute |
| 200 | U+00EE | î | i-circumflex |
| 201 | U+00EF | ï | i-diaeresis |
| 202 | U+011F | ğ | g-breve |
| 203 | U+00F1 | ñ | n-tilde |
| 204 | U+00F2 | ò | o-grave |
| 205 | U+00F3 | ó | o-acute |
| 206 | U+00F4 | ô | o-circumflex |
| 207 | U+00F5 | õ | o-tilde |
| 208 | U+00F6 | ö | o-diaeresis |
| 209 | U+00F8 | ø | o-slash |
| 210 | U+00F9 | ù | u-grave |
| 211 | U+00FA | ú | u-acute |
| 212 | U+00FB | û | u-circumflex |
| 213 | U+00FC | ü | u-diaeresis |
| 214 | U+0131 | ı | dotless-i |
| 215 | U+015F | ş | s-cedilla |
| 216 | U+00FF | ÿ | y-diaeresis |

---

## §J.8 Text Escape Sequences in Strings

Complete reference of all special characters and `@`-escape sequences
available in Inform 6 string literals.

| Syntax | Meaning |
|--------|---------|
| `^` | Newline (ZSCII 10) |
| `~` | Double-quote `"` (ZSCII 34) |
| `@@`*nnn* | Literal ZSCII value *nnn* (decimal, 0–1023) |
| `@`*xy* | Accent character (see §J.6) |
| `@{`*hex*`}` | Unicode character (1–6 hex digits) |
| `@`*nn* | Dynamic string variable (2 decimal digits, 00–99) |
| `@(`*id*`)` | Dynamic string variable by constant name |

### Examples

```inform6
print "caf@'e";              ! prints: café
print "Stra@sse";            ! prints: Straße  (using @ss)
print "Price: @{a3}5";       ! prints: Price: £5
print "line one^line two";   ! prints with newline between
print "He said ~hello~.";    ! prints: He said "hello".
print "literal caret: @@94"; ! prints: literal caret: ^
print "literal tilde: @@126";! prints: literal tilde: ~
```

### Notes on `@@nnn`

The `@@` escape inserts a literal ZSCII character by its decimal value. Two
values are special-cased by the compiler:

- **`@@94`** (caret `^`): The compiler uses a 4-Z-char escape sequence to
  prevent `^` from being interpreted as newline.
- **`@@126`** (tilde `~`): The compiler uses a 4-Z-char escape sequence to
  prevent `~` from being interpreted as double-quote.

---

## §J.9 The `Zcharacter` Directive

**[Z-machine]**

The `Zcharacter` directive controls Z-machine character encoding. It has four
forms.

### Form 1: Replace all three alphabets

```inform6
Zcharacter "abcdefghijklmnopqrstuvwxyz"
           "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
           "0123456789.,!?_#'~/@\-:()>";
```

- A0 and A1 must have exactly **26** characters each.
- A2 must have exactly **23** characters (positions 3–25; positions 0–2 are
  fixed: padding, newline, ZSCII escape).

### Form 2: Add a single character to the alphabet

```inform6
Zcharacter '@{e9}';
```

Attempts to place the character in an unused A2 slot (positions 2–25, excluding
positions 12, 13, and 19). The character must already exist in the ZSCII
translation table.

### Form 3: Replace the Unicode translation table

```inform6
Zcharacter table
    $e4 $f6 $fc $c4 $d6 $dc;
```

Clears the existing ZSCII-to-Unicode table and populates ZSCII 155+ with the
given Unicode values.

### Form 4: Append to the Unicode translation table

```inform6
Zcharacter table +
    $0153 $0152;
```

Appends to the existing table without clearing it.

### Limits

The maximum number of entries in the translation table is **97** (ZSCII
155–251). An error is reported if the table is full.

### Glulx restriction

This directive is not available when targeting Glulx. Attempting to use it
produces the error: *"The Zcharacter directive has no meaning in Glulx."*

---

## §J.10 Unicode Translation Table in the Story File

**[Z-machine]**

When the ZSCII-to-Unicode mapping has been modified (either by a non-default
character set via `-C` or by a `Zcharacter table` directive), the compiler
writes translation tables to the story file.

### Unicode translation table

The Unicode translation table has the following format:

- **Format**: 1 byte (count) followed by count × 2 bytes (big-endian Unicode
  values).
- **Written only** when `zscii_defn_modified` is true.
- Each entry maps ZSCII (155 + index) to a Unicode code point.
- Characters are limited to U+0000–U+FFFF.
- The table address is stored in header extension table word 3.

### Alphabet table

Written when `alphabet_modified` is true:

- **Format**: 78 bytes — 3 alphabets × 26 ZSCII values.
- During the write, tilde (`~`, value 0x7E) is replaced with double-quote
  (`"`, value 0x22).
- The table address is stored in the header at offset $34 (word address).

---

## §J.11 Library Case Conversion (ZSCII)

### §J.11.1 Z-machine Case Conversion

**[Z-machine]**

The standard library provides `UpperCase` and `LowerCase` routines
(`english.h` lines 764–786) for ZSCII case conversion matching the default
Latin-1 table.

**`UpperCase` mapping** (from `english.h` lines 776–784):

| Input ZSCII | Offset | Output ZSCII | Characters |
|-------------|--------|--------------|------------|
| 97–122 (`a`–`z`) | −32 | 65–90 (`A`–`Z`) | ASCII letters |
| 155–157 | +3 | 158–160 | äöü → ÄÖÜ |
| 164, 165 | +3 | 167, 168 | ëï → ËÏ |
| 169–174 | +6 | 175–180 | áéíóúý → ÁÉÍÓÚÝ |
| 181–185 | +5 | 186–190 | àèìòù → ÀÈÌÒÙ |
| 191–195 | +5 | 196–200 | âêîôû → ÂÊÎÔÛ |
| 201 | +1 | 202 | å → Å |
| 203 | +1 | 204 | ø → Ø |
| 205–207 | +3 | 208–210 | ãñõ → ÃÑÕ |
| 211 | +1 | 212 | æ → Æ |
| 213 | +1 | 214 | ç → Ç |
| 215, 216 | +2 | 217, 218 | þð → ÞÐ |
| 220 | +1 | 221 | œ → Œ |

**`LowerCase`** uses the inverse offsets (negating the offset values above).

### §J.11.2 Glulx Case Conversion

**[Glulx]**

On Glulx, case conversion delegates to the Glk API:

- `glk_char_to_lower(c)` — convert to lowercase
- `glk_char_to_upper(c)` — convert to uppercase

See `english.h` lines 790–791.

---

## §J.12 Glulx Character Handling Differences

**[Glulx]**

Key differences from Z-machine character handling:

1. **No `Zcharacter` directive**: Using `Zcharacter` produces an error.

2. **Dictionary encoding**: Words are stored as Unicode values directly, not
   Z-char sequences. Character size is controlled by `DICT_CHAR_SIZE`: 1 byte
   for Latin-1 or 4 bytes for full Unicode.

3. **String compression**: Uses Huffman encoding rather than Z-char alphabet
   encoding. Unicode characters above U+00FF use `@U` escape sequences with
   4 hex-encoded digits.

4. **Full Unicode via Glk**: Glulx interpreters provide Unicode I/O through
   the Glk API (`glk_put_char_uni()`, `glk_char_to_lower()`, etc.).

5. **Source encoding**: The `-Cu` switch enables UTF-8 source input for both
   Z-machine and Glulx targets.

---

## §J.13 Character Set Selection (`-C` Switch)

The `-C` compiler switch selects the source character set and the default
ZSCII-to-Unicode mapping:

| Switch | Character Set | ISO Standard | High ZSCII Entries |
|--------|--------------|--------------|-------------------|
| `-C0` | Plain ASCII | — | 69 |
| `-C1` | Latin-1 (default) | ISO 8859-1 | 69 |
| `-C2` | Latin-2 | ISO 8859-2 | 81 |
| `-C3` | Latin-3 | ISO 8859-3 | 71 |
| `-C4` | Latin-4 | ISO 8859-4 | 82 |
| `-C5` | Cyrillic | ISO 8859-5 | 92 |
| `-C6` | Arabic | ISO 8859-6 | 48 |
| `-C7` | Greek | ISO 8859-7 | 71 |
| `-C8` | Hebrew | ISO 8859-8 | 27 |
| `-C9` | Latin-5 (Turkish) | ISO 8859-9 | 62 |
| `-Cu` | UTF-8 source | Unicode | 69 (as Latin-1) |

The default is **`-C1`** (ISO 8859-1 / Latin-1).

**`-Cu`** enables `character_set_unicode` for UTF-8 source file parsing. The
ZSCII mapping table remains the same as `-C1` because the first 256 Unicode
code points coincide with ISO 8859-1.

The compiler maps character set names to their corresponding ISO standard
designations.
