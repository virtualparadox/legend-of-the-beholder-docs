# P2 · M1 — EOB2 static render — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> to implement this plan task-by-task (fresh implementer + independent review per task,
> a whole-branch review at the end, `--no-ff` merge on the maintainer's eyeball verdict;
> no worktree). TDD per task (RED → GREEN → commit). Steps use checkbox (`- [ ]`) syntax.
> Style follows the P1 milestone plans: the decode/geometry/render tasks are *discovery
> against real data + a read-only oracle* (the "Design facts" below + each task's
> differential are the concrete spec); every task obeys the Global Constraints.

**Goal:** Render a real EOB2 level's first-person view headlessly, end-to-end through the
*shared* IR — decode EOB2 assets (reused decoders), lift the EOB2 `.INF` **geometry
declaration** into the shared `LevelGeometry`/`Door`/`Decoration`, wire the EOB2 tileset
into a `TileSet`/`AppearancePack`, and produce the `176x120` ARGB frame via the existing
`FirstPersonRenderer`. First milestone of P2; opens the second Westwood importer path.

**Architecture:** A new EOB2-specific `.INF` geometry-header parser (`Eob2InfScript` +
`Eob2InfGeometry`) lives **alongside** the EOB1 `Inf*` chain in `beholder-importer-westwood`
and emits **only** the shared `beholder-ir` values — the parallel-importer-paths / one-IR
shape fixed in the P2 design. The Westwood decoders (`Maze`, `VcnAtlas`, `VmpProjection`,
`VgaPalette`, `CpsImage`) and the render/appearance assembly (`WestwoodToIr.tileSet`,
`AppearancePackBuilder`, `FirstPersonRenderer.render`) are **reused unchanged** on EOB2's
loose-file data. `WallArchetype` grows only where real `LEVEL1` forces it.

**Tech Stack:** Java 21, Maven multi-module, JUnit 5 + AssertJ. Read-only oracles: kyra
`EoBCoreEngine`/`EoBInfProcessor`/`darkmoon.cpp` + `eob2_dos.h`. Real data: the user's own
loose EOB2 files under `~/Games/Eye of the Beholder II The Legend of Darkmoon.app/…/game`.

## Global Constraints

- **JDK 21** (`JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home`);
  `mvn spotless:apply` then `mvn verify`; single-module `-pl <m> -am` **or** root verify (the
  stale `~/.m2` upstream-jar trap).
- **All `CODING_STANDARDS` gates green:** one `return`/method (`OnlyOneReturn`), no
  `instanceof` (use `getClass()`/polymorphic dispatch), no `switch yield`, cyclomatic ≤ 8
  (PMD counts boolean ops), NPath ≤ 120, ≤ 12 fields / ≤ 18 methods / ≤ 5 effective params,
  coupling ≤ 10, no `return null` (Optional/throw/empty-collection), params `final` + not
  reassigned, braces on all control-flow, `L` for longs, ≤ 140-char lines, no `System.out`/
  `printStackTrace`, NullAway `ERROR` for `eu.virtualparadox`, Javadoc `doclint=all` (every
  public/protected type + method, `@param`/`@return`), **every magic value a named
  `ALL_CAPS` constant + a what-and-why comment** (clean-room formats).
- **Coverage: JaCoCo ≥ 0.90 branch AND line at bundle level.** Exempt: `**/model/**`,
  `**/*Record.class`, `**/dto/**` (and the render-libgdx `gl/` composition-glue exclusion).
  New IR value types go under `ir/model/` (exempt) but their **guards + value-equality are
  behaviour-tested**; the importer's parse/lift logic is **not** exempt and must be
  branch-covered by synthetic fixtures.
- **Clean-room:** kyra (`~/Workspaces/CPP/ScummVM/engines/kyra`) + `eoblib` are **read-only
  oracles** — never copy. jCodeMunch local-only. **Never commit game data:** real-data ITs
  are `assumeTrue`-skip-safe; any rendered PNG/asset goes to **git-ignored `target/`**. A
  committed golden is our own synthetic content only.
- **Package roots** `eu.virtualparadox.beholder.{importer.westwood,ir,render}`.
- Branch `feat/p2-m1-eob2-static-render`; commit per task; whole-branch review + `--no-ff`
  merge at the end. Base = current `main` (post-P1, `926cf31`).

---

## Plan-level design decisions (the shapes the design §3 deferred to the plan)

- **D-A · Parallel EOB2 classes, shared IR.** The EOB2 `.INF` geometry parse lives in new
  `Eob2InfScript` (the raw block-A decoder) + `Eob2InfGeometry` (the maze+script → shared IR
  assembler), mirroring the EOB1 `InfScript` + `InfWallGeometry` split. They emit **only**
  `LevelGeometry`/`Door`/`Decoration`/`WallArchetype` — **zero** EOB2-specific IR types.
- **D-B · Loose-file loader, no PAK.** EOB2 ships loose files (no `EOBDATA*.PAK`), so the M1
  read path is `Files.readAllBytes(gameDir.resolve("LEVEL1.INF"))` etc. — not `PakArchive`.
  A test-only skip-safe helper `RealEob2Level1` mirrors `RealLevel1Archives` but returns raw
  `byte[]` from the EOB2 game dir (`~/Games/Eye of the Beholder II The Legend of
  Darkmoon.app/Contents/Resources/game`).
- **D-C · Reuse the decoders + tileset + renderer verbatim.** `Maze.parse`, `VcnAtlas.parse`,
  `VmpProjection.parse`, `VgaPalette.parse`, `CpsImage.parse`, `WestwoodToIr.tileSet`,
  `AppearancePackBuilder.build`, and `FirstPersonRenderer.render(LevelGeometry, TileSet,
  GridPosition, Facing)` are used **unchanged**. If any decoder rejects real EOB2 bytes, that
  is a finding to document, not a silent workaround (fail-closed, clean-room discipline).
- **D-D · Static render only — no engine.** M1 renders via the 4-arg
  `FirstPersonRenderer.render(geo, tiles, party, facing)` directly (no `GameSession`/
  `RuntimeState`/movement — that is M2). Decorations/door-leaves may be wired via the IR-fed
  `IrDecorationScene`/`IrDoorLeafScene` overloads if `LEVEL1`'s geometry declares them, but
  the milestone's core golden is the plain wall+tileset frame.
- **D-E · `WallArchetype` extension is data-driven and shared.** M1 extends the shared
  `WallArchetype` enum **iff** real `LEVEL1` bytes contain an archetype the current
  `{OPEN, SOLID, DOOR}` cannot represent (the §3.4-bounded candidate is `WALL_OF_FORCE`; a
  clickable wall may instead map to `SOLID`/`DOOR` + a later `CLICKED` trigger — decided from
  the data in Task [geometry], with the decision recorded in that task's outcome).
- **D-F · Message model / events are OUT of M1.** M1 parses only block A (the declarative
  geometry header). Block B (script) + block C (block→script binding) are M3. The `.INF`
  string table and triggers are not read here.

## Design facts (reusable interfaces — grounded in the current code)

**Reused decoders (all `public static … parse(final byte[])`):**
- `Maze.parse(byte[]) -> Maze`; `.width()`, `.height()`, `.wall(x,y,dir)` (dir 0..3 = N/E/S/W);
  confirmed EOB2 `LEVEL1.MAZ` header `32/32/4`, 4102 B — identical to EOB1.
- `VcnAtlas.parse(byte[]) -> VcnAtlas`; `VmpProjection.parse(byte[]) -> VmpProjection`;
  `VgaPalette.parse(byte[]) -> VgaPalette`; `CpsImage.parse(byte[]) -> CpsImage`.
- `WestwoodToIr.tileSet(VcnAtlas, VmpProjection, VgaPalette) -> TileSet` — reused verbatim.

**IR targets (shared, unchanged):**
- `LevelGeometry.of(int width, int height, WallArchetype[] archetypes, int[] wallSets,
  String irVersion) -> LevelGeometry` — flattened row-major `(y*width+x)*4+dir`; an
  `OPEN` edge MUST carry `wallSet == 0`; version `LevelGeometry.CURRENT_IR_VERSION = "1.0.0"`.
- `WallArchetype` = `{ OPEN, SOLID, DOOR }` today (closed enum; D-E may add one member).
- `Door(GridPosition block, int edge, Door.DoorState state, int doorGraphicId)`; edge 0..3;
  M1 doors start `Door.DoorState.CLOSED` (base state; runtime overlay is M2+).
- `Decoration(GridPosition block, int edge, String overlayId, int decorationId)`; edge 0..3.
- `AppearancePack.of(String id, TileSet, DecorationTable, Map<String,PixelSheet>
  decorationSheets, PixelSheet doorLeafSheet)`; `AppearancePackBuilder.build(String id,
  TileSet, DecorationDat, Map<String,CpsImage> decorationCps, CpsImage doorCps)`.
- `GridPosition.of(int x, int y)`; `Facing` = `{ NORTH, EAST, SOUTH, WEST }` with `.right()`.

**Render entry (reused verbatim):**
- `FirstPersonRenderer.render(LevelGeometry geo, TileSet tiles, GridPosition party, Facing
  facing) -> int[]` — a fresh `176*120` packed-ARGB frame (`VIEWPORT_WIDTH=176`,
  `VIEWPORT_HEIGHT=120`). Decoration/door-leaf overloads exist but are optional for M1.

**EOB1 parallels the EOB2 path mirrors (read-only reference, not reused):**
- `InfScript.parse(byte[])` with `wallMapping(int)`, `hasWallMapping(int)`, `mazeName()`,
  `vcnVmpName()`, `overlays() -> List<InfOverlay>`, `decorationOverlay(int)`; `InfWallMapping(
  int wallType, int decorationId, int eventMask, int flags)`; `InfOverlay(String cpsName,
  String datName)`.
- `InfWallGeometry.geometry(Maze, InfScript) -> LevelGeometry` +
  `.doors(Maze, InfScript) -> List<Door>` — the exact structural template for
  `Eob2InfGeometry.geometry(...)` / `.doors(...)`.
- Render-golden pattern: `LevelOneEntryRenderIT` (real EOB1) — builds Maze/InfScript/palette,
  renders each facing to `int[]`, writes PNGs to git-ignored `target/`, asserts facings
  differ. The EOB2 M1 IT mirrors it with `FirstPersonRenderer.render` (no `GameSession`).
- Skip-safe real-data helper: `RealLevel1Archives.assumeReadable(name)` (EOB1 PAK). The EOB2
  analog `RealEob2Level1` resolves loose files under the EOB2 game dir + `assumeTrue`.

**EOB2 `.INF` block-A grounding — byte-level (kyra `initLevelData` + real `LEVEL1.INF`,
reproduced against the actual bytes).**

- **CRITICAL: the `.INF` file is itself a CPS/format80-compressed wrapper**, not raw level
  data. Raw header @0: `[u16 fileSize-2][u16 compType=4=format80][u32 uncompressedSize][u16
  palSize=0][compressed data @10]`. Real `LEVEL1.INF` (4712 B) → decompresses to **exactly
  7462 B**. **Every block-A offset below is into the DECOMPRESSED buffer.** (The raw bytes
  `…02 ec level1.maz…` I earlier read as "block A" were a format80 literal-run coincidence —
  do NOT parse block A from the raw file. Reuse `WestwoodContainer.decompress(byte[])`, or
  `Format80Decompressor.decompress(raw, 10, u32LE@4)`; assert the result is 7462 B.)
- **Three-block layout of the decompressed buffer:** `[0..1] u16 = 1037` → Block B offset;
  `[2..1036]` **Block A** (declarative sub-level headers; `initLevelData` parses from off 2);
  `[1037..7075]` Block B (`[u16 = 7076 → Block C off]` + optional `0xEC` monster block + INF
  script); `[7076..7461]` Block C (`[u16 count=64][count × (u16 block, u16 flags, u16 objs)]`,
  6-byte stride — EOB2 `uint16` flags vs EOB1's `uint8`). Blocks B/C are **M3**, not M1.
- **Markers:** `0xEC` = "section present"; `0xEA` = alternate initial door-state (open);
  `0xFF`/`0xFFFF` = "absent/none". Read as raw `uint8`, never dispatched. `slen` (name-slot
  width) = **13** for EOB2 (12 for EOB1).
- **Block A per-sub-record walk** (decompressed buffer `buf`, `data = buf+2`; parser
  cheat-sheet — the concrete spec for Task 2/3):
  ```
  Sub record @p: nextLink = u16LE(p) [rel to data]; p += 2; assert buf[p++] == 0xEC
    MAZ name  = cstr13(p);  p += 13          // "level1.maz"
    GFX name  = cstr13(p);  p += 13          // "dung"  — VMP + PAL/EGA + VCN base (ONE shared slot)
    if buf[p++] != 0xFF: PAL2 = cstr13(p); p += 13   // absent in LEVEL1 (gate byte 0xFF)
    SOUND name= cstr13(p);  p += 13          // "catacomb"
    for slot in 0,1:                          // doors
        t = buf[p++]
        if t in {0xEC,0xEA}: file=cstr13(p); wallId=buf[p+13]; type=buf[p+14]; noSw=buf[p+15]; p += 16+48
        // t==0xEC → initial CLOSED, t==0xEA → OPEN; else (0xFF) empty, just the 1 tag byte
    steps = u16LE(p); p += 2                  // stepsUntilScriptCall
    for slot in 0,1: if buf[p++]==0xEC { mult=buf[p]; idx=buf[p+1]; name=cstr13(p+2); flag=buf[p+15]; p += 16 }
    monsterProps(p)  // skip per darkmoon.cpp:336-413: [u8 id]; while id!=0xFF { 16 fixed + 4×u16
                     //   + [u8,s8,s8] + [u8 nRemote] + gate(if≠0xFF: [u8][u8 nW][nW×2]) + [s8,u8]
                     //   + [3×u8 deco] + next id }   (M1 skips this section; monsters are out of scope)
    if buf[p++]==0xEC {                        // wall/decoration block
        num = u16LE(p); p += 2
        repeat num: d = buf[p++]
            if d==0xEC: cps=cstr13(p); dec=cstr13(p+13); p += 26         // decoration-load
            else: wall=buf[p]; vmp=buf[p+1]; decIdx=(s8)buf[p+2]; special=buf[p+3]; flags=buf[p+4]; p += 5
    }
    scriptTimers: while (s16)u16LE(p) != -1: p += 4
  ```
  `cstr13(p)` = read up to the first `\0` within a fixed 13-byte field (trailing bytes are
  format80 garbage — ignore them). The sub-record chain: sub 0 @ data(off 2) links to sub 1
  @ off 743, which links to off 1037 (= Block B, chain end). LEVEL1 has **2 sub-levels**;
  `loadLevel` re-applies `initLevelData(i)` cumulatively for `i=0..requested`, so name/tileset
  state ends on the requested sub. Wall geometry itself is `LEVEL1.MAZ` (reused `Maze`).
- **Wall-assign semantics** (`assignWallsAndDecorations`): `_wllVmpMap[wall]=vmp` (VMP
  template → the IR `wallSet`), `_specialWallTypes[wall]=special`, `_wllWallFlags[wall]=flags^4`
  (runtime flag = on-disk `flags` XOR 4), decoration linked-list from `decIdx` (`-1`/`0xFF` =
  none). In LEVEL1 the leading discriminator `d` is always `0xFB` for wall-assigns.
- **Doors:** a door record's `wallId` (0,1 in LEVEL1) expands via `toggleWallState` to the
  wall-index band `wallId*10 + 3 + {0,1,2,3,5,6,7,8}` (i.e. `+0..+8` skipping `+4`): door 0 →
  indices {3,4,5,6,8,9,10,11}, door 1 → {13,14,15,16,18,19,20,21} (the door's open/close
  animation frames). A maze cell whose wall-index is in a declared door's band is a
  `WallArchetype.DOOR` edge.

**Concrete real-`LEVEL1` differential values (the agent verified these — use as test oracles):**
- Decompressed size 7462 B; 2 sub-levels; block-B off 1037, block-C off 7076.
- Names: MAZ `level1.maz`, GFX/tileset **`dung`** (`DUNG.VCN` 41549 B / `DUNG.VMP` 5834 B /
  `DUNG.PAL` 768 B — all loose in the game dir; NOT crimson/mezz), sound `catacomb`.
- Doors: **2**, file `door1` (`DOOR1.CPS`), both initial tag `0xEC` (CLOSED), wall-ids 0 and 1.
- Decoration-loads: **2** — `brown1.cps`+`brown.dec`, `brown2.cps`+`brown.dec`.
- Wall/deco block: sub0 `num=33` (2 deco-loads + 31 wall-assigns), sub1 `num=24` (2 + 22).
  Sample sub0 wall-assigns `(wall,vmp,dec,special,flags)`: `(27,1,18,3,04)`, `(28,1,19,4,04)`,
  `(44,0,-1,0,07)` (teleporter id 44, plain), `(45,1,13,10,84)` (niche), `(49,1,44,2,04)`
  (button), `(74,0,-1,0,04)` (wall-of-force id 74, plain). **Wall-of-force (74) is a
  spell-created dynamic wall — it is NOT expected in the static `LEVEL1.MAZ`; M1 need not
  render it** (confirm no maze cell uses index 74 in Task 4).
- **`WallArchetype` decision (D-E), pre-resolved from the data:** the specialTypes LEVEL1
  uses (1 door-switch, 2/8 button, 3/4 lever, 10 niche, 0xFF auto) are **clickable behaviour**
  (M3 events) and/or **decorations** (their `decIdx` shape), **not** distinct render geometry —
  they render as `SOLID` + a `Decoration`. The one id-keyed render special (wall-of-force 74)
  is dynamic and absent from the static maze. **Therefore M1 keeps `WallArchetype =
  {OPEN, SOLID, DOOR}` unchanged** (archetype = OPEN if wall-index 0, DOOR if in a door band,
  else SOLID). Task 4 confirms this against the real maze; a genuine counterexample (a static
  maze cell that only a new archetype can render) would reopen it — none is expected.

## File structure

- `beholder-importer-westwood/.../importer/westwood/` **(new EOB2 classes)** —
  `Eob2InfScript.java` (raw block-A geometry-header decoder: name slots, wall-mapping table,
  doors, decorations), `Eob2InfGeometry.java` (maze+script → shared `LevelGeometry` + `doors`
  + `decorations`), `Eob2WallMapping.java` (the EOB2 wall-mapping record, EOB2's analog of
  `InfWallMapping`).
- `beholder-ir/.../ir/WallArchetype.java` — **modified only if** D-E's data-driven check
  forces a new member.
- Test tree — `Eob2InfScriptTest`/`Eob2InfGeometryTest` (synthetic byte fixtures),
  `Eob2InfGeometryIT` (real LEVEL1, skip-safe), and the render-golden
  `Eob2LevelOneRenderIT` (real LEVEL1 → `FirstPersonRenderer.render` → PNG to `target/`),
  plus the `RealEob2Level1` skip-safe loose-file helper.

---

## Task 1: `RealEob2Level1` skip-safe loose-file helper + reused-decoder validation

**Retires the "do the shared decoders accept EOB2 data?" (D-C) risk first, before any EOB2
parser is built on them.**

**Files:**
- Create (test): `beholder-importer-westwood/src/test/java/.../westwood/RealEob2Level1.java`
- Create (test): `beholder-importer-westwood/src/test/java/.../westwood/Eob2DecoderReuseIT.java`

**Interfaces:**
- Produces: `RealEob2Level1.assumeReadable(String fileName) -> byte[]` — resolves `fileName`
  under `~/Games/Eye of the Beholder II The Legend of Darkmoon.app/Contents/Resources/game`,
  `assumeTrue(Files.isReadable(...))`-skips if absent, else returns `Files.readAllBytes(...)`.
  Mirrors `RealLevel1Archives` (EOB1) but returns raw bytes (loose files, no PAK).

- [ ] **Step 1: Write `RealEob2Level1`** — the game dir as a `Path.of(System.getProperty(
  "user.home"), "Games/Eye of the Beholder II The Legend of Darkmoon.app/Contents/Resources/
  game")` constant; one `static byte[] assumeReadable(String)` throwing `IOException`, with a
  `assumeTrue(..., "EOB2 game data present at " + path)` skip message (copy the EOB1 helper's
  shape). No test yet (a helper).
- [ ] **Step 2: Write `Eob2DecoderReuseIT` (RED)** — one `@Test` that:
  `Maze.parse(RealEob2Level1.assumeReadable("LEVEL1.MAZ"))` → assert `width()==32 &&
  height()==32`; `VgaPalette.parse(assumeReadable("DUNG.PAL"))` → assert 256 colours resolvable
  (`palette.argb(255)` does not throw); `VcnAtlas.parse(assumeReadable("DUNG.VCN"))` → assert
  `tileCount() > 0`; `VmpProjection.parse(assumeReadable("DUNG.VMP"))` → assert
  `wallSetCount() > 0`. Run: `mvn -q -pl beholder-importer-westwood test
  -Dtest=Eob2DecoderReuseIT`.
- [ ] **Step 3: Run — it PASSES immediately if the decoders accept EOB2 data** (they should —
  same Westwood formats). If any parser throws on real EOB2 bytes, **stop and document the
  exact failure** (a real format delta, not a workaround) — that becomes a new finding and
  possibly a new task; do not silently adapt.
- [ ] **Step 4: `mvn -q -pl beholder-importer-westwood verify`; commit** —
  `test(p2-m1): skip-safe EOB2 loose-file helper + reused-decoder validation IT`.

---

## Task 2: `Eob2InfScript` — CPS-unwrap + Block-A name slots

**Files:**
- Create: `beholder-importer-westwood/.../westwood/Eob2InfScript.java`
- Test: `.../westwood/Eob2InfScriptTest.java` (synthetic decompressed buffers),
  `.../westwood/Eob2InfScriptIT.java` (real LEVEL1, skip-safe)

**Interfaces:**
- Consumes: `WestwoodContainer.decompress(byte[]) -> byte[]` (CPS/format80 unwrap).
- Produces: `Eob2InfScript.parse(byte[] infBytes) -> Eob2InfScript`; package-private
  `Eob2InfScript.fromDecompressed(byte[] page5) -> Eob2InfScript` (the test seam — lets unit
  tests hand-build a decompressed buffer with **no** compression); accessors `mazeName()`,
  `gfxName()`, `soundName()` (all `String`), `subLevelCount() -> int`.

- [ ] **Step 1: `Eob2InfScriptTest` (RED)** — build a synthetic decompressed `byte[]` (helper
  in the test): `u16LE(1037)` at 0, then at offset 2 one sub-record = `u16LE(link)`,
  `0xEC`, `cstr13("maze.maz")`, `cstr13("gfx")`, `0xFF` (no 2nd pal), `cstr13("snd")`, then a
  self-referential/terminal link so `subLevelCount()==1`. Assert `fromDecompressed(buf)` yields
  `mazeName()=="maze.maz"`, `gfxName()=="gfx"`, `soundName()=="snd"`, `subLevelCount()==1`. Run
  → FAIL.
- [ ] **Step 2: Implement `Eob2InfScript`** — `parse(raw)` = `fromDecompressed(
  WestwoodContainer.decompress(raw))`; `fromDecompressed` walks the cheat-sheet's name-slot
  prefix (block B off from `u16LE@0`; `data = 2`; per sub-record: skip link u16, assert
  `0xEC`, read the 3 (or 4) 13-byte `cstr13` slots + the 2nd-pal gate). Follow the sub-record
  link chain until it reaches the block-B offset, counting sub-levels; keep the **last**
  sub-record's names (the `loadLevel` cumulative-apply semantics). Named constants for every
  magic value (`NAME_SLOT_LEN=13`, `SECTION_PRESENT=0xEC`, `ABSENT=0xFF`, `BLOCK_B_OFFSET_POS=0`,
  `BLOCK_A_START=2`) + what/why comments. Run → GREEN.
- [ ] **Step 3: `Eob2InfScriptIT` (real LEVEL1, skip-safe)** — `Eob2InfScript.parse(
  RealEob2Level1.assumeReadable("LEVEL1.INF"))`; assert `mazeName().equals("level1.maz")`,
  `gfxName().equals("dung")`, `soundName().equals("catacomb")`, `subLevelCount()==2`. Also
  assert `WestwoodContainer.decompress(rawInf).length == 7462` (the decompression sanity). Run.
- [ ] **Step 4: `mvn -q -pl beholder-importer-westwood verify`; commit** —
  `feat(p2-m1): Eob2InfScript CPS-unwrap + block-A name slots`.

---

## Task 3: `Eob2InfScript` — block-A wall-assign table, doors, decoration-loads

**Files:**
- Modify: `.../westwood/Eob2InfScript.java` (add the geometry-block parse); Create:
  `.../westwood/Eob2WallMapping.java`, `.../westwood/Eob2DoorDecl.java`
- Test: extend `Eob2InfScriptTest`, `Eob2InfScriptIT`

**Interfaces:**
- Produces (records): `Eob2WallMapping(int vmpIndex, int decIndex, int specialType, int flags)`
  — `decIndex == -1` means no decoration; `Eob2DoorDecl(String shapeFile, int wallId, int
  doorType, boolean initiallyOpen)`.
- Produces (on `Eob2InfScript`): `wallMapping(int wallIndex) -> Eob2WallMapping` (fail-closed
  on an unassigned index — see `hasWallMapping`), `hasWallMapping(int wallIndex) -> boolean`,
  `doors() -> List<Eob2DoorDecl>`, `decorationOverlays() -> List<InfOverlay>` (reusing the
  existing `InfOverlay(String cpsName, String datName)` — here `(cps, dec)`).

- [ ] **Step 1: Extend `Eob2InfScriptTest` (RED)** — grow the synthetic buffer past the name
  slots per the cheat-sheet: 2 door slots (one `0xEC` door `cstr13("door1")` + `wallId=0` +
  `type`/`noSw` + 48 zero bytes; one `0xFF` empty), `u16LE(steps)`, 2 monster-shape slots (both
  `0xFF` empty — the simplest skip), a minimal `monsterProps` = just `0xFF` (empty, id==0xFF
  terminator), then the `0xEC` wall/deco block `u16LE(num=2)` = one deco-load
  (`0xEC cstr13("brown1") cstr13("brown")`) + one wall-assign (`0xFB 27 1 18 3 4`), then
  script-timers `0xFFFF`. Assert: `doors()` has 1 entry `("door1", 0, …, false)`;
  `hasWallMapping(27)` true, `wallMapping(27)` == `(vmp 1, dec 18, special 3, flags 4)`;
  `decorationOverlays()` == `[("brown1","brown")]`. Run → FAIL.
- [ ] **Step 2: Implement the geometry-block parse** — continue `fromDecompressed` past the
  name slots exactly per the cheat-sheet: doors (2 slots; `0xEC`→closed/`0xEA`→open door record
  of `1+16+48` bytes, else 1 byte), `steps` u16, monster-shape slots (2× `0xEC`→16-byte skip /
  else 1-byte), `monsterProps` skip-parse (walk `[u8 id]` records until `id==0xFF`, per the
  cheat-sheet's field widths — M1 only skips them), then the `0xEC` wall/deco block: `u16 num`,
  then `num` entries branching on the discriminator (`0xEC`→26-byte deco-load into
  `decorationOverlays`, else→5-byte wall-assign into a `Map<Integer,Eob2WallMapping>`, casting
  `decIdx` as a signed byte). Keep per-section helpers small (cyclomatic/param gates). Every
  offset a named constant + comment. Run → GREEN.
- [ ] **Step 3: Extend `Eob2InfScriptIT` (real LEVEL1)** — assert on the **agent-verified**
  oracle values: `doors().size()==2`, both `shapeFile().equals("door1")`, wallIds `{0,1}`, both
  `!initiallyOpen()`; `decorationOverlays()` contains `("brown1","brown")` and
  `("brown2","brown")`; `hasWallMapping(27)` and `wallMapping(27)` == `(1,18,3,4)`;
  `wallMapping(44)` == `(0,-1,0,7)`; `wallMapping(74)` == `(0,-1,0,4)`; the wall-assign count is
  31 (sub0's `num=33` − 2 deco-loads). Run.
- [ ] **Step 4: `mvn -q -pl beholder-importer-westwood verify`; commit** —
  `feat(p2-m1): Eob2InfScript block-A wall/door/decoration table`.

---

## Task 4: `Eob2InfGeometry.geometry` → shared `LevelGeometry` (+ the D-E archetype decision)

**Files:**
- Create: `.../westwood/Eob2InfGeometry.java`
- Test: `.../westwood/Eob2InfGeometryTest.java` (synthetic), `.../westwood/Eob2InfGeometryIT.java`
  (real LEVEL1, skip-safe)

**Interfaces:**
- Consumes: `Maze`, `Eob2InfScript` (Task 3), `Eob2DoorDecl` (for the door-band predicate).
- Produces: `Eob2InfGeometry.geometry(Maze maze, Eob2InfScript script) -> LevelGeometry`;
  package-private `Eob2InfGeometry.doorWallIndexBand(Eob2DoorDecl door) -> java.util.Set<Integer>`
  (the `wallId*10 + 3 + {0,1,2,3,5,6,7,8}` band); package-private
  `Eob2InfGeometry.archetypeFor(int wallIndex, java.util.Set<Integer> doorBand, Eob2InfScript
  script) -> WallArchetype`.

- [ ] **Step 1: `Eob2InfGeometryTest` (RED)** — a synthetic `Maze` (build a tiny raw MAZ via
  `Maze.parse` with a hand-built 6-byte header + walls, or reuse an existing test-maze builder)
  + a synthetic `Eob2InfScript` (from Task 2's `fromDecompressed`) whose wall-assign table maps
  index 5 → `(vmp 2, …)` and declares a door at `wallId 0` (band includes index 3). Place maze
  edges: one edge index 0 (→ `OPEN`, wallSet 0), one index 5 (→ `SOLID`, wallSet 2), one index 3
  (→ `DOOR`). Assert `geometry(...)`'s `archetype`/`wallSet` at those edges. Run → FAIL.
- [ ] **Step 2: Implement `Eob2InfGeometry.geometry`** — mirror `InfWallGeometry.geometry`'s
  structure (flattened `WallArchetype[]` + `int[] wallSets`, `fillCellEdges`/`fillOneCell`
  helpers, `LevelGeometry.of(width, height, archetypes, wallSets, CURRENT_IR_VERSION)`). Per
  edge: `wallIndex = maze.wall(x,y,dir)`; `archetypeFor` returns `OPEN` iff `wallIndex == 0`,
  `DOOR` iff `wallIndex` is in any declared door's band (union over `script.doors()`), else
  `SOLID`; `wallSet` = `0` for OPEN, else `script.hasWallMapping(wallIndex) ?
  script.wallMapping(wallIndex).vmpIndex() : 0` (fail-open to background for an unassigned
  index, matching EOB1's fallback). Concrete `archetypeFor`:
  ```java
  private static WallArchetype archetypeFor(
          final int wallIndex, final Set<Integer> doorBand, final Eob2InfScript script) {
      final WallArchetype archetype;
      if (wallIndex == OPEN_WALL_INDEX) {
          archetype = WallArchetype.OPEN;
      } else if (doorBand.contains(wallIndex)) {
          archetype = WallArchetype.DOOR;
      } else {
          archetype = WallArchetype.SOLID;
      }
      return archetype;
  }
  ```
  Run → GREEN.
- [ ] **Step 3: `Eob2InfGeometryIT` (real LEVEL1)** — `geometry(Maze.parse(...INF via
  Eob2InfScript...))`; assert `width()==32 && height()==32`; **assert no maze edge resolves to a
  wall-index 74** (wall-of-force is dynamic — confirms D-E: no new archetype needed); assert at
  least one `DOOR` edge exists (the 2 declared doors are placed in the maze) and at least one
  `SOLID` and one `OPEN`; assert every `OPEN` edge carries `wallSet 0` (the `LevelGeometry`
  invariant). **Record the D-E outcome in the commit body:** `{OPEN, SOLID, DOOR}` unchanged.
- [ ] **Step 4: `mvn -q -pl beholder-importer-westwood verify`; commit** —
  `feat(p2-m1): Eob2InfGeometry → shared LevelGeometry; WallArchetype {OPEN,SOLID,DOOR} suffices`.

---

## Task 5: `Eob2InfGeometry.doors` → shared `List<Door>`

**Files:**
- Modify: `.../westwood/Eob2InfGeometry.java`; Test: extend `Eob2InfGeometryTest`/`…IT`

**Interfaces:**
- Produces: `Eob2InfGeometry.doors(Maze maze, Eob2InfScript script) -> List<Door>` — one
  `Door(GridPosition.of(x,y), dir, Door.DoorState.CLOSED, wallIndex)` per maze edge whose
  wall-index is in a declared door's band (base state = CLOSED; runtime overlay is M2+).

- [ ] **Step 1: Extend `Eob2InfGeometryTest` (RED)** — the synthetic maze's index-3 edge →
  exactly one `Door(block, dir, CLOSED, 3)`; an undoored maze → empty list. Mirror
  `InfWallGeometry.doors`'s row-major/N-E-S-W scan. Run → FAIL.
- [ ] **Step 2: Implement `doors`** — reuse the door-band union from Task 4; scan every cell
  edge row-major then N/E/S/W, emitting a `Door` where `band.contains(wallIndex)`. Run → GREEN.
- [ ] **Step 3: Extend `Eob2InfGeometryIT` (real LEVEL1)** — `doors(...)` is non-empty; every
  entry's `state()==CLOSED` and `doorGraphicId()` (the wall-index) is in a declared band. Run.
- [ ] **Step 4: `mvn -q -pl beholder-importer-westwood verify`; commit** —
  `feat(p2-m1): Eob2InfGeometry doors → shared List<Door>`.

---

## Task 6: EOB2 static render golden — `Eob2LevelOneRenderIT` (walls + DUNG tileset)

**The milestone's demonstrable end state: a real EOB2 level's first-person view, rendered
headless through the shared IR + renderer.**

**Files:**
- Test: `beholder-importer-westwood/src/test/java/.../westwood/Eob2LevelOneRenderIT.java`
  (or, if the render classpath needs it, `beholder-render-libgdx/src/test/.../Eob2LevelOneRenderIT.java`
  — place it where `FirstPersonRenderer` + the importer are both on the test classpath; the
  render-libgdx test tree already depends on both, mirroring `LevelOneEntryRenderIT`).

**Interfaces:**
- Consumes: `Eob2InfScript` (Task 2/3), `Eob2InfGeometry.geometry` (Task 4),
  `Maze.parse`, `VcnAtlas.parse`/`VmpProjection.parse`/`VgaPalette.parse`,
  `WestwoodToIr.tileSet(VcnAtlas, VmpProjection, VgaPalette)`,
  `FirstPersonRenderer.render(LevelGeometry, TileSet, GridPosition, Facing)`.

- [ ] **Step 1: Write `Eob2LevelOneRenderIT` (RED / eyeball probe, real LEVEL1, skip-safe)** —
  load: `Maze maze = Maze.parse(assumeReadable("LEVEL1.MAZ"))`; `Eob2InfScript script =
  Eob2InfScript.parse(assumeReadable("LEVEL1.INF"))`; `TileSet tiles = WestwoodToIr.tileSet(
  VcnAtlas.parse(assumeReadable("DUNG.VCN")), VmpProjection.parse(assumeReadable("DUNG.VMP")),
  VgaPalette.parse(assumeReadable("DUNG.PAL")))`; `LevelGeometry geo = Eob2InfGeometry.geometry(
  maze, script)`. Pick a start block that faces walls for a meaningful eyeball — use a concrete
  interior cell verified non-open in Task 4's data (e.g. a cell adjacent to a `SOLID`/`DOOR`
  edge; document the chosen `(col,row)` + the derivation in a Javadoc note). Render all four
  facings via `FirstPersonRenderer.render(geo, tiles, start, facing)`, write each to a PNG under
  git-ignored `target/` (reuse the `writePng` shape from `LevelOneEntryRenderIT`) plus a contact
  sheet. **Assert** the four frames are not all identical (`frames.get(0) != frames.get(1)`,
  `!= frames.get(2)`) — a real assertion, not just "files written".
- [ ] **Step 2: Run it** (`mvn -q -pl <module> test -Dtest=Eob2LevelOneRenderIT`) and **inspect
  the PNGs by eye** against the DUNG dungeon look (kyra `darkmoon.cpp`/`eob2_dos.h` geometry as
  the read-only oracle). The maintainer checkpoint for this milestone is this eyeball + the
  green assertions (pixel-exact kyra golden deferred — no built ScummVM).
- [ ] **Step 3: `mvn -q verify` (root); commit** —
  `feat(p2-m1): headless EOB2 LEVEL1 first-person render through the shared IR + renderer`.

---

## Task 7: Decorations — `Eob2InfGeometry.decorations` + `AppearancePack` + decorated render

**The M1 stretch: renders the DUNG level's wall decorations. May surface a `.DEC`-vs-EOB1-`.DAT`
format delta (the agent noted EOB2 `.DEC` shape coords are `s16`); if so, a new EOB2-specific
`Eob2DecorationDat` decodes them but still emits the **shared** `DecorationTable`/`Decoration`/
`AppearancePack` — zero new IR types.**

**Files:**
- Modify: `.../westwood/Eob2InfGeometry.java` (add `decorations(...)`); Create **iff** the `.DEC`
  format differs: `.../westwood/Eob2DecorationDat.java`
- Test: extend `Eob2InfGeometryIT`; Create `.../westwood/Eob2DecorationRenderIT.java`

**Interfaces:**
- Produces: `Eob2InfGeometry.decorations(Maze maze, Eob2InfScript script) -> List<Decoration>` —
  one `Decoration(GridPosition.of(x,y), dir, overlayId, decorationId)` per maze edge whose
  wall-index has an assigned `decIndex != -1` (the `overlayId` = the decoration-load's `cps`
  name that the wall-assign's decoration belongs to; `decorationId` = the wall-assign's
  `decIndex`). Reuse `DecorationDat.parse` for `BROWN.DEC` if its record layout matches; else
  `Eob2DecorationDat.parse` produces the same `DecorationDat`-shaped accessors.

- [ ] **Step 1: Verify the `.DEC` format vs `DecorationDat` (spike, real data).** In a scratch
  step, `DecorationDat.parse(assumeReadable("BROWN.DEC"))` and check it decodes sane
  `decorationCount()`/`rectangleCount()`/`destX`/`destY` (the agent's `.DEC` layout: `[u16
  count]` + count×`[10×u8 shapeIndex][u8 next][u8 flags][10×s16 shapeX][10×s16 shapeY]` + `[u16
  rectCount]` + rectCount×`[4×u16 x,y,w,h]`). **Decide + record:** if `DecorationDat` reads
  `destX/destY` as the same `s16` width, reuse it; if EOB1 reads them narrower, create
  `Eob2DecorationDat` with the same public accessors. (This decision is the task's first commit
  note.)
- [ ] **Step 2: `Eob2InfGeometryIT` decoration assertions (RED→GREEN)** — `decorations(maze,
  script)` is non-empty; every entry's `overlayId` is one of `{"brown1","brown2"}` and
  `decorationId` matches the wall-assign `decIndex` for that edge's wall-index (e.g. an edge
  with wall-index 27 → `decorationId 18`). Implement `decorations` (mirror `DecorationPlacements`
  /`InfWallGeometry` scan structure). Run → GREEN.
- [ ] **Step 3: `Eob2DecorationRenderIT` (real LEVEL1, skip-safe)** — build the `AppearancePack`
  via `AppearancePackBuilder.build("dung", tiles, brownDat, Map.of("brown1", CpsImage.parse(
  assumeReadable("BROWN1.CPS")), "brown2", CpsImage.parse(assumeReadable("BROWN2.CPS"))),
  CpsImage.parse(assumeReadable("DOOR1.CPS")))`; render via `FirstPersonRenderer.render(geo,
  tiles, start, facing, new IrDecorationScene(decorations, pack))`; write PNGs to `target/`;
  assert the decorated frame differs from the undecorated frame of Task 6 (decorations actually
  composite). Eyeball the DUNG decorations. **Note:** `AppearancePackBuilder.build` requires a
  single shared `dat` — LEVEL1 satisfies this (both overlays share `brown.dec`), matching the
  builder's precondition.
- [ ] **Step 4: `mvn -q verify` (root); commit** —
  `feat(p2-m1): EOB2 decorations → shared Decoration/AppearancePack; decorated LEVEL1 render`.

---

## Self-review (writing-plans)

- **Spec coverage (design §4 M1):** asset decode + tileset wiring → Tasks 1, 6; `.INF`
  geometry-decl lift (block model, wall-mappings, doors, decorations, magic wall ids) → Tasks
  2–5, 7; `WallArchetype` growth iff forced → Task 4 (pre-resolved: none forced); headless
  first-person view + golden → Task 6; decorations → Task 7. All M1 scope covered.
- **One-IR invariant:** every task emits only shared `beholder-ir` types (`LevelGeometry`,
  `Door`, `Decoration`, `WallArchetype`, `AppearancePack`, `TileSet`); the only EOB2-specific
  types are importer-internal decode POJOs (`Eob2InfScript`/`Eob2WallMapping`/`Eob2DoorDecl`/
  optional `Eob2DecorationDat`) — none in `beholder-ir`. ✔
- **Type consistency:** `Eob2WallMapping(vmpIndex, decIndex, specialType, flags)`,
  `Eob2DoorDecl(shapeFile, wallId, doorType, initiallyOpen)`, `Eob2InfScript.{parse,
  fromDecompressed, mazeName, gfxName, soundName, subLevelCount, wallMapping, hasWallMapping,
  doors, decorationOverlays}`, `Eob2InfGeometry.{geometry, doors, decorations}` — used
  consistently across Tasks 2–7. ✔
- **Out of scope (deferred to M2/M3):** movement/collision, block B (script) + block C
  (block→script binding), monster/spell content, events. Task 3 only *skips* the monster
  sections to reach the geometry block. ✔
