# P3 · M1 — EOB3 AESOP importer + walkable third game (design)

*Date: 2026-07-16. Phase 3 (EOB3 / AESOP) milestone 1. Feeds `writing-plans` → subagent-driven
development.*

## 1. Context & goal

Phases 1–2 (EOB1, EOB2) are complete and merged: two Westwood importers feed **one shared IR**
(`beholder-ir`), rendered/walked through `beholder-render-core` / `beholder-engine-core` /
`beholder-render-libgdx`. We are going **vertical to EOB3**, which runs on the **AESOP** bytecode VM
(John Miles, 1992) — a different engine family, the project's primary risk and the real test of the
"one IR for all SSI games" thesis.

Two recon efforts already de-risked this (both grounded in the `thirdeye` GPL oracle + original
AESOP source + real `EYE.RES`):
- **Recon note:** `notes/2026-07-15-eob3-aesop-recon.md` — the `.RES` container, the AESOP VM, the
  geometry (AESOP-native 32×32 occupancy grid), and how EOB3 events map onto the shared IR.
- **Render spikes:** `spikes/2026-07-16-eob3-render-spike.md` — two throwaway spikes on real bytes
  proved (1) EOB3 occupancy→per-edge geometry renders + walks through the **existing**
  `FirstPersonRenderer`, and (2) real EOB3 wall art (`1.10` VFX + `PAL`) composes into an authentic
  EOB3-native first-person view. The whole vertical is proven end-to-end.

**Goal of M1 (this milestone):** a **third game walkable from the shared IR**. Read `EYE.RES`, lift a
real EOB3 level's 32×32 occupancy grid into the shared `LevelGeometry`, and walk it in the existing
libGDX front-end. Walls use a **synthesized placeholder appearance** (the existing EOB1-shaped
projection) — the *real* EOB3 whole-sprite render is **M2** (maintainer decision, 2026-07-16). M1's
point is to prove, in production: the `.RES` reader, occupancy→per-edge geometry, walkability, and
the IR round-trip — with **zero changes to `beholder-ir` or `beholder-engine-core`**.

## 2. Scope

**In (M1):**
- New module `beholder-importer-aesop` (the AESOP decoder quarantine): `.RES` archive reader, ROED
  name→number resolver, the 32×32 occupancy-maze reader, and the occupancy→`LevelGeometry` mapping.
- A **synthesized placeholder `AppearancePack`** (flat/procedural walls in the existing `TileSet`
  format) so the extractor needs only `EYE.RES` — no EOB1 data dependency. Precedent: P2 M4
  `SyntheticLevel`'s synthetic wall-sets.
- `Eob3Extractor implements GameExtractor` in `beholder-app`, plus `GameKind.EOB3`, `GameDetector`
  (marker: `EYE.RES` present), and `Extract` dispatch.
- Walkable via the **existing** `Boot` + `FirstPersonRenderer` + `GameWindow`; skip-safe real-data
  ITs against the maintainer's `EYE.RES`; an IR round-trip proving the codec is game-neutral for
  EOB3.

**Out (M2+):**
- The **EOB3-native whole-sprite render path** (real `1.10` VFX walls + `PAL`, the reconstructed
  projection) — **M2**. The `VfxShapeTable` / `AesopPalette` decoders land there.
- The AESOP **event lift** (triggers/doors/decorations as AESOP objects) — **M3**, the thesis test.
- The AESOP **VM / bytecode** (not needed until the event lift; M1 reads only the map + dictionaries).
- Dungeon Hack (a later consumer of the same AESOP foundation).

## 3. Architecture

Dependency direction is unchanged: `beholder-importer-aesop` depends only on `beholder-ir` (it
returns IR value types; the `IrDocument` — an `ir-json` type — is assembled in `beholder-app`'s
`Eob3Extractor`, exactly as `Eob1Extractor` wraps `WestwoodToIr`'s output). The shared IR/engine stay
untouched — the one-IR boundary holds.

```
EYE.RES ──▶ beholder-importer-aesop ──▶ IrDocument ──▶ (existing) Boot ──▶ walkable libGDX
            ResArchive / Roed / OccupancyMaze / AesopToIr / PlaceholderPack
            + beholder-app: Eob3Extractor, GameKind.EOB3, GameDetector, Extract
```

### 3.1 `beholder-importer-aesop` classes
- **`ResArchive`** — the clean-room `.RES` reader (from recon facts, verified on real bytes): 36-byte
  header (`first_directory_block @0x18`), linked 644-byte directory blocks, 12-byte entry headers,
  verbatim little-endian payloads; lookup by resource number and (via `Roed`) by name;
  `data_attributes & 0x10000000` = placeholder/no-payload. House pattern: private ctor, static
  factory (`parse(byte[])`), fail-closed validation, scalar accessors.
- **`Roed`** — parses resource 0's dictionary blob (`u16 bucketCount`, `u32` chain offsets, chains of
  length-prefixed `key,value` strings, value = ASCII-decimal resource number) → an immutable
  name→number map. This is how a level's map grid is resolved (`"Mausoleum 1"` = #5).
- **`OccupancyMaze`** — the 32×32 occupancy grid (1 byte/cell): `0xff` = open floor, else solid.
  Immutable value; scalar `isOpen(x,y)` accessor; validates a 1024-byte payload.
- **`AesopToIr`** — the anti-corruption assembler (the Westwood `WestwoodToIr`'s AESOP twin):
  `OccupancyMaze` → `LevelGeometry` via the **occupancy→per-edge derivation** (§3.3), the
  `WallMappingTable`, and the placeholder pack id. Returns the base `LevelDocument`.
- **`PlaceholderPack`** — builds a self-contained synthetic `AppearancePack` (a valid `TileSet` whose
  wall-set(s) render as flat/simple walls through the existing 431-code projection) so M1 needs no
  EOB1 assets. Assembled in the extract phase, the way `Eob1Extractor` calls `Level1AppearancePack`
  (which lives in `render-libgdx/gl`); modelled on P2 M4's `SyntheticLevel` synthetic wall-sets. Its
  exact module home (a `render-libgdx` helper vs `beholder-app`) is settled in `writing-plans`.

### 3.2 `beholder-app` wiring
- **`Eob3Extractor implements GameExtractor`**: `extract(gameFolder, level)` reads `gameFolder/EYE.RES`
  via `ResArchive`, resolves `level` (a map name, e.g. `"Mausoleum 1"`) via `Roed`, decodes the
  `OccupancyMaze`, and assembles the `IrDocument` (`AesopToIr` geometry + empty `LevelEvents` +
  `PlaceholderPack`). No triggers/doors/decorations yet (empty events — the event lift is M3).
- **`GameKind.EOB3`**, **`GameDetector`** (marker: `EYE.RES` present, vs EOB1's `EOBDATA*.PAK` / EOB2's
  loose `LEVEL1.INF`), **`Extract`** map entry.

### 3.3 The occupancy→per-edge derivation (the one lossy transform M1 owns)
The shared IR is **per-edge** (each cell's 4 edges carry `WallArchetype` OPEN/SOLID/DOOR + a
`wallSet`). EOB3 stores a **single scalar per cell**. Derivation: for cell `C` direction `d` with
neighbour `N` (off-grid = solid), the edge is **SOLID** iff exactly one of `{C, N}` is open (a wall
stands between a walkable cell and solid rock / the boundary), else **OPEN**. This is consistent for
both cells sharing the edge, blocks entry into solid cells (so `MovementCommand` walkability is
correct with **zero engine change**), and encloses corridors. SOLID edges carry the placeholder
pack's standard wall-set index; OPEN edges carry `0`. No `DOOR` archetypes in M1 (doors are AESOP
objects — M3).

## 4. Data flow & the walk
`Extract.extract(gameFolder, "Mausoleum 1", irOut, baseName)` → detect EOB3 → `Eob3Extractor` →
`IrDocument` written by `IrJson.writeDocument`. Then `Main play <irFolder> <col> <row> <facing>` →
`Boot.boot` → `GameWindow.play` → the maintainer walks the Mausoleum. Starting cell: an open cell of
the grid (e.g. the recon's graveyard/mausoleum start, or any open cell), passed as `play` args.

## 5. Testing (coding-standards gates: ≥90% branch coverage, full Javadoc, `-Werror`)
- **Unit tests** (`*Test`, synthetic fixtures, always run): `ResArchive` header/dir/entry parsing +
  by-number/by-name lookup + placeholder handling; `Roed` dict parsing (incl. multi-bucket chains);
  `OccupancyMaze` open/solid + validation; `AesopToIr` occupancy→edge truth table (open-open→OPEN,
  open-solid→SOLID both directions, boundary→SOLID, solid-solid→OPEN); `PlaceholderPack` builds a
  valid `TileSet`/`AppearancePack`.
- **Skip-safe ITs** (`*IT`, real `EYE.RES`, `assumeTrue` on readability, never committed data):
  `Eob3ExtractorIT` decodes a real level end-to-end into a self-consistent `IrDocument`;
  `Eob3WalkableIT`/render-smoke proves a booted frame renders (mirrors EOB1/EOB2 walkable ITs);
  `Eob3LosslessRoundTripIT` proves the IR JSON codec round-trips EOB3 byte-stably (codec
  game-neutral — the milestone's IR-thesis check).
- Real-level artifacts go to git-ignored `target/`; no game data or game-art image is committed.

## 6. Risks & mitigations
- **Placeholder pack validity** — a synthetic `TileSet` must satisfy the render's 431-code/VCN/palette
  contract. *Mitigation:* reuse the P2 M4 `SyntheticLevel` pattern; a render-smoke IT catches an
  invalid pack at boot (fail-closed).
- **Level/name resolution** — `Roed` value encoding (ASCII-decimal) must be exact. *Mitigation:*
  verified on real bytes in the spike; unit-test multi-bucket chains; IT asserts `"Mausoleum 1"`→#5.
- **Grid orientation** — the occupancy grid's row/col order vs the IR's may be flipped/mirrored.
  *Mitigation:* self-consistent either way for walkability; eyeball against the ASCII map + a walk;
  exact orientation nailed for real in M2 against DOSBox ground truth.
- **`-am` reactor / stale `~/.m2`** — build the new module with `-am` or a root verify (see memory
  `maven-reactor-am-trap`).

## 7. Milestone boundaries
- **M-EOB3.1 (this):** importer + walkable, placeholder walls. Deliverable: EOB3 Mausoleum walkable
  from the shared IR; container + geometry + walkability + round-trip proven in production.
- **M-EOB3.2:** the EOB3-native whole-sprite render path — real `1.10` VFX walls + `PAL`, the
  reconstructed projection, exact placement tables baked from the dungeon draw handler.
- **M-EOB3.3:** the AESOP event lift (triggers/doors/decorations → the shared
  Action/Condition/Reason/TriggerBinding vocabulary) — the real one-IR thesis test.

## 8. References
`notes/2026-07-15-eob3-aesop-recon.md`, `spikes/2026-07-16-eob3-render-spike.md`, memory
`aesop-oracle-stash` / `package-organization-eobN-subpackages` / `agents-md-and-env-map`. Clean-room:
`thirdeye` / original AESOP source / DAESOP are read-only fact oracles (isolated in subagents); no
oracle code copied; no game assets committed.
