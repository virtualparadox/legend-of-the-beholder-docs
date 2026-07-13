# P2 · M3 — EOB2 `.INF` event lift — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> to implement this plan task-by-task (fresh implementer + independent review per task, a
> whole-branch review at the end, `--no-ff` merge on the maintainer's eyeball verdict; no
> worktree). TDD per task (RED → GREEN → commit). Steps use checkbox (`- [ ]`) syntax. Style
> follows the P1 M4 + P2 M1/M2 plans: the lift/parse tasks are *discovery against a read-only
> oracle + real data* (the "Format facts" + each task's differential are the concrete spec);
> the IR/engine/boot tasks are *pure logic* and carry concrete code. Every task obeys the
> Global Constraints.

**Goal:** Lift the EOB2 `.INF` trigger bytecode (block B/C) into the **shared** event graph at
import time (a control-flow-reconstructed `event → condition → action`, **not** a VM), fire
**one real EOB2 trigger end-to-end** through the M2 extract/boot pipeline, and close the two M2
carry-forwards (the IR-carried wall-mapping table → a game-agnostic `SET_WALL` boot resolver;
the `initialDoorState` edge fix).

**Architecture:** A parallel `Eob2InfLifter` (+ the EOB2 opcode map + action/condition/handler
lifters) mirrors the P1 M4 EOB1 `InfLifter` chain — the lift *competency* (CFG reconstruction,
RPN→AST) is reused (EOB2 is the same VM), but the opcode map is EOB2-specific; it emits **only**
shared `LevelEvents`/`TriggerBinding`/`Action`/`Condition`/`Reason`. `LevelGeometry` gains a
per-index wall-mapping table so `boot` builds the `SceneOverlay` resolver from IR (replacing M2's
inert identity). Both extractors emit non-empty triggers + the table (undoing M2's base-EOB1
regression); the shared `EventExecutor` fires them unchanged.

**Tech Stack:** Java 21, Maven multi-module, JUnit 5 + AssertJ. Read-only oracle: kyra
`EoBInfProcessor` (`script_eob.cpp`) + `scene_eob.cpp`/`darkmoon.cpp`. Real data: the user's own
EOB2 `LEVEL1.INF`.

## Global Constraints

- **JDK 21** (`JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home`);
  `mvn spotless:apply` then `mvn verify`; single-module `-pl <m> -am` **or** root verify.
- **All `CODING_STANDARDS` gates green:** one `return`/method, no `instanceof` (visitor/`getClass`),
  no `switch yield`, cyclomatic ≤ 8, NPath ≤ 120, ≤ 12 fields / ≤ 18 methods / ≤ 5 effective
  params, coupling ≤ 10, no `return null`, `final` params, braces everywhere, ≤ 140-char lines,
  no `System.out`, NullAway `ERROR`, full Javadoc, **every magic value a named `ALL_CAPS`
  constant + a what-and-why comment**.
- **Coverage: JaCoCo ≥ 0.90 branch AND line.** `jacoco:check` counts **Surefire (unit) only** —
  skip-safe ITs (Failsafe) do NOT count (verified in M2/M3). So every new production class
  (the lifter, the parse extensions) needs **data-independent synthetic-fixture UNIT coverage**
  (e.g. a synthetic uncompressed-CPS block-B/C buffer, mirroring M1's `Eob2InfScriptTest`);
  `model/`/records are exempt but their guards + value-equality are behaviour-tested.
- **Two importers, one IR (MASTER §4.1/§10):** every EOB2 construct binds to the shared
  `beholder-ir` types; vocabulary extensions grow the **shared** enums, shaped for the family.
  **No embedded VM** — a lift miss means *extend the vocabulary*, never interpret. A shared
  Westwood-`.INF`-lift abstraction is extracted **only where real duplication forces it**
  (never top-down).
- **Clean-room:** kyra is a read-only oracle — never copied. **Never commit game data:**
  real-data ITs are `assumeTrue`-skip-safe; artifacts to git-ignored `target/`; synthetic
  fixtures/goldens are our own content and ARE committed.
- **IR back-compat (ADR-0001):** the new wall-mapping-table field is **additive/optional** — M2's
  already-written IR (without it) still loads (default/fallback), so no `irVersion` major bump.
- **Package roots** `eu.virtualparadox.beholder.{importer.westwood,ir,ir.json,engine,app}`.
- Branch `feat/p2-m3-eob2-inf-events`; commit per task; whole-branch review + `--no-ff` merge.
  Base = current `main` (post-M2, `13db07f`).

---

## Plan-level design decisions (the shapes the design deferred to the plan)

- **D-A · Parallel `Eob2` lift chain, shared IR.** `Eob2InfLifter` + `Eob2InfScriptOpcodes` (the
  EOB2 opcode map) + the EOB2 action/condition/handler lifters live alongside the EOB1 `Inf*`
  chain in `beholder-importer-westwood`, emitting **only** shared `LevelEvents`/`TriggerBinding`/
  `Action`/`Condition`/`Reason`. If the implementation finds an action/condition/handler lifter
  byte-identical to EOB1's (same shared vocabulary, same RPN model), extract a shared core **then**
  — never upfront.
- **D-B · The IR wall-mapping table = a new `WallMappingTable` value type on `LevelDocument`**
  (revised from the design's `LevelGeometry` lean after the recon). Rationale: the table is a
  per-**level** dictionary (`wall-index 0..255 → wall-set`), not a per-**cell** array like
  `LevelGeometry`'s `archetypes`/`wallSets`; putting it on `LevelGeometry` (a not-a-record
  array-holder with 9 test refs, an `of(...)` factory called by 3 producers, and an
  `equals`/`hashCode` + open-edge invariant) churns a per-cell type with a per-level concern. So:
  a new `WallMappingTable` value type (final class, `int[256]` behind a scalar `wallSetForIndex(int)
  -> int` accessor + `of(int[])` factory + value `equals`/`hashCode`, following the
  `LevelGeometry`/`TileSet`/`PixelSheet` not-a-record-array pattern that dodges SpotBugs
  `EI_EXPOSE_REP`), carried as a **`wallMappings` field on `LevelDocument`**. **Back-compat:**
  the field is **optional in JSON** (not in the schema `required` list); the deserializer defaults
  it to an empty table when absent, so M2's already-written IR (no `wallMappings`) still loads (its
  resolver is never invoked — M2 has no `SET_WALL`). Both extractors populate it; `boot` builds
  `SceneOverlay.wallMappingResolver = i -> doc.wallMappings().wallSetForIndex(i)`, replacing M2's
  `IntUnaryOperator.identity()` — **without importing any `beholder-importer-westwood` type**
  (the one-IR boundary).
- **D-C · Both games get non-empty triggers + the table in M3.** With the resolver IR-carried,
  the M2 base-EOB1 regression is undone: `Eob1Extractor` writes `InfLifter.lift(script).triggers()`
  (its real EOB1 triggers restored) + the table; `Eob2Extractor` writes `Eob2InfLifter.lift(script)`
  's triggers + the table. The shared `EventExecutor` fires both games' triggers unchanged.
- **D-D · Minimal vocabulary; extend only where the demo trigger forces it.** Messages: the lift
  dereferences EOB2's string-table index → the existing `Action.ShowMessage(text, colour)` (no IR
  change). Opaque cutscenes (`dialogue`/`specialEvent`/`drawScene`) → a shared opaque
  named-cutscene `Action` **iff** the demo trigger's block references one. New `Action`/`Condition`/
  `Reason` members added only if the chosen real trigger's opcodes/RPN demand them. If every real
  LEVEL1 trigger needs RPN outside the minimal set (the EOB1 M4 precedent), a documented
  **half-synthetic** demo trigger bridges. *(The exact trigger + delta are pinned by the recon in
  the tasks below.)*
- **D-E · The `initialDoorState` edge fix.** `RuntimeState.initialDoorState` filters
  `level.events().doors()` by `block` only, ignoring `Door.edge()`; fixed to match `(block, edge)`
  before a trigger can open one door on a multi-door block.

## Design facts (grounded in the code + the kyra oracle)

### A · The EOB1 lift chain — the exact template `Eob2*` mirrors (all in `…importer.westwood`)
- `InfLifter.lift(InfScript script) -> LevelEvents` — orchestrator: iterates `script.triggers()`,
  `bindings.add(InfTriggerBinder.bind(trigger, handlerLifter))` per trigger, lenient (catches
  per-trigger `IllegalArgumentException` → a skip list), returns `new LevelEvents(bindings,
  List.of(), List.of())` (doors/decorations supplied separately by the caller). Also
  `liftReport(InfScript) -> InfLiftResult` (audit: events + `List<Skipped>`).
- `InfTriggerBinder.bind(InfTrigger t, InfHandlerLifter h) -> TriggerBinding`: `reasons =
  decodeReasons(t.reasonFlags())` (`refireMask = ((flags & 0xFFF8) >> 3) | 0xE0`, mapped over
  the reason bits), `handler = h.lift(t.scriptOffset())`, `block = InfDecode.toPosition(t.block())`.
  **EOB2 divergence point:** the `0xE0` always-on mask (timer 0x20, click 0x40) is EOB1-specific,
  and EOB2 adds reasons (0x80 timer, 0x200/0x400/0x800 monster/spell) the EOB1 binder ignores.
- `InfHandlerLifter(byte[] rawScript, int scriptBaseOffset)`; `.lift(int scriptOffset) -> Handler`
  — reconstructs reducible control flow (jump / call inlined to depth 10 / conditional / leaf
  action); a conditional builds `new Handler.Guard(condition, thenSteps)`; `index(offset) = offset
  - scriptBaseOffset` with range checks; fails closed on back-edges / non-terminating branches.
- `InfConditionLifter.lift(byte[] rawScript, int conditionalOffset) -> ConditionResult(Condition
  condition, int nextIndex, int falseTarget)` — RPN stack machine → `Condition` AST; `0xEE` ends,
  then a `u16 falseAddress`. Supports flag/party value-ops; excludes `GET_TRIGGER_FLAG`.
- `InfActionLifter.lift(byte[] rawScript, int cursor) -> ActionResult(List<Action> actions, int
  nextIndex)` — opcode → `Action` (SET_WALL one/all-sides, OPEN/CLOSE_DOOR, MESSAGE, SET/CLEAR_FLAG).
- `InfDecode`: `readU8(byte[],int)`, `readU16(byte[],int)` (LE), `toPosition(int block) ->
  GridPosition.of(block & 31, block / 32)` — **game-agnostic; reuse verbatim IF EOB2's block
  packing is the same 32-wide grid** (verify in the recon).
- `InfScriptOpcodes` — the EOB1 opcode/RPN/flag-target constants + `describe(int)`. **EOB2 needs its
  own `Eob2InfScriptOpcodes` if the byte values differ** (they do — different `_commandMin` & set).
- `record InfTrigger(int block, int reasonFlags, int scriptOffset)`;
  `record InfLiftResult(LevelEvents events, List<Skipped> skipped)` + `record Skipped(int block,
  int scriptOffset, String reason)`.
- `InfScript`: `rawScript() -> byte[]`, `scriptBaseOffset() -> int`, `triggers() -> List<InfTrigger>`,
  `wallMapping(int) -> InfWallMapping` (`.wallType()`), `hasWallMapping(int)`.
- **`Eob2InfScript` today has NO script surface** — only block-A geometry
  (`mazeName/gfxName/soundName/subLevelCount/doors/decorationOverlays/wallMapping/hasWallMapping`).
  M3 **adds** `rawScript()`/`scriptBaseOffset()`/`triggers()` + the string-pool. The decompressed
  buffer opens with a `u16 LE` block-B offset at position 0 (`BLOCK_B_OFFSET_POS`) — where block A
  ends and block B (script + string pool) begins. Its wall-mapping returns `Eob2WallMapping(vmpIndex,
  decIndex, specialType, flags)` — not EOB1's `InfWallMapping`.

### B · The shared event IR the lifter builds (all in `…ir.event`, exact constructors)
- `record TriggerBinding(GridPosition block, Set<Reason> reasons, Handler handler)` — `reasons`
  **non-empty**.
- `record Handler(List<Step> steps)` — **non-empty**; `sealed interface Step permits Action, Guard`;
  `record Handler.Guard(Condition when, List<Step> then)` — `then` **non-empty**.
- `sealed interface Action`: `SetWall(GridPosition block, int side/*0..3*/, int wallIndex/*>=0*/)`,
  `OpenDoor(GridPosition block)`, `CloseDoor(GridPosition block)`, `ShowMessage(String text, int
  colour/*0..255*/)`, `SetFlag(FlagRef)`, `ClearFlag(FlagRef)`.
- `sealed interface Condition`: `Comparison(Op, lhs, rhs)`, `BoolOp(BoolKind, lhs, rhs)`,
  `Literal(int)`, `FlagValue(FlagRef)`, `TriggerReasonValue()`, `WallIndexAt(GridPosition, int
  side)`, `PartyAt(GridPosition)`; `enum Op{EQ,NE,LT,LE,GT,GE}`, `enum BoolKind{AND,OR}`.
- `enum Reason{STEP_ONTO,STEP_OFF,CLICKED,ITEM_DROPPED,ITEM_TAKEN,FLYING_LANDED,TIMER}` — **ordinal
  order is load-bearing** (JSON enum + `ConditionEvaluator` use `.ordinal()`); a new member appends.
- `record FlagRef(Scope scope, int id)`; `enum Scope{LEVEL,GLOBAL}`.
- `record LevelEvents(List<TriggerBinding> triggers, List<Door> doors, List<Decoration> decorations)`.

### C · The engine executor (`…engine`) + the D-E fix target
- `EventExecutor.fire(RuntimeState state, Level level, GridPosition block, Reason reason)` — fires
  bindings where `block.equals(binding.block()) && binding.reasons().contains(reason)`.
- `EventActionExecutor` (ActionVisitor): `visitSetWall → state.setWallOverride(block, side,
  wallIndex)`, `visitOpenDoor → setDoorState(block, OPEN)`, etc.
- `RuntimeState`: `startingAt(Party)`, `applyCommand(MovementCommand, Level)` (**auto-fires only
  `STEP_ONTO` on a block change**; other reasons via `EventExecutor.fire` directly),
  `wallOverride(GridPosition block, int side, Level level) -> int` (override else
  `level.geometry().wallSet(...)`), `wallOverrideRaw(block, side) -> Optional<Integer>` (raw, no
  fallback — carries the raw `.INF` index a `SET_WALL` installed), `doorState(block, level)`,
  `flag(FlagRef)`, `messages() -> List<Action.ShowMessage>`.
- **D-E target — `RuntimeState.initialDoorState` (private static, edge-blind), current body:**
  ```java
  private static Door.DoorState initialDoorState(final GridPosition block, final Level level) {
      return level.events().doors().stream()
              .filter(door -> door.block().equals(block))
              .map(Door::state).findFirst().orElse(Door.DoorState.CLOSED);
  }
  ```
  Called from `doorState(block, level)`. Fix: key on `(block, edge)`. But note `doorState` takes
  only `(block, level)` — the edge must be threaded in (a signature change through `doorState` →
  its callers, incl. `IrDoorLeafScene`'s closed-test lambda). `Door` = `Door(GridPosition block,
  int edge, Door.DoorState state, int doorGraphicId)`; `Door.DoorState{OPEN,CLOSED}`.

### D · P1 M4 test templates (the M3 trigger IT/lift-IT mirror)
- `EventExecutorIT` (engine-core, real-data, skip-safe) — the **direct template**: compose a `Level`
  from `WestwoodToIr.geometry`/`.tileSet` + `InfLifter.lift(InfScript.parse(...))`, `RuntimeState.
  startingAt`, `applyCommand(FORWARD)` (fires STEP_ONTO), assert `state.wallOverride(target, side,
  level)` becomes the wiki index. For M3: swap `InfScript`/`InfLifter` → the EOB2 decoder/
  `Eob2InfLifter`, geometry → `Eob2InfGeometry.geometry(maze, script)`.
- `InfLifterIT` (importer, real-data) — asserts the lifted IR directly: `binding.handler()
  .isEqualTo(new Handler(List.of(new Action.SetWall(GridPosition.of(2,1), 3, 49))))`, reasons
  `containsExactlyInAnyOrder(...)`, and `liftReport(...)` leniency counts. Template for
  `Eob2InfLifterIT`.
- `M4TriggerIT` (render-libgdx, live) — shows the `SceneOverlay` resolver wiring (`idx ->
  InfWallGeometry.wallType(script, idx)`) M3 replaces with the IR-table resolver.

### E · The IR wall-mapping-table serde targets (D-B, `beholder-ir` + `beholder-ir-json`)
- `record LevelDocument(String irVersion, LevelGeometry geometry, LevelEvents events, String
  appearancePackId)` — M3 adds a `WallMappingTable wallMappings` component (see D-B).
- `LevelDocumentSerializer`/`LevelDocumentDeserializer` (`…ir.json.serde`) — the deserializer uses
  `Nodes.requireOnlyFields(node, "LevelDocument", "irVersion","appearancePackId","geometry",
  "events")` (a **closed whitelist** — add `"wallMappings"`) and constructs the record; the
  serializer writes a fixed field order. **Back-compat:** `wallMappings` optional → default an empty
  table when absent.
- Schema `beholder-ir-json/src/main/resources/schema/level-document-1.json` — `additionalProperties:
  false`, so add `wallMappings` under `properties` (NOT in `required`).
- **Boot (`beholder-app`) — the M2 placeholder to replace:** `Boot.java` wires `new
  SceneOverlay(decorations, doorLeaves, IDENTITY_WALL_RESOLVER)` (`IntUnaryOperator.identity()`);
  M3 replaces it with `i -> doc.wallMappings().wallSetForIndex(i)` (pure IR — no importer import).
- **Extractors populate the table:** EOB1 from `InfWallGeometry.wallType(script, i)` over 0..255;
  EOB2 from `Eob2InfScript.wallMapping(i).vmpIndex()` over the assigned indices (unassigned → 0,
  matching `Eob2InfGeometry`'s fail-open). `Eob2Extractor` also replaces its `new LevelEvents(
  List.of(), doors, decorations)` with the `Eob2InfLifter` output (D-C).

### F · EOB2 block-B/C + opcode map + the liftable LEVEL1 triggers (byte-verified against real LEVEL1)
**The `.INF` is CPS/format80-compressed** → a 7462-byte decompressed buffer (reuse
`WestwoodContainer.decompress`, as M1 does). Anchors: `u16LE@0 = 1037` (block-B offset),
`u16LE@1037 = 7076` (block-C offset), `byte@1039 = 0xEC` (monster block present), block-C
`count = 64`. **Block packing is the same 32-wide grid** — `InfDecode.toPosition(block) =
GridPosition.of(block & 31, block / 32)` works for EOB2 (verified: `0x0146 → (6,10)`), so
**reuse `InfDecode` verbatim**.

- **Block C (block→script binding) @7076:** `[u16 count=64][64 × (u16 block, u16 flags, u16
  assignedObjects)]`. `assignedObjects` = the per-block script offset **relative to the INF
  blob** (`_scriptData`); `0` = no script. `flags` bits 3..15 are the trigger mask:
  `subMask = ((flags & 0xFFF8) >> 3) | 0xE0`; a flag bit `1<<(3+k)` enables reason bit `1<<k`.
  (EOB2 reads `flags` as `u16`; EOB1 as a byte.) The `|0xE0` forces CLICK(0x40)/timer(0x20)/
  script-timer(0x80) always-on — the reason-decode's EOB2 divergence.
- **Block B (script blob):** @1037 `[u16 len=7076]`; @1039 a `0xEC`-gated monster block (in
  LEVEL1: 5 interval bytes + 30×14 monster records → the **INF bytecode begins at decompressed
  offset 1465**). The blob = `[1465, 7076)`. **String pool:** the blob's first `u16` = the
  byte offset (relative to the blob) of a NUL-separated Latin-1 string pool (LEVEL1: `4138` →
  33 strings); `getString(index)` walks it skipping `index` NULs, `0xFFFF` → none. The
  **executable bytecode starts at blob offset 2** (after the pool-offset word); per-block entry
  points are the `assignedObjects` offsets (all `< 4138`, below the pool).
- **Opcode dispatch:** `cmd = (int8)*pos++`; if `cmd <= -31 (EOB2 _commandMin) || cmd >= 0` →
  skip; else opcode index `= -(cmd+1)` (cmd bytes `0xE2..0xFF`). **The minimal set M3 lifts
  (all map to the EXISTING vocabulary):**
  | cmd | name | operand → shared Action/Condition |
  |---|---|---|
  | `0xFF` | setWallType | sel `-23`: `[u16 blk, i8 dir, i8 val]` → `SetWall(blk, dir, val)`; sel `-9`: `[u16 blk, i8 val]` → **4× `SetWall(blk, side, val)` sides 0..3** (mirrors EOB1's all-sides expansion — no new vocab) |
  | `0xFD` | openDoor | `[u16 block]` → `OpenDoor(block)` |
  | `0xFC` | closeDoor | `[u16 block]` → `CloseDoor(block)` |
  | `0xF8` | printMessage_v2 | `[u16 strIndex, i8 col, i8 pad]` → `ShowMessage(getString(strIndex), col)` (the lift **resolves the pool index → text**; no IR change — the M1 message-model finding) |
  | `0xF7` | setFlags | sel `-17`: `[i8]` → `SetFlag(FlagRef(LEVEL, id))`; sel `-16`: `[i8]` → `SetFlag(FlagRef(GLOBAL, id))` |
  | `0xF5` | removeFlags | sel `-17`/`-16` → `ClearFlag(FlagRef(LEVEL/GLOBAL, id))` |
  | `0xF2` | jump | `[u16 target]` (target relative to blob) |
  | `0xF1` | end | none (terminates) |
  | `0xEF` | callSubroutine | `[u16 offs]` (push return, jump; 10-deep) |
  | `0xEE` | eval_v2 (RPN) | RPN stream, terminator `-18`, then `[u16 elseTarget]` (= jump-when-FALSE) |
  **eval_v2 RPN** (`switch(cmd+50)`): value pushers `-46` PUSH `[i16]` → `Literal`; `-17`
  levelFlag `[i8]` → `FlagValue(FlagRef(LEVEL, id))`; `-16` globalFlag → `FlagValue(GLOBAL)`;
  `-9`/`-23` wall-index → `WallIndexAt`; `-19` currentDirection; `-32` scriptFlags(the fired
  reason) → `TriggerReasonValue`. Operators (cmd `-8..-1` → cases 42-49): `OR, AND, <=, <, >=,
  >, !=, ==` → `BoolOp`/`Comparison` (pop order: `a` = most-recently-pushed = rhs). **Unmodeled
  opcodes** (playSound `0xF6`, sequence/dialogue/specialEvent/monster/item ops) → the trigger
  is **SKIPPED** (lenient, mirroring `InfLifter`'s skip list) — so no new vocabulary is forced.
- **The demo triggers (byte-verified, real LEVEL1):**
  - **idx 4 · block `0x0146`, STEP_ONTO, scriptOff 152** → `FD 06 01  F1` = `OpenDoor(0x0106)`;
    `END`. Zero RPN, zero new vocab — the trivial lift.
  - **idx 14 · block `0x00B4`, STEP_ONTO, scriptOff 673 (THE end-to-end demo)** → `FF F7 B0 00
    00  F1` = `setWallType -9 (all-sides) block 0x00B0 val 0`; `END` → lifts to 4× `SetWall(
    0x00B0→(16,5), side, 0)` = the **secret-passage wall-removal**. Firing it (walk onto
    `0x00B4→(20,5)`) installs raw wall-index 0 (open) on `(16,5)`'s four sides; the **new IR
    resolver renders the change** through boot (index 0 → wall-set 0 = open).
  - idx 34 · block `0x0266` uses a `Comparison(FlagValue(LEVEL,7) == Literal(0))` guard +
    `ShowMessage` + `playSound` + `SetFlag` — its `playSound` is unmodeled, so it **skips**
    (its RPN path is instead proven by a **synthetic** condition-lifter unit test).
- **Vocabulary delta = NONE.** Every minimal-set opcode maps to an existing shared `Action`/
  `Condition`/`Reason` (`SetWall` all-sides → 4× SetWall; `FlagRef.Scope{LEVEL,GLOBAL}` already
  exists; `printMessage_v2` → importer-resolved `ShowMessage`). No new enum members. (The
  strongest form of the §5 thesis: the shared vocabulary absorbs EOB2's minimal triggers
  unextended.)

## File structure

- `beholder-importer-westwood/.../westwood/` — `Eob2InfTrigger.java` (record), `Eob2InfScript.java`
  (**modified**: block-B/C parse), `Eob2InfLifter.java`, `Eob2InfScriptOpcodes.java`, and the EOB2
  action/condition/handler/binder lifters (count/decomposition per the EOB1 template + D-A).
- `beholder-ir/.../ir/model/LevelGeometry.java` (**modified**: the wall-mapping table + accessor,
  D-B).
- `beholder-ir-json/.../serde/LevelGeometrySerializer.java`/`LevelGeometryDeserializer.java` +
  `resources/schema/level-document-1.json` (**modified**: the additive field's serde).
- `beholder-engine-core/.../engine/RuntimeState.java` (**modified**: the `initialDoorState` edge
  fix, D-E).
- `beholder-app/.../app/Eob1Extractor.java`/`Eob2Extractor.java` (**modified**: emit triggers + the
  table, D-C), `Boot.java` (**modified**: the IR-table-backed resolver, D-B).
- Test tree: synthetic + real fixtures per the M1/M2 rhythm; the end-to-end EOB2-trigger IT
  (`beholder-app`).

---

## Tasks

> The lift/parse tasks (1–5) are *discovery against §F's byte facts + the kyra oracle + real
> LEVEL1* — §F's tables + each task's differential are the concrete spec (not fully pre-coded, the
> P1/M1/M2 rhythm). The IR/engine/wiring tasks (6–9) carry concrete code. Every new production class
> needs **synthetic-fixture UNIT coverage** (ITs don't count — Global Constraints).

### Task 1: `Eob2InfTrigger` + `Eob2InfScript.triggers()` (block-C binding table)

**Files:** Create `.../westwood/Eob2InfTrigger.java`; Modify `.../westwood/Eob2InfScript.java`;
Test: extend `Eob2InfScriptTest` (synthetic) + `Eob2InfScriptIT` (real LEVEL1).

**Interfaces produced:** `record Eob2InfTrigger(int block, int reasonFlags, int scriptOffset)`
(parallel to `InfTrigger`); `Eob2InfScript.triggers() -> List<Eob2InfTrigger>`.

- [ ] **Step 1 (RED):** `Eob2InfScriptTest` — a synthetic decompressed buffer with `u16LE@0 = blockBOff`,
  and a block-C region at the block-B `[u16 len]` offset: `[u16 count][count × (u16 block, u16 flags,
  u16 assignedObjects)]`. Assert `triggers()` returns the entries in order with the right
  `(block, reasonFlags, scriptOffset)`.
- [ ] **Step 2:** Implement: read the block-B offset (`u16LE@0`), read `[u16 len]` at it → block-C
  offset, then `[u16 count][count × 6-byte]` (per §F). Named constants + comments.
- [ ] **Step 3 (real IT):** `Eob2InfScript.parse(assumeReadable("LEVEL1.INF")).triggers()`: `size()==64`;
  the idx-4 entry `(block=0x0146, reasonFlags=0x0008, scriptOffset=152)` and idx-14
  `(0x00B4, 0x0008, 673)` are present (§F). Run (data present).
- [ ] **Step 4:** `mvn -q -pl beholder-importer-westwood -am verify`; commit.

### Task 2: `Eob2InfScript` block-B script blob + string pool

**Files:** Modify `Eob2InfScript.java`; Test: extend `Eob2InfScriptTest`/`Eob2InfScriptIT`.

**Interfaces produced:** `Eob2InfScript.rawScript() -> byte[]` (the INF bytecode blob, defensive
clone), `scriptBaseOffset() -> int` (the base such that a trigger's `scriptOffset` indexes
`rawScript()`), `getString(int index) -> String` (resolves the block-B string pool; `0xFFFF` →
empty).

- [ ] **Step 1 (RED synthetic):** a buffer whose block-B region is `[u16 len]` + a `0xEC`-gated
  monster block + a bytecode blob whose first `u16` = the pool offset + NUL-separated strings.
  Assert `rawScript()` is the blob, a trigger `scriptOffset` indexes it correctly, and
  `getString(k)` returns the k-th string.
- [ ] **Step 2:** Implement per §F: from the block-B offset, skip `[u16 len]`, the `0xEC` monster
  block (the interval-pair list terminated by `0xFF`, then `30 × 14` records), to the bytecode blob;
  `rawScript()` = the blob `[bytecodeStart, blockCOff)`; the string pool = the blob's `u16LE@0`
  offset; `getString` walks NUL-separated strings. Pin `scriptBaseOffset` so idx-4's `scriptOffset
  152` indexes `FD 06 01 F1`.
- [ ] **Step 3 (real IT):** `getString(12)` contains "secret passage", `getString(31)` == "you can't
  go that way." (§F); the byte at `rawScript()[trigger.scriptOffset()]` for idx-4 is `0xFD`
  (openDoor). Run.
- [ ] **Step 4:** verify; commit.

### Task 3: `Eob2InfScriptOpcodes` + `Eob2InfActionLifter` (opcode → shared `Action`)

**Files:** Create `.../westwood/Eob2InfScriptOpcodes.java`, `.../westwood/Eob2InfActionLifter.java`;
Test: `.../Eob2InfActionLifterTest.java` (synthetic).

**Interfaces:** Consumes `InfDecode.readU8`/`readU16`/`toPosition` (**reused verbatim** — EOB2 block
packing is the same 32-wide grid). Produces `Eob2InfActionLifter.lift(byte[] rawScript, int cursor,
Eob2InfScript script) -> ActionResult(List<Action> actions, int nextIndex)` (the `script` is needed
for `getString` on `printMessage_v2`).

- [ ] **Step 1 (RED):** synthetic byte sequences → expected `Action`s per §F: `FD 06 01` →
  `[OpenDoor((6,8))]`; `FF F7 B0 00 00` (setWallType sel `-9` all-sides, blk `0x00B0`, val 0) →
  `[SetWall((16,5),0,0), SetWall((16,5),1,0), SetWall((16,5),2,0), SetWall((16,5),3,0)]`;
  `FF E9 …` (sel `-23` per-side) → one `SetWall`; a `printMessage_v2` (`F8 <u16 idx> <col> <pad>`)
  with a stub `getString` → `[ShowMessage(text, col)]`; `F7 EF 07` (setFlags sel `-17` level, id 7) →
  `[SetFlag(FlagRef(LEVEL, 7))]`; an unmodeled opcode (e.g. `F6` playSound) → **throws**
  `IllegalArgumentException` (so the trigger skips).
- [ ] **Step 2:** Implement `Eob2InfScriptOpcodes` (the §F opcode + selector constants, each named +
  commented) + `Eob2InfActionLifter` (dispatch on the opcode/selector; all-sides expands to 4
  `SetWall`; `printMessage_v2` resolves `script.getString(idx)`; setFlags/removeFlags → SetFlag/
  ClearFlag with `FlagRef.Scope` from the selector). Small per-opcode helpers (cyclomatic gate). No
  `instanceof`.
- [ ] **Step 3:** verify; commit.

### Task 4: `Eob2InfConditionLifter` (eval_v2 RPN → shared `Condition`)

**Files:** Create `.../westwood/Eob2InfConditionLifter.java`; Test: `.../Eob2InfConditionLifterTest.java`.

**Interfaces:** Produces `Eob2InfConditionLifter.lift(byte[] rawScript, int evalOffset) ->
ConditionResult(Condition condition, int nextIndex, int falseTarget)` (parallel to
`InfConditionLifter`; the trailing `u16` is the **jump-when-FALSE** target).

- [ ] **Step 1 (RED synthetic — this proves the RPN competency, since LEVEL1's clean RPN triggers
  skip on `playSound`):** the RPN stream for `levelFlag(7) == 0` — value ops `-17`(`0xEF`) `[i8 7]`,
  `-46`(`0xD2`) PUSH `[i16 0]`, operator `==` (cmd `-1`), terminator `-18`(`0xEE`), `[u16
  elseTarget]` — → `Comparison(EQ, FlagValue(FlagRef(LEVEL,7)), Literal(0))`, `falseTarget` = the
  u16. Also a `BoolOp(AND, …)` case and a `TriggerReasonValue` (`-32` scriptFlags) case.
- [ ] **Step 2:** Implement the RPN stack machine per §F (`switch(cmd+50)` equivalent, a `Deque<
  Condition>`; value pushers → `Literal`/`FlagValue`/`WallIndexAt`/`TriggerReasonValue`; operators
  cmd `-8..-1` → `BoolOp`/`Comparison` with the §F pop order `a`=rhs; `-18` terminates then reads the
  u16 false-target). No `instanceof`.
- [ ] **Step 3:** verify; commit.

### Task 5: `Eob2InfHandlerLifter` + `Eob2InfTriggerBinder` + `Eob2InfLifter` (assemble → `LevelEvents`)

**Files:** Create `.../westwood/Eob2InfHandlerLifter.java`, `.../westwood/Eob2InfTriggerBinder.java`,
`.../westwood/Eob2InfLifter.java`, `.../westwood/Eob2InfLiftResult.java` (record); Test:
`.../Eob2InfLifterTest.java` (synthetic) + `.../Eob2InfLifterIT.java` (real LEVEL1).

**Interfaces:** Consumes Tasks 3/4. Produces: `Eob2InfHandlerLifter(byte[] rawScript, int
scriptBaseOffset, Eob2InfScript script).lift(int scriptOffset) -> Handler` (CFG: jump/call-inline/
conditional-via-eval/`end`/linear-actions → `Handler`; a conditional builds `Handler.Guard(condition,
thenSteps)`; fails closed on back-edges/non-terminating branches, mirroring `InfHandlerLifter`);
`Eob2InfTriggerBinder.bind(Eob2InfTrigger, Eob2InfHandlerLifter) -> TriggerBinding` (reason decode:
`subMask = ((flags & 0xFFF8) >> 3) | 0xE0`, mapping flag bit `1<<(3+k)` → the shared `Reason` for bit
`1<<k`; EOB2-only reason bits without a shared `Reason` — 0x80 timer / 0x100+ — are **ignored**;
block via `InfDecode.toPosition`); `Eob2InfLifter.lift(Eob2InfScript) -> LevelEvents` +
`liftReport(...) -> Eob2InfLiftResult(LevelEvents, List<Skipped>)` (lenient: a trigger throwing
`IllegalArgumentException` — e.g. an unmodeled opcode — is skipped, not fatal).

- [ ] **Step 1 (RED synthetic):** a synthetic script with (a) a linear `openDoor;end` → `Handler([
  OpenDoor])`; (b) an all-sides setWall → `Handler([4× SetWall])`; (c) an `eval → then → end`
  conditional → `Handler([Guard(condition, thenSteps)])` (using Task 4); (d) an unmodeled-opcode
  trigger → skipped. Assert the bound `TriggerBinding` (block, reasons, handler).
- [ ] **Step 2:** Implement the three classes mirroring `InfHandlerLifter`/`InfTriggerBinder`/
  `InfLifter` with EOB2 tokens (jump `0xF2`, call `0xEF`, eval `0xEE`, end `0xF1` per §F). Keep the
  handler decomposed (per-token helpers) under the gates.
- [ ] **Step 3 (real IT — the lift proof, `InfLifterIT` template):** `Eob2InfLifter.lift(
  Eob2InfScript.parse(LEVEL1.INF))`: the binding at block `(6,10)` equals
  `new TriggerBinding(GridPosition.of(6,10), Set.of(Reason.STEP_ONTO), new Handler(List.of(new
  Action.OpenDoor(GridPosition.of(6,8)))))` (idx 4); the binding at `(20,5)` has a `Handler` of 4×
  `SetWall((16,5), 0..3, 0)` (idx 14); `liftReport(...)` records the skips (idx 34 etc. with unmodeled
  opcodes) so `lifted + skipped == 64`. Run.
- [ ] **Step 4:** verify; commit.

### Task 6: `WallMappingTable` value type + `LevelDocument.wallMappings` + serde (D-B)

**Files:** Create `beholder-ir/.../ir/model/WallMappingTable.java`; Modify `.../ir/model/
LevelDocument.java`, `beholder-ir-json/.../serde/LevelDocumentSerializer.java`/`…Deserializer.java`,
`beholder-ir-json/src/main/resources/schema/level-document-1.json`; Test:
`.../model/WallMappingTableTest.java`, extend the ir-json round-trip + back-compat tests.

**Interfaces:** `WallMappingTable.of(int[] wallSetByIndex/*len 256*/) -> WallMappingTable`,
`wallSetForIndex(int index/*0..255*/) -> int`, `empty() -> WallMappingTable` (all-zero), value
`equals`/`hashCode` (array behind a scalar accessor — the `LevelGeometry`/`TileSet` not-a-record
pattern, dodging `EI_EXPOSE_REP`); `LevelDocument` gains a `WallMappingTable wallMappings` component.

- [ ] **Step 1:** `WallMappingTableTest` (RED→GREEN) — `of(int[256])` guards length; `wallSetForIndex`
  returns the entry; out-of-range throws; two equal tables `.equals`. Implement.
- [ ] **Step 2:** Add `wallMappings` to `LevelDocument` (guard non-null; its own value equality). Fix
  the 2 existing producers (`Eob1Extractor`/`Eob2Extractor` — Task 7 populates; here pass
  `WallMappingTable.empty()` to keep them compiling) and any `LevelDocument` test.
- [ ] **Step 3 (serde, additive/optional — back-compat):** serializer writes `wallMappings` (a JSON
  array of 256 ints) as a new field; deserializer adds `"wallMappings"` to `Nodes.requireOnlyFields`
  and reads it, **defaulting to `WallMappingTable.empty()` when the field is absent** (so M2's IR
  loads); schema adds `wallMappings` under `properties` (NOT `required`). Test: a document with a
  populated table round-trips byte-stably + value-equal; a JSON **without** `wallMappings` (M2's
  shape) still loads (→ empty table).
- [ ] **Step 4:** `mvn -q -pl beholder-ir-json -am verify`; commit.

### Task 7: Extractors populate the table + wire triggers; `Boot` builds the resolver (D-B/D-C)

**Files:** Modify `beholder-app/.../app/Eob1Extractor.java`, `Eob2Extractor.java`, `Boot.java`; Test:
extend the extractor unit tests + `Eob1ExtractIT`/`Eob2ExtractIT`.

**Interfaces:** Consumes `Eob2InfLifter.lift` (Task 5), `WallMappingTable` (Task 6),
`InfWallGeometry.wallType(script, i)` (EOB1, public).

- [ ] **Step 1:** `Eob1Extractor` — replace `new LevelEvents(List.of(), doors, decorations)` with
  `new LevelEvents(InfLifter.lift(script).triggers(), doors, decorations)` (**restore EOB1's real
  triggers** — undoing M2's base-EOB1), and build `WallMappingTable.of(table)` where
  `table[i] = InfWallGeometry.wallType(script, i)` for `i` 0..255; pass it into the `LevelDocument`.
- [ ] **Step 2:** `Eob2Extractor` — `new LevelEvents(Eob2InfLifter.lift(script).triggers(), doors,
  decorations)`, and `table[i] = script.hasWallMapping(i) ? script.wallMapping(i).vmpIndex() : 0`.
- [ ] **Step 3:** `Boot` — replace `IDENTITY_WALL_RESOLVER` with `i -> doc.wallMappings()
  .wallSetForIndex(i)` (pure IR — **no importer import**; the one-IR boundary holds).
- [ ] **Step 4 (tests):** the extractor synthetic-fixture unit tests now assert **non-empty
  triggers + a populated table**; the real `Eob1ExtractIT`/`Eob2ExtractIT` assert triggers non-empty
  + the table has the expected entries (EOB2: `wallSetForIndex` matches `Eob2InfScript.wallMapping`
  for a spot index). `mvn -q verify`; commit.

### Task 8: `RuntimeState.initialDoorState` edge fix (D-E)

**Files:** Modify `beholder-engine-core/.../engine/RuntimeState.java` (thread `edge` through
`doorState`), and the callers of the `DoorClosedTest` seam (`beholder-render-core/.../
IrDoorLeafScene.java` + `Boot`/the overlay wiring that supplies the closed-test); Test: extend
`RuntimeStateTest`/`RuntimeStateDoorCollisionTest`.

**Interfaces:** `RuntimeState.doorState(GridPosition block, int edge, Level level) -> Door.DoorState`
(edge added); `initialDoorState(block, edge, level)` matches `door.block().equals(block) &&
door.edge() == edge`. The `DoorClosedTest` seam becomes `(GridPosition block, int edge) -> boolean`;
`IrDoorLeafScene` passes the edge it renders.

- [ ] **Step 1 (RED):** a `Level` with a two-door block (edges 0 and 2), one door OPEN'd via a
  runtime override on edge 0 only; assert `doorState(block, 0, level) == OPEN` and `doorState(block,
  2, level) == CLOSED` (today's edge-blind code returns the same for both). Also a single-door block
  unchanged.
- [ ] **Step 2:** Thread `edge` through `doorState`/`initialDoorState`; update the `DoorClosedTest`
  seam + `IrDoorLeafScene` + the `Boot`/`SceneOverlay` closed-test lambda (`(b, e) -> state.doorState(
  b, e, level) == CLOSED`) and any test double. Keep one `return`/method.
- [ ] **Step 3:** `mvn -q verify` (root — this ripples across engine + render + app); commit.

### Task 9: End-to-end EOB2 trigger IT + the EOB1 render-identity update (the milestone end state)

**Files:** Create `beholder-app/.../app/Eob2TriggerIT.java` (real EOB2, skip-safe); Modify
`.../app/Eob1BootWalkIdentityIT.java` (EOB1 now has triggers).

- [ ] **Step 1 (`Eob2TriggerIT`, real EOB2, skip-safe):** `Extract.extract(eob2Dir, "LEVEL1", tmpIr,
  "level1")` → `bg = Boot.boot(tmpIr, "level1", <start near (20,5)>, <facing>)`. **The SET_WALL demo
  (idx 14):** move the party onto block `(20,5)` (STEP_ONTO auto-fires) → assert the four edges of
  block `(16,5)` now resolve **open** — `bg.state().wallOverrideRaw((16,5), side)` present == 0 for
  each side, and `bg.state().wallOverride((16,5), side, level)` == the resolver's value for raw 0
  (open). **The door demo (idx 4):** move onto `(6,10)` → `bg.state().doorState((6,8), <edge>,
  level) == Door.DoorState.OPEN`. **Render the SET_WALL change through boot:** render the view of
  `(16,5)` before/after via a `GameSession(...).tick()` + `RecordingFrameSink`; assert the frame
  changed (the wall is gone) — proving the IR-carried resolver renders a fired `SET_WALL` game-
  agnostically. Write the before/after PNGs to git-ignored `target/` for the maintainer eyeball. Run
  (data present).
- [ ] **Step 2 (`Eob1BootWalkIdentityIT` update):** EOB1's extract now emits real triggers, so the
  in-process reference build in the render-identity test must also use `InfLifter.lift(script)
  .triggers()` (not `List.of()`); the entry frame is trigger-independent so the pixel-identity still
  holds — update + confirm it passes.
- [ ] **Step 3:** `mvn -q verify` (root); commit.

## Self-review (writing-plans)

- **Spec coverage:** the Eob2 lift chain (block B/C parse → opcode/RPN/CFG lift → LevelEvents) →
  Tasks 1–5; the IR wall-mapping-table + game-agnostic boot resolver (§7 carry-forward) → Tasks 6–7;
  both games get triggers (D-C, undoing the M2 base-EOB1 regression) → Task 7; the `initialDoorState`
  edge fix (carry-forward) → Task 8; one real EOB2 trigger fires end-to-end + renders (the demo) →
  Task 9. All design sections covered.
- **Two importers, one IR / no VM / zero-vocab-delta:** the lift emits only shared `LevelEvents`/
  `TriggerBinding`/`Action`/`Condition`/`Reason` (§F: the minimal set maps to existing vocabulary —
  no new enum members); unmodeled opcodes skip leniently (never interpreted). The `Eob2*` lifters
  are parallel to the `Inf*` chain; `InfDecode` is the one reused (shared block packing). ✔
- **Coverage:** every new lifter/parse class has synthetic-fixture UNIT tests (Tasks 1–5); the
  condition lifter (Task 4) is proven by synthetic RPN since LEVEL1's clean RPN triggers skip. ✔
- **Type consistency:** `Eob2InfTrigger(block, reasonFlags, scriptOffset)`; `Eob2InfScript.{triggers,
  rawScript, scriptBaseOffset, getString}`; `Eob2InfActionLifter.lift(rawScript, cursor, script) ->
  ActionResult`; `Eob2InfConditionLifter.lift(rawScript, evalOffset) -> ConditionResult`;
  `Eob2InfHandlerLifter.lift(scriptOffset) -> Handler`; `Eob2InfLifter.lift(script) -> LevelEvents`;
  `WallMappingTable.of/wallSetForIndex/empty`; `doorState(block, edge, level)` — used consistently.
- **Carry-forwards from M2 both closed:** the resolver leak (Tasks 6–7) and `initialDoorState`
  (Task 8).
