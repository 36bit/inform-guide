# Pass 3 — Part 1 Language chapters

Audit against Inform 6.44 canonical source (`Inform6-6.44/`) and
`I6-Addendum.md`. Sections where I use the word "confirmed" were checked
against the named source file directly.

## Summary table

| Chapter | OK | MINOR | WRONG | MISSING-CITATION | UNVERIFIABLE |
|---------|----|-------|-------|------------------|--------------|
| ch01 — Lexical structure | ~majority | 2 | 1 | 0 | 1 |
| ch02 — Types and values | ~majority | 1 | 2 | 0 | 0 |
| ch03 — Variables and scope | ~majority | 1 | 2 | 0 | 0 |
| ch04 — Expressions/Operators | ~majority (NUM_OPERATORS 68, level range 0–13 confirmed) | 2 | 0 | 1 | 1 |
| ch05 — Statements | ~majority (31 statement codes match states.c) | 1 | 0 | 0 | 1 |
| ch06 — Routines | ~majority | 1 | 0 | 0 | 0 |
| ch07 — Objects/Classes | majority | 1 | 3 | 0 | 0 |
| ch08 — Arrays (5 types confirmed) | ~majority | 2 | 1 | 0 | 0 |
| ch09 — Dictionary | ~majority (7 DFLAGs confirmed) | 1 | 0 | 0 | 0 |
| ch10 — Directives (40/40 covered) | ~majority | 1 | 1 | 0 | 0 |

---

## ch01 — Lexical structure

### WRONG
- **§1.1.1 "ZSCII — the Z-machine's 10-bit character set (values 0–1023)."**
  ZSCII is a 10-bit set per the Z-Machine Standard, but in practice the
  defined ZSCII range goes 0–1023 only nominally; the compiler stores and
  uses values up to 1023 but characters above 255 are rare. Statement is
  acceptable but the word "10-bit" is OK. Keep OK. (Removed after re-read.)
  _Actually nothing WRONG here; disregard._ (Moved to NOTE.)
- **§1.3.2 token-type codes**: The table lists `UQ_TT = 4` as a distinct
  type. In lexer.c the code names match, but `UQ_TT` (unquoted) is in fact
  emitted only from specific lexer entry points. Minor-only.

  The sole clearly WRONG claim in ch01 is in **§1.5.2**:
  "`$DeadBeef ! 3735928559 (Glulx only — too large for Z-machine)`".
  $DEADBEEF = 3735928559 = 0xDEADBEEF, but Inform numeric literals are
  stored as 32-bit signed internally (header.h/lexer.c); on Z-machine this
  value would be truncated to $BEEF = 48879 (not rejected), so the "too
  large for Z-machine" wording is misleading — it is truncated, not
  forbidden. (Per §1.5.6 of the same chapter the rule is "silently
  truncated".) MINOR/WRONG.

### MINOR
- **§1.1.1 character-representation list** numbers ZSCII as "10-bit (0–1023)".
  chars.c comments describe ZSCII with values up to 1023 reserved, but
  only codes in the Z-Machine Standard (0–255 primary, 155–251 for
  accented, with further zscii-to-unicode extension) are actually used.
  The claim is technically accurate but slightly misleading; a reader
  may think all 0–1023 values are ordinary characters. MINOR.
- **§1.4.4 Reserved words**: the **Directives** list is 40 items
  (Abbreviate … Zcharacter). This matches the 40 keywords in
  `header.h:1377-1416` / the `directives` keyword_group. Confirmed OK.
  This corrects the task prompt's claim of "34 per prompt".

### UNVERIFIABLE
- **§1.3.3 "12,288 possible lexical states"** — not checked against
  source; origin in lexer.c unknown without deeper audit.

---

## ch02 — Types and values

### WRONG
- **§2.5.1 Object numbering / MAX_OBJECTS**: "In versions 5 and 8, the
  limit is configurable via the `MAX_OBJECTS` compiler setting but
  defaults to 512."
  In 6.44, `MAX_OBJECTS` is listed in `options.c:438` as
  `OPTUSE_OBSOLETE_I6` — it is **obsolete** and no longer configurable.
  Array growth is dynamic. The Z-machine cap (v4+) is 65535, not 512.
  WRONG.
- **§2.5.1 Class/Object numbering order** ("Classes defined with `Class`
  receive the lowest numbers, followed by objects defined with `Object`.")
  Not matched by objects.c: classes and objects are numbered in
  declaration order (Class `Class` is object 1, then each class and each
  object follows source order). Potentially WRONG — source does not
  segregate class numbers from object numbers. Flag for verification.

### MINOR
- "`'x'` is interpreted by the compiler as the ASCII value of the
  character x" — on Z-machine the value is the ZSCII code (which for
  0x20–0x7E coincides with ASCII); on Glulx it is the Unicode code point.
  Saying "ASCII" is imprecise. MINOR.

---

## ch03 — Variables and scope

### WRONG
- **§3.6.1 `self` global slot** — "`self` occupies global slot 255
  (the last slot)".
  `arrays.c:877` shows the default Z layout assigns
  `globalv_z_self = 251`. The layout is
  `temp_var1=255, temp_var2=254, temp_var3=253, temp_var4=252,
   self=251, sender=250, sw__var=249`. So `self` is **slot 251**, not 255.
  WRONG.
- **§3.6.2 `sender` global slot** — "`sender` occupies global slot 249".
  Per `arrays.c:878`, `sender = 250`. WRONG (off by one; 249 is
  `sw__var`).

### MINOR
- "`self` ... occupies global index `MAX_LOCAL_VARIABLES + 4` (index 123)
  … `sender` ... `MAX_LOCAL_VARIABLES + 5` (index 124) on Glulx" —
  confirmed by `symbols.c:955,957`. OK.
- §3.1.4 "On Glulx it is 119" for MAX_LOCAL_VARIABLES — confirmed.
- §3.1.2 "approximately 233 slots" — 240 − 7 reserved = 233 exactly
  (temp_var1-4, self, sender, sw__var per symbols.c:921-927). Claim is
  fine; "approximately" is understated but not wrong.

---

## ch04 — Expressions and operators

### Structural verification
- **NUM_OPERATORS**: `header.h:1716` defines `NUM_OPERATORS 68`.
  The `operators[]` table in `expressc.c:92-321` has exactly 68 entries.
  ch04 does not state a count, so no contradiction.
- **Precedence levels**: operators span level 0 through level 14
  in `expressc.c`. Level 14 has a single Glulx-only "push on stack"
  internal entry. The guide's §4.13 table covers levels 0–13 = 14 levels
  (matches prompt claim "14 precedence levels"). OK — level 14 is an
  internal compiler operator, not user-written.
- **Per-operator audit (representative)**:
  - `,` → L0 IN L_A   ✓ (expressc.c:98)
  - `=` → L1 IN R_A   ✓ (expressc.c:104)
  - `&& || ~~` → L2   ✓ (111-116)
  - `==, ~=, >=, >, <=, <, has, hasnt, in, notin, ofclass, provides` → L3 ✓
    (127-158)
  - `or` → L4 IN L_A ✓ (164)
  - `+ -` → L5 ✓ (170-171)
  - `* / % & | ~` → L6 ✓ (178-187)
  - `-> -->` → L7 L_A ✓ (193-196)
  - unary `-` → L8 PRE R_A ✓ (202)
  - `++ -- (pre/post)` → L9 ✓ (210-217)
  - `.& .# ..& ..#` → L10 ✓ (224-231)
  - `(` (call) → L11 ✓ (237)
  - `. ..` → L12 ✓ (244-247)
  - `::` → L13 ✓ (253)
  All match the §4.13 table.

### MINOR
- **§4.4.4 / §4.13.1** explains that `~~` has nominal precedence 2 but
  "effectively" binds less tightly (parsed as `~~(A && B)`). This matches
  I6-Addendum's "effective precedence 1.5". The note is correct, but
  describing this as a "shift-reduce parser" subtlety is inaccurate —
  `expressp.c` uses an operator-precedence parser with a precedence
  comparison table (see `find_prec`). MINOR (mechanism description, not
  semantics).
- **§4.13.1 "Function call precedence anomaly"** — the stated behavior
  is correct (11 vs 12 interaction), but the explanatory sentence about
  "a special rule in the precedence comparison" is oversimplified:
  the real mechanism is the `prec_table` in `expressp.c` and the way
  postfix `(` is handled. MINOR.

### MISSING-CITATION
- §4.13 table assigns precedence numbers without citing the source
  (`expressc.c:92-321`). Since the numbers are verbatim from that
  table, a footnote or citation would be useful.

### UNVERIFIABLE
- §4.12.2 "Message sends compile to the `CA__Pr` veneer routine" —
  plausible (veneer.c defines `CA__Pr`), but not verified for every
  call form. Low-risk.

---

## ch05 — Statements and control flow

### Structural verification
Statement set defined in `states.c` (per `grep case *_CODE:`):
`box, break, continue, do, font, for, give, if, inversion, jump, move,
new_line, objectloop, print, print_ret, quit, read, remove, restore,
return, rfalse, rtrue, save, spaces, string, style, switch, while` plus
handler cases for `default`, `else`, `until`. The `statements` keyword
group in ch01 lists all of these, and ch05 documents each. Matches
source. OK.

### MINOR
- **§5.17 `inversion`**: "This reads bytes 60–63 of the story file header
  (the 'Inform version' field)." Z-Machine header places the 4-byte
  Inform version string at bytes $3C–$3F = 60–63. Correct for Z-machine;
  however the guide doesn't note that on Glulx the offset differs (Glulx
  stores the compiler version as 4 bytes near the start of the ROM header,
  not at offset 60). MINOR gap.

### UNVERIFIABLE
- **§5.23 `string` statement** — "The first argument is the slot number
  (0–31 on most Z-machine versions)" — I did not verify the exact upper
  bound against states.c/STRING_CODE, but documented bound matches the
  Z-Machine specification's LOWSTRINGS count for v4+. Low risk.

---

## ch06 — Routines

### MINOR
- **§6.4.1 "excess arguments are silently ignored on the Z-machine"** —
  the Z-machine `call_vs`/`call_vs2` family limits total arguments to 7;
  passing more than the routine declared but within the call-opcode limit
  is ignored at the callee side (the callee doesn't see them). Passing
  more than 7 to a call is a *compile-time* error (§6.7.2). Claim is
  consistent but slightly oversimplified; clarifying "beyond the
  call-opcode limit it is a compile-time error, otherwise silently
  discarded" would be more precise. MINOR.

Otherwise ch06's treatment of `return`/`rtrue`/`rfalse`, default return
conventions (stand-alone → true, embedded → false), and `_vararg_count`
(verified in `asm.c:1773`, `veneer.c:1088+`) match source. OK.

---

## ch07 — Objects, classes, inheritance

### WRONG
- **§7.6.2 Class instance count**: "`Class Weapon(10)` ... The `(10)`
  means at most 10 instances of `Weapon` may exist."
  `objects.c:1973-1981` parses the parenthesised number as the number of
  **duplicate instances to manufacture automatically**
  (`duplicates_to_make = n + 1`). It is not a cap on user instances — the
  user may create further `Weapon foo;` declarations after the class,
  regardless of the number. WRONG.
- **§7.14 Private Properties**: "Properties defined inside a `Class` body
  that are not common properties are private by default."
  Not supported by `objects.c`. Only properties introduced under the
  `private` segment marker (`PRIVATE_SEGMENT`, objects.c:1823, 1217-1229)
  are private. Properties under the `with` segment are accessible from
  outside. WRONG.
- **§7.14 "Attempting to access `obj.hidden_data` from outside the class
  results in a runtime error."** — follows from the previous wrong
  premise; not the actual compiler behavior. WRONG.

### MINOR
- **§7.5.1 Z-machine attributes** — "Up to 48 attributes (numbered 0–47).
  The standard library uses approximately 30." Z-machine v3 has only 32
  attributes; v4+ has 48. The guide fails to distinguish versions in the
  main text (only says "Up to 48"). Minor gap.
- **§7.15 limits table** — "Maximum objects: 255 (v3), 65535 (v4+);
  Maximum objects Glulx: Configurable (`MAX_OBJECTS`)". MAX_OBJECTS is
  obsolete in 6.44 (same issue as ch02). MINOR/WRONG (already noted in
  ch02).
- Tree navigation function `children(obj)` documented as "The number of
  direct children of `obj`" — this is a veneer-provided function; works
  correctly but not strictly a VM primitive. OK (documented fine).

---

## ch08 — Arrays

### Structural verification
- Five array types (`->`, `-->`, `string`, `table`, `buffer`) confirmed
  in `header.h:1807-1811`: `BYTE_ARRAY 0, WORD_ARRAY 1, STRING_ARRAY 2,
  TABLE_ARRAY 3, BUFFER_ARRAY 4`. Matches §8.2.

### WRONG
- **§8.3.1 "[Z-machine] The maximum array size is 32767 entries."**
  No such explicit limit exists. Arrays occupy dynamic memory, whose
  upper bound depends on the Z-machine dynamic-memory cap (~64K minus
  globals/objects/dictionary). The practical byte limit for a byte array
  is well under 65535 bytes; for a word array about 32767 entries (byte
  size 65534), so the stated number is a (rough) upper bound for word
  arrays but not byte arrays (which can hold up to ~65535 entries).
  WRONG/oversimplified; the guide states it as a hard universal rule.

### MINOR
- **§8.8.1 Input buffer** — "The Z-machine `read` opcode expects a byte
  array … byte 0 holds the maximum number of characters allowed, and
  after reading, byte 1 holds the number of characters typed, with
  characters stored from byte 2 onwards." This matches Z-Machine Standard
  v4+; for v3 the layout is byte 0 = max, byte 1 onward = chars
  (null-terminated), with no "chars typed" byte. The guide doesn't
  mention the v3/v5 distinction. MINOR.
- **§8.2.5 Buffer arrays** — "the standard library uses this layout for
  text input buffers (see §8.8), where `buffer-->0` stores a character
  count". Confirmed by veneer library conventions. OK.

---

## ch09 — Dictionary

### Structural verification
- Dict flags and values confirmed against `header.h:1301-1314`:
  `NONE_DFLAG 0, VERB_DFLAG 1, META_DFLAG 2, PLURAL_DFLAG 4,
   PREP_DFLAG 8, SING_DFLAG 16, BIT5_DFLAG 32 (unused),
   TRUNC_DFLAG 64, NOUN_DFLAG 128`. Matches §9.4 table.
- Suffix-flag syntax (`//p`, `//s`, `//n`, `//~p` …) matches
  documented text encoding in text.c.
- `$DICT_WORD_SIZE`, `$DICT_CHAR_SIZE`, `$DICT_IMPLICIT_SINGULAR`,
  `$DICT_TRUNCATE_FLAG`, `$ZCODE_LESS_DICT_DATA` all exist
  (`header.h:2624-2636`, `directs.c:1148`).

### MINOR
- **§9.8.3 "In version 3, the offsets are `#dict_par1` = 4, `#dict_par2`
  = 5, `#dict_par3` = 6. In versions 4+, the offsets are `#dict_par1`
  = 6, `#dict_par2` = 7, `#dict_par3` = 8."** — Correct for the default
  3-byte dict-data configuration; when `ZCODE_LESS_DICT_DATA` is set the
  third byte is omitted and `#dict_par3` does not exist. The guide does
  not note the effect of `ZCODE_LESS_DICT_DATA` on these offsets. MINOR.

Otherwise the Z-machine entry format (4-byte word + 3 data bytes in v3;
6-byte word + 3 data bytes in v4+) and Glulx entry format (1 tag +
DICT_WORD_SIZE × DICT_CHAR_SIZE word + 3 × 2 data bytes) match text.c
and tables.c. OK.

---

## ch10 — Compiler directives

### Structural verification — all 40 directives covered
Cross-reference against `directives` keyword_group
(`header.h:1377-1416`; 40 keywords):

Abbreviate ✓§10.6.1, Array ✓§10.1.3, Attribute ✓§10.2.4, Class ✓§10.2.3,
Constant ✓§10.1.1, Default ✓§10.1.4, Dictionary ✓§10.7.1, End ✓§10.10,
Endif ✓§10.5 (implicit), Extend ✓§10.7.2, Fake_action ✓§10.7.3,
Global ✓§10.1.2, Ifdef ✓§10.5.1, Ifndef ✓§10.5.1, Ifnot ✓§10.5.3,
Ifv3 ✓§10.5.4, Ifv5 ✓§10.5.4, Iftrue ✓§10.5.2, Iffalse ✓§10.5.2,
Import ✓§10.11, Include ✓§10.3.1, Link ✓§10.3.2, Lowstring ✓§10.6.2,
Message ✓§10.8.1, Nearby ✓§10.2.2, Object ✓§10.2.1, Origsource ✓§10.8.3,
Property ✓§10.2.5, Release ✓§10.4.1, Replace ✓§10.3.3, Serial ✓§10.4.2,
Switches ✓§10.4.5, Statusline ✓§10.4.3, Stub ✓§10.3.4,
System_file ✓§10.3.5, Trace ✓§10.8.2, Undef ✓§10.9.1, Verb ✓§10.7.2,
Version ✓§10.4.4, Zcharacter ✓§10.6.3 = **40/40**.

### WRONG
- **§10.6.2 Lowstring** — "The constant `MyText` holds the packed address
  divided by 2. To print the string: `print (address) (2 * MyText);`"
  The `(address)` print rule is for *byte* addresses of dictionary words,
  not for packed-string low-strings. Low-strings are normally printed
  via the `@@` escape or via the `string` statement's slot mechanism
  (see ch05 §5.23). Using `(address)` here will not produce the expected
  result. WRONG.

### MINOR
- **§10.4.4 "Valid values are 3, 4, 5, 6, 7, and 8."** The compiler
  today accepts 3,4,5,6,7,8 (`directs.c` `Version` case). v6 support is
  partial. OK.
- **§10.3.4 `Stub`** — "takes a specified number of local variables
  (0–4)". Valid range per `directs.c` Stub handling is 0–4 on Z-machine
  and larger on Glulx (tracked against `MAX_LOCAL_VARIABLES`). MINOR.

Otherwise ch10's directive semantics (including the special `Replace
name newname` form since 6.33, the obsoleted `Import`/`Link`, the
behaviour of `Default`, `Undef`, `Message`, `Trace`) match directs.c.

---

## Cross-cutting findings (Part 1)

1. **Directive count reconciliation**: ch01 §1.4.4 lists 40 directive
   keywords; ch10 documents 40/40; this matches `header.h:1377-1416` and
   corrects the task-prompt figure of "34". Appendix K's count of 40 is
   vindicated.

2. **MAX_OBJECTS obsolete** — mentioned as configurable in ch02 §2.5.1
   and ch07 §7.15. Both are stale relative to 6.44 (`options.c:438`,
   flagged OPTUSE_OBSOLETE_I6). The guide should state the limit is set
   dynamically.

3. **Reserved-global slot numbers** inconsistent between ch03 (§§3.6.1,
   3.6.2) and actual `arrays.c:873-879` assignments. The correct Z-machine
   layout for the default configuration is:
   255=temp_var1, 254=temp_var2, 253=temp_var3, 252=temp_var4,
   251=self, 250=sender, 249=sw__var. The guide's slot 255 for `self`
   and slot 249 for `sender` are wrong.

4. **Operator-table size consistency**: ch04 §4.13 table enumerates
   precedence 0–13 (14 levels); this matches user-accessible operators
   in `expressc.c:92-321`. Internal level 14 ("push on stack", Glulx) is
   not user-writeable and its omission is acceptable. The **total 68**
   operator entries (`NUM_OPERATORS`, `header.h:1716`) is made up of
   the user-visible operators plus 20 lvalue-assignment/inc-dec variants
   plus 2 property-call variants plus 1 Glulx push; the guide does not
   mention the 68 count explicitly, which avoids a potential confusion.

5. **Class semantics**: ch07 §7.6.2 and §7.14 both misdescribe Inform's
   class model — `Class(N)` is duplicate-manufacture count (not a cap),
   and "`with`"-properties are public (not private by default). These
   are material errors; the only private mechanism is the `private`
   segment marker (objects.c PRIVATE_SEGMENT).

6. **v3 vs v4+ distinctions** are sometimes glossed: ch07 attribute
   limit (32 vs 48), ch08 read-buffer layout (no chars-typed byte in v3),
   ch09 dictionary-entry size, ch05 `font off` and `style` support
   (V4+ only — ch05 does note this).

---

_End of pass 3._
