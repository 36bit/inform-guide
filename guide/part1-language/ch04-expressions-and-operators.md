# Chapter 4: Expressions and Operators

This chapter describes every operator in Inform 6, how expressions are
evaluated, and how the compiler interprets expressions differently
depending on the context in which they appear. The information here is
derived directly from the operator table in the compiler source
(`expressc.c`) and the expression parser (`expressp.c`), supplemented by
the I6-Addendum's clarifications on precedence and the `or` operator.

## 4.1 Expression Evaluation

### 4.1.1 Left-to-Right Evaluation

Inform 6 evaluates most binary operators **left to right**, consistent with
the left-associative grouping rules defined for each precedence level. In
an expression like `a + b + c`, the compiler first evaluates `a + b` and
then adds `c` to the result.

```inform6
[ Main  x;
    x = 10 - 3 - 2;
    print x;          ! prints 5: (10 - 3) - 2, not 10 - (3 - 2)
];
```

The one syntactic exception is the assignment operator `=`, which is
**right-associative**. The expression `a = b = 0` assigns 0 to `b` first,
then assigns the result (0) to `a`.

### 4.1.2 Short-Circuit Evaluation

The logical operators `&&` (and) and `||` (or) use **short-circuit
evaluation**: the right-hand operand is not evaluated if the left-hand
operand already determines the result.

- For `&&`: if the left operand is **false** (zero), the entire expression
  is false and the right operand is skipped.
- For `||`: if the left operand is **true** (non-zero), the entire
  expression is true and the right operand is skipped.

```inform6
[ SafeDivide a b;
    if (b ~= 0 && a/b > 10)
        print "Ratio exceeds 10.^";
    ! If b is 0, the division a/b never executes.
];
```

Short-circuit evaluation is essential for guarding operations that would
cause runtime errors. The compiler implements this by generating
conditional branch instructions: for `&&`, if the left operand is false,
a branch jumps past the right operand; for `||`, if the left operand is
true, a branch jumps past the right operand.

At the constant-folding level, the compiler also applies short-circuit
rules: `0 && <anything>` folds to 0, and `1 || <anything>` folds to 1,
without evaluating the right-hand expression.

## 4.2 Arithmetic Operators

Inform 6 provides the standard five arithmetic operators. All operate on
**signed** machine words (16-bit on the Z-machine, 32-bit on Glulx).

| Operator | Meaning | Precedence | Associativity |
| -------- | ------- | ---------- | ------------- |
| `+` | Addition | 5 | Left |
| `-` | Subtraction | 5 | Left |
| `*` | Multiplication | 6 | Left |
| `/` | Division (truncated toward zero) | 6 | Left |
| `%` | Remainder | 6 | Left |

### 4.2.1 Addition and Subtraction

```inform6
[ Arithmetic  a b;
    a = 100 + 23;       ! a is 123
    b = a - 50;          ! b is 73
    print a + b;         ! prints 196
];
```

Addition and subtraction share precedence level 5 and associate left to
right. Overflow wraps silently: on the Z-machine, `32767 + 1` yields
`-32768`.

### 4.2.2 Multiplication

```inform6
[ Area w h;
    return w * h;
];

[ Main;
    print Area(7, 8);   ! prints 56
];
```

### 4.2.3 Division and Remainder

Division truncates toward zero (not toward negative infinity), following
the Z-machine and Glulx specifications:

```inform6
[ Main;
    print 7 / 2;        ! prints 3
    print -7 / 2;       ! prints -3 (truncation toward zero)
    print 7 % 2;        ! prints 1
    print -7 % 2;       ! prints -1
];
```

Division by zero is a runtime error handled by the virtual machine. On
the Z-machine, it is typically fatal. On Glulx, behavior depends on the
interpreter.

### 4.2.4 Unary Minus

The unary minus operator `-` negates its operand. It has precedence 8
(higher than binary arithmetic) and associates right to left.

```inform6
[ Negate n;
    return -n;           ! returns the negation of n
];
```

The compiler distinguishes unary minus from binary subtraction by
context: if `-` appears at the start of an expression, after an operator,
or after an opening parenthesis, it is parsed as unary.

## 4.3 Comparison Operators

Comparison operators test the relationship between two values and yield
a result usable in a condition. All comparisons are **non-associative**:
the expression `a < b < c` is a compile-time error, not a chained
comparison.

| Operator | Meaning | Precedence |
| -------- | ------- | ---------- |
| `==` | Equal to | 3 |
| `~=` | Not equal to | 3 |
| `>` | Greater than | 3 |
| `<` | Less than | 3 |
| `>=` | Greater than or equal to | 3 |
| `<=` | Less than or equal to | 3 |

```inform6
[ Compare a b;
    if (a == b) print "equal";
    if (a ~= b) print "not equal";
    if (a > b)  print "greater";
    if (a < b)  print "less";
    if (a >= b) print "greater or equal";
    if (a <= b) print "less or equal";
];
```

All comparisons use **signed** arithmetic. On the Z-machine, the value
`$FFFF` is −1, not 65535, so `$FFFF < 0` is true.

Each comparison operator has a **negation partner** in the compiler's
operator table: `==` and `~=` are negations of each other, `>` and `<=`
are negations of each other, and `<` and `>=` are negations of each other.
The compiler uses this pairing to invert branch conditions efficiently.

> **Note:** Inform uses `==` for equality testing (not `=`, which is
> assignment). Using `=` in a condition is a common mistake that assigns
> rather than compares.

## 4.4 Logical Operators

The logical operators combine boolean conditions. In Inform 6, any
non-zero value is **true** and zero is **false**.

| Operator | Meaning | Precedence | Associativity |
| -------- | ------- | ---------- | ------------- |
| `&&` | Logical AND | 2 | Left |
| `\|\|` | Logical OR | 2 | Left |
| `~~` | Logical NOT | 2 | Right (prefix) |

### 4.4.1 Logical AND (`&&`)

True when both operands are non-zero. Uses short-circuit evaluation: if the
left operand is false, the right is never evaluated.

```inform6
[ CanPickUp obj;
    if (obj has takeable && obj notin player)
        print (The) obj, " is available.^";
];
```

### 4.4.2 Logical OR (`||`)

True when at least one operand is non-zero. Uses short-circuit evaluation:
if the left operand is true, the right is never evaluated.

```inform6
[ IsExit dir;
    if (dir == n_to || dir == s_to || dir == e_to || dir == w_to)
        rtrue;
];
```

### 4.4.3 Logical NOT (`~~`)

Yields true (1) when its operand is false (zero), and false (0) when its
operand is non-zero. This is a prefix operator.

```inform6
[ Main;
    if (~~(score > 100))
        print "Score is still 100 or below.^";
];
```

### 4.4.4 Precedence of `~~`

The DM4's operator table (§Table1B) shows `&&`, `||`, and `~~` as having
equal precedence. In practice, the compiler has always parsed `~~` as
binding **less tightly** than `&&` and `||`. The expression `~~X && Y` is
parsed as `~~(X && Y)`, not `(~~X) && Y`:

```inform6
! These two statements are equivalent:
val = ~~X && Y;
val = ~~(X && Y);

! To negate only X, use explicit parentheses:
val = (~~X) && Y;
```

The I6-Addendum describes this as `~~` having an effective precedence of
"1.5"—higher than assignment but lower than the other logical operators.
The compiler implements this by giving `~~` the same numeric precedence
level (2) as `&&` and `||` but marking it as a prefix operator, which
the shift-reduce parser resolves by reducing the entire right-hand
subexpression before applying the negation.

## 4.5 Bitwise Operators

Bitwise operators manipulate individual bits of machine words. They share
precedence level 6 with the multiplicative arithmetic operators.

| Operator | Meaning | Precedence | Associativity |
| -------- | ------- | ---------- | ------------- |
| `&` | Bitwise AND | 6 | Left |
| `\|` | Bitwise OR | 6 | Left |
| `~` | Bitwise NOT | 6 | Right (prefix) |

### 4.5.1 Bitwise AND (`&`)

Sets each bit of the result to 1 only if both corresponding input bits
are 1.

```inform6
[ HasBit value mask;
    if (value & mask)
        print "Bit is set.^";
];
```

### 4.5.2 Bitwise OR (`|`)

Sets each bit of the result to 1 if either corresponding input bit is 1.

```inform6
[ SetFlags flags new_flag;
    return flags | new_flag;
];
```

### 4.5.3 Bitwise NOT (`~`)

Inverts every bit of its operand. This is a prefix unary operator.

```inform6
[ ClearBit value mask;
    return value & (~mask);    ! clear the bits specified by mask
];
```

As with logical `~~`, the bitwise `~` binds less tightly than the binary
bitwise operators at the same precedence level. The expression `~X & Y`
is parsed as `~(X & Y)`. Use parentheses to negate only `X`:

```inform6
result = (~X) & Y;     ! negate X, then AND with Y
result = ~X & Y;       ! equivalent to ~(X & Y)
```

### 4.5.4 Distinguishing Logical and Bitwise Operators

Inform 6 uses distinct symbols for logical and bitwise operations:

| Logical | Bitwise | Operation |
| ------- | ------- | --------- |
| `&&` | `&` | AND |
| `\|\|` | `\|` | OR |
| `~~` | `~` | NOT |

Despite different symbols, the bitwise and logical operators have similar
precedence levels, which can cause confusion. When mixing them, always use
parentheses to make intent explicit.

## 4.6 The `or` Operator

The `or` operator has unique semantics in Inform 6. It is **not** a general
boolean connective; it may only appear on the **right-hand side** of a
comparison or condition operator, providing a list of alternative values to
test against.

### 4.6.1 Basic Semantics

```inform6
if (x == 1 or 2 or 3)      ! true if x equals 1, 2, or 3
if (obj has light or open)  ! true if obj has light or open
if (obj in box or chest)    ! true if obj is in box or in chest
```

The compiler expands `x == 1 or 2 or 3` into the equivalent of
`(x == 1) || (x == 2) || (x == 3)`, but more efficiently: it generates a
sequence of branch instructions testing the left operand against each right
value in turn, jumping to the "true" target as soon as any match succeeds.

### 4.6.2 Negative-Sense Operators with `or`

For operators with negative sense (`~=`, `hasnt`, `notin`), `or` tests
that the left operand matches **none** of the right values:

```inform6
if (x ~= 1 or 2 or 3)        ! true if x is not 1, 2, or 3
if (obj hasnt light or open)  ! true if obj lacks both light and open
if (obj notin box or chest)   ! true if obj is in neither box nor chest
```

This follows natural English phrasing rather than strict boolean logic.
Logically, `x ~= 1 or 2 or 3` means `(x ~= 1) && (x ~= 2) && (x ~= 3)`.

### 4.6.3 Ordering Comparisons with `or`

The `or` operator works with the comparison operators `>`, `<`, `>=`,
and `<=`, though the results can be unintuitive:

```inform6
if (x > 100 or y)     ! true if (x > 100) or (x > y)
if (x < 100 or y)     ! true if (x < 100) or (x < y)
```

The `>=` and `<=` operators are implemented internally as the negations of
`<` and `>` respectively. This means they interact with `or` using
**negative-sense** semantics:

```inform6
if (x <= 100 or y)    ! true if (x <= 100) AND (x <= y)
if (x >= 100 or y)    ! true if (x >= 100) AND (x >= y)
```

This is counterintuitive. As of compiler version 6.36, using `or` with
`>=` or `<=` produces a compiler warning. The clearest approach is to
avoid `or` with ordering comparisons and write the conditions out
explicitly.

### 4.6.4 Restrictions on `or`

The `or` operator has precedence 4, placing it between the logical
operators (precedence 2) and the arithmetic operators (precedence 5).
It may only appear as the right operand of a comparison or condition
operator. Using `or` in any other position is a compile-time error:

```inform6
! INVALID — or cannot be used as a standalone boolean connective:
! if (x or y) ...           ! compile-time error

! VALID — or as part of a comparison:
if (x == 1 or 2) ...
```

## 4.7 Condition Operators

Condition operators test object relationships and class membership. Each
has a positive and a negated form. All are **infix**, **non-associative**,
and share precedence level 3 with the comparison operators.

### 4.7.1 `has` and `hasnt`

Test whether an object possesses (or lacks) a given attribute.

```inform6
Object lamp "brass lamp" has light switchable;

[ Main;
    if (lamp has light)    print "The lamp shines.^";
    if (lamp hasnt open)   print "The lamp is not open.^";
];
```

On the Z-machine, `has` compiles to the `test_attr` instruction. `hasnt`
compiles to the negated form of the same instruction.

### 4.7.2 `in` and `notin`

Test whether an object is (or is not) a direct child of another object
in the object tree.

```inform6
Object bag "leather bag";
Object coin "gold coin";

[ Main;
    move coin to bag;
    if (coin in bag)       print "The coin is in the bag.^";
    if (coin notin player) print "You don't have the coin.^";
];
```

On the Z-machine, `in` compiles to the `jin` (jump-if-in) instruction.

### 4.7.3 `ofclass` and `not ofclass`

Test whether an object is (or is not) a member of a given class. The
negated form uses `~~` with `ofclass`:

```inform6
Class Treasure;
Treasure diamond "diamond";
Object rock "ordinary rock";

[ Main;
    if (diamond ofclass Treasure)
        print "The diamond is a treasure.^";
    if (~~(rock ofclass Treasure))
        print "The rock is not a treasure.^";
];
```

The compiler does not provide a keyword `notofclass` for use in source
code. The negated form is represented internally but must be expressed with
`~~` or by inverting the logic in source.

### 4.7.4 `provides` and `not provides`

Test whether an object defines (or does not define) a given property.

```inform6
Object lamp "brass lamp"
  with brightness 3;

Object rock "ordinary rock";

[ Main;
    if (lamp provides brightness)
        print "Lamp has brightness property.^";
    if (~~(rock provides brightness))
        print "Rock has no brightness property.^";
];
```

Like `ofclass`, the negated form of `provides` exists internally but has
no dedicated keyword. Use `~~` to negate.

### 4.7.5 Using `or` with Condition Operators

All condition operators work with `or` to test multiple values:

```inform6
if (obj has light or open)      ! has light, or has open
if (obj in box or bag or chest) ! in any of the three containers
```

## 4.8 Property Access Operators

Property operators read and inspect the properties of objects. They allow
programs to access property values, property data addresses, and property
data lengths.

### 4.8.1 Property Read (`.`)

The `.` operator reads the current value of a common property from an
object. If the object does not provide the property, the property's default
value is returned.

```inform6
Object door "oak door"
  with description "A heavy oak door.",
       strength 5;

[ Main;
    print door.strength;    ! prints 5
];
```

On the Z-machine, `.` compiles to the `get_prop` instruction.

Property read is also the basis for **message sending**: when a property
holds a routine address, calling `object.property()` invokes that routine
with `self` set to the object (see §4.12.2).

### 4.8.2 Property Data Address (`.&`)

The `.&` operator returns the **byte address** of the raw data stored for
a property on a specific object. If the object does not provide the
property, the result is 0 (false).

```inform6
Object box "box"
  with contents 10 20 30;

[ Main  addr;
    addr = box.&contents;
    if (addr) print "Property exists at address ", addr, ".^";
];
```

On the Z-machine, `.&` compiles to the `get_prop_addr` instruction. The
address points into the object's property table in dynamic memory.

### 4.8.3 Property Data Length (`.#`)

The `.#` operator returns the **length in bytes** of the data stored for a
property on a specific object. Combined with `.&`, it allows iteration
over multi-word property values.

```inform6
Object box "box"
  with contents 10 20 30;

[ ShowContents obj  addr len i;
    addr = obj.&contents;
    len  = obj.#contents;
    if (addr == 0) return;
    for (i = 0 : i < len / WORDSIZE : i++)
        print (addr-->i), " ";
];

[ Main;
    ShowContents(box);     ! prints "10 20 30 "
];
```

On the Z-machine, `.#` compiles to a sequence of `get_prop_addr` followed
by `get_prop_len`. If the property address is 0 (property not provided),
the result is 0.

## 4.9 Array Access Operators

Inform 6 provides two array-access operators for reading from and writing
to memory at computed addresses.

### 4.9.1 Byte Access (`->`)

The `->` operator reads a single **byte** from memory. The left operand
is a base address and the right operand is a byte offset.

```inform6
Array flags -> 10;       ! 10-byte array

[ Main;
    flags->0 = 1;         ! set first byte to 1
    flags->9 = 255;       ! set last byte to 255
    print flags->0;       ! prints 1
];
```

On the Z-machine, `->` compiles to the `loadb` (load byte) instruction.
Values read with `->` are always in the range 0–255.

### 4.9.2 Word Access (`-->`)

The `-->` operator reads a full **machine word** from memory. The left
operand is a base address and the right operand is a word index (not a
byte offset).

```inform6
Array scores --> 5;       ! 5-word array

[ Main;
    scores-->0 = 100;
    scores-->4 = 999;
    print scores-->0;     ! prints 100
];
```

On the Z-machine, `-->` compiles to the `loadw` (load word) instruction.
Each element occupies `WORDSIZE` bytes (2 on Z-machine, 4 on Glulx), and
the index is scaled accordingly.

### 4.9.3 Array Operators as L-values

Both `->` and `-->` can appear on the left side of assignments, and with
increment/decrement operators:

```inform6
Array data --> 10;

[ Main;
    data-->3 = 42;         ! word assignment
    data-->3++;             ! post-increment
    ++data-->3;             ! pre-increment
    data-->3--;             ! post-decrement
    --data-->3;             ! pre-decrement
    print data-->3;         ! prints 42 (net zero change after inc/dec)
];
```

The compiler translates these into combined load/store sequences using the
appropriate byte (`storeb`) or word (`storew`) store instructions.

## 4.10 The Superclass Operator (`::`)

The `::` operator accesses a **property value as defined by a specific
class**, bypassing the object's own definition. It has the highest
precedence of any operator (level 13).

```inform6
Class Vehicle
  with max_speed 60;

Class SportsCar
  class Vehicle
  with max_speed 200;

SportsCar ferrari "Ferrari";

[ Main;
    print ferrari.max_speed;              ! prints 200 (own value)
    print ferrari.Vehicle::max_speed;     ! prints 60  (inherited value)
];
```

The syntax is `object.Class::property`, which reads the value of
`property` as defined by `Class` rather than as overridden by the object
or a derived class. This is particularly useful when a derived class
wants to invoke the superclass's version of a method:

```inform6
Class Animal
  with speak [; print "...^"; ];

Class Dog
  class Animal
  with speak [;
      print "Woof! ";
      self.Animal::speak();    ! call the superclass method
  ];
```

The compiler implements `::` using the `RA__Sc` veneer routine, which
searches the class hierarchy for the property value belonging to the
specified superclass.

## 4.11 Assignment Operators

### 4.11.1 Simple Assignment (`=`)

The `=` operator stores a value into a variable, property, or array
element. It has precedence 1 (the lowest of any operator except the comma)
and is **right-associative**, allowing chained assignments:

```inform6
[ Main  a b c;
    a = b = c = 0;       ! assigns 0 to c, then b, then a
];
```

The result of an assignment expression is the assigned value, which is
why chaining works.

### 4.11.2 Increment and Decrement (`++`, `--`)

Inform 6 provides both prefix and postfix forms of increment and decrement.
All require their operand to be an **l-value** (a variable, array element,
or property).

| Operator | Form | Returns | Precedence |
| -------- | ---- | ------- | ---------- |
| `++x` | Prefix increment | New value (after increment) | 9 |
| `x++` | Postfix increment | Old value (before increment) | 9 |
| `--x` | Prefix decrement | New value (after decrement) | 9 |
| `x--` | Postfix decrement | Old value (before decrement) | 9 |

```inform6
[ Main  x y;
    x = 5;
    y = ++x;     ! x becomes 6, y is 6
    y = x++;     ! y is 6 (old value), x becomes 7
    y = --x;     ! x becomes 6, y is 6
    y = x--;     ! y is 6 (old value), x becomes 5
];
```

Increment and decrement work on array elements and properties as well as
simple variables:

```inform6
Array counters --> 5;

[ Main;
    counters-->2 = 10;
    counters-->2++;        ! now 11
    ++counters-->2;        ! now 12
];
```

```inform6
Object lamp "lamp"
  with brightness 3;

[ Main;
    lamp.brightness++;     ! now 4
    --lamp.brightness;     ! back to 3
];
```

The compiler translates property increments and decrements into calls to
veneer routines that handle the load-modify-store sequence for the
appropriate property type.

### 4.11.3 Compound Assignment Through Properties and Arrays

Inform 6 does not provide C-style compound assignment operators like `+=`,
`-=`, `*=`, etc. To achieve the equivalent effect, write the operation
explicitly:

```inform6
Global score;

[ AddScore points;
    score = score + points;        ! no += operator in Inform 6
];
```

The `++` and `--` operators are the only compound modification operators
available.

## 4.12 Function Calls

### 4.12.1 Direct Function Calls

A function call is written as a routine name (or expression evaluating to
a routine address) followed by parenthesized arguments:

```inform6
[ Square n;
    return n * n;
];

[ Main;
    print Square(7);       ! prints 49
];
```

Function calls have precedence 11, placing them higher than most operators
but lower than property selection (precedence 12) and the superclass
operator (precedence 13).

Arguments are passed by value. Extra arguments beyond the routine's
declared locals are silently discarded. Missing arguments cause the
corresponding locals to receive zero.

### 4.12.2 Message Sends

The expression `object.property(args)` is a **message send**: it reads
the property value from the object, treats it as a routine address, sets
`self` to the object, and calls the routine with the given arguments.

```inform6
Object lamp "brass lamp"
  with describe [;
      print "The ", (name) self, " glows softly.^";
  ];

[ Main;
    lamp.describe();       ! self is lamp during the call
];
```

Message sends compile to the `CA__Pr` veneer routine, which handles
property lookup, `self`/`sender` management, and the actual call.

The `.()` syntax (call through common property) and `..()` syntax (call
through individual property) are both supported. The `.()` form is the
standard one; `..()` is used when working directly with individual
properties (a rare, advanced technique).

### 4.12.3 Indirect Function Calls

A function can also be called indirectly through a variable holding a
routine address:

```inform6
[ Greet;
    print "Hello!^";
];

[ Main  fn;
    fn = Greet;
    indirect(fn);          ! calls Greet
];
```

The `indirect()` built-in handles indirect calls with zero arguments.
For calls with arguments, `indirect(fn, arg1, arg2, ...)` passes them
through. On Glulx, the number of arguments that `indirect()` can accept
is essentially unlimited. On the Z-machine, it is limited to the operand
count of the `call_vs2` instruction (up to 7 arguments).

## 4.13 Operator Precedence Table

The following table lists every Inform 6 operator from **lowest** to
**highest** precedence. Operators at the same precedence level are
evaluated according to their stated associativity. The precedence numbers
correspond to the internal levels defined in the compiler's operator table
(`expressc.c`).

| Prec. | Operators | Type | Assoc. | Description |
| ----- | --------- | ---- | ------ | ----------- |
| 0 | `,` | Infix | Left | Comma (sequence) |
| 1 | `=` | Infix | Right | Assignment |
| 2 | `&&` | Infix | Left | Logical AND (short-circuit) |
| 2 | `\|\|` | Infix | Left | Logical OR (short-circuit) |
| 2 | `~~` | Prefix | Right | Logical NOT (see note below) |
| 3 | `==` `~=` | Infix | None | Equality / inequality |
| 3 | `>` `<` `>=` `<=` | Infix | None | Ordering comparisons |
| 3 | `has` `hasnt` | Infix | None | Attribute testing |
| 3 | `in` `notin` | Infix | None | Containment testing |
| 3 | `ofclass` | Infix | None | Class membership testing |
| 3 | `provides` | Infix | None | Property existence testing |
| 4 | `or` | Infix | Left | Multi-value condition (see §4.6) |
| 5 | `+` `-` | Infix | Left | Addition, subtraction |
| 6 | `*` `/` `%` | Infix | Left | Multiplication, division, remainder |
| 6 | `&` | Infix | Left | Bitwise AND |
| 6 | `\|` | Infix | Left | Bitwise OR |
| 6 | `~` | Prefix | Right | Bitwise NOT (see note below) |
| 7 | `->` | Infix | Left | Byte array access |
| 7 | `-->` | Infix | Left | Word array access |
| 8 | `-` (unary) | Prefix | Right | Unary negation |
| 9 | `++` `--` (prefix) | Prefix | Right | Pre-increment, pre-decrement |
| 9 | `++` `--` (postfix) | Postfix | Right | Post-increment, post-decrement |
| 10 | `.&` | Infix | Left | Property data address |
| 10 | `.#` | Infix | Left | Property data length (bytes) |
| 11 | `()` | — | Left | Function call |
| 12 | `.` | Infix | Left | Property access / message send |
| 13 | `::` | Infix | Left | Superclass property access |

### 4.13.1 Precedence Notes

**Logical NOT (`~~`) and bitwise NOT (`~`):** Although the table assigns
`~~` precedence 2 (equal to `&&` and `||`) and `~` precedence 6 (equal to
`&` and `|`), both NOT operators are **prefix** unary operators. The
compiler's shift-reduce parser treats them as binding to their entire
right-hand subexpression at their own precedence level. The practical
effect is:

- `~~A && B` is parsed as `~~(A && B)`, as if `~~` had precedence 1.5.
- `~A & B` is parsed as `~(A & B)`, as if `~` had precedence 5.5.

To negate only the first operand, use parentheses: `(~~A) && B` or
`(~A) & B`.

**Non-associative operators:** The comparison and condition operators
(precedence 3) have **no** associativity. Writing `a == b == c` or
`a < b < c` is a compile-time error. Each comparison must be joined with
an explicit logical operator:

```inform6
if (a == b && b == c) ...     ! correct
! if (a == b == c) ...        ! compile-time error
```

**Function call precedence anomaly:** Function calls (precedence 11)
interact specially with property selection (precedence 12). In the
expression `alpha.beta(gamma)`, the `.` binds first (reading `alpha`'s
`beta` property), and the result is called with `gamma` as the argument.
In `beta(gamma).alpha`, the function call binds first, and `.alpha` is
then applied to the return value. The compiler handles this through a
special rule in the precedence comparison: when the left operator is a
function call (precedence 11) and the right operator has precedence
greater than 11, the function call is evaluated first.

**The comma operator:** The comma at precedence 0 is used primarily in
`for` loop headers to separate multiple initializers or update
expressions. It evaluates both operands and yields the value of the right
operand.

## 4.14 Expression Contexts

The Inform 6 compiler evaluates expressions differently depending on the
**context** in which they appear. The compiler defines nine expression
contexts, each with its own rules about what expressions are valid and how
their values are used.

### 4.14.1 Void Context (`VOID_CONTEXT`)

An expression whose value is **discarded**. This is the context for
standalone expression statements.

```inform6
[ Main  x;
    x++;                   ! increment is the side effect; value is discarded
    MyFunction();          ! return value is discarded
    print "hello^";        ! print is in void context
];
```

In void context, the compiler generates code for any side effects but does
not store the expression's result. An expression with no side effects in
void context (such as a bare variable name or literal) triggers the
warning "expression has no side effects."

### 4.14.2 Condition Context (`CONDITION_CONTEXT`)

An expression used to determine the direction of a conditional branch:
the condition in an `if` statement, `while` loop, or `for` test.

```inform6
if (x > 0) ...             ! x > 0 is in condition context
while (count--) ...         ! count-- is in condition context
```

In condition context, the compiler generates branch instructions rather
than computing a value. Comparison and condition operators compile directly
to their corresponding branch opcodes, which is more efficient than
computing a boolean value and then branching on it.

### 4.14.3 Constant Context (`CONSTANT_CONTEXT`)

An expression that must be **fully evaluable at compile time**. Used in
`Constant` directives, array size specifications, and `#if` / `Iftrue`
conditions.

```inform6
Constant SIZE = 10 + 5;    ! 10 + 5 is in constant context
Array table --> SIZE;       ! SIZE is in constant context
```

In constant context, the compiler performs constant folding and rejects
any expression containing variables, function calls, or other runtime
constructs. Only literals, previously defined constants, and arithmetic on
constants are permitted.

### 4.14.4 Quantity Context (`QUANTITY_CONTEXT`)

An expression whose **value is needed** as a data quantity. This is the
most general context for subexpressions: the right-hand side of an
assignment, a function argument, a `print` argument, or the operand of an
arithmetic operator.

```inform6
x = a + b;                 ! a + b is in quantity context
MyFunc(x * 2);             ! x * 2 is in quantity context
print score;               ! score is in quantity context
```

In quantity context, the compiler generates code to compute the value and
leave it in a specified result location (a variable or the stack).

### 4.14.5 Action Context (`ACTION_Q_CONTEXT`)

An expression appearing where an **action value** is expected, such as in
angle-bracket action statements:

```inform6
<Take lamp>;               ! Take is in action context
```

In action context, the compiler recognizes bare action names (without the
`##` prefix) as action constants.

### 4.14.6 Assembly Context (`ASSEMBLY_CONTEXT`)

An expression appearing as an operand of an **assembly language**
instruction (following `@`). In this context, the pseudo-variable `sp` is
recognized for direct stack manipulation.

```inform6
@add x 1 -> y;             ! x, 1, and y are in assembly context
@push sp;                   ! sp is recognized in assembly context
```

### 4.14.7 Array Context (`ARRAY_CONTEXT`)

An expression appearing in an **array initializer** list. Array context is
similar to constant context, but additionally permits certain forward
references that are resolved by back-patching.

```inform6
Array rooms --> Kitchen Hallway Garden;   ! each name is in array context
```

### 4.14.8 For-Loop Initialization Context (`FORINIT_CONTEXT`)

The initialization clause of a `for` loop. This context behaves like void
context but allows the parser to recognize the `:` separator that divides
the `for` header into its three clauses.

```inform6
for (i = 0 : i < 10 : i++) ...   ! i = 0 is in for-init context
```

### 4.14.9 Return Context (`RETURN_Q_CONTEXT`)

An expression appearing in a `return` statement. This context behaves like
quantity context and produces the routine's return value.

```inform6
[ Double n;
    return n * 2;           ! n * 2 is in return context
];
```

### 4.14.10 Context Effects on Compilation

The distinction between contexts has practical consequences:

1. **Condition vs. quantity:** In condition context, comparisons compile
   directly to branch instructions. In quantity context, the same
   comparison must produce a value (0 or 1), which requires additional
   instructions. Writing `x = (a > b)` forces the compiler to generate a
   less efficient sequence than `if (a > b) x = 1; else x = 0;`.

2. **Void context warnings:** The compiler can detect useless expressions
   in void context—those with no side effects—and warn about them. This
   catches common mistakes like writing `x == 5;` (comparison, no effect)
   when `x = 5;` (assignment) was intended.

3. **Constant context restrictions:** Only constant context prohibits
   runtime constructs entirely. All other contexts permit any valid
   expression, though the generated code differs.

---

*Copyright © 2026 Software Freedom Conservancy, Inc.*
*Licensed under the GNU General Public License, version 3 or later.*
