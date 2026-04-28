# Chapter 7: Objects, Classes, and Inheritance

<!--
Copyright (C) 2026 Software Freedom Conservancy, Inc.

This document is free documentation: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This document is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this document. If not, see <https://www.gnu.org/licenses/>.
-->

Inform 6 is fundamentally an object-oriented language. Objects are the primary building blocks of any Inform program, representing everything from rooms and items to abstract concepts and the player character. This chapter defines the object system, including object declarations, the object tree, properties, attributes, classes, and inheritance.

---

## 7.1 Object Declarations

An object is declared with the `Object` directive. The full syntax is:

```
Object [->...] [name] ["short name"] [parent]
    [with    property_list]
    [private property_list]
    [has     attribute_list]
    [class   class_list];
```

All parts except `Object` and the terminating semicolon are optional.
The four body segments (`with`, `private`, `has`, `class`) may appear
in any order and may be repeated. `with` is described in §7.4, `has`
in §7.5, `class` in §7.7, and `private` in §7.14.

### 7.1.1 Basic Object Syntax

The simplest object declaration:

```inform6
Object lamp;
```

This creates an object with the internal identifier `lamp`, no short name, no parent, no properties, and no attributes.

A more typical declaration includes a short name, parent, properties, and attributes:

```inform6
Object lamp "brass lamp" study
    with name 'brass' 'lamp' 'lantern',
         description "A well-polished brass lamp.",
         before [;
             Burn: "The lamp is already quite hot.";
         ],
    has  light;
```

### 7.1.2 Components of an Object Declaration

- **Arrow (`->`)**: A run of arrows sets the new object's nesting depth to the number of arrows. The parent is then the most recently defined object whose own depth is one less than that. So `->` (depth 1) makes the new object a child of the most recent root-level object, `-> ->` (depth 2) makes it a child of the most recent depth-1 object, and so on.
- **Identifier**: The internal name used in source code to refer to the object. This is optional; anonymous objects can be created.
- **Short name**: A double-quoted string used for display to the player at runtime. It is stored in the object's built-in short-name slot (in the Z-machine object header, or as the `shortname` field of a Glulx object), which is what `print (object) obj` outputs. Note that this is distinct from the standard library's `short_name` *property*, which (when defined) the library's `(name)`/`(the)` printing routines consult in preference to the built-in short name.
- **Parent**: An identifier naming the parent object in the object tree. The object is placed as a child of this parent.

### 7.1.3 The Arrow Notation

The arrow notation provides a convenient way to build object hierarchies without explicitly naming parents:

```inform6
Object kitchen "Kitchen";

Object -> table "wooden table" with name 'table' 'wooden';

Object -> -> plate "china plate" with name 'plate' 'china';

Object -> chair "chair" with name 'chair';
```

Here, `table` and `chair` are children of `kitchen`. The `plate` is a child of `table`. Each `->` represents one level of nesting relative to the most recent object at the previous level.

---

## 7.2 The `Nearby` Directive

`Nearby` is a convenience form for declaring an object as a child of the
most recently defined root-level object:

```inform6
Object cave "Dark Cave";

Nearby rock "heavy rock"
    with name 'rock' 'heavy' 'boulder',
         description "An enormous boulder.";
```

It is exactly equivalent to `Object -> rock "heavy rock" ...`. In
particular, `Nearby` always sets the new object's tree depth to **1**
(child of the most recent depth-0 object); it does not vary its placement
based on the previous object's depth, and it is not equivalent to a chain
of nested arrows.

---

## 7.3 The Object Tree

All objects in an Inform program exist within a single tree structure. Every object (except root-level objects) has exactly one parent. Each object may have zero or more children. Siblings are children of the same parent, linked in a chain.

### 7.3.1 Tree Navigation Functions

The following system functions navigate the object tree:

| Function | Returns |
|---|---|
| `parent(obj)` | The parent of `obj`, or `nothing` (0) if `obj` is a root |
| `child(obj)` | The first child of `obj`, or `nothing` if no children |
| `children(obj)` | The number of direct children of `obj` |
| `sibling(obj)` | The next sibling of `obj`, or `nothing` if last |
| `elder(obj)` | The previous sibling of `obj`, or `nothing` if `obj` is the first child |
| `eldest(obj)` | Same as `child(obj)` — the first child |
| `younger(obj)` | Same as `sibling(obj)` — the next sibling |
| `youngest(obj)` | The last child of `obj` |

All eight tree-navigation functions above are system functions
recognised directly by the compiler. `parent`, `child`/`eldest`, and
`sibling`/`younger` translate straight to the underlying VM tree
opcodes; `children`, `youngest`, and `elder` are compiled inline as
short loops over those same opcodes.

### 7.3.2 Tree Manipulation Statements

Two statements modify the object tree at runtime:

```inform6
move obj to new_parent;
```

Removes `obj` from its current position in the tree and makes it the first child of `new_parent`. It is a runtime error to move an object to be a descendant of itself.

```inform6
remove obj;
```

Removes `obj` from the tree entirely, setting its parent to `nothing`. The object still exists but is no longer part of any hierarchy.

### 7.3.3 Iterating Over the Tree

The `objectloop` statement iterates over objects in the tree:

```inform6
! All children of a parent
objectloop (x in kitchen)
    print (name) x, "^";

! All objects in the game
objectloop (x)
    if (x has light) print (name) x, " gives light.^";
```

See Chapter 5 for the full `objectloop` syntax.

---

## 7.4 Properties

Properties are named data slots attached to objects. Each property holds one or more word-sized values.

### 7.4.1 Common Properties

Common properties are shared across all objects in the game. They are defined with the `Property` directive:

```inform6
Property weight;
```

Every object in the game can then use the `weight` property. If an object does not explicitly define a common property, accessing it returns the property's default value (0 unless set otherwise).

The number of common properties is limited:
- **Z-machine**: Common properties occupy property numbers 1–63 on
  Z-machine versions 4 and later, or 1–31 on version 3. Two of those
  numbered slots are reserved by the compiler for internal use, and
  property 1 is always the built-in `name` property, so the
  user-declarable limit is 61 common properties on v4 and later (one
  built-in `name` plus up to 60 declared with `Property`) and 29 on
  version 3 (one built-in `name` plus up to 28 declared with
  `Property`). The standard library declares 47 with `Property`,
  which together with `name` gives 48 user-visible common properties.
- **Glulx**: Limit depends on the `INDIV_PROP_START` setting (default
  256), so by default common properties are numbered 1–255. In
  practice the limit is much higher than the standard library
  requires.

A default value can be set:

```inform6
Property weight 0;
```

### 7.4.2 Individual Properties

Properties defined within a `Class` or `Object` that are not declared as common properties become individual properties. Individual properties exist only on objects that explicitly define them:

```inform6
Object widget "widget"
    with gizmo_factor 42;
```

If `gizmo_factor` was never declared with `Property`, it becomes an individual property. Attempting to read an individual property from an object that does not provide it causes a runtime error (unless runtime checking is disabled).

### 7.4.3 Property Values

A property can hold one or more word values:

```inform6
Object box "box"
    with name 'box' 'container' 'crate',
         description "A sturdy wooden crate.",
         capacity 10;
```

The `name` property holds three dictionary word values. The `description` property holds a string address. The `capacity` property holds the integer 10.

A property value may also be an embedded routine:

```inform6
Object door "oak door"
    with description [;
             if (self has open) "The oak door stands open.";
             "A heavy oak door, firmly closed.";
         ];
```

### 7.4.4 Property Access Operators

Properties are accessed with the `.` operator and related operators:

| Operator | Meaning |
|---|---|
| `obj.prop` | Read the first word of the property value |
| `obj.prop = val` | Write the first word of the property value |
| `obj.&prop` | Get the byte address of the property data |
| `obj.#prop` | Get the length of the property data in bytes |

For multi-valued properties, the address and length operators are essential:

```inform6
! Print all name words of an object
addr = obj.&name;
len = obj.#name / WORDSIZE;  ! WORDSIZE is 2 (Z) or 4 (Glulx)
for (i = 0 : i < len : i++)
    print (address) addr-->i, " ";
```

### 7.4.5 Additive Properties

A property declared as `additive` accumulates values through inheritance rather than being overridden:

```inform6
Property additive before;
```

When a class defines `before` and a subclass also defines `before`, the values are concatenated rather than replaced. The standard library declares `before`, `after`, `life`, `orders`, `describe`, `time_out`, and `each_turn` as additive. The compiler also treats its built-in `name` property as additive without an explicit declaration.

### 7.4.6 Long Properties

Any property can hold more than one word of data. This is a long property:

```inform6
Object thing
    with data 10 20 30 40;
```

The `data` property holds four words. Access individual words through the address:

```inform6
addr = thing.&data;
val = addr-->2;  ! Gets the third word (30)
```

---

## 7.5 Attributes

Attributes are boolean flags that can be set or cleared on any object. They are declared with the `Attribute` directive:

```inform6
Attribute explosive;
```

### 7.5.1 Attribute Limits

- **Z-machine**: Up to 48 attributes (numbered 0–47) on Z-machine
  versions 4 and later (32 attributes on version 3). The standard
  library declares approximately 32.
- **Glulx**: Configurable at compile time via the `$NUM_ATTR_BYTES`
  setting. The **default** is 7 bytes, giving 56 attributes; the
  maximum is 39 bytes, giving 312 attributes. Set it on the command
  line, e.g. `$NUM_ATTR_BYTES=15`, to expand the attribute space.

### 7.5.2 Setting and Testing Attributes

Attributes are set in object declarations with `has`:

```inform6
Object bomb "bomb" with name 'bomb', has explosive;
```

At runtime, attributes are manipulated with `give` and tested with `has`/`hasnt`:

```inform6
give bomb ~explosive;       ! Clear the explosive attribute
give bomb light;            ! Set the light attribute

if (bomb has explosive)     ! Test if set
    "It might explode!";

if (lamp hasnt light)       ! Test if cleared
    "The lamp is dark.";
```

---

## 7.6 The `Class` Directive

A class is a template for creating objects. Classes are defined with the `Class` directive:

```inform6
Class Treasure
    with value 0,
         description "A glittering treasure.",
    has  scored;
```

### 7.6.1 Creating Instances

Objects can be created from a class by naming the class before the object identifier:

```inform6
Treasure gold_coin "gold coin"
    with name 'gold' 'coin',
         value 50;
```

The `gold_coin` object inherits all properties and attributes from `Treasure`, with `value` overridden to 50.

### 7.6.2 Pre-Created Instances

A class can specify a number of pre-created duplicate instances, made
available as a pool for the `create`/`destroy`/`recreate`/`copy`
class methods (see §7.11):

```inform6
Class Weapon(10)
    with damage 1;
```

The `(10)` causes the compiler to manufacture 10 anonymous instances
of `Weapon` at compile time. Each duplicate is initially placed as a
child of the `Weapon` class object and serves as a ready-to-use
instance that `Weapon.create()` can hand out and `Weapon.destroy(obj)`
can return to the pool. The number of duplicates must be in the range
0 to 10000; if `(N)` is omitted, no duplicates are pre-created and the
class has no pool.

The `(N)` notation does **not** impose a global limit on how many
instances of the class may exist: ordinary instance declarations
(`Weapon sword ...;`) are unaffected by it.

### 7.6.3 Class as Object

Every `Class` is itself an object in the object tree. The class object can be referred to by name and used in expressions. `metaclass(Treasure)` returns `Class`.

---

## 7.7 Inheritance

### 7.7.1 Single Inheritance

When an object is declared as an instance of a class, it inherits all properties and attributes from that class. Explicitly specified properties override inherited ones (unless the property is additive):

```inform6
Class Container
    with capacity 10,
    has  container open;

Container chest "wooden chest"
    with name 'chest' 'wooden',
         capacity 20;   ! Overrides inherited value
```

### 7.7.2 Multiple Inheritance

An object or class can inherit from multiple classes:

```inform6
Class Lockable
    with with_key 0,
    has  lockable;

Class Container
    with capacity 10,
    has  container openable open;

Class LockableContainer
    class Lockable Container;

LockableContainer safe "iron safe"
    with name 'safe' 'iron',
         with_key brass_key;
```

When more than one class is listed, properties resolve in the following
order:

1. **The object's own `with` segment always wins.** Properties defined
   directly on the object override any value an inherited class would
   have supplied (for non-additive properties).
2. **For inherited non-additive properties**, the **first** class in the
   `class` list that defines the property contributes its value;
   later classes are ignored for that property.
3. **For additive properties** (declared with `Property additive ...`,
   for example `name` and `before`), values **accumulate** across the
   chain: the object's own value comes first, followed by the
   contributions of each listed class in declaration order.

### 7.7.3 Attribute Inheritance

Attributes are combined with logical OR across all parent classes. If any parent class has an attribute, the child inherits it. An explicit `has ~attr` in the child can clear an inherited attribute:

```inform6
Class Fixture has static scenery;

Fixture pillar "stone pillar"
    with name 'pillar' 'stone',
    has  ~scenery;  ! Overrides: still static but not scenery
```

---

## 7.8 The `metaclass()` Function

The `metaclass()` system function determines the type of a value at runtime:

| Value Type | `metaclass()` Returns |
|---|---|
| An ordinary object | `Object` |
| A class object | `Class` |
| A routine address | `Routine` |
| A string address | `String` |
| `nothing` (0) or invalid | `nothing` (0) |

```inform6
if (metaclass(x) == Object) print "It's an object.^";
if (metaclass(x) == Class)  print "It's a class.^";
```

---

## 7.9 The `ofclass` Operator

The `ofclass` operator tests whether an object is an instance of a given class:

```inform6
if (obj ofclass Treasure) print "It's a treasure!^";
```

An object is considered `ofclass` a class if:
- It was directly created from that class, or
- It inherits from that class through any chain of class inheritance.

The negated form uses standard Inform negation:

```inform6
if (obj ofclass Treasure) ...  ! positive test
if (~~(obj ofclass Treasure)) ...  ! negated test
```

---

## 7.10 The `provides` Operator

The `provides` operator tests whether an object has a given property:

```inform6
if (obj provides description)
    print (string) obj.description;
else
    print "You see nothing special.^";
```

This is essential for individual properties, which may not exist on all objects. Reading an individual property that an object does not provide returns 0 silently in non-strict builds, but causes a runtime error under strict-mode runtime checking (see §7.4.2), so `provides` should be used to check first.

---

## 7.11 The `create`, `recreate`, `destroy`, `copy`, and `remaining` Class Methods

Classes provide built-in methods for managing a pool of pre-created
instances (see §7.6.2). All of them are invoked on the *class* object,
not on an individual instance:

### 7.11.1 `create`

```inform6
new_obj = ClassName.create();
```

Removes one of the class's pre-created duplicates from the pool and
returns it as a fresh instance. Returns `false` (0) if the pool is
empty. Up to three extra arguments may be passed; if the new object
provides a `create` property, it is called with those arguments.

### 7.11.2 `recreate`

```inform6
ClassName.recreate(obj);
```

Resets `obj` to its initial class-defined state, restoring all
property values and attributes to those of the class prototype. `obj`
must be `ofclass ClassName`. Up to three extra arguments may be
passed, in which case the object's `create` property (if any) is then
called with them. This is useful for object pooling.

```inform6
Class Bullet(20)
    with damage 5,
         speed 100;

! Reset a used bullet to its original state
Bullet.recreate(bullet_obj);
```

### 7.11.3 `destroy`

```inform6
ClassName.destroy(obj);
```

Resets `obj` to its initial state and returns it to the class's pool
of available duplicates so that a subsequent `ClassName.create()`
call can re-issue it. `obj` must be `ofclass ClassName`. If the
object provides a `destroy` property, it is called first.

### 7.11.4 `copy`

```inform6
ClassName.copy(dst, src);
```

Copies attributes and property values from `src` to `dst`. The first
argument is the destination and the second is the source. Both `src`
and `dst` must be `ofclass ClassName`.

All attributes (positive and negative) are copied. For properties,
only those that are declared by **both** `src` and `dst` and have the
**same length** in both objects are copied; properties unique to one
side, or declared with different lengths, are left unchanged. The
copy is shallow: any addresses or object references stored in
property data are duplicated as-is, not deep-copied.

### 7.11.5 `remaining`

```inform6
n = ClassName.remaining();
```

Returns the number of unused duplicates still in the class's pool.

---

## 7.12 Message Passing

When a property holds a routine, accessing the property with `()` calls the routine as a message:

```inform6
obj.description();
```

This calls the embedded routine stored in `obj.description`, with `self` set to `obj`.

### 7.12.1 The `self` Variable

Inside any routine called as a property value (via message passing), the special variable `self` refers to the object that received the message:

```inform6
Object mirror "mirror"
    with description [;
             print "You look at ", (the) self, " and see your reflection.^";
         ];
```

`self` is automatically set before the routine is called and restored afterward. It should not be explicitly assigned in normal code.

### 7.12.2 `PrintOrRun` Semantics

When the library accesses a property value, it typically uses `PrintOrRun` semantics: if the value is a string, it is printed; if it is a routine, it is called. The `.` operator returns the first word of the property data, which may be either a string address or a routine address.

---

## 7.13 The Superclass Operator `::`

The `::` operator accesses a property as defined by a specific class, bypassing any overrides:

```inform6
val = obj.ClassName::property;
```

This retrieves the value of `property` as defined by `ClassName`, even if `obj` has overridden it:

```inform6
Class Vehicle with speed 60;

Vehicle car "car" with speed 120;

! car.speed is 120
! car.Vehicle::speed is 60
```

---

## 7.14 Property Visibility

Inform 6 supports a single, narrow form of access control: an
individual property may be declared in a `private` segment of an
object or class definition instead of (or in addition to) the usual
`with` segment.

```inform6
Object safe "iron safe"
    with description "A heavy iron safe.",
    private combination 4271;
```

A property declared in a `private` segment of an object can only be
read while the object itself is the current message recipient
(i.e. while `self == obj`). All of the runtime helpers that
implement individual-property access — those backing `obj.prop`,
`obj.&prop`, `obj.#prop`, and `obj provides prop` — check whether
`self == obj`; if not, a private property is reported as not
provided (the `.&` and `.#` operators return 0, `provides` returns
false, and reading the value with `.` triggers a "no such property"
runtime error in strict mode).

Restrictions on `private`:

- `private` applies only to **individual** properties. A name that has
  already been declared with the `Property` directive (i.e. a common
  property) cannot appear in a `private` segment.
- The check is enforced only at the runtime helpers. Code that
  reaches into the property table directly through raw memory
  (`->`/`-->` on a stored property address, for example) is not
  subject to the `self == obj` test.
- Apart from the `private` segment, there is no further notion of
  property privacy: any non-private property of any object is
  accessible from anywhere in the program through the property
  operators (§7.4.4) and the `provides` operator (§7.10), subject
  only to the runtime checks that confirm the object actually
  supplies that property.

---

## 7.15 Object Limits

| Limit | Z-machine | Glulx |
|---|---|---|
| Maximum objects | 255 (v3); v4+ limited only by memory | Limited only by memory |
| Maximum common properties | 31 (v3, of which 29 are user-declarable), 63 (v4+, of which 61 are user-declarable) | `INDIV_PROP_START - 1` (default 255) |
| Maximum attributes | 32 (v3), 48 (v4+) | `NUM_ATTR_BYTES × 8` (default 56, max 312) |
| Maximum data per common property | 8 bytes / 4 words (v3), 64 bytes / 32 words (v4+) | 32768 values per property |
| Maximum data per individual property | 64 bytes / 32 words | 32768 values per property |
| Maximum classes | Limited only by the maximum number of objects (every class is itself an object and consumes an object-number slot, so on v3 the four built-in metaclasses plus all user classes share the 255-object cap) | Limited only by memory (every class is an object and consumes an object-number slot) |

---

## 7.16 Object Numbering

Objects are numbered sequentially in the order they are encountered
in source code. The first four object numbers are reserved for the
built-in metaclasses, in this order: `Class` (1), `Object` (2),
`Routine` (3), and `String` (4). User-defined classes and objects
share a single counter and are numbered from 5 upward in declaration
order. The constant `nothing` (0) represents the absence of an
object.

In Z-machine version 3 the object number must fit in a single byte,
limiting the total to 255. In v4+ object numbers are 16-bit, and in
Glulx they are 32-bit; in both cases the practical limit is set by
available memory rather than a fixed cap.

Object numbers are stable within a single compilation but may change
if the source is modified. Code should always refer to objects by
name, never by number.

---

*Next: [Chapter 8: Arrays](ch08-arrays.md)*

---

Copyright (C) 2026 Software Freedom Conservancy, Inc. Licensed under the GNU General Public License v3.0 or later.
