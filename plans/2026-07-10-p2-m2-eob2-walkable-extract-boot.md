# P2 · M2 — EOB2 walkable + extract/boot generalization — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> to implement this plan task-by-task (fresh implementer + independent review per task, a
> whole-branch review at the end, `--no-ff` merge on the maintainer's eyeball verdict; no
> worktree). TDD per task (RED → GREEN → commit). Steps use checkbox (`- [ ]`) syntax. Style
> follows the P1/M1 plans: the boot/extract/CLI tasks are *pure wiring* and carry concrete code;
> the synthetic-fixture + real-data tasks are *discovery against existing patterns + real data*
> (the Design facts + each task's differential are the spec). Every task obeys the Global
> Constraints.

**Goal:** Make a real EOB2 level **walkable** through the game-neutral engine, **booted from
IR-on-disk, not the original files** — via a new `beholder-app` module with a two-phase
`extract` (per-game importer → IR folder) / `boot` (game-agnostic IR → `GameSession`) pipeline,
and rewire the EOB1 launcher onto the same path. Proves the engine is game-neutral (one
`GameSession` walks both games) and makes the composition root IR-pure + testable.

**Architecture:** New `beholder-app` module = the top-level composition. `Extract` (per-game)
runs the EOB1/EOB2 importer → `IrDocument` (`LevelDocument` with **empty triggers** — base
level — + doors + decorations, and an `AppearancePack`) → `IrJson.writeDocument` (JSON + asset
folder, M5's codec). `Boot` (game-agnostic) = `IrJson.readDocument` → `Level` + `RuntimeState`
+ `SceneOverlay` → a `BootedGame` a `GameSession` is built from — **it never imports a Westwood
decoder** (MASTER §7.3). The libGDX window becomes a thin `GameWindow.play(BootedGame)` frontend
in render-libgdx; the CLI (`extract`/`play`) lives in `beholder-app`.

**Tech Stack:** Java 21, Maven multi-module, JUnit 5 + AssertJ. Reuses the whole engine
(`GameSession`/`RuntimeState`/movement/collision), the render scenes, the M5 `beholder-ir-json`
codec, and the M1 EOB2 + EOB1 importers — **no new engine/render/IR code**.

## Global Constraints

- **JDK 21** (`JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home`);
  `mvn spotless:apply` then `mvn verify`; single-module `-pl <m> -am` **or** root verify (stale
  `~/.m2` upstream-jar trap). The reactor enforces JDK `[21,22)`.
- **All `CODING_STANDARDS` gates green:** one `return`/method, no `instanceof` (visitor/`getClass`),
  no `switch yield`, cyclomatic ≤ 8, NPath ≤ 120, ≤ 12 fields / ≤ 18 methods / ≤ 5 effective
  params, coupling ≤ 10, no `return null` (Optional/throw/empty), `final` params, braces
  everywhere, ≤ 140-char lines, no `System.out`/`printStackTrace`, NullAway `ERROR`, full Javadoc
  (`@param`/`@return`), **every magic value a named `ALL_CAPS` constant + a what-and-why comment**.
- **Coverage: JaCoCo ≥ 0.90 branch AND line at bundle level.** The genuinely window-driving glue
  (the `GameWindow` LWJGL adapter) follows the existing render-libgdx `gl/**` JaCoCo-exclusion
  precedent; **`Boot`/`Extract`/`GameDetector`/the extractors are NOT exempt** and are branch-
  covered by the synthetic-IR + real-data tests.
- **New module deps are intra-project + allow-listed.** `beholder-app` adds **no new third-party
  dependency**; it depends on existing modules (`beholder-importer-westwood`, `beholder-ir-json`,
  `beholder-engine-core`, `beholder-render-core`, `beholder-render-libgdx`). The reactor license
  gate must stay green (Task 1 verifies).
- **Clean-room:** kyra/eoblib read-only oracles; jCodeMunch local-only. **Never commit game
  data:** real-data ITs are `assumeTrue`-skip-safe; extracted IR folders + PNGs go to git-ignored
  `target/`. The **synthetic IR fixture is our own content and IS committed** (the game-data-free
  boot test).
- **Package root** `eu.virtualparadox.beholder.app`.
- Branch `feat/p2-m2-eob2-walkable-extract-boot`; commit per task; whole-branch review + `--no-ff`
  merge. Base = current `main` (post-M1, `e8c1a4a`).

---

## Plan-level design decisions (resolving the design's deferred points)

- **D-A · Party start = boot parameter, NOT IR-carried (M2).** The design (§4.2) offered both and
  leaned IR-carried; the recon shows IR-carried touches **five** surfaces (`LevelDocument` record +
  serializer + deserializer allow-list + JSON Schema + a **new `Facing` serde**, which does not
  exist) and risks the ADR-0001 major-version gate. Per YAGNI and the design's explicit "simpler"
  option, M2 makes `start` a parameter to `boot`/`play`: `Boot.boot(irFolder, baseName, start,
  facing)`. Tests pass explicit starts; the `play` CLI uses a documented per-game default (or a
  `--start` arg). IR-carried "the level is its spawn" is a clean *additive* extension deferred to a
  later milestone (add an **optional** `start` field then — v1-compatible, no major bump).
- **D-B · Base level for BOTH games (empty triggers) in M2.** `Extract` writes `new LevelEvents(
  List.of(), doors, decorations)` for EOB1 **and** EOB2. Rationale (maintainer decision): booting
  EOB1's real `SET_WALL` triggers needs the importer-backed `SceneOverlay.wallMappingResolver`
  (the §7 leak) which is M3's fix (IR-carried wall-mapping table). Consequence: the manually-run
  EOB1 live window temporarily loses its M4 trigger demo — **nothing automated regresses** (the M4
  `M4TriggerIT`/`DoorRenderIT` test the engine/render directly, not the launcher; the engine
  trigger capability is untouched). M3 restores triggers for both games. `boot` uses
  `IntUnaryOperator.identity()` as the (never-invoked, no-`SET_WALL`) resolver.
- **D-C · Boot builds a LIVE door-closed test.** `IrDoorLeafScene`'s `DoorClosedTest` reads the
  live `RuntimeState` (`block -> state.doorState(block, level) == Door.DoorState.CLOSED`), like the
  current `LibGdxLauncher` (not the `LosslessRoundTripIT`'s static `ALL_DOORS_CLOSED`) — so an
  opened door hides its leaf. In M2 (no triggers) every door stays closed, but the wiring is the
  live one.
- **D-D · Game detection = auto (with the EOB2 `.app` loose layout vs EOB1 PAK).** `GameDetector`:
  an `EOBDATA*.PAK` present in the folder → EOB1; a loose `LEVEL1.INF` present → EOB2; neither →
  fail-closed `IllegalArgumentException` naming the folder. (An explicit override is not built in
  M2 — YAGNI; add a `--game` flag only if a real folder defeats detection.)
- **D-E · The EOB1 rewire retires `LibGdxLauncher`'s in-process loading.** Its window adapter moves
  to render-libgdx `GameWindow.play(BootedGame)` (generalized, EOB1-loading removed); its `main`
  becomes `beholder-app`'s `Main`. The `Level1AppearancePack` EOB1 assembler stays in render-libgdx
  and is called by `Eob1Extractor` (app depends on render-libgdx). The render-libgdx `exec`/`run`
  profile's main class moves to `eu.virtualparadox.beholder.app.Main` (relocated to the app pom).

## Design facts (exact signatures — grounded in the current code; the spec for the wiring)

**M5 codec (`beholder-ir-json`, `eu.virtualparadox.beholder.ir.json`):**
- `IrJson.writeDocument(Path dir, String baseName, IrDocument doc) throws IOException` — writes
  `<baseName>.json` + `<baseName>-assets/`.
- `IrJson.readDocument(Path dir, String baseName) -> IrDocument` — fail-closed
  (`IllegalArgumentException` on any malformation / id mismatch).
- `record IrDocument(LevelDocument level, AppearancePack pack)` — fail-closed ctor (ids match, every
  decoration resolves in the pack); value `equals`/`hashCode`.

**IR (`beholder-ir`, `…ir.model` / `…ir.event`):**
- `record LevelDocument(String irVersion, LevelGeometry geometry, LevelEvents events, String
  appearancePackId)` — **no start field**. `LevelGeometry.CURRENT_IR_VERSION` (="1.0.0").
- `new LevelEvents(List<TriggerBinding> triggers, List<Door> doors, List<Decoration> decorations)`
  — M2 passes `List.of()` for triggers.

**Engine host (build a `GameSession` from IR):**
- `new GameSession(Level level, RuntimeState state, InputSource input, FrameSink sink, SceneOverlay
  overlays)`; `session.tick()`, `session.hudLine()`, `session.toggleNoclip()`.
- `record SceneOverlay(DecorationScene decorations, DoorLeafScene doorLeaves, IntUnaryOperator
  wallMappingResolver)`; `SceneOverlay.NONE`. Inert resolver = `IntUnaryOperator.identity()`.
- `RuntimeState.startingAt(Party party) -> RuntimeState`; `state.applyCommand(MovementCommand cmd,
  Level level)`; `state.doorState(GridPosition block, Level level) -> Door.DoorState`; `state.party()
  -> Party` (with `.position() -> GridPosition`, `.facing() -> Facing`).
- `Party.of(GridPosition position, Facing facing) -> Party`.
- `new IrDecorationScene(List<Decoration> decorations, AppearancePack pack)`.
- `new IrDoorLeafScene(List<Door> doors, DoorClosedTest closed, PixelSheet doorLeafSheet, int[]
  paletteArgb)` — `DoorClosedTest` is `block -> boolean`.
- `Level.of(LevelGeometry geometry, TileSet tileset, LevelEvents events) -> Level`.
- Seams: `interface FrameSink { void present(int[] frame); }`, `interface InputSource { List<
  MovementCommand> poll(); }` (both `…render.libgdx`).

**The boot template (from `LosslessRoundTripIT.renderEntryFrame`, the exact shape `Boot` mirrors):**
```java
Level level = Level.of(doc.geometry(), pack.tileSet(), doc.events());
RuntimeState state = RuntimeState.startingAt(Party.of(start, facing));
DecorationScene decorations = new IrDecorationScene(doc.events().decorations(), pack);
DoorLeafScene doorLeaves = new IrDoorLeafScene(doc.events().doors(),
        block -> state.doorState(block, level) == Door.DoorState.CLOSED,   // D-C live test
        pack.doorLeafSheet(), pack.tileSet().paletteArgb());
SceneOverlay overlays = new SceneOverlay(decorations, doorLeaves, java.util.function.IntUnaryOperator.identity());
// BootedGame = (level, state, overlays); GameSession is built by the caller with its FrameSink/InputSource.
```

**Extract-phase importer seams (`beholder-importer-westwood`, `…importer.westwood`):**
- EOB1: `PakArchive.parse(byte[])`/`pak.file(String)`; `Maze.parse`, `InfScript.parse`,
  `VgaPalette.parse`, `VcnAtlas.parse`, `VmpProjection.parse`; `InfWallGeometry.geometry(maze,
  script)`/`.doors(maze, script)`; `DecorationPlacements.scan(maze, script)`;
  `WestwoodToIr.tileSet(vcn, vmp, palette)`; `Level1AppearancePack.build(script, tileset, pak3,
  pak4)` (in render-libgdx `…gl`). Canonical EOB1 LEVEL1: PAK `EOBDATA3.PAK`+`EOBDATA4.PAK`,
  `BRICK.PAL/VCN/VMP`.
- EOB2: loose files; `Maze.parse`, `Eob2InfScript.parse`; `Eob2InfGeometry.geometry(maze, script)`/
  `.doors(maze, script)`/`.decorations(maze, script)`; `WestwoodToIr.tileSet(DUNG.VCN, DUNG.VMP,
  DUNG.PAL)`; the M1 pack assembly `AppearancePackBuilder.build("dung", tiles,
  DecorationDat.parse(BROWN.DEC), Map.of("brown1", CpsImage.parse(BROWN1.CPS), "brown2",
  CpsImage.parse(BROWN2.CPS)), CpsImage.parse(DOOR1.CPS))` (overlay names from
  `script.decorationOverlays()`). Real EOB2 game dir: `~/Games/Eye of the Beholder II The Legend of
  Darkmoon.app/Contents/Resources/game`.

**Structural walk invariants (`WalkableLevelIT`, the shape the M2 walk tests reuse):** four
`TURN_RIGHT` restore start facing + position; 40× `FORWARD` never leaves `[0, 31]` and one more
`FORWARD` at a wall is a no-op. Driven via `state.applyCommand(cmd, level)` directly (no window).

**Module pom (from `beholder-render-core/pom.xml`):** `<parent>` =
`eu.virtualparadox.beholder:legend-of-the-beholder:0.1.0-SNAPSHOT`; intra deps spell out
`groupId`/`artifactId`/`version 0.1.0-SNAPSHOT`; `junit-jupiter`+`assertj-core` `scope test`, no
version. Root `pom.xml` `<modules>` gets `<module>beholder-app</module>` appended.

## File structure

- `beholder-app/pom.xml` **(new module)**; `src/main/java/eu/virtualparadox/beholder/app/`:
  `BootedGame.java` (record: level + state + overlays), `Boot.java` (`boot(...)`),
  `GameKind.java` (enum EOB1/EOB2), `GameDetector.java` (`detect(Path)`), `GameExtractor.java`
  (interface `IrDocument extract(Path gameFolder, String level)`), `Eob1Extractor.java`,
  `Eob2Extractor.java`, `Extract.java` (facade: detect → extractor → `IrJson.writeDocument`),
  `Main.java` (CLI `extract`/`play`). Test tree: `SyntheticIr.java` (a committed minimal valid
  `IrDocument` builder), `BootWalkTest.java` (synthetic, skip-free), `GameDetectorTest.java`,
  `Eob1ExtractIT.java`, `Eob2ExtractIT.java`, `Eob1BootWalkIdentityIT.java`,
  `Eob2BootWalkIT.java` (all real-data ITs skip-safe).
- `beholder-render-libgdx/.../gl/GameWindow.java` **(new)** — the generalized `play(BootedGame)`
  window; **delete** `LibGdxLauncher.java`. `beholder-render-libgdx/pom.xml` — the `exec`/`run`
  profile relocates to `beholder-app/pom.xml` (main class `…app.Main`).
- Root `pom.xml` — add `<module>beholder-app</module>`.

---

## Task 1: `beholder-app` module skeleton + pom + license gate green

**Retires the module/dependency/license risk before any logic is built.**

**Files:**
- Create: `beholder-app/pom.xml`, `beholder-app/src/main/java/eu/virtualparadox/beholder/app/package-info.java`
- Modify: root `pom.xml` (`<modules>` + `<module>beholder-app</module>`)
- Test: `beholder-app/src/test/java/eu/virtualparadox/beholder/app/ModuleCanaryTest.java`

**Interfaces produced:** a buildable `beholder-app` module in the reactor.

- [ ] **Step 1: Write `beholder-app/pom.xml`** — `<parent>` = the reactor root
  (`eu.virtualparadox.beholder:legend-of-the-beholder:0.1.0-SNAPSHOT`); `<artifactId>beholder-app`;
  intra deps (each with explicit `<version>0.1.0-SNAPSHOT</version>`): `beholder-importer-westwood`,
  `beholder-ir-json`, `beholder-engine-core`, `beholder-render-core`, `beholder-render-libgdx`;
  `junit-jupiter` + `assertj-core` (`scope test`, no version). Add `<module>beholder-app</module>`
  to the root `pom.xml` after `beholder-render-libgdx`.
- [ ] **Step 2: Write `package-info.java`** (Javadoc for the package — doclint requires it) and a
  trivial `ModuleCanaryTest` asserting a constant (so the module has a green test + coverage isn't
  zero-divided). RED not needed (canary).
- [ ] **Step 3: Verify the reactor + license gate.** `mvn -q verify` (root) → `BUILD SUCCESS`
  including `license:aggregate-add-third-party`. `beholder-app` adds no new third-party dep, so the
  gate stays green; if a transitive spelling trips it, add a `<licenseMerge>` under the root
  (normalisation only). Record the outcome.
- [ ] **Step 4: Commit** — `feat(app): beholder-app module skeleton + reactor wiring`.

---

## Task 2: `BootedGame` + `Boot` — game-agnostic boot from IR (+ the committed synthetic-IR walk test)

**The heart of the milestone: boot a level from IR alone and walk it, with ZERO game data.**

**Files:**
- Create: `beholder-app/.../app/BootedGame.java`, `.../app/Boot.java`
- Test: `beholder-app/src/test/java/.../app/SyntheticIr.java` (committed minimal IR builder),
  `.../app/BootWalkTest.java`

**Interfaces:**
- Consumes: `IrJson.readDocument(Path, String) -> IrDocument`; the boot-template signatures above.
- Produces: `record BootedGame(Level level, RuntimeState state, SceneOverlay overlays)`;
  `Boot.boot(Path irFolder, String baseName, GridPosition start, Facing facing) -> BootedGame`
  (reads the IR, builds `Level`/`RuntimeState`/`SceneOverlay` per the boot template + D-C live
  door test + `IntUnaryOperator.identity()` resolver — **no importer import**);
  `SyntheticIr.minimalDocument() -> IrDocument` (a committed, hand-authored minimal valid document:
  a small `LevelGeometry` with at least one OPEN and one SOLID edge, empty events, and a minimal
  valid `AppearancePack`).

- [ ] **Step 1: Build `SyntheticIr.minimalDocument()` (discovery against existing patterns).** A
  minimal valid `IrDocument`: `LevelGeometry.of(w,h, archetypes, wallSets, CURRENT_IR_VERSION)` (a
  tiny grid — e.g. 5×5 — with a border of SOLID and an interior of OPEN so a party can walk and hit
  a wall), `new LevelEvents(List.of(), List.of(), List.of())`, and a minimal valid `AppearancePack`
  (`AppearancePack.of(id, TileSet, DecorationTable, Map.of(), PixelSheet doorLeafSheet)`). **Reuse
  the synthetic-pack construction already in the codebase** — `beholder-importer-westwood`'s
  `AppearancePackBuilderTest`/`AppearancePackTest` and `beholder-ir-json`'s synthetic-golden test
  build minimal `TileSet`/`DecorationTable`/`PixelSheet` — mirror their factory calls (the exact
  minimal array shapes are what those tests already satisfy against the `of(...)` guards). Assert
  `new IrDocument(doc, pack)` constructs (fail-closed ctor passes).
- [ ] **Step 2: Write `BootWalkTest` (RED, skip-free, committed).** Write `SyntheticIr`'s document
  to a JUnit `@TempDir` via `IrJson.writeDocument(tmp, "synthetic", ir)`; `Boot.boot(tmp,
  "synthetic", interiorStart, Facing.NORTH)`; assert the returned `BootedGame.level().geometry()`
  round-trips the synthetic geometry (value-equal), and drive the **walk invariants** on
  `bg.state()`/`bg.level()` via `state.applyCommand(...)`: four `TURN_RIGHT` restore start
  facing+position; forward into the SOLID border is a no-op; forward through OPEN advances. Run →
  FAIL (no `Boot`).
- [ ] **Step 2b: Run — expect FAIL:** `mvn -q -pl beholder-app -am test -Dtest=BootWalkTest`.
- [ ] **Step 3: Implement `BootedGame` + `Boot.boot`** per the boot template (readDocument →
  `Level.of` + `RuntimeState.startingAt(Party.of(start,facing))` + `IrDecorationScene` +
  `IrDoorLeafScene` with the D-C live closed-test + `SceneOverlay` with the identity resolver).
  Small helpers to stay under the gates; every value from the IR only. Run → PASS.
- [ ] **Step 4: `mvn -q -pl beholder-app -am verify`; commit** — `feat(app): game-agnostic
  Boot from IR + committed synthetic-IR headless walk test (no game data)`.

---

## Task 3: `Eob1Extractor` — EOB1 game folder → `IrDocument` (base level)

**Files:**
- Create: `beholder-app/.../app/GameExtractor.java` (interface), `.../app/Eob1Extractor.java`
- Test: `beholder-app/.../app/Eob1ExtractIT.java` (real EOB1, skip-safe)

**Interfaces:**
- Produces: `interface GameExtractor { IrDocument extract(Path gameFolder, String level); }`;
  `Eob1Extractor implements GameExtractor` — reads `EOBDATA3.PAK`/`EOBDATA4.PAK` from `gameFolder`,
  runs the EOB1 chain (`Maze`/`InfScript`/`VgaPalette`/`VcnAtlas`/`VmpProjection` →
  `InfWallGeometry.geometry`/`.doors` + `DecorationPlacements.scan` + `WestwoodToIr.tileSet` +
  `Level1AppearancePack.build`), assembles `new LevelDocument(CURRENT_IR_VERSION, geometry, new
  LevelEvents(List.of(), doors, decorations), pack.id())` (**empty triggers — D-B**) + `new
  IrDocument(document, pack)`.

- [ ] **Step 1: `Eob1ExtractIT` (RED, real EOB1, skip-safe).** Reuse `RealLevel1Archives` /
  `assumeTrue` (game dir `~/Games/Eye of the Beholder.app/…/game`); `Eob1Extractor.extract(gameDir,
  "LEVEL1")`; assert: `doc.geometry().width()==32 && height()==32`; `doc.events().triggers()`
  **empty**; `doc.events().doors()` non-empty; `doc.events().decorations()` non-empty; `ir.pack().
  id()` equals `doc.appearancePackId()`; `new IrDocument(...)` constructed without throwing (all
  decorations resolve in the pack). Run → FAIL.
- [ ] **Step 2: Implement `Eob1Extractor`** (mirror `LosslessRoundTripIT.buildRealLevel1Ir` /
  `LibGdxLauncher.loadGame`, minus the triggers). Keep the file/PAK reads in small helpers. Run →
  PASS (must actually run on this machine — data present).
- [ ] **Step 3: `mvn -q -pl beholder-app -am verify`; commit** — `feat(app): Eob1Extractor →
  base-level IrDocument`.

---

## Task 4: `Eob2Extractor` — EOB2 loose folder → `IrDocument` (base level)

**Files:**
- Create: `beholder-app/.../app/Eob2Extractor.java`
- Test: `beholder-app/.../app/Eob2ExtractIT.java` (real EOB2, skip-safe)

**Interfaces:**
- Produces: `Eob2Extractor implements GameExtractor` — reads loose EOB2 files from `gameFolder`
  (`<level>.MAZ`, `<level>.INF`, `DUNG.VCN/VMP/PAL`, `BROWN1/2.CPS`, `BROWN.DEC`, `DOOR1.CPS`),
  runs the M1 EOB2 chain (`Maze`/`Eob2InfScript` → `Eob2InfGeometry.geometry`/`.doors`/
  `.decorations` + `WestwoodToIr.tileSet` + the M1 `AppearancePackBuilder.build` assembly),
  assembles the base `LevelDocument` (empty triggers) + `IrDocument`. The DUNG asset names come
  from the level's gfx-name slot + `script.decorationOverlays()`.

- [ ] **Step 1: `Eob2ExtractIT` (RED, real EOB2, skip-safe).** Game dir `~/Games/Eye of the
  Beholder II The Legend of Darkmoon.app/…/game`; `Eob2Extractor.extract(gameDir, "LEVEL1")`;
  assert: geometry 32×32; triggers empty; doors non-empty (the 2 `door1` doors placed in the maze);
  decorations non-empty with `overlayId` ∈ `{brown1, brown2}`; `IrDocument` constructs (fail-closed
  ctor passes — every decoration resolves in the DUNG pack). Run → FAIL.
- [ ] **Step 2: Implement `Eob2Extractor`** (mirror M1's `Eob2DecorationRenderIT` fixture assembly:
  parse the tileset from the `gfxName`, build the pack with the `brown1`/`brown2` sheets + `BROWN.DEC`
  + `DOOR1.CPS`). Run → PASS (must run — data present).
- [ ] **Step 3: `mvn -q -pl beholder-app -am verify`; commit** — `feat(app): Eob2Extractor →
  base-level IrDocument`.

---

## Task 5: `GameDetector` + `Extract` facade + `Main extract` CLI

**Files:**
- Create: `beholder-app/.../app/GameKind.java`, `.../app/GameDetector.java`, `.../app/Extract.java`,
  `.../app/Main.java`
- Test: `beholder-app/.../app/GameDetectorTest.java` (synthetic dirs), and an `extract`-to-folder
  assertion folded into `Eob1ExtractIT`/`Eob2ExtractIT` (write → reload).

**Interfaces:**
- Produces: `enum GameKind { EOB1, EOB2 }`; `GameDetector.detect(Path gameFolder) -> GameKind`
  (EOBDATA*.PAK → EOB1; loose `LEVEL1.INF` → EOB2; neither → `IllegalArgumentException`);
  `Extract.extract(Path gameFolder, String level, Path irOut, String baseName)` (detect → the
  matching `GameExtractor` → `IrJson.writeDocument(irOut, baseName, doc)`); `Main.main(String[])`
  dispatching `extract <game-folder> <level> <ir-out>`.

- [ ] **Step 1: `GameDetectorTest` (RED).** Two `@TempDir` folders — one containing an empty
  `EOBDATA3.PAK` file → `detect` returns `EOB1`; one containing an empty `LEVEL1.INF` → `EOB2`; an
  empty folder → `assertThatThrownBy(... ).isInstanceOf(IllegalArgumentException.class)`. (Detection
  keys on file **presence**, not content — cheap + robust.) Run → FAIL.
- [ ] **Step 2: Implement `GameKind`/`GameDetector`/`Extract` (all I/O-free + tested) + the `Main`
  entry.** `Extract.extract(...)` wires detect → extractor → `writeDocument` and is pure (returns/
  throws — no console I/O — so it is fully unit-tested). **CLI output vs the no-`System.out` gate
  (a real new constraint — the project has no prior CLI; the window uses `Gdx.app` logging):** keep
  the tested logic I/O-free; route the `Main` entry's usage/result text through an injected
  `java.io.PrintStream out` seam (a `main(String[])` that delegates to `run(String[] args,
  PrintStream out)`), so `run` is testable with a `ByteArrayOutputStream`. First **check whether the
  `System.out` gate (a Checkstyle/PMD `RegexpSingleline`-style rule) permits the single
  `Main.main`'s `run(args, System.out)` call**; if it does not, add the `app` CLI entry to a module
  JaCoCo/gate exclusion mirroring render-libgdx's `gl/**` window-glue exclusion (the `main` bootstrap
  is untestable-entry glue, like the window). Record which applied. Run → PASS.
- [ ] **Step 3: Extend `Eob1ExtractIT`/`Eob2ExtractIT`** with a `Extract.extract(...)` →
  `IrJson.readDocument(...)` round-trip into git-ignored `target/`: the reloaded `IrDocument`
  `.equals` the extracted one (the app-level round-trip; leans on M5). Run.
- [ ] **Step 4: `mvn -q -pl beholder-app -am verify`; commit** — `feat(app): game auto-detect +
  Extract facade + `extract` CLI`.

---

## Task 6: `GameWindow` + `Main play` + retire `LibGdxLauncher` (the EOB1 rewire, D-E)

**Files:**
- Create: `beholder-render-libgdx/.../gl/GameWindow.java` (the generalized `play(BootedGame)`)
- Modify: `beholder-app/.../app/Main.java` (add `play <ir-folder>`), `beholder-app/pom.xml` (the
  `exec`/`run` profile, main class `…app.Main`), `beholder-render-libgdx/pom.xml` (remove its
  launcher `exec` profile)
- **Delete:** `beholder-render-libgdx/.../gl/LibGdxLauncher.java`

**Interfaces:**
- Produces: `GameWindow.play(BootedGame game)` — opens the LWJGL3 window, builds `new GameSession(
  game.level(), game.state(), new LibGdxInputSource(new InputMapper()), new GlFrameSink(),
  game.overlays())`, ticks per frame + draws the HUD (transcribe `LibGdxLauncher.EobApplicationAdapter`'s
  window/threading logic, minus the EOB1 `loadGame`/`buildLevel`/demo triggers). `Main play
  <ir-folder>` = `Boot.boot(...)` (per-game default start; see D-A) → `GameWindow.play(...)`.

- [ ] **Step 1: Create `GameWindow`** by moving `LibGdxLauncher.EobApplicationAdapter`'s window +
  macOS-threading logic into a `play(BootedGame)` entry (drop `create()`'s EOB1 loading — the
  `BootedGame` is supplied). Keep the `gl/**` JaCoCo exclusion (untestable window glue).
- [ ] **Step 2: Wire `Main play`** — `Boot.boot(irFolder, baseName, DEFAULT_START.get(kind),
  DEFAULT_FACING)` → `GameWindow.play(bg)`. Move the `exec`/`run`(-macos) profile from
  render-libgdx's pom to app's pom, main class `eu.virtualparadox.beholder.app.Main`.
- [ ] **Step 3: Delete `LibGdxLauncher`**; ensure the reactor compiles (no dangling refs;
  `Level1AppearancePack` stays and is used by `Eob1Extractor`).
- [ ] **Step 4: `mvn -q verify` (root, all modules); commit** — `feat(app,render-libgdx): GameWindow
  frontend + `play` CLI; retire LibGdxLauncher's in-process EOB1 loading`.

---

## Task 7: EOB1 extract→boot→walk + **render-identity net** IT (skip-safe)

**Proves the EOB1 rewire is behaviour-preserving.**

**Files:**
- Test: `beholder-app/.../app/Eob1BootWalkIdentityIT.java` (real EOB1, skip-safe)

**Interfaces:** consumes `Extract`/`Boot`; the EOB1 in-process render for the identity reference.

- [ ] **Step 1: Write `Eob1BootWalkIdentityIT` (real EOB1, skip-safe).** (a) `Extract.extract(EOB1
  dir, "LEVEL1", target/, "level1")` → `Boot.boot(...)` at the canonical EOB1 start
  (`GridPosition.of(10,15)`, `Facing.NORTH` — kyra `_currentBlock=490`); drive the walk invariants
  on the booted state. (b) **Render-identity:** build the EOB1 entry frame two ways and assert
  pixel-equal — (i) the booted `GameSession` first frame, and (ii) a reference frame built the
  pre-M2 in-process way *inside the test* (`InfWallGeometry.geometry`/`.doors` +
  `Level1AppearancePack.build` + `Level.of` + one `GameSession.tick`, exactly as `LibGdxLauncher.
  buildLevel` did, minus triggers). Assert `bootFrame` equals `referenceFrame` byte-for-byte. This
  is the safety net that the rewire changed nothing the static render can see (it leans on M5's
  proven `extract→IR→reload` render-identity). Run (must not skip).
- [ ] **Step 2: `mvn -q -pl beholder-app -am verify`; commit** — `test(app): EOB1 boot-from-IR
  render-identity net + walk invariants`.

---

## Task 8: EOB2 extract→boot→walk IT + rendered eyeball (skip-safe) — the milestone end state

**Files:**
- Test: `beholder-app/.../app/Eob2BootWalkIT.java` (real EOB2, skip-safe)

**Interfaces:** consumes `Extract`/`Boot`; drives the booted EOB2 `GameSession` headlessly.

- [ ] **Step 1: Write `Eob2BootWalkIT` (real EOB2, skip-safe).** `Extract.extract(EOB2 dir,
  "LEVEL1", target/, "level1")` → `Boot.boot(...)` at a documented interior start facing walls
  (reuse M1 Task 6's `(5,4)` derivation or re-derive from the booted geometry). Assert: (a) the walk
  invariants (four-turn identity, never-off-grid, settle-at-wall) on the booted EOB2 state; (b)
  **door collision** — a `FORWARD` into a CLOSED door edge is a no-op (the door blocks); (c) render
  a scripted path through a `GameSession` with a `RecordingFrameSink`, write the frames + a contact
  sheet to git-ignored `target/` for the **maintainer eyeball** (decoration occlusion + the DUNG
  walk). Assert the frames are not all identical. Run (must not skip).
- [ ] **Step 2: `mvn -q verify` (root); commit** — `feat(app): EOB2 walkable through boot-from-IR
  (walk invariants + door collision + rendered eyeball)`.

---

## Self-review (writing-plans)

- **Spec coverage (design §2–§6):** the `beholder-app` module + `extract`/`play` CLI → Tasks 1,5,6;
  game-agnostic `boot(IR)` → Task 2; EOB2 extract → Task 4; EOB1 rewire to boot-from-IR → Tasks
  3,6,7; the walkable proof (structural invariants + door collision + decoration occlusion) →
  Tasks 2,7,8; the synthetic-IR headless boot (no game data) → Task 2; the EOB1 render-identity net
  → Task 7. All in scope covered. Out-of-scope (triggers, runtime/save, multi-level, the M4
  round-trip/kyra-diff gates) is honored — D-B keeps triggers empty for both games.
- **One-IR boundary:** `Boot`/`BootedGame`/`GameWindow` import **only** `beholder-ir`/`-json`/
  engine/render types — no Westwood decoder. Only the two `*Extractor`s import the importer. ✔
- **Placeholder scan:** the synthetic-pack construction (Task 2 Step 1) and the extractor
  assemblies (Tasks 3–4) are discovery-altitude against named existing patterns + real-data
  differentials (the P1/M1 convention), not "TBD"; the CLI I/O seam (Task 5 Step 2) names the
  constraint (no `System.out` in production) and the resolution shape. No `TODO`/"add error
  handling".
- **Type consistency:** `BootedGame(level, state, overlays)`, `Boot.boot(Path, String,
  GridPosition, Facing)`, `GameExtractor.extract(Path, String) -> IrDocument`,
  `Extract.extract(Path, String, Path, String)`, `GameDetector.detect(Path) -> GameKind`,
  `GameWindow.play(BootedGame)` — consistent across Tasks 2–8. `IrJson.writeDocument(dir, baseName,
  doc)` / `readDocument(dir, baseName)` used consistently.
- **D-A/D-B (deviations from the design's leans) are documented** for the maintainer's plan-review:
  boot-param start (not IR-carried) and empty EOB1 triggers (base level) — both were the design's
  explicit "simpler"/maintainer-chosen options.
