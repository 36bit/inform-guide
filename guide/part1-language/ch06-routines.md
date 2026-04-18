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

# Chapter 6: Routines

This chapter describes routines (functions): how they are declared, how
arguments and local variables work, how values are returned, and the
calling conventions that govern their behavior on the Z-machine and Glulx
virtual machines.

## 6.1 Routine Declaration Syntax

A routine is declared with square brackets enclosing a name, an optional
list of local variables, and a body of statements:

```
[ RoutineName local1 local2 ... localN;
    statements
];
```

The opening `[` begins the routine. The routine name follows immediately.
After the name come zero or more local variable names, separated by
whitespace. A semicolon ends the header and begins the body. The closing
`]` followed by `;` ends the routine.

```inform6
[ Double x;
    return x * 2;
];
```

Local variable names are untyped — each is a single machine word, just like
all Inform 6 values (see Chapter 2).

## 6.2 Stand-Alone Routines

A **stand-alone routine** (also called a **top-level routine**) is defined
at the outermost level of the source file, outside any object or class
definition. It is callable by name from anywhere in the program:

```inform6
[ Greet;
    print "Hello, adventurer!^";
];

[ Main;
    Greet();
];
```

Stand-alone routines are the primary mechanism for organizing code in
programs. The name becomes a global constant whose value is the
packed address of the routine.

## 6.3 Embedded Routines

An **embedded routine** is defined inline as the value of an object
property or in any other context where a routine value is expected. It has
no explicit name:

```inform6
Object lamp "brass lamp"
  with description [;
           print "A well-polished brass lamp.^";
       ],
       before [;
           Take: print "You pick up the lamp carefully.^";
               rtrue;
       ];
```

The compiler assigns an internal name to each embedded routine (typically
derived from the object and property names). Embedded routines are not
callable by name from other parts of the program — they are invoked
automatically by the library when the relevant property is accessed.

### 6.3.1 Embedded Routines in Expressions

Embedded routines can also appear in non-property contexts, producing a
routine value:

```inform6
[ Main fn;
    fn = [; print "Anonymous!^"; ];
    indirect(fn);
];
```

## 6.4 Parameters and Local Variables

The local variables listed in a routine's header serve a dual purpose: the
first *N* locals receive the *N* arguments passed by the caller, and any
remaining locals are initialized to zero.

```inform6
[ Add a b;
    return a + b;
];

[ Main result;
    result = Add(3, 4);     ! a=3, b=4, result is 7
];
```

### 6.4.1 Missing and Excess Arguments

If the caller supplies **fewer** arguments than the routine declares
locals, the unmatched locals are initialized to zero:

```inform6
[ Flexible a b c;
    print a, " ", b, " ", c, "^";
];

[ Main;
    Flexible(10);            ! prints: 10 0 0
    Flexible(10, 20);        ! prints: 10 20 0
    Flexible(10, 20, 30);    ! prints: 10 20 30
];
```

If the caller supplies **more** arguments than the routine has locals, the
excess arguments are silently ignored on the Z-machine. They are evaluated
(so side effects occur) but their values are discarded.

### 6.4.2 Local Variable Limits

On the Z-machine, a routine may declare at most **15 local variables**
(the virtual machine's hard limit; the compiler constant
`MAX_LOCAL_VARIABLES` is 16, which counts the `sp` pseudo-slot at index
0). On Glulx, the limit is **118 local variables** — `MAX_LOCAL_VARIABLES`
is 119, again including `sp`. Both limits are fixed compiler constants
and are not user-configurable.

### 6.4.3 Locals Are Not Persistent

Local variables exist only for the duration of a single call. Each
invocation of a routine receives its own fresh set of locals. Recursive
calls each get independent copies (see §6.10).

## 6.5 Return Values

Every routine returns a single value to its caller. Inform 6 provides
several ways to specify the return value.

### 6.5.1 Explicit `return`

The `return` statement exits the routine immediately with the given value:

```inform6
[ Max a b;
    if (a >= b) return a;
    return b;
];
```

`return` with no expression returns `true` (1), regardless of whether
the routine is stand-alone or embedded.

### 6.5.2 `rtrue` and `rfalse`

These shorthand statements return `true` (1) and `false` (0) respectively:

```inform6
[ IsPositive n;
    if (n > 0) rtrue;
    rfalse;
];
```

### 6.5.3 String-as-Statement Shorthand

A bare double-quoted string as a statement is equivalent to `print_ret`:
it prints the string, prints a newline, and returns `true`:

```inform6
[ Blocked;
    "You can't go that way.";
    ! equivalent to: print_ret "You can't go that way.";
];
```

## 6.6 The Default Return Convention

When execution reaches the closing `]` of a routine without encountering
an explicit `return`, `rtrue`, or `rfalse`, the return value depends on
the routine type:

| Routine type | Default return value |
| ------------ | -------------------- |
| Stand-alone | `true` (1) |
| Embedded | `false` (0) |

```inform6
[ StandAlone;
    print "Doing work.^";
    ! falls off the end → returns true
];

Object thing "thing"
  with before [;
           print "Before.^";
           ! falls off the end → returns false
       ];
```

This convention exists because the library interprets return values from
embedded property routines as "should the default action be suppressed?"
Returning `false` means "no, continue with the default", which is the
safe default for embedded routines. Stand-alone routines, by contrast,
often represent actions that "succeed" by default.

> **Tip:** Always use explicit return statements in non-trivial routines.
> Relying on the implicit default makes code harder to understand.

## 6.7 Calling Conventions

### 6.7.1 Argument Passing

Arguments are passed by value. The caller evaluates each argument
expression and places the results in the callee's local variables. Changes
to local variables inside the routine do not affect the caller's variables:

```inform6
[ Increment x;
    x = x + 1;
    print x, "^";
];

[ Main n;
    n = 5;
    Increment(n);    ! prints 6
    print n, "^";    ! prints 5 — n is unchanged
];
```

### 6.7.2 Z-Machine Argument Limit

On the Z-machine, a single call instruction can pass at most **7
arguments**. Attempting to call a routine with more than 7 arguments
produces a compile-time error. The Z-machine also limits routines to 15
local variables total (parameters plus non-parameter locals).

### 6.7.3 Glulx Argument Limit

On Glulx, the argument limit is significantly higher and is effectively
constrained only by available stack space. This makes Glulx suitable for
routines that need many parameters.

### 6.7.4 Return Value Handling

Every call expression produces a value — the routine's return value. If
the caller does not use the return value (e.g., a bare `MyRoutine();`
expression statement), the value is discarded.

## 6.8 `indirect()` and Indirect Calls

The built-in function `indirect()` calls a routine whose address is stored
in a variable or expression. The first argument is the routine address;
subsequent arguments are passed to the called routine:

```inform6
[ Apply fn x;
    return indirect(fn, x);
];

[ Double n;
    return n * 2;
];

[ Main result;
    result = Apply(Double, 21);
    print result, "^";          ! prints 42
];
```

### 6.8.1 Use Cases

Indirect calls are useful for:

- **Callback patterns** — passing behavior as an argument.
- **Dispatch tables** — storing routine addresses in arrays and calling
  them by index.
- **Plugin architectures** — swapping behavior at runtime.

```inform6
Array handlers --> ProcessA ProcessB ProcessC;

[ Dispatch n arg;
    return indirect(handlers-->n, arg);
];
```

### 6.8.2 Argument Limits for `indirect()`

On the Z-machine, `indirect()` supports up to 7 arguments (including the
routine address, so effectively 6 passed arguments). The compiler
generates different call opcodes depending on the number of arguments.

## 6.9 Variable-Argument Routines (Glulx)

On Glulx, a routine can accept a variable number of arguments by naming
its first parameter `_vararg_count`. When the compiler sees this special
name, it generates code that passes arguments on the VM stack rather than
in fixed local variable slots:

```inform6
[ PrintAll _vararg_count  val;
    while (_vararg_count > 0) {
        @copy sp val;
        print val, " ";
        _vararg_count--;
    }
    new_line;
];

[ Main;
    PrintAll(10, 20, 30);     ! prints: 10 20 30
    PrintAll(1);              ! prints: 1
];
```

### 6.9.1 How It Works

When a `_vararg_count` routine is called:

1. All arguments are pushed onto the Glulx stack.
2. `_vararg_count` is set to the number of arguments actually passed.
3. The routine body uses `@copy sp localvar` (Glulx assembly) to pop
   arguments from the stack one at a time.

### 6.9.2 Z-Machine Limitation

This mechanism is **Glulx-only**. The Z-machine has no equivalent; on
the Z-machine, the number and identity of arguments is fixed at compile
time by the routine's local variable list.

## 6.10 Recursion

Inform 6 supports full recursion. A routine may call itself directly or
indirectly, and each call receives its own set of local variables on the
virtual machine's stack:

```inform6
[ Factorial n;
    if (n <= 1) return 1;
    return n * Factorial(n - 1);
];

[ Main;
    print "5! = ", Factorial(5), "^";    ! prints: 5! = 120
];
```

### 6.10.1 Stack Depth

Recursive calls consume stack space proportional to the depth of recursion.
On the Z-machine, stack space is limited (typically a few hundred frames,
depending on the story file format and interpreter). Deep recursion can
exhaust the stack and cause a runtime error.

On Glulx, the stack is generally much larger, but extremely deep recursion
can still overflow it.

### 6.10.2 Tail Recursion

The compiler does not perform tail-call optimization. A recursive
call in tail position still allocates a new stack frame. If deep recursion
is needed, consider rewriting the algorithm iteratively:

```inform6
[ FactorialIter n result;
    result = 1;
    for (: n > 1 : n--)
        result = result * n;
    return result;
];
```

## 6.11 Routine References as Values

Because a routine name is a compile-time constant holding the routine's
packed address, routines can be stored in variables, passed as arguments,
and placed in arrays:

```inform6
[ Hello;
    print "Hello!^";
];

[ Main fn;
    fn = Hello;
    indirect(fn);         ! calls Hello
];
```

### 6.11.1 Identifying Routines with `metaclass`

The `metaclass()` built-in function can determine whether a value is a
routine:

```inform6
[ Identify val;
    if (metaclass(val) == Routine)
        print "It's a routine.^";
    else if (metaclass(val) == Object)
        print "It's an object.^";
    else if (metaclass(val) == Class)
        print "It's a class.^";
    else if (metaclass(val) == String)
        print "It's a string.^";
    else
        print "It's nothing special.^";
];
```

The four metaclasses — `Routine`, `Object`, `Class`, and `String` — cover
all non-integer values in Inform 6. An integer (including zero) returns
`nothing` (0) from `metaclass()`.

### 6.11.2 Routine References in Properties

A common pattern stores routine references in object properties for
later invocation:

```inform6
[ DefaultReact;
    "Nothing happens.";
];

[ SpecialReact;
    "Something amazing happens!";
];

Object widget "widget"
  with react_fn DefaultReact;

[ TriggerReaction obj;
    indirect(obj.react_fn);
];

[ Main;
    TriggerReaction(widget);       ! "Nothing happens."
    widget.react_fn = SpecialReact;
    TriggerReaction(widget);       ! "Something amazing happens!"
];
```

### 6.11.3 Routine References in Arrays

Dispatch tables are a natural use of routine references:

```inform6
[ HandleNorth; print "You go north.^"; ];
[ HandleSouth; print "You go south.^"; ];
[ HandleEast;  print "You go east.^";  ];
[ HandleWest;  print "You go west.^";  ];

Array direction_handlers -->
    HandleNorth HandleSouth HandleEast HandleWest;

[ GoDirection dir;
    if (dir >= 0 && dir < 4)
        indirect(direction_handlers-->dir);
    else
        print "Invalid direction.^";
];
```

## 6.12 The `Replace` Directive

The `Replace` directive lets a later definition supersede an earlier one
for a named routine. It is the standard mechanism for overriding a
routine supplied by an included library or extension. `Replace` must
appear **before** the file containing the original definition is
included.

### 6.12.1 One-Argument Form

```inform6
Replace RoutineName;
```

This form discards any later definition's identification with the
original symbol: when `RoutineName` is subsequently defined in the
included source, that new definition replaces the original. The original
body is silently dropped from the compiled output, although references
elsewhere in the program are routed to the replacement.

```inform6
Replace InScope;       ! discard the library's InScope
Include "Parser";
Include "VerbLib";

[ InScope actor;
    ! ... custom scope routine ...
];
```

### 6.12.2 Two-Argument Form (compiler 6.33+)

```inform6
Replace RoutineName OriginalName;
```

The two-argument form is the same as the one-argument form except that
the original (replaced) routine remains callable under the second name.
This is useful when the new definition wants to delegate to or extend
the original behaviour rather than discarding it entirely.

```inform6
Replace DrawStatusLine OriginalDrawStatusLine;
Include "Parser";
Include "VerbLib";

[ DrawStatusLine;
    ! Pre-processing of the status line
    OriginalDrawStatusLine();
    ! Post-processing of the status line
];
```

The original-form symbol becomes available as a regular routine constant
holding the packed address of the original definition; calling it
behaves exactly like calling the original routine.

### 6.12.3 Replacing System Functions

`Replace` can also be applied to a small set of built-in system functions
(such as `random`) that are normally implemented by the virtual machine.
In this case only the one-argument form is permitted, and the directive
must appear before the system function is referenced anywhere in the
program; otherwise the compiler reports
`You can't 'Replace' a system function already used`.

### 6.12.4 Restrictions

- `Replace` must precede the original definition; the compiler reports
  `name of routine not yet defined` if the named symbol has already
  been defined.
- The original symbol must be a routine, not a constant, object, or
  other declaration.
- A routine cannot be replaced more than once.
- The two-argument form requires compiler 6.33 or later.
```
