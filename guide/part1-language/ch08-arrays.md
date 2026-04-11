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

# Chapter 8: Arrays

This chapter describes arrays in Inform 6: how they are declared, the five
array types and their distinct memory layouts, how entries are read and
written, and the idioms used to work with arrays in practice. The
information here is derived from the array parser in the compiler source
(`arrays.c`), the expression code generator (`expressc.c`), and the
I6-Addendum.

## 8.1 Array Declarations

An array is declared at the top level of a source file using the `Array`
directive. The general syntax is:

```
Array name [static] type initializer ;
```

where *name* is a new identifier, *type* is one of the five array type
specifiers (`->`, `-->`, `string`, `table`, or `buffer`), and
*initializer* provides the array's size or initial values (see §8.3).

```inform6
Array inventory_flags -> 20;
Array room_exits --> 50;
Array title string "Zork";
Array scores table 0 0 0 0 0;
Array input_buffer buffer 120;
```

The `Array` directive creates a global constant whose value is the byte
address of the start of the array's data area in memory. This address
can be stored in variables, passed to routines, and used with the `->` and
`-->` operators to access individual entries.

```inform6
[ PrintArrayAddress arr;
    print "Array starts at byte address ", arr, "^";
];

[ Main;
    PrintArrayAddress(inventory_flags);
];
```

Array names occupy the same namespace as other global identifiers. An array
name must not duplicate the name of any global variable, constant, routine,
object, or other array. The compiler reports an error if the name is already
defined.

## 8.2 Array Types

Inform 6 provides five array types, differing in entry size and whether
an automatic length field is stored at the front of the array.

### 8.2.1 Byte Arrays (`->`)

A **byte array** stores one byte (0–255) per entry. Entries are accessed
with the `->` operator:

```inform6
Array flags -> 10;            ! 10 bytes, all initialised to 0

[ Main;
    flags->0 = 1;              ! set first entry
    flags->9 = 255;            ! set last entry
    print flags->0, "^";      ! prints 1
];
```

Byte arrays have no overhead — the array identifier points directly to the
first data byte, and `N` entries occupy exactly `N` bytes.

### 8.2.2 Word Arrays (`-->`)

A **word array** stores one machine word per entry. The `-->` operator
accesses entries by word index:

```inform6
Array high_scores --> 5;       ! 5 words, all initialised to 0

[ Main;
    high_scores-->0 = 1000;
    high_scores-->4 = 500;
    print high_scores-->0, "^";  ! prints 1000
];
```

> **[Z-machine]** Each word is 2 bytes. A word array of `N` entries
> occupies `N * 2` bytes. Words are stored in big-endian order (most
> significant byte first).

> **[Glulx]** Each word is 4 bytes. A word array of `N` entries occupies
> `N * 4` bytes. Words are likewise stored in big-endian order.

### 8.2.3 String Arrays

A **string array** stores byte-sized entries with an automatic length byte
at position 0. The length byte records the number of data entries (not
including itself). Data entries begin at index 1:

```inform6
Array greeting string "Hello";

[ Main;
    print greeting->0, "^";     ! prints 5 (the length)
    print (char) greeting->1;   ! prints 'H'
    print (char) greeting->5;   ! prints 'o'
];
```

Because the length is stored in a single byte (0–255), a string array can
hold at most 255 data entries (indices 1–255). The compiler reports an
error if a string array would exceed this limit. Total memory consumed is
`1 + N` bytes, where `N` is the number of entries.

### 8.2.4 Table Arrays

A **table array** stores word-sized entries with an automatic word-length
count at position 0. The count records the number of data entries. Data
entries begin at index 1:

```inform6
Array primes table 2 3 5 7 11 13;

[ Main  i;
    print primes-->0, "^";      ! prints 6 (the count)
    for (i = 1 : i <= primes-->0 : i++)
        print primes-->i, " ";  ! prints 2 3 5 7 11 13
];
```

Table arrays have no practical limit on entry count (beyond available
memory). Total memory consumed is `(1 + N) * WORDSIZE` bytes.

### 8.2.5 Buffer Arrays

A **buffer array** is a hybrid type: it stores a word-sized length count
followed by byte-sized data entries. The length word occupies the first
`WORDSIZE` bytes (2 on the Z-machine, 4 on Glulx). Data bytes begin at
byte offset `WORDSIZE`:

```inform6
Array input_buffer buffer 120;

[ Main;
    print input_buffer-->0, "^";             ! prints 120 (the count)
    print input_buffer->(WORDSIZE), "^";     ! first data byte
];
```

Buffer arrays are designed for use with the `read` statement (see §8.8),
which expects exactly this layout: a word at the start giving the buffer
capacity, followed by byte-sized character storage.

Total memory consumed is `WORDSIZE + N` bytes, where `N` is the number of
data entries.

### 8.2.6 Summary of Array Types

| Type | Entry size | Length field | Data starts at | Access operator |
| ---- | ---------- | ----------- | -------------- | --------------- |
| `->` (byte) | 1 byte | None | index 0 | `arr->i` |
| `-->` (word) | 1 word | None | index 0 | `arr-->i` |
| `string` | 1 byte | 1 byte at `arr->0` | index 1 | `arr->i` |
| `table` | 1 word | 1 word at `arr-->0` | index 1 | `arr-->i` |
| `buffer` | 1 byte | 1 word at `arr-->0` | byte offset `WORDSIZE` | `arr->i` (for data) |

## 8.3 Initializers and Sizing

Every array declaration must specify either a size (number of entries) or a
list of initial values. The compiler supports four forms of initializer.

### 8.3.1 Numeric Size (Zero-Filled)

A single numeric expression after the type keyword creates an array of that
many entries, all initialised to zero:

```inform6
Array buffer -> 100;           ! 100 zero-initialised bytes
Array rooms --> 20;            ! 20 zero-initialised words
Array text_buf buffer 80;      ! 80-byte buffer with length word
```

The size expression must be a compile-time constant. The compiler reports
an error if the value is zero or negative.

> **[Z-machine]** The maximum array size is 32767 entries.

### 8.3.2 Value List

A sequence of two or more expressions after the type keyword creates an
array with one entry per expression, initialised to the given values:

```inform6
Array directions --> 1 2 3 4 5 6 7 8;
Array vowels -> 'a' 'e' 'i' 'o' 'u';
```

Each expression is evaluated as a compile-time constant. Constants,
literal values, previously defined symbols, and expressions involving
these are all permitted:

```inform6
Constant BASE = 100;
Array thresholds --> BASE BASE*2 BASE*3 BASE+50;
```

For byte arrays (`->`, `string`, `buffer`), each value must fit in the
range 0–255. If a value exceeds this range, the compiler issues a warning
and truncates the value to its low byte.

### 8.3.3 String Initializer

A double-quoted string after the type keyword initialises the array from
the character codes of the string:

```inform6
Array message -> "Hello, world!";
Array encoded string "xyzzy";
```

Each character in the string becomes one array entry. On the Z-machine,
characters are converted to ZSCII values. On Glulx, characters are stored
as Unicode code points. Characters outside the representable range produce
a compiler error.

String initialisers are valid for all five array types, though they are
most natural with byte arrays and string arrays.

### 8.3.4 Bracketed Value List

A list of expressions enclosed in square brackets provides the same
semantics as an unbracketed value list, but allows the values to span
multiple source lines. Semicolons between entries are optional:

```inform6
Array colour_table -->
    [   $FF0000     ! red
        $00FF00     ! green
        $0000FF     ! blue
        $FFFF00     ! yellow
    ];
```

The bracket form is also the only way to declare a one-entry array with
an initial value, since a single expression without brackets is
interpreted as a size:

```inform6
Array single --> [ 42 ];       ! one-entry array holding 42
Array forty_two --> 42;        ! 42-entry array of zeroes (NOT the same)
```

Zero-entry arrays are never permitted; the compiler requires at least one
entry regardless of the initializer form.

## 8.4 Static Arrays

By default, arrays are placed in writable (dynamic) memory and can be
modified at runtime. The `static` keyword places an array in read-only
memory instead:

```inform6
Array sine_table static --> 0 174 342 500 643 766 866 940 985 1000;
Array help_text static string "Type HELP for instructions.";
```

The `static` keyword must appear between the array name and the type
keyword. A static array should always provide initial values, since its
contents cannot be changed after compilation.

```inform6
Array lookup static -> 10;     ! legal but pointless: 10 zeroes, read-only
```

> **[Z-machine]** Static arrays are placed in readable memory above the
> static memory boundary (after the dictionary, before code storage). They
> can be read at runtime but writes are undefined — the virtual machine may
> silently ignore the write or may halt with an error, depending on the
> interpreter.

> **[Glulx]** Static arrays are placed in the ROM area (after string
> storage). Attempting to write to ROM is a memory access violation and
> will produce a runtime error in conforming interpreters.

Static arrays are useful for lookup tables, constant data, and any array
whose values are known at compile time and should not be altered during
play. Using static arrays where possible reduces the amount of writable
memory consumed, which can be important on the Z-machine where dynamic
memory is limited.

## 8.5 Reading and Writing Array Elements

### 8.5.1 Byte Access (`->`)

The `->` operator reads or writes a single byte at a given byte offset
from the array's base address:

```inform6
Array data -> 5;

[ Main;
    data->0 = 10;              ! write byte at offset 0
    data->4 = 20;              ! write byte at offset 4
    print data->0;             ! read byte at offset 0 → prints 10
];
```

The expression `arr->index` computes the byte address `arr + index` and
loads or stores one byte at that address. The index is zero-based.

### 8.5.2 Word Access (`-->`)

The `-->` operator reads or writes a machine word at a given word offset
from the array's base address:

```inform6
Array data --> 5;

[ Main;
    data-->0 = 1000;           ! write word at index 0
    data-->4 = 2000;           ! write word at index 4
    print data-->0;            ! read word at index 0 → prints 1000
];
```

The expression `arr-->index` computes the byte address
`arr + index * WORDSIZE` and loads or stores one word at that address.

### 8.5.3 Accessing String, Table, and Buffer Arrays

For **string** and **table** arrays, index 0 holds the count of data
entries. Data entries begin at index 1:

```inform6
Array names table 'a' 'b' 'c';

[ Main  i;
    for (i = 1 : i <= names-->0 : i++)
        print (char) names-->i, " ";
    ! prints: a b c
];
```

For **buffer** arrays, the count is a word at byte offset 0 (accessed as
`arr-->0`), and data bytes begin at byte offset `WORDSIZE` (accessed as
`arr->WORDSIZE`, `arr->(WORDSIZE+1)`, etc.):

```inform6
Array buf buffer 10;

[ Main  i;
    ! Manually fill buffer data
    buf->WORDSIZE     = 'X';
    buf->(WORDSIZE+1) = 'Y';
    for (i = 0 : i < 2 : i++)
        print (char) buf->(WORDSIZE + i);
    ! prints: XY
];
```

### 8.5.4 Assignment Forms

Array element references are valid on the left-hand side of any assignment
operator:

```inform6
Array counters --> 4;

[ Main;
    counters-->0 = 100;        ! simple assignment
    counters-->0++;             ! post-increment
    ++counters-->1;             ! pre-increment
    counters-->2 = counters-->2 + 5;   ! compound update
];
```

### 8.5.5 No Bounds Checking

Inform 6 does **not** perform bounds checking on array accesses by default.
Reading or writing beyond the declared extent of an array accesses whatever
bytes happen to lie at that memory address — potentially other arrays,
global variables, or object data. The behaviour is undefined and can cause
subtle, hard-to-diagnose bugs:

```inform6
Array small -> 3;

[ Main;
    small->3 = 99;             ! out of bounds — undefined behaviour!
    small->100 = 1;            ! far out of bounds — may corrupt anything
];
```

It is the programmer's responsibility to ensure that all array indices are
within the valid range. Keeping the array size in a separate constant (or
using a table/string/buffer array whose length field tracks the count) is
strongly recommended.

## 8.6 Array Memory Layout

Understanding how arrays are laid out in memory is important for
interoperability with assembly code, for debugging, and for writing
platform-portable programs.

### 8.6.1 Byte Arrays (`->`)

A byte array of `N` entries occupies `N` contiguous bytes. The array
identifier holds the address of the first byte. No header or padding is
added.

```
Offset:   0     1     2         N-1
        +-----+-----+-----+...+-----+
        | b0  | b1  | b2  |   |bN-1 |
        +-----+-----+-----+...+-----+
```

### 8.6.2 Word Arrays (`-->`)

A word array of `N` entries occupies `N * WORDSIZE` contiguous bytes. Each
word is stored in big-endian order.

> **[Z-machine]** `WORDSIZE` is 2. A word array of 4 entries occupies 8
> bytes. Each entry is a 2-byte big-endian value.

> **[Glulx]** `WORDSIZE` is 4. A word array of 4 entries occupies 16
> bytes. Each entry is a 4-byte big-endian value.

```
                  Z-machine (WORDSIZE = 2)
Offset:   0   1   2   3   4   5   6   7
        +---+---+---+---+---+---+---+---+
        | w0_hi | w0_lo | w1_hi | w1_lo | ...
        +---+---+---+---+---+---+---+---+

                  Glulx (WORDSIZE = 4)
Offset:   0   1   2   3   4   5   6   7  ...
        +---+---+---+---+---+---+---+---+
        |      word 0       |      word 1   ...
        +---+---+---+---+---+---+---+---+
```

### 8.6.3 String Arrays

A string array has a one-byte length prefix at offset 0, followed by `N`
data bytes at offsets 1 through `N`. The length byte holds `N` (the count
of data entries, not the total allocation). Because the length byte is one
byte, `N` may not exceed 255.

```
Offset:   0     1     2         N
        +-----+-----+-----+...+-----+
        | len | d1  | d2  |   | dN  |
        +-----+-----+-----+...+-----+
```

Total size: `1 + N` bytes.

### 8.6.4 Table Arrays

A table array has a one-word length prefix at word index 0, followed by
`N` data words at word indices 1 through `N`:

```
Word index:   0       1       2           N
            +-------+-------+-------+...+-------+
            | count | d1    | d2    |   | dN    |
            +-------+-------+-------+...+-------+
```

Total size: `(1 + N) * WORDSIZE` bytes.

### 8.6.5 Buffer Arrays

A buffer array has a one-word length prefix (occupying `WORDSIZE` bytes)
followed by `N` data bytes:

```
Byte offset:  0 .. WORDSIZE-1   WORDSIZE   WORDSIZE+1       WORDSIZE+N-1
            +-----------------+----------+----------+...+----------+
            |    count (word) |    d1    |    d2    |   |    dN    |
            +-----------------+----------+----------+...+----------+
```

The count word is accessed as `arr-->0`. Data bytes are accessed as
`arr->WORDSIZE`, `arr->(WORDSIZE+1)`, and so on.

Total size: `WORDSIZE + N` bytes.

## 8.7 Multi-Dimensional Arrays

Inform 6 has no built-in syntax for multi-dimensional arrays. All arrays
are one-dimensional sequences of bytes or words. However, two-dimensional
(and higher) arrays can be simulated with index arithmetic.

### 8.7.1 Simulating a 2D Array

The standard convention is to store a two-dimensional grid in row-major
order in a word array, and compute the linear index as
`row * COLS + col`:

```inform6
Constant ROWS = 4;
Constant COLS = 5;
Array grid --> ROWS * COLS;    ! 20-entry word array

[ GridGet row col;
    return grid-->(row * COLS + col);
];

[ GridSet row col value;
    grid-->(row * COLS + col) = value;
];

[ Main  r c;
    ! Fill the grid with r*10 + c
    for (r = 0 : r < ROWS : r++)
        for (c = 0 : c < COLS : c++)
            GridSet(r, c, r * 10 + c);

    ! Read back a value
    print GridGet(2, 3), "^";  ! prints 23
];
```

### 8.7.2 Array of Arrays

An alternative approach uses a word array of pointers to other arrays:

```inform6
Array row0 --> 10 20 30;
Array row1 --> 40 50 60;
Array row2 --> 70 80 90;

Array matrix --> row0 row1 row2;

[ MatrixGet row col;
    return (matrix-->row)-->col;
];

[ Main;
    print MatrixGet(1, 2), "^";  ! prints 60
];
```

This approach uses more memory (one pointer per row plus the row arrays
themselves) but allows rows of different lengths.

## 8.8 Common Patterns

### 8.8.1 Text Input with Buffer Arrays

The `read` statement (or `aread` on Glulx) expects two arrays: a **text
buffer** to receive the raw characters typed by the player, and a **parse
buffer** to receive tokenised words. The text buffer is conventionally a
buffer array:

```inform6
Array text_buffer buffer 120;
Array parse_buffer -> 65;      ! Z-machine parse buffer (byte array)

[ Main;
    print "What now? ";
    read text_buffer parse_buffer;
];
```

After `read` executes, `text_buffer-->0` holds the number of characters
actually typed, and the characters themselves are stored starting at
`text_buffer->WORDSIZE`.

### 8.8.2 Lookup Tables

Table arrays are natural for lookup tables where the code iterates over
a known set of entries:

```inform6
Array colour_names table
    "red" "green" "blue" "yellow" "white" "black";

[ PrintColour n;
    if (n >= 1 && n <= colour_names-->0)
        print (string) colour_names-->n;
    else
        print "unknown";
];
```

Because `colour_names-->0` holds the entry count, adding or removing
entries from the source declaration automatically updates the loop bound —
no separate constant is needed.

### 8.8.3 Parallel Arrays

When multiple pieces of data are associated with each logical record,
parallel arrays provide a simple solution:

```inform6
Constant MAX_ENEMIES = 10;
Array enemy_health -> MAX_ENEMIES;
Array enemy_damage -> MAX_ENEMIES;
Array enemy_x --> MAX_ENEMIES;
Array enemy_y --> MAX_ENEMIES;

[ HurtEnemy i amount;
    if (enemy_health->i <= amount)
        enemy_health->i = 0;
    else
        enemy_health->i = enemy_health->i - amount;
];
```

### 8.8.4 Tracking Array Bounds

Since Inform 6 performs no runtime bounds checking, disciplined size
tracking is essential. The three main strategies are:

1. **Use a constant** for byte and word arrays:

    ```inform6
    Constant BUF_SIZE = 64;
    Array buf -> BUF_SIZE;

    [ SafeWrite index value;
        if (index >= 0 && index < BUF_SIZE)
            buf->index = value;
    ];
    ```

2. **Use the built-in length field** for table, string, and buffer arrays:

    ```inform6
    Array items table 'a' 'b' 'c';

    [ PrintAll  i;
        for (i = 1 : i <= items-->0 : i++)
            print (char) items-->i, " ";
    ];
    ```

3. **Use a sentinel value** to mark the end of data:

    ```inform6
    Array names --> "alpha" "beta" "gamma" 0;

    [ CountNames  i;
        for (i = 0 : names-->i ~= 0 : i++) ;
        return i;
    ];
    ```

### 8.8.5 Static Lookup Tables

Constant data that never changes at runtime should use the `static`
keyword to conserve writable memory:

```inform6
Array directions static table
    "north" "south" "east" "west"
    "northeast" "northwest" "southeast" "southwest"
    "up" "down" "in" "out";

[ DirectionName n;
    if (n >= 1 && n <= directions-->0)
        return directions-->n;
    return "nowhere";
];
```

### 8.8.6 String Arrays for Character Data

String arrays provide a compact way to store short, fixed character
sequences where the length is needed at runtime:

```inform6
Array vowels static string "aeiou";

[ IsVowel ch  i;
    for (i = 1 : i <= vowels->0 : i++)
        if (vowels->i == ch) rtrue;
    rfalse;
];
```

### 8.8.7 Copying Arrays

Inform 6 has no built-in array copy operation. Copying must be done
element by element:

```inform6
Constant SIZE = 10;
Array source --> SIZE;
Array dest --> SIZE;

[ CopyWordArray src dst n  i;
    for (i = 0 : i < n : i++)
        dst-->i = src-->i;
];

[ Main;
    source-->0 = 42;
    source-->1 = 99;
    CopyWordArray(source, dest, SIZE);
    print dest-->0, " ", dest-->1, "^";   ! prints 42 99
];
```
