# Chapter 3: Variables and Scope

This chapter describes how Inform 6 stores and retrieves named data: how
variables are declared, what limits each target platform imposes, how names
resolve when locals and globals share the same identifier, and how
compile-time constants differ from runtime storage. Understanding these
rules is essential for writing correct programs, particularly when moving
code between the Z-machine and Glulx or when debugging name-shadowing
problems.

## 3.1 Global Variables

A **global variable** is a named storage location accessible from every
routine in a program. Globals are declared at the top level of source with
the `Global` directive, outside any routine or object definition.

### 3.1.1 The `Global` Directive

The basic syntax is:

```inform6
Global score;
Global turns;
Global max_carried = 100;
```

Each `Global` directive introduces one variable name. An optional
initializer, preceded by `=` or simply a space, sets the value the variable
will hold when the program starts. If no initializer is given, the
variable starts at zero.

```inform6
Global location = Kitchen;      ! initialized to the object Kitchen
Global darkness_flag = 1;       ! initialized to integer 1
Global saved_score;             ! initialized to 0
```

As of compiler version 6.40, the `=` sign is optional when giving an
initial value. The following two declarations are equivalent:

```inform6
Global foo = 12;
Global foo 12;
```

This brings `Global` into harmony with `Constant`, for which `=` has
always been optional.

As of compiler version 6.43, it is safe to declare a global variable more
than once. If a global is declared more than once, at most one declaration
should give an initial value. It is also safe to give each declaration the
*same* initial value, as long as the value is a compile-time constant.

```inform6
Global shared_flag;             ! first declaration, no initial value
Global shared_flag;             ! redeclaration permitted since 6.43
```

### 3.1.2 Z-Machine Limits

> **[Z-machine]** The Z-machine provides exactly **240** global variable
> slots (numbered 16–255 in the underlying architecture). Seven of these
> are reserved by the compiler for internal use (temporaries, `self`,
> `sender`, and `sw__var`), leaving approximately **233** slots for
> user-declared globals. Attempting to declare more produces a fatal
> compile-time error.

The hard limit of 240 is defined by the Z-machine specification: global
variables occupy entries 16–255 of the variable space (entries 0–15 are
local variables). The compiler constant `MAX_ZCODE_GLOBAL_VARS` records
this limit.

### 3.1.3 Glulx Limits

> **[Glulx]** Glulx places no fixed upper bound on the number of global
> variables. The compiler allocates them dynamically, so a program may
> declare as many globals as memory permits.

In practice, programs rarely approach even the Z-machine limit. The
absence of a ceiling on Glulx simplifies ports of large Z-machine programs
that previously needed to pack data into arrays to conserve global slots.

### 3.1.4 Global Variable Storage

Internally, the compiler indexes every variable—local and global—with a
single numbering scheme. Local variables occupy indices 0 through
`MAX_LOCAL_VARIABLES − 1`, and globals occupy indices starting at
`MAX_LOCAL_VARIABLES`. On the Z-machine, `MAX_LOCAL_VARIABLES` is 16, so
the first user global is index 16. On Glulx it is 119, so the first user
global is index 119.

In the Z-machine, the values of all global variables are stored in a
contiguous 480-byte table (240 words of 2 bytes each) located at the start
of dynamic memory. This table is part of the game file's header-specified
globals area. In Glulx, global variables are stored in a RAM segment whose
size adjusts to the number of globals actually declared.

### 3.1.5 Obsolete `Global` Array Syntax

In early versions of Inform, the `Global` directive could also define
arrays using forms like `Global array --> 8;`. As of compiler version
6.40, these obsolete forms have been removed. Use `Global` for variables
and `Array` for arrays.

## 3.2 Local Variables

**Local variables** exist only within the routine in which they are
declared. They are introduced as the parameter list of a routine header
and cease to exist when the routine returns.

### 3.2.1 Declaration Syntax

Local variables are declared in the header of a routine definition, between
the opening bracket `[` and the semicolon `;`:

```inform6
[ MyRoutine x y z;
    ! x, y, z are local variables
    x = 10;
    y = x + 1;
    z = x * y;
    return z;
];
```

Every name listed before the semicolon is a local variable. When the
routine is called, the first local receives the first argument, the second
local receives the second argument, and so on. Locals for which no
argument was passed are initialized to zero.

```inform6
[ AddThree a b c;
    return a + b + c;
];

[ Main;
    print AddThree(10, 20, 30);    ! prints 60
    print AddThree(10, 20);        ! prints 30 (c is 0)
    print AddThree(10);            ! prints 10 (b and c are 0)
    print AddThree();              ! prints 0  (all are 0)
];
```

### 3.2.2 Default Values for Locals

Inform 6 does not provide syntax for giving local variables default
values in the declaration. Every local variable begins life as zero unless
a corresponding argument is passed by the caller. If you need a local
initialized to a non-zero value, assign it at the start of the routine
body:

```inform6
[ SearchRadius dist;
    if (dist == 0) dist = 5;      ! "default" of 5 when no arg passed
    ! ... use dist ...
];
```

A common idiom uses extra local variables that are never intended to
receive arguments, treating them as scratch storage:

```inform6
[ Process  arg   i len tmp;
    ! 'arg' is an actual parameter; 'i', 'len', 'tmp' are scratch locals
    for (i = 0 : i < 10 : i++) {
        tmp = arg + i;
        ! ...
    }
];
```

There is no syntactic distinction between a parameter and a scratch local;
the division is purely a matter of convention and documentation.

### 3.2.3 Z-Machine Limits

> **[Z-machine]** Each routine may declare at most **15** local variables.
> The Z-machine architecture allocates local variable slots 1–15 (slot 0
> is reserved for the stack pointer pseudo-variable `sp`). The compiler
> constant `MAX_LOCAL_VARIABLES` is 16, which includes `sp`.

Any attempt to declare more than 15 locals in a Z-machine build produces
a compile-time error. Programs approaching this limit can refactor by
moving scratch variables into globals or by splitting the routine.

### 3.2.4 Glulx Limits

> **[Glulx]** Each routine may declare at most **118** local variables.
> Glulx sets `MAX_LOCAL_VARIABLES` to 119 (including the `sp` slot),
> giving 118 usable locals.

In practice, 118 locals is far more than any routine will use. The higher
limit is a natural consequence of the Glulx architecture's 4-byte local
frame rather than the Z-machine's 1-byte slot indices.

## 3.3 The Stack Pointer (`sp`)

Both the Z-machine and Glulx reserve local variable slot 0 to represent
the top of the evaluation stack. In assembly language, this slot is named
`sp`. It is a pseudo-variable: reading `sp` pops the top value from the
stack, and writing to `sp` pushes a value onto it.

```inform6
[ StackDemo;
    @push 42;           ! push 42 onto the stack
    @push 58;           ! push 58 onto the stack
    @add sp sp -> sp;   ! pop 58 and 42, push 100
    @print_num sp;      ! pop 100 and print it
];
```

The identifier `sp` is recognized **only in assembly-language context**—
that is, within `@`-prefixed instructions. In ordinary Inform 6
expressions, the compiler manages the stack implicitly and the name `sp`
has no special meaning unless you have defined a local or global variable
with that name (which is strongly discouraged).

Because `sp` occupies slot 0, user local variables begin at slot 1. This
is why the maximum number of user-declared locals is one fewer than
`MAX_LOCAL_VARIABLES`.

## 3.4 Scope Rules

Inform 6 uses a simple, two-level scoping model: names are either **local**
(visible only within one routine) or **global** (visible everywhere). There
is no block-level scoping; a local variable declared in a routine header is
visible throughout the entire body of that routine, including nested blocks.

### 3.4.1 Local Variables Shadow Globals

When a local variable has the same name as a global variable, the local
takes precedence within its routine. The global is completely hidden—there
is no syntax for accessing it—for the duration of that routine's execution.

```inform6
Global count = 100;

[ ShowCount;
    print count;             ! prints 100—the global
];

[ ResetCount count;
    ! 'count' here is a local, shadowing the global
    count = 0;
    print count;             ! prints 0—the local
    ShowCount();             ! prints 100—the global is unchanged
];
```

The compiler does not warn about shadowing. This can be a source of subtle
bugs: a routine that intends to modify a global may inadvertently declare a
local with the same name and leave the global untouched.

### 3.4.2 No Block Scoping

Unlike C, Inform 6 does not restrict the visibility of a variable to the
block in which it is introduced. A local declared in the routine header is
accessible in every `if` branch, `for` loop, `while` loop, and `switch`
case within that routine:

```inform6
[ Example x;
    if (1) {
        x = 42;
    }
    print x;    ! prints 42; x is in scope here
];
```

There is no way to declare a variable that is visible only inside a loop
body or conditional branch. All locals share the same flat scope: the
entire routine.

### 3.4.3 Scope of Constants and Other Identifiers

Constants, object names, class names, routine names, and property names are
all **global in scope**. They are visible from the point of declaration
onward through the rest of the source (and, for forward references resolved
by back-patching, sometimes from earlier points as well). There is no
mechanism for restricting a constant or object to a particular module or
file.

## 3.5 Variable Initialization

### 3.5.1 Global Initialization

Every global variable is initialized at compile time. If the `Global`
directive specifies an explicit value, that value is used; otherwise the
default is **0**.

```inform6
Global flags = 0;          ! explicitly zero (same as omitting the value)
Global lamp_brightness = 3; ! explicitly 3
Global turn_count;          ! implicitly 0
```

The initializer is evaluated in constant context: it must be a
compile-time constant expression. Object names, routine addresses, string
addresses, and literal integers are all valid. Expressions involving
runtime computation are not.

```inform6
Object lamp "brass lamp";

Global starting_lamp = lamp;          ! valid: object number is constant
Global greeting = "Hello, world!";    ! valid: string address is constant
```

At runtime, global variable values are mutable. The initial values are
stored in the game file and loaded into memory when the program starts.
Restarting the game resets all globals to their initial values.

### 3.5.2 Local Initialization

Local variables are initialized at **call time**, not at compile time.
When a routine is entered:

1. Each local that corresponds to a passed argument receives the value of
   that argument.
2. All remaining locals are set to **0**.

This happens automatically on every call. There is no way to give a local
a compile-time default other than zero.

```inform6
[ Triple n;
    return n * 3;       ! n holds whatever the caller passed
];

[ Main;
    print Triple(7);    ! n starts as 7, result is 21
    print Triple();     ! n starts as 0, result is 0
];
```

## 3.6 Special Variables

The compiler pre-defines two global variables that carry information
about the current message-sending context.

### 3.6.1 `self`

The variable `self` refers to the **object currently receiving a
message**—that is, the object whose property routine is executing. When a
property routine is called via the `.` (property) or `..` (individual
property) operators, the interpreter sets `self` to the object on the left
side of the operator before entering the routine.

```inform6
Object lamp "brass lamp"
  with brightness 3,
       describe [;
           print "The ", (name) self, " glows softly.^";
           ! 'self' is 'lamp' here
       ];
```

Outside a property routine, `self` retains whatever value it had from the
most recent message send. It is not automatically cleared.

The compiler stores `self` in a reserved global variable slot. In Z-machine
builds, `self` occupies global slot 255 (the last slot). In Glulx builds,
it occupies global index `MAX_LOCAL_VARIABLES + 4` (index 123 with the
default configuration).

### 3.6.2 `sender`

The variable `sender` records the **object that initiated the current
message**. When an object's property routine calls another object's
property, `sender` in the callee is set to the calling object.

```inform6
Object guard "burly guard"
  with react_before [;
      if (sender == player) "The guard eyes you suspiciously.";
  ];
```

Like `self`, `sender` is a compiler-reserved global. It occupies global
slot 249 on the Z-machine and global index `MAX_LOCAL_VARIABLES + 5`
(index 124) on Glulx.

In practice, `sender` is used much less frequently than `self`. Most
programs interact with `self` constantly and with `sender` only in
advanced message-routing patterns.

### 3.6.3 Internal Temporary Variables

The compiler also reserves several global variables for internal
expression evaluation. These are named `temp_global`, `temp__global2`,
`temp__global3`, and `temp__global4`. A further reserved variable,
`sw__var`, is used in the compilation of `switch` statements. These
variables should never be accessed directly by user code; their values are
ephemeral and may change between any two statements.

## 3.7 System-Defined Globals

The standard Inform library declares a number of global variables that
form the backbone of the interactive-fiction world model. These include:

| Variable | Purpose |
| -------- | ------- |
| `location` | The room the player currently occupies |
| `player` | The object representing the player character |
| `actor` | The actor performing the current action |
| `action` | The action currently being processed |
| `noun` | The primary object of the current action |
| `second` | The secondary object (e.g., "put X **in** Y") |
| `inp1`, `inp2` | Raw parsed inputs (before noun resolution) |
| `the_time` | Current game time (in minutes since midnight) |
| `score` | The player's current score |
| `turns` | The number of turns elapsed |
| `deadflag` | Non-zero when the game is over |
| `notify_mode` | Whether to print score-change notifications |
| `inventory_stage` | Controls two-stage inventory listing |
| `real_location` | The player's actual room (may differ from `location` in darkness) |

These variables are ordinary globals declared with `Global` in the library
source files. They have no special status in the compiler; their meaning
comes entirely from the library code that reads and writes them. A full
treatment of library globals appears in Part III.

When writing library-independent code (for example, using an alternative
library such as PunyInform), some of these names may differ or be absent.
The core compiler recognizes only `self`, `sender`, and its internal
temporaries—everything else is library-defined.

## 3.8 Constants

A **constant** is a named value fixed at compile time. Constants cannot be
assigned to at runtime; they exist only in the compiler's symbol table and
are substituted directly into the generated code wherever they appear.

### 3.8.1 The `Constant` Directive

Constants are declared with the `Constant` directive:

```inform6
Constant MAX_SCORE = 350;
Constant STARTING_ROOM = Kitchen;
Constant BANNER_TEXT = "Welcome to Adventure!";
```

The `=` sign is optional:

```inform6
Constant MAX_SCORE 350;           ! equivalent to the above
```

Multiple constants may be declared in a single directive, separated by
commas:

```inform6
Constant NORTH = 1, SOUTH = 2, EAST = 3, WEST = 4;
```

A constant declared with no value receives the value **0**:

```inform6
Constant DEBUG_MODE;              ! value is 0
```

### 3.8.2 Constant Expressions

The value of a constant must be a **compile-time constant expression**—an
expression that the compiler can evaluate fully without running the
program. Valid constant expressions include:

- Integer literals: `42`, `$FF`, `$$10101010`
- Character literals: `'a'`
- String literals: `"hello"`
- Other constant names: `MAX_SCORE`
- Object names: `Kitchen`
- Routine addresses: `MyFunc`
- Arithmetic on constants: `MAX_SCORE + 1`
- The `#` notation for system constants: `#n$word`

Invalid constant expressions include anything that depends on runtime
state, such as global or local variable values, function calls, or
property lookups.

```inform6
Constant DOUBLE_MAX = MAX_SCORE * 2;     ! valid: both operands are constants
Constant TABLE_SIZE = 10 + HEADER_WORDS; ! valid if HEADER_WORDS is a constant
```

### 3.8.3 Predefined Constants

The compiler automatically defines a number of constants that describe the
compilation environment:

| Constant | Value |
| -------- | ----- |
| `TARGET_ZCODE` | Defined when compiling for the Z-machine |
| `TARGET_GLULX` | Defined when compiling for Glulx |
| `WORDSIZE` | 2 on Z-machine, 4 on Glulx |
| `DICT_WORD_SIZE` | Number of bytes per dictionary word |
| `NULL` | 0 (the null value) |
| `true` | 1 |
| `false` | 0 |
| `nothing` | 0 (the null object) |

Additional system constants are accessible through the `#` prefix
notation, such as `#largest_object`, `#dictionary_table`, and
`#grammar_table`. These are described in detail in the virtual-machine
chapters (Part IV).

### 3.8.4 Constants vs. Global Variables

It is important to distinguish constants from globals:

| Feature | `Constant` | `Global` |
| ------- | ---------- | -------- |
| Mutable at runtime? | No | Yes |
| Occupies memory at runtime? | No (inlined) | Yes (one word) |
| Valid in constant contexts? | Yes | No |
| Can be used in `#if` / `Iftrue`? | Yes | No |
| Counted toward global limit? | No | Yes |

Because constants are inlined, they incur no runtime memory cost and no
variable-slot cost. Prefer `Constant` over `Global` for any value that
does not change during execution.

### 3.8.5 Redefining Constants

By default, redefining a constant that already has a value produces a
compile-time error. However, the compiler allows redefinition in specific
cases:

- A constant declared with the `Default` directive is a **weak**
  definition: it takes effect only if no prior `Constant` with that name
  exists. This is the standard mechanism for library defaults.

```inform6
Default MAX_SCORE 100;            ! used if MAX_SCORE not already defined
```

- Conditional compilation (`Ifdef`, `Ifndef`) can guard constant
  declarations to prevent redefinition.

- The `Undef` directive (available since compiler version 6.33) removes
  a constant's definition, allowing it to be redefined:

```inform6
Constant VERSION = 1;
Undef VERSION;
Constant VERSION = 2;             ! now VERSION is 2
```

---

*Copyright © 2026 Software Freedom Conservancy, Inc.*
*Licensed under the GNU General Public License, version 3 or later.*
