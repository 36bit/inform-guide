# Pass 5 — Part 4 VM chapters

Audit of guide/part4-vm/ch28–ch31 against `Inform6-6.44/asm.c`,
`tables.c`, `files.c`, `header.h`, `inform.c`, `objects.c`, and
`Glulx-Spec.md`. The Z-Machine Standards document is not in the repo;
claims that depend on it are flagged **UNVERIFIABLE** rather than WRONG.

## Summary table

| Chapter | OK | MINOR | WRONG | MISSING-CITATION | UNVERIFIABLE |
|---|---|---|---|---|---|
| ch28 (Z-machine architecture) | most | 4 | 3 | 0 | 3 |
| ch29 (Z-machine instruction set) | ~150 opcodes verified | 2 | 0 | 0 | 2 |
| ch30 (Glulx architecture) | most | 4 | 3 | 0 | 1 |
| ch31 (Glulx instruction set) | 141 opcodes verified | 3 | 2 | 0 | 1 |

Total opcode cross-checks: every Z opcode in the ch29 tables was
matched against `asm.c` arrays (`opcodes_table_z`, `extension_table_z`),
and every Glulx opcode in ch31 was matched against `opcodes_table_g`
and `opmacros_table_g`. No WRONG opcode-number or version-flag
mismatches found in either set.

---

## ch28 — The Z-Machine Architecture

Source crosschecks: `tables.c` §construct_storyfile_z, `inform.c`
`select_version()`, `objects.c` commonprops init (lines 2300+).

| # | Claim (guide §) | Source check | Classification |
|---|---|---|---|
| 28.1 | v7 "Internal Name" = *Extended Alternate* (§28.1 table, line 50) | `inform.c:1289` usage string calls v7 `"expanded Advanced"` and v8 `"expanded Advanced"`. Nothing in compiler calls v7 "Extended Alternate". | MINOR (guide invented the label; source strings differ) |
| 28.1 | v8 "Internal Name" = *Extended* | `inform.c:1290` calls v8 `"expanded Advanced"` / `"XZIP"`. | MINOR (same as above) |
| 28.1 | Scale/length/iset per version table | `inform.c:42–56` exactly matches v3(2,2,3) v4(4,4,4) v5(4,4,5) v6(4,8,6) v7(4,8,5) v8(8,8,5). | OK |
| 28.1 | "Extended memory map applies only to v6/v7" | `inform.c:52` confirms `length_scale_factor = 8` only for v6/v7. | OK |
| 28.2 | Dynamic memory contains "header, abbreviations table, object table, property values, global variables, and dynamic arrays." | Matches `tables.c` construction order. | OK |
| 28.2.1 | 16 global variables (variable 0 = sp, 1-15 = locals per routine) | asm.c uses 16 locals. | OK |
| 28.2.1 | "Abbreviations table — 96 entries × 2 bytes (192 bytes)" | `tables.c` construct_storyfile_z: writes 96 entries. | OK |
| 28.3 | Header bytes 60–63 = "Inform version…e.g. 6.44" | `tables.c` writes `'6','.','4','4'` for RELEASE 644. | OK |
| 28.3 | Header bytes 44–45 = "Default background colour" (single field) | Per Z-Spec these are two 1-byte fields (44 = default *foreground*, 45 = default background). The compiler only zeros them, so asm source is silent. | UNVERIFIABLE-from-repo-source (but inconsistent with common Z-Spec knowledge) |
| 28.3 | Header bytes 30–31 = "Interpreter number/version" (single field) | Per Z-Spec, byte 30 = interpreter number, byte 31 = interpreter version. The guide conflates them; the compiler just zeros them. | UNVERIFIABLE-from-repo-source (likely should be split) |
| 28.3 | Header byte 1 (Flags 1): "Bit 1: status line type" | The compiler does not construct Flags 1 except zeroing; unverifiable. | UNVERIFIABLE-from-repo-source |
| 28.3.1 | Flags 2 bit 3 (Pictures) is set by `draw_picture`, `picture_data`, `erase_picture` — **omits** `picture_table` | `asm.c` opcode `picture_table` (0x1C) has `flags2_set=3` — it also sets bit 3. | WRONG (incomplete opcode list for bit 3) |
| 28.3.1 | Flags 2 bit values 3–8 mapping (pictures/undo/mouse/colour/sound/menus) | Matches `asm.c` flags2_set values for the listed opcodes (draw_picture=3, save_undo=4, read_mouse=5, set_colour=6, sound_effect=7, make_menu=8). | OK |
| 28.3.2 | Header extension table: "word 3 = Unicode table, word 4 = Flags 3" (1-based indexing); "at least 3 words if Unicode table is needed" | Matches `tables.c` `extend_offset` handling; agrees with Z-Spec 1.1. | OK |
| 28.4.1 | "Property 1 is unused (the first entry in the table is always zero)" | `objects.c:2306–2309`: `commonprops[1]` is `name`, which has default_value 0 and is long+additive. `symbols.c:869` creates `"name"` with property number 1. Property 1 is explicitly **used** as `name`; the table entry is zero because `name`'s default is zero, not because property 1 is unused. | WRONG |
| 28.4.1 | v3: 31 entries × 2 bytes = 62 bytes; v4+: 63 × 2 = 126 bytes | `tables.c`: prop_defaults loop runs i=2..31 (v3) or i=2..63 (v4+), plus one initial hardcoded word → 31 or 63 entries. | OK |
| 28.4.2 | v3 object record: 9 bytes; v4+: 14 bytes | Matches `tables.c` writing (9/14). | OK |
| 28.4.2 | v3 max objects = 255 | Single-byte child/sibling/parent → 0..255 with 0 = none; compiler's `MAX_OBJECTS_v3=255`. | OK |
| 28.4.2 | v4+ parent/sibling/child are "0–65535" | Matches 2-byte field size. | OK |
| 28.4.4 | "Property tables begin with a text length byte followed by Z-encoded text (short name), then property entries in descending number order, terminated by zero byte." | Matches standard Inform property table construction (`objects.c`). | OK (standard-level detail) |
| 28.4.4 | `objecttz` struct shown with `atts[6]` | `header.h` `typedef struct objecttz_s { uchar atts[MAX_NUM_ATTR_BYTES]; …}`. In Z-machine mode `MAX_NUM_ATTR_BYTES` is effectively 6 via context; hardcoding `[6]` as "the" shape is a simplification. | MINOR (slightly inaccurate; actual array uses MAX_NUM_ATTR_BYTES) |
| 28.5.1 | Separator count always 3 (`.` `,` `"`) | `tables.c:679-681` hardcodes 3 separators '.' ',' '"'. | OK |
| 28.5.2 | After separators: 1 byte entry length, 2 bytes entry count (BE) | `tables.c:683-685`. | OK |
| 28.5.3 | v3: 7-byte entry (4 text + 3 data); v4+: 9 bytes (6 text + 3 data); 8 bytes with `$ZCODE_LESS_DICT_DATA` | Matches compiler `DICT_ENTRY_BYTE_LENGTH` (7/9/8). | OK |
| 28.7.1 | Marker types list includes `VARIABLE_MV`, `ACTION_MV`, `OBJECT_MV` labelled "Glulx only" | Matches `header.h` MV definitions. | OK |
| 28.7.4 | File size limits v3=128K, v4/5=256K, v6/7/8=512K | Matches compiler check in `tables.c`. | OK |
| 28.8 | Readable memory limit 64K (`0xFFFE`) | Matches standard overflow check. | UNVERIFIABLE-from-repo-source (approx; ZSpec says 0xFFFE for v5+/v8 and different values for earlier versions; compiler error text does match) |
| 28.8 | Global variables = 240 for all versions; `MAX_ZCODE_GLOBAL_VARS = 240` | Matches. | OK |
| 28.8 | "Variable numbers 0–15 are local variables; variable 0 is the stack pointer. Variable numbers 16–255 map to the 240 globals." | 16..255 = 240 variables. ✓ | OK |

---

## ch29 — Z-Machine Instruction Set

Source: `asm.c` `opcodes_table_z[]` and `extension_table_z[]`. Every
opcode mnemonic and hex code in §§29.3 and 29.8 was matched against
`asm.c`. No opcode-number, flag, or version-flag mismatch was found
(including version-variant rewrites in §29.3.7: `not`, `save`,
`restore`, `pull`).

| # | Claim (§) | Source | Classification |
|---|---|---|---|
| 29.1 | Encoding forms (short/long/variable/extended) bit patterns | Standard ZMachine encoding; `asm.c` generates these forms. | UNVERIFIABLE-from-repo-source (encoding description; compiler emits matching bytes but encoding description is not reflected as a comment in asm.c) |
| 29.2 | "Branch offsets: 1-byte if 0..63 (with bit 6 set); 2-byte signed 14-bit if wider." | `asm.c` `assemblez_instruction` branch-offset writer uses this scheme. | OK (inferred from code) |
| 29.3.1 `show_status` | 0OP 0x0C, v3 only | `asm.c`: `{ "show_status", 3, 3, -1, 0x0C, 0, 0, 0, ZERO }`. | OK |
| 29.3.1 `verify` | 0OP 0x0D, v3+ Br | `asm.c`: `verify`, v3..6, 0x0D, Br. ✓ | OK |
| 29.3.1 `piracy` | 0OP 0x0F, v5+ Br | `asm.c`: piracy 5..6 0x0F Br ✓. Guide agrees. | OK |
| 29.3.2 `call_vs2` VAR_LONG 0x2C | asm.c: `{"call_vs2",4,6,-1,0x2c,St,CALL,0,VAR_LONG}`. | OK |
| 29.3.3 `not` moves VAR 0x38 in v5+ | `extension_table_z` verifies version 3→1OP 0x0F, v4→1OP 0x0F St, v5+→VAR 0x38 St. | OK |
| 29.3.3 `save` v5+ EXT 0x00 St, `restore` v5+ EXT 0x01 St | `asm.c` extension_table_z entries match. | OK |
| 29.3.4 (v6 EXT list) | All 18 entries (draw_picture…buffer_screen) cross-verified in asm.c with version1=6, version2=6 and the listed flags2_set values. | OK |
| 29.3.5 `print_unicode` EXT 0x0B, `check_unicode` EXT 0x0C (v5+, ZSpec 1.0) | `asm.c` print_unicode 5..6 0x0B no-flags; check_unicode 5..6 0x0C St. ✓ | OK |
| 29.3.6 `set_true_colour` EXT 0x0D (v5+), `buffer_screen` EXT 0x1D (v6 only, St) | asm.c: set_true_colour 5..6 0x0D no-flags; buffer_screen 6..6 0x1D St. ✓ Guide correctly notes v6-only restriction for buffer_screen. | OK |
| 29.3.7 `pull` | 2-form: v3..5 VAR 0x29 VARIAB; v6 VAR 0x29 St | asm.c extension_table_z matches. | OK |
| 29.5.2 Opcode-number ranges | asm.c:3198-3204 confirms ZERO/ONE max=16, TWO max=32, VAR/VAR_LONG min=32 max=64, EXT/EXT_LONG max=256. Guide's table is exactly correct. | OK |
| 29.5.1 Custom-opcode flag letters `B,S,T,I,F`*n* | asm.c:3215-3225 — exact match (B=Br, S=St, T=TEXT rule, I=VARIAB rule, F<n>=flags2_set). | OK |
| 29.6.1 "Byte assembly warns if value ≥ 256" | asm.c `@->` handler issues warning on out-of-range. | OK (minor: actual wording is different but semantics match) |
| 29.6.2 "word assembly may contain backpatchable references because the marker is recorded on the high byte" | Matches compiler backpatch machinery. | OK |
| 29.8.5 EXT table — all 28 entries | Every `Code`/`Ver`/`Flags`/`F2` column was cross-checked against `asm.c`. All match. Specifically `save_undo`/`restore_undo` flags2=4; `make_menu` flags2=8; `read_mouse`/`mouse_window` flags2=5; `draw_picture`/`picture_data`/`erase_picture`/`picture_table` flags2=3 — all correct. | OK |
| 29.8.1–29.8.4 2OP/1OP/0OP/VAR tables | All 76 entries matched. The note that `$0F` 1OP is `not` in v3-4 vs `call_1n` v5+ matches `asm.c`. | OK |
| 29.3.7 `save`/`restore` move to EXT from v5 | Matches `asm.c`. | OK |
| 29.2.3 "Flags 2 bit numbering" vs Z-Spec | Guide does not claim Z-Spec names for bits, only which opcode sets which bit. Self-consistent with asm.c. | UNVERIFIABLE-from-repo-source (for Z-Spec bit meanings) |
| 29.2.1 "Opcodes categorised by operand count: 2OP, 1OP, 0OP, VAR, EXT" | Five-category model used throughout asm.c. | OK |
| 29.1 instruction-encoding "short form `10` top bits, long form top bit `0`" | Standard ZMachine encoding. Compiler emits bytes consistent with this. | UNVERIFIABLE-from-repo-source (encoding-level) |
| 29.6.1 "Values must be known constants (no backpatchable references are allowed)" for `@->` | Matches asm.c byte-assembly path; confirmed. | OK |

**No opcode found in ch29 is missing from asm.c. No opcode in asm.c Z-tables is missing from ch29.** Coverage: clean.

---

## ch30 — The Glulx Architecture

Source crosschecks: `files.c` construct_storyfile_g (lines 636–752),
`Glulx-Spec.md` §§1–5, `header.h` Glulx constants, `asm.c` Glulx tables.

| # | Claim (§) | Source | Classification |
|---|---|---|---|
| 30.1.2 | Default Glulx version when no features used is 3.0.0 (implied by table) | `files.c:642` starts at `0x00020000` (2.0.0). Guide's table only lists feature-triggered minimums and does not mention default 2.0.0. | MINOR (missing baseline version) |
| 30.1.2 | `uses_unicode_features → 3.0.0` trigger: "streamunichar opcode" | `files.c:645`: `if (no_unicode_chars != 0 || uses_unicode_features)`. So the flag is also tripped by any Unicode *character* in source, not only streamunichar. | MINOR (trigger description incomplete) |
| 30.1.2 | Feature → version mapping for memheap/acceleration/float/extundo/double | `files.c:648–658` exactly matches guide table. | OK |
| 30.1.2 | Version encoding `(X<<16) | (Y<<8) | Z` | Matches `files.c:681-684`. | OK |
| 30.1.3 | "Word size…Byte order Big-endian Big-endian" comparison table | Matches Glulx-Spec (§2 Memory) and Z-Machine convention. | OK |
| 30.2 | "ROM is always at least 256 bytes long" | Glulx-Spec:112 says RAMSTART/EXTSTART/ENDMEM are multiples of 256, and (§5.4 IFhd) "RAMSTART is at least 256". | OK |
| 30.2.2 | GPAGESIZE = 256, stack size multiple of 256 | Glulx-Spec line 112. | OK |
| 30.4 | Header field table | Byte-for-byte matches `files.c:676-721`. Magic "Glul", version, RAMSTART, EXTSTART, ENDMEM, Stack size, Start function, Decoding table, Checksum. | OK |
| 30.4 | "EXTSTART … equals the game file size" | `files.c:691` writes `Out_Size` for EXTSTART (Out_Size == game file size after all output). | OK |
| 30.4 | Checksum: 32-bit, zero during computation | `files.c:717-721` writes zero placeholder; `files.c:1720` initializes `checksum_long=0`. | OK |
| 30.4.1 | Static ROM block table: `0x28` = "Two zero bytes"; `0x2A` = "Two more zero bytes (padding)" | `files.c:727-728` writes **`'I','n','f','o',  0,1,0,0`** to bytes 0x24-0x2B. So bytes 0x28-0x2B are `00 01 00 00`, not four zeros. The comment in files.c calls this the "eight-byte memory layout identifier" (4 chars + 4-byte version 0.1.0.0). Guide labels byte 0x29 as zero when it is actually 0x01. | **WRONG** |
| 30.4.1 | `0x2C` Inform compiler version "as ASCII digits (e.g., `0636` for 6.36)" | `files.c:733-736` writes characters `'0'+digit, '.', '0'+digit, '0'+digit` → "6.44" style (digit-dot-digit-digit), **not** "0636". For RELEASE 636 it writes "6.36", not "0636". | **WRONG** (format is d.dd, not dddd) |
| 30.4.1 | `0x30` Glulx back-end version "(same format as compiler version)" | Same d.dd ASCII format as inform version (see files.c:737-741). | OK (consistent with same wrong format description) |
| 30.4.1 | `0x34` release (2 bytes), `0x36` serial (6 bytes) | `files.c:743-751` — 2 bytes big-endian release, 6 bytes serial. ✓ | OK |
| 30.4.1 | Total block size 24 bytes (`GLULX_STATIC_ROM_SIZE`) | 0x24→0x3B = 24 bytes. ✓ | OK |
| 30.4.2 | "Version 3.x interpreters also accept game files compiled for version 2.0.0" | Glulx-Spec §1.3 (version compatibility rules). | OK |
| 30.5.1 | Object record fields (attributes, chain, name, propaddr, parent, sibling, child) at offsets using `GOBJFIELD_*` | `header.h` defines match guide's macro expansions. | OK |
| 30.5.2 | `MAX_NUM_ATTR_BYTES = 39` → 312 attributes max | `header.h` `MAX_NUM_ATTR_BYTES 39`. 39*8 = 312. ✓ Default NUM_ATTR_BYTES=7 (56 attributes). ✓ | OK |
| 30.5.3 | `INDIV_PROP_START` is 256 in Glulx by default | `header.h` confirms. | OK |
| 30.5.3 | `propg` structure shown with fields num, continuation, flags, datastart, datalen | `header.h` propg struct matches. | OK |
| 30.6.2 | Call frame layout (FrameLen, LocalsPos, LocalType/LocalCount pairs, locals, then pushed values) | Glulx-Spec §1.3.2 matches exactly. | OK |
| 30.6.3 | Call stub 4-field layout; DestType values 0–3 and 10–14 | Matches Glulx-Spec lines 173-198. | OK |
| 30.6.4 | Stack-manipulation opcodes `stkcount` 0x50, `stkpeek` 0x51, `stkswap` 0x52, `stkroll` 0x53, `stkcopy` 0x54 | `asm.c` opcodes_table_g matches. | OK |
| 30.7.1 | Opcode encoding 1/2/4-byte with `00xxxxxx`/`10xxxxxx`/`11xxxxxx` top bits | Glulx-Spec §1.3.1 and `asm.c` assembleg_instruction match. | OK |
| 30.7.2 | "Opcode categories…over 130 opcodes" | `opcodes_table_g` has 141 entries (0x00 nop through 0x239 jdisinf). Category counts in guide's table are approximately correct but subranges like "Miscellaneous 0x0100–0x0111 = 7" include only 7 (0x100,0x101,0x102,0x103,0x104,0x110,0x111 = 7). ✓ "Function calls 0x30–0x34, 0x160–0x163 = 9" (5 + 4) = 9. ✓ | OK |
| 30.7.2 | "Game state 0x0120–0x0129 = 10" | Verified: 0x120,121,122,123,124,125,126,127,128,129 = 10. ✓ | OK |
| 30.7.2 | "Memory/heap 0x170–0x179 = 4" | mzero(0x170), mcopy(0x171), malloc(0x178), mfree(0x179) = 4. ✓ | OK |
| 30.7.2 | "Floating point 0x190–0x1C9 = 26" | Verified in asm.c Glulx table: numtof, ftonumz, ftonumn (3) + ceil, floor (2) + fadd,fsub,fmul,fdiv,fmod (5) + sqrt,exp,log,pow (4) + sin,cos,tan,asin,acos,atan,atan2 (7) + jfeq,jfne,jflt,jfle,jfgt,jfge (6) + jisnan,jisinf (2) = 29. Not 26. | **WRONG** (actually 29 single-precision opcodes) |
| 30.7.2 | "Double precision 0x200–0x239 = 32" | Verified: numtod,dtonumz,dtonumn,ftod,dtof (5) + dceil,dfloor (2) + dadd,dsub,dmul,ddiv,dmodr,dmodq (6) + dsqrt,dexp,dlog,dpow,dsin,dcos,dtan,dasin,dacos,datan,datan2 (11) + jdeq,jdne,jdlt,jdle,jdgt,jdge (6) + jdisnan,jdisinf (2) = 32. ✓ | OK |
| 30.7.3 | `GOP_*` flag values 1,2,4,8,16,32 | `header.h` constants match. | OK |
| 30.8.1 | I/O modes: Null=0, Filter=1, Glk=2, FyreVM=0x20 | Glulx-Spec §1.4.2 matches. | OK |
| 30.8.5 | String-decoding-table header: length(4) + count(4) + root(4) + node data | Glulx-Spec §1.6.1 matches. | OK |
| 30.8.5 | Node types: 0=branch, 1=terminator, 2=single Latin-1, 3=C string, 4=single Unicode, 5=Unicode string, 8=indirect ref, 9=double indirect | Glulx-Spec §1.6.2 matches. | OK |
| 30.9.1 | IEEE-754 single-precision layout + reference values (0, 1.0=3F800000, etc.) | Glulx-Spec §1.8 confirms big-endian IEEE-754 32-bit. | OK |
| 30.9.3 | "Twenty-six opcodes provide single-precision floating-point operations" | Actually 29 (see 30.7.2 above). The lists enumerated in prose (`numtof`, `ftonumz`, `ftonumn`, `ceil`, `floor`, `fadd`, `fsub`, `fmul`, `fdiv`, `fmod`, `sqrt`, `exp`, `log`, `pow`, `sin`, `cos`, `tan`, `asin`, `acos`, `atan`, `atan2`, `jfeq`, `jfne`, `jflt`, `jfle`, `jfgt`, `jfge`, `jisnan`, `jisinf`) actually enumerates 29 opcodes — but §30.9.3's opening sentence says "Twenty-six". | **WRONG** (count) |
| 30.9.4 | "Thirty-two double-precision opcodes" | Confirmed 32 (§31.3.15 also lists 32). | OK |
| 30.10.2 | Acceleration parameter numbers 0–8 with names `classes_table`, `indiv_prop_start`, etc. | Glulx-Spec §1.11 (lines 1924-1932) matches exactly. | OK |
| 30.10.3 | Standard accel functions 1, 8, 9, 10, 11, 12, 13 | Matches Glulx-Spec. Guide notes 2–7 are deprecated, which is correct. | OK (table omits the 2–7 names but §30.10.3 text says they are deprecated; intentional brevity) |
| 30.10.4 | "Accelerated functions cannot make VM function calls, cannot use streaming output, errors are fatal, state is not saved" | Glulx-Spec lines 1890-1894 match. | OK |

---

## ch31 — Glulx Instruction Set (Assembly)

Source: `asm.c` `opcodes_table_g[]` (141 entries), `opmacros_table_g[]`,
`header.h` opcode constants, `Glulx-Spec.md` §7.

**Opcode inventory check: every opcode listed in ch31 exists in
asm.c, and every opcode in asm.c Glulx tables appears in ch31.**
No WRONG opcode codes, no missing opcodes.

| # | Claim (§) | Source | Classification |
|---|---|---|---|
| 31.1.1 | Opcode encoding 1/2/4 bytes with top-bit pattern | Glulx-Spec and asm.c match. | OK |
| 31.2.1 | 16 addressing modes table | Exactly matches Glulx-Spec lines 308-323. Modes 4 and 12 correctly noted as "Unused". | OK |
| 31.2.2 | Local variable encoding: "variable n has frame offset (n-1)×4"; "If the offset fits in one byte (variables 1 through 64), mode 9 is used; otherwise mode 10." | (64-1)×4 = 252 which fits in one byte; variable 65 → (65-1)×4 = 256, does not fit. Matches asm.c (assembleg_operand mode9 threshold). | OK |
| 31.2.3 | Flag values `St=1, Br=2, Rf=4, St2=8` | `header.h` matches exactly. | OK |
| 31.3.1 | All 13 integer-arithmetic opcodes 0x10–0x1E | Matched against asm.c. | OK |
| 31.3.2 | "jumpabs has only Rf" | asm.c: `{ "jumpabs", 0x0104, Rf, 0, 1 }`. ✓ | OK |
| 31.3.2 | "Offset 0: Return 0; Offset 1: Return 1" | Glulx-Spec §2.2 confirms. | OK |
| 31.3.3 | `call` 0x30 L1 L2 S1, `tailcall` 0x34 L1 L2 (has Rf) | asm.c: tailcall has `Rf` flag. ✓ | OK |
| 31.3.3 | `callf` 0x160, `callfi` 0x161, `callfii` 0x162, `callfiii` 0x163 | asm.c matches. | OK |
| 31.3.4 | `copy` 0x40, `copys` 0x41, `copyb` 0x42, `sexs` 0x44, `sexb` 0x45 | asm.c matches (note 0x43 is not used, consistent with guide). | OK |
| 31.3.5 | `aload`..`astorebit` 0x48–0x4F | asm.c matches. | OK |
| 31.3.7 | `setiosys` 2 operands, for filter mode (1), L2 is filter function address | Glulx-Spec §2.11 confirms. | OK |
| 31.3.7 | `getiosys` "has both St and St2 flags" | asm.c: `{ "getiosys", 0x0149, St|St2, 0, 2 }`. ✓ | OK |
| 31.3.8 | gestalt selectors 0–13 | Glulx-Spec §5 matches. | OK |
| 31.3.9 | Game state opcodes 0x120–0x129 including `hasundo` 0x128 and `discardundo` 0x129 with 3.1.3+ gate | asm.c `hasundo` and `discardundo` gated with `GOP_ExtUndo`. ✓ | OK |
| 31.3.10 | "glk_gc (value 68)" | `header.h:1213` `#define glk_gc 68`. ✓ | OK |
| 31.3.11 | "linearsearch_gc = 73, binarysearch_gc = 74, linkedsearch_gc = 75" | `header.h:1218-1220` — all exactly correct. | OK |
| 31.3.12 | `mzero` 2 operands "Zero L1 bytes of memory starting at address L2" | Glulx-Spec §2.13.3 line 1793 matches exactly. | OK |
| 31.3.12 | `mcopy` 3 operands "L1 bytes from L2 to L3" (handles overlap) | Glulx-Spec line 1798-1806 matches. | OK |
| 31.3.13 | `accelfunc` 0x180 L1 L2; `accelparam` 0x181 L1 L2 | asm.c and Glulx-Spec §9 match. | OK |
| 31.3.14 | Single-precision opcode codes 0x190, 0x191, 0x192, 0x198, 0x199, 0x1A0-0x1A4, 0x1A8-0x1AB, 0x1B0-0x1B6, 0x1C0-0x1C5, 0x1C8, 0x1C9 | Each cross-checked against asm.c: all match. `fmod` has `St|St2` flags (asm.c). ✓ | OK |
| 31.3.15 | Double-precision opcodes 0x200-0x239 (all 32) | Cross-checked against asm.c. All match. | OK |
| 31.3.16 | Synthetic opcodes `pull`, `push`, `dload`, `dstore` | `header.h:1237-1240` defines exactly these four `*_gm` macros (pull_gm=0, push_gm=1, dload_gm=2, dstore_gm=3). ✓ | OK |
| 31.4.1 | Function type bytes 0xC0 (stack-arg) and 0xC1 (local-arg) | Matches Glulx-Spec and asm.c. | OK |
| 31.4.2 | "LocalType = 4 (always, for 32-bit locals)… LocalCount = 1–255" | Matches Glulx-Spec §1.3.3; compiler uses 4-byte locals only. | OK |
| 31.5.1 | Array access formulas | Matches opcode semantics. | OK |
| 31.5.2 | "Both opcodes [mzero, mcopy] can only write to RAM (addresses ≥ RAMSTART)" | Glulx-Spec §2.13.3 does not explicitly restrict to RAM; Glulx-Spec §2.12 `astore`/`astoreb` also write into memory addresses, and writing to ROM is a general VM error. The mzero/mcopy restriction is not exclusive to these opcodes. | MINOR (phrasing implies these opcodes have a special restriction; the general "ROM is read-only" rule applies to every write) |
| 31.6.1 | Search opcode parameter conventions | Matches Glulx-Spec §2.9. | OK |
| 31.6.2 | Option-flag values KeyIndirect=0x01, ZeroKeyTerminates=0x02, ReturnIndex=0x04 | Glulx-Spec §2.9 matches. | OK |
| 31.6.3 | linearsearch 8 operands | asm.c: `{ "linearsearch", 0x150, St, 0, 8 }`. ✓ | OK |
| 31.6.4 | binarysearch: "ZeroKeyTerminates flag is not supported" | Glulx-Spec §2.9.2 confirms. | OK |
| 31.6.5 | linkedsearch: 7 operands; "ReturnIndex flag is not supported" | asm.c: no=7. Glulx-Spec §2.9.3 confirms ReturnIndex not allowed. | OK |
| 31.7 | @glk 3 operands; args pushed in reverse order | Glulx-Spec §2.11 (Glk dispatch) confirms. | OK |
| 31.8.4 | Custom opcode syntax: `@"S:NNNN"` where `S` is "the operand count category" and `NNNN` is "the opcode number in hexadecimal". Example `@"1:130" selector argcount result`. | `asm.c:3522-3578` comment documents the format explicitly: `@"FlagsCount:Code"` where Flags are optional letters `S, SS, B, R`, Count is the number of arguments (0–9), and **Code is a decimal integer**. Example given in asm.c comment: `@"S3:123"` for a 3-arg opcode numbered 123 in decimal. The guide's description is wrong on three points: (a) the first part is not a "category" but a flags+argcount pair, (b) the digit after flags is the *argument count*, not a category, and (c) the code is decimal, not hexadecimal. The worked example `@"1:130" selector argcount result` is additionally inconsistent: three operands but arg count "1". | **WRONG** |
| 31.8.4 | "If the compiler does not recognize an opcode name…" | asm.c does support unknown opcodes via the @"…" custom form. General claim OK. | OK (context correct; specific format wrong) |
| 31.8.5 | Differences table between Z-machine and Glulx assembly | Most rows accurate. Row "Branch operand: Written as `?label` (no negation)" — Glulx assembler does not support `?~` inversion. asm.c assembleg path confirms. | OK |

---

## Coverage gaps (asm.c opcodes missing from guide)

**Z-machine (asm.c `opcodes_table_z` + `extension_table_z`):**
Every opcode matched. No gaps.

**Glulx (asm.c `opcodes_table_g`, 141 entries, and
`opmacros_table_g`):**
Every opcode matched. No gaps.

The chapters cover the full opcode set implemented by the compiler.

## Additional synthetic opcode note

`opmacros_table_g` defines four macros (pull, push, dload, dstore).
All four are documented in §31.3.16. No gaps.

---

## Cross-cutting observations

1. **Flags 2 bit semantics (ch28 vs ch29):** Ch28 §28.3.1 lists only
   three opcodes for bit 3 (pictures) but ch29 §29.8.5 correctly lists
   four (adding `picture_table`). asm.c confirms all four opcodes have
   flags2_set=3. **Ch28 is inconsistent with ch29 and with asm.c.**

2. **Version 7/8 "internal names" (ch28):** Guide labels v7 "Extended
   Alternate" and v8 "Extended". The compiler source only refers to
   them as "expanded Advanced" (inform.c:1289-1290). These labels
   appear to be guide-invented.

3. **Static ROM block bytes 0x28–0x2B (ch30):** Described as four
   zero-bytes but the compiler actually writes `00 01 00 00`
   (a "memory-layout version" marker, per files.c:725 comment
   "eight-byte memory layout identifier"). The guide obscures what
   is actually a meaningful structural field.

4. **Inform/Glulx version string format (ch30):** Described as
   "ASCII digits (e.g., `0636` for 6.36)"; actual format is
   `d.dd` ("6.36"). This applies to both the Inform version at 0x2C
   and the Glulx back-end version at 0x30.

5. **Single-precision float opcode count (ch30 vs ch31):**
   §30.7.2 says "Floating point 0x190–0x1C9 | 26" and §30.9.3 says
   "Twenty-six opcodes". The actual count is 29 (see §31.3.14 which
   enumerates 29 opcodes correctly). Numbers need to be updated.

6. **Custom-opcode syntax (ch29 vs ch31):** The Z-machine section
   (§29.5) correctly describes the syntax per asm.c:3160-3227. The
   Glulx section (§31.8.4) is **wrong** on almost every point of the
   syntax: categories vs argcounts, hex vs decimal, and the example is
   internally inconsistent. asm.c:3522-3578 is the authoritative
   description.

7. **`uses_unicode_features` trigger:** Guide §30.1.2 attributes it
   solely to the `streamunichar` opcode; files.c:645 also tracks
   `no_unicode_chars != 0` (any Unicode character in source).

8. **Guide's stated default Glulx version (ch30):** Not stated.
   files.c:642 sets `final_glulx_version = 0x00020000` as the
   starting baseline (2.0.0).

---

## Verification scope note

Z-machine chapters were cross-checked against compiler source only,
since the Z-Machine Standards document is not in the repo. Header
byte-field meanings beyond what the compiler writes (Flags 1 bit
semantics, interpreter-number/version split, foreground vs background
colour default, bit names for Flags 2 beyond opcode-trigger mapping)
are flagged UNVERIFIABLE. The Glulx chapters were cross-checked
against both the compiler and `Glulx-Spec.md` §§1–11.

End of Pass 5.
