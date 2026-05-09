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

# Appendix K: Inform 6 BNF Grammar

This appendix provides a formal grammar specification of the language as
accepted by the compiler targeting both Z-machine and Glulx virtual
machines. The grammar is expressed in a variant of Backus–Naur Form (BNF).

**Notation conventions:**

| Notation | Meaning |
|----------|---------|
| `::=` | "is defined as" |
| `\|` | alternatives (logical OR) |
| `[ ... ]` | optional (zero or one occurrence) |
| `{ ... }` | repetition (zero or more occurrences) |
| `{ ... }+` | repetition (one or more occurrences) |
| `'keyword'` | a literal keyword or token (case-insensitive) |
| `"text"` | a literal separator or operator |
| *italic* | a nonterminal symbol |
| `CAPS` | a terminal token class from the lexer |
| `(* ... *)` | a prose comment or constraint |

---

## §K.1 Program Structure

A program is a sequence of top-level constructs — directives, routine
definitions, and class-instantiated object definitions — processed until
end-of-file or the `End` directive.


```
program
    ::= { top-level-construct }

top-level-construct
    ::= routine-definition
    |   class-instantiation
    |   directive
```

A *routine-definition* begins with `[` and a *class-instantiation* begins
with a symbol whose type is `CLASS_T`. Everything else at the top level is
a *directive*.

```
routine-definition
    ::= "[" IDENTIFIER [ "*" ] { local-variable-name } ";"
        routine-body
        "]" ";"

class-instantiation
    ::= CLASS-NAME body-of-definition ";"
    (* Where CLASS-NAME is a previously declared class symbol.
       This is syntactic sugar for creating an instance of that class. *)
```

The `*` debug marker must appear before any local variable names
(i.e., when no locals have yet been declared).


---

## §K.2 Lexical Elements

### §K.2.1 Character Set

Source files use the ASCII character set (bytes 0x20–0x7E plus whitespace
characters 0x09 tab, 0x0A newline, 0x0D carriage return). The compiler
recognizes the Latin-1 range (0xA1–0xFF) in string literals and character
constants.

### §K.2.2 Comments


```
line-comment
    ::= "!" { any-character-except-newline } NEWLINE
```

Comments begin with `!` and extend to the end of the line. There are no
block comments in Inform 6.

### §K.2.3 Identifiers

```
IDENTIFIER
    ::= letter { letter | digit | "_" }

letter
    ::= "a" | ... | "z" | "A" | ... | "Z"

digit
    ::= "0" | ... | "9"
```

Identifiers are case-insensitive. The maximum length is implementation-defined
(limited by available memory).

### §K.2.4 Numeric Literals


```
numeric-literal
    ::= decimal-literal
    |   hex-literal
    |   binary-literal
    |   float-literal          (* Glulx only *)

decimal-literal
    ::= digit { digit }
    |   "-" digit { digit }

hex-literal
    ::= "$" hex-digit { hex-digit }

binary-literal
    ::= "$$" bin-digit { bin-digit }

float-literal
    ::= "$+" float-chars       (* positive float; Glulx only *)
    |   "$-" float-chars       (* negative float; Glulx only *)
    |   "$<+" float-chars      (* positive float, low word; Glulx double *)
    |   "$<-" float-chars      (* negative float, low word; Glulx double *)
    |   "$>+" float-chars      (* positive float, high word; Glulx double *)
    |   "$>-" float-chars      (* negative float, high word; Glulx double *)

hex-digit
    ::= digit | "a" | ... | "f" | "A" | ... | "F"

bin-digit
    ::= "0" | "1"
```

### §K.2.5 String Literals

```
double-quoted-string
    ::= '"' { string-character | escape-sequence } '"'

string-character
    ::= any-character-except-quote-caret-tilde-at

escape-sequence
    ::= "^"                    (* newline *)
    |   "~"                    (* literal double-quote *)
    |   "@@" decimal-digits    (* ZSCII by number *)
    |   "@{" hex-digits "}"    (* Unicode by hex codepoint *)
    |   "@'" accent-code       (* accent escape — see Appendix J *)
    |   "@" IDENTIFIER         (* named accent — see Appendix J *)
```

### §K.2.6 Character Constants

```
single-quoted-constant
    ::= "'" character-or-escape "'"

character-or-escape
    ::= any-single-character
    |   "@{" hex-digits "}"
    |   "@'" accent-code
    |   "@" IDENTIFIER
```

### §K.2.7 Dictionary Words

```
dictionary-word-literal
    ::= "'" { dict-char }+ "'"
    [ "//" dict-flags ]

dict-char
    ::= any-character-except-quote-and-separator

dict-flags
    ::= { [ "~" ] ( "p" | "s" | "n" ) }
    (* p = plural, s = singular, n = noun;
       a leading "~" negates the flag (e.g. "//~p" clears the plural bit) *)
```

Dictionary word literals in single quotes with length ≥ 2 are dictionary
entries, not character constants.

### §K.2.8 Hash Tokens


```
hash-token
    ::= "##" IDENTIFIER        (* action name → action number *)
    |   "#a$" IDENTIFIER       (* legacy: action number *)
    |   "#r$" IDENTIFIER       (* legacy: routine packed address *)
    |   "#n$" IDENTIFIER       (* legacy: dictionary word address *)
    |   "#g$" IDENTIFIER       (* legacy: global variable number *)
    |   "#w$" IDENTIFIER       (* legacy: dictionary word address, Glulx *)
    |   "#" IDENTIFIER         (* system constant *)
```

### §K.2.9 System Constants


```
system-constant
    ::= "#" system-constant-name

system-constant-name
    ::= 'adjectives_table'    | 'actions_table'       | 'classes_table'
    |   'identifiers_table'   | 'preactions_table'    | 'version_number'
    |   'largest_object'      | 'strings_offset'      | 'code_offset'
    |   'dict_par1'           | 'dict_par2'           | 'dict_par3'
    |   'actual_largest_object'
    |   'static_memory_offset'| 'array_names_offset'  | 'readable_memory_offset'
    |   'cpv__start'          | 'cpv__end'            | 'ipv__start'
    |   'ipv__end'            | 'array__start'        | 'array__end'
    |   'lowest_attribute_number'   | 'highest_attribute_number'
    |   'attribute_names_array'
    |   'lowest_property_number'    | 'highest_property_number'
    |   'property_names_array'
    |   'lowest_action_number'      | 'highest_action_number'
    |   'action_names_array'
    |   'lowest_fake_action_number' | 'highest_fake_action_number'
    |   'fake_action_names_array'
    |   'lowest_routine_number'     | 'highest_routine_number'
    |   'routines_array'            | 'routine_names_array'
    |   'routine_flags_array'
    |   'lowest_global_number'      | 'highest_global_number'
    |   'globals_array'             | 'global_names_array'
    |   'global_flags_array'
    |   'lowest_array_number'       | 'highest_array_number'
    |   'arrays_array'              | 'array_names_array'
    |   'array_flags_array'
    |   'lowest_constant_number'    | 'highest_constant_number'
    |   'constants_array'           | 'constant_names_array'
    |   'lowest_class_number'       | 'highest_class_number'
    |   'class_objects_array'
    |   'lowest_object_number'      | 'highest_object_number'
    |   'oddeven_packing'
    |   'grammar_table'             | 'dictionary_table'
    |   'dynam_string_table'
    |   'highest_meta_action_number'
```

### §K.2.10 Separators and Operators


The following separator tokens are recognized by the lexer. Their role as
operators is defined in §K.7.

| Token | Symbol | Token | Symbol |
|-------|--------|-------|--------|
| `->` | ARROW | `-->` | DARROW |
| `--` | DEC | `-` | MINUS |
| `++` | INC | `+` | PLUS |
| `*` | TIMES | `/` | DIVIDE |
| `%` | REMAINDER | `\|\|` | LOGOR |
| `\|` | ARTOR | `&&` | LOGAND |
| `&` | ARTAND | `~~` | LOGNOT |
| `~=` | NOTEQUAL | `~` | ARTNOT |
| `==` | CONDEQUALS | `=` | SETEQUALS |
| `>=` | GE | `>` | GREATER |
| `<=` | LE | `<` | LESS |
| `(` | OPENB | `)` | CLOSEB |
| `,` | COMMA | `.&` | PROPADD |
| `.#` | PROPNUM | `..&` | MPROPADD |
| `..#` | MPROPNUM | `..` | MESSAGE |
| `.` | PROPERTY | `::` | SUPERCLASS |
| `:` | COLON | `@` | AT |
| `;` | SEMICOLON | `[` | OPEN_SQUARE |
| `]` | CLOSE_SQUARE | `{` | OPEN_BRACE |
| `}` | CLOSE_BRACE | `$` | DOLLAR |
| `##` | HASHHASH | `#` | HASH |

### §K.2.11 Keywords

Keywords are context-sensitive in Inform 6 — the lexer enables and disables
keyword groups depending on what construct is being parsed.

**Statement keywords** — active inside routine bodies:

> `box` `break` `continue` `default` `do` `else` `font` `for` `give`
> `if` `inversion` `jump` `move` `new_line` `objectloop` `print`
> `print_ret` `quit` `read` `remove` `restore` `return` `rfalse`
> `rtrue` `save` `spaces` `string` `style` `switch` `until` `while`

**Condition keywords** — active inside expressions:

> `has` `hasnt` `in` `notin` `ofclass` `or` `provides`

**System function keywords** — active inside expressions:

> `child` `children` `elder` `eldest` `indirect` `metaclass` `parent`
> `random` `sibling` `younger` `youngest` `glk`

**Directive keywords** — active at the top level:

> `abbreviate` `array` `attribute` `class` `constant` `default`
> `dictionary` `end` `endif` `extend` `fake_action` `global` `ifdef`
> `ifndef` `ifnot` `ifv3` `ifv5` `iftrue` `iffalse` `import`
> `include` `link` `lowstring` `message` `nearby` `object` `origsource`
> `property` `release` `replace` `serial` `switches` `statusline`
> `stub` `system_file` `trace` `undef` `verb` `version` `zcharacter`

**Segment markers** — active in object/class bodies:

> `class` `has` `private` `with`

**Misc keywords** — active in print statements and
other contexts:

> `char` `name` `the` `a` `an` `The` `number` `roman` `reverse` `bold`
> `underline` `fixed` `on` `off` `to` `address` `string` `object`
> `near` `from` `property` `A`

**Directive sub-keywords** :

> `alias` `long` `additive` `score` `time` `noun` `held` `multi`
> `multiheld` `multiexcept` `multiinside` `creature` `special` `number`
> `scope` `topic` `reverse` `meta` `only` `replace` `first` `last`
> `string` `table` `buffer` `data` `initial` `initstr` `with` `private`
> `has` `class` `error` `fatalerror` `warning` `terminating` `static`
> `individual`

---

## §K.3 Directives

All directives are terminated by `;`. Directives may optionally be prefixed
with `#` when used at the top level. Inside routine bodies and object
definitions, only certain directives are permitted and must be prefixed
with `#`.


### §K.3.1 Permitted Internal Directives

When used inside a routine body or object/class definition (with `#` prefix),
only these directives are allowed:

```
internal-directive
    ::= '#' ( 'Ifdef' | 'Ifndef' | 'Iftrue' | 'Iffalse'
            | 'Ifv3' | 'Ifv5' | 'Ifnot' | 'Endif'
            | 'Message' | 'Origsource' | 'Trace' ) ... ";"
```

### §K.3.2 Abbreviate


```
abbreviate-directive
    ::= 'Abbreviate' STRING { STRING } ";"
```

Declares one or more abbreviation strings for Z-machine text compression.
The hard ceiling is 96 abbreviations in Z-code; the default `$MAX_ABBREVS`
setting is 64, and the 96-slot pool is shared with `$MAX_DYNAMIC_STRINGS`.
Not supported in Glulx mode.

### §K.3.3 Array


```
array-directive
    ::= 'Array' IDENTIFIER [ 'static' ] array-type array-initializer ";"

array-type
    ::= "->"                   (* byte array *)
    |   "-->"                  (* word array *)
    |   'string'               (* byte array with length prefix *)
    |   'table'                (* word array with length prefix *)
    |   'buffer'               (* byte array with word-sized length prefix *)

array-initializer
    ::= numeric-expression                (* count of zero-filled entries *)
    |   expression { expression }         (* list of initial values *)
    |   STRING                            (* ASCII character values *)
    |   "[" expression { [ ";" ] expression } "]"
                                          (* bracketed list of values *)
```

When the initializer is a single expression followed by `;`, it is the
number of entries (initialized to zero). When multiple expressions appear
before `;`, they are the data values. A `string` or `table` array has a
length word at index 0; a `buffer` array has a word-sized length at
byte offset 0.

### §K.3.4 Attribute


```
attribute-directive
    ::= 'Attribute' IDENTIFIER [ 'alias' IDENTIFIER ] ";"
```

### §K.3.5 Class


```
class-directive
    ::= 'Class' IDENTIFIER [ "(" numeric-expression ")" ]
        body-of-definition ";"
```

The optional parenthesized expression specifies the number of duplicate
instances to create.

### §K.3.6 Constant


```
constant-directive
    ::= 'Constant' constant-def { "," constant-def } ";"

constant-def
    ::= IDENTIFIER [ [ "=" ] constant-expression ]
```

If no value is given, the constant is defined with value 0.

### §K.3.7 Default


```
default-directive
    ::= 'Default' IDENTIFIER [ "=" ] constant-expression ";"
```

Defines a constant only if it has not already been defined.

### §K.3.8 Dictionary


```
dictionary-directive
    ::= 'Dictionary' dict-word [ constant-expression [ constant-expression ] ] ";"

dict-word
    ::= DICTIONARY-WORD
    |   STRING
```

Creates a dictionary entry with optional flag values for `dict_par1` and
`dict_par3`.

### §K.3.9 End


```
end-directive
    ::= 'End' ";"
```

Terminates compilation immediately.

### §K.3.10 Extend


```
extend-directive
    ::= 'Extend' extend-mode ";"

extend-mode
    ::= 'only' verb-words mode-keyword grammar-body
    |   verb-word mode-keyword grammar-body

mode-keyword
    ::= [ 'replace' | 'first' | 'last' ]

verb-words
    ::= verb-word { verb-word }

verb-word
    ::= STRING
```

See §K.5 for *grammar-body*.

### §K.3.11 Fake_Action


```
fake-action-directive
    ::= 'Fake_Action' IDENTIFIER ";"
```

### §K.3.12 Global


```
global-directive
    ::= 'Global' IDENTIFIER [ [ "=" ] constant-expression ] ";"
```

### §K.3.13 Conditional Compilation


```
ifdef-directive
    ::= 'Ifdef' IDENTIFIER ";"

ifndef-directive
    ::= 'Ifndef' IDENTIFIER ";"

ifv3-directive
    ::= 'Ifv3' ";"
    (* True when targeting Z-machine version ≤ 3 *)

ifv5-directive
    ::= 'Ifv5' ";"
    (* True when targeting Z-machine version > 3 *)

iftrue-directive
    ::= 'Iftrue' constant-expression ";"
    (* True when expression evaluates to non-zero *)

iffalse-directive
    ::= 'Iffalse' constant-expression ";"
    (* True when expression evaluates to zero *)

ifnot-directive
    ::= 'Ifnot' ";"
    (* Inverts the sense of the most recent If... *)

endif-directive
    ::= 'Endif' ";"
```

`Ifdef` checks whether a symbol is defined. The special form `VN_nnnn`
(where `nnnn` is a 4-digit number) is considered defined if the compiler
version number is ≥ `nnnn`. Conditional directives may be nested up to 32
levels deep.

### §K.3.14 Import (Deprecated)


```
import-directive
    ::= 'Import' ... ";"
    (* No longer supported; produces an error *)
```

### §K.3.15 Include


```
include-directive
    ::= 'Include' STRING ";"
```

The string is a filename. If it begins with `>`, the file is loaded from
the same directory as the current source file rather than from the include
path.

### §K.3.16 Link (Deprecated)


```
link-directive
    ::= 'Link' STRING ";"
    (* No longer supported; produces an error *)
```

### §K.3.17 Lowstring

 — Z-machine only.

```
lowstring-directive
    ::= 'Lowstring' IDENTIFIER STRING ";"
```

### §K.3.18 Message


```
message-directive
    ::= 'Message' STRING ";"
    |   'Message' 'error' STRING ";"
    |   'Message' 'fatalerror' STRING ";"
    |   'Message' 'warning' STRING ";"
```

### §K.3.19 Nearby


See §K.4.

### §K.3.20 Object


See §K.4.

### §K.3.21 Origsource


```
origsource-directive
    ::= 'Origsource' [ STRING [ NUMBER [ NUMBER ] ] ] ";"
```

With no arguments, clears the original-source annotation. With arguments,
sets the original source filename and optionally line and character number.

### §K.3.22 Property


```
property-directive
    ::= 'Property' { 'long' | 'additive' | 'individual' } IDENTIFIER
        [ 'alias' IDENTIFIER
        | constant-expression ] ";"
```

The keywords `long`, `additive`, and `individual` may appear in any order
(and any number) before the property name. The `long` keyword is accepted
for compatibility but is deprecated — all properties are now automatically
long. The `individual` keyword explicitly creates an individual property
(in which case no `alias` clause or default value is permitted, and
`individual` is incompatible with `additive`). The `alias` clause is
incompatible with `additive`.

### §K.3.23 Release


```
release-directive
    ::= 'Release' constant-expression ";"
```

### §K.3.24 Replace


```
replace-directive
    ::= 'Replace' IDENTIFIER [ IDENTIFIER ] ";"
```

The first identifier is the routine to be replaced. The optional second
identifier receives the original definition of the replaced routine.

### §K.3.25 Serial


```
serial-directive
    ::= 'Serial' STRING ";"
```

The string must be exactly 6 digits (a date in `YYMMDD` format).

### §K.3.26 Statusline


```
statusline-directive
    ::= 'Statusline' ( 'score' | 'time' ) ";"
```

### §K.3.27 Stub


```
stub-directive
    ::= 'Stub' IDENTIFIER NUMBER ";"
```

The number (0–4) specifies how many local variables the stub routine has.
The stub is only created if the identifier has not been defined.

### §K.3.28 Switches (Deprecated)


```
switches-directive
    ::= 'Switches' UNQUOTED-TEXT ";"
```

### §K.3.29 System_file


```
system-file-directive
    ::= 'System_file' ";"
```

Declares the current source file as a system file (affecting `Replace`
behavior).

### §K.3.30 Trace


```
trace-directive
    ::= 'Trace' [ trace-keyword ] [ 'on' | 'off' | NUMBER ] ";"

trace-keyword
    ::= 'dictionary' | 'symbols' | 'objects' | 'verbs'
    |   'assembly' | 'expressions' | 'lines' | 'tokens' | 'linker'
```

Without a keyword, defaults to `assembly`. `dictionary`, `symbols`,
`objects`, and `verbs` display a table immediately; the others set
a tracing level.

### §K.3.31 Undef


```
undef-directive
    ::= 'Undef' IDENTIFIER ";"
```

Removes a previously defined constant from the symbol table.

### §K.3.32 Verb


See §K.5.

### §K.3.33 Version (Deprecated)


```
version-directive
    ::= 'Version' constant-expression ";"
```

Sets the Z-machine version (3–8). Deprecated in favor of command-line
switches.

### §K.3.34 Zcharacter

 — Z-machine only.

```
zcharacter-directive
    ::= 'Zcharacter' STRING STRING STRING ";"
    |   'Zcharacter' CHARACTER-CONSTANT ";"
    |   'Zcharacter' 'table' [ "+" ] { NUMBER | CHARACTER-CONSTANT } ";"
    |   'Zcharacter' 'terminating' { NUMBER } ";"
```

The three-string form defines new Z-machine alphabets A0, A1, and A2.

---

## §K.4 Object and Class Definitions


### §K.4.1 Object

```
object-definition
    ::= 'Object' [ arrows ] [ IDENTIFIER ] [ STRING ] [ parent-ref ]
        body-of-definition ";"

nearby-definition
    ::= 'Nearby' [ IDENTIFIER ] [ STRING ] body-of-definition ";"

arrows
    ::= "->" { "->" }
```

The *arrows* syntax is an alternative to `Nearby` for setting the object's
parent. Each `->` represents one level of tree depth; `Nearby` is equivalent
to a single `->`.

The header components are parsed in order:

1. Optional sequence of `->` arrows (not with `Nearby`)
2. Optional internal name (an identifier)
3. Optional textual short name (a double-quoted string)
4. Optional parent object (a previously defined object or class name)

```
parent-ref
    ::= OBJECT-NAME | CLASS-NAME
```

### §K.4.2 Class

```
class-definition
    ::= 'Class' IDENTIFIER [ "(" constant-expression ")" ]
        body-of-definition ";"
```

### §K.4.3 Body of Definition

```
body-of-definition
    ::= { segment }

segment
    ::= 'with' property-list
    |   'private' property-list
    |   'has' attribute-list
    |   'class' class-list
```

Segments may appear in any order and may be separated by commas. Each
segment is terminated by the start of the next segment, a comma, or `;`.


### §K.4.4 Property List


```
property-list
    ::= property-entry { "," property-entry }

property-entry
    ::= IDENTIFIER { property-value }

property-value
    ::= expression
    |   embedded-routine
    |   STRING                 (* in 'name' property: treated as dict word *)
    |   DICTIONARY-WORD        (* in 'name' property *)

embedded-routine
    ::= "[" { local-variable-name } [ "*" ] ";" routine-body "]"
```

A property with no values is initialized to 0. The `name` property
(property number 1) treats double-quoted strings as dictionary words
rather than static strings. Embedded routines are named
`ObjectName.PropertyName` (for objects) or `ClassName::PropertyName` (for
classes).

### §K.4.5 Attribute List


```
attribute-list
    ::= { [ "~" ] IDENTIFIER }
```

Attributes are separated by whitespace. The `~` prefix clears the attribute
rather than setting it.

### §K.4.6 Class List


```
class-list
    ::= CLASS-NAME { CLASS-NAME }
```

A class may not inherit from itself.

---

## §K.5 Verb and Grammar Definitions


### §K.5.1 Verb Directive

```
verb-directive
    ::= 'Verb' [ 'meta' ] verb-words
        ( "=" verb-word | grammar-body ) ";"

verb-words
    ::= verb-word { verb-word }

verb-word
    ::= STRING                 (* English verb word, e.g., "take" *)
```

The `= "existing-verb"` form makes the new verb synonymous with an
existing one. Otherwise, one or more grammar lines follow.

### §K.5.2 Grammar Body

```
grammar-body
    ::= { grammar-line }

grammar-line
    ::= "*" { grammar-token } "->" IDENTIFIER [ 'reverse' ] [ 'meta' ]
```

The `*` introduces each grammar line. The identifier after `->` is the
action name. `reverse` swaps noun and second (grammar version 2 or later
only). `meta` marks this action as a meta-action; it requires
`$GRAMMAR_META_FLAG` to be set, but works in any grammar version.

### §K.5.3 Grammar Tokens


```
grammar-token
    ::= 'noun'                 (* standard noun phrase *)
    |   'held'                 (* object held by player *)
    |   'multi'                (* multiple objects *)
    |   'multiheld'            (* multiple held objects *)
    |   'multiexcept'          (* multiple except one *)
    |   'multiinside'          (* multiple inside *)
    |   'creature'             (* animate object *)
    |   'special'              (* special token *)
    |   'number'               (* numeric input *)
    |   'topic'                (* text topic; grammar version 2+ only *)
    |   DICTIONARY-WORD [ "/" DICTIONARY-WORD { "/" DICTIONARY-WORD } ]
                               (* preposition with optional alternatives *)
    |   'noun' "=" IDENTIFIER  (* noun with filter routine *)
    |   'scope' "=" IDENTIFIER (* custom scope routine *)
    |   IDENTIFIER             (* general parsing routine *)
    |   ATTRIBUTE-NAME         (* filter by attribute *)
```

The `/` separator allows alternative prepositions within a single token
position (e.g., `'in'/'inside'/'into'`).

---

## §K.6 Routines


### §K.6.1 Routine Definition

```
routine-definition
    ::= "[" IDENTIFIER [ "*" ] { local-variable-name } ";"
        routine-body
        "]" ";"

local-variable-name
    ::= IDENTIFIER
```

The `*` before the local variables enables debug tracing for this routine.
It must appear before any local variable names. A routine may have at most
15 local variables in Z-code (including parameters) or up to 118 in Glulx.

### §K.6.2 Routine Body

```
routine-body
    ::= { statement | switch-case | default-case }
```

At the top level of a routine (not inside a `switch` block), the compiler
also accepts switch cases and a `default` clause, forming an implicit
switch on the first argument. This is the "action routine" pattern.

### §K.6.3 Switch-Case at Routine Level


```
switch-case
    ::= case-values ":" statement-sequence

default-case
    ::= 'default' ":" statement-sequence

case-values
    ::= case-value { "," case-value }

case-value
    ::= constant-expression [ 'to' constant-expression ]
```

---

## §K.7 Expressions


### §K.7.1 Expression Grammar

The Inform 6 expression parser is a shift-reduce parser with a
precedence table. The following grammar is equivalent to the parser's
behavior.

```
expression
    ::= assignment-expression { "," assignment-expression }

assignment-expression
    ::= logical-expression [ "=" assignment-expression ]

logical-expression
    ::= logical-unary-expression { ( "&&" | "||" ) logical-unary-expression }

logical-unary-expression
    ::= [ "~~" ] comparison-expression

comparison-expression
    ::= or-expression [ comparison-operator or-expression ]

comparison-operator
    ::= "==" | "~=" | ">" | ">=" | "<" | "<="
    |   'has' | 'hasnt' | 'in' | 'notin' | 'ofclass' | 'provides'

or-expression
    ::= additive-expression { 'or' additive-expression }

additive-expression
    ::= multiplicative-expression { ( "+" | "-" ) multiplicative-expression }

multiplicative-expression
    ::= unary-expression { ( "*" | "/" | "%" | "&" | "|" ) unary-expression }

unary-expression
    ::= [ "~" ] array-expression

array-expression
    ::= unary-minus-expression { ( "->" | "-->" ) unary-minus-expression }

unary-minus-expression
    ::= [ "-" ] inc-dec-expression

inc-dec-expression
    ::= "++" property-expression
    |   "--" property-expression
    |   property-expression [ "++" | "--" ]

property-expression
    ::= call-expression { property-operator IDENTIFIER }

property-operator
    ::= ".&" | ".#" | "..&" | "..#"
    |   "." | ".."
    |   "::"

call-expression
    ::= primary-expression { "(" [ expression { "," expression } ] ")" }

primary-expression
    ::= IDENTIFIER
    |   numeric-literal
    |   STRING
    |   DICTIONARY-WORD
    |   "##" IDENTIFIER                     (* action value *)
    |   "#" system-constant-name            (* system constant *)
    |   system-function-call
    |   "(" expression ")"
    |   action-expression
```

### §K.7.2 Operator Precedence Table

The following table gives the complete operator precedence, from lowest
(level 0) to highest (level 14), as defined in the compiler's operator
table.


| Level | Operators | Assoc. | Type | Description |
|-------|-----------|--------|------|-------------|
| 0 | `,` | Left | Binary | Comma (sequence) |
| 1 | `=` | Right | Binary | Assignment |
| 2 | `&&` `\|\|` | Left | Binary | Logical AND, OR |
| 2 | `~~` | Right | Prefix | Logical NOT |
| 3 | `==` `~=` | — | Binary | Equality, inequality |
| 3 | `>` `>=` `<` `<=` | — | Binary | Relational |
| 3 | `has` `hasnt` | — | Binary | Attribute test |
| 3 | `in` `notin` | — | Binary | Object tree test |
| 3 | `ofclass` `provides` | — | Binary | Class/property test |
| 4 | `or` | Left | Binary | Condition alternative |
| 5 | `+` `-` | Left | Binary | Addition, subtraction |
| 6 | `*` `/` `%` | Left | Binary | Multiplication, division, remainder |
| 6 | `&` `\|` | Left | Binary | Bitwise AND, OR |
| 6 | `~` | Right | Prefix | Bitwise NOT |
| 7 | `->` `-->` | Left | Binary | Byte/word array access |
| 8 | `-` (unary) | Right | Prefix | Unary minus |
| 9 | `++` `--` | Right | Prefix | Pre-increment, pre-decrement |
| 9 | `++` `--` | Right | Postfix | Post-increment, post-decrement |
| 10 | `.&` `.#` `..&` `..#` | Left | Binary | Property address/length |
| 11 | `(` `)` | Left | — | Function call |
| 12 | `.` `..` | Left | Binary | Property/message access |
| 13 | `::` | Left | Binary | Superclass property access |
| 14 | (push) | — | — | Stack push (Glulx only, internal) |

### §K.7.3 Compound Assignment and Mutation Operators

The compiler internally generates compound operators for assignment and
increment/decrement applied to array elements and properties. These are
not directly written in source code but arise from combining operators:


| Source Pattern | Internal Operator | Description |
|----------------|-------------------|-------------|
| `a->b = v` | ARROW_SETEQUALS | Byte array entry assignment |
| `a-->b = v` | DARROW_SETEQUALS | Word array entry assignment |
| `obj..p = v` | MESSAGE_SETEQUALS | Individual property assignment |
| `obj.p = v` | PROPERTY_SETEQUALS | Common property assignment |
| `++a->b` | ARROW_INC | Byte array entry pre-increment |
| `a->b++` | ARROW_POST_INC | Byte array entry post-increment |
| `--a->b` | ARROW_DEC | Byte array entry pre-decrement |
| `a->b--` | ARROW_POST_DEC | Byte array entry post-decrement |
| `++obj.p` | PROPERTY_INC | Property pre-increment |
| `obj.p++` | PROPERTY_POST_INC | Property post-increment |
| `--obj.p` | PROPERTY_DEC | Property pre-decrement |
| `obj.p--` | PROPERTY_POST_DEC | Property post-decrement |
| `++a-->b` | DARROW_INC | Word array entry pre-increment |
| `a-->b++` | DARROW_POST_INC | Word array entry post-increment |
| `--a-->b` | DARROW_DEC | Word array entry pre-decrement |
| `a-->b--` | DARROW_POST_DEC | Word array entry post-decrement |
| `++obj..p` | MESSAGE_INC | Individual property pre-increment |
| `obj..p++` | MESSAGE_POST_INC | Individual property post-increment |
| `--obj..p` | MESSAGE_DEC | Individual property pre-decrement |
| `obj..p--` | MESSAGE_POST_DEC | Individual property post-decrement |
| `obj.p(...)` | PROP_CALL | Call to common property routine |
| `obj..p(...)` | MESSAGE_CALL | Call to individual property routine |

### §K.7.4 System Function Calls


```
system-function-call
    ::= 'parent' "(" expression ")"
    |   'child' "(" expression ")"
    |   'children' "(" expression ")"
    |   'sibling' "(" expression ")"
    |   'elder' "(" expression ")"
    |   'eldest' "(" expression ")"
    |   'younger' "(" expression ")"
    |   'youngest' "(" expression ")"
    |   'metaclass' "(" expression ")"
    |   'random' "(" expression { "," expression } ")"
    |   'indirect' "(" expression { "," expression } ")"
    |   'glk' "(" expression "," expression ")"
                                    (* Glulx only *)
```

When `random` is called with multiple arguments, it selects one at random.

### §K.7.5 Action Expressions


```
action-expression
    ::= "<" action-spec ">"
    |   "<<" action-spec ">>"

action-spec
    ::= action-name [ expression [ expression ] [ "," expression ] ]

action-name
    ::= IDENTIFIER             (* bare action name, resolved to ##Name *)
    |   "(" expression ")"     (* computed action value *)
```

The single-bracket form `<...>` calls `R_Process()`. The double-bracket
form `<<...>>` calls `R_Process()` and then returns `true`. The arguments
after the action name are noun, second, and actor (the actor being
preceded by a comma).

### §K.7.6 Constant Expressions

A *constant-expression* is an expression that can be fully evaluated at
compile time. It appears in directive contexts (array sizes, `Constant`
values, `Iftrue` conditions, etc.). Only literal values, previously
defined constants, and simple arithmetic on constants are permitted.

---

## §K.8 Statements


### §K.8.1 Statement Grammar

```
statement
    ::= expression-statement
    |   compound-statement
    |   if-statement
    |   switch-statement
    |   while-statement
    |   do-until-statement
    |   for-statement
    |   objectloop-statement
    |   print-statement
    |   print-ret-statement
    |   string-print-statement
    |   return-statement
    |   break-statement
    |   continue-statement
    |   jump-statement
    |   give-statement
    |   move-statement
    |   remove-statement
    |   read-statement
    |   action-statement
    |   box-statement
    |   font-statement
    |   style-statement
    |   new-line-statement
    |   spaces-statement
    |   string-statement
    |   inversion-statement
    |   quit-statement
    |   save-statement
    |   restore-statement
    |   assembly-statement
    |   label-definition

statement-sequence
    ::= { statement }
```

### §K.8.2 Expression Statement

```
expression-statement
    ::= expression ";"
```

### §K.8.3 Compound Statement (Block)

```
compound-statement
    ::= "{" statement-sequence "}"
```

### §K.8.4 If Statement

```
if-statement
    ::= 'if' "(" expression ")" statement-or-block [ 'else' statement-or-block ]

statement-or-block
    ::= compound-statement
    |   statement
```

### §K.8.5 Switch Statement

```
switch-statement
    ::= 'switch' "(" expression ")" "{" { switch-case } [ default-case ] "}"

switch-case
    ::= case-values ":" statement-sequence

default-case
    ::= 'default' ":" statement-sequence

case-values
    ::= case-value { "," case-value }

case-value
    ::= constant-expression [ 'to' constant-expression ]
```

Case values must be compile-time constants. The `to` keyword specifies
an inclusive range. In Z-code, up to 3 values can be tested in a single
branch instruction; in Glulx, only 1.

### §K.8.6 While Statement

```
while-statement
    ::= 'while' "(" expression ")" statement-or-block
```

### §K.8.7 Do-Until Statement

```
do-until-statement
    ::= 'do' statement-or-block 'until' "(" expression ")" ";"
```

Note: unlike C's `do-while`, Inform uses `until` (the loop continues
while the condition is *false*).

### §K.8.8 For Statement

```
for-statement
    ::= 'for' "(" [ for-init ] ":" [ expression ] ":" [ expression ] ")"
        statement-or-block

for-init
    ::= expression
```

**Important:** Inform uses `:` (colon) to separate the three parts of a
`for` loop, not `;` (semicolon) as in C. A `;` in this position produces
a warning and is treated as `:`.


### §K.8.9 Objectloop Statement


```
objectloop-statement
    ::= 'objectloop' "(" objectloop-header ")" statement-or-block

objectloop-header
    ::= IDENTIFIER                             (* iterate all objects *)
    |   IDENTIFIER 'near' expression           (* children of object *)
    |   IDENTIFIER 'from' expression           (* start from object *)
    |   IDENTIFIER 'in' expression             (* children of object — Inform 5 style *)
    |   IDENTIFIER condition-expression        (* filter expression *)
```

The `near` form iterates direct children of the given object. The `from`
form iterates the given object's parent's children. The `in` form is an
Inform 5 compatibility synonym for iteration over children.

When only an identifier is given (no keyword), the loop iterates over all
objects from 1 to the highest object number, optionally filtered by the
condition expression.

### §K.8.10 Print Statement


```
print-statement
    ::= 'print' print-list ";"

print-ret-statement
    ::= 'print_ret' print-list ";"

string-print-statement
    ::= STRING ";"
    (* Equivalent to: print_ret STRING; *)

print-list
    ::= print-item { "," print-item }

print-item
    ::= STRING
    |   expression                         (* prints as signed number *)
    |   "(" print-format ")" expression
    |   "(" IDENTIFIER ")" expression      (* custom print routine *)

print-format
    ::= 'char'                             (* print single character *)
    |   'address'                          (* print string at byte address *)
    |   'string'                           (* print string at packed address *)
    |   'a'                                (* print with indefinite article *)
    |   'an'                               (* synonym for 'a' *)
    |   'the'                              (* print with definite article *)
    |   'The'                              (* capitalized definite article *)
    |   'A'                                (* capitalized indefinite article — Glulx *)
    |   'name'                             (* print object short name *)
    |   'number'                           (* print number in words *)
    |   'property'                         (* print property name *)
    |   'object'                           (* low-level object name *)
```

`print_ret` additionally prints a newline and returns `true`.

A bare double-quoted string as a statement is equivalent to
`print_ret "string";`.

### §K.8.11 Return Statements

```
return-statement
    ::= 'return' [ expression ] ";"
    (* Without expression, returns true *)

rtrue-statement
    ::= 'rtrue' ";"

rfalse-statement
    ::= 'rfalse' ";"
```

### §K.8.12 Break and Continue

```
break-statement
    ::= 'break' ";"

continue-statement
    ::= 'continue' ";"
```

### §K.8.13 Jump and Labels

```
jump-statement
    ::= 'jump' IDENTIFIER ";"

label-definition
    ::= "." IDENTIFIER ";"
```

Labels are prefixed with `.` and terminated with `;`.

### §K.8.14 Give Statement

```
give-statement
    ::= 'give' expression give-list ";"

give-list
    ::= give-item { give-item }

give-item
    ::= [ "~" ] expression
```

The `~` prefix clears the attribute rather than setting it.

### §K.8.15 Move and Remove Statements

```
move-statement
    ::= 'move' expression 'to' expression ";"

remove-statement
    ::= 'remove' expression ";"
```

### §K.8.16 Read Statement


```
read-statement
    ::= 'read' expression expression [ expression ] ";"
```

Arguments are: text buffer, parse table, and optional status-line routine
(Z-machine version 4+ only). In Z-machine version 5+, the `aread` opcode
is used; in version 3–4, `sread` is used.

### §K.8.17 Box Statement

```
box-statement
    ::= 'box' STRING { STRING } ";"
```

Displays a centred box of text. Z-machine only (uses the `Box__Routine`
veneer).

### §K.8.18 Font and Style Statements

```
font-statement
    ::= 'font' ( 'on' | 'off' ) ";"

style-statement
    ::= 'style' style-keyword ";"

style-keyword
    ::= 'roman' | 'bold' | 'reverse' | 'underline' | 'fixed'
```

`font off` enables fixed-pitch; `font on` restores proportional. The
`style` statement requires Z-machine version 4 or later.

### §K.8.19 Other Statements

```
new-line-statement
    ::= 'new_line' ";"

spaces-statement
    ::= 'spaces' expression ";"

string-statement
    ::= 'string' expression ( STRING | constant-expression ) ";"
    (* Sets a dynamic string table entry: first argument is the
       table index, second is the replacement text or a string
       address *)

inversion-statement
    ::= 'inversion' ";"
    (* Prints the compiler version *)

quit-statement
    ::= 'quit' ";"

save-statement
    ::= 'save' LABEL ";"           (* Z-machine only *)

restore-statement
    ::= 'restore' LABEL ";"        (* Z-machine only *)
```

The label in `save`/`restore` is the target to branch to on success.
These statements are Z-machine only; in Glulx, save and restore are
handled through the `@save` and `@restore` assembly instructions or
through library routines.

### §K.8.20 Assembly Statement

```
assembly-statement
    ::= "@" opcode-name { assembly-operand } [ "->" store-operand ]
        [ "?" [ "~" ] branch-label ] ";"

assembly-operand
    ::= expression

store-operand
    ::= IDENTIFIER | 'sp'

branch-label
    ::= IDENTIFIER | 'rtrue' | 'rfalse'
```

The `@` prefix invokes a Z-machine or Glulx instruction directly. The
`->` before a store operand indicates where to store the result. The `?`
indicates a branch; `?~` branches on the opposite condition.

In Glulx, the `sp` keyword denotes the stack pointer as a store or load
target.

---

## §K.9 Expression Contexts

The compiler parses expressions differently depending on context, affecting
which operators and forms are permitted.


| Context | Description | Restrictions |
|---------|-------------|--------------|
| VOID | Statement expression | Comma allowed; value discarded |
| CONDITION | Boolean test | Result used as branch condition |
| CONSTANT | Compile-time | Must evaluate to constant |
| QUANTITY | Value expression | Standard expression |
| ACTION_Q | Action name | Restricted to action values |
| ASSEMBLY | Assembly operand | No `->` operator |
| ARRAY | Array initializer | No `::` operator |
| FORINIT | For-loop init | No `::` operator |
| RETURN_Q | Return value | Bare property names allowed |

---

## §K.10 Lexer Disambiguation Rules

The lexer applies several context-dependent disambiguation rules when
tokenizing.


### §K.10.1 Unary Minus vs. Binary Minus

The `-` token is treated as unary minus (UNARY_MINUS) when preceded by:
- An operator
- An open parenthesis `(`
- The start of an expression

Otherwise, it is binary minus (subtraction).

### §K.10.2 Pre-increment vs. Post-increment

The `++` token is treated as post-increment (POST_INC) when preceded by:
- A variable
- A close parenthesis `)`
- A number

Otherwise, it is pre-increment. The same rule applies to `--`.

### §K.10.3 Comma as Operator vs. Separator

The `,` token is treated as the comma operator only when:
- The expression context allows commas (VOID_CONTEXT), *and*
- The bracket nesting level is 0

Otherwise, it terminates the expression.

### §K.10.4 Arrow as Operator vs. Separator

The `->` token is treated as the byte-array operator only when:
- The expression context allows arrows (not ASSEMBLY_CONTEXT)

In assembly context, `->` is a store separator.

---

## §K.11 Array Type Summary


| Type | Keyword | Entry Size | Length Prefix |
|------|---------|------------|---------------|
| Byte array | `->` | 1 byte | None |
| Word array | `-->` | 2 bytes (Z) / 4 bytes (G) | None |
| String array | `string` | 1 byte | 1 byte at index 0 |
| Table array | `table` | 2 bytes (Z) / 4 bytes (G) | 1 entry at index 0 |
| Buffer array | `buffer` | 1 byte | 1 word at byte offset 0 |

`static` arrays are placed in read-only memory.

---

## §K.12 Grammar Version Differences


The compiler supports three grammar table encodings, selected by the
`Grammar__Version` constant:

| Version | Availability | Max Tokens/Line | Notes |
|---------|-------------|-----------------|-------|
| 1 | Z-code only | 6 | Legacy format; 3 bytes/token |
| 2 | Z-code and Glulx | 6 (Z) / varies (G) | Default; 3 bytes/token |
| 3 | Z-code only | 31 | Compact; 2 bytes/token |

Grammar version 1 does not support the `topic` token type or the
per-grammar-line `reverse` keyword. (Per-action `meta` is independent of
grammar version — it is gated by `$GRAMMAR_META_FLAG`.) Grammar version 3
is a compact encoding where the token count is embedded in the action
word, eliminating the terminator byte.

---

## §K.13 Complete Token Type Summary


| Code | Name | Description |
|------|------|-------------|
| 0 | SYMBOL_TT | Identifier (variable, routine, object, class, etc.) |
| 1 | NUMBER_TT | Numeric literal |
| 2 | DQ_TT | Double-quoted string literal |
| 3 | SQ_TT | Single-quoted character/dictionary word |
| 4 | UQ_TT | Unquoted text (identifiers when symbol table is disabled) |
| 5 | SEP_TT | Separator or operator token |
| 6 | EOF_TT | End of file |
| 100 | STATEMENT_TT | Statement keyword |
| 101 | SEGMENT_MARKER_TT | Object body segment marker (`with`/`has`/`class`/`private`) |
| 102 | DIRECTIVE_TT | Compiler directive keyword |
| 103 | CND_TT | Condition keyword (`has`/`in`/`or`/etc.) |
| 105 | SYSFUN_TT | System function keyword |
| 106 | LOCAL_VARIABLE_TT | Local variable name |
| 107 | OPCODE_NAME_TT | Assembly opcode name |
| 108 | MISC_KEYWORD_TT | Miscellaneous keyword (print formats, etc.) |
| 109 | DIR_KEYWORD_TT | Directive sub-keyword |
| 110 | TRACE_KEYWORD_TT | Trace option keyword |
| 111 | SYSTEM_CONSTANT_TT | System constant (`#name`) |
| 112 | OPCODE_MACRO_TT | Pseudo-opcode macro (Glulx: `pull`/`push`/`dload`/`dstore`) |
| 200 | OP_TT | Operator (internal expression tree) |
| 201 | ENDEXP_TT | End of expression (internal) |
| 202 | SUBOPEN_TT | Subexpression open `(` (internal) |
| 203 | SUBCLOSE_TT | Subexpression close `)` (internal) |
| 204 | LARGE_NUMBER_TT | Large constant (internal) |
| 205 | SMALL_NUMBER_TT | Small constant 0–255 (internal) |
| 206 | VARIABLE_TT | Variable reference (internal) |
| 207 | DICTWORD_TT | Dictionary word value (internal) |
| 208 | ACTION_TT | Action number (internal) |

---

## §K.14 Symbol Types


| Type | Description |
|------|-------------|
| ROUTINE_T | Stand-alone or embedded routine |
| CONSTANT_T | Named constant |
| GLOBAL_VARIABLE_T | Global variable |
| ARRAY_T | Dynamic array |
| STATIC_ARRAY_T | Static (ROM) array |
| OBJECT_T | Object |
| CLASS_T | Class |
| ATTRIBUTE_T | Attribute |
| PROPERTY_T | Common property |
| INDIVIDUAL_PROPERTY_T | Individual property |
| FAKE_ACTION_T | Fake action |

---

## §K.15 Collected Syntax Diagrams

This section collects the major syntactic forms for quick reference.

### Routine

```
[ Name * arg1 arg2 ... argN ;
    ... statements ...
]  ;
```

### Object

```
Object  [->]  name  "short name"  parent
    with  prop1 val1,
          prop2 [ locals ; ... ],
    has   attr1 attr2 ~attr3,
    class SuperClass
;
```

### Class

```
Class  ClassName(N)
    with  prop1 val1,
    has   attr1
;
```

### Array

```
Array  name  -->  100;              ! 100 zero-filled words
Array  name  ->   10 20 30;         ! 3 bytes: 10, 20, 30
Array  name  table  [ 1; 2; 3 ];    ! table with length prefix
Array  name  string "Hello";        ! string array with length prefix
Array  name  buffer 80;             ! buffer with word-sized length prefix
Array  name  static --> 256;        ! static (ROM) word array
```

### Verb

```
Verb  meta  "save" "store"
    * "game"                    -> Save
    * "transcript"              -> ScriptOn
;
```

### Extend

```
Extend  only  "take" "get"  replace
    * multi                     -> Take
    * 'off' noun                -> Disrobe
;
```

### Conditional Compilation

```
#Ifdef  DEBUG;
    ...
#Ifnot;
    ...
#Endif;
```

### Action Statements

```
<Take lamp>;                    ! Call Take action on lamp
<<Drop sword>>;                 ! Call Drop on sword, then return true
<PutOn ball table>;             ! Two-noun action
<Take ball, thief>;             ! Action with actor
```

### For Loop

```
for (i = 0 : i < 10 : i++)  { ... }
```

### Objectloop

```
objectloop (x in location)  { ... }
objectloop (x near player)  { ... }
objectloop (x has edible)   { ... }
objectloop (x)              { ... }     ! all objects
```

---

## §K.16 Differences from Other Language Grammars

Several aspects of Inform 6's grammar differ from common programming
languages in ways that may surprise programmers:

1. **Colon in `for` loops.** The separator is `:` not `;`.

2. **`do ... until` (not `do ... while`).** The loop repeats while the
   condition is false, the opposite of C.

3. **Case-insensitive keywords and identifiers.** `If`, `IF`, and `if`
   are all the same.

4. **Context-sensitive keywords.** `has`, `in`, `with`, `class`,
   `string`, `table`, etc., are only keywords in specific parsing
   contexts. They can be used as identifiers elsewhere.

5. **The `or` operator.** Unlike `||` (logical OR), `or` provides
   alternative values within a comparison: `if (x == 1 or 2 or 3)`.

6. **Property syntax in expressions.** The `.`, `..`, `.&`, `.#`, `..&`,
   `..#` operators for property access have distinct semantics and
   precedence levels — `.` and `..` are at level 12 while `.&`, `.#`,
   `..&`, `..#` are at level 10.

7. **No `else if` keyword.** Chained conditionals are written as
   `if ... else if ...` (two separate keywords), not a single `elseif`.

8. **Routine brackets.** Routines are delimited by `[` and `]`, not
   `{` and `}`. Braces are used only for compound statements within
   routine bodies.

9. **Action statements.** The `<Action>` and `<<Action>>` forms are
   unique statement types with their own syntax.

10. **The `give` statement.** Attribute manipulation uses its own
    statement form rather than assignment operators.
