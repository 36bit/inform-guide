# The Inform 6 Programmer's Guide

A comprehensive reference for the Inform 6 programming language, compiler, and
standard library.

This guide covers **Inform 6 compiler version 6.44** and **Inform standard
library version 6.12.8**.

## About This Guide

The Inform 6 Programmer's Guide is a freely licensed, community-maintainable
reference document for the Inform 6 programming language. It provides thorough
coverage of the language syntax and semantics, the compiler toolchain, the
standard library, and both target virtual machines (the Z-machine and Glulx).

The guide is written in Markdown and organized into a series of chapters grouped
into five major parts, plus front matter and appendices.

## Directory Structure

```
guide/
├── README.md                  This file
├── front-matter/              Title page, preface, and conventions
│   ├── title.md
│   ├── preface.md
│   └── conventions.md
├── part1-language/            The Inform 6 Language
├── part2-compiler/            The Inform 6 Compiler
├── part3-library/             The Inform Standard Library
├── part4-virtual-machines/    Target Virtual Machines (Z-machine & Glulx)
├── part5-advanced/            Advanced Topics
└── appendices/                Appendices and reference tables
```

### Front Matter

The `front-matter/` directory contains introductory material:

- **title.md** — Title page with version coverage and license information.
- **preface.md** — Preface describing the purpose, scope, audience, and
  organization of this guide.
- **conventions.md** — Typographic and notational conventions used throughout
  the guide.

### Part 1: The Inform 6 Language

`part1-language/` covers the Inform 6 programming language itself: lexical
structure, data types, expressions, statements, objects, classes, routines,
strings, and the grammar system.

### Part 2: The Inform 6 Compiler

`part2-compiler/` documents the Inform 6 compiler (version 6.44): command-line
usage, compilation switches, memory settings, ICL commands, error messages, and
the compilation model.

### Part 3: The Inform Standard Library

`part3-library/` describes the Inform standard library (version 6.12.8): the
world model, parsing and grammar, actions, the object tree, rooms and map
connections, the player, NPCs, light and darkness, scope, timers and daemons,
and the full set of library routines, entry points, and defined constants.

### Part 4: Target Virtual Machines

`part4-virtual-machines/` covers the two target platforms that the Inform 6
compiler can produce code for: the Z-machine and Glulx. This part documents
the capabilities and constraints of each VM, memory layout, instruction sets,
and how the compiler maps Inform 6 constructs onto each architecture.

### Part 5: Advanced Topics

`part5-advanced/` addresses advanced usage: replacing and extending library
routines, internationalization and localization, multimedia features, compiler
optimization, and debugging and testing techniques.

### Appendices

`appendices/` provides reference tables, grammar token charts, action lists,
attribute and property tables, and other supplementary material.

## How to Read This Guide

The guide is designed to work both as a linear tutorial and as a reference.

- **New to Inform 6?** Start with Part 1, which introduces the language from
  the ground up.
- **Looking up a specific topic?** Each chapter is self-contained enough to be
  read independently. Cross-references point you to related material elsewhere
  in the guide.
- **Experienced developers** may want to start with Part 3 (the library) or
  Part 5 (advanced topics) and refer back to earlier parts as needed.

Each Markdown file is intended to be read in a standard Markdown viewer, on
GitHub, or assembled into other formats (HTML, PDF, EPUB) using a document
processing toolchain.

## Version Coverage

| Component                | Version |
| ------------------------ | ------- |
| Inform 6 compiler        | 6.44    |
| Inform standard library  | 6.12.8  |

## Contributing and Feedback

Contributions, feedback, and problem reports are welcome. To contribute
changes, send feedback, or report problems, please email
[j@jxself.org](mailto:j@jxself.org).

## Copyright and License

Copyright (C) 2026 Software Freedom Conservancy, Inc.

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
