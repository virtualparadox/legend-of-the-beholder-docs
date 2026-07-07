# P1 · M4 — EOB1 `.INF` event graph + doors + decorations + one trigger — Design

> **Status:** design (brainstorming deliverable). The detailed, TDD task-by-task
> implementation plan is produced against this design by `superpowers:writing-plans`
> and appended below as "Tasks" once the maintainer approves this design. This document
> fixes the scope, the `.INF` decode + lift model, the IR event layer (a minimal but
> correctly-shaped slice of ADR-0002), the door/decoration model, the executor, and the
> differential — before any code is written.

**Goal.** Bring EOB1 level 1's **content behaviour** to life: decode the `.INF` level script,
**lift** its trigger bytecode into the declarative `event → condition → action` IR (ADR-0002), run
it in the engine, and render the **doors** and **wall decorations** the `.INF`'s `0xFB`
wall-mapping + `0xEC` overlays define — so that **one real trigger fires end-to-end**: the party
steps on a block and a door opens (visibly passable + rendered), matching `kyra`/DOSBox.

## Where this sits — the P1 milestone ladder

This is **M4** of the five-milestone P1 EOB1 vertical slice (`PHASES.md` §"Phase 1"). M1 (CPS decode),
M2 (static render), M3 (walkable level) are **done**. M4 introduces the **lift competency** the docs
flag as the load-bearing skill that must exist from the first Westwood importer (MASTER §6; PHASES
§3.3): the `.INF` is itself a small imperative bytecode VM (a 10-deep call stack, a fused
condition+branch RPN evaluator, static jump/call offsets), and M4 lifts it — at import time — into
the closed-vocabulary event graph ADR-0002 designs. M5 (JSON lossless round-trip) follows. M4's
differential retires the **`.INF` lift + trigger-fidelity risk**.

**Executed as one milestone** (maintainer's choice), with a **minimal but correctly-shaped**
event-model slice (maintainer's choice): the vocabulary is closed (sealed) from day one, but its
membership is only what LEVEL1's demo trigger + a few common opcodes need — it grows as EOB2/later
levels land (MASTER §10; ADR-0002 "closed in principle, membership grows"). Even minimal this is a
large milestone (~M2+ sized: decode + lift + IR event layer + executor + door rendering + decoration
rendering), realistically 12–18 tasks.

## Scope

**In scope (tight whole-M4):**
- **`.INF` full decode** (`importer-westwood`): the header (incl. the 3 timer-callback bytes eoblib
  skips), the `0xEC` decoration-overlay + `0xFB` wall-mapping commands, the raw script blob, and the
  trigger table. Byte-exact cross-check vs `eoblib` where comparable.
- **The lift** (`importer-westwood`): decode the ~10 opcodes M4 needs, structure control flow at
  **import time** (ADR-0002 D7), lift the RPN condition to a `Condition` expression, map opcodes to a
  **minimal closed `Action` vocabulary**.
- **IR event layer** (`beholder-ir`): the ADR-0002 minimal-slice value types (`Reason`,
  `TriggerBinding`, `Handler`, `Action`, `Condition`, `Flag`, `LevelEvents`), `WallArchetype.DOOR`, a
  typed `Door` feature, and a `Decoration` model.
- **`engine-core` executor**: fire `STEP_ONTO` on movement → run the block's handler → evaluate
  conditions → execute actions (mutate runtime flags / door states / wall overrides); collision
  extended so a `DOOR` is passable iff open.
- **Doors**: `WallArchetype.DOOR` + a typed open/closed `Door` state (**snap**, no slide animation) +
  **real EOB1 door-leaf rendering** (spike the door-shape asset format first; wall-swap fallback if
  intractable) + collision.
- **Decorations**: the LEVEL1 wall decorations (buttons/plates/grates) rendered via the `0xEC`
  overlay (`.cps` + `.dat`).
- **The demo trigger**: step on a pressure plate → a door opens → visibly passable + rendered.

**Deferred (explicit):**
- **Door slide animation** — snap open/closed for M4 (consistent with M3's no-tween ruling,
  ADR-0003); the sliding animation + the per-level door scale tables' animation dimension are a later
  QoL.
- **The full action vocabulary** — only LEVEL1's demo-needed actions; the rest (combat/spell/AI,
  `NEW_ITEM`, `DAMAGE`, `ENCOUNTER`, `GIVE_XP`, `CONSUME`, `STEAL`, `IDENTIFY`, `LAUNCHER`,
  `CREATE_MONSTER`) grow as later games/levels need them.
- **`CHANGE_LEVEL`** (stairs to level 2) — needs multi-level loading; a later milestone.
- **Cutscene/dialogue escape hatches** (`sequence`/`dialogue`/`specialEvent`, ADR-0002 D9) — deferred.
- **Mouse / `CLICKED` reason** (clicking wall switches) — M4's demo is step-onto; the mouse grey-zone
  (ADR-0003 #4) and the `CLICKED` wiring stay deferred (the `Reason` enum still enumerates it).
- **Timer reasons** (`TIMER`) — enumerated, not wired for M4's demo.
- **EOB2-only reasons/opcodes** (monster placed/removed, spell-cast, tick-timer).

## Decisions settled in brainstorming (2026-07-08)

- **Whole M4 as one milestone** (not decomposed) — maintainer's choice; the tight scope above keeps
  it achievable.
- **Minimal but correctly-shaped event model** (ADR-0002 slice) — the sealed vocabulary's membership
  is only what real LEVEL1 needs; grows later (YAGNI + MASTER §10).
- **Real EOB1 door-leaf rendering**, gated on a first-task **door-shape asset spike**; **wall-swap
  fallback** (open door = wall-index swap, no leaf) if the asset format proves intractable — the DOOR
  archetype, collision, open/close state, and the whole trigger pipeline are identical either way.
- **Doors snap open/closed** (no slide animation) — M3 no-tween posture.
- **Door state is a typed `Door` feature (ADR-0002 D8), NOT the kyra wall-type-index-run hack** —
  a clean-room improvement: kyra overloads the wall-type value (a door occupies a run of 5 wall
  indices, incremented per animation tick); we model door state explicitly as a typed field, so the
  IR is data, not packed index arithmetic to be re-executed.

## Format facts (clean-room, grounded in eoblib + the EOB RE wiki + kyra as read-only oracle)

**`.INF` file layout** (eoblib `Inf.java`; wiki `eob.inf`): `triggersOffset u16 LE @0` (the file offset
where the trigger table begins) · maze/vcnVmp/palette names (12 B NUL each) · **9 skipped bytes** =
`TypeOfCallingCommand1 u8` (0=off/1=by-ticks/2=by-steps) + `freqInTicks u16` + `freqInSteps u16` (the
periodic-timer config eoblib discards) · monster defs · `nbrECFBCommands u16`. Then the EC/FB command
stream; then the **raw script blob** `[scriptEnd, triggersOffset)`; then the **trigger table**
`[triggersOffset, EOF)`.

**Wall-mapping** — 23 hardcoded defaults (indices 0..22): `wallType ∈ {0,1,2,3}` (0=open air, 1/2=two
plain-wall variants, 3=door/decorated); an index is a *plain wall* iff `wallType ∈ {1,2}`. Any index
≥23 must be defined by a `0xFB` command, else the maze cell falls back to index 0.
- **`0xEC` command** — load a decoration overlay: `cpsName` (12 B) → `.cps` graphics, `datName` (12 B)
  → `.dat` rectangle table.
- **`0xFB` command** — 5 bytes: `wallMappingIndex, wallType, decorationID (0xFF=none), eventMask,
  flags`. (The wiki's byte-4/5 labels differ; **eoblib's runtime-verified semantics are canonical**:
  byte-2 = passability category, byte-5 = a per-index behaviour bitmask.)

**Trigger table**: `count u16` then `count ×` records, each **`block u16, reasonFlags u8,
scriptOffset u16`** (5 B). The `reasonFlags` byte's bits 3..7 (`(f & 0xFFF8) >> 3`, OR'd with `0xE0`)
select which reasons re-fire the block's script; **click + periodic-timer always fire**. Reasons
(post-shift bit): `0x01` step-onto, `0x02` step-off, `0x04` item-dropped/flying-landed, `0x08`
item-taken, `0x10` flying-object-new-block, `0x20` timer, `0x40` clicked (EOB2 adds monster/spell).
LEVEL1 has **38 triggers**: 20× click/timer-only (mostly `MESSAGE` flavour), 13× step-onto, plus a
few weight-plate/door ones.

**Bytecode opcodes** (~27 EOB1; single negative-byte IDs in kyra, `0xFF..0xE5` in eoblib). The ~10 M4
needs: `SET_WALL` (write a wall-mapping index into one/all sides of a block), `OPEN_DOOR`/`CLOSE_DOOR`
(block-id operand), `MESSAGE` (string+colour), `SET_FLAG`/`CLEAR_FLAG` (level/global/monster scope),
`CONDITIONAL` (RPN + `falseAddress` jump), `JUMP`/`CALL` (literal absolute u16 offset), `RETURN`/`END`.
Control flow: static offsets, a **10-deep call stack**, no back-edges/computed jumps observed
(ADR-0002 D7: reducible → structure at import). **RPN condition**: a sub-op stream (values push,
operators pop-2-push-1, literals 0..0x7F) terminated by a sentinel, then the u16 false-branch target
— a total, side-effect-free boolean stack (EOB1; EOB2's side-effecting predicates are D5's concern,
out of scope).

**The demo trigger candidates** (real LEVEL1):
- **Simplest (guaranteed first end-to-end, no door asset):** triggers #37/#38 — step onto block
  `[1,2]`/`[1,1]` → unconditional `SET_WALL(one-side, block [2,1], side=W, → wall index 49/69)`. A
  wall-swap "secret passage": event(block, step-onto) → *no condition* → one `SET_WALL` action.
- **Primary demo (real door):** a pressure-plate/weight-plate → `OPEN_DOOR(doorBlock)`; e.g. trigger
  #19 opens door `[17,23]` (with an RPN item-count condition — exercises the full pipeline incl. a
  guard). The differential targets a door opening.

**Doors (kyra oracle → clean-room re-model):** kyra encodes door state *as the wall-type value* (a
door = a run of 5 consecutive wall indices; a timer nudges the value ±1 per tick to slide/animate;
`_wllWallFlags[value]` bit0=passable, bit3=is-door, bit4=open-terminal, bit5=closed-terminal). **We
re-model this cleanly**: a `Door` is a typed feature `(block, edge, state∈{OPEN,CLOSED},
doorGraphicId)`; `WallArchetype.DOOR` marks the edge; collision reads the door's state (open ⇒
passable). Snap between OPEN/CLOSED (no per-tick slide). Door-leaf graphics: **asset format TBD — a
first-task spike** (starting points: `eoblib`, `wallsetviewer`, kyra's `_doorShapes[6]` /
`_dscDoor*` load path in `staticres_eob.cpp`).

**Decorations (kyra oracle):** a wall-mapping index → a decoration chain (buttons/levers/plates/
alcoves/grates) drawn over the wall at the far→near view slots, from the `0xEC` overlay's `.cps`
shapes + `.dat` rectangle table (`_dscShapeIndex`/`_dscShapeCoords` in kyra are the projection).

## Architecture — module map

Current reactor (acyclic): `ir ← importer-westwood`, `ir ← render-core`, `ir ← engine-core`,
`engine-core + render-core + importer + libGDX ← render-libgdx`. M4 adds within these modules; the
graph is unchanged.

- **`beholder-importer-westwood`** — the intelligence lives here (ADR-0002 D7 "the importer is where
  the intelligence lives"): `InfScript` decoder (header + `0xEC`/`0xFB` + script blob + trigger
  table) + the **lift** (`InfLifter`: bytecode → structured event graph) + the door/decoration asset
  decoders + `WestwoodToIr` extensions producing the IR event layer.
- **`beholder-ir`** — the event-layer value types + `WallArchetype.DOOR` + `Door` + `Decoration` +
  `LevelEvents` (all immutable, house pattern).
- **`beholder-engine-core`** — the `EventExecutor` (closed-vocabulary interpreter — a *simple*
  executor, not a re-hosted VM) + the runtime mutation surface (flags, door states, wall overrides)
  on `RuntimeState` + collision extension.
- **`beholder-render-core`** — door-leaf rendering + decoration rendering (new far→near projections,
  reusing the M2 wall-slot approach).
- **`beholder-render-libgdx`** — fire `STEP_ONTO` on movement, render the new content.

## IR event layer (ADR-0002 minimal slice)

Immutable value types in `beholder-ir` (house pattern: final class, private ctor, `of(...)` factory,
fail-closed guards):
- **`Reason`** (closed enum): `STEP_ONTO, STEP_OFF, CLICKED, ITEM_DROPPED, ITEM_TAKEN, FLYING_LANDED,
  TIMER`. M4 wires `STEP_ONTO`; the rest are enumerated (illegal states unrepresentable, ADR-0002 D10).
- **`TriggerBinding`**: `(GridPosition block, Set<Reason> reasons) → Handler`. Per ADR-0002 D10 the
  reason is an ordinary operand, so one binding can multiplex reasons.
- **`Handler`** (ADR-0002 D1): an ordered list of **guarded steps** — each step is either an `Action`
  or a `when Condition: [steps] [else: steps]` guard with early return. For M4's minimal shapes this
  is a flat action list plus at most a single leading guard (LEVEL1's demo triggers are that simple).
- **`Action`** (sealed, minimal M4 set): `SetWall(block, side, wallIndex)`, `OpenDoor(block)`,
  `CloseDoor(block)`, `ShowMessage(text, colour)`, `SetFlag(scope, id)`, `ClearFlag(scope, id)`. Sealed
  so adding a member is a reviewed schema change.
- **`Condition`** (expression tree, total/loop-free, ADR-0002 D3): `Comparison(op, lhs, rhs)`,
  `BoolOp(and/or, …)`, `Literal(int)`, and value queries `FlagValue(scope, id)`, `TriggerReason`,
  `WallIndexAt(block, side)`, `PartyAt(block)`, `ItemCountAt(block, …)` — only the ones LEVEL1's demo
  needs; grows later.
- **`Flag`**: a small closed namespace, scopes `LEVEL` + `GLOBAL` (typed boolean/counter). Per-monster
  scope deferred with monsters.
- **`Door`** (feature, ADR-0002 D8): `(GridPosition block, int edge, DoorState state, int doorGraphicId)`.
- **`Decoration`**: `(wallMappingIndex → decoration shape refs)` — the appearance overlay.
- **`LevelEvents`**: the per-level `List<TriggerBinding>` + the flag declarations + the door/decoration
  tables, bundled with `LevelGeometry`/`TileSet` into the `Level` (base = immutable; runtime = the
  mutable overlay, below).

## The lift (importer)

`InfLifter` runs at **import time** (ADR-0002 D7): decode the trigger table + the referenced bytecode,
then per handler:
1. **Decode** the opcode stream into raw tokens (the ~10 M4 opcodes).
2. **Structure control flow** into `Handler` steps: `CONDITIONAL(RPN, falseAddr)` → a `when cond:`
   guard whose body is the fall-through up to `falseAddr`; `JUMP` forward → a converging exit/early
   return; `CALL`/`RETURN` (static offsets, ≤10 deep) → **inline** the callee's steps (flatten); `END`
   → handler end. Scoped to LEVEL1's real shapes; any irreducible flow (none expected) is flagged for
   review (ADR-0002 D7 fallback: a labelled step inside one handler only — not a VM).
3. **Lift the RPN** to the `Condition` expression tree (stack-balance-checked, per eoblib's own
   `main()` verifier).
4. **Map opcodes → the minimal `Action` vocabulary.**
Decode is cross-checked byte-exact vs `eoblib`'s `Inf`/`Script` where comparable; the lifted structure
is cross-checked against the EOB RE wiki's decompiled LEVEL1 script (`files/level1.txt`).

## Doors (feature + collision + rendering)

- **Geometry/state:** `WallArchetype.DOOR` on the edge; a `Door` feature carries `OPEN`/`CLOSED`.
  `RuntimeState` gains the mutable door-state overlay. Collision (extends M3): a `DOOR` edge is
  passable iff its door is `OPEN`; `CLOSED` blocks (like `SOLID`). Snap between states.
- **Rendering (spike-gated):** first task recons/decodes the **EOB1 door-shape asset format**. On
  success: render the door leaf (CLOSED) / open frame (OPEN) at the far→near slots (a new
  `DoorRasterizer` beside the M2 `WallSlotRasterizer`, using the kyra `_dscDoor*` projection as the
  read-only oracle for slot geometry — a fixed engine constant, not IR). **Fallback** (asset
  intractable): a `DOOR` renders via its wall-mapping index (CLOSED = a solid/door-look index, OPEN =
  an open index — a wall-swap), so the door is still visibly open/closed + passable, just without a
  bespoke leaf. Either way the collision + trigger pipeline is identical.

## Decorations (rendering)

Decode the `0xEC` overlay's `.dat` rectangle table (+ the `.cps` shapes, reusing M1's CPS decoder);
a `DecorationRasterizer` draws each wall-mapping index's decoration shapes over the wall at the
far→near view slots (kyra `_dscShapeIndex`/`_dscShapeCoords` as the read-only projection oracle). M4
renders the LEVEL1 decorations (buttons/plates/grates) the maintainer observed missing.

## Executor + wiring (engine-core)

- `EventExecutor.fire(runtimeState, level, block, reason)`: look up the block's `TriggerBinding`; if
  it matches `reason`, run the `Handler` — evaluate each guard's `Condition` against the runtime state
  (a small, total expression evaluator — **not** a bytecode VM), execute the `Action`s in order (each
  mutates the runtime overlay: set/clear flag, open/close door, set-wall override), honour early
  return.
- **Wiring:** on a successful M3 move into a block, `EventExecutor.fire(..., STEP_ONTO)`. The runtime
  mutation surface (flags + door states + wall-index overrides) is the real payload of M3's
  base-vs-runtime split. Rendering reads the runtime overlay (a door's live state; a wall's override
  index) so the trigger's effect is visible.

## Differential correctness (§8)

Four legs (no ScummVM; matching M1–M3 posture):
1. **`.INF` decode byte-exact vs `eoblib`** (in-gate IT on real `LEVEL1.INF`, `assumeTrue`-skip): the
   header, wall-mapping table, and trigger table decode identically to eoblib.
2. **The lifted event graph matches the wiki's decompiled LEVEL1 script** (structural): the demo
   trigger's lifted `(block, reason) → actions` equals `files/level1.txt`'s decompilation — a
   hand-checkable structural golden.
3. **The trigger fires correctly** (in-gate state test): a scripted step onto the plate block →
   assert the door's runtime state flips to OPEN and the door edge becomes passable (and the SET_WALL
   variant swaps the wall index) — self-consistent, deterministic.
4. **Maintainer eyeball** — doors + decorations now render; walking onto the plate opens the door
   in the interactive window (the M2/M3 human-verdict leg; a contact-sheet + the live run).

## Risks + internal sequencing

- **Risks:** (a) the **EOB1 door-shape asset format** (unknown — spike first; wall-swap fallback);
  (b) the **lift's control-flow coverage** (scope to LEVEL1's real shapes; the recon shows they're
  shallow/reducible, but the CALL/RETURN flatten + the RPN lift are the novel bits).
- **Internal sequencing** (the natural task order — retires risk incrementally):
  1. `.INF` decode (header + `0xEC`/`0xFB` + script blob + trigger table), byte-exact vs eoblib.
  2. IR event-layer value types (Reason/Handler/Action/Condition/Flag/Door/Decoration/LevelEvents).
  3. The lift (bytecode → structured event graph) — the research core; validated vs the wiki
     decompilation + the byte-exact decode.
  4. `engine-core` executor + `STEP_ONTO` wiring + runtime mutation surface → **the `SET_WALL`
     wall-swap demo works end-to-end here (no door asset) — retires the lift risk.**
  5. Doors: DOOR archetype + collision + open/close state (the door-opening trigger now fires + is
     passable, even before leaf rendering).
  6. Door-shape asset spike + door-leaf rendering (or the wall-swap fallback).
  7. Decoration decode + rendering.
  8. The door-opening demo end-to-end + the maintainer eyeball.

## Self-Review (design)

- **Placeholders:** none — the one genuine unknown (door-shape asset format) is an explicit,
  scoped first-task spike with a defined fallback, not a gap.
- **Consistency:** the event layer follows ADR-0002 (minimal slice: D1 handler shape, D3 expression
  language, D7 import-time lift, D8 typed feature state, D10 closed reason taxonomy); the door model
  is the clean-room re-model of kyra's wall-index-run (ADR-0002 D8 typed field, not packed
  arithmetic); the no-tween snap-door is consistent with ADR-0003/M3; the module graph is unchanged
  and acyclic; the differential matches the M1–M3 no-ScummVM posture.
- **Scope:** focused on one trigger end-to-end + the doors/decorations it needs; combat/items/
  multi-level/cutscenes/mouse/timers/animation explicitly deferred with their owning future work.
- **Ambiguity:** the two genuine forks (door-visual fidelity; demo-trigger choice) are settled —
  real door leaf (spike + fallback); the door-opening trigger is the differential, with the SET_WALL
  wall-swap as the guaranteed-simplest first end-to-end.

---

**Next:** on the maintainer's approval of this design, `superpowers:writing-plans` expands it into the
TDD task-by-task implementation plan (the internal sequencing above becomes ~12–18 tasks, each with
RED→GREEN steps, `mvn verify` evidence, and per-task review), appended to this document.

**Execution outcome (2026-07-…).** *(To be filled after execution — matching the M1–M3 execution-outcome sections.)*
