# Westwood `.INF` lift feasibility spike

> **Date:** 2026-07-06
> **Type:** early risk-retirement spike for the IR (MASTER §6, PHASES §5) —
> the lighter, first-in-line cousin of the AESOP spike; MASTER §6 explicitly
> warns the lift competency must exist from the *first* Westwood importer,
> not only for AESOP.
> **Question:** Does a real EOB `.INF` block script lift cleanly into the
> IR's declarative `event → condition → action` form with a closed action
> vocabulary — and where does it strain (control-flow reconstruction,
> side-effecting conditions, the three escape hatches)?
> **Verdict up front:** **Yes, and more cleanly than AESOP.** `.INF` is a
> small, fully-enumerable bytecode VM (~30 opcodes total, ~26 shared between
> EOB1/EOB2) whose control flow is shallow and its subroutine calls are
> **statically resolvable** (no polymorphism, no dynamic dispatch — unlike
> AESOP's `PASS`). The one place it genuinely strains a "conditions are pure"
> model is EOB2's RPN evaluator, which lets a handful of *predicates*
> reach into interpreter state (`_activeCharacter`) as a side effect of being
> evaluated, and have that state read back by a *later, separate* action.
> The three escape hatches (`sequence`, `dialogue`, `specialEvent`) are real
> and must be modelled as opaque, closed-vocabulary calls-by-id — one of them
> (`specialEvent`) is *by design* a per-level hardcoded native dispatch table
> in the original engine, not something we should try to decompose further.

All raw disassembly (derived from copyrighted `LEVEL*.INF` game data) stays in
scratch. This note quotes only a handful of short illustrative byte/pseudo
lines; the analysis is ours.

---

## 1. Toolpath used

**Winner: the already-decompiled EOB1 `level1.txt`/`level2.txt` (eob.phpwiki
archive), cross-checked against three independent sources.** Recon flagged two
possible paths — build `eoblib`'s `Script` disassembler against a loose
`LEVEL*.INF`, or use the wiki's pre-decompiled text. I tried the former first
and it does not work out-of-the-box against the real files, which is itself a
useful finding, so I fell back to the latter and cross-validated hard:

1. **`eoblib`'s `Inf.java`/`Script.java` do not parse the real loose EOB2
   `LEVEL1.INF`.** I hex-dumped
   `~/Games/Eye of the Beholder II The Legend of Darkmoon.app/Contents/Resources/game/LEVEL1.INF`
   directly (no unpacking needed, it's stored loose) and hand-decoded the
   header: `triggersOffset` (first `u16`) = `0x1266`, but the maze filename
   string `"level1.maz"` actually starts at file offset `0x10`, not offset `2`
   as `Inf.java` assumes. Cross-checking against ScummVM kyra's
   `EoBCoreEngine::initLevelData()` (`engines/kyra/engine/scene_eob.cpp`)
   confirmed why: EOB2's header has a materially different, longer shape than
   EOB1's (13-byte vs 12-byte name fields, extra `0xEC`/`0xEA` door-shape
   markers, an extra sound-file name, an EOB2-only script-timer table) before
   the script bytes begin. `eoblib`'s `Inf.java` matches **EOB1's** layout
   only (verified below). Replicating kyra's EOB2 header parser byte-exact
   was judged out of scope for a *lift-feasibility* spike — it's importer
   plumbing, not a question about the scripting model — so I used kyra's
   `script_eob.cpp` directly as EOB2 ground truth for opcode *semantics*
   instead (see next point), and the wiki's *EOB1* decompile for the concrete
   hand-lifts.
2. **Cross-check #1 — the wiki's decompiled text vs. the wiki's independently
   labelled 68k-assembler dump of the same file.** The `eob.inf` page contains
   a second, differently-produced disassembly of `LEVEL1.INF` (a 68000-asm
   `dc.b` resourcing with symbolic jump labels, by a different tool/author
   than the one that produced `level1.txt`). I decoded a ~200-byte span by
   hand from the raw byte table using only `eoblib`'s opcode enum, *before*
   reading the pre-decompiled `level1.txt`, then diffed my by-hand decode
   against `level1.txt`'s automated output — they agree exactly, byte for
   byte, address for address. This is a strong independent confirmation that
   the opcode grammar (see §2) is right and that both wiki artefacts describe
   the real game file.
3. **Cross-check #2 — ScummVM kyra `script_eob.cpp` (GPL, read only, never
   copied).** This is the actual shipping interpreter for both EOB1 and EOB2,
   with parallel `_v1`/`_v2` opcode handlers where the two games diverge. It
   gave ground truth for everything the wiki's opcode table alone couldn't
   settle: the exact 10-deep call-stack behaviour, the trigger-dispatch
   mechanism (block + event-reason bitmask), the EOB2-only side-effecting
   predicates, and the literal C++ bodies of the three escape hatches.
4. **Cross-check #3 — a real EOB2 fragment.** The same `eob.inf` wiki page
   also contains a *second* 68k-asm labelled dump, this time of EOB2's
   `LEVEL10.INF` (different tool: "EOB2 Language File ReSourcerINF"). It
   shows the identical staircase-trigger *shape* as my EOB1 hand-lift #1
   (§4a) — same opcodes, same structure — which is good evidence that the
   opcode grammar, not just the header, really is shared across both games.

## 2. What I disassembled / decoded

**Hand-lift #1 source:** EOB1 `LEVEL1.INF`, bytecode offset `0x02CF`–`0x0395`
(a "go down the stairs" block trigger — the very first non-header script in
the level, and a good representative sample: conditions, a one-time flag, a
random-encounter check, an inter-level teleport, and a direction-gated
fallback).

**Hand-lift #2 source:** EOB1 `LEVEL2.INF`, two related sites: the shared
subroutine body at `0x02FD`–`0x035C`, and one of its three call sites at
`0x084B`–`0x0878` (an item-socket puzzle — "does the dagger fit"). Two
sibling call sites at `0x092D` and `0x0B87` call the same subroutine (not
transcribed in full — confirmed same shape via the wiki's `;Call 0x02fd`
markers).

### The `.INF` model, as observed (grounded in `eoblib` + kyra + the wiki)

- **A level = one flat bytecode blob + a trigger table.** The trigger table
  is an array of `(maze-block-position, event-reason-flags, byte-offset)`
  triples (`eoblib`'s `Trigger` class matches this exactly). Only **12**
  distinct `flags` byte values are ever used on disk
  (`eoblib.Trigger.collapseTriggerFlags`) — this is a small closed enum, not
  an open bitfield in practice.
- **Dispatch = `run(block, eventReasonMask)`.** kyra's `EoBInfProcessor::run`
  looks up the block's stored bytecode offset, masks the block's own stored
  flags against the caller-supplied event-reason bitmask, and only then
  executes. Grepping every call site of `runLevelScript(block, flags)` across
  the engine gives the closed vocabulary of **event reasons**: step onto
  block, step off block, click wall (two sub-variants), timer fire,
  step-counter fire, use/drop item on block, monster step, monster death,
  spell hit, dialogue-result. This *is* the "event" half of
  `event → condition → action` — already a closed, enumerable set on the
  original's own terms.
- **A trigger's bytecode = straight-line opcodes + `Conditional` blocks +
  forward `Jump`s + `Call`/`Return`.** Opcodes are single signed bytes acting
  as an index into a fixed ~30-entry table (see §6); operands follow inline.
- **`Conditional` (`0xEE`) is a self-contained RPN block**, not a bare
  compare-and-branch: it pushes a run of value-opcodes and operators (`EQ`,
  `NEQ`, `LT`, `LTE`, `GT`, `GTE`, `AND`, `OR`), terminates on an
  `END_CONDITIONAL` marker, and is *itself* followed by a 2-byte
  "false" jump address — i.e. the condition and its else-branch are one
  syntactic unit, matching `eoblib`'s `ConditionalToken` exactly. Verified
  values pushed include: trigger-invocation reason, party facing, a maze/
  level/global flag, item counts, dice rolls, wall ids, race/class party
  membership.
- **`Jump`/`Call`/`Return`/`End`** are the only control-flow primitives.
  `Call` pushes a **fixed, statically-known** return address onto a
  **10-deep** stack (`kyra`: `new int8*[10]`) and jumps; past depth 10 the
  `Call` is silently dropped (falls through past its own operand — no crash,
  no error, just a missed call). `Return` pops the stack or, if empty, aborts
  the whole script.
- **Escape hatches:** `sequence` (numeric cutscene id), `dialogue` (a small
  sub-opcode-switch UI state machine: draw bitmap / init or restore dialogue
  mode / draw dialogue box / run an indexed Y-N-cancel dialogue writing its
  result into a `_dlgResult` register / print dialogue text), and
  `specialEvent` (a numeric sub-opcode switch that, in the one level sampled
  via kyra source, dispatches to: a lightning-bolt screen effect, a
  character-select dialogue, an XP level-up, a resurrection-select dialogue,
  a conditional NPC recruit, a targeted item deletion, and a graphics reload
  — each hand-written in C++ for that level's specific need).

---

## 3. A tiny pseudo-schema for the lift

Same throwaway-notation spirit as the AESOP note (not a proposal for the IR's
concrete syntax):

```
on <EventReason> at <Block>:
    when <condition-expr>:            # RPN block; may itself contain nested
        <action>                      #   guards via forward Jump
        …
    else: …
# actions: SetWall | ChangeWall | OpenDoor | CloseDoor | CreateMonster |
#          Teleport | StealItem | Message | SetFlag/ClearFlag | Sound |
#          Damage/Heal | ChangeLevel | GiveXP | NewItem | Launch | Turn |
#          IdentifyItems | Encounter | Wait | Call <subroutine> | Return |
#          Cutscene(id) | Dialogue(...) | SpecialEvent(id)
# conditions: comparisons over trigger-reason, party facing, flags
#             (level/global/monster), item/monster counts, dice, party
#             race/class membership, wall ids
```

Three action "kinds" fall out, mirroring AESOP's split: **domain actions**
(the ~26 shared opcodes — wall/door/monster/item/party/level mutations),
**named opaque calls** (`Cutscene(id)`, `Dialogue(...)`, `SpecialEvent(id)` —
the escape hatches), and **structural** (`Call`, `Return`, `Jump` — resolved
away at import time, see §5).

---

## 4. The two hand-lifts

### 4a. EOB1 `LEVEL1.INF` — "stairs down" (the clean-but-branchy case)

Raw shape (our re-expression, addresses from the wiki dump; both independent
wiki transcriptions and my own by-hand decode agree on these bytes):

```
02CF: Conditional[ trigger.reason == 1 ] else -> 035A
02D6: Conditional[ party.facing   == 1 ] else -> 0308
02DD: Conditional[ maze.flag(31)  == 0 ] else -> 02EA
02E5: Encounter(10)
02E7: SetFlag(Maze, 31)
02EA: Message("going down...", color=5)
02FB: Sound(0x4F)
02FF: ChangeLevel(level=2, pos=(19,23), facing=1)
0305: Jump 035A
0308: Message("you can't go that way.", color=15)
0322: Sound(0x1D)
0326: Conditional[ party.facing==0 ] -> Teleport(bump, dest A)
0333: Conditional[ party.facing==1 ] -> Teleport(bump, dest B)
0340: Conditional[ party.facing==2 ] -> Teleport(bump, dest C)
034D: Conditional[ party.facing==3 ] -> Teleport(bump, dest D)   # falls into 035A
035A: Conditional[ trigger.reason == 4 ] else -> 0395
       … (same 4-way facing/Teleport fallback shape as above) …
0395: End
```

Lifted:

```
on StepOnto(block=STAIRS_1_2) for trigger.reason == 1:
    when party.facing != EAST:
        SELF.bumpForFacing(party.facing)     # 4-way Teleport nudge table
        return
    when maze.flag(31):                       # already used these stairs
        Teleport(bumpForFacing(party.facing)) # same fallback, reason-4 path
        return
    Encounter(10)                              # random-encounter roll
    maze.setFlag(31)
    Message("going down...")
    Sound(0x4F)
    ChangeLevel(to=2, pos=(19,23), facing=EAST)

on BumpInto(block=STAIRS_1_2) for trigger.reason == 4:
    # identical guard/fallback shape, reused verbatim
    …
```

- **Event** = the trigger-table entry itself (block + one of the 12 event
  reasons); the block's bytecode further **branches on `trigger.reason`**
  inside the body (`==1` vs `==4`), i.e. *one* physical trigger multiplexes
  *two* logical events by reading back "why was I invoked" as an ordinary
  condition value. This is the first real strain point: the vocabulary needs
  the invocation reason itself queryable as a condition operand, not only as
  the dispatch key.
- **Conditions** = party facing, a level flag (one-shot gate), and (in the
  wrong-direction branch) four mutually exclusive facing comparisons that a
  structuring pass collapses into a `switch(party.facing)`.
- **Actions** = ordered, side-effecting, terminating in either a
  `ChangeLevel` (success) or a small cosmetic `Teleport` "bump" (failure).
  The forward `Jump 035A` is pure control-flow sugar — both the "wrong flag"
  path and the "wrong direction, facing=3" path converge on the same second
  trigger's body, which a structuring pass turns into a shared exit block,
  not a `goto`.

This lifts cleanly. The only real work is a **structuring pass** over forward
jumps and multi-way facing checks — bounded, reducible, no loops seen.

### 4b. EOB1 `LEVEL2.INF` — item-socket + shared subroutine (the `Call` case)

```
084B: Conditional[ pointerItem.type==5  OR  maze.flag(4)==false ] else -> 087B
0858: Sound(0x0E)
085C: Message("the dagger fits.", color=5)
0870: ConsumeItem(at=mousePointer)
0872: SetFlag(Maze, 4)
0875: Call 02FD
0878: Jump 08A1                                  # skip the "doesn't fit" branch
…
02FD: Conditional[ maze.flag(4) OR maze.flag(5) OR maze.flag(6) OR maze.flag(7) ]
       else -> 035C
0314: Conditional[ global.flag(2) == true ] else -> 035C   # already rewarded
031C: Message("special quest for this level!", color=3)
033D: Sound(0x1C)
0341: SetFlag(Global, 2)
0344: NewItem(#55, pos1) ; 034A: NewItem(#55, pos2)
0350: NewItem(#55, pos3) ; 0356: NewItem(#55, pos4)
035C: Return
```

Lifted:

```
on UseItemOnBlock(block=DAGGER_SOCKET) when heldItem.type == DAGGER:
    Sound(0x0E); Message("the dagger fits.")
    ConsumeHeldItem()
    maze.setFlag(4)
    Call CheckQuestReward                # shared across 3 sockets

action CheckQuestReward:                 # named, reusable action-sequence
    when not(maze.flag(4) or maze.flag(5) or maze.flag(6) or maze.flag(7)):
        return
    when global.flag(2):                 # already granted
        return
    Message("special quest for this level!"); Sound(0x1C)
    global.setFlag(2)
    NewItem(#55, pos1); NewItem(#55, pos2); NewItem(#55, pos3); NewItem(#55, pos4)
```

- **Event** = "use/drop held item on this maze block", with the **held item
  itself as an implicit event parameter** (its `type` is read directly in the
  guard).
- **Condition** = item-type match plus a "not already triggered" flag guard;
  the reward subroutine's own guard is an OR of four independent quest-stage
  flags plus a "not already granted" global flag — a small, pure boolean
  expression, nothing resists lifting.
- **Actions** = consume item, set flag, **`Call` into a shared, statically
  resolved subroutine** invoked from (at least) three different trigger
  sites, which itself does the actual reward (message, sound, flag, four item
  drops) and `Return`s.

The interesting structural point is the **`Call`/`Return` pair itself**.
Unlike AESOP's `PASS` (which requires walking a live inheritance graph to
resolve which parent handler runs), `.INF`'s `Call` target is a **fixed
literal byte offset baked into the bytecode at build time** — there is no
polymorphism, no runtime resolution, nothing to walk. That makes it *strictly
easier* than AESOP's escape valve: the importer can either (a) inline the
subroutine body at each of the 3 call sites (cheap here — one screen of code,
called 3 times), or (b) keep it as one **named, shared action-sequence** in
the IR and have each trigger's action list end with a `Call
CheckQuestReward`-style reference. Both are fully static, fully declarative,
and require no interpreter — this is exactly the "closed, resolvable at
import time" shape MASTER §4.1 wants. This is also where the 10-deep call
stack matters in principle (bounding worst-case inline expansion / recursion
depth) even though no real script sampled here nests more than one level
deep.

---

## 5. Feasibility verdict

**Yes — `.INF` lifts cleanly, and the lift is *easier* than AESOP's.** Every
block script sampled (directly, plus the independent EOB2 `LEVEL10.INF`
cross-check) is a shallow, terminating sequence of
`guard → ordered side-effects → (branch | return | end)`. Concretely:

- **Control flow is not a real obstacle.** `Jump`/`Call`/`Return`/`Conditional`
  only ever produce reducible `if`/`switch`/shared-exit shapes in every
  sample seen — no loops, no back-edges, no dynamic jump targets (every
  `Jump`/`Call` operand is a literal offset baked in at compile time, unlike
  AESOP's `SEND`, which resolves a method by name at the *target object's*
  class). A basic-block + forward-jump structuring pass (the same kind the
  AESOP spike already calls for) fully reconstructs the `if/switch` shape;
  `Call`/`Return` resolve to either inlining or a named shared action list,
  entirely at import time, bounded by the real 10-deep stack.
- **The action vocabulary is genuinely small and already closed on the
  original's own terms**: 30 opcodes total (26 shared EOB1/EOB2, +4 EOB2-only:
  `delay`, `drawScene`, `dialogue`, `specialEvent`), each with a fixed operand
  layout. Nearly all of them (`SetWall`, `Teleport`, `CreateMonster`,
  `SetFlag`, `Message`, `Sound`, `ChangeLevel`, `NewItem`, `GiveXP`,
  `Damage`/`Heal`, `Turn`, `Encounter`, `Wait`, …) map **1:1** onto a
  campaign-author-facing domain action. Nothing here needs a "pure
  expression built-ins" tier the way AESOP's `RTCODE.C` did — the RPN
  evaluator's value-opcodes (flag/wall/item/dice/race/class queries) are
  already a short, fixed enum, not general string/math routines.
- **The event taxonomy is already closed and small** (≈12 trigger-reason
  values on disk; a similarly small, enumerable set of *why* `runLevelScript`
  is invoked from engine call sites) — this is a gift relative to AESOP,
  where event taxonomy had to be reconstructed from `notify`/`post_event`
  call sites across 376 objects.

**The one real strain point is EOB2's side-effecting RPN predicates**
(`_activeCharacter`, and to a lesser extent `_dlgResult`) — see §6. It is a
smaller, more contained problem than AESOP's cross-object state (§6 of the
AESOP note), because it's a single hidden register, not an addressable object
graph, but it is real interpreter state threading *between* a condition and a
later, syntactically separate action, and a naively "pure" condition/action
split will silently drop it.

Estimated fit against MASTER's ≥95% target: comfortably met. The escape
hatches are rare (one `specialEvent`/`dialogue` cluster per "special" level
moment, not per ordinary trigger) and are exactly the kind of opaque,
enumerable-by-id call the vocabulary is designed to absorb, not a sign the
vocabulary is incomplete.

---

## 6. Strain points, concretely

**Control-flow reconstruction.** Not a real strain in the samples examined:
every `Jump` target and `Call` target is a literal, statically known offset
(no computed jumps anywhere in `.INF` — contrast AESOP's `SEND`, which is a
named, dynamically resolved dispatch). The importer's job is a standard
basic-block-and-dominator structuring pass: `Conditional`'s embedded
false-address becomes an `if/else`, chains of mutually exclusive
`Conditional`s over the same value (the 4-way facing check in §4a) become a
`switch`, and converging forward `Jump`s become a shared exit / early
`return`. `Call`/`Return` resolve to inlining or a named shared action list
(§4b) — never a runtime call stack. **Watch item:** confirm across a wider
sample (only 12 EOB1 levels + one EOB2 fragment examined here) that no level
uses a *computed* jump/call address (none seen; kyar's `oeob_jump`/
`oeob_callSubroutine` always read a literal `u16` operand, so this would
require the compiler to have emitted one, which the tooling that built these
`.INF` files is not documented to do) — if this recon holds across all 44
level files (12 EOB1 + 32 EOB2/EOB3 dungeons out of scope here), `Call`/`Jump`
never need more than import-time resolution.

**Side-effecting conditions (EOB2 only).** `oeob_eval_v2`'s class/race/
portrait predicates (kyra `script_eob.cpp` cases 0/14/15) pick a *specific*
party member (via a dice-rolled scan) and, as a side effect of the predicate
being true, set the interpreter's `_activeCharacter` register — which a
*later, separate* opcode (`printMessage_v2`'s character-name substitution;
`modifyCharacterHitPoints`/`calcAndInflictCharacterDamage`'s "no explicit
target" case) reads back as an implicit default target. A condition that
silently commits to *which* character is a genuinely non-declarative
operation: two independent evaluations of "does the party contain a cleric"
are not guaranteed to name the same character (the scan starts from a fresh
dice roll each time), and the identity of that character only exists as a
side channel to the next action, not as a return value visible to the IR's
condition layer. **Absorb by promoting the hidden register to a first-class
IR binding**: model these predicates as
`when party.memberWithClass(CLERIC) as subject:` — a condition that both
tests *and binds* a name, with `subject` then explicitly threaded into the
following actions (`Message(..., speaker=subject)`,
`Damage(target=subject, ...)`) instead of being an invisible VM register. This
is a strictly more disciplined version of the same value than the original
had; it removes the "spooky action at a distance" without losing any
expressiveness the original scripts used. EOB1's `oeob_eval_v1` has the
analogous class/race-membership predicates but **only returns a boolean** —
no character identity leaks out — so this strain is EOB2-specific and,
notably, is exactly the kind of thing that gets *worse*, not better, in the
AESOP-based EOB3/Dungeon Hack games (object-relative state is the AESOP
spike's #1 finding) — good evidence the IR's binding/addressing primitive
needs to be designed once, generally, and reused by both lifts.

**The three escape hatches.** All three are opaque-by-necessity, and the
right lift differs per hatch:
- **`sequence(id)`** — a closed table of numbered cutscenes (plus 3 hardcoded
  special ids for endgame/portal/password). Lift as
  `PlayCutscene(id: enum)` — a named action, content (the cutscene itself) is
  an engine/importer asset, not something the IR's condition/action language
  needs to see inside of.
- **`dialogue(...)`** — a small, self-contained UI state machine (draw
  bitmap, enter/exit dialogue mode, draw box, run an indexed Y/N/cancel
  dialogue, print text) whose *result* (`_dlgResult`) is read back by
  subsequent script code exactly like `_activeCharacter` above. Lift as a
  `RunDialogue(id) -> result` **query action** (an action that also produces
  a bindable value, same shape as the class/race predicate fix above), not a
  bare fire-and-forget call.
- **`specialEvent(id)`** — genuinely different in kind from the other two:
  the kyra source shows it is a **per-level hardcoded C++ dispatch table**
  (screen effects, character-select/resurrection dialogues, XP grants, NPC
  recruitment, targeted item deletion, asset reloads) that the original
  developers extended ad hoc, level by level, as bespoke needs arose. This is
  *not* a closed content vocabulary the way `sequence`/`dialogue` are — it is
  an acknowledgement, baked into the original engine itself, that some
  moments are one-off enough not to deserve their own opcode. **The IR must
  model this as an explicitly bespoke, per-level named hook**
  (`SpecialEvent("level10_lightning_puzzle")`) implemented by the *importer*
  as hand-written IR content (an authored sequence of ordinary domain
  actions) wherever feasible, and by an engine-side named service only where
  it truly cannot be expressed declaratively (e.g. the lightning screen
  effect is presentation, not logic, and belongs in the same "engine
  services" tier the AESOP note already proposed for AESOP's `GRAPHICS.C`).
  Treat every `specialEvent`/`dialogue` sighting as a **flag for manual
  review at import time**, not something the importer can blindly translate.

**Where someone is tempted to embed a VM, and how to avoid it.**
1. *The `trigger.reason == N` re-check inside a block's own body* (§4a) looks
   like it wants a generic "read the dispatch key" primitive. → Model
   trigger-reason as an ordinary, enumerable condition operand
   (`when trigger.reason == BUMP`), not a special case; it is already closed
   (12 values).
2. *`Call`/`Return` chains* look like they need a runtime call stack. →
   They don't: targets are static, depth is bounded (10), so **flatten at
   import** (inline, or emit a named shared action list) exactly as the
   AESOP note recommends for `PASS`/`JSR`.
3. *`_activeCharacter`/`_dlgResult`* look like they need mutable interpreter
   registers. → Promote to **explicit named bindings** produced by the
   condition/query that created them and consumed by name in later actions
   (see above) — never a hidden register.
4. *`specialEvent`* looks like "just add an opcode for it." → Resist per-id
   opcodes; it is bespoke content, not vocabulary — author it as ordinary
   IR content (a named authored action sequence) at import time instead.

---

## 7. What the IR event model MUST carry to absorb `.INF`

1. **A closed trigger/event-reason enum** bound to a grid block (or, for
   EOB2, potentially multiple concurrent bindings per block — the trigger
   table already stores this per (block, reason) pair on disk). This is
   cheaper than AESOP's requirement — no object identity needed, just
   `(block, reason)`.
2. **Ordered, side-effecting action lists with mid-body guards and early
   return/`switch`** — same requirement AESOP surfaced; `.INF`'s version is
   simpler because guards never reference other objects, only this trigger's
   own block, party state, and global/level/monster flags.
3. **A condition/query that also binds a name** (`when <predicate> as
   <ident>: … use <ident> …`), to absorb `_activeCharacter`-style
   EOB2 predicates and `dialogue`'s `_dlgResult` without a hidden register.
   This is the same primitive the AESOP spike's inter-object *query* actions
   need (a `SEND` that returns a value) — **this is a convergence point**:
   both lifts want "a condition/action that produces a bindable value
   consumed later," not a VM register. Design it once.
4. **A named, reusable action-sequence** (a "sub-action" or "procedure"
   reference, fully static, no recursion beyond a small fixed bound) to
   represent shared `Call` targets without duplicating action lists at every
   call site. Distinct from AESOP's `PASS` (dynamic, class-hierarchy-resolved)
   — `.INF`'s version is simpler (static) and could plausibly be the *same*
   IR primitive AESOP's flattened-`PASS` output also uses, since both end up
   as "this handler's actions include another named handler's actions
   inline or by reference."
5. **A small, closed, enumerated flag namespace** with three scopes (level,
   global/campaign, per-monster) as **typed boolean/counter fields**, not a
   raw bitfield — matches the AESOP note's §7.7 recommendation for
   `decflags` almost exactly; both games want the same thing.
6. **Named opaque actions in (at least) two flavours**: fire-and-forget
   (`PlayCutscene(id)`) and query-producing (`RunDialogue(id) -> result`,
   reusing item 3's binding primitive). Plus an explicit **"authored escape
   hatch"** classification (`specialEvent`-style) that importers are
   expected to hand-author as ordinary IR content per occurrence, flagged
   for human review — this is a process requirement on the importer/editor,
   not a new IR construct.
7. **Import-time control-flow structuring + call resolution** is assumed:
   the importer runs a basic-block/dominator pass to recover `if/switch`,
   and resolves every `Call` to an inline copy or a named shared
   action-sequence, bounded by the original's own 10-deep limit. The IR
   receives already-structured, already-flattened handlers.

**Closed-vocabulary risk flags.** (i) If the importer treats
`trigger.reason` as a magic dispatch key instead of an ordinary enumerable
condition value, every "one physical trigger, multiple logical events" script
(§4a) becomes unrepresentable except by re-embedding the reason-check as a
special case — keep it a plain condition operand. (ii) If `_activeCharacter`/
`_dlgResult` are lifted as "the action just knows who" instead of an explicit
binding, the importer will be pushed toward keeping a tiny runtime register
to fake it — exactly the VM-in-disguise anti-pattern MASTER §4.1 warns
against. (iii) `specialEvent` is the one place where "just enumerate it" is
wrong on principle, not just impractical — its whole nature in the original
is "we didn't want to enumerate this," and the IR should preserve that as an
explicit, reviewed, per-instance authoring step rather than pretend it's a
closed table like `sequence` is.

---

## 8. EOB1 vs EOB2 differences (what actually changes)

| Aspect | EOB1 | EOB2 |
|---|---|---|
| `.INF` header before script bytes | Short, fixed layout (matches `eoblib.Inf`; triggersOffset + 12-byte name fields at small fixed offsets) | Longer, block-structured (`block_A`/`block_B`/`block_C` per the wiki's own EOB2 section), 13-byte name fields, extra door-shape and sound-file data, EOB2-only script-timer table |
| Opcode count | 26 of the 30 (`_commandMin = -27`) | All 30 (`_commandMin = -31`); adds `delay`, `drawScene`, `dialogue`, `specialEvent` |
| RPN evaluator | `oeob_eval_v1` — pure boolean stack machine | `oeob_eval_v2` — adds side-effecting character-selecting predicates (§6) and reads `_dlgResult`/`_activeSpell` |
| `printMessage`/`movePartyOrObject`/`createItem` operand encoding | `_v1` layout | `_v2` layout (different byte counts/fields per kyra's parallel handlers) |
| Escape hatches available | `sequence` only (no `dialogue`/`specialEvent`/`drawScene`/`delay` — EOB1 has no full-screen NPC dialogue system) | All three (`sequence`, `dialogue`, `specialEvent`) plus `drawScene`/`delay` |
| Trigger/event shape | Same `(block, reason, offset)` triple; same `Conditional`/`Jump`/`Call`/`Return` grammar | Identical (confirmed via the independent `LEVEL10.INF` cross-check, §1) |

**Net assessment:** the *scripting model* (opcode grammar, control flow,
trigger dispatch) is essentially one shared VM across both games — the real
EOB1/EOB2 delta is (a) container/header format, which is importer plumbing,
not lift-model risk, and (b) a handful of operand-encoding differences plus
the four EOB2-only opcodes, all of which are additive, not structural. An
importer built against EOB1 first (per PHASES' sequencing) should expect to
extend, not redesign, when EOB2 arrives — confidence-building for MASTER §10's
"then EOB2 — cheap" sequencing claim.

---

## 9. Comparison with the AESOP spike — convergence points for ONE IR

Both spikes independently landed on the **same three IR requirements**,
which is the strongest signal that they belong in the core event/condition/
action schema rather than being lift-specific bolt-ons:

1. **A condition/query that also produces a bindable value.** AESOP needs it
   for `SEND`-as-query (cross-object property reads) and for teleport's
   `mover.report(...)`; `.INF` needs it for EOB2's character-selecting
   predicates and for `dialogue`'s result. **Design one primitive, not two.**
2. **Ordered, side-effecting action lists with mid-body guards.** Identical
   requirement, independently confirmed against two unrelated bytecode VMs.
3. **A named, reusable action-sequence / handler-flattening story at import
   time**, whether the source mechanism is dynamic (`PASS`, resolved via a
   live class hierarchy) or static (`Call`, a literal offset). The IR only
   ever needs the *result* of flattening — a resolved, structured handler —
   so the same downstream representation serves both.

**Where they differ, and why the IR must still support both shapes:**
AESOP's hard problem is *object-relative addressing* (other objects' state,
message sends to named targets) — `.INF` has none of that; its state is
flat (block/level/global/monster flags, party state), because Westwood levels
have no object graph, only a grid and a flag bank. `.INF`'s hard problem is
*hidden interpreter registers as an implicit channel between two opcodes* —
AESOP's handlers are, by contrast, cleanly scoped per message send with no
analogous cross-opcode side channel observed in the AESOP sample. So: the
IR needs entity/object addressing for AESOP but can treat `.INF` content as
addressing only "this trigger's own block" — the addressing primitive should
be present but its EOB1/2 importer usage will be nearly trivial (self plus a
flat flag/party namespace) compared to its EOB3/DH usage. Building it against
`.INF` first (cheap, low-cardinality) and proving it out fully against AESOP
(object graph, higher-cardinality) is a reasonable, low-risk order — which
matches MASTER §10's sequencing (Westwood first, AESOP as "the risk") for a
reason beyond mere convenience: the Westwood lift is a good, cheap smoke test
of the *shape* of the binding/addressing primitive before AESOP stress-tests
its *scale*.

---

## 10. Open questions

1. **Coverage across all 44 levels.** This spike sampled 2 EOB1 triggers in
   depth (12 EOB1 level texts and one EOB2 fragment available for spot
   checks) — a systematic sweep (opcode-frequency histogram, `Call`-target
   fan-in, any computed-jump sighting) across all EOB1+EOB2 levels would
   firm up "no computed jumps" and "`specialEvent` is rare" as *proven*
   rather than *observed-in-sample*.
2. **EOB2 header replicator.** Not attempted here (out of scope for
   feasibility). Needed before Phase 1's real EOB1→EOB2 importer work
   (PHASES §5) — kyra's `initLevelData()` is the ground truth to port.
3. **Exact semantics of `trigger.reason` bit values.** `eoblib`'s 12-entry
   `collapseTriggerFlags` map gives the closed cardinality but not each
   value's name; cross-referencing every `runLevelScript(block, flagsBit)`
   call site across `engines/kyra/**/*.cpp` (started here, not exhaustive)
   would produce the authoritative enum names for the IR.
4. **`dialogue`/`specialEvent` census.** How many distinct `dialogue`
   sub-opcodes and `specialEvent` ids exist across all 12+16 levels, and how
   many are truly one-off vs. reusable patterns (e.g. "class/race rune door"
   recurs in `LEVEL1.INF` per the wiki dump `0x087D`/`0x08D5` and would be
   worth a real named domain action rather than an authored escape hatch)?
5. **Monster-scoped flags/state.** `SetFlagToken`/`ClearFlagToken` support a
   per-monster target in addition to level/global — not exercised in either
   hand-lift here; worth a dedicated sample once monster AI/combat scripting
   is in scope.
