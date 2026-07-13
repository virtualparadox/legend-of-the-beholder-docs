# P2 · M3 — EOB2 `.INF` event lift — Design

> **Status:** Design (awaiting maintainer review before the implementation plan
> is written). Third milestone of Phase 2 (PHASES §"Phase 2 — EOB2").
> **Date:** 2026-07-13
> **Grounding:** MASTER §4.1 (data-vs-behaviour — triggers are *lifted* into a
> declarative `event → condition → action`, a closed vocabulary, **no embedded
> VM**), §5 (the event/trigger model is the IR's stress test), §8 (differential
> correctness), §10 (schema grows from real games; two importers, one IR);
> **ADR-0002** (the closed event vocabulary); the P2 design
> (`2026-07-09-p2-eob2-design.md`), M1 (merged `e8c1a4a`), and M2 (merged
> `13db07f`, whose design §7/§10 flagged the two carry-forwards this milestone
> closes). **Template:** the P1 M4 EOB1 lift (`InfLifter` + its action/condition/
> handler lifters + `InfTriggerBinder`; `InfScript.triggers()`/`rawScript()`).
> **Format facts:** the M1 recon confirmed EOB2 uses the *same* imperative
> script-VM as EOB1 (negative-byte opcodes, a 10-deep call stack, an RPN condition
> evaluator, an 18×`u32` flag table), widened to 30 opcodes; block B = the script
> bytecode + a null-separated string pool; block C = the block→script binding
> table (`u16 block`, `u16 flags`, `u16 assignedObjects`).

---

## 1. Goal

Lift the EOB2 `.INF` trigger bytecode (block B/C) into the **shared** event graph
**at import time** — a control-flow-reconstructed `event → condition → action`, not
an interpreter — and fire **one real EOB2 trigger end-to-end** through the M2
extract/boot pipeline (so `LevelEvents.triggers` is non-empty for the first time on
EOB2). This is Phase 2's research core (MASTER §5): the event vocabulary's stress
test on a second Westwood game — does the *same* closed `Action`/`Condition`/
`Reason` vocabulary + the *same* engine executor absorb EOB2's script without a
fork?

The milestone also **closes the two carry-forwards** M2's boundary surfaced:
1. **The `SceneOverlay` wall-mapping resolver leak** (M2 design §7): the IR carries
   a per-level wall-mapping table (wall-index → renderable wall-set) so `boot`
   builds a *game-agnostic* resolver — a `SET_WALL` trigger's wall change then
   renders through the booted engine (both games), replacing M2's inert identity
   resolver.
2. **`RuntimeState.initialDoorState`'s edge-blind lookup** (M2 §10): fixed before a
   trigger can open one door on a multi-door block.

## 2. Scope

**In scope:**
- **`Eob2InfScript` block B/C parsing:** `rawScript()`/`scriptBaseOffset()` (the
  block-B bytecode + string pool) and `triggers() -> List<Eob2InfTrigger>` (the
  block-C binding table). Mirrors `InfScript`'s script-side surface.
- **`Eob2InfLifter`:** the EOB2 opcode/RPN → shared `LevelEvents` lift, mirroring
  `InfLifter` + its action/condition/handler lifters + `InfTriggerBinder`.
- **One real EOB2 LEVEL1 trigger fired end-to-end** through `extract → IR → boot →`
  the shared engine executor.
- **The IR wall-mapping-table extension + the boot resolver** (closes carry-forward
  1), populated by *both* extractors.
- **The `initialDoorState` edge fix** (closes carry-forward 2).
- **Shared-vocabulary growth only where a real LEVEL1 trigger forces it.**

**Out of scope (deferred):**
- The full EOB2 opcode vocabulary — only the minimal set the one demo trigger needs
  (P1 M4 parallel).
- Decomposing the opaque cutscene opcodes (`dialogue`/`specialEvent`/`drawScene`) —
  they become **opaque named-cutscene actions** (PHASES §3 item 3), not decomposed.
- Monsters, items, combat, AD&D spell resolution; runtime/save-state serialization;
  multi-level / inter-sub-level movement; the M4 acceptance gates (byte-stable
  re-serialize + diff vs `kyra`).

## 3. Architecture

### 3.1 The parallel `Eob2` lift chain (two importers, one IR)
`Eob2InfScript` grows a **block-B/C parse** alongside its M1 block-A geometry parse:
- `rawScript() -> byte[]` + `scriptBaseOffset() -> int` (the block-B bytecode +
  where the script proper begins after the string-pool pointer), and the
  string-pool accessor (`getString(int index)`), mirroring EOB1's `EoBInfProcessor`
  string table.
- `triggers() -> List<Eob2InfTrigger>` — `Eob2InfTrigger(int block, int reasonFlags,
  int scriptOffset)` decoded from block C (`u16 block`, `u16 flags` — EOB2's wider
  mask, `u16 assignedObjects` = the script offset). Parallel to `InfTrigger`.

`Eob2InfLifter.lift(Eob2InfScript) -> LevelEvents` mirrors `InfLifter`:
control-flow reconstruction (the CFG from jumps/call-stack) → a side-effect-free
`Condition` AST from the RPN sub-stream → a `Handler` of `Action`s, bound to a
`TriggerBinding` per block via the `reasonFlags → Set<Reason>` decode. **The lift
competency is reused** (EOB2 is the *same* VM — CFG reconstruction, RPN→AST, the
call-stack model all carry over), but the **opcode map is EOB2-specific** (30
opcodes, the EOB2-only markers/operand layouts). A shared Westwood-`.INF`-lift
abstraction is extracted **only where real duplication forces it** (MASTER §10 — now
the second game legitimises it, but from two concrete lifters, never top-down).

### 3.2 The IR wall-mapping-table + the game-agnostic boot resolver
The `SceneOverlay.wallMappingResolver` (an `IntUnaryOperator`: raw wall-index →
renderable wall-set) is today importer-backed — `InfWallGeometry.wallType(script,
i)` for EOB1 — which `boot` cannot call (it holds no script). M3 makes it
IR-carried:
- The IR gains a **per-level wall-mapping table** — a 256-entry `wall-index →
  wall-set` map. **Home: leaning `LevelGeometry`** (it is the geometry/wall-set
  resolver already; it would expose `wallSetForIndex(int) -> int`). *(LevelGeometry
  vs a new `LevelDocument` field vs the appearance pack is a plan-level call; the
  commitment is: IR-carried, per-level, populated by both extractors, consumed by
  boot.)*
- **Both extractors populate it:** EOB1 from `InfWallGeometry.wallType`/the
  `InfScript` wall-mapping table; EOB2 from `Eob2InfScript.wallMapping(i).vmpIndex()`.
- **`boot` builds the resolver from the IR table** (`i -> table.wallSetForIndex(i)`)
  and passes it into the `SceneOverlay`, replacing M2's `IntUnaryOperator.identity()`.
  So a `SET_WALL` override installed by a fired trigger renders correctly through the
  booted engine — the leak is closed, game-agnostically.
- **Versioned, additive IR extension** (ADR-0001): a new field on the geometry/
  document, back-compatible (older IR without it falls back to identity, or the
  loader defaults it), so M2's already-written IR shape is not broken.

### 3.3 Vocabulary-extension policy (ADR-0002, MASTER §4.1)
Extend the **shared** closed `Action`/`Condition`/`Reason` enums **only** where a
real LEVEL1 trigger forces it, shaped for the family (never an EOB2 silo):
- **Messages:** EOB2's string-table `printMessage` → the lift **dereferences the
  index** at import time → the existing `Action.ShowMessage(text, colour)` (no IR
  change — the same importer-resolves-it finding M1 made for decorations).
- **Opaque cutscenes:** `dialogue`/`specialEvent`/`drawScene` → an **opaque
  named-cutscene `Action`** (e.g. `NamedCutscene(String name)`) — added to the
  shared vocabulary *iff* the demo trigger's block references one; not decomposed.
- **New trigger reasons / conditions / actions:** added only if the chosen real
  trigger's opcodes/RPN demand them. The exact delta is plan-level (recon-decided).

## 4. The demo / differential evidence (MASTER §8)
One real EOB2 LEVEL1 trigger fires end-to-end: `extract` (now emitting non-empty
`LevelEvents.triggers` **and** the wall-mapping table) → `IrJson` round-trip →
`boot` → the shared `EventExecutor` fires it → the effect is observed. If the effect
is a `SET_WALL`, the changed wall **renders** through the booted engine (via the new
resolver). **Differential:** the trigger's effect matches `kyra`'s script behaviour
for that block (the disassembly cross-checked against the read-only oracle; pixel-
exact `kyra` golden stays deferred — no built ScummVM). **P1 M4 precedent:** if
LEVEL1's real triggers all need RPN outside the minimal opcode set (as EOB1's real
door triggers did — item-count RPN), a documented **half-synthetic** demo trigger
bridges, exactly the idiom P1 M4 established. The `initialDoorState` edge fix is
verified by a two-door-block test.

## 5. Global constraints (carried)
- **JDK 21**; `mvn spotless:apply` then `mvn verify`; `-pl <m> -am` or root verify.
- **All `CODING_STANDARDS` gates green** (one `return`/method, no `instanceof`,
  cyclomatic ≤ 8, ≥ 0.90 branch+line coverage, NullAway, full Javadoc, every magic
  value a named constant + comment). New lift/parse logic is **not** coverage-exempt
  and is branch-covered by **synthetic fixtures** (the M2/M3 fact: `jacoco:check`
  counts Surefire only; skip-safe ITs do not — so the lifter needs data-independent
  unit coverage, e.g. a synthetic uncompressed-CPS block-B/C buffer, mirroring M1's
  `Eob2InfScriptTest`).
- **Two importers, one IR:** every EOB2 construct binds to the shared `beholder-ir`
  types; vocabulary extensions grow the shared enums. **No embedded VM** — a miss is
  a signal to extend the vocabulary, never to interpret (MASTER §4.1).
- **Clean-room** (kyra read-only oracle); **never commit game data**.
- **Process:** `writing-plans` → `subagent-driven-development`, TDD, whole-branch
  review, `--no-ff` merge on the maintainer's verdict; conventional branches.

## 6. Open questions (plan-level, recon-decided)
- **Which real LEVEL1 trigger + opcode subset** to lift (the block-C entry whose
  script lifts cleanly under a minimal vocabulary); whether a half-synthetic bridge
  is needed (P1 M4 precedent).
- **The exact `Action`/`Condition`/`Reason` delta** forced by that trigger.
- **The IR wall-mapping-table's home** (`LevelGeometry` — leaning — vs a
  `LevelDocument` field vs the appearance pack) and its serde shape.
- The block-B string-pool + block-C offset arithmetic (byte-level, from the kyra
  oracle + the real LEVEL1 decompressed buffer — the M1 "format facts" rhythm).

## 7. Checkpoint (MASTER §12)
Maintainer review of: the fired EOB2 trigger's effect (state + render) vs the kyra
oracle; the shape of any shared-vocabulary extension (does it read as family-
coherent, not an EOB2 patch?); the IR wall-mapping-table extension (is boot now
genuinely game-agnostic for `SET_WALL`?); and the two closed carry-forwards. On
approval, the IR's **event model is proven against a second game** — the last piece
before M4 (round-trip + kyra diff) closes Phase 2, and a real down-payment on the
Phase 3 AESOP lift (the project's primary risk), which stresses the same event
substrate hardest.
