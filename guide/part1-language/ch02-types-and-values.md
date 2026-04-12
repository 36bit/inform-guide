# Chapter 2: Types and Values

This chapter describes the value model: how data is represented,
interpreted, and manipulated at runtime. The language is **untyped**:
every value is a machine word, and the interpretation of that word
depends entirely on context. Understanding this model is essential for
writing correct programs and avoiding subtle errors that arise from treating
a value as the wrong kind of thing.

## 2.1 The Untyped Nature of Inform 6

Inform 6 has no static type system. Every variable, property value, array
element, routine argument, and return value occupies exactly one **machine
word**, and the compiler imposes no type discipline on how that word is used.
A variable that holds an object number can be passed to an arithmetic
operator; a dictionary address can be compared to an integer; a routine
address can be stored in a property alongside a string address. The compiler
will not complain about any of these operations.

This design reflects the underlying virtual machines. Both the Z-machine and
Glulx are stack-based architectures whose instructions operate on untyped
words. The compiler translates Inform 6 source into sequences of these
instructions and does not insert runtime type checks unless the programmer
explicitly requests them (for example, by calling `metaclass()`).

The consequence is that **the programmer bears full responsibility for
interpreting values correctly**. A value of `0` might mean the integer zero,
the boolean `false`, the object `nothing`, or a null pointer, depending on
context. The language provides tools for runtime type inspection (§2.12) but
no compile-time safety net.

```inform6
Global x;

[ Main;
    x = 42;          ! x holds an integer
    x = "hello";     ! now x holds a string address
    x = Lamp;        ! now x holds an object number
    print x + 1;     ! perfectly legal: adds 1 to the object number
];
```

The above program compiles without warnings. Whether `x + 1` produces a
meaningful result depends entirely on what the programmer intends.

## 2.2 Word Size by Target

The size of a machine word — and therefore the range of representable
values — depends on the target virtual machine.

### 2.2.1 Z-Machine (16-Bit Words)

> **[Z-machine]** On the Z-machine, each word is 16 bits (2 bytes). The
> compiler sets `WORDSIZE` to 2 and `MAXINTWORD` to `$7FFF`.

- **Signed range:** −32768 to 32767 (−2<sup>15</sup> to 2<sup>15</sup>−1)
- **Unsigned range:** 0 to 65535 (0 to 2<sup>16</sup>−1)

All values — integers, object numbers, addresses, dictionary references —
must fit within this 16-bit range.

### 2.2.2 Glulx (32-Bit Words)

> **[Glulx]** On Glulx, each word is 32 bits (4 bytes). The compiler sets
> `WORDSIZE` to 4 and `MAXINTWORD` to `$7FFFFFFF`.

- **Signed range:** −2147483648 to 2147483647 (−2<sup>31</sup> to
  2<sup>31</sup>−1)
- **Unsigned range:** 0 to 4294967295 (0 to 2<sup>32</sup>−1)

The larger word size permits more objects, longer strings, bigger arrays, and
a far greater address space.

### 2.2.3 Conditional Compilation for Word Size

Portable code that depends on numeric range must account for the target.
The compiler predefines `TARGET_ZCODE` or `TARGET_GLULX` (but not both)
and provides `WORDSIZE` as a runtime constant:

```inform6
Constant LIMIT = WORDSIZE * 8;   ! 16 or 32

[ ShowRange;
    #Ifdef TARGET_ZCODE;
    print "Z-machine: 16-bit words^";
    #Ifnot;
    print "Glulx: 32-bit words^";
    #Endif;
];
```

## 2.3 Integers and Arithmetic

### 2.3.1 Signed Integer Representation

Inform 6 integers use **two's complement** representation, the standard
encoding for signed integers on both virtual machines. In two's complement,
the most significant bit serves as the sign bit: 0 for non-negative values,
1 for negative values.

For 16-bit words (Z-machine):

| Binary | Signed value | Unsigned value |
| ------ | ------------ | -------------- |
| `$0000` | 0 | 0 |
| `$0001` | 1 | 1 |
| `$7FFF` | 32767 | 32767 |
| `$8000` | −32768 | 32768 |
| `$FFFF` | −1 | 65535 |

For 32-bit words (Glulx), the same principle applies with a 32-bit range.

### 2.3.2 Arithmetic Operators

The standard arithmetic operators (`+`, `-`, `*`, `/`, `%`) operate on
signed integers. The compiler emits the corresponding VM instructions,
which perform two's complement arithmetic:

```inform6
[ ArithmeticDemo x;
    x = 10 + 3;    ! x == 13
    x = 10 - 15;   ! x == -5
    x = 7 * 6;     ! x == 42
    x = 17 / 5;    ! x == 3  (truncated toward zero)
    x = 17 % 5;    ! x == 2  (remainder)
    x = -17 / 5;   ! x == -3 (truncated toward zero)
    x = -17 % 5;   ! x == -2 (sign follows dividend)
];
```

### 2.3.3 Division and Remainder Semantics

Integer division truncates toward zero, and the remainder operator preserves
the sign of the dividend. These semantics are defined by the Z-machine
specification and by the Glulx `div` and `mod` opcodes:

- `17 / 5` evaluates to `3` (not `4`).
- `-17 / 5` evaluates to `-3` (not `-4`).
- `17 % 5` evaluates to `2`.
- `-17 % 5` evaluates to `-2`.

Division by zero causes a runtime error on both platforms. The Z-machine
halts execution; Glulx behaviour depends on the interpreter but typically
produces an error.

### 2.3.4 Overflow Behaviour

Arithmetic overflow wraps silently according to two's complement rules. No
exception or error is raised:

```inform6
! Z-machine example (16-bit):
! 32767 + 1 wraps to -32768
[ OverflowDemo x;
    x = 32767;
    x = x + 1;
    print x, "^";   ! prints -32768 on Z-machine
];
```

On Glulx, the same principle applies with the 32-bit range: `2147483647 + 1`
wraps to `−2147483648`. Code that operates near the boundaries of the signed
range must account for this.

### 2.3.5 Bitwise Operations

Inform 6 provides bitwise operators that treat values as unsigned bit
patterns:

| Operator | Meaning |
| -------- | ------- |
| `&`      | Bitwise AND |
| `\|`     | Bitwise OR |
| `~`      | Bitwise NOT (unary) |

```inform6
[ BitwiseDemo x;
    x = $FF00 & $0F0F;   ! x == $0F00
    x = $FF00 | $00FF;   ! x == $FFFF
    x = ~$00FF;           ! x == $FF00 on Z-machine
];
```

Bitwise operations are commonly used to pack and unpack flags, manipulate
dictionary word flags, and perform low-level address calculations.

## 2.4 Boolean Values

Inform 6 does not have a dedicated boolean type. Boolean logic uses integer
conventions:

- **`true`** is predefined as the constant `1`.
- **`false`** is predefined as the constant `0`.
- **`nothing`** is predefined as `0` (see §2.5).

In conditional contexts (`if`, `while`, `do`–`until`, `&&`, `||`, `~~`),
any **nonzero** value is treated as true and **zero** is treated as false:

```inform6
[ TruthDemo x;
    x = 42;
    if (x) print "truthy^";         ! prints: 42 is nonzero

    x = 0;
    if (~~x) print "falsy^";        ! prints: 0 is false

    if (nothing) print "never^";    ! nothing is 0, so this is skipped
];
```

Conditional operators and comparison operators produce `1` for true and `0`
for false:

```inform6
[ CompareDemo x;
    x = (3 > 2);   ! x == 1 (true)
    x = (3 < 2);   ! x == 0 (false)
];
```

Because any nonzero value is truthy, object references, routine addresses,
string addresses, and dictionary words all test as true in boolean contexts.
Only the value `0` — which serves simultaneously as `false`, `nothing`, and
the null pointer — tests as false.

## 2.5 Object References

Objects in Inform 6 are identified by **object numbers**: small positive
integers assigned sequentially by the compiler, starting from 1. The special
value `0`, represented by the constant `nothing`, means "no object."

### 2.5.1 Object Numbering

The compiler assigns object numbers in the order that `Object` and `Class`
declarations appear in the source. Classes defined with `Class` receive the
lowest numbers, followed by objects defined with `Object`. The exact
numbering depends on declaration order and on how the library defines its own
objects.

> **[Z-machine]** In Z-machine version 3, the maximum number of objects is
> 255. In versions 5 and 8, the limit is configurable via the `MAX_OBJECTS`
> compiler setting but defaults to 512.

> **[Glulx]** Glulx imposes no hard architectural limit on the number of
> objects. The practical limit is determined by available memory and the
> `MAX_OBJECTS` compiler setting.

### 2.5.2 The `nothing` Constant

The constant `nothing` has the value `0` and type `OBJECT_T` in the
compiler's symbol table. It is used wherever an object reference is expected
but no object is appropriate:

```inform6
[ FindContainer obj;
    obj = parent(player);
    if (obj == nothing)
        print "The player has no parent.^";
];
```

Properties that hold object references conventionally use `nothing` (or
simply `0`) to indicate "no object." The tree-manipulation routines
`parent()`, `child()`, `sibling()`, and `children()` return `nothing` when
the requested relation does not exist.

### 2.5.3 Objects as Values

Because object numbers are ordinary integers, they participate in all the
usual operations. They can be stored in variables, passed as arguments,
returned from routines, placed in arrays, and compared with `==`, `~=`,
`>`, and `<`:

```inform6
[ HigherNumbered a b;
    if (a > b) return a;
    return b;
];
```

However, the numeric ordering of objects has no semantic significance and
should not be relied upon for game logic. Use the object tree (`in`,
`notin`, `parent`, `child`, `sibling`) and property tests (`provides`,
`has`, `hasnt`) for meaningful queries.

### 2.5.4 Type-Checking with `metaclass()`

Because an object number is just an integer, the programmer may need to
verify at runtime that a value actually refers to an object. The
`metaclass()` function (§2.12) returns `Object` for valid object references
and `Class` for class objects:

```inform6
[ SafePrint val;
    switch (metaclass(val)) {
        Object: print (name) val;
        Class:  print (name) val, " [class]";
        default: print "(not an object)";
    }
];
```

## 2.6 Addresses and Pointers

Values in Inform 6 frequently represent memory addresses. The language
provides two forms of array access based on the kind of address:

### 2.6.1 Byte Addresses

The `->` operator accesses individual bytes in memory. When applied to an
array or address value, `a->n` reads the byte at address `a + n`:

```inform6
Array data -> 5;   ! a byte array of 5 elements

[ ByteDemo;
    data->0 = 65;
    data->1 = 66;
    print (char) data->0, (char) data->1, "^";   ! prints AB
];
```

Byte addresses are simple offsets into the story file's memory. They are not
scaled by word size.

### 2.6.2 Word Addresses

The `-->` operator accesses word-sized values. `a-->n` reads the word at
byte address `a + n * WORDSIZE`:

```inform6
Array table --> 4;   ! a word array of 4 elements

[ WordDemo;
    table-->0 = 100;
    table-->1 = 200;
    print table-->0, " ", table-->1, "^";   ! prints 100 200
];
```

On the Z-machine, each word is 2 bytes, so `a-->n` accesses address
`a + 2*n`. On Glulx, each word is 4 bytes, so `a-->n` accesses address
`a + 4*n`.

### 2.6.3 Packed Addresses (Z-Machine)

> **[Z-machine]** Certain values in the Z-machine are stored as **packed
> addresses** rather than byte addresses. A packed address is divided by a
> **scale factor** to fit into a 16-bit word:
>
> | Z-machine version | Scale factor |
> | ------------------ | ------------ |
> | Version 3          | 2            |
> | Versions 4–5       | 4            |
> | Version 8          | 8            |
>
> Routine addresses and string addresses in the Z-machine are always packed.
> The interpreter multiplies the packed address by the scale factor to
> recover the true byte address before accessing memory.
>
> Packed addresses allow the Z-machine to reference memory locations beyond
> the 64 KB that a raw 16-bit value can address. With a scale factor of 8
> (version 8), the addressable range extends to 512 KB.

The programmer does not normally need to perform packed-address arithmetic
manually. The compiler emits the correct instructions, and the interpreter
handles unpacking. However, awareness of packed addresses is important when
working with low-level operations or debugging.

### 2.6.4 Direct Addresses (Glulx)

> **[Glulx]** Glulx uses direct 32-bit addresses throughout. There is no
> address packing; routine addresses and string addresses are stored as
> their actual byte offsets within the story file. The compiler sets the
> internal `scale_factor` to 0 (unused).

This simplification means that all addresses in Glulx are uniformly
comparable and arithmetic on them is straightforward.

## 2.7 Routine References

A routine (function) in Inform 6 can be treated as a first-class value: its
address can be stored in a variable, placed in a property, passed as an
argument, and invoked at runtime.

### 2.7.1 Routines as Values

When a routine name appears in an expression context where it is not being
called directly, the compiler emits its address as a constant value:

```inform6
[ Greet;
    print "Hello!^";
];

[ Main fn;
    fn = Greet;           ! fn now holds the address of Greet
    print fn, "^";        ! prints the numeric address
];
```

> **[Z-machine]** Routine addresses are stored as packed addresses. The
> scale factor depends on the Z-machine version (see §2.6.3).

> **[Glulx]** Routine addresses are stored as direct byte addresses.

### 2.7.2 Calling Routine Values with `indirect()`

The `indirect()` system function calls a routine through its address. It
accepts the routine address as its first argument, followed by up to seven
arguments to pass to the routine:

```inform6
[ Add a b;
    return a + b;
];

[ Main fn result;
    fn = Add;
    result = indirect(fn, 10, 20);
    print result, "^";                ! prints 30
];
```

The `indirect()` function is a compiler built-in (system function number 4).
It generates the appropriate call instruction for the target VM.

### 2.7.3 Routines in Properties

Object properties commonly hold routine values to implement behaviour that
varies per object. When the library evaluates a property and discovers that
it contains a routine address (determined via `metaclass()`), it calls the
routine rather than using the value directly:

```inform6
Object lamp "brass lamp"
    with description [;
             if (self has on) "The lamp glows brightly.";
             "The lamp is dark.";
         ],
    has  switchable;
```

Here, `description` holds an **embedded routine** — an anonymous routine
whose address the compiler generates and stores in the property. The library
calls this routine when printing the lamp's description.

Alternatively, a named routine can be stored explicitly:

```inform6
[ LampDesc;
    if (lamp has on) "The lamp glows brightly.";
    "The lamp is dark.";
];

Object lamp "brass lamp"
    with description LampDesc,
    has  switchable;
```

### 2.7.4 Distinguishing Routines from Other Values

A value known to be either a routine or another type can be tested with
`metaclass()`:

```inform6
[ RunOrPrint val;
    if (metaclass(val) == Routine)
        indirect(val);
    else
        print (string) val;
];
```

The `Z__Region()` veneer routine, which underlies `metaclass()`,
distinguishes routine addresses from other values by checking whether the
address falls within the code segment of the story file (see §2.12).

## 2.8 String References

Strings in Inform 6 are compiled into a dedicated region of the story file,
and references to strings are stored as addresses.

### 2.8.1 Strings as Values

A double-quoted string in an expression context evaluates to its address:

```inform6
Global greeting = "Hello, world!";

[ Main;
    print (string) greeting, "^";   ! prints Hello, world!
];
```

> **[Z-machine]** String addresses are packed addresses, subject to the
> same scale factor as routine addresses (see §2.6.3).

> **[Glulx]** String addresses are direct byte addresses.

### 2.8.2 Printing String Values

A string address can be printed with the `(string)` print rule:

```inform6
[ PrintMessage msg;
    print (string) msg, "^";
];
```

Using `print` without `(string)` on a string address prints its numeric
value, not the text it represents.

### 2.8.3 Strings vs. Routines

Both string addresses and routine addresses are stored in properties and
variables as simple word values. The `metaclass()` function distinguishes
them: it returns `String` for string addresses and `Routine` for routine
addresses. The library uses this mechanism extensively to handle properties
that may contain either strings or routines:

```inform6
[ PrintOrRun obj prop flag;
    if (metaclass(obj.prop) == Routine)
        return RunRoutines(obj, prop);
    if (metaclass(obj.prop) == String) {
        print (string) obj.prop;
        if (flag) print "^";
        rtrue;
    }
];
```

## 2.9 Dictionary Word Values

When the source code contains a single-quoted word literal such as `'take'`
or `'lamp'`, the compiler looks up (or creates) an entry in the story file's
**dictionary** and produces the address of that entry as a compile-time
constant.

### 2.9.1 Dictionary Literals

A dictionary word literal is written in single quotes:

```inform6
if (w == 'take') print "You said take!^";
```

The expression `'take'` evaluates to the address of the dictionary entry for
"take". This is a numeric value — a word-sized address — that can be stored,
compared, and passed like any other value.

Single-character dictionary literals require special handling. Because `'x'`
is interpreted by the compiler as the ASCII value of the character `x`
(not a dictionary word), the form `#n$x` must be used to obtain the
dictionary address of a single-character word:

```inform6
Constant DICT_A = #n$a;   ! dictionary address of "a"
```

For words of two or more characters, `'word'` is the standard form and
`#n$word` is a deprecated equivalent.

### 2.9.2 Dictionary Address Semantics

Two dictionary word expressions are equal if and only if they refer to the
same dictionary entry. Because the dictionary stores words in a truncated
and normalised form (see §1.4), words that differ only beyond the truncation
limit compare as equal:

```inform6
! Z-machine: dictionary words are truncated to 6 characters (v3)
! or 9 characters (v5+). On Glulx, the limit is also 9 by default.
if ('examine' == 'examin') ...   ! true if truncated to 6 characters
```

Dictionary comparisons are simple integer equality tests on the addresses.
There is no string comparison involved at runtime.

### 2.9.3 Testing Input Against Dictionary Words

Dictionary word values are most commonly used in `parse_name` routines, verb
grammar, and `Consult` handling, where the parsed input tokens are compared
to known dictionary entries:

```inform6
Object lever "lever"
    with parse_name [ w n;
             w = NextWord();
             while (w == 'rusty' or 'old' or 'lever') {
                 n++;
                 w = NextWord();
             }
             return n;
         ],
    has  static;
```

## 2.10 Action Values

An **action** is an integer constant that identifies a particular kind of
player command (such as taking, dropping, or examining). The compiler assigns
action numbers sequentially as actions are defined.

### 2.10.1 The `##` Syntax

The `##` prefix followed by an action name produces the action's integer
constant:

```inform6
[ before;
    if (action == ##Take)
        "You can't take that.";
];
```

The expression `##Take` evaluates to whatever integer the compiler assigned
to the `Take` action. This value can be stored, compared, and used in
`switch` statements:

```inform6
[ ReactToAction act;
    switch (act) {
        ##Take:    print "taking^";
        ##Drop:    print "dropping^";
        ##Examine: print "examining^";
    }
];
```

### 2.10.2 Action Numbers

Action numbers are plain integers with no inherent structure. The compiler
assigns them based on `Verb` directive order. The programmer should never
hard-code specific action numbers; always use the `##ActionName` syntax to
refer to actions by name:

```inform6
! WRONG: fragile, depends on compilation order
if (action == 37) ...

! RIGHT: portable and readable
if (action == ##Examine) ...
```

The library variable `action` holds the current action number during action
processing, and the `action_to_be` variable holds the action being
considered before `GamePreRoutine` and `before`/`after` processing.

## 2.11 Floating-Point Values (Glulx)

> **[Glulx]** Glulx version 3.1.2 and later supports **IEEE 754
> single-precision floating-point** values. These occupy a single 32-bit
> word — the same size as an integer — and are distinguished from integers
> only by the operations applied to them.

> **[Z-machine]** Floating-point values are not available on the Z-machine.
> The compiler issues an error if a floating-point literal appears in
> Z-code mode.

### 2.11.1 Floating-Point Literal Syntax

Floating-point constants are written with a `$+` or `$-` prefix, followed
by a decimal number with an optional fractional part and optional exponent:

```inform6
Constant PI       = $+3.14159;
Constant NEGONE   = $-1.0;
Constant AVOGADRO = $+6.022e23;
Constant TINY     = $+1.5e-10;
```

The `$+` prefix indicates a non-negative float; `$-` indicates a negative
float. The sign prefix is mandatory — bare `$3.14` is interpreted as a
hexadecimal integer, not a float.

### 2.11.2 IEEE 754 Representation

A Glulx floating-point value occupies a 32-bit word with the standard
IEEE 754 single-precision layout:

| Bits | Field | Width |
| ---- | ----- | ----- |
| 31 | Sign | 1 bit |
| 30–23 | Exponent | 8 bits |
| 22–0 | Mantissa | 23 bits |

Special values include positive and negative infinity and NaN (not a
number). The compiler generates these when a literal overflows or is
otherwise unrepresentable.

### 2.11.3 Floating-Point Operations

Glulx provides dedicated opcodes for floating-point arithmetic. These are
accessed in Inform 6 through assembly statements:

```inform6
[ FloatAdd a b result;
    @fadd a b -> result;
    return result;
];

[ FloatMul a b result;
    @fmul a b -> result;
    return result;
];
```

The key floating-point opcodes include:

| Opcode | Operation |
| ------ | --------- |
| `@fadd` | Addition |
| `@fsub` | Subtraction |
| `@fmul` | Multiplication |
| `@fdiv` | Division |
| `@fmod` | Remainder |
| `@floor` | Floor (float to int) |
| `@ceil` | Ceiling (float to int) |
| `@sqrt` | Square root |
| `@exp` | Exponential |
| `@log` | Natural logarithm |
| `@pow` | Power |
| `@sin` | Sine |
| `@cos` | Cosine |
| `@tan` | Tangent |
| `@asin` | Arcsine |
| `@acos` | Arccosine |
| `@atan` | Arctangent |
| `@atan2` | Two-argument arctangent |
| `@jfeq` | Branch if equal (with epsilon) |
| `@jfne` | Branch if not equal |
| `@jflt` | Branch if less than |
| `@jfle` | Branch if less than or equal |
| `@jfgt` | Branch if greater than |
| `@jfge` | Branch if greater than or equal |
| `@jisnan` | Branch if NaN |
| `@jisinf` | Branch if infinite |
| `@numtof` | Integer to float |
| `@ftonumz` | Float to integer (truncate toward zero) |
| `@ftonumn` | Float to integer (round to nearest) |

Standard Inform 6 arithmetic operators (`+`, `-`, `*`, `/`) perform
**integer** arithmetic even when applied to values that contain float bit
patterns. To perform floating-point arithmetic, the programmer must use the
assembly opcodes listed above.

### 2.11.4 Double-Precision Extensions

> **[Compiler 6.40+]** The compiler also supports 64-bit double-precision
> float literals using `$>` (high 32 bits) and `$<` (low 32 bits) prefixes.
> These split an IEEE 754 double into two 32-bit words:

```inform6
! The double 3.14159265358979 as two 32-bit words:
Constant PI_HI = $>+3.14159265358979;
Constant PI_LO = $<+3.14159265358979;
```

Whether these are useful depends on the availability of Glulx interpreter
support for double-precision opcodes.

## 2.12 The `metaclass()` Function

The `metaclass()` function is the primary mechanism for runtime type
inspection in Inform 6. It examines a value and returns one of five results:

| Return value | Meaning |
| ------------ | ------- |
| `Object`   | The value is an ordinary object (instance) number |
| `Class`    | The value is a class object |
| `Routine`  | The value is a routine address |
| `String`   | The value is a string address |
| `nothing`  | The value is `0`, out of range, or not a recognised type |

### 2.12.1 How `metaclass()` Works

The `metaclass()` function is implemented as a **veneer routine** — a small
piece of code that the compiler automatically inserts into every compiled
program. It delegates to the internal `Z__Region()` veneer routine, which
determines the broad category of a value by examining its numeric range.

> **[Z-machine]** `Z__Region()` classifies a value by comparing it against
> the boundaries of the object table, the code segment, and the strings
> segment:
>
> - Values from `1` to the largest object number → region 1 (object)
> - Values at or above the code offset → region 2 (routine)
> - Values at or above the strings offset → region 3 (string)
> - `0`, `−1`, and values beyond memory → region 0 (nothing)
>
> Within region 1, `metaclass()` further checks whether the object is a
> class (contained in object 1, i.e. the metaclass `Class`, or is one of
> the four built-in metaclass objects) and returns `Class`; otherwise it
> returns `Object`.

> **[Glulx]** `Z__Region()` examines the byte at the given address in
> memory. Glulx marks the first byte of each data structure with a type
> tag:
>
> - Bytes `$E0`–`$FF` → region 3 (string)
> - Bytes `$C0`–`$DF` → region 2 (routine)
> - Bytes `$70`–`$7F` at or above the RAM start → region 1 (object)
> - All other values → region 0 (nothing)
>
> The Glulx version of `metaclass()` then uses the same class-membership
> test to distinguish `Class` from `Object` within region 1.

### 2.12.2 Using `metaclass()` for Type Checking

The `metaclass()` function is essential for writing code that handles
polymorphic values — properties or variables that might contain objects,
routines, or strings:

```inform6
[ DescribeValue val;
    switch (metaclass(val)) {
        nothing: print "Nothing (0 or invalid)";
        Object:  print "Object: ", (name) val;
        Class:   print "Class: ", (name) val;
        Routine: print "Routine at address ", val;
        String:  print "String: ", (string) val;
    }
    print "^";
];
```

The library makes extensive use of `metaclass()` in its property-handling
code. For instance, when evaluating an object's `description` property, the
library checks whether the value is a `String` (to be printed directly) or a
`Routine` (to be called):

```inform6
[ ExamineObj obj addr;
    addr = obj.description;
    switch (metaclass(addr)) {
        String:  print (string) addr, "^";
        Routine: indirect(addr);
    }
];
```

### 2.12.3 Limitations

The `metaclass()` function relies on address-range heuristics, not on stored
type tags (except on Glulx, where memory tags assist). It cannot determine
whether an arbitrary integer is "intended" to be an object number, an action
number, or a plain integer — it can only check whether the numeric value
falls within a range that corresponds to a known object, routine, or string.

A value that happens to equal a valid object number but was intended as a
plain integer will be reported as `Object`. Conversely, a stale object
reference whose object has been removed will produce unpredictable results.

## 2.13 Type Coercion and Interpretation

### 2.13.1 No Implicit Type Conversion

Inform 6 performs **no implicit type conversion**. The compiler does not
convert between integers, objects, strings, or routines. Every value is a raw
machine word, and every operation interprets that word according to its own
rules:

- Arithmetic operators interpret their operands as signed integers.
- The `print` statement with `(name)` interprets its operand as an object
  number.
- The `print` statement with `(string)` interprets its operand as a string
  address.
- The `in` and `notin` operators interpret their operands as object numbers.
- The `has` and `hasnt` operators interpret the right operand as an attribute
  number.

There is no conversion step: if a value is not what the operation expects,
the operation proceeds anyway with undefined or erroneous results.

### 2.13.2 The Programmer's Responsibility

Because the language imposes no type discipline, the programmer must maintain
a mental model of what each variable, property, and array element holds at
each point in the program. Defensive programming practices include:

1. **Use `metaclass()` before acting on unknown values.** When a value might
   be an object, a routine, or a string, test it before use.

2. **Use clear naming conventions.** Names like `target_obj`, `action_fn`,
   and `msg_str` convey the intended type to the reader.

3. **Document property types.** When an object property may hold either a
   string or a routine, document this in a comment.

4. **Avoid storing unrelated types in the same variable.** While the language
   permits it, reusing a variable for values of different types obscures
   intent and invites errors.

### 2.13.3 Common Pitfalls

**Printing a string address as an integer.** Using `print val;` when `val`
holds a string address prints the numeric address, not the string text. Use
`print (string) val;` instead.

**Treating an object number as a pointer.** Object numbers are indices into
the object table, not memory addresses. Applying `->` or `-->` to an object
number accesses arbitrary memory relative to a small integer, which is almost
never correct.

**Comparing values of different kinds.** An expression like
`if (obj == "lamp")` compares an object number to a string address. These
are unrelated numeric values and the comparison is almost certainly
meaningless. To test an object's name, use `if (obj == lamp)` (comparing
object numbers) or compare against a dictionary word:
`if (WordInProperty('lamp', obj, name))`.

**Arithmetic on non-integer values.** Adding 1 to a routine address does
not produce the address of the "next" routine. Adding 1 to a string address
does not produce a meaningful string. Arithmetic on non-integer values is
legal but rarely intentional.

**Assuming boolean results are `true` (1) or `false` (0).** While
comparison operators produce `1` or `0`, other functions may return arbitrary
nonzero values to indicate truth. Always test with `if (x)` rather than
`if (x == true)`:

```inform6
! FRAGILE: fails if SomeTest returns 2, 3, etc.
if (SomeTest() == true) ...

! CORRECT: any nonzero value is truthy
if (SomeTest()) ...
```

---

*Copyright © 2026 Software Freedom Conservancy, Inc.*
*Licensed under the GNU General Public License, version 3 or later.*
