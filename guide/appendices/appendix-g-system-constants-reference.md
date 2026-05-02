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

# Appendix G: System Constants Reference

This appendix provides a complete reference for all 63 system constants
recognized by the Inform 6 compiler (version 6.44). System constants
provide named access to compiler-internal values such as table addresses,
symbol counts, and memory layout offsets. They are accessed using the `#`
prefix in expressions:

```inform6
x = #dictionary_table;      ! Address of the dictionary
x = #highest_object_number;  ! Number of the last object
```

For the formal grammar, see §K.2.9. For an overview of system constant
syntax and the `#` prefix, see §1.10.4.

---

## §G.1 Overview

System constants belong to the `SYSTEM_CONSTANT_TT` token type. When the
compiler encounters `#` followed by one of the recognized names listed
below, it resolves the name to an integer value. Some system constants are
resolved immediately at compile time (those whose values depend only on the
compiler version or settings), while others are backpatched — their final
values are filled in during the linking phase after all tables have been
laid out.

**VM availability.** Most system constants are available on both the
Z-machine and Glulx, but some are restricted to one VM. The table entries
below indicate availability with **Z** (Z-machine only), **G** (Glulx
only), or **Z/G** (both).

**Debug dependency.** Several system constants refer to the symbol table
embedded in the story file for debugging purposes. When `$OMIT_SYMBOL_TABLE`
is set, these constants are unavailable and produce a compile-time error.
They are marked with † in the tables below.

---

## §G.2 Compilation and Version Constants

These constants provide information about the compilation itself.

| Constant | VM | Description |
|----------|-----|-------------|
| `version_number` | Z | The Z-machine version number being targeted (3, 4, 5, 6, 7, or 8). Resolved at compile time. |
| `oddeven_packing` | Z | The state of the `-B` (odd/even packing) switch: 1 if enabled, 0 if disabled. Resolved at compile time. This switch controls whether routines and strings are packed to different address ranges. |

---

## §G.3 Object and Class Constants

These constants provide counts and table addresses related to objects and
classes.

| Constant | VM | Description |
|----------|-----|-------------|
| `largest_object` | Z | The highest object number with a base offset of 256, equal to 256 + (total number of objects) − 1. This is a legacy value used in some older Inform code. |
| `actual_largest_object` | Z | The total count of objects defined in the game file, without any offset. |
| `lowest_object_number` | Z/G | Always 1. The lowest valid object number. |
| `highest_object_number` | Z/G | The highest valid object number. The valid range of object numbers is `#lowest_object_number` through `#highest_object_number` inclusive. |
| `lowest_class_number` | Z/G | Always 0. Classes are numbered starting from 0. |
| `highest_class_number` | Z/G | The index of the last class, equal to (total classes) − 1. |
| `class_objects_array` | Z/G | Address of the class-numbers table. This table maps class indices to their corresponding object numbers. On Glulx, the address is in RAM. |

---

## §G.4 Attribute Constants

These constants provide the range and name table for attributes.

| Constant | VM | Description |
|----------|-----|-------------|
| `lowest_attribute_number` | Z/G | Always 0. Attributes are numbered from 0. |
| `highest_attribute_number` | Z | The index of the last attribute, equal to (total attributes) − 1. |
| `attribute_names_array` † | Z | Address of the attribute-names table in the story file. Contains packed string addresses for attribute names, indexed by attribute number. |

---

## §G.5 Property Constants

These constants provide the range and name table for properties (including
individual properties).

| Constant | VM | Description |
|----------|-----|-------------|
| `lowest_property_number` | Z/G | Always 1. Properties are numbered starting from 1. |
| `highest_property_number` | Z | The index of the last individual property, equal to (total individual properties) − 1. This count includes both common and individual properties. |
| `property_names_array` † | Z | Address of the property-names table. Equal to the identifiers table address plus 2. Contains packed string addresses for property names. |

---

## §G.6 Action Constants

These constants provide counts, ranges, and name tables for actions and
fake actions.

| Constant | VM | Description |
|----------|-----|-------------|
| `lowest_action_number` | Z/G | Always 0. Actions are numbered from 0. |
| `highest_action_number` | Z/G | The index of the last action, equal to (total actions) − 1. |
| `action_names_array` † | Z/G | Address of the action-names table. Contains packed string addresses for action names. On Glulx, the address is in RAM. |
| `lowest_fake_action_number` | Z/G | The starting number for fake actions. In grammar version 1: 256. In grammar version 2: 4096. Resolved at compile time. |
| `highest_fake_action_number` | Z/G | The number of the last fake action, equal to `lowest_fake_action_number` + (total fake actions) − 1. |
| `fake_action_names_array` † | Z/G | Address of the fake-action-names table. On Glulx, the address is in RAM. |
| `highest_meta_action_number` | Z/G | The highest action number flagged as meta, or −1 (as a 16-bit value, `$FFFF` on Z-machine) if no meta actions exist. Requires `$GRAMMAR_META_FLAG` to be set; produces a compile-time error otherwise. See §22.3 for details on meta action sorting. |

---

## §G.7 Table Address Constants

These constants provide addresses of major compiler-generated tables in the
story file.

| Constant | VM | Description |
|----------|-----|-------------|
| `adjectives_table` | Z | Address of the adjectives table. [Z-machine only; not implemented on Glulx.] |
| `actions_table` | Z/G | Address of the actions table, which maps action numbers to routine addresses. On Glulx, this is the address of the action dispatch table. |
| `classes_table` | Z/G | Address of the class-numbers table, mapping class indices to object numbers. On Glulx, the address is in RAM. |
| `identifiers_table` † | Z/G | Address of the identifiers table, which contains names of properties, attributes, and other symbols for debugging. On Glulx, the address is in RAM. |
| `preactions_table` | Z | Address of the preactions table. [Z-machine only; not implemented on Glulx.] |
| `grammar_table` | Z/G | Address of the grammar table (verb definitions). On Z-machine, this equals the static memory offset. On Glulx, this is the grammar table offset. |
| `dictionary_table` | Z/G | Address of the dictionary in the story file. |
| `dynam_string_table` | G | [Glulx only] Address of the dynamic string table, used for `@streamstr` and printing variables. On Glulx, this is the abbreviations offset. |

---

## §G.8 Memory Layout Constants

These constants provide addresses and offsets describing the memory layout
of the compiled story file.

| Constant | VM | Description |
|----------|-----|-------------|
| `strings_offset` | Z | The packed address of the start of the strings area, divided by the Z-machine packing scale factor. |
| `code_offset` | Z | The packed address of the start of the code area, divided by the Z-machine packing scale factor. |
| `static_memory_offset` | Z | The byte address where static memory begins in the story file. On the Z-machine, this is also the grammar table location. |
| `readable_memory_offset` | Z | The byte address where readable (writable) memory ends and the code area begins (the `Write_Code_At` boundary). |
| `cpv__start` | Z/G | Start address of the common property values area. On Z-machine, this is the property values offset. On Glulx, this is the property defaults offset. |
| `cpv__end` | Z/G | End address of the common property values area. Equal to the class-numbers table address. On Glulx, the address is in RAM. |
| `ipv__start` | Z | Start address of the individual property values area. |
| `ipv__end` | Z | End address of the individual property values area. Equal to the start of the global variables area. |
| `array__start` | Z | Start address of the arrays area in the story file. |
| `array__end` | Z | End address of the arrays area. Equal to the static memory offset. |

---

## §G.9 Dictionary Parameter Constants

These constants provide byte offsets within dictionary entries for
accessing the flag fields. They are used by the library to test dictionary
word flags (such as noun, verb, and preposition markers).

| Constant | VM | Description |
|----------|-----|-------------|
| `dict_par1` | Z/G | Byte offset within a dictionary entry for the first flag field. On Z-machine v3: 4; on v4+: 6. On Glulx: `DICT_ENTRY_FLAG_POS + 1` (points to the lower byte of a two-byte flag field). Resolved at compile time. |
| `dict_par2` | Z/G | Byte offset for the second flag field. On Z-machine v3: 5; on v4+: 7. On Glulx: `DICT_ENTRY_FLAG_POS + 3`. Resolved at compile time. |
| `dict_par3` | Z/G | Byte offset for the third flag field. On Z-machine v3: 6; on v4+: 8. On Glulx: `DICT_ENTRY_FLAG_POS + 5`. Resolved at compile time. Unavailable when `$ZCODE_LESS_DICT_DATA` is set. |

---

## §G.10 Symbol Table Constants

These groups of constants provide access to the compiler's symbol table
data embedded in the story file. They are primarily used by debugging tools
and the `Inform Library` itself for runtime introspection. Each category of
symbol follows a consistent pattern:

- `lowest_X_number` — the first valid index (always 0 or 1)
- `highest_X_number` — the last valid index
- `Xs_array` or `X_names_array` — address of the data/names table
- `X_flags_array` — address of the flags table (where applicable)

All name and flag arrays are only meaningful when the symbol table is
included in the story file. Constants marked with † produce a compile-time
error when `$OMIT_SYMBOL_TABLE` is set.

### §G.10.1 Routine Constants

| Constant | VM | Description |
|----------|-----|-------------|
| `lowest_routine_number` | Z/G | Always 0. Named routines are numbered from 0. |
| `highest_routine_number` | Z | The index of the last named routine, equal to (total named routines) − 1. |
| `routines_array` | Z | Address of the routines table. Contains packed addresses of all named routines. |
| `routine_names_array` † | Z | Address of the routine-names table. Contains packed string addresses for routine names. |
| `routine_flags_array` | Z | Address of the routine-flags table. Contains a byte of flags for each routine. |

### §G.10.2 Global Variable Constants

| Constant | VM | Description |
|----------|-----|-------------|
| `lowest_global_number` | Z/G | Always 16. On the Z-machine, globals 0–15 are reserved for the VM's local variable area. User globals start at index 16. |
| `highest_global_number` | Z | The index of the last global variable, equal to 16 + (total user globals) − 1. |
| `globals_array` | Z/G | Address of the global variables area. On Z-machine, this is the variables offset. On Glulx, this is the variables offset. |
| `global_names_array` † | Z | Address of the global-names table. |
| `global_flags_array` | Z | Address of the global-flags table. |

### §G.10.3 Array Constants

| Constant | VM | Description |
|----------|-----|-------------|
| `lowest_array_number` | Z/G | Always 0. Arrays are numbered from 0. |
| `highest_array_number` | Z | The index of the last array, equal to (total arrays) − 1. |
| `arrays_array` | Z | Address of the arrays table. Contains the starting addresses of all arrays. |
| `array_names_array` | Z | Address of the array-names table. |
| `array_names_offset` † | Z/G | Address of the array-names offset within the identifiers table. On Glulx, the address is in RAM. |
| `array_flags_array` | Z | Address of the array-flags table. |

### §G.10.4 Named Constant Constants

| Constant | VM | Description |
|----------|-----|-------------|
| `lowest_constant_number` | Z/G | Always 0. Named constants are numbered from 0. |
| `highest_constant_number` | Z | The index of the last named constant, equal to (total named constants) − 1. |
| `constants_array` | Z | Address of the constants table. Contains the values of all named constants. |
| `constant_names_array` † | Z | Address of the constant-names table. |

---

## §G.11 VM Availability Summary

The following table summarizes which system constants are available on each
VM. Constants listed as "Z" produce a compile-time error if used when
targeting Glulx, and vice versa.

**Z-machine only** (not implemented on Glulx):

`adjectives_table`, `preactions_table`, `version_number`,
`largest_object`, `strings_offset`, `code_offset`,
`actual_largest_object`, `static_memory_offset`,
`readable_memory_offset`, `ipv__start`, `ipv__end`,
`array__start`, `array__end`, `highest_attribute_number`,
`attribute_names_array`, `highest_property_number`,
`property_names_array`, `highest_routine_number`, `routines_array`,
`routine_names_array`, `routine_flags_array`,
`highest_global_number`, `global_names_array`,
`global_flags_array`, `highest_array_number`, `arrays_array`,
`array_names_array`, `array_flags_array`,
`highest_constant_number`, `constants_array`,
`constant_names_array`, `oddeven_packing`.

**Glulx only**:

`dynam_string_table`.

**Both VMs**:

`actions_table`, `classes_table`, `identifiers_table`,
`dict_par1`, `dict_par2`, `dict_par3`,
`array_names_offset`, `cpv__start`, `cpv__end`,
`lowest_attribute_number`, `lowest_property_number`,
`lowest_action_number`, `highest_action_number`,
`action_names_array`,
`lowest_fake_action_number`, `highest_fake_action_number`,
`fake_action_names_array`,
`lowest_routine_number`, `lowest_global_number`, `globals_array`,
`lowest_array_number`, `lowest_constant_number`,
`lowest_class_number`, `highest_class_number`,
`class_objects_array`,
`lowest_object_number`, `highest_object_number`,
`grammar_table`, `dictionary_table`,
`highest_meta_action_number`.

---

## §G.12 Usage Examples

### Iterating Over All Actions

```inform6
[ ListActions i;
    for (i = #lowest_action_number : i <= #highest_action_number : i++)
        print i, ": ", (string) #action_names_array-->i, "^";
];
```

### Checking the Z-machine Version

```inform6
[ CheckVersion;
    if (#version_number >= 5)
        print "Advanced Z-machine features available.^";
    else
        print "Running on Z-machine version ", #version_number, ".^";
];
```

### Accessing Dictionary Flags

```inform6
[ WordIsVerb w;
    ! Check the verb flag in the dictionary entry
    if ((w->#dict_par1) & 1)
        print "This word is a verb.^";
];
```

### Testing for Meta Actions

```inform6
! Requires $GRAMMAR_META_FLAG to be set
[ IsMetaAction act;
    return (act <= #highest_meta_action_number);
];
```

---

*Copyright © 2026 Software Freedom Conservancy, Inc.*
*Licensed under the GNU General Public License, version 3 or later.*
