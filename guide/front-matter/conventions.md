# Conventions Used in This Guide

This chapter describes the typographic, notational, and organizational
conventions used throughout the guide. Familiarizing yourself with these
conventions will help you read the guide efficiently and interpret code
examples, syntax descriptions, and cross-references correctly.

## Typographic Conventions

The following typographic conventions are used in the text:

- **`Monospaced text`** is used for Inform 6 source code, identifiers
  (variable names, routine names, property names, attribute names, class
  names, and action names), compiler switches, file names, and literal values.
  Examples: `MyRoutine`, `has light`, `-v5`, `GamePreRoutine`.

- **Bold text** is used for terms being defined for the first time, for
  emphasis, and for the names of concepts when they are introduced. Example:
  "An **attribute** is a boolean flag that can be set or cleared on any
  object."

- *Italic text* is used for book titles, document titles, placeholder names in
  syntax descriptions (see "Syntax Notation" below), and occasional emphasis.
  Example: "The *player object* is the object representing the human player
  in the game world."

- "Quoted text" is used when referring to specific input or output strings, or
  when mentioning a word as a word rather than using it. Example: The verb
  "take" is defined by the grammar line for `##Take`.

## Code Examples

Code examples appear in fenced code blocks:

```inform6
[ Main;
    print "Hello, world!^";
];
```

### Complete vs. Partial Examples

Some examples are **complete programs** that can be compiled and run as shown.
These are identified by the presence of a `Main` routine or an `Initialise`
entry point, or by an explicit note that the example is a complete program.

Most examples, however, are **code fragments** that illustrate a particular
feature or technique. These fragments are meant to be read in context and may
need to be placed inside a routine, an object definition, or a complete source
file before they can be compiled.

### Example Output

When the output produced by a code example is shown, it appears in a separate
block immediately following the code, prefixed by "Output:":

```inform6
print "The answer is ", 6 * 7, ".^";
```

Output:

```
The answer is 42.
```

### Line Numbers

Code examples in this guide do not include line numbers unless a specific line
is being discussed. When a specific line must be referenced, it is identified
by its content or by an inline comment.

### Ellipses in Code

An ellipsis (`...`) within a code example indicates that code has been omitted
for brevity. The omitted code is not relevant to the point being illustrated:

```inform6
Object cave "Dark Cave"
    with description "A damp, echoing cavern.",
         ...
    has  light;
```

## Cross-References

Cross-references to other sections of this guide use the format
**§*Chapter.Section*** or simply **§*Chapter*** when referring to an entire
chapter. For example:

> See §3.2 for a complete discussion of object properties.

> See §12 for the full list of compiler memory settings.

When a cross-reference points to a different part of the guide, the part name
may be included for clarity:

> See Part 3, §7.4 for the library's scope rules.

## Z-machine and Glulx Notation

The Inform 6 compiler can produce code for two target virtual machines: the
**Z-machine** and **Glulx**. Most of the Inform 6 language and standard library
works identically on both platforms. When a feature, behavior, or limitation is
specific to one VM, it is marked with one of the following indicators:

> **[Z-machine]** This paragraph or section describes behavior specific to the
> Z-machine. It does not apply when compiling for Glulx.

> **[Glulx]** This paragraph or section describes behavior specific to Glulx.
> It does not apply when compiling for the Z-machine.

> **[Z-machine/Glulx difference]** This paragraph or section describes a
> feature that exists on both platforms but behaves differently on each. The
> differences are explained in the text.

Material that is not marked with any VM indicator applies to both platforms.

### Compiler Version Notation

Where a feature was introduced or changed in a specific compiler or library
version, this is noted in the text:

> **[Compiler 6.40+]** This feature requires Inform compiler version 6.40 or
> later.

> **[Library 6.12+]** This feature requires Inform standard library version
> 6.12 or later.

## Syntax Notation

Formal syntax descriptions in this guide use a BNF-like (Backus–Naur Form)
notation. The following conventions apply:

### Terminal and Non-Terminal Symbols

- **`monospaced bold`** text represents literal keywords and punctuation that
  must appear exactly as shown. Example: **`if`**, **`(`**, **`;`**

- *italic* text represents non-terminal symbols—placeholders for syntactic
  categories that are defined elsewhere. Example: *expression*, *statement*,
  *identifier*

### Repetition and Optionality

| Notation               | Meaning                                           |
| ---------------------- | ------------------------------------------------- |
| *item*?                | Zero or one occurrence of *item* (optional)        |
| *item*\*               | Zero or more occurrences of *item*                 |
| *item*+                | One or more occurrences of *item*                  |
| *item*<sub>list</sub>  | A comma-separated list of one or more *item*       |

### Grouping and Alternation

- Parentheses **( )** group elements together.
- A vertical bar **|** separates alternatives. Example:

  *access-level* → **`private`** | **`public`**

### Syntax Description Format

A complete syntax rule is written as:

> *non-terminal* → *definition*

For example:

> *if-statement* → **`if`** **`(`** *expression* **`)`** *statement*
> ( **`else`** *statement* )?

This reads: "An *if-statement* consists of the keyword `if`, a left
parenthesis, an *expression*, a right parenthesis, a *statement*, and
optionally the keyword `else` followed by another *statement*."

When a syntax rule is too long for a single line, continuation lines are
indented:

> *object-definition* → **`Object`** *arrow*? *identifier*? *string*?
>     *parent-expression*?
>     *body*? **`;`**

### Syntax Diagrams vs. Prose

Not every feature is described with formal syntax notation. In many cases,
especially for commonly used constructs, the syntax is described in prose and
illustrated with examples. Formal syntax descriptions are provided for the
complete language grammar in the reference appendices and are used in the body
of the guide when precision is important.

## Terminology

The following terms have specific meanings in this guide:

- **Program** or **source file**: The Inform 6 source code being compiled,
  which may consist of one or more `.inf` or `.h` files.
- **Story file**: The compiled binary output produced by the Inform 6 compiler,
  in either Z-machine (`.z5`, `.z8`, etc.) or Glulx (`.ulx`) format.
- **Interpreter** or **terp**: A program that executes a compiled story file.
- **VM**: Virtual machine—either the Z-machine or Glulx.
- **Object**: An Inform 6 object, as defined by an `Object` or `Class`
  declaration. Not to be confused with the general programming concept of an
  object in object-oriented programming, though Inform 6 objects do support
  inheritance, properties, and message passing.
- **Routine**: A callable unit of code in Inform 6, defined with the
  `[ RoutineName; ... ];` syntax. Routines are called "functions" in most
  other programming languages.
- **Entry point**: A routine that the library calls at a defined point during
  execution, allowing the author to customize behavior. Examples include
  `Initialise`, `GamePreRoutine`, and `DeathMessage`.
- **Directive**: A top-level compiler instruction such as `Object`, `Class`,
  `Global`, `Constant`, `Include`, or `Ifdef`. Directives are processed at
  compile time.
- **Statement**: An executable instruction within a routine, such as an
  assignment, a `print` statement, a conditional, or a loop.
- **Action**: A unit of game-world activity triggered by player input and
  processed by the library's action-handling framework. Examples: `##Take`,
  `##Go`, `##Examine`.
