# Preface

## Why This Guide Exists

Inform 6 is a mature, powerful programming language purpose-built for creating
interactive fiction. Since its introduction in the mid-1990s, the language has
been used to produce thousands of works of interactive fiction, and it continues
to be actively developed, extended, and relied upon by a worldwide community of
authors and developers.

The Inform 6 compiler and standard library have continued to evolve over the
years. New compiler releases have introduced Glulx support, expanded memory
models, additional language features, and numerous bug fixes and quality-of-life
improvements. The standard library has likewise grown, with new entry points,
refined parsing behavior, improved support for both target virtual machines, and
corrections to long-standing issues. These ongoing changes mean that developers
need reference documentation that accurately reflects the current state of the
tools they are using.

However, the existing reference literature for Inform 6 has not kept pace with
these developments. The primary reference works for the language were written
to cover much earlier versions of the compiler and library, and they are
published under proprietary terms that prevent the community from updating,
correcting, or extending them. As a result, developers must supplement outdated
reference material with scattered release notes, errata documents, community
forum posts, and direct source-code reading. This situation is particularly
difficult for newcomers, who have no single authoritative source that documents
the language as it exists today.

The Inform 6 Programmer's Guide was created to fill this gap. It is a
comprehensive, up-to-date reference for the Inform 6 language, compiler, and
standard library, published under the GNU General Public License so that it can
be freely redistributed, corrected, and improved by anyone. This guide is
intended to be the definitive reference that the Inform 6 community can maintain
and evolve alongside the tools it documents.

## Scope

This guide provides comprehensive coverage of four major areas:

1. **The Inform 6 Language.** The full syntax and semantics of the Inform 6
   programming language, including lexical structure, data types, expressions,
   statements, objects and classes, routines, strings, the grammar definition
   system, and all other language constructs. The language is documented as
   implemented by version 6.44 of the compiler.

2. **The Inform 6 Compiler.** The operation and configuration of the Inform 6
   compiler, version 6.44. This includes command-line invocation, compilation
   switches and settings, memory configuration, ICL (Inform Command Language)
   commands, source file organization, conditional compilation, error and
   warning messages, and the overall compilation model.

3. **The Inform Standard Library.** The standard library, version 6.12.8,
   which provides the world model, parser, action-processing framework, and
   runtime infrastructure used by most Inform 6 programs. This includes the
   object hierarchy, parsing and grammar, actions and action processing,
   rooms and the map, the player object, non-player characters, light and
   darkness, scope, timers and daemons, menus and other UI elements, and the
   complete set of library routines, entry points, properties, attributes,
   global variables, and defined constants.

4. **The Target Virtual Machines.** The two virtual machines that the Inform 6
   compiler targets: the Z-machine and Glulx. This includes the capabilities
   and constraints of each platform, memory architecture, how Inform 6
   constructs map onto each VM, and practical guidance on choosing between
   them and writing portable code.

In addition, an advanced topics section addresses techniques for extending and
customizing the library, working at a low level with VM memory, performance
tuning, and other subjects relevant to experienced Inform 6 developers.

## Audience

This guide is written for anyone who programs in Inform 6 or needs to
understand Inform 6 source code:

- **Newcomers to Inform 6** will find that the guide introduces the language
  systematically, building from fundamental concepts to advanced features. No
  prior knowledge of Inform 6 is assumed, though familiarity with general
  programming concepts is expected.

- **Experienced Inform 6 developers** will find this guide useful as a
  reference for the precise behavior of language features, compiler switches,
  library routines, and VM capabilities. The guide aims to be thorough enough
  to serve as a definitive reference when questions arise about exactly how
  some feature works.

- **Library and extension authors** will benefit from the detailed coverage of
  the standard library's architecture, its extension and replacement
  mechanisms, and the advanced topics section.

- **Tool developers and platform maintainers** will find the compiler and
  virtual machine sections useful for understanding the compilation model and
  target platform constraints.

## What This Guide Is Not

This guide is a **language and tools reference**. It documents the Inform 6
programming language, the compiler, the standard library, and the target
virtual machines.

This guide is **not a game design manual**. It does not teach the craft of
interactive fiction writing—narrative structure, puzzle design, pacing, player
psychology, or the many other considerations that go into creating a compelling
work of interactive fiction. While the examples in this guide often involve
typical interactive fiction scenarios (rooms, objects, characters, and actions),
the focus is always on how to express these things in Inform 6 rather than on
whether they make for good game design.

This guide is also **not a tutorial for absolute beginners to programming**.
Readers are expected to understand basic programming concepts such as
variables, control flow, data types, and functions. The guide does not teach
these concepts from scratch, though it explains how they are realized in
Inform 6.

## Organization

This guide is organized into five parts, plus front matter and appendices:

**Part 1: The Inform 6 Language** introduces the programming language itself.
It covers lexical conventions, data types, constants and variables, operators
and expressions, statements and control flow, routines, objects and classes,
strings and text, the grammar definition system, and message passing. By the
end of Part 1, the reader should have a thorough understanding of every
construct available in the Inform 6 language.

**Part 2: The Inform 6 Compiler** documents the compiler toolchain. It covers
command-line usage and switches, memory and table settings, ICL commands,
source file inclusion and organization, conditional compilation, compiler
directives, linking, and the compilation model. It also documents the
compiler's error and warning messages.

**Part 3: The Inform Standard Library** describes the standard library that
provides the runtime framework for most Inform 6 programs. It covers the world
model (the object tree, containment, supporters, doors, and other spatial
concepts), the parser and grammar system at a practical level, actions and
action processing, the player object, non-player characters, light and
darkness, scope rules, timers and daemons, scoring, the status line, and all
library-provided routines, entry points, properties, attributes, and constants.

**Part 4: Target Virtual Machines** covers the Z-machine and Glulx, the two
platforms that the compiler can produce executables for. It documents the
capabilities, limitations, and memory architecture of each platform, explains
how Inform 6 language constructs map onto VM operations, and provides guidance
for writing code that works on both platforms or that takes advantage of
platform-specific features.

**Part 5: Advanced Topics** addresses subjects relevant to experienced Inform 6
developers. This includes techniques for extending and replacing library
routines, writing reusable library extensions, working directly with VM memory,
manipulating the dictionary and grammar tables at runtime, performance
considerations, and interfacing with external systems where the target VM
supports it.

**Appendices** provide quick-reference material including grammar token tables,
action reference charts, attribute and property listings, character set tables,
compiler memory setting reference, and other supplementary data.

## Acknowledgments

This guide was made possible by the long history of the Inform 6 community:
the compiler and library authors and maintainers, the writers of extensions and
tools, the authors of interactive fiction works in Inform 6, and the many
community members who have answered questions, reported bugs, written
tutorials, and kept the ecosystem alive and growing for decades. Their
collective work forms the foundation on which this guide is built.
