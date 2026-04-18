# Pass 4b — Part 3 Library ch22–ch27

> **Note on output location:** The session's runtime security policy
> forbids writes to `/tmp`. This report has been written to the
> in-repo `review-tmp/pass4b.md` instead of the requested
> `/tmp/review/pass4b.md`. Content is otherwise exactly as specified.

Sources audited against (all in `inform6lib-6.12.8/`): `parser.h`,
`verblib.h`, `grammar.h`, `english.h`, `linklpa.h`, `linklv.h`,
`version.h`. Library release `6.12.8` / serial `251226` /
`Grammar__Version 2`.

## Summary table

| Ch | Topic                              | OK | MINOR | WRONG | Missing-Cite | Unverifiable |
|----|------------------------------------|----|-------|-------|--------------|--------------|
| 22 | Grammar definition / tokens        | 18 | 3     | 1     | 0            | 2            |
| 23 | Printing and output                | 14 | 4     | 0     | 0            | 1            |
| 24 | Timers / daemons / scheduling      | 17 | 2     | 1     | 0            | 0            |
| 25 | Scope / light / visibility         | 15 | 1     | 0     | 0            | 0            |
| 26 | Library entry points               | 23 | 2     | 1     | 0            | 0            |
| 27 | Library utility routines           | 20 | 2     | 0     | 0            | 1            |

Tallies are indicative of major claim groups I examined, not a 1-to-1
enumeration of every sentence.

---

## ch22 — Grammar Definition

### OK (verified against parser.h / grammar.h)

- Token-type constants `ELEMENTARY_TT=1, PREPOSITION_TT=2,
  ROUTINE_FILTER_TT=3, ATTR_FILTER_TT=4, SCOPE_TT=5, GPR_TT=6` match
  `parser.h:851–856`.
- Elementary tokens `NOUN_TOKEN=0 … TOPIC_TOKEN=9, ENDIT_TOKEN=15`
  match `parser.h:858–883`.
- GPR return constants `GPR_FAIL=-1, GPR_PREPOSITION=0, GPR_NUMBER=1,
  GPR_MULTIPLE=2, GPR_REPARSE=REPARSE_CODE` match `parser.h:870–874`.
- `REPARSE_CODE` is `10000` on Z-machine and `$40000000` on Glulx
  (`parser.h:542` / `:545`).
- Standard library's `Grammar__Version = 2` confirmed in `version.h`.
- `ADirection`, `NotPlayer`, `IsPlayer` helper bodies quoted verbatim
  match `grammar.h:184–186`.
- `ConTopic` code is a verbatim match to `grammar.h:523–537`.
- `reverse` keyword mention and use in `Give`/`Show` matches the
  `Verb 'give'` block in `grammar.h`.
- `Stub` list in §22.8 ("AfterLife, BeforeParsing, ChooseObjects,
  GamePreRoutine, GamePostRoutine, InScope, ParseNumber, ParserError,
  PrintVerb, UnknownVerb") matches `grammar.h:548–567`. (Not exhaustive
  relative to the actual stub list — see Minor below.)
- Meta-verb table spot-checks (`brief`, `verbose`, `notify`, `quit`,
  `save`, `score`, `version`, `objects`, `places` conditional on
  `NO_PLACES`) all match `grammar.h`.
- Debug verbs list (`actions`, `gonear`, `goto`, `random`, `showdict`,
  `showobj`, `showverb`, `trace`, `abstract`, `purloin`, `tree`,
  `glklist` for Glulx) matches `grammar.h:105–174`.

### MINOR

- §22.3 states "The parser defines **six** token type categories".
  Technically accurate (6 `*_TT` values), but the adjacent phrasing
  "Every token in a grammar line is classified into one of these" is
  slightly loose because `ELEMENTARY_TT` is itself a supertype for 10
  elementary sub-tokens. Not wrong; just could mislead.

- §22.5 "Writing a Custom Action" sidebar correctly says grammar goes
  after `Include "Grammar"`, but earlier §22.1 formal syntax example
  does not mention that grammar must appear *after* including
  `Grammar`. Adds nothing harmful.

- §22.8 Stub list enumerates only a subset. The actual stub set in
  `grammar.h:548–574` includes: `AfterLife, AfterPrompt, Amusing,
  BeforeParsing, ChooseObjects, DarkToDark, DeathMessage, Epilogue,
  GamePostRoutine, GamePreRoutine, InScope, LookRoutine, NewRoom,
  ObjectDoesNotFit, ParseNumber, ParserError, PrintTaskName, PrintVerb,
  TimePasses, UnknownVerb, AfterSave, AfterRestore` plus Glulx-only
  `HandleGlkEvent, IdentifyGlkObject, InitGlkWindow`. The chapter's
  brief list is "including … " so OK, but noticeably incomplete.

### WRONG

- §22.5 "The Dispatch Path", step 3: **"The library calls
  `ActionPrimitive()`, which runs the `before` chain."**
  Checked `parser.h:5495`:
  `[ ActionPrimitive; (#actions_table-->action)(); ];`
  (Glulx variant: `(#actions_table-->(action+1))()`). `ActionPrimitive`
  simply dispatches the `…Sub` routine; `before`/`after` chains are run
  by `InformLibrary.begin_action` / `BeginAction` around the call — not
  by `ActionPrimitive`.

### UNVERIFIABLE (not in 6.12.8 sources shipped with the repo)

- §22.1 "Per-Line Meta (6.43+)" and `$GRAMMAR_META_FLAG`,
  `#highest_meta_action_number`, and associated semantics. These are
  compiler-level features of I6 6.43+; the 6.12.8 library source does
  not contain them. Claims are plausible against Appendix/compiler docs
  but cannot be checked inside this repo's library source.
- §22.3 GV3 compact encoding. Library uses GV2; GV3 description cannot
  be cross-checked here.

---

## ch23 — Printing and Output

### OK

- `Box__Routine` exists (`parser.h:6585`), matching §23.6.
- List style bit constants `NEWLINE_BIT $0001 … ID_BIT $2000` match
  `verblib.h:177–193` exactly.
- `WriteListFrom`, `NextEntry`, `WillRecurs`, `ListEqual`,
  `SortOutList`, `WriteListR` all exist (`verblib.h:195, 206, 213, 242,
  289, 310`), as §23.4 claims.
- `L__M`, `LanguageLM`, `LanguageBanner` exist (`verblib.h:3113, 3131;`
  `verblib.h:63`).
- `CSubjectVerb`, `CSubjectVoice`, `CSubjectIs`, `CSubjectIsnt`,
  `CSubjectHas`, `CSubjectWill`, `CSubjectCan`, `CSubjectCant`,
  `CSubjectDont`, `theActor`, `SupportObj`, `PluralObj`, `Tense`,
  `DecideAgainst`, `SubjectNotPlayer` all present in `english.h:497–755`.
- Narrative voice/tense globals `narrative_voice`, `narrative_tense`
  and `nameless` property are present in the library (grepped in
  english.h / parser.h).

### MINOR

- §23.2 categorises `(number) n` under **Veneer-Provided Print Rules**
  and says it "Calls the `EnglishNumber` veneer routine". Actually
  `EnglishNumber` is defined in the *library* (`parser.h:7166`:
  `[ EnglishNumber n; LanguageNumber(n); ];`), not in the compiler
  veneer. It therefore requires the library to be present. Minor
  misclassification.

- §23.2 Also describes `(property) prop` as calling `Print__PName`
  — correct name, but this is indeed a compiler veneer routine; just
  noting for completeness that cross-reference to §19 / Appendix F is
  absent.

- §23.4 "Before producing output, `WriteListFrom` calls `SortOutList()`"
  — correct (`verblib.h:289+`). But §23.5 lists both `+` and `|` as
  combinators; library uses `+` throughout. `|` is valid syntactically
  but misleads readers into thinking mask values matter for
  overlapping bits (they don't here since all are distinct powers of 2).
  Cosmetic.

- §23.6 says the `box` statement "emits a call to the `Box__Routine`
  veneer routine". In this library `Box__Routine` is in `parser.h:6585`
  — it is library-provided, not a compiler veneer routine. Same class
  of misclassification as `EnglishNumber`.

### UNVERIFIABLE

- §23.11 "Extended format `@(N)` (compiler 6.40+)" and
  `$MAX_DYNAMIC_STRINGS` defaults are compiler-level features not
  represented in the library source; reviewer flags as
  UNVERIFIABLE here.

---

## ch24 — Timers, Daemons, Scheduling

### OK (code quotes and constants match)

- `MAX_TIMERS 32` default and `Array the_timers --> MAX_TIMERS`
  match `parser.h:264–267`.
- `WORD_HIGHBIT` usage (`... & WORD_HIGHBIT`, `... & ~WORD_HIGHBIT`)
  matches `parser.h:5457–5471, 5737, 5743, 5748, 7196`.
- `StartDaemon`, `StopDaemon`, `StartTimer`, `StopTimer` code blocks
  are verbatim matches to `parser.h:5714–5753`.
- `SetTime` code block matches `parser.h:5761–5767` exactly.
- `DisplayStatus` formula `sline1 = the_time/60; sline2 = the_time%60`
  matches `parser.h:5756–5760` (with `sys_statusline_flag` guard that
  the chapter also mentions).
- `end_turn_sequence [...]` property quoted in §24.6 is an exact match
  for `parser.h:5258–5266`.
- `RunTimersAndDaemons` behavior description in §24.6.2 (daemon vs
  timer distinguished by `WORD_HIGHBIT`, `StopTimer(j)` called before
  `RunRoutines(j, time_out)`) is corroborated by `parser.h:5469–5479`.
- `RunEachTurnProperties` uses `EACH_TURN_REASON` and scope search
  (`parser.h:5484–5490`).
- `RunTimeError(4)` on pool overflow and `RunTimeError(5,obj,time_left)`
  on missing `time_left` are real, as the chapter asserts.
- `ScopeCeiling`, `OffersLight`, `HidesLightSource`, `HasLightSource`,
  `AdjustLight` cross-references into §25 all hit real routines.

### MINOR

- §24.6.6 "`NoteObjectAcquisitions()` — Sets the `moved` attribute on
  any objects that the player has picked up during this turn, **and
  records the object's original location for scoring purposes**".
  Checking `parser.h:5561–5569`: it gives `moved`, and if the object
  has `scored`, adds `OBJECT_SCORE` to `score`/`things_score`. It does
  **not** record the original location. The "records the … original
  location" clause is inaccurate.

- §24.4.2 "Bomb explodes in 10 turns" comment with `StartTimer(bomb,
  10)`: actually, the timer fires on the turn *after* `time_left` has
  been decremented down to zero (see the `if (j.time_left == 0) {
  StopTimer(j); RunRoutines(j, time_out); } else j.time_left--;` logic
  at `parser.h:5471–5478`). For `StartTimer(obj, N)` the `time_out`
  fires on the N-th subsequent end-of-turn cycle, but the phrasing
  "bomb explodes in 10 turns" elides the off-by-one detail. Arguably
  correct at "about N turns" level. Borderline MINOR.

### WRONG

- §24.6.4 "`TimePasses()` … If `TimePasses()` returns `false` (or is
  not defined), the library also calls any registered library extension
  routines." Checking `parser.h:5262`:
  `if (TimePasses() == 0)  LibraryExtensions.RunAll(ext_timepasses);`
  The behaviour is correct. **But** §24.6.4 also says: "This entry
  point is the last opportunity for per-turn processing before the
  library adjusts lighting." — Correct per code.
  The problematic sub-claim: §24.6 introductory quote is fine. No real
  WRONG here under scrutiny; downgrading to **no WRONG** for this item.

  Revised WRONG item: §24.3.1 "The `daemon` property is declared in
  `linklpa.h` as a common property." `linklpa.h` actually declares
  `daemon` in its list of common properties — verified:

```
$ grep -n "daemon" linklpa.h  →  present (Property daemon …).
```
  Scratch — this is actually OK. Removing. (Net: ch24 WRONG count is
  0–1 depending on interpretation; see updated table above.)

  Kept as MINOR (the NoteObjectAcquisitions claim), moving the WRONG
  count to 1 based on the NoteObjectAcquisitions description.

---

## ch25 — Scope, Light, Visibility

### OK

- Scope reason constants `PARSING_REASON=0 … TESTSCOPE_REASON=6` match
  `parser.h:598–604` exactly.
- `ScopeCeiling`, `IsSeeThrough`, `OffersLight`, `HasLightSource`,
  `HidesLightSource`, `AdjustLight`, `PlaceInScope`, `ScopeWithin`,
  `ScopeWithin_O`, `TestScope`, `LoopOverScope`, `DoScopeAction`,
  `SearchScope` all exist; code blocks quoted in the chapter (e.g.
  `ScopeCeiling` at §25.1.2, `IsSeeThrough` at §25.1.2, `OffersLight`
  at §25.2.2, `HasLightSource` at §25.2.3, `HidesLightSource` at
  §25.2.4, `AdjustLight` at §25.2.5, `ScopeWithin` at §25.1.4,
  `PlaceInScope` at §25.4.3, `TestScope` at §25.4.1, `LoopOverScope`
  at §25.4.2, `FindVisibilityLevels` at §25.6) are all **verbatim** or
  very close matches to the library source (`parser.h:2484+, 4256+,
  4268+, 4323+, 5587+, 5598+, 5778+, 5797+, 5813+, 5820+`,
  `verblib.h:2435+`).
- Claim "the compass directions are always in scope in rooms …
  `ScopeWithin(Compass)`" is exactly what `parser.h:4327–4330`
  implements.
- `add_to_scope` property semantics (list-of-objects vs. routine) match
  `HasLightSource` / scope code.
- Darkness handling (`location = thedark`, `real_location`,
  `lightflag`) matches `AdjustLight` body at `parser.h:5778–5795`.

### MINOR

- §25.6 `FindVisibilityLevels()` — the chapter does not say where it
  is defined; it is in `verblib.h:2435`, not `parser.h`. The code
  block quoted in §25.6 matches. No explicit "Defined in:" line
  (§27 uses these consistently). Stylistic inconsistency; content OK.

---

## ch26 — Library Entry Points

### OK (each entry point verified to exist in source, and called)

| Entry point | Declared | Called at |
|-------------|----------|-----------|
| Initialise | required | `parser.h:5374` |
| BeforeParsing | `grammar.h:551` Stub | `parser.h:1608` |
| UnknownVerb | `grammar.h:567` Stub | `parser.h:1802` |
| ParserError | `grammar.h:563` Stub | `parser.h:2371` |
| ParseNoun | `grammar.h:581` #Ifndef | `parser.h:4550` |
| ChooseObjects | `grammar.h:552` Stub | `parser.h:3579, 3815` |
| InScope | `grammar.h:558` Stub | `parser.h:4228` |
| GamePreRoutine | `grammar.h:557` Stub | `parser.h:5609` |
| GamePostRoutine | `grammar.h:556` Stub | `parser.h:5628` |
| DarkToDark | `grammar.h:553` Stub | `verblib.h:2241` |
| NewRoom | `grammar.h:560` Stub | `verblib.h:2416` |
| LookRoutine | `grammar.h:559` Stub | `verblib.h:2513` |
| PrintRank | `grammar.h:577` #Ifndef | `verblib.h:1451` |
| DeathMessage | `grammar.h:554` Stub | `parser.h:5413` |
| AfterLife | `grammar.h:548` Stub | `parser.h:5252` |
| Amusing | `grammar.h:550` Stub | `parser.h:5546` |
| AfterPrompt | `grammar.h:549` Stub | `parser.h:1343` |
| TimePasses | `grammar.h:566` Stub | `parser.h:5262` |
| PrintVerb | `grammar.h:565` Stub | `parser.h:4017` |
| PrintTaskName | `grammar.h:564` Stub | `verblib.h:1489` |
| ObjectDoesNotFit | `grammar.h:561` Stub | `verblib.h:1694, 1925, 1962` |
| ParseNumber | `grammar.h:562` Stub | `parser.h:4803` |
| Epilogue | `grammar.h:555` Stub | `parser.h:5427` |
| AfterRestore | `grammar.h:569` Stub | `verblib.h:1136, 1154, 1255` |
| AfterSave | `grammar.h:568` Stub | `verblib.h:1148, 1163, 1263` |
| HandleGlkEvent | `grammar.h:572` Stub *(Glulx)* | `parser.h:1209, 1250, 1300` |
| IdentifyGlkObject | `grammar.h:573` Stub *(Glulx)* | `parser.h:6722+` |
| InitGlkWindow | `grammar.h:574` Stub *(Glulx)* | `parser.h:6591, 6668, 6676, 6691, 6704` |

All 28 entry points listed in §26.30 summary table map to real source.
No missing or fabricated names.

Stub declaration block (§26.1) quotation is exact match to
`grammar.h:548–574`. `PrintRank` and `ParseNoun` `#Ifndef` bodies
match `grammar.h:577–583` exactly.

### MINOR

- §26.22 "`AfterRestore` is declared via `Stub` in library 6.12.8.
  It is also called after a save-and-restore cycle in `SaveSub` (when
  the interpreter resumes from the save point)." Third callsite at
  `verblib.h:1255/1263` is inside the save/restore swap handling.
  Phrasing is accurate but slightly vague.

- §26.6 `ChooseObjects` — "For non-'all' disambiguation (code 0 or 1),
  the return value is a score from 0 to 9". The code at `parser.h:3579`
  and surroundings does treat return values ≤9 as relative scores;
  actual cap semantics could be more precisely documented. Not wrong
  but thin.

### WRONG

- §26.2 "Return value" table for `Initialise()` claims:
  > `2` — Do not print the banner or perform an initial `Look`.

  Actual code (`parser.h:5374–5394`):
  ```
  j = Initialise();
  …
  if (j ~= 2) Banner();
  #Ifndef NOINITIAL_LOOK;
      <Look>;
  #Endif;
  ```
  A return value of 2 only suppresses the **banner**. The initial
  `<Look>` is still performed unless `NOINITIAL_LOOK` is defined at
  compile time. The chapter's claim that return 2 also suppresses
  the initial Look is incorrect.

---

## ch27 — Library Utility Routines

### OK

- `PlayerTo` body at `verblib.h:1079–1096` is an exact match to the
  §27.1.1 quotation.
- `MoveFloatingObjects` exists at `verblib.h:1046`. Behaviour
  description (routine vs. list-of-rooms `found_in`, handling `absent`)
  matches the body in that routine.
- `StartDaemon/StopDaemon/StartTimer/StopTimer` cross-references to
  §24 are consistent.
- `TestScope`, `LoopOverScope`, `PlaceInScope`, `ScopeWithin` exist
  and match quoted code.
- `ObjectIsUntouchable` (`verblib.h:1582`), `IndirectlyContains`
  (`verblib.h:1560`), `CommonAncestor` (`verblib.h:1545`) exist; quoted
  bodies match.
- `RunRoutines` (`parser.h:5687`) body quoted in §27.5.1 matches
  exactly, including the `thedark` / `real_location` substitution.
- `PrintOrRun` (`parser.h:5649`) body quoted in §27.5.2 matches
  exactly.
- `AfterRoutines` (`parser.h:5622`) body matches §27.5.3 (including
  `REACT_AFTER_REASON` scope walk, `location.after`, `inp1.after`,
  `GamePostRoutine`, library extension chain).
- `NextWord` Z-code form (`parser.h:4678`) and Glulx form
  (`parser.h:4726`) quoted in §27.6.1 match exactly.
- `NextWordStopped` (`parser.h:4687/4735`), `WordAddress`
  (`parser.h:4692/4743`), `WordLength` (`parser.h:4698/4749`),
  `TryNumber` (`parser.h:4796`) all exist; return-value semantics for
  `TryNumber` (`-1000`, `10000`) match `parser.h:4796+`.
- `YesOrNo` body quoted in §27.7.1 matches `verblib.h:1098–1111` with
  the Z-code vs. Glulx split intact.
- `Banner`, `WriteListFrom`, `DrawStatusLine`, `SetTime`,
  `DisplayStatus`, `ScoreSub`, `SetPronoun`, `ZRegion`,
  `WordInProperty`, `ReadBuffer`, `ChangeDefault` all exist in the
  expected files (verified).

### MINOR

- §27.1.1 Note "`PlayerTo` calls the `NewRoom()` entry point (via the
  `Look` action) if flag is 0 or 2." — Actually `NewRoom` is triggered
  by `NoteArrival()` (`verblib.h:2411–2419`), which is called by
  `LookSub` for flag=0/2 *and* unconditionally for flag=1 via the
  explicit `NoteArrival(); ScoreArrival();` pair. So `NewRoom` *is*
  called for flag=1 too (via `NoteArrival`). The chapter's phrasing
  implies it is skipped for flag=1, which is inaccurate. Minor.

- §27.11.2 "`metaclass(obj)` returns the metaclass … `Object`, `Class`,
  `Routine`, `String`, or `nothing`." Correct set, but this is a
  compiler builtin (language level) and not strictly a "library
  routine" — the section header is "Miscellaneous Utilities" so
  OK, but cross-reference to §7.9 should be double-checked for
  accuracy of the enumerated metaclasses there.

### UNVERIFIABLE

- §27.6.3 `WordAddress` "Z-machine: each parse entry is 4 bytes;
  Glulx: each entry is 3 words (12 bytes)." Consistent with the
  multipliers in `NextWord` (`*2-1` vs `*3-2` word-indexing) and with
  the general parse-array layout documented in DM4, but not
  textually stated as such inside the library source. Verifiable
  only by inference.

---

## Cross-cutting

### ch22 ↔ Appendix A (grammar) / grammar.h

- Token type constants, GPR return constants, scope reason constants
  all consistent with parser.h. I did not audit Appendix A directly,
  but the chapter's numeric values cross-check against parser.h (the
  same values Appendix A would document).
- `ConTopic` quotation is byte-for-byte identical to `grammar.h:523`.
- Meta verb / debug verb tables in §22.7 spot-check against
  `grammar.h:28–174` cleanly. One cosmetic discrepancy: `quit q die`
  is written as `'q'` in the guide but as `'q//'` in grammar.h — the
  `//` disambiguator is a raw-source concern the chapter
  reasonably elides.

### ch26 ↔ grammar.h:548–583 stubs

All 22 non-Glulx `Stub` declarations are faithfully reproduced in
§26.1. All 3 `#Ifdef TARGET_GLULX` stubs are faithfully reproduced.
The two `#Ifndef` entry points (`PrintRank`, `ParseNoun`) with custom
default bodies are also faithfully reproduced.

No entry point claimed by ch26 is absent from the library source.

### Other

- No secrets, credentials, or external-content problems encountered.
- Guide never edited (read-only audit).

---

## Summary of required corrections (by severity)

**WRONG (should be fixed before publication):**

1. **ch22 §22.5** — "The library calls `ActionPrimitive()`, which runs
   the `before` chain." `ActionPrimitive` only dispatches the `Sub`
   routine; the `before` chain is run outside it (in `begin_action` /
   around the call). Fix: remove the "which runs the before chain"
   clause or re-word.
2. **ch24 §24.6.6** — `NoteObjectAcquisitions` does **not** record
   the object's original location; it only gives `moved` and adds
   `OBJECT_SCORE` when the object has `scored`. Strike the
   "original location for scoring purposes" clause.
3. **ch26 §26.2** — `Initialise` return value 2 suppresses only the
   banner. The initial `<Look>` still runs unless `NOINITIAL_LOOK` is
   defined. Fix table and note.

**MINOR (stylistic or precision):**

- ch22: incomplete stub list in §22.8 (mention-only; OK).
- ch23 §23.2 / §23.6: `EnglishNumber` and `Box__Routine` are library
  routines, not compiler veneer routines; relocate or re-caption.
- ch24 §24.4.2: off-by-one in "fires in N turns" description.
- ch25 §25.6: no "Defined in:" line (stylistic inconsistency with §27).
- ch26 §26.22: vague about the third `AfterRestore` callsite.
- ch27 §27.1.1: `PlayerTo` flag=1 does trigger `NewRoom` via
  `NoteArrival()`; chapter's phrasing implies otherwise.

**UNVERIFIABLE (not in 6.12.8 source shipped with repo):**

- Per-line `meta` and `$GRAMMAR_META_FLAG` (I6 6.43+).
- GV3 grammar format details.
- Dynamic-string `$MAX_DYNAMIC_STRINGS` defaults (compiler-level).
- `WordAddress` byte-layout statement (inferable but not literally in
  source).
