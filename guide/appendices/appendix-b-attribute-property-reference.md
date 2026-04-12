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

# Appendix B: Attribute and Property Reference

This appendix provides complete tables of every standard attribute and
property defined by the standard library and the compiler. For detailed
explanations, usage examples, and semantic discussions, see Chapter 18
(Common Properties) and Chapter 19 (Common Attributes).

---

## §B.1 Attributes

Attributes are single-bit boolean flags stored on every object. The library
declares them in `linklpa.h` using the `Attribute` directive. Each attribute
is assigned a number automatically, starting from 0, in declaration order.

### §B.1.1 Standard Attributes

The following 31 attributes are declared by the standard library in `linklpa.h`
(lines 34–65). The Number column shows the attribute number assigned by the
compiler in declaration order.

| Number | Name           | Alias            | Category     | Set By             | Description                                              | Reference |
|-------:|----------------|------------------|--------------|--------------------|----------------------------------------------------------|-----------|
|      0 | `animate`      |                  | Object type  | Game author        | Object is a living creature; enables `life` processing   | §19.2.1   |
|      1 | `absent`       | `non_floating`   | State        | Game author / lib  | Suppresses `found_in` floating; temporarily removes from play | §19.2.2   |
|      2 | `clothing`     |                  | Object type  | Game author        | Object can be worn by the player                         | §19.2.3   |
|      3 | `concealed`    |                  | Visibility   | Game author        | Hidden from room descriptions but reachable by name      | §19.2.4   |
|      4 | `container`    |                  | Object type  | Game author        | Object can hold other objects inside it                  | §19.2.5   |
|      5 | `door`         |                  | Object type  | Game author        | Object is a door connecting two rooms                    | §19.2.6   |
|      6 | `edible`       |                  | Physical     | Game author        | Object can be eaten (removed from play on `Eat`)         | §19.2.7   |
|      7 | `enterable`    |                  | Physical     | Game author        | Player can enter or sit on this object                   | §19.2.8   |
|      8 | `general`      |                  | State        | Game author        | General-purpose flag with no built-in semantics          | §19.2.9   |
|      9 | `light`        |                  | Physical     | Game author / lib  | Object provides illumination                             | §19.2.10  |
|     10 | `lockable`     |                  | Physical     | Game author        | Container or door can be locked/unlocked                 | §19.2.11  |
|     11 | `locked`       |                  | State        | Game author / lib  | Container or door is currently locked                    | §19.2.12  |
|     12 | `moved`        |                  | State        | Library            | Set when player first picks up the object                | §19.2.13  |
|     13 | `on`           |                  | State        | Library            | Switchable object is currently switched on               | §19.2.14  |
|     14 | `open`         |                  | State        | Game author / lib  | Container or door is currently open                      | §19.2.15  |
|     15 | `openable`     |                  | Physical     | Game author        | Container or door can be opened and closed               | §19.2.16  |
|     16 | `proper`       |                  | Gender/name  | Game author        | Proper noun — printed without an article                 | §19.2.17  |
|     17 | `scenery`      |                  | Visibility   | Game author        | Part of room description; not listed separately; not takeable | §19.2.18  |
|     18 | `scored`       |                  | State        | Game author        | Contributes to automatic scoring when visited/taken      | §19.2.19  |
|     19 | `static`       |                  | Visibility   | Game author        | Cannot be taken; listed normally in room descriptions    | §19.2.20  |
|     20 | `supporter`    |                  | Object type  | Game author        | Objects can be placed on top of this object              | §19.2.21  |
|     21 | `switchable`   |                  | Physical     | Game author        | Object has an on/off state controlled by `SwitchOn`/`SwitchOff` | §19.2.22  |
|     22 | `talkable`     |                  | Object type  | Game author        | Non-animate object can receive conversation commands     | §19.2.23  |
|     23 | `transparent`  |                  | Visibility   | Game author        | Contents visible even when closed                        | §19.2.24  |
|     24 | `visited`      |                  | State        | Library            | Room has been entered by the player at least once        | §19.2.25  |
|     25 | `workflag`     |                  | Internal     | Library            | Temporary flag used internally by library list routines  | §19.2.26  |
|     26 | `worn`         |                  | State        | Library            | Clothing is currently being worn by the player           | §19.2.27  |
|     27 | `male`         |                  | Gender/name  | Game author        | Object uses masculine pronouns (he/him)                  | §19.2.28  |
|     28 | `female`       |                  | Gender/name  | Game author        | Object uses feminine pronouns (she/her)                  | §19.2.29  |
|     29 | `neuter`       |                  | Gender/name  | Game author        | Object uses neuter pronouns (it/its)                     | §19.2.30  |
|     30 | `pluralname`   |                  | Gender/name  | Game author        | Object uses plural pronouns (they/them)                  | §19.2.31  |

**Alias:** `non_floating` is declared as an alias for `absent` on the same
source line. Both names refer to attribute number 1.

### §B.1.2 Conditional Attributes

The following attribute is only defined when the game is compiled with Infix
debugging support:

| Number | Name                | Condition         | Description                                       |
|-------:|---------------------|-------------------|----------------------------------------------------|
|     31 | `infix__watching`   | `#Ifdef INFIX`    | Marks an object as being watched by the Infix debugger |

When `INFIX` is not defined, attribute 31 is available for user-defined
attributes. When `INFIX` is defined, the compiler pre-creates the
`infix__watching` symbol as attribute 0 before processing any source files,
and the library conditionally declares it to
avoid a duplicate definition.

> **Note:** The compiler pre-creates `infix__watching` at number 0,
> but since `no_attributes` starts at 1 when `INFIX` is set,
> the library's `Attribute` declarations begin
> numbering from 1. The library's conditional `Attribute infix__watching`
> block in `linklpa.h` is guarded by `#Ifndef infix__watching` to avoid
> re-declaring a symbol the compiler already created.

### §B.1.3 VM Attribute Capacity

The number of attributes an object can hold is determined by the target
virtual machine:

**[Z-machine]** Each object has 6 attribute bytes, supporting 48 attributes
(numbered 0–47). The library uses attributes 0–30 (or 0–31 with Infix),
leaving 17–18 attributes available for game-specific use. The value 6 is
fixed and cannot be changed.

**[Glulx]** Each object has `NUM_ATTR_BYTES` attribute bytes, defaulting to
7 (56 attributes, numbered 0–55). This can be increased via the compiler
setting `$NUM_ATTR_BYTES`. The value must be a multiple of four, plus three
(i.e., 7, 11, 15, 19, …). The maximum is `MAX_NUM_ATTR_BYTES` = 39, giving
up to 312 attributes.

To define game-specific attributes, simply use the `Attribute` directive in
your source code after the library includes:

```inform6
Attribute is_magic;       ! Next available number after library attributes
Attribute is_fragile;
```

---

## §B.2 Common Properties (Library-Defined)

Common properties are declared by the library in `linklpa.h` using the
`Property` directive. Every object in the game has a slot for each common
property, initialized to the declared default value. The compiler also
creates three built-in common properties before any library code is processed.

### §B.2.1 Complete Property Table

Properties are numbered starting from 1. The compiler reserves properties
1–3 internally; the library declares
properties starting from number 4. The table
below lists all common properties in numeric order.

| Number | Name                | Default    | Additive | Value Type(s)                        | Description                                                     | Reference |
|-------:|---------------------|------------|:--------:|--------------------------------------|-----------------------------------------------------------------|-----------|
|      1 | `name`              | `0`        | Yes      | Dictionary words                     | Words the parser uses to recognize this object                  | §18.6.1   |
|      2 | *(class inheritance)* | `0`      | Yes      | *(compiler internal)*                | Stores class membership chain; not directly accessible          | —         |
|      3 | *(instance variables table)* | `0` | No    | *(compiler internal, Z-code only)*   | Address of individual property table; reserved in Glulx          | —         |
|      4 | `before`            | `NULL`     | Yes      | Routine                              | Intercepts actions before processing; return `true` to block    | §18.3.1   |
|      5 | `after`             | `NULL`     | Yes      | Routine                              | Called after successful action; return `true` to suppress message | §18.3.2   |
|      6 | `life`              | `NULL`     | Yes      | Routine                              | Handles actions directed at animate objects                     | §18.3.3   |
|      7 | `n_to`              | `0`        | No       | Room, door, routine, or string       | North exit destination                                          | §18.4.1   |
|      8 | `s_to`              | `0`        | No       | Room, door, routine, or string       | South exit destination                                          | §18.4.1   |
|      9 | `e_to`              | `0`        | No       | Room, door, routine, or string       | East exit destination                                           | §18.4.1   |
|     10 | `w_to`              | `0`        | No       | Room, door, routine, or string       | West exit destination                                           | §18.4.1   |
|     11 | `ne_to`             | `0`        | No       | Room, door, routine, or string       | Northeast exit destination                                      | §18.4.1   |
|     12 | `nw_to`             | `0`        | No       | Room, door, routine, or string       | Northwest exit destination                                      | §18.4.1   |
|     13 | `se_to`             | `0`        | No       | Room, door, routine, or string       | Southeast exit destination                                      | §18.4.1   |
|     14 | `sw_to`             | `0`        | No       | Room, door, routine, or string       | Southwest exit destination                                      | §18.4.1   |
|     15 | `u_to`              | `0`        | No       | Room, door, routine, or string       | Up exit destination                                             | §18.4.1   |
|     16 | `d_to`              | `0`        | No       | Room, door, routine, or string       | Down exit destination                                           | §18.4.1   |
|     17 | `in_to`             | `0`        | No       | Room, door, routine, or string       | In exit destination                                             | §18.4.1   |
|     18 | `out_to`            | `0`        | No       | Room, door, routine, or string       | Out exit destination                                            | §18.4.1   |
|     19 | `door_to`           | `0`        | No       | Room or routine                      | Destination when going through a door                           | §18.4.2   |
|     20 | `with_key`          | `0`        | No       | Object                               | Key object required to lock/unlock                              | §18.5.2   |
|     21 | `door_dir`          | `0`        | No       | Direction property or routine        | Direction property the door occupies                            | §18.4.3   |
|     22 | `invent`            | `0`        | No       | Routine                              | Custom inventory listing routine                                | §18.2.13  |
|     23 | `plural`            | `0`        | No       | String                               | Plural form of the object name                                  | §18.2.7   |
|     24 | `add_to_scope`      | `0`        | No       | Object, list, or routine             | Adds objects to scope dynamically                               | §18.6.2   |
|     25 | `list_together`     | `0`        | No       | String, number, or routine           | Groups objects together in listings                             | §18.2.14  |
|     26 | `react_before`      | `0`        | No       | Routine                              | Reacts to any action while this object is in scope (before)     | §18.3.4   |
|     27 | `react_after`       | `0`        | No       | Routine                              | Reacts to any action while this object is in scope (after)      | §18.3.5   |
|     28 | `grammar`           | `0`        | No       | Routine                              | Custom grammar for NPC command parsing                          | §18.6.3   |
|     29 | `orders`            | `0`        | Yes      | Routine                              | Handles commands given to this NPC                              | §18.3.6   |
|     30 | `initial`           | `0`        | No       | String or routine                    | Description of object in initial (unmoved) state                | §18.2.8   |
|     31 | `when_open`         | `0`        | No       | String or routine                    | Description when container/door is open                         | §18.2.9   |
|     32 | `when_closed`       | `0`        | No       | String or routine                    | Description when container/door is closed                       | §18.2.9   |
|     33 | `when_on`           | `0`        | No       | String or routine                    | Description when switchable object is on                        | §18.2.10  |
|     34 | `when_off`          | `0`        | No       | String or routine                    | Description when switchable object is off                       | §18.2.10  |
|     35 | `description`       | `0`        | No       | String or routine                    | Printed when player examines the object                         | §18.2.4   |
|     36 | `describe`          | `NULL`     | Yes      | Routine                              | Custom room-listing routine; return `true` to override default  | §18.2.11  |
|     37 | `article`           | `"a"`      | No       | String                               | Indefinite article for the object                               | §18.2.5   |
|     38 | `cant_go`           | `0`        | No       | String or routine                    | Message printed when movement is blocked                        | §18.4.4   |
|     39 | `found_in`          | `0`        | No       | Room list, class list, or routine    | Rooms/conditions where a floating object appears                | §18.6.4   |
|     40 | `time_left`         | `0`        | No       | Number                               | Turns remaining on a running timer                              | §18.7.3   |
|     41 | `number`            | `0`        | No       | Number                               | General-purpose numeric value for author use                    | §18.7.4   |
|     42 | `time_out`          | `NULL`     | Yes      | Routine                              | Called when timer expires (`time_left` reaches 0)               | §18.7.2   |
|     43 | `daemon`            | `0`        | No       | Routine                              | Per-turn daemon routine; use `StartDaemon`/`StopDaemon`         | §18.7.1   |
|     44 | `each_turn`         | `NULL`     | Yes      | Routine                              | Called each turn while object is in scope                       | §18.7.5   |
|     45 | `capacity`          | `100`      | No       | Number                               | Maximum number of objects this container/supporter can hold     | §18.5.1   |
|     46 | `short_name`        | `0`        | No       | String or routine                    | Printed name of the object in listings                          | §18.2.2   |
|     47 | `short_name_indef`  | `0`        | No       | String or routine                    | Object name in indefinite contexts                              | §18.2.3   |
|     48 | `parse_name`        | `0`        | No       | Routine                              | Custom parsing routine for complex name matching                | §18.6.1   |
|     49 | `articles`          | `0`        | No       | Array                                | Language-specific article forms                                 | §18.2.6   |
|     50 | `inside_description`| `0`        | No       | String or routine                    | Printed when the player is inside this object                   | §18.2.12  |

### §B.2.2 Notes on Specific Properties

**The `name` property (number 1):** This property is created by the compiler
itself, not by the library. It is long and additive.
A special compiler rule applies: when a double-quoted string appears as a
value of the `name` property, the compiler treats it as a dictionary entry
rather than a static string. The compiler
issues a warning if non-dictionary values are used in `name`.

**Additive properties:** Seven library properties are declared additive:
`before`, `after`, `life`, `orders`, `describe`, `time_out`, and
`each_turn`. When a class defines an additive property and a subclass or
instance also defines it, the values **accumulate** (are concatenated) rather
than being overridden. This is essential for action-handling chains where
both a class and an instance need to process actions. The `name` property
(compiler built-in) is also additive.

**Properties 2 and 3 (compiler internal):** Property 2 stores the class
inheritance chain and property 3 stores the instance variables table address.
Both are created by the compiler. Property 3
is only meaningful in Z-code; in Glulx its slot is reserved but unused.
Neither property is accessible by name in source code.

**The `article` property:** Defaults to the string `"a"` rather than `0`,
so most objects automatically receive the indefinite article "a". Objects
with the `proper` attribute skip article printing entirely.

**The `capacity` property:** Defaults to `100`, setting the baseline for
how many objects a container, supporter, or the player can hold.

---

## §B.3 Compiler Built-in Individual Properties (Class System)

The compiler defines eight individual properties used by the class system.
These are created during symbol initialization and are available on every class object.

### §B.3.1 Individual Property Table

| Z-code Number | Glulx Number          | Name              | Description                                                        |
|--------------:|----------------------:|-------------------|--------------------------------------------------------------------|
|            64 | `INDIV_PROP_START+0`  | `create`          | Called when a new instance is created from a class                  |
|            65 | `INDIV_PROP_START+1`  | `recreate`        | Re-initializes an existing instance to class defaults               |
|            66 | `INDIV_PROP_START+2`  | `destroy`         | Called when an instance is being destroyed/recycled                 |
|            67 | `INDIV_PROP_START+3`  | `remaining`       | Returns the number of unallocated instances of a class             |
|            68 | `INDIV_PROP_START+4`  | `copy`            | Copies all property values from one instance to another            |
|            69 | `INDIV_PROP_START+5`  | `call`            | Calls a routine stored in an individual property by number         |
|            70 | `INDIV_PROP_START+6`  | `print`           | Prints the short name of an object                                 |
|            71 | `INDIV_PROP_START+7`  | `print_to_array`  | Prints the short name of an object into a byte array               |

These properties are individual properties (not common properties). They
cannot be accessed with the ordinary `.` operator on non-class objects.
They are called as methods on class objects:

```inform6
Treasure remaining_count = Treasure.remaining();
obj = Treasure.create();
Treasure.destroy(obj);
```

### §B.3.2 INDIV_PROP_START

The `INDIV_PROP_START` constant marks the boundary between common properties
and individual properties in the property numbering space.

**[Z-machine]** `INDIV_PROP_START` is fixed at 64 and cannot be changed.
The compiler enforces this. Common properties
are numbered 1–63; individual properties start at 64.

**[Glulx]** `INDIV_PROP_START` defaults to 256 and can be set to any value
≥ 256 via the compiler setting `$INDIV_PROP_START=N`. Common properties are numbered 1–255;
individual properties start at 256 (or higher if configured).

User-defined individual properties — those defined using `with` on an object
or class but not declared globally with `Property` — are numbered from
`INDIV_PROP_START+8` upward (Z-code: 72+, Glulx: 264+). The compiler tracks
this via `no_individual_properties`, initialized to `INDIV_PROP_START+8`.

---

## §B.4 Property Numbering Summary

The following diagram shows the complete property number space for both
target platforms:

### §B.4.1 Z-machine Property Number Space

```
Number    Usage
------    -----
     0    (unused)
     1    name                          (compiler built-in, common)
     2    (class inheritance)           (compiler internal, common)
     3    (instance variables table)    (compiler internal, common)
  4–50    Library common properties     (from linklpa.h)
 51–63    Available for user-defined common properties
    64    create                        (class-system individual property)
    65    recreate                      (class-system individual property)
    66    destroy                       (class-system individual property)
    67    remaining                     (class-system individual property)
    68    copy                          (class-system individual property)
    69    call                          (class-system individual property)
    70    print                         (class-system individual property)
    71    print_to_array                (class-system individual property)
  72+     User-defined individual properties
```

Z-code allows a maximum of 63 common properties (numbered 1–63). With the
library occupying properties 1–50, there are 13 slots available for
user-defined common properties (51–63).

### §B.4.2 Glulx Property Number Space

```
Number      Usage
------      -----
       0    (unused)
       1    name                        (compiler built-in, common)
       2    (class inheritance)         (compiler internal, common)
       3    (instance variables table)  (compiler internal, reserved)
    4–50    Library common properties   (from linklpa.h)
  51–255    Available for user-defined common properties
     256    create                      (class-system individual property)
     257    recreate                    (class-system individual property)
     258    destroy                     (class-system individual property)
     259    remaining                   (class-system individual property)
     260    copy                        (class-system individual property)
     261    call                        (class-system individual property)
     262    print                       (class-system individual property)
     263    print_to_array              (class-system individual property)
   264+     User-defined individual properties
```

Glulx allows up to `INDIV_PROP_START-1` common properties (default: 255).
With the library occupying properties 1–50, there are 205 slots available
for user-defined common properties (51–255).

---

## §B.5 Accessing Properties at Runtime

The following operators are available for reading and manipulating property
values at runtime. For full details, see §4.9 (Property Access Expressions)
and §7.3 (Properties).

| Operator      | Syntax                 | Description                                                    |
|---------------|------------------------|----------------------------------------------------------------|
| `.`           | `obj.prop`             | Reads or writes the first word of a property value             |
| `.&`          | `obj.&prop`            | Returns the byte address of the property data                  |
| `.#`          | `obj.#prop`            | Returns the length of the property data in bytes               |
| `::`          | `Class::prop`          | Accesses the original value of a property as defined by a class |
| `provides`    | `obj provides prop`    | Tests whether the object provides the named property           |
| `has`         | `obj has attr`         | Tests whether the object has the named attribute               |
| `hasnt`       | `obj hasnt attr`       | Tests whether the object lacks the named attribute             |

**Reading a property value:**

```inform6
x = lamp.description;            ! Read the description property
addr = lamp.&name;                ! Get address of name data
len = lamp.#name;                 ! Get byte length of name data
count = lamp.#name / WORDSIZE;    ! Number of words in name
```

**Testing and modifying attributes:**

```inform6
if (obj has light) ...            ! Test for attribute
if (obj hasnt visited) ...        ! Test for absence
give obj light;                   ! Set attribute
give obj ~light;                  ! Clear attribute
```

**Testing property existence:**

```inform6
if (obj provides description)     ! True if obj defines 'description'
    print (string) obj.description;
```

**Class property access:**

```inform6
x = MyClass::capacity;           ! Original class-defined value
```

---
