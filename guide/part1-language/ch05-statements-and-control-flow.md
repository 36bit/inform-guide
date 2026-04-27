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

# Chapter 5: Statements and Control Flow

This chapter describes every statement form recognized by the compiler and
the control-flow constructs that determine the order in which statements
execute.

## 5.1 Expression Statements

Any valid expression followed by a semicolon is a statement. The compiler
evaluates the expression for its side effects and discards the result.

```inform6
[ Main x;
    x = 42;              ! assignment expression
    x++;                 ! increment expression
    MyRoutine(x);        ! function call expression
    x + 1;               ! legal but pointless — result discarded
];
```

An expression statement is the most common statement form. Assignments,
function calls, and increment/decrement operations are all expressions in
Inform 6, so they become statements simply by appending a semicolon.

## 5.2 Compound Statements (Blocks)

Braces group a sequence of statements into a single **compound statement**
(or **block**). Any place the grammar expects one statement, a block may
appear instead.

```inform6
if (x > 0) {
    print "x is positive.^";
    x = 0;
    ResetState();
}
```

Blocks may be nested to any depth. A block may also be empty:

```inform6
while (Advance()) { }   ! loop until Advance() returns false
```

## 5.3 `if` / `else`

The `if` statement executes a statement (or block) only when a condition is
true, optionally followed by an `else` clause for the false case.

```inform6
if (score > 100)
    print "You win!^";
```

```inform6
if (door has open) {
    print "The door is already open.^";
} else {
    give door open;
    print "You open the door.^";
}
```

### 5.3.1 Dangling `else`

When `if` statements are nested without braces, an `else` binds to the
**nearest preceding unmatched `if`**. This is the classic "dangling else"
rule.

```inform6
! The else belongs to the inner if, not the outer one.
if (x > 0)
    if (x > 100)
        print "Large.^";
    else
        print "Small but positive.^";
```

To associate the `else` with the outer `if`, use braces:

```inform6
if (x > 0) {
    if (x > 100)
        print "Large.^";
} else {
    print "Non-positive.^";
}
```

### 5.3.2 `if` with `rtrue` / `rfalse` Shorthand

A common Inform idiom combines `if` with an immediate return:

```inform6
if (x == 0) rfalse;     ! return false immediately
if (x > 10) rtrue;      ! return true immediately
```

## 5.4 `switch`

The `switch` statement provides multi-way branching based on the value of
an expression. Unlike C, each case is **independent** — there is no
fall-through from one case to the next.

```inform6
switch (colour) {
    1: print "red";
    2: print "green";
    3: print "blue";
    default: print "unknown";
}
```

### 5.4.1 Multiple Values per Case

A single case arm can match several values separated by commas:

```inform6
switch (ch) {
    'a', 'e', 'i', 'o', 'u':
        print "vowel";
    default:
        print "consonant";
}
```

### 5.4.2 Case Ranges

The `to` keyword specifies an inclusive range of values:

```inform6
switch (n) {
    1 to 5:   print "small";
    6 to 10:  print "medium";
    11 to 99: print "large";
    default:  print "out of range";
}
```

Ranges and individual values may be mixed freely on the same case arm.

### 5.4.3 Case Expressions (Compiler 6.42+)

Starting with compiler version 6.42, case values may be **parenthesized
compile-time constant expressions**, not merely bare literals or named
constants.

```inform6
Constant BASE = 10;

[ ShowRange n;
    switch (n) {
        (BASE - 5) to (BASE + 5):
            print "Near the base.^";
        (BASE * 2):
            print "Double the base.^";
        (BASE | $80):
            print "Base with high bit.^";
        default:
            print "Something else.^";
    }
];
```

The parentheses are required so the compiler can distinguish a case
expression from a bare literal. Note that range endpoints specified with
`to` cannot themselves be parenthesized expressions; only individual case
values support this syntax.

### 5.4.4 No Fall-Through

Each case is independent. When the statements in a case complete, execution
continues after the closing brace of the `switch` — there is no fall-through
to the next case as in C, so no `break` is needed at the end of a case. (A
`break` inside a case is still legal and jumps past the closing brace, as
described in 5.9; it is simply rarely needed.)

```inform6
switch (command) {
    1: print "one";    ! after printing, jumps past the switch
    2: print "two";    ! independent — never reached from case 1
}
```

### 5.4.5 `default`

The `default` case matches any value not matched by the preceding cases. It
is optional and, if present, must be the last case in the switch body.

## 5.5 `for` Loops

The `for` loop in Inform 6 follows the same three-part structure as in C,
but uses **colons** (`:`) to separate the parts, not semicolons.

```
for ( initializer : condition : update ) statement
```

> **Warning:** Using semicolons instead of colons is a common mistake for
> programmers coming from C. The compiler will accept semicolons with a
> warning, but the correct separator is the colon.

### 5.5.1 Basic `for` Loop

```inform6
[ CountToTen i;
    for (i = 1 : i <= 10 : i++)
        print i, " ";
    new_line;
];
```

### 5.5.2 Omitting Parts

Any of the three parts may be omitted. An omitted condition is treated as
always true, creating a loop that runs until a `break`, `return`, or other
transfer of control.

```inform6
for (::)               ! infinite loop — all three parts empty
    if (finished) break;

for (i = 0 : i < n :)  ! no update — must be handled in the body
    i = NextPrime(i);

for (: i > 0 : i--)    ! no initializer
    ProcessItem(i);
```

An entirely empty specification `for ()` is equivalent to `for (::)` and
produces an infinite loop.

### 5.5.3 Complex Initializers

The initializer part may contain only a single expression statement. To
initialize multiple variables, use a helper routine or initialize them
before the loop:

```inform6
[ PairLoop i j;
    i = 0;
    j = 100;
    for (: i < j : i++, j--) {
        print i, " ", j, "^";
    }
];
```

Note that the update part may use the comma operator to perform several
updates.

## 5.6 `while` Loops

The `while` loop tests its condition **before** each iteration. If the
condition is initially false, the body never executes.

```inform6
[ DrainPool pool;
    while (pool > 0) {
        print "Draining...^";
        pool = pool - 10;
    }
];
```

## 5.7 `do` ... `until`

The `do`–`until` loop tests its condition **after** each iteration, so the
body always executes at least once. Note that the keyword is `until`, not
`while` — the loop repeats *until* the condition becomes true.

```inform6
[ GetInput ch;
    do {
        print "Enter Y or N: ";
        ch = ReadKeyPress();
    } until (ch == 'Y' or 'N');
    return ch;
];
```

The condition is enclosed in parentheses and the construct ends with a
semicolon after the closing parenthesis.

> **Contrast with C:** In C the equivalent construct is `do { ... }
> while (condition)` with positive logic. Inform's `until` uses negative
> logic: the loop exits when the condition is **true**.

## 5.8 `objectloop`

The `objectloop` statement iterates over objects in the game's object tree.
It has several forms, each selecting a different subset of objects.

### 5.8.1 All Objects

The simplest form iterates over every object in the game:

```inform6
[ CountItems count obj;
    count = 0;
    objectloop (obj)
        if (obj has takeable)
            count++;
    return count;
];
```

The loop variable visits objects in internal numbering order (the order they
were defined in source code, approximately).

### 5.8.2 Children of a Parent (`in`)

To iterate over the immediate children of a specific parent object:

```inform6
[ ListContents container obj;
    print "The ", (name) container, " contains:^";
    objectloop (obj in container) {
        print "  ", (a) obj, "^";
    }
];
```

### 5.8.3 Scope Proximity (`near`)

To iterate over objects that are "near" another object — that is, objects
sharing the same parent:

```inform6
[ ShowNearby obj x;
    objectloop (x near obj)
        print (The) x, " is nearby.^";
];
```

### 5.8.4 Starting Object (`from`)

To iterate starting from a particular object, walking through its siblings:

```inform6
[ ListSiblings start obj;
    objectloop (obj from start)
        print (The) obj, "^";
];
```

The loop variable is set to `start` and then advances through the sibling
chain of `start`'s parent, ending when the chain runs out.

### 5.8.5 Class Membership (`ofclass`)

To iterate over instances of a particular class:

```inform6
Class Weapon;
Weapon -> sword "sword" with name 'sword';
Weapon -> axe "axe" with name 'axe';

[ ListWeapons w;
    objectloop (w ofclass Weapon)
        print (The) w, "^";
];
```

### 5.8.6 Attribute Test (`has`)

To iterate over objects that possess a particular attribute:

```inform6
[ CountLitObjects count obj;
    count = 0;
    objectloop (obj has light)
        count++;
    return count;
];
```

### 5.8.7 Property Test (`provides`)

To iterate over objects that define a given property (whether common or
individual):

```inform6
[ CallEveryDescribe obj;
    objectloop (obj provides describe)
        obj.describe();
];
```

The `provides` filter visits every object whose property table or
inherited-class chain supplies the named property; absence of the
property simply skips the object.

### 5.8.8 General Condition Form

A general boolean condition may be placed after the loop variable. The loop
iterates over all objects and executes the body only for those where the
condition is true:

```inform6
objectloop (obj has edible && obj in player)
    print (The) obj, " looks tasty.^";
```

### 5.8.9 Modifying the Object Tree During `objectloop`

For an `objectloop (x in parent)`, after each iteration of the body the
compiler advances the loop variable by taking *the sibling of the current
value of `x`*. This means modifying the object tree during iteration
needs care:

- If the body `move`s `x` somewhere else, the next iteration will visit
  whatever sibling `x` now has under its new parent — almost certainly
  not what you wanted.
- If the body `remove`s `x`, its sibling is `0` (nothing), so the loop
  ends early.
- Moving or removing *other* children of the same parent during the loop
  can also produce unpredictable results.

If you need to modify the tree while iterating, a robust pattern is to
first collect the relevant objects into a temporary list and then process
that list.

## 5.9 `break` and `continue`

The `break` statement exits the innermost enclosing loop immediately.
The `continue` statement skips the remainder of the current iteration and
jumps to the loop's next test (or update, in a `for` loop).

```inform6
[ FindFirst arr len i;
    for (i = 0 : i < len : i++) {
        if (arr-->i == 0) continue;    ! skip zero entries
        if (arr-->i == -1) break;      ! stop at sentinel
        ProcessEntry(arr-->i);
    }
];
```

Both `break` and `continue` apply only to loops and (in the case of
`break`) `switch` blocks. Using them outside a loop or `switch` body
produces a compile-time error. In a `switch`, `break` jumps past the
closing brace — the same effect as reaching the end of a case, so it is
rarely needed. `continue` applies only to loops, not to `switch`.

## 5.10 `return`, `rtrue`, `rfalse`

These statements exit the current routine and return a value to the caller.

### 5.10.1 `return`

The `return` statement returns an explicit value:

```inform6
[ Max a b;
    if (a > b) return a;
    return b;
];
```

`return` with no expression is equivalent to `rtrue` — it always returns
`true` (1), regardless of what kind of routine you are in. The
"return-true-by-default" versus "return-false-by-default" distinction
applies only when execution reaches the end of a routine *without* an
explicit `return`: stand-alone routines implicitly return `true`, while
embedded routines (such as those used as property values; see Chapter 6)
implicitly return `false`.

### 5.10.2 `rtrue` and `rfalse`

These are shorthand for `return true` and `return false`:

```inform6
[ IsAdult age;
    if (age >= 18) rtrue;
    rfalse;
];
```

The compiler optimizes these into single Z-machine instructions.

## 5.11 `print` Statement

The `print` statement is one of the most versatile constructs in Inform 6.
It accepts a comma-separated list of **print items**, each of which may be
a string literal, a numeric expression, or a format specifier.

### 5.11.1 String Literals

A double-quoted string prints its text literally. The caret character `^`
in a string translates to a newline:

```inform6
print "Hello, world!^";       ! prints: Hello, world! (then newline)
print "Line 1.^Line 2.^";    ! prints two lines
```

### 5.11.2 Integer Expressions

A bare expression (not in quotes) prints as a signed decimal integer:

```inform6
print "The answer is ", 6 * 7, ".^";   ! prints: The answer is 42.
```

### 5.11.3 Format Specifiers

Parenthesized keywords control how a value is printed:

| Specifier | Meaning |
| --------- | ------- |
| `(number) expr` | Print integer as English words via `EnglishNumber` (e.g., "forty-two") |
| `(char) expr` | Print the character with the given ZSCII/Unicode code |
| `(string) expr` | Print the string at the given packed address |
| `(address) expr` | Print the string at the given byte address (dictionary word) |
| `(name) expr` | Print the short name of an object |
| `(object) expr` | Print the short name of an object (low-level, no article) |
| `(a) expr` | Print the object name with an indefinite article ("a sword") |
| `(an) expr` | Synonym for `(a)` |
| `(A) expr` | Print the object name with a capitalized indefinite article ("A sword") |
| `(the) expr` | Print the object name with a definite article ("the sword") |
| `(The) expr` | Print with a capitalized definite article ("The sword") |
| `(property) expr` | Print the name of a property (debugging aid) |

```inform6
[ Describe obj;
    print "You see ", (a) obj, " here. ";
    print (The) obj, " looks interesting.^";
];
```

### 5.11.4 Custom Print Rules

A routine name in parentheses calls that routine with the expression as its
single argument. The routine should print something and return:

```inform6
[ BoldPrint txt;
    style bold;
    print (string) txt;
    style roman;
];

[ Main;
    print "This is ", (BoldPrint) "important", " text.^";
];
```

This mechanism allows user-defined formatting and is used extensively by the
standard library for printing object names.

### 5.11.5 Multiple Print Items

Items are separated by commas. The compiler generates code to print each
item in sequence:

```inform6
print "Score: ", score, " / ", max_score, " (", (number) score, ")^";
```

## 5.12 `print_ret`

The `print_ret` statement prints its arguments (exactly like `print`),
then prints a newline, and then returns `true`:

```inform6
[ Greet name;
    print_ret "Hello, ", (string) name, "!";
    ! equivalent to: print "Hello, ..."; new_line; rtrue;
];
```

A bare double-quoted string as a statement is syntactic sugar for
`print_ret`:

```inform6
[ Greet;
    "Hello, adventurer!";
    ! identical to: print_ret "Hello, adventurer!";
];
```

## 5.13 `new_line`

The `new_line` statement prints a newline character. It is equivalent to
`print "^"` but generates a single opcode:

```inform6
print "First line.";
new_line;
print "Second line.";
new_line;
```

## 5.14 `spaces`

The `spaces` statement prints a given number of space characters:

```inform6
[ Indent depth;
    spaces depth * 4;
    print "Indented text.^";
];
```

The argument is any expression that evaluates to a non-negative integer. If
the value is zero or negative, no spaces are printed.

## 5.15 `box`

The `box` statement displays a text box centred on the screen, typically
used for title screens or quotations. Each argument is a string that
becomes one line of the box:

```inform6
[ ShowTitle;
    box "The Lurking Horror"
        "An Interactive Fiction"
        "by Dave Lebling";
];
```

The `box` statement is a Z-machine feature and uses the upper window. On
Version 3 the statement is ignored. The box is typically displayed briefly
and then erased.

## 5.16 `font on/off` and `style`

These statements control text presentation.

### 5.16.1 `font on` / `font off`

Switches between the normal proportional font and a fixed-width font:

```inform6
font off;
print "  FIXED WIDTH TEXT  ^";
font on;
print "Back to normal.^";
```

On Z-machine Version 3, `font off` has no effect. On Version 5+, it maps
to the `@set_font` opcode.

### 5.16.2 `style`

The `style` statement selects a text style. Available styles are:

| Statement | Effect |
| --------- | ------ |
| `style roman` | Normal text (cancels all styles) |
| `style bold` | **Bold** text |
| `style underline` | Underlined text |
| `style reverse` | Reverse video (swapped foreground/background) |
| `style fixed` | Fixed-width (monospace) font |

```inform6
[ Emphasize;
    style bold;
    print "Important!";
    style roman;
    new_line;
];
```

Styles may be combined on some interpreters, but portable code should use
only one style at a time, resetting with `style roman` between changes.

The `style` statement is available from Z-machine Version 4 onwards and
compiles to the `@set_text_style` opcode.

## 5.17 `inversion`

The `inversion` statement prints the compiler version string that was
embedded in the story file header at compile time:

```inform6
[ ShowVersion;
    print "Compiled with Inform ";
    inversion;
    new_line;
];
```

This reads bytes 60–63 of the story file header (the "Inform version"
field).

## 5.18 `read`

The `read` statement reads a line of player input into buffer and parse
arrays:

```inform6
[ GetCommand buffer parse;
    read buffer parse;
];
```

The first argument is the **text buffer** array (to receive the raw typed
characters), and the second is the **parse buffer** array (to receive the
tokenized words).

### 5.18.1 Version-Dependent Behavior

On Z-machine Version 3, `read` also updates the status line before
accepting input. The statement form of `read` does not yield a value; to
capture the terminating character on Version 5+ (where the underlying
`@aread` opcode stores a result), use the assembly form:

```inform6
[ GetInput buffer parse result;
    @aread buffer parse -> result;
    if (result == 13)
        print "(Pressed Enter.)^";
];
```

### 5.18.2 Status-Line Routine

On Version 4+, `read` can accept a third argument — a routine to call
immediately before accepting input. On Version 4 this is typically used
to redraw the status line (since Version 4 does not update it
automatically the way Version 3 does):

```inform6
read buffer parse DrawStatusLine;
```

The routine is called once each time the `read` statement executes, just
before the interpreter begins waiting for input. For true timed input
(periodic callbacks while waiting), use the `@aread` assembly instruction
with its time-interval and routine operands.

## 5.19 `give`

The `give` statement sets or clears attributes on an object. To set an
attribute, name it directly. To clear an attribute, prefix it with `~`:

```inform6
give lamp light;         ! set the 'light' attribute
give lamp ~light;        ! clear the 'light' attribute
```

Multiple attributes can be set or cleared in a single `give` statement,
separated by spaces:

```inform6
give treasure light ~concealed scored;
! sets light, clears concealed, sets scored
```

## 5.20 `move` and `remove`

These statements manipulate the object tree, the central data structure
that represents spatial containment in an Inform game.

### 5.20.1 `move`

The `move` statement places an object as a child of another object:

```inform6
move sword to player;        ! player now carries the sword
move player to DarkRoom;     ! player enters DarkRoom
```

The keyword `to` is required. If the object is already a child of another
parent, it is first removed from its current parent before being inserted
under the new one.

### 5.20.2 `remove`

The `remove` statement detaches an object from the object tree entirely,
making it parentless:

```inform6
remove sword;    ! the sword is now nowhere — not in any room or container
```

A removed object has no parent, no siblings, and retains its children (if
any). It can be reinserted later with `move`.

## 5.21 `jump` and Labels

The `jump` statement transfers control unconditionally to a named label.
Labels are declared with a dot prefix and terminated with a semicolon:

```inform6
[ Example x;
    if (x == 0) jump skip;
    print "x is not zero.^";
    .skip;
    print "Done.^";
];
```

Labels are local to the enclosing routine. The name after `jump` must match
a label declared (with `.`) somewhere in the same routine. Forward
references are permitted — you can jump to a label that appears later in
the source.

> **Style note:** The use of `jump` and labels is generally discouraged in
> favor of structured control flow (`if`, `while`, loops). They exist
> primarily for low-level code and as a compilation target for macros.

## 5.22 `quit`, `save`, `restore`

These statements manage the game session itself.

### 5.22.1 `quit`

The `quit` statement terminates the program immediately:

```inform6
[ GiveUp;
    print "Goodbye!^";
    quit;
];
```

### 5.22.2 `save` and `restore`

The `save` statement saves the entire game state to a file. The `restore`
statement loads a previously saved state. Both take a label to jump to on
success (on Z-machine versions below 5) or return a value indicating
success or failure (on version 5+):

```inform6
[ DoSave flag;
    @save -> flag;       ! Z-machine V5+: result in flag
    if (flag == 0) {
        print "Save failed.^";
        rfalse;
    }
    print "Saved (or just restored).^";
    rtrue;
];
```

On Z-machine Version 5+, `save` and `restore` are typically used via their
assembly-language forms (`@save`, `@restore`) to capture the return value.
The statement forms (`save label;`, `restore label;`) use branching
semantics from earlier Z-machine versions.

## 5.23 `string`

The `string` statement assigns a string value to one of the Z-machine's
dynamic string variables (low strings). These are numbered slots that can
be referenced in printed text:

```inform6
[ Main;
    string 0 "world";
    print "Hello, @00!^";   ! prints: Hello, world!
    string 0 "Inform";
    print "Hello, @00!^";   ! prints: Hello, Inform!
];
```

The first argument is the slot number. By default, slots `0` through `31`
are available (this is the default value of the compiler setting that
controls the dynamic-string table size). The maximum number of slots can
be raised — up to 96 in Z-code — by setting `$MAX_DYNAMIC_STRINGS` higher.
The second argument is a string literal. The `@00` notation in printed
strings expands to whatever string is currently assigned to slot 0.

Dynamic strings allow text to change at runtime without rebuilding the
string. This is useful for the player's name, gender-specific pronouns, and
other variable text.

## 5.24 Assembly Language (`@`)

Inform 6 allows direct embedding of virtual machine instructions using the
`@` prefix. This provides access to low-level operations not exposed by
any high-level statement.

### 5.24.1 Z-Machine Assembly

A Z-machine assembly instruction has the form:

```
@ opcode operand1 operand2 ... ;
```

#### Store Instructions

Instructions that produce a result use `->` to specify the destination
variable:

```inform6
@ mul 6 7 -> result;         ! result = 6 * 7
@ random 100 -> roll;        ! roll = random number 1–100
@ loadw table 0 -> value;    ! value = table-->0
```

#### Branch Instructions

Instructions that branch use `?` for "branch if true" and `?~` for "branch
if false", followed by a label name:

```inform6
@ je x 0 ?is_zero;           ! jump to .is_zero if x == 0
@ jg x 100 ?~not_big;        ! jump to .not_big if x <= 100
```

The special labels `rtrue` and `rfalse` may be used as branch targets:

```inform6
@ je x 0 ?rtrue;              ! return true if x == 0
```

#### Combined Store and Branch

Some instructions both store a result and branch:

```inform6
@ scan_table val table len -> result ?found;
.found;
print "Found at address ", result, "^";
```

#### Text Instructions

Certain opcodes (such as `@print`) embed literal text directly:

```inform6
@ print "Hello from assembly.^";
```

### 5.24.2 Glulx Assembly

Glulx assembly uses the same `@` prefix but with Glulx opcode names and
conventions:

```inform6
@ add 10 20 result;           ! result = 10 + 20 (store is last operand)
@ random 100 roll;            ! roll = random number in 0–99
```

In Glulx, the store destination is the **last operand** rather than being
preceded by `->`.

Branch syntax for Glulx uses `?` similarly:

```inform6
@ jeq x 0 ?is_zero;           ! jump if x == 0
```

### 5.24.3 Custom Opcode Encoding

For opcodes not known to the compiler, a string descriptor specifies the
format. For the Z-machine:

```
@ "TYPE:CODE" operands ...
```

Where `TYPE` is the opcode class (`0OP`, `1OP`, `2OP`, `VAR`, `EXT`,
`VAR_LONG`, `EXT_LONG`) and `CODE` is the decimal opcode number. Optional
flags may follow: `S` (stores), `B` (branches), `T` (text), `I` (indirect
variable reference).

```inform6
@ "2OP:20" x y -> result;     ! Z-machine opcode 20 (2-operand, stores)
```

For Glulx:

```
@ "FlagsCount:CodeNum" operands ...
```

Where `Flags` includes `S` (store), `SS` (two stores), `B` (branch), `R`
(execution never continues after the opcode), and `Count` is the number
of operands (0–9).

```inform6
@ "S3:123" a b c result;      ! Glulx: 3-arg opcode 123 with store
```

### 5.24.4 Inline Byte and Word Data

Assembly directives can inject raw bytes or words into the instruction
stream:

```inform6
@ -> $8B $04 $D2;              ! insert literal bytes
@ --> $00FF $1234;             ! insert literal words (big-endian)
```

This is an advanced feature introduced in compiler 6.42, useful for
encoding custom opcodes or data tables directly in the output.

### 5.24.5 Indirect Variable References

Some Z-machine opcodes take "indirect" variable references — they operate
on a variable whose number is given by an operand. The compiler marks these
with square brackets:

```inform6
@ pull [result];               ! pull from stack into variable #result
@ inc [counter];               ! increment variable #counter
```

### 5.24.6 When to Use Assembly

Assembly is appropriate for:

- Accessing Z-machine or Glulx features with no high-level equivalent
  (e.g., `@set_colour`, `@gestalt`, `@glk`).
- Performance-critical inner loops where high-level overhead matters.
- Implementing functionality that targets a specific virtual machine
  version.
- Interfacing with interpreter extensions or non-standard opcodes.

For most game programming, high-level statements are preferred.
