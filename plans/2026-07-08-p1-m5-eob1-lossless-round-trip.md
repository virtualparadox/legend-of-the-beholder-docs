# P1 · M5 — EOB1 lossless round-trip — Design

> **Status:** Design (awaiting maintainer review before the implementation plan
> is written). Closes Phase 1 (the EOB1 vertical slice).
> **Date:** 2026-07-08
> **Grounding:** MASTER §2 (the IR is the product), §5 (versioned, fail-closed,
> closed vocabulary), §8 (differential correctness / goldens), §10 (schema grows
> from real games, editor last); **ADR-0001** (IR serialization format —
> *Accepted*: JSON / Jackson / JSON Schema / deterministic writer / fail-closed
> loader); ADR-0002 (event model, whose closed vocabulary the JSON must express).
> **Carries forward:** the M4 whole-branch review's **M5 must-fix** — decorations
> currently bypass the IR and a round-trip would silently drop them.

---

## 1. Goal

Make the IR a real, persisted product. Prove the canonical on-disk form is
**lossless** and **byte-stable**:

```
extract → IR_a → canonical JSON → (fail-closed reload) → IR_b
  such that  IR_a == IR_b,  re-serialize(IR_b) is byte-identical,
  and IR_b renders/plays identically to IR_a.
```

This is the Phase-1 closer and the MASTER §8 acceptance test for the whole EOB1
slice. It also forces the decoration-IR fix M4 deferred.

## 2. The output artifact (maintainer-shaped)

An IR document is **a canonical JSON file plus a sibling folder of binary
assets**, the JSON referencing each asset by a stable ID:

```
<level-or-campaign>.json          # the behaviour IR (this milestone)
<level-or-campaign>-assets/       # binaries, referenced by ID from the JSON
    tileset/  vcn-atlas … vmp … palette …
    doors/    DOOR.CPS-derived leaf sheet …
    decorations/  overlay .cps + .dat rectangle tables …
```

**Forward-looking principle (not M5 scope).** This folder is the growth point for
*all* future game media — monster sprites, item sprites, intro/cutscene images,
music. The **reference mechanism** (JSON → asset-by-ID → folder) is designed here
to hold any of them without change; M5 populates only what an EOB1 *level* needs
(tileset / decoration / door binaries). Extracting the full media set is later
phases — **YAGNI / MASTER §10** (no speculative asset schema ahead of a game
needing it). For the real-level round-trip the asset folder is generated into
git-ignored `target/` (extracted from the user's own PAKs — MASTER §9 — never
committed).

## 3. Confirmed decisions (maintainer)

1. **Serialization scope = the full behaviour IR.** Geometry (32×32 grid, cells,
   4 wall archetypes + wall-set handles), **decorations**, the event graph
   (`event → condition → action`, ADR-0002), doors, and the **base** level state.
   The **appearance/tileset pack is referenced by ID**, its pixel data living in
   the asset folder — *not* inlined into the JSON.
2. **Round-trip proof + golden.** (a) a real-`LEVEL1` **test-time** round-trip
   (skip-safe): `import → IR_a → JSON → fail-closed reload → IR_b`, asserting
   `IR_a.equals(IR_b)` (record value-equality) **+** byte-stable re-serialize **+**
   the two IRs **render identically** (the M2/M3 renderer, pixel-equality) — no
   real content committed; **(b)** a committed **canonical-JSON golden of a small
   hand-authored synthetic level** (our own content, fully diffable) as the
   human-readable format ratchet and the fail-closed-loader negative-test fixture.
3. **Decorations become first-class IR** (the must-fix — §5 below).
4. **JSON Schema is generated-from / checked-against the records in CI** (not
   hand-authored in parallel — resolves ADR-0001's open "authored vs generated"
   question), so the two artefacts cannot drift.

## 4. Architecture

### 4.1 A dedicated JSON-codec module — keep the engine format-free
ADR-0001 centralises the loader in "the `ir` module"; to honour MASTER §7.3 (no
serialization format leaking into the runtime) we split it: the **records + enums
stay in `beholder-ir`** (framework-neutral values, at most `jackson-annotations`
— a light Apache-2.0 annotation-only jar, no `databind`), and a **new codec
module `beholder-ir-json`** holds `jackson-databind`, the JSON Schema, the single
pinned canonical `ObjectWriter`, and the fail-closed load pipeline. `engine-core`
depends on `beholder-ir` only and never transitively pulls `databind`. *(Module
name/placement is a plan-level detail; the invariant — the engine stays
Jackson-free — is the design commitment.)*

### 4.2 The fail-closed load pipeline (ADR-0001 §5, mechanised)
`irVersion` semver major-gate → structural parse (Jackson,
`FAIL_ON_UNKNOWN_PROPERTIES=true`) → JSON-Schema validation
(`additionalProperties:false`, hard enum-failure) → bind to records → semantic /
referential validation (every wall-set handle exists, every `Call`-target action
sequence is defined, every decoration/door reference resolves). Aborts and
**rejects** on first failure — never a partial load (MASTER §7.1). Mods and
hand-edited JSON are untrusted input.

### 4.3 Deterministic writer
A single pinned `ObjectWriter`: sorted keys, stable record property order,
`INDENT_OUTPUT`, pinned pretty-printer, LF + trailing newline. The **only**
sanctioned emitter, so the same IR always serializes byte-identically (makes the
round-trip testable and diffs minimal — ADR-0001 §4).

## 5. The decoration-IR fix (the P1-closing key)

Today decorations flow importer `DecorationDat` → composition-root bridge →
render-core, never entering the IR (unlike doors, which produce `List<Door>`).
M5 wires them in as **first-class, lossless IR**:

- **IR `Decoration` = placement + a decoration-graphic handle** — the decorated
  wall-mapping index (or block+edge) and a `decorationId` referencing the
  appearance pack's decoration table. It carries *behaviour/placement*, never
  pixel data; the `.cps`/`.dat` rectangle chain lives in the referenced asset
  folder, keyed by `decorationId` (MASTER §3 — appearance is a swappable pack).
- **The importer produces `List<Decoration>`** into `LevelEvents`/the level IR.
- **The appearance pack gains a decoration table** (`decorationId` → rectangle
  chain) and the render consumes the **IR `Decoration` + the pack table** instead
  of the raw `DecorationDat`. This removes the "orphaned IR type" and makes the
  round-trip carry decorations losslessly.

*(Exact record shape — whether placement is per-wall-index or per-edge — is a
plan-level design choice; the commitment is: lossless, placement-plus-handle,
appearance-by-reference.)*

## 6. Differential evidence (MASTER §8)

- **Real `LEVEL1` round-trip IT** (skip-safe, real data, PNGs/asset-folder →
  git-ignored `target/`): `IR_a == IR_b`, byte-stable re-serialize, render-identity
  (pixel-equal frames from `IR_a` and `IR_b`).
- **Committed synthetic-level golden**: a small hand-authored level's canonical
  JSON, checked byte-for-byte, exercising every IR node kind at least once; plus
  **negative** fixtures (unknown enum, extra property, missing required field,
  unresolved reference, wrong `irVersion` major) each asserted to be **rejected**.
- No committed game assets; the real-level artifacts stay in `target/`.

## 7. Task breakdown (sketch — the plan refines)

1. **The JSON codec core** (`beholder-ir-json`): Jackson binding on the existing
   records + the pinned canonical `ObjectWriter` + the versioned fail-closed load
   pipeline + the record-derived JSON Schema (+ its CI drift-check).
2. **Decoration-IR redesign**: the lossless `Decoration` record; importer emits
   `List<Decoration>`; the appearance pack's decoration table; the render reads the
   IR `Decoration` (retire the raw-`DecorationDat` render path).
3. **Cover the whole behaviour IR** with the codec — geometry, walls, decorations,
   events/conditions/actions, doors, base state — every node a bound record with
   schema coverage; the asset-folder reference mechanism (JSON `assetId` → folder).
4. **The round-trip harness + evidence**: the real-`LEVEL1` round-trip IT
   (equality + byte-stable + render-identity), the synthetic-level golden, and the
   fail-closed negative fixtures.

## 8. Deferred (documented, out of M5 scope)

- **Runtime / save-state serialization** (the base-vs-runtime split's *runtime*
  half — save games). M5 serializes the **base** level.
- **Appearance-pack (tileset) as JSON** — it is a binary pack in the asset folder,
  referenced by ID, not JSON-inlined.
- **The full media extraction** — monster/item sprites, intro/cutscene images,
  music — the asset folder is *designed* for them but M5 populates only the
  level's needed binaries (YAGNI).
- **Campaign / multi-level packaging** (a folder of levels + shared assets).
- **The editor** (PHASES P5) — the round-trip is its precondition, not its start.

## 8a. Not an EOB1 lock-in — the growth substrate for all four games

M5 persists the **EOB1** behaviour IR, but nothing here freezes the IR to EOB1.
The whole point of the platform (MASTER §2) is one IR that eventually carries
EOB1/2/3 + Dungeon Hack content — grown **per game, never top-down** (§5/§10: an
IR no engine has consumed is an unvalidated, likely-wrong IR). M5 deliberately
lays the machinery that *lets* it grow without a rewrite:

- **Versioned schema** (`irVersion` semver, ADR-0001) — later games bump minor/
  add nodes; the loader gates on major.
- **Closed-but-extensible vocabulary** (§7.2) — a second/third game adds enum
  members (wall archetypes, action vocabulary), never an open property bag, so
  the IR absorbs new constructs without becoming a mush.
- **Asset-folder-by-ID** — EOB1's tileset today; EOB3's monster sprites / music /
  intro images ride the *same* reference mechanism tomorrow (the maintainer's
  "JSON + a folder of binaries" shape).
- **Rules-vs-content split** (§4.1) — content is IR, rules (AD&D tables) are
  engine-side, which is what lets one IR span a shared-rules family.

The hardest validation is still ahead — **EOB3/AESOP** (§6, the project's primary
risk): the ~178 AESOP runtime functions may not all map to clean declarative
actions, and the engine-specific ones are the pressure point that tempts an
embedded VM (resisted per §4.1 — they become named actions / engine services).
Spiked in P0, proven in P3. M5's job is to make the persistence *ready* for that
growth, not to prove it now.

## 9. Checkpoint (MASTER §12)

Human review of: the round-trip evidence (real-level equality/byte-stability/
render-identity + the synthetic golden), the canonical JSON shape (is it the
IR we want as the product?), and the decoration-IR record design. On approval,
Phase 1 (the EOB1 vertical slice) is complete.

---

---

# P1 · M5 — EOB1 lossless round-trip — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> to implement this plan task-by-task (fresh implementer + independent review per task,
> an opus whole-branch review at the end, `--no-ff` merge on the maintainer's eyeball
> verdict; no worktree). TDD per task (RED → GREEN → commit). Steps use checkbox
> (`- [ ]`) syntax. Style matches the M4 plan: the decode/asset/render tasks are
> *discovery against real data + a read-only oracle* (the "Design facts" below + each
> task's differential are the concrete spec, not fully pre-coded); the codec tasks are
> *pure logic* and carry concrete code. Every task obeys the Global Constraints.

**Goal:** Make the IR a persisted product — prove EOB1 LEVEL1's canonical on-disk form
(one JSON file + a sibling asset folder) is **lossless** and **byte-stable**:
`extract → IR_a → JSON+assets → fail-closed reload → IR_b`, with `IR_a.equals(IR_b)`,
byte-identical re-serialize, and `IR_b` renders pixel-identically to `IR_a`. Closes Phase 1.

**Architecture:** A new Jackson-free-engine-preserving codec module `beholder-ir-json`
holds `jackson-databind` + the `networknt` JSON-Schema validator, a single pinned
deterministic `ObjectWriter`, custom (de)serializers (so `beholder-ir` stays
Jackson-annotation-free), a record-derived JSON Schema, and the versioned fail-closed load
pipeline. The **behaviour IR** (geometry, events, doors, decorations, base door state)
serializes to JSON; the **appearance pack** (tileset + decoration + door-leaf pixels)
serializes to a sibling asset folder as **framework-neutral decoded binaries referenced by a
stable id** (never raw Westwood `.cps`/`.dat` — MASTER §7.3 forbids the original format past
the importer). Decorations become first-class IR (the M4 must-fix): the importer emits a
placement `List<Decoration>` and the render consumes IR `Decoration` + the pack, retiring the
composition-root raw-`DecorationDat` reader.

**Tech Stack:** Java 21, Maven multi-module, Jackson (`jackson-databind`/`-core`, custom
serde), `com.networknt:json-schema-validator` (Draft 2020-12), JUnit 5 + AssertJ.

## Global Constraints

- **JDK 21** (`JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home`);
  `mvn spotless:apply` then `mvn verify`; single-module `-pl <m> -am` **or** root verify (the
  stale `~/.m2` upstream-jar trap — see the M3 execution outcome).
- **All `CODING_STANDARDS` gates green:** one `return`/method (`OnlyOneReturn`), no
  `instanceof` (use `getClass()` or polymorphic/visitor dispatch), no `switch yield`,
  cyclomatic ≤ 8 (PMD counts boolean ops), NPath ≤ 120, ≤ 12 fields / ≤ 18 methods / ≤ 5
  effective params (a 6-param list is a violation), coupling ≤ 10, no `return null`
  (Optional/throw/empty-collection), params `final` + not reassigned, braces on all
  control-flow, `L` for longs, ≤ 140-char lines, no `System.out`/`printStackTrace`, NullAway
  `ERROR` for `eu.virtualparadox`, Javadoc `doclint=all` (every public/protected type +
  method, `@param`/`@return`), **every magic value a named `ALL_CAPS` constant + a
  what-and-why comment** (clean-room formats).
- **Coverage: JaCoCo ≥ 0.90 branch AND line at bundle level.** Exempt: `**/*Record.class`,
  `**/model/**`, `**/dto/**`, `**/*Dto.class` (and the render-libgdx `gl/` subpackage's
  existing exclusion for untestable-without-real-assets composition glue). New IR value types
  go under `ir/model/` (exempt) but their **guards + value-equality are behavior-tested**; the
  codec's logic is **not** exempt and must be branch-covered by synthetic fixtures.
- **New dependencies are license-allowlisted (ADR-0001).** `com.fasterxml.jackson.core:*` is
  Apache-2.0 (POM license string `"The Apache Software License, Version 2.0"` — on the
  allowlist); `com.networknt:json-schema-validator` is Apache-2.0 (`"Apache License, Version
  2.0"` — on the allowlist). The reactor license gate runs `aggregate-add-third-party` at the
  root and **fails on any transitive dep with missing/blacklisted license metadata**; Task 1
  verifies the gate is green before any codec code is written (a `licenseMerge` is added only
  if a transitive dep's SPDX-id spelling needs normalising onto an already-allowed family, as
  the root pom already does for lwjgl). Versions are **pinned in the new module's pom** (the
  parent `dependencyManagement` manages neither Jackson nor networknt).
- **Clean-room:** kyra (`~/Workspaces/CPP/scummvm/engines/kyra`) + `eoblib` are **read-only
  oracles** — never copy. jCodeMunch local-only. **Never commit game data:** real-data ITs are
  `assumeTrue`-skip-safe (via the existing `RealLevel1Archives.assumeReadable(...)` helper,
  game dir `~/Games/Eye of the Beholder.app/Contents/Resources/game`, PAKs `EOBDATA3.PAK` +
  `EOBDATA4.PAK`); the real-level JSON + asset folder + PNGs go to **git-ignored `target/`**.
  The synthetic-level golden is **our own hand-authored content** and IS committed.
- **Package roots** `eu.virtualparadox.beholder.{importer.westwood,ir,ir.json,engine,render}`.
- Branch `feat/p1-m5-lossless-round-trip`; commit per task; whole-branch opus review + `--no-ff`
  merge at the end. Base = current `main` (`d5d85f4`).

---

## Plan-level design decisions (the shapes the design §5/§4 deferred to the plan)

These resolve the design's explicit "plan-level design choice" points. They are the spec for
the tasks below.

- **D-A · Decoration placement = per `(block, edge)`, not per wall-mapping-index.** The IR
  `Decoration` carries a `GridPosition block` + an `int edge` (0..3), exactly like `Door`.
  Rationale: the M2/M3 `LevelGeometry` stores a *resolved* per-edge `wallSet` (the VMP look
  category), **not** the raw MAZ wall-mapping index, so an edge cannot be mapped back to a
  wall-mapping index from the IR geometry alone — a per-`(block,edge)` list is therefore the
  render-direct, geometry-untouching, lossless choice the design sanctions ("the decorated
  wall-mapping index *or block+edge*"). The importer expands the source's per-wall-mapping-index
  decoration table across the maze at import time (deterministic → byte-stable).
- **D-B · Decoration handle = `(String overlayId, int decorationId)`.** `overlayId` is the
  `0xEC` overlay's `cpsName` (e.g. `"brick1"`) selecting which decoration shape-sheet to cut
  from; `decorationId` is the `.dat` decoration-chain **start node** (the source's
  `InfWallMapping.decorationId`). Both are needed because real LEVEL1 loads **three** overlays
  (`brick1`/`brick2`/`brick3`) that **share one** `brick.dat` table but cut from three distinct
  `.cps` sheets (`InfOverlay` Javadoc). So the full IR `Decoration` =
  `(GridPosition block, int edge, String overlayId, int decorationId)`.
- **D-C · The appearance pack stores framework-neutral *decoded* data, never raw Westwood
  bytes (MASTER §7.3).** `AppearancePack` = `(String id, TileSet tileSet, DecorationTable
  decorationTable, Map<String,PixelSheet> decorationSheets, PixelSheet doorLeafSheet)`. A
  `PixelSheet` is a decoded CPS = `(int width, int height, byte[] paletteIndices)`. A
  `DecorationTable` is the decoded `.dat` = the decoration records (per record: 10
  `shapeIndex`, `next`, `flags`, 10 `destX`, 10 `destY`) + the rectangle table + the
  `noShape`/`endOfChain` sentinels — i.e. `DecorationDat`'s fields re-homed as a
  framework-neutral IR value. The palette lives once in `tileSet.paletteArgb()`. **No `.cps`/
  `.dat` parser runs past the importer**; the render cuts + palette-resolves + chain-walks from
  these decoded arrays (the same logic the composition-root `Level1DecorationScene`/
  `Level1DoorLeafScene` run today, moved into IR-fed render-core components). The door leaf is
  the decoded `DOOR.CPS` sheet + three fixed distance-band rectangles that stay a **render
  constant** (like `WallSlotRasterizer.SLOT_DRAWS`), not IR.
- **D-D · JSON carries behaviour + an appearance-pack *reference*; the pack goes to the asset
  folder.** The JSON root is a new IR aggregate `LevelDocument = (String irVersion,
  LevelGeometry geometry, LevelEvents events, String appearancePackId)`. The `appearancePackId`
  is the only tie between the JSON and the sibling `<name>-assets/` folder (the design's
  asset-by-ID growth substrate — §2/§8a). `irVersion` is the **top-level** version-gate key
  (semver; reuses `LevelGeometry.CURRENT_IR_VERSION = "1.0.0"`; the loader rejects an unknown
  **major**). `TileSet` pixels are **not** JSON-inlined (design §8 deferral).
- **D-E · `beholder-ir` stays Jackson-free; the codec owns the binding via custom serde.** The
  house IR value types are not canonical-constructor records (`LevelGeometry`/`TileSet`/
  `GridPosition`/`PixelSheet`/`AppearancePack` are final classes with private ctors, `of(...)`
  factories, and array state behind scalar accessors) — Jackson cannot auto-bind them and
  annotations alone can't reach array state. So the codec registers a Jackson `Module` of
  explicit per-type serializers/deserializers; the sealed `Action`/`Condition`/`Handler.Step`
  hierarchies serialize with an explicit `"kind"` discriminator (dispatched via the existing
  visitors — no `instanceof`). This is the strongest form of MASTER §7.3/§4.1: `beholder-ir`
  and `beholder-engine-core` never see Jackson.
- **D-F · Base vs runtime.** M5 serializes the **base** level (doors start `CLOSED`, flags
  unset — `InfScript` exposes no initial-flag/timer/door-open state, and `InfWallGeometry.doors`
  already synthesises every door as `CLOSED`). The runtime overlay (`RuntimeState` — save
  games) is the design §8 deferral. No separate base-state struct is needed: `LevelDocument`'s
  geometry + events *is* the base.
- **D-G · Value equality on the array-backed IR types.** `IR_a.equals(IR_b)` requires value
  equality; records already have it, `GridPosition` has it, but `LevelGeometry`/`TileSet`/the
  new `PixelSheet`/`DecorationTable`/`AppearancePack` need `equals`/`hashCode` over their array
  state (`Arrays.equals`/`deepEquals`). Added and behavior-tested.
- **D-H · Retire the composition-root raw readers.** On completion, `Level1DecorationScene` and
  `Level1DoorLeafScene` (which read raw importer `DecorationDat`/`CpsImage`/`InfScript`) are
  **deleted**; the live app + all ITs render through the IR-fed render-core scenes fed by
  `LevelEvents` + `AppearancePack` (design §5; the M4 "orphaned IR type" follow-up).

## Design facts (grounded in the current code + eoblib/kyra oracles — see the M4 outcome)

- **Canonical LEVEL1 extract sequence** (from `LibGdxLauncher.loadGame`/`buildLevel` +
  `M4TriggerIT#loadFixtures`): `PakArchive.parse(EOBDATA3.PAK)` + `…(EOBDATA4.PAK)` →
  `Maze.parse("LEVEL1.MAZ")`, `InfScript.parse("LEVEL1.INF")`, `VgaPalette.parse("BRICK.PAL")`,
  `VcnAtlas.parse("BRICK.VCN")`, `VmpProjection.parse("BRICK.VMP")`,
  `DecorationDat.parse("BRICK.DAT")`, one `CpsImage.parse(<cpsName>.CPS)` per
  `script.overlays()` (3 for LEVEL1), `CpsImage.parse("DOOR.CPS")` from PAK4. Assembled via
  `InfWallGeometry.geometry(maze,script)` + `.doors(maze,script)`, `WestwoodToIr.tileSet(vcn,
  vmp,palette)`, `InfLifter.lift(script).triggers()`.
- **`InfScript` surface** (relied on by the importer tasks): `wallMapping(int)` /
  `hasWallMapping(int)` → `InfWallMapping(int wallType, int decorationId, int eventMask, int
  flags)` (`decorationId == 0xFF` = no decoration); `decorationOverlay(int wallMappingIndex)` →
  `InfOverlay(String cpsName, String datName)` (throws if the index has no decoration);
  `overlays()` → `List<InfOverlay>`; `triggers()`, `rawScript()`, `scriptBaseOffset()`,
  `mazeName()`, `vcnVmpName()`. No base-state/timer accessor exists (D-F).
- **Current decoration render** (`Level1DecorationScene`, the logic to move IR-fed):
  `spritesAt(col,row,dir,shapeSlot)` → `wallIndex = maze.wall(col,row,dir)`; if
  `script.hasWallMapping(wallIndex) && script.wallMapping(wallIndex).decorationId() != 0xFF`:
  pick sheet by `script.decorationOverlay(wallIndex).cpsName()`, walk the `DecorationDat`
  `next`-chain from `decorationId` (cap = `decorationCount`, terminate on `next==0`), per node
  emit a `DecorationSprite.of(argb, w, destX, destY, flags)` cut from the sheet's rectangle
  (`.dat` rectangle-table index = `shapeIndex(node, slot)`, skip if `== 0xFF`), palette index
  0 → transparent, CPS width 320.
- **Current door-leaf render** (`Level1DoorLeafScene`, the logic to move IR-fed):
  `leafAt(col,row,dir,distanceBand)` → `Optional<DoorLeafSprite>`; present iff a `Door` at
  `(block,edge)` is `CLOSED`; sprite = one of three fixed cut-rects into `DOOR.CPS` indexed by
  band (`FAR (200,75,37,29)`, `MID (128,151,62,48)`, `NEAR (0,0,80,72)`), palette index 0 →
  transparent, CPS width 320. `doorGraphicId` is unread (dead); leaf art is band-only.
- **Render seams (unchanged):** `DecorationScene.spritesAt(int col,int row,int dir,int
  shapeSlot) -> List<DecorationSprite>`; `DoorLeafScene.leafAt(int col,int row,int dir,int
  distanceBand) -> Optional<DoorLeafSprite>`; the fixed slot geometry lives in
  `DecorationRasterizer`/`DoorLeafRasterizer`. Only the *implementations* of the two seams
  change (raw-fed → IR-fed); the rasterizers and sprite value types are untouched.

## File structure

- `beholder-ir-json/` **(new module)** — `pom.xml` (deps: `beholder-ir`, `jackson-databind`,
  `json-schema-validator`); `src/main/java/eu/virtualparadox/beholder/ir/json/`:
  `IrJson.java` (public facade: `writeDocument`/`readDocument`, the pinned `ObjectWriter`),
  `IrJsonModule.java` (registers the serde), `serde/*Serializer.java`/`*Deserializer.java`,
  `LoadPipeline.java` (version gate → parse → schema → bind → semantic), `SchemaValidator.java`,
  `AssetFolderCodec.java` (pack ↔ folder binary), `IrDocument.java` (LevelDocument + pack);
  `src/main/resources/schema/level-document-1.json` (the JSON Schema).
- `beholder-ir/.../ir/model/` **(new IR values)** — `Decoration.java` (**rewritten**),
  `PixelSheet.java`, `DecorationTable.java`, `AppearancePack.java`, `LevelDocument.java`; and
  **equals/hashCode** added to `LevelGeometry.java`/`TileSet.java`.
- `beholder-importer-westwood/.../` **(pack + placement builders)** —
  `AppearancePackBuilder.java` (decode → `AppearancePack`), `DecorationPlacements.java`
  (maze scan → `List<Decoration>`).
- `beholder-render-core/.../render/` **(IR-fed seam impls)** — `IrDecorationScene.java`,
  `IrDoorLeafScene.java`.
- `beholder-render-libgdx/.../gl/` — `LibGdxLauncher`/`SceneOverlay` rewired to the IR-fed
  scenes; **delete** `Level1DecorationScene.java`, `Level1DoorLeafScene.java`; the round-trip
  IT + synthetic-golden test + negative fixtures in the test tree (test-dep on
  `beholder-ir-json`).

---

## Task 1: `beholder-ir-json` module skeleton + allowlisted deps + license gate green

**Retires the dependency/license/version/gate risk first, before any codec logic is built on
it.**

**Files:**
- Create: `beholder-ir-json/pom.xml`
- Modify: `pom.xml` (add `<module>beholder-ir-json</module>`)
- Create: `beholder-ir-json/src/main/java/eu/virtualparadox/beholder/ir/json/IrJson.java`
  (minimal canary: a pinned `ObjectWriter` + a `canonicalJson(GridPosition)` round-trip)
- Test: `beholder-ir-json/src/test/java/.../IrJsonCanaryTest.java`

**Interfaces produced:** `IrJson.canonicalWriter() -> ObjectWriter` (the single sanctioned
emitter: `ORDER_MAP_ENTRIES_BY_KEYS`, `INDENT_OUTPUT`, a pinned `DefaultPrettyPrinter` with LF
`\n` line separator + a trailing newline); a private `ObjectMapper` with
`FAIL_ON_UNKNOWN_PROPERTIES=true`, `READ_UNKNOWN_ENUM_VALUES_AS_NULL=false`,
`FAIL_ON_READING_DUP_TREE_KEY=true`.

- [ ] **Step 1: Write `beholder-ir-json/pom.xml`** with `<parent>` = the reactor root,
  `beholder-ir` (project version), and pinned Jackson + networknt. Pin versions as `<properties>`
  (e.g. `jackson.version` 2.17.x, `json-schema-validator.version` 1.4.x — pick the latest that
  resolves in `~/.m2`/central at build time and note it in the pom). Add the module to the root
  `pom.xml` `<modules>` (place after `beholder-ir` so reactor order builds it before consumers).
- [ ] **Step 2: Write the canary test (RED)** — `IrJsonCanaryTest`: serialize a `GridPosition`
  via a trivial hand-written serializer to `{"x":1,"y":2}`, assert the exact canonical string
  (LF, 2-space indent, trailing newline), and assert a second serialize of the same value is
  byte-identical.
- [ ] **Step 3: Run it — expect FAIL** (classes not present): `mvn -q -pl beholder-ir-json -am
  test`. Expected: compile error / assertion fail.
- [ ] **Step 4: Implement `IrJson` canary + the pinned writer** to make it pass.
- [ ] **Step 5: Verify the module + the license gate at the reactor root.** Run `mvn -q verify`
  (root) and confirm `BUILD SUCCESS` including the `license:aggregate-add-third-party` execution.
  If the gate reports a Jackson/networknt (or transitive, e.g. `com.ethlo.time:itu`) license
  string it does not recognise, add a `<licenseMerge>` under the root pom's existing
  `license-maven-plugin` execution mapping the reported SPDX-id onto its allow-listed canonical
  family name (the same mechanism already used for lwjgl) — **only** normalisation of an
  already-permitted family, never a new allow. Re-run until green. Record the outcome.
- [ ] **Step 6: Commit** — `feat(ir-json): module skeleton + pinned Jackson/networknt + canonical writer canary`.

---

## Task 2: Value equality for `LevelGeometry` + `TileSet` (round-trip foundation, D-G)

**Files:**
- Modify: `beholder-ir/.../ir/model/LevelGeometry.java`, `beholder-ir/.../ir/model/TileSet.java`
- Test: `beholder-ir/.../model/LevelGeometryEqualityTest.java`, `TileSetEqualityTest.java`

**Interfaces produced:** value `equals`/`hashCode` on both (used by `IR_a.equals(IR_b)` in
Tasks 12–13).

- [ ] **Step 1: Write the failing tests (RED).** For each type: two instances built from
  equal-valued inputs are `.equals` and share `hashCode`; instances differing in one field/one
  array element are not equal; not equal to `null`/another type. (Build inputs via the existing
  `of(...)` factories.)
- [ ] **Step 2: Run — expect FAIL** (`Object.equals` identity): `mvn -q -pl beholder-ir test`.
- [ ] **Step 3: Implement `equals`/`hashCode`** using `getClass()` comparison (no `instanceof`),
  `Arrays.equals`/`Arrays.deepEquals` over the array fields + scalar/`String` fields, one
  `return` per method (combine into a single boolean expression). Add `@Override` + Javadoc.
- [ ] **Step 4: Run — expect PASS**; then `mvn -q -pl beholder-ir verify` (gates + coverage —
  these are `model/`, coverage-exempt, but the guards/equality tests keep behaviour covered).
- [ ] **Step 5: Commit** — `feat(ir): value equals/hashCode on LevelGeometry + TileSet`.

---

## Task 3: Decoration-IR redesign + importer emits `List<Decoration>` (the must-fix, placement side)

**Files:**
- Rewrite: `beholder-ir/.../ir/model/Decoration.java` → `record Decoration(GridPosition block,
  int edge, String overlayId, int decorationId)` (D-A/D-B). Guards: `block` non-null, `edge`
  0..3, `overlayId` non-blank, `decorationId` non-negative.
- Create: `beholder-importer-westwood/.../DecorationPlacements.java`
- Test: `beholder-ir/.../model/DecorationTest.java` (guards), `.../DecorationPlacementsTest.java`
  (synthetic maze+script), `.../DecorationPlacementsIT.java` (real LEVEL1, skip-safe)

**Interfaces produced:** `DecorationPlacements.scan(Maze maze, InfScript script) ->
List<Decoration>` — walks every cell edge (row-major, N/E/S/W), and for each edge whose resolved
wall-mapping has `decorationId != 0xFF`, emits `new Decoration(block, dir,
script.decorationOverlay(index).cpsName(), script.wallMapping(index).decorationId())`. Consumes
Task-nothing (uses existing importer types); produced list feeds `LevelEvents.decorations()`
(Task 7 wiring).

- [ ] **Step 1: Rewrite `Decoration` + its guard test (RED first).** Delete the old `(int
  wallMappingIndex, int graphicId, int rectangleIndex)` shape (it is the M4 "orphaned type").
  Write `DecorationTest` for the four new guards; run `mvn -q -pl beholder-ir test` → FAIL.
- [ ] **Step 2: Implement the new `Decoration` record**; run → PASS. (Grep for old-field callers:
  none in production — it was never populated/consumed; fix any test references.)
- [ ] **Step 3: Write `DecorationPlacementsTest` (RED)** over a synthetic `Maze`+`InfScript`
  fixture (reuse the M4 `InfScript`-fixture builders if present, else a minimal `0xEC`+`0xFB`+
  trigger-table byte vector): a maze with one decorated wall-mapping index (with a defined
  overlay) on two edges → asserts exactly two `Decoration`s with the right `(block, edge,
  overlayId, decorationId)`, and an undecorated maze → empty list. Run → FAIL.
- [ ] **Step 4: Implement `DecorationPlacements.scan`**; run → PASS. Keep helpers small
  (per-cell / per-edge) to stay under the cyclomatic/params gates, mirroring
  `InfWallGeometry.doors`'s structure.
- [ ] **Step 5: Write `DecorationPlacementsIT` (real LEVEL1, skip-safe)** via
  `RealLevel1Archives.assumeReadable("EOBDATA3.PAK")`: `scan(Maze.parse(LEVEL1.MAZ),
  InfScript.parse(LEVEL1.INF))` returns a non-empty list, every entry's `overlayId` is one of
  `script.overlays()`' `cpsName`s, and the set of decorated `(block,edge)` equals the set of
  edges the current `Level1DecorationScene` would decorate (assert against the same
  `hasWallMapping && decorationId != 0xFF` predicate). Run (must NOT skip on this machine).
- [ ] **Step 6: `mvn -q -pl beholder-importer-westwood -am verify`; commit** —
  `feat(ir,importer): first-class Decoration placement IR + maze-scan producer`.

---

## Task 4: Appearance-pack IR values — `PixelSheet`, `DecorationTable`, `AppearancePack` (D-C, D-G)

**Files:**
- Create: `beholder-ir/.../ir/model/PixelSheet.java`, `.../DecorationTable.java`,
  `.../AppearancePack.java`
- Test: `.../PixelSheetTest.java`, `.../DecorationTableTest.java`, `.../AppearancePackTest.java`

**Interfaces produced:**
- `PixelSheet.of(int width, int height, byte[] paletteIndices) -> PixelSheet`; accessors
  `width()`, `height()`, `index(int x,int y) -> int` (0..255), defensive; value `equals`/`hashCode`.
- `DecorationTable.of(int decorationCount, int[] shapeIndex, int[] next, int[] flags, int[]
  destX, int[] destY, DecorationTable.Rect[] rectangles) -> DecorationTable` (arrays flattened
  `decorationCount * 10` for the per-slot arrays, mirroring `DecorationDat`); accessors
  `shapeIndex(dec,slot)`, `next(dec)`, `flags(dec)`, `destX(dec,slot)`, `destY(dec,slot)`,
  `rectangle(i) -> Rect(int x,int y,int w,int h)`, `decorationCount()`, `rectangleCount()`,
  static `noShape()` (0xFF), `endOfChain()` (0); value `equals`/`hashCode`. (Group the
  constructor args to stay ≤ 5 params — e.g. pass the five parallel `int[]`s as one `int[][]`
  "slots" param + `rectangles`, or a small builder; document the grouping like `TileSet` does.)
- `AppearancePack.of(String id, TileSet tileSet, DecorationTable decorationTable,
  Map<String,PixelSheet> decorationSheets, PixelSheet doorLeafSheet) -> AppearancePack`;
  accessors + `decorationSheet(String overlayId) -> PixelSheet` (fail-closed on unknown id);
  value `equals`/`hashCode`.

- [ ] **Step 1: `PixelSheetTest` (RED)** — `of` guards (positive dims, `indices.length ==
  width*height`), `index(x,y)` returns the byte unsigned, out-of-range throws; two equal sheets
  `.equals`. Implement `PixelSheet` (house pattern, array behind scalar accessor + defensive
  clone). Run → GREEN.
- [ ] **Step 2: `DecorationTableTest` (RED)** — build a 2-decoration/3-rectangle synthetic
  table; assert accessors, `noShape`/`endOfChain`, guards (index ranges), value equality.
  Implement `DecorationTable`. Run → GREEN.
- [ ] **Step 3: `AppearancePackTest` (RED)** — build a minimal pack; assert accessors,
  `decorationSheet` fail-closed on unknown `overlayId`, `id` non-blank guard, value equality
  (deep over the map + `deepEquals` of arrays). Implement `AppearancePack`. Run → GREEN.
- [ ] **Step 4: `mvn -q -pl beholder-ir verify`; commit** — `feat(ir): appearance-pack value
  types (PixelSheet, DecorationTable, AppearancePack)`.

---

## Task 5: Importer builds the `AppearancePack` from a real decode (D-C)

**Files:**
- Create: `beholder-importer-westwood/.../AppearancePackBuilder.java`
- Test: `.../AppearancePackBuilderTest.java` (synthetic), `.../AppearancePackBuilderIT.java`
  (real LEVEL1, skip-safe)

**Interfaces produced:** `AppearancePackBuilder.build(String id, TileSet tileSet, DecorationDat
dat, Map<String,CpsImage> decorationCps, CpsImage doorCps) -> AppearancePack` — copies `dat`'s
fields into a `DecorationTable`, each `CpsImage` into a `PixelSheet` (keyed by the same
`overlayId`=`cpsName` the placements use), and `doorCps` into `doorLeafSheet`. Fail-closed if
the level's overlays reference more than one distinct `datName` (M5 supports LEVEL1's single
shared `.dat`; a loud `IllegalArgumentException` naming the extra table, correctly-shaped for
later growth). Consumes `DecorationDat`/`CpsImage` (existing) + Task-4 IR values.

- [ ] **Step 1: `AppearancePackBuilderTest` (RED)** — synthetic `DecorationDat` + two synthetic
  `CpsImage`s + a synthetic `doorCps`; assert the built pack's `DecorationTable` mirrors the
  `dat` field-for-field, each `PixelSheet`'s bytes equal the source `CpsImage.pixels()`, and the
  door sheet matches. Run → FAIL.
- [ ] **Step 2: Implement `AppearancePackBuilder.build`** (small copy helpers per array to stay
  under complexity gates). Run → PASS.
- [ ] **Step 3: `AppearancePackBuilderIT` (real LEVEL1, skip-safe)** — reuse the canonical
  extract (Design facts): build `TileSet` via `WestwoodToIr.tileSet`, `DecorationDat.parse(
  BRICK.DAT)`, a `CpsImage` per `script.overlays()` cpsName, `CpsImage.parse(DOOR.CPS)` from
  PAK4; assert the pack has 3 decoration sheets, a non-empty decoration table, a `doorLeafSheet`
  of the expected `DOOR.CPS` dimensions, and that a spot-checked decoration rectangle's pixels
  match the raw `CpsImage` sample. Run (no skip on this machine).
- [ ] **Step 4: `mvn -q -pl beholder-importer-westwood -am verify`; commit** —
  `feat(importer): assemble the framework-neutral AppearancePack from decoded assets`.

---

## Task 6: IR-fed render seams — `IrDecorationScene` + `IrDoorLeafScene` (D-C, D-H)

**Files:**
- Create: `beholder-render-core/.../render/IrDecorationScene.java`,
  `.../render/IrDoorLeafScene.java`
- Test: `.../IrDecorationSceneTest.java`, `.../IrDoorLeafSceneTest.java` (synthetic)

**Interfaces produced:**
- `IrDecorationScene` implements `DecorationScene`. Ctor: `(List<Decoration> decorations,
  AppearancePack pack)`. `spritesAt(col,row,dir,shapeSlot)` looks up the `Decoration` at
  `(block=GridPosition.of(col,row), edge=dir)` (a `Map<(block,edge)>` built in the ctor); if
  present, walks `pack.decorationTable()`'s `next`-chain from `decorationId`, cutting each
  slot's rectangle from `pack.decorationSheet(overlayId)` (palette index 0 → transparent, else
  `pack.tileSet().paletteArgb()[idx]`), producing the same `DecorationSprite`s
  `Level1DecorationScene` produces today.
- `IrDoorLeafScene` implements `DoorLeafScene`. Ctor: `(List<Door> doors, DoorClosedTest closed,
  PixelSheet doorLeafSheet, int[] paletteArgb)` where `DoorClosedTest` is a small functional seam
  `boolean isClosed(GridPosition block)` (so render-core stays free of `engine-core`; the live
  app passes a `RuntimeState`-backed test, the round-trip passes "base = every door closed").
  `leafAt(col,row,dir,band)` → `Optional<DoorLeafSprite>` cut from the band's fixed rect (moved
  here as a `render` constant) when a door at `(block,edge)` is closed.

- [ ] **Step 1: `IrDecorationSceneTest` (RED)** — a synthetic `AppearancePack` (one overlay,
  one 2-node decoration chain, one rectangle) + one `Decoration` at `(1,1,N)`; assert
  `spritesAt(1,1,0,slot)` returns the expected `DecorationSprite` (dims, destX/destY/flags,
  and a spot pixel), and an undecorated edge returns an empty list. Run → FAIL.
- [ ] **Step 2: Implement `IrDecorationScene`** — port the chain-walk/cut/palette-resolve from
  `Level1DecorationScene` (the read-only reference), sourced from IR + pack. Run → PASS.
- [ ] **Step 3: `IrDoorLeafSceneTest` (RED)** — a synthetic `doorLeafSheet` + one `Door` at
  `(2,2,E)` + a `closed`-test returning true; assert `leafAt(2,2,1,band)` returns a leaf of the
  band rect's dims for each band, `empty` when the closed-test is false, `empty` for a
  non-door edge. Run → FAIL.
- [ ] **Step 4: Implement `IrDoorLeafScene`** (the three band rects as named constants + a
  what/why comment citing `DOOR.CPS` visual-inspection provenance, transcribed from
  `Level1DoorLeafScene`). Run → PASS.
- [ ] **Step 5: `mvn -q -pl beholder-render-core -am verify` (branch coverage of both — not
  `model/`); commit** — `feat(render): IR-fed decoration + door-leaf scenes`.

---

## Task 7: Rewire the live app + ITs to the IR-fed scenes; retire the raw readers (D-H)

**Files:**
- Modify: `beholder-render-libgdx/.../gl/LibGdxLauncher.java` (build `List<Decoration>` via
  `DecorationPlacements.scan`, the `AppearancePack` via `AppearancePackBuilder.build`, feed
  `IrDecorationScene`/`IrDoorLeafScene`; populate `LevelEvents` with the real decorations),
  `.../gl/SceneOverlay.java` if its ctor types change; the ITs that build the two scenes
  (`M4TriggerIT`, `LevelOneEntryRenderIT`, `DecorationRenderIT`, the shared
  `RealLevel1DecorationScenes` helper).
- **Delete:** `beholder-render-libgdx/.../gl/Level1DecorationScene.java`,
  `.../gl/Level1DoorLeafScene.java` (and `RealLevel1DecorationScenes` if it only built the old
  scene).
- Test: `beholder-render-libgdx/.../RenderIdentityIT.java` (new, real LEVEL1, skip-safe)

**Interfaces produced:** the live `GameSession` renders decorations/doors from `LevelEvents` +
`AppearancePack` only (no raw `DecorationDat`/`InfScript` in the render path).

- [ ] **Step 1: Write `RenderIdentityIT` (RED, real LEVEL1, skip-safe)** — the safety net for
  the deletion: build the *old* `Level1DecorationScene`/`Level1DoorLeafScene` AND the *new*
  `IrDecorationScene`/`IrDoorLeafScene` from the same real decode, render the canonical entry
  view (`(10,15)` NORTH — the `LevelOneEntryRenderIT` viewpoint) through both, and assert the
  two `int[]` frames are **pixel-identical**. Run → it should PASS immediately if Task 6 is
  faithful (this is the proof the IR-fed path reproduces the raw path); if it FAILs, fix Task 6
  before proceeding.
- [ ] **Step 2: Rewire `LibGdxLauncher`** to the IR-fed scenes + populate
  `LevelEvents(triggers, doors, DecorationPlacements.scan(maze,script))`. Run the module build.
- [ ] **Step 3: Rewire the ITs** (`M4TriggerIT`/`LevelOneEntryRenderIT`/`DecorationRenderIT`/
  helpers) to the IR-fed scenes; keep their existing assertions green.
- [ ] **Step 4: Delete the raw readers** (`Level1DecorationScene`, `Level1DoorLeafScene`, dead
  helpers). Re-run `RenderIdentityIT` against a saved golden `int[]` (or fold the "old" side into
  a committed hash captured in Step 1) so the identity proof survives the deletion.
- [ ] **Step 5: `mvn -q verify` (root, all modules); commit** — `feat(render-libgdx): render
  decorations+doors from IR + AppearancePack; retire raw composition-root readers`.

---

## Task 8: `LevelDocument` + canonical JSON writer + behaviour-IR serializers (D-D, D-E)

**Files:**
- Create: `beholder-ir/.../ir/model/LevelDocument.java` — `record LevelDocument(String
  irVersion, LevelGeometry geometry, LevelEvents events, String appearancePackId)` (guards:
  non-null; `irVersion`/`appearancePackId` non-blank).
- Create: `beholder-ir-json/.../json/IrJsonModule.java`, `.../json/serde/*Serializer.java`
- Modify: `beholder-ir-json/.../json/IrJson.java` (`toCanonicalJson(LevelDocument) -> String`)
- Test: `beholder-ir-json/.../IrJsonWriteTest.java`

**Interfaces produced:** `IrJson.toCanonicalJson(LevelDocument) -> String` — deterministic,
byte-stable; the JSON root object begins with `"irVersion"`. Serializers for: `LevelDocument`,
`LevelGeometry` (width/height + per-edge `{archetype, wallSet}` arrays), `GridPosition`,
`LevelEvents` (`triggers`/`doors`/`decorations`), `TriggerBinding` (block/reasons/handler),
`Handler`+`Guard`+`Action.*`+`Condition.*` (sealed → `"kind"` discriminator, dispatched via the
existing `ActionVisitor`/`ConditionVisitor`/`StepVisitor`), `FlagRef`, `Reason` (enum name),
`Door`, `Decoration`. `TileSet`/`AppearancePack` are **not** serialized here (asset folder,
Task 11).

- [ ] **Step 1: `LevelDocument` + its guard test (RED→GREEN)** in `beholder-ir`; commit-worthy
  but fold into this task.
- [ ] **Step 2: `IrJsonWriteTest` (RED)** — build a small synthetic `LevelDocument` (2×2 grid,
  one trigger with a `Guard(Comparison(EQ, WallIndexAt, Literal), [SetWall, OpenDoor])`, one
  `ShowMessage`, one `SetFlag`, one door, one decoration) and assert `toCanonicalJson` equals a
  hand-written expected string (exact bytes) **and** re-serializing is byte-identical. Run → FAIL.
- [ ] **Step 3: Implement the serializers + register them in `IrJsonModule`.** The sealed-type
  serializers emit `{"kind":"SetWall", …}` etc. via the visitors (no `instanceof`/`switch
  yield`). Wire the module into `IrJson.canonicalWriter()`. Run → PASS.
- [ ] **Step 4: Determinism check** — a property-style test: serialize the same document twice →
  identical; serialize two `equals` documents built independently → identical.
- [ ] **Step 5: `mvn -q -pl beholder-ir-json -am verify`; commit** — `feat(ir-json):
  LevelDocument + deterministic canonical writer + behaviour-IR serializers`.

---

## Task 9: Behaviour-IR deserializers + fail-closed load pipeline (ADR-0001 §5, D-D)

**Files:**
- Create: `beholder-ir-json/.../json/serde/*Deserializer.java`, `.../json/LoadPipeline.java`
- Modify: `.../json/IrJson.java` (`fromJson(String) -> LevelDocument`, running the pipeline
  minus the schema step, which Task 10 inserts)
- Test: `.../IrJsonReadTest.java`, `.../LoadPipelineNegativeTest.java`

**Interfaces produced:** `IrJson.fromJson(String json) -> LevelDocument` — pipeline: (1)
**version gate** — read top-level `irVersion` first; reject if its major ≠ the supported major
(`1`) with a clear message; (2) **structural parse** with `FAIL_ON_UNKNOWN_PROPERTIES=true`;
(3) **bind** via the deserializers (unknown `"kind"`/enum → hard fail, no default); (4)
**semantic/referential validation** — `edge`/`side` in 0..3, grid coords in range, `Guard.then`
/`Handler.steps` non-empty, `TriggerBinding.reasons` non-empty (delegated to the record guards),
`appearancePackId` non-blank. (Schema validation is Task 10, inserted between 2 and 3.)

- [ ] **Step 1: `IrJsonReadTest` (RED)** — `fromJson(toCanonicalJson(doc)).equals(doc)` for the
  Task-8 synthetic document (full behaviour-IR round-trip in memory). Run → FAIL.
- [ ] **Step 2: Implement the deserializers + `LoadPipeline`** (version gate → parse → bind →
  semantic). Sealed types read `"kind"` and dispatch to a small per-kind factory map (no
  `instanceof`). Run → PASS.
- [ ] **Step 3: `LoadPipelineNegativeTest` (RED→GREEN)** — assert each is **rejected** with a
  specific message: unknown `Action` `"kind"`; unknown `Reason` enum value; an extra/unknown
  property (`FAIL_ON_UNKNOWN_PROPERTIES`); a missing required field; `irVersion` `"2.0.0"`
  (wrong major); `irVersion` `"1.9.0"` (same major → accepted, forward-compatible minor). Each
  asserts the thrown type + a message substring.
- [ ] **Step 4: `mvn -q -pl beholder-ir-json -am verify`; commit** — `feat(ir-json): fail-closed
  load pipeline (version gate → parse → bind → semantic) + deserializers`.

---

## Task 10: JSON Schema (record-derived) + schema-validation step + no-drift guard (ADR-0001 §3)

**Files:**
- Create: `beholder-ir-json/src/main/resources/schema/level-document-1.json` (Draft 2020-12,
  `additionalProperties:false` at every object, closed `enum` lists for `Reason`/`Op`/`BoolKind`/
  `FlagRef.Scope`/`WallArchetype`/`DoorState`/each `"kind"`)
- Create: `.../json/SchemaValidator.java`; Modify: `.../json/LoadPipeline.java` (insert schema
  validation between parse and bind)
- Test: `.../SchemaValidationTest.java`, `.../SchemaDriftGuardTest.java`

**Interfaces produced:** `SchemaValidator.validate(JsonNode) -> void` (throws listing every
violation) invoked in the pipeline after structural parse, before bind — enforcing required
fields, `additionalProperties:false`, and enum membership at the schema layer.

- [ ] **Step 1: `SchemaValidationTest` (RED)** — the Task-8 canonical JSON validates clean; the
  Task-9 negatives (extra property, unknown enum, missing required) are **also** caught here at
  the schema layer (belt-and-suspenders with the parse/bind layer). Run → FAIL (no schema yet).
- [ ] **Step 2: Author the schema + `SchemaValidator`** (load the resource once, cache the
  compiled `JsonSchema`). Insert into `LoadPipeline`. Run → PASS.
- [ ] **Step 3: `SchemaDriftGuardTest` (RED→GREEN)** — the "checked-against-the-records in CI"
  mechanism resolving ADR-0001's open question: for every closed IR enum
  (`Reason`, `Condition.Op`, `Condition.BoolKind`, `FlagRef.Scope`, `WallArchetype`,
  `Door.DoorState`) assert the schema's corresponding `enum` array **equals** the Java enum's
  `values()` names (so adding a Java enum constant without updating the schema fails the build);
  and for every `Action`/`Condition`/`Step` sealed variant, assert its `"kind"` appears in the
  schema's discriminator `enum`. Build a *maximal* synthetic `LevelDocument` exercising **every**
  Action + Condition + Reason + Scope + archetype + door state at least once, serialize it, and
  assert it validates clean (proves the schema accepts every valid node kind).
- [ ] **Step 4: `mvn -q -pl beholder-ir-json -am verify`; commit** — `feat(ir-json): JSON Schema
  + schema-validation pipeline step + enum/kind no-drift guard`.

---

## Task 11: Asset-folder binary codec — write/read `AppearancePack` by id (D-C, D-D)

**Files:**
- Create: `beholder-ir-json/.../json/AssetFolderCodec.java`
- Test: `.../AssetFolderCodecTest.java`

**Interfaces produced:** `AssetFolderCodec.write(Path dir, AppearancePack pack) -> void` and
`AssetFolderCodec.read(Path dir, String appearancePackId) -> AppearancePack` — a **deterministic,
framework-neutral** binary layout under `dir` (e.g. `tileset.bin`, `decorations/<overlayId>.bin`,
`decorations/table.bin`, `doors/leaf.bin`, plus a small `pack.json` manifest binding the id +
listing the overlay ids). Big-endian length-prefixed ints/bytes; **no Westwood format** written.
`read` is fail-closed: missing file / id mismatch / truncated stream → `IllegalArgumentException`.

- [ ] **Step 1: `AssetFolderCodecTest` (RED)** — build the Task-4/5 synthetic `AppearancePack`;
  `write(tmp, pack)` then `read(tmp, pack.id()).equals(pack)`; assert byte-stability (write
  twice → the produced files are byte-identical); assert fail-closed on a deleted file, on a
  wrong id, and on a truncated `tileset.bin`. Run → FAIL.
- [ ] **Step 2: Implement `AssetFolderCodec`** — one small writer/reader helper per asset kind
  (tileset arrays, decoration table arrays, each pixel sheet, door leaf) to stay under the
  complexity/method gates; use `try-with-resources` streams; a fixed magic + version header per
  file (named constants + why-comment). Run → PASS.
- [ ] **Step 3: `mvn -q -pl beholder-ir-json -am verify`; commit** — `feat(ir-json): deterministic
  asset-folder codec (AppearancePack ↔ framework-neutral binaries by id)`.

---

## Task 12: `IrDocument` write/read — JSON + asset folder as one artifact (D-D)

**Files:**
- Create: `beholder-ir-json/.../json/IrDocument.java` — `record IrDocument(LevelDocument level,
  AppearancePack pack)` with a guard that `level.appearancePackId().equals(pack.id())`.
- Modify: `.../json/IrJson.java` — `writeDocument(Path dir, String baseName, IrDocument doc)`
  (writes `<baseName>.json` via the canonical writer + `<baseName>-assets/` via
  `AssetFolderCodec`) and `readDocument(Path dir, String baseName) -> IrDocument` (reads +
  fail-closed cross-check that the JSON's `appearancePackId` resolves in the asset folder).
- Test: `.../IrDocumentRoundTripTest.java`

**Interfaces produced:** `IrJson.writeDocument`/`readDocument` — the full-artifact codec the
round-trip IT drives.

- [ ] **Step 1: `IrDocumentRoundTripTest` (RED)** — synthetic `IrDocument`; `writeDocument(tmp,
  "syn", doc)`; assert `syn.json` + `syn-assets/` exist; `readDocument(tmp,"syn").equals(doc)`;
  re-`writeDocument` produces a byte-identical `syn.json`; and a tampered `appearancePackId`
  (JSON references a pack the folder lacks) is **rejected**. Run → FAIL.
- [ ] **Step 2: Implement `IrDocument` + `writeDocument`/`readDocument`.** Run → PASS.
- [ ] **Step 3: `mvn -q -pl beholder-ir-json -am verify`; commit** — `feat(ir-json): IrDocument
  write/read (canonical JSON + asset folder, fail-closed pack-id cross-check)`.

---

## Task 13: Real-LEVEL1 lossless round-trip IT — the acceptance test (design §6)

**Files:**
- Create: `beholder-render-libgdx/.../LosslessRoundTripIT.java` (real LEVEL1, skip-safe)
- Modify: `beholder-render-libgdx/pom.xml` (test-scope dep on `beholder-ir-json`)

**Interfaces produced:** none (a proof). Uses the canonical extract (Design facts) to build
`IR_a` = `IrDocument(LevelDocument(irVersion, geometry, events-with-decorations,
appearancePackId), AppearancePack)`.

- [ ] **Step 1: Write the IT (real LEVEL1, skip-safe).** `RealLevel1Archives.assumeReadable(
  "EOBDATA3.PAK","EOBDATA4.PAK")`. Build `IR_a` from the real decode (geometry via
  `InfWallGeometry.geometry`; `events = LevelEvents(InfLifter.lift(script).triggers(),
  InfWallGeometry.doors(maze,script), DecorationPlacements.scan(maze,script))`; pack via
  `AppearancePackBuilder.build`). `writeDocument(target/roundtrip, "level1", IR_a)`;
  `IR_b = readDocument(target/roundtrip, "level1")`.
- [ ] **Step 2: Assert the three legs.** (a) **equality:** `IR_b.equals(IR_a)`; (b)
  **byte-stability:** `toCanonicalJson(IR_b.level())` equals the on-disk `level1.json` bytes, and
  a re-`writeDocument` of `IR_b` reproduces byte-identical `level1.json` + asset files; (c)
  **render-identity:** render the canonical entry view (`(10,15)` NORTH) from `IR_a` and from
  `IR_b` through the IR-fed scenes (`FirstPersonRenderer` + `DecorationRasterizer` +
  `DoorLeafRasterizer`, base door state = all CLOSED) and assert the two `int[]` frames are
  **pixel-identical**. Write a before/after contact-sheet PNG to git-ignored `target/`.
- [ ] **Step 3: Run it (must NOT skip on this machine); confirm no game data is committed**
  (`git status` shows only source/tests; `target/` is git-ignored). `mvn -q verify` (root).
- [ ] **Step 4: Commit** — `test(ir-json): real-LEVEL1 lossless round-trip IT (equals +
  byte-stable + render-identity)`.

---

## Task 14: Committed synthetic-level golden + fail-closed negative fixtures (design §6)

**Files:**
- Create: `beholder-ir-json/src/test/resources/golden/synthetic-level.json` (committed —
  **our own hand-authored content**)
- Create: `beholder-ir-json/src/test/resources/negative/*.json` (committed malformed fixtures)
- Create: `.../SyntheticGoldenTest.java`, `.../NegativeFixtureTest.java`
- Create: a tiny `SyntheticLevel.java` test factory (builds the hand-authored `LevelDocument`
  exercising every IR node kind at least once)

**Interfaces produced:** none (the human-readable format ratchet + the negative-fixture proof).

- [ ] **Step 1: Build `SyntheticLevel` (RED via the golden test).** A small hand-authored
  `LevelDocument` (e.g. a 3×3 grid) that uses **every** node kind at least once: each
  `WallArchetype`, each `Action` variant, each `Condition` variant, each `Reason`, both
  `FlagRef.Scope`s, a `Guard` with an `else`-less early return, a door, a decoration. Write
  `SyntheticGoldenTest`: `toCanonicalJson(SyntheticLevel.document())` equals the committed
  `synthetic-level.json` **byte-for-byte**, and `fromJson(<file>).equals(SyntheticLevel.document())`.
- [ ] **Step 2: Generate the golden.** Run the test once to print the canonical JSON, write it
  to `synthetic-level.json`, re-run → PASS. (This file is the reviewable "is this the JSON we
  want as the product?" artifact for the checkpoint.)
- [ ] **Step 3: `NegativeFixtureTest` (RED→GREEN)** — commit one malformed JSON per failure mode,
  each asserted **rejected** by `IrJson.fromJson`/`readDocument` with a specific message:
  (1) `unknown-enum.json` (a `Reason` not in the vocabulary); (2) `extra-property.json` (an
  unknown field → `additionalProperties:false` + `FAIL_ON_UNKNOWN_PROPERTIES`); (3)
  `missing-required.json` (a `TriggerBinding` with no `block`); (4) `unresolved-reference.json`
  (an `appearancePackId` with no matching asset folder / an `overlayId` absent from the pack);
  (5) `wrong-major.json` (`irVersion` `"2.0.0"`). Each fixture is minimal + commented in the test.
- [ ] **Step 4: `mvn -q -pl beholder-ir-json -am verify`; commit** — `test(ir-json): committed
  synthetic-level golden + fail-closed negative fixtures`.

---

## Self-Review (plan vs. design)

- **Spec coverage.** Design §3.1 full-behaviour-IR serialization → Tasks 8–10 (geometry, events,
  doors, decorations) + §3.1 tileset-by-ID → Tasks 11–12 (asset folder). §3.2(a) real-LEVEL1
  round-trip → Task 13. §3.2(b) synthetic golden + negatives → Task 14. §3.3 decorations
  first-class → Tasks 3–7 (placement IR + pack + IR-fed render + retire raw reader). §3.4 schema
  checked-against-records → Task 10 drift guard. §4.1 engine-Jackson-free codec module → Task 1 +
  D-E. §4.2 fail-closed pipeline → Task 9 + 10. §4.3 deterministic writer → Task 1 + 8. §5
  decoration-IR fix → Tasks 3–7. §7 task sketch (1 codec core / 2 decoration redesign / 3 cover
  whole IR + asset-by-ID / 4 harness+evidence) → mapped across Tasks 1,8–12 / 3–7 / 8–12 / 13–14.
  §8a growth substrate (versioned schema + closed-but-extensible vocab + asset-by-ID) → the
  version gate (T9), the enum drift guard (T10), and the asset-by-ID codec (T11–12).
- **Placeholder scan.** The decode/asset/render tasks (3,5,6,7,13) are discovery-against-real-data
  with concrete differentials (the Design facts + each IT's assertions are the spec, matching the
  approved M4 style); the codec tasks (1,8–12,14) carry concrete interfaces, exact JSON/assertions.
  No "TBD"/"handle edge cases"/"similar to Task N".
- **Type consistency.** `Decoration(block,edge,overlayId,decorationId)` is produced by
  `DecorationPlacements.scan` (T3), stored in `LevelEvents.decorations()` (T7), consumed by
  `IrDecorationScene` (T6) + serialized (T8) + validated (T9/T10). `AppearancePack`
  (id/tileSet/decorationTable/decorationSheets/doorLeafSheet) is built by `AppearancePackBuilder`
  (T5), consumed by the IR-fed scenes (T6), written/read by `AssetFolderCodec` (T11), bundled in
  `IrDocument` (T12). `LevelDocument(irVersion,geometry,events,appearancePackId)` is the JSON root
  (T8) round-tripped by the pipeline (T9) and the full document (T12). `overlayId` = `cpsName`
  throughout; `decorationId` = `.dat` chain start node throughout; `edge`/`side` 0..3 N/E/S/W
  throughout.
- **Ordering / risk retirement.** Deps+license first (T1); the must-fix (decorations first-class +
  render-identity) is proven **before** the JSON codec (T3–T7), so a regression there surfaces
  independently of serialization; the codec is built writer→reader→schema→asset→document
  (T8–T12) each with its own gate; the two acceptance proofs (real round-trip, synthetic golden +
  negatives) come last (T13–T14). Engine stays Jackson-free by construction (no task adds a codec
  dep to `beholder-engine-core`).
