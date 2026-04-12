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

# Chapter 10: Compiler Directives

Compiler directives are top-level instructions processed at compile time.
They define program structure, declare data, configure compilation, and
control conditional inclusion. Unlike statements (which appear inside
routines), directives appear at the outermost level of source files, with
a few exceptions noted in §10.12.

Directives are not case-sensitive. The following declarations are
equivalent:

```inform6
Global varname;
GLOBAL varname;
global varname;
```

Directives may optionally be written with a `#` prefix:

```inform6
#Global varname;
```

For top-level directives the `#` is optional. However, the small set of
directives permitted inside routines and object definitions (§10.12) must
use the `#` prefix.

---

## 10.1 Data Definition Directives

### 10.1.1 Constant

The `Constant` directive defines a named compile-time constant. It has
several forms:

```inform6
Constant MAX_ITEMS;               ! defines MAX_ITEMS with value 0
Constant MAX_ITEMS = 20;          ! defines with explicit value
Constant MAX_ITEMS 20;            ! equivalent (= is optional)
Constant TOTAL = MAX_ITEMS * 3;   ! computed from a constant expression
```

As of compiler 6.42, multiple constants may be declared in a single
directive, separated by commas:

```inform6
Constant NORTH = 1, SOUTH = 2, EAST = 3, WEST = 4;
```

Redefining a constant with the same value is permitted; redefining it with
a different value produces a compiler error. This allows library files and
game source to agree on shared values without conflict:

```inform6
Constant DEBUG;           ! first definition (value 0)
Constant DEBUG;           ! permitted: same value
! Constant DEBUG = 1;    ! ERROR: different value
```

Constants defined with no value are typically used as flags to be tested
with `#Ifdef` (see §10.5.1).

### 10.1.2 Global

The `Global` directive declares a global variable. See §3.1 for a full
discussion of global variable storage and limits.

```inform6
Global turn_count;            ! initialized to 0
Global score = 100;           ! initialized to 100
Global score 100;             ! equivalent (= is optional since 6.40)
```

As of compiler 6.43, it is safe to declare a global variable more than
once. At most one declaration should give an initial value. Providing the
same compile-time constant value in multiple declarations is also
accepted.

> **[Z-machine]** The Z-machine provides exactly **240** global variable
> slots. Seven are reserved by the compiler for internal use, leaving
> approximately 233 for user-declared globals.

> **[Glulx]** Glulx places no fixed upper bound on the number of global
> variables; they are allocated dynamically.

### 10.1.3 Array

The `Array` directive declares an array. See Chapter 8 for full
documentation of array types, initializers, and access patterns.

```inform6
Array inventory -> 20;
Array exits --> 50;
Array title string "Zork";
Array scores table 0 0 0 0 0;
Array input_buffer buffer 120;
Array flags static -> 1 0 1 0;
```

### 10.1.4 Default

The `Default` directive defines a constant only if it has not already been
defined. This is the primary mechanism for library files to provide
default values that games can override.

```inform6
Default MAX_CARRIED 100;
Default AMUSING_PROVIDED 0;
```

If the name is already defined (whether by `Constant`, as a routine name,
or any other symbol), the `Default` directive has no effect. A typical
pattern is:

```inform6
! In the game file (compiled first):
Constant MAX_CARRIED 10;

! In a library file (included later):
Default MAX_CARRIED 100;    ! no effect — MAX_CARRIED is already 10
```

---

## 10.2 Object and Class Directives

### 10.2.1 Object

The `Object` directive declares an object. See §7.1 for the full syntax
and detailed discussion.

```inform6
Object lamp "brass lamp" study
    with name 'brass' 'lamp',
         description "A well-polished brass lamp.",
    has  light;
```

### 10.2.2 Nearby

The `Nearby` directive is equivalent to `Object ->`. It places the new
object as a child of the most recently declared object at the previous
nesting level.

```inform6
Object study "Study";

Nearby desk "oak desk"
    with name 'oak' 'desk',
         description "A sturdy desk.";

Nearby chair "wooden chair"
    with name 'wooden' 'chair';
```

This is equivalent to:

```inform6
Object study "Study";
Object -> desk "oak desk" ...;
Object -> chair "wooden chair" ...;
```

### 10.2.3 Class

The `Class` directive declares a class. See §7.5 for the full syntax
and inheritance rules.

```inform6
Class Treasure(10)
    with value 0,
         description "A precious treasure.",
    has  scored;
```

### 10.2.4 Attribute

The `Attribute` directive declares a boolean attribute that can be
assigned to objects with the `has` keyword and tested at runtime with
`has` and `hasnt`.

```inform6
Attribute light;
Attribute edible;
Attribute locked alias lockable;   ! alias for an existing attribute
```

The `alias` form creates a second name for an existing attribute. Both
names refer to the same underlying bit.

> **[Z-machine]** The Z-machine supports up to **48** attributes
> (numbered 0–47). The compiler and library reserve some of these.

> **[Glulx]** The number of attributes is controlled by the
> `$NUM_ATTR_BYTES` setting. The default is 7 bytes (56 attributes),
> but this can be increased.

### 10.2.5 Property

The `Property` directive declares a common property. There are several
forms:

```inform6
Property door_dir;                   ! common property, no default
Property capacity 100;               ! common property with default value
Property additive describe;          ! additive: inherited values accumulate
Property individual preference;      ! individual property
Property door_dir alias door_to;     ! alias for an existing property
```

The `additive` keyword causes a property's values to accumulate through
the inheritance chain rather than being overridden. This is used by the
library for properties such as `name`.

The `individual` keyword declares a property that is stored per-object
rather than in the common property table. Individual properties have no
default value, cannot be `additive`, and cannot `alias` another property.

The `long` keyword is accepted for backward compatibility but has no
effect; all properties have been word-sized since Inform 5.

> **[Z-machine]** The Z-machine (version 4 and later) allows up to
> **63** common properties (numbered 1–63). The compiler reserves
> three, so 60 are available for the library and game.

---

## 10.3 Source Organization Directives

### 10.3.1 Include

The `Include` directive inserts the contents of another source file at the
current position.

```inform6
Include "Parser";          ! searches include path
Include ">MyExtras";       ! searches same directory as current file
```

The `">"` prefix tells the compiler to look for the file in the same
directory as the file containing the `Include` directive. Without it, the
compiler searches the current directory first, then each directory
specified by the `+include_path` command-line option.

Files are included textually: the effect is as if the contents of the
included file replaced the `Include` directive.

### 10.3.2 Link

The `Link` directive was formerly used for linking pre-compiled modules.
As of compiler 6.40, module linking has been removed and this directive
produces a compiler error.

```
Link "module";             ! ERROR in compiler 6.40+
```

### 10.3.3 Replace

The `Replace` directive allows a routine defined later in compilation
(typically in a library file) to be replaced by a new definition.

```inform6
Replace DrawStatusLine;

[ DrawStatusLine;
    ! Custom status line drawing code
    @set_cursor 1 1;
    print "My Custom Status";
];

Include "Parser";          ! the library's DrawStatusLine is ignored
```

A second form, added in compiler 6.33, preserves access to the original
routine under a new name:

```inform6
Replace BurnSub OriginalBurnSub;

[ BurnSub;
    if (noun == candle) return OriginalBurnSub();
    "You shouldn't play with fire.";
];
```

When multiple definitions of a replaced routine exist, the last-defined
version in a normal file takes precedence. Definitions in `System_file`
files and the veneer are only used if no normal-file definition exists.

### 10.3.4 Stub

The `Stub` directive creates a minimal do-nothing routine if no routine
with that name has been defined. The routine takes a specified number of
local variables (0–4) and returns `false`.

```inform6
Stub AfterLife 0;
Stub ChooseObjects 2;
```

`Stub` is used by the library to provide default entry points. If the game
defines its own version of the routine, the stub is not created.

### 10.3.5 System_file

The `System_file` directive marks the current source file as a system file
(a library or infrastructure file, as opposed to game author code).

```inform6
System_file;
```

This affects how errors and warnings are reported: the compiler may
suppress or reformat messages originating from system files. It also
affects `Replace` precedence (§10.3.3): definitions in system files are
overridden by definitions in normal files.

---

## 10.4 Compilation Configuration

### 10.4.1 Release

The `Release` directive sets the release number stored in the story file
header.

```inform6
Release 7;
```

The release number is a single integer. It has no effect on compilation
but is embedded in the output file for version-tracking purposes.

### 10.4.2 Serial

The `Serial` directive sets the six-character serial number stored in the
story file header. By default, the compiler uses the current date in
`YYMMDD` format.

```inform6
Serial "250115";           ! must be exactly 6 characters
```

### 10.4.3 Statusline

The `Statusline` directive controls the format of the status line
displayed by the interpreter.

```inform6
Statusline score;          ! show score and turns (default)
Statusline time;           ! show time of day
```

This directive sets a flag in the story file header. Its effect depends on
the interpreter honouring the flag.

### 10.4.4 Version

The `Version` directive sets the Z-machine version number for the output
file.

```inform6
Version 5;                 ! compile for Z-machine version 5
```

Valid values are 3, 4, 5, 6, 7, and 8. As of compiler 6.40, this
directive generates a deprecation warning; the preferred method is the
`-v` command-line switch or `!% -v5` header comment.

> **[Z-machine]** This directive applies only to Z-machine compilation.
> It has no effect when compiling for Glulx.

### 10.4.5 Switches

The `Switches` directive passes command-line switch options from within
the source file.

```inform6
Switches v5s;              ! equivalent to command-line -v5 -s
```

As of compiler 6.40, this directive is deprecated and generates a warning.
Use `!%` header comments instead:

```inform6
!% -v5
!% -s
```

The `Switches` directive must appear before the first constant or routine
definition.

---

## 10.5 Conditional Compilation

Conditional compilation directives allow sections of source code to be
included or excluded based on compile-time conditions. They can be nested
up to 32 levels deep.

### 10.5.1 Ifdef and Ifndef

`Ifdef` compiles a block only if a named symbol is defined. `Ifndef` is
the inverse.

```inform6
#Ifdef DEBUG;
[ DebugPrint txt;
    print (string) txt, "^";
];
#Endif;

#Ifndef CUSTOM_BANNER;
[ Banner;
    print "Welcome to the game!^";
];
#Endif;
```

The tested symbol can be any identifier: a constant, global variable,
routine name, object, class, or attribute. The test checks only whether
the symbol has been defined at the point of the directive.

### 10.5.2 Iftrue and Iffalse

`Iftrue` compiles a block when a constant expression evaluates to a
nonzero (true) value. `Iffalse` compiles when the expression evaluates to
zero (false).

```inform6
#Iftrue MAX_ITEMS > 50;
Array large_inventory --> MAX_ITEMS;
#Endif;

#Iffalse WORDSIZE == 2;
! This code is only compiled on Glulx (WORDSIZE is 4).
Global extra_data;
#Endif;
```

The expression must be evaluable at compile time, using only constants,
compiler-defined symbols, and arithmetic operators.

### 10.5.3 Ifnot

`Ifnot` provides an else-branch for any conditional directive. It must
appear between the opening `Ifdef`/`Ifndef`/`Iftrue`/`Iffalse` and the
closing `Endif`.

```inform6
#Ifdef TARGET_ZCODE;
! Z-machine-specific code
Constant HEADER_SIZE 64;
#Ifnot;
! Glulx-specific code
Constant HEADER_SIZE 36;
#Endif;
```

Each conditional block may contain at most one `Ifnot`.

### 10.5.4 IFV3 and IFV5

`IFV3` compiles a block only when targeting Z-machine version 3.
`IFV5` compiles a block for Z-machine version 4 or later, or for Glulx.

```inform6
#IFV3;
! V3-only code (e.g., limited status line handling)
#Endif;

#IFV5;
! V4+ and Glulx code
#Endif;
```

Despite its name, `IFV5` does not restrict compilation to version 5
specifically — it covers all versions from 4 onward, including Glulx.
The name is a historical artifact. `IFV3` is equivalent to:

```inform6
#Ifdef TARGET_ZCODE;
#Iftrue (#version_number == 3);
! ...
#Endif;
#Endif;
```

### 10.5.5 Nesting and Scope

Conditional directives can be nested up to 32 levels deep:

```inform6
#Ifdef TARGET_ZCODE;
    #Iftrue (#version_number >= 5);
        #Ifdef DEBUG;
        ! Z-code v5+ debug-only code
        #Endif;
    #Endif;
#Endif;
```

Conditional compilation is one of the few directive categories that can
appear inside routines and object definitions when prefixed with `#`
(see §10.12):

```inform6
[ MyRoutine;
    print "Starting...^";
    #Ifdef DEBUG;
    print "[Debug mode active]^";
    #Endif;
    rfalse;
];
```

---

## 10.6 Text and Character Directives

### 10.6.1 Abbreviate

The `Abbreviate` directive declares a text string to be used for Z-machine
text compression. As of compiler 6.40, abbreviations are only created when
the economy (`-e`) switch is active; without it, the directive is silently
skipped.

```inform6
Abbreviate "the ";
Abbreviate "You ";
Abbreviate "ing ";
```

Abbreviations reduce the size of encoded text in the story file. The
compiler substitutes frequently occurring strings with compact tokens.

> **[Z-machine]** Z-machine version 3 supports up to **32**
> abbreviations; version 4 and later support up to **96**. As of
> compiler 6.42, abbreviation strings may be of any length (earlier
> versions limited them to 64 characters).

> **[Glulx]** The `Abbreviate` directive has no effect when compiling
> for Glulx, which uses a different text encoding.

### 10.6.2 Lowstring

The `Lowstring` directive defines an encoded string placed within readable
memory (the first 64K of the Z-machine address space).

```inform6
Lowstring MyText "Hello, world!";
```

The constant `MyText` holds the packed address divided by 2. To print the
string:

```inform6
print (address) (2 * MyText);
```

`Lowstring` is rarely needed in Inform 6; it exists for backward
compatibility with Inform 5. The more common approach for dynamic string
references is the `string` statement.

> **[Z-machine]** `Lowstring` is supported only on the Z-machine.

> **[Glulx]** `Lowstring` is not supported on Glulx.

### 10.6.3 Zcharacter

The `Zcharacter` directive customizes the Z-character alphabet table used
to encode text in the story file.

```inform6
Zcharacter 'ä';                      ! add a character to the alphabet
Zcharacter table 'a' 'b' 'c' ...;   ! define a custom alphabet table
Zcharacter table + '@{e9}' '@{e8}';  ! add accented characters
```

By adding frequently used characters (such as accented letters) to the
alphabet, the compiler can encode them more compactly. Characters not in
the alphabet require multi-byte escape sequences.

> **[Z-machine]** `Zcharacter` applies only to Z-machine text encoding.

> **[Glulx]** This directive has no effect when compiling for Glulx,
> which uses Unicode strings natively.

---

## 10.7 Dictionary and Action Directives

### 10.7.1 Dictionary

The `Dictionary` directive forces a word into the dictionary and
optionally sets flag values. See §9.6 for full dictionary documentation.

```inform6
Dictionary 'plugh';                ! add word with no special flags
Dictionary 'xyzzy' 128;           ! add word and set dict_par1 flag
Dictionary 'abracadabra' 128 4;   ! set both dict_par1 and dict_par3
```

If the word already exists, the flag values are bitwise-OR'd with the
existing flags. The `dict_par2` field (verb number) cannot be set by this
directive.

> **[Z-machine]** Flag values must be in the range 0–255 (one byte).

> **[Glulx]** Flag values must be in the range 0–65535 (two bytes).

### 10.7.2 Verb and Extend

The `Verb` directive defines verb grammar, and `Extend` modifies existing
grammar. These are documented fully in Part 3 (the parser and grammar
system).

```inform6
Verb 'take' 'get'
    * noun                  -> Take
    * 'off' worn            -> Disrobe;

Extend 'take'
    * 'inventory'           -> Inv;
```

As of compiler 6.43, two `Verb` declarations for the same word are
accepted (equivalent to `Extend last`), but this usage generates a
warning. The `Extend` form is preferred.

### 10.7.3 Fake_action

The `Fake_action` directive declares a fake action. Fake actions have
action numbers but no associated grammar — they cannot be triggered by
player input. They are used for `before`/`after` hooks and internal
signalling.

```inform6
Fake_action Salute;

Object guard "guard"
    with before [;
        Salute: "The guard salutes you back!";
    ];
```

A fake action can be sent to an object using the action-invocation syntax:

```inform6
<Salute guard>;
```

---

## 10.8 Informational Directives

### 10.8.1 Message

The `Message` directive prints a message during compilation. It does not
affect the compiled output.

```inform6
Message "Compiling the main game file...";
Message warning "This feature is experimental.";
Message error "Missing required constant STORY_NAME.";
Message fatalerror "Cannot continue without STORY_NAME.";
```

The `warning` form issues a compiler warning. The `error` form issues a
non-fatal error (compilation continues but no output file is produced).
The `fatalerror` form halts compilation immediately.

`Message` can be used inside routines with the `#` prefix (see §10.12):

```inform6
[ Init;
    #Message "Compiling Init routine...";
    location = study;
];
```

### 10.8.2 Trace

The `Trace` directive adjusts trace and diagnostic settings during
compilation. As of compiler 6.40, it is deprecated in favor of trace
options (`$!` settings), but it retains the unique ability to print
tables and change trace levels partway through compilation.

```inform6
Trace dictionary;          ! print dictionary as of this point
Trace objects;             ! print object tree as of this point
Trace symbols;             ! print symbol table (Trace symbols 2 for detail)
Trace verbs;               ! print grammar table as of this point

Trace assembly on;         ! enable assembly tracing
Trace assembly off;        ! disable assembly tracing
Trace expressions 2;       ! set expression trace to level 2
Trace tokens on;           ! enable token tracing
```

The bare form `Trace N;` is equivalent to `Trace assembly N;`.

### 10.8.3 Origsource

The `Origsource` directive marks the following source lines as having
originated from a different file, primarily for tools that generate
Inform 6 code from other languages (such as Inform 7). Added in compiler
6.34.

```inform6
Origsource "story.ni" 42 1;   ! lines from story.ni, line 42, char 1
! ... generated code ...
Origsource;                    ! reset to current file position
```

The declaration holds until the next `Origsource` directive but does not
apply to included files. Error messages and debug output will reference
the declared source location rather than the actual I6 file position.

---

## 10.9 Symbol Management

### 10.9.1 Undef

The `Undef` directive removes a previously defined constant. Added in
compiler 6.33.

```inform6
Constant BETA_TEST;
! ... code that uses #Ifdef BETA_TEST ...
Undef BETA_TEST;
! ... from here, #Ifdef BETA_TEST will be false ...
```

`Undef` only works on constants defined with `Constant`. It cannot
undefine globals, routines, objects, or other symbol types. If the
named constant was never defined, the directive does nothing.

A constant is considered undefined only *after* the `Undef` directive,
just as it is considered defined only *after* its `Constant` declaration.

---

## 10.10 End Directive

The `End` directive marks the end of the source file. Everything after
`End;` is ignored by the compiler.

```inform6
[ Main;
    print "Hello!^";
];

End;

This text is completely ignored by the compiler.
It could be notes, design documents, or anything.
```

`End` is rarely used explicitly, since reaching the physical end of the
source file has the same effect.

---

## 10.11 Import

The `Import` directive was formerly part of the module-linking system,
which allowed pre-compiled modules to share symbols. As of compiler 6.40,
the module system has been removed and this directive produces a compiler
error.

---

## 10.12 Directives Allowed Inside Routines

Most directives can only appear at the top level of a source file, outside
routine and object definitions. The following directives may also appear
inside routines and object bodies when prefixed with `#`:

- **Conditional compilation**: `#Ifdef`, `#Ifndef`, `#Iftrue`, `#Iffalse`,
  `#Ifnot`, `#Endif`, `#IFV3`, `#IFV5`
- **Informational**: `#Message`, `#Trace`
- **Source tracking**: `#Origsource`

The `#` prefix is mandatory in this context. Omitting it produces a
compiler error.

```inform6
Object torch "torch"
    with description [;
        print "A wooden torch";
        #Ifdef LIT_DESCRIPTIONS;
        if (self has light) print " (lit)";
        #Endif;
        ".";
    ],
    #Ifdef BURNABLE;
    before [;
        Burn: give self light; "You light the torch.";
    ],
    #Endif;
    has ;
```

Inside routines, these directives are most commonly used for conditional
compilation of platform-specific or debug code:

```inform6
[ MovePlayer dest;
    #Ifdef DEBUG;
    print "[Moving player to ", (name) dest, "]^";
    #Endif;

    PlayerTo(dest);

    #Iftrue MAX_ITEMS > 100;
    CheckEncumbrance();
    #Endif;
];
```
