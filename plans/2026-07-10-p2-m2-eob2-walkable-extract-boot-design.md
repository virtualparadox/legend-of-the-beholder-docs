# P2 · M2 — EOB2 walkable + the extract/boot generalization — Design

> **Status:** Design (awaiting maintainer review before the implementation plan
> is written). Second milestone of Phase 2 (PHASES §"Phase 2 — EOB2").
> **Date:** 2026-07-10
> **Grounding:** MASTER §2 (the IR is the product), §4 (importers = anti-corruption
> layer; the engine eats IR only), §7.3 ("the engine eats IR and nothing else — no
> original-format code path reaches the runtime"), §8 (differential correctness),
> §10 (schema grows from real games; two importers, one IR); **ADR-0001** (IR
> serialization — JSON + asset folder, deterministic writer, fail-closed loader);
> the P2 design (`2026-07-09-p2-eob2-design.md`) and its M1 (merged, `e8c1a4a`).
> **Builds on:** M5's `beholder-ir-json` codec (the `IrJson.writeDocument` /
> `readDocument` + asset-folder boundary), and M1's EOB2 importer
> (`Eob2InfScript`/`Eob2InfGeometry` → shared IR).

---

## 1. Goal

Make a real EOB2 level **walkable** through the *game-neutral* engine, **booted
from IR-on-disk — not the original data files**. The milestone does two things at
once:

1. **Proves the engine is game-neutral** (the P2 M2 thesis): the *same*
   `GameSession` / `RuntimeState` / movement / collision / door / decoration code
   that plays EOB1 also walks EOB2, with **no new engine or render code**.
2. **Makes the composition root IR-pure and testable** (the maintainer's
   requirement): a two-phase `extract` → `boot` pipeline where `boot` only ever
   sees IR. Today the live `LibGdxLauncher` violates MASTER §7.3 — it wires the
   EOB1 importer's POJOs straight into the engine *in-process*, never crossing the
   IR-on-disk boundary. M2 makes the boundary real, so the engine host can be
   booted from a **synthetic IR folder with zero original game data** — its first
   automated test.

## 2. Scope

**In scope:**
- A new **`beholder-app`** module: the top-level composition (`extract` + `boot`)
  and a real CLI (`extract` / `play`).
- A **game-agnostic `boot(irFolder) → GameSession`** (headless-testable; the
  libGDX window is a thin frontend over it).
- An **EOB2 `extract`**: the M1 EOB2 importer → `LevelDocument` + `AppearancePack`
  → IR folder (via M5's codec).
- **Rewiring the EOB1 launcher to boot-from-IR** — one code path for both games
  (M5's round-trip already proves EOB1 IR renders identically; a safety-net IT
  confirms).
- The **walkable proof** on real EOB2: structural walk invariants + door collision
  + decoration occlusion, driven headlessly through the booted engine.

**Out of scope (deferred):**
- `.INF` triggers / the event graph — in M2 the IR's `events.triggers` is **empty**
  (a *base* walkable level); EOB2 trigger lifting is **M3**.
- Runtime / save-state serialization; multi-level / campaign packaging;
  inter-sub-level movement.
- The full M4 acceptance test (byte-stable re-serialize + diff vs `kyra`). M2
  front-loads the round-trip *machinery* but does not add M4's byte-stability +
  kyra-diff gates.

**The load-bearing boundary (made real):** after M2, `boot` and everything
downstream (`Level`, `RuntimeState`, `GameSession`, the render) touch **only** IR
(`LevelDocument` + `AppearancePack`, read from disk). Every original-format decoder
(`PakArchive`/CPS/VCN/VMP/`InfScript`/`Eob2InfScript`) lives **only** behind
`extract`.

## 3. Architecture — the two-phase pipeline

```
 game folder ──extract(per-game)──▶  IR folder  ──boot(game-agnostic)──▶ GameSession ──▶ window (play)
  EOB1 EOBDATA*.PAK                LevelDocument.json                 Level                  OR
  EOB2 loose LEVEL*.INF/.MAZ       <name>-assets/ (tileset,          + RuntimeState        headless (test)
                                    decorations, door leaf)          + SceneOverlay
```

### 3.1 The `beholder-app` module
A new reactor module (the P0-sketched-but-never-built `app`). Dependencies:
`beholder-importer-westwood` (extract's per-game importers), `beholder-ir-json`
(read/write IR), `beholder-engine-core` (`Level`/`RuntimeState`),
`beholder-render-core` (scenes), `beholder-render-libgdx` (`GameSession` + the
window). It hosts `Extract`, `Boot`, the `Main` CLI, and the window frontend
generalization. (It depends on render-libgdx because `GameSession` lives there and
is framework-neutral/headless-capable — the same seam the existing headless ITs
drive; the actual libGDX window is only touched by the `play` command, never by
`boot` or the headless tests.)

### 3.2 The quarantine line (MASTER §7.3, made real)
`extract` is the **only** code that opens a PAK, a CPS, a `.MAZ`, or an `.INF`.
`boot` is handed a folder of IR and never imports a Westwood decoder. This is the
in-process leak the current `LibGdxLauncher` has, fixed at the composition root.

## 4. Components

### 4.1 `Extract` (per-game)
`extract <game-folder> [level] <ir-out-folder>`. **Auto-detects the game** (the
proposed default): an `EOBDATA*.PAK` present → EOB1; loose `LEVEL*.INF` present →
EOB2. Runs the game's importer:
- **EOB1:** `PakArchive` → `Maze`/`InfScript`/palette → `InfWallGeometry.geometry`/
  `.doors` + `DecorationPlacements.scan` + `Level1AppearancePack.build` +
  `WestwoodToIr.tileSet`.
- **EOB2:** loose files → `Maze`/`Eob2InfScript` → `Eob2InfGeometry.geometry`/
  `.doors`/`.decorations` + the M1 `AppearancePack` assembly + `WestwoodToIr.tileSet`.
Then assembles a `LevelDocument(irVersion, geometry, LevelEvents(triggers=[], doors,
decorations), appearancePackId)` + the `AppearancePack`, and writes them via
`IrJson.writeDocument(...)` (JSON + asset folder) — **M5's codec, reused verbatim**.
`triggers` is empty in M2 (base level).

### 4.2 `Boot` (game-agnostic, headless)
`Boot.boot(irFolder) → BootedGame`:
- `IrJson.readDocument(irFolder)` (the M5 fail-closed load pipeline) →
  `LevelDocument` + `AppearancePack`.
- `Level.of(doc.geometry(), pack.tileSet(), doc.events())` +
  `RuntimeState.startingAt(Party.of(startBlock, startFacing))` + the IR-fed
  `SceneOverlay` (`IrDecorationScene(events.decorations(), pack)` +
  `IrDoorLeafScene(events.doors(), closedTest, pack.doorLeafSheet(),
  pack.tileSet().paletteArgb())`).
- Returns a `BootedGame` value (level + state + overlays + pack) — everything a
  `GameSession` needs. The **caller** supplies the `FrameSink` (window `GlFrameSink`
  or a test `RecordingFrameSink`) and the `InputSource` (`LibGdxInputSource` or a
  scripted test source). **No original-file code path is reachable from here.**
- **Start position** comes from the IR document. M5's `LevelDocument` does not yet
  carry a party start; M2 adds a minimal `start` (block + facing) to the document
  (a small, additive IR extension — the walkable level needs a spawn point) or,
  simpler, `boot` takes an explicit start param the CLI/test supplies. *(Which of
  the two — IR-carried vs boot-param — is a plan-level call; see §7.)*

### 4.3 CLI (`Main`)
- `extract <game-folder> [level] <ir-out>` → writes the IR folder.
- `play <ir-folder>` → `boot` + open the libGDX window over the booted
  `GameSession`. This is the maintainer's manual test workflow: run `extract`,
  inspect the IR folder, run `play`.

### 4.4 Window frontend
render-libgdx's Lwjgl3 window (today `LibGdxLauncher`'s nested
`EobApplicationAdapter`), generalized to play **any** booted `GameSession` — it
loses its EOB1-specific `loadGame`/`buildLevel` (moved to `extract`) and instead takes a
`BootedGame`, building the `GameSession` from it with the window's `GlFrameSink` +
`LibGdxInputSource` — the headless tests build the same `GameSession` from a `BootedGame` with
a test `FrameSink`/`InputSource` (one seam, two frontends).

### 4.5 EOB1 rewire
`LibGdxLauncher`'s in-process EOB1 loading is replaced by `extract(EOB1 folder) →
IR → boot`. Because M5's `LosslessRoundTripIT` already proves EOB1 `extract → IR →
reload` renders pixel-identically, this is low-risk; a safety-net IT (§5) pins that
the booted-from-IR EOB1 frame equals the pre-M2 in-process frame.

## 5. Testing strategy (the payoff)

- **Headless boot from a *synthetic* IR folder (no game data — the launcher's
  first automated test).** A tiny hand-authored `LevelDocument` + minimal asset
  folder (our own content) is committed; `Boot.boot(...)` → `GameSession` with a
  scripted `InputSource` + a `RecordingFrameSink` → assert the structural walk
  invariants. Deterministic, **skip-free** (no `assumeTrue`), fully in-repo. This
  is what "keep the launcher general so I can test it" buys.
- **Real EOB2 `extract → boot → walk` IT (skip-safe).** `extract` real EOB2
  `LEVEL1` → IR into git-ignored `target/` → `boot` → scripted walk → the same
  invariants + door collision (passable-iff-open) + decoration occlusion. Maintainer
  eyeballs the live `play`.
- **EOB1 render-identity safety-net IT.** The booted-from-IR EOB1 entry frame ==
  the pre-M2 in-process path's frame (leaning on M5's proven round-trip), so the
  rewire is provably behaviour-preserving.
- **Structural walk invariants (the P1 M3 shape, reused):** turn-cycle identity
  (four rights = identity), never-off-grid, settle-at-wall (a blocked forward step
  is a no-op), blocked-step-no-op across all four facings — asserted on the booted
  EOB2 level.

## 6. Differential evidence (MASTER §8)

The proof is *sameness*: one `boot` + one `GameSession` walks **both** games —
EOB1 render-identically to the pre-M2 path, and EOB2 for the first time — with the
engine/render/IR code **unchanged** (the diff touches only the new `beholder-app`
module + the `LibGdxLauncher` rewire). No game data committed; real IR artifacts to
git-ignored `target/`; the synthetic IR folder is our own content and IS committed.
Pixel-exact kyra golden stays deferred (no built ScummVM); the oracle is the
EOB1-render-identity net + the structural walk invariants + maintainer eyeball.

## 7. Open questions & arc impact

- **Party start: IR-carried vs boot-param.** The base walkable level needs a spawn
  (block + facing). Either add a minimal `start` to `LevelDocument` (additive IR
  extension, versioned — the honest "the level knows where you begin") or pass it to
  `boot`/`play` explicitly (no IR change, but the spawn lives outside the IR).
  Resolved in the plan; leaning IR-carried (a level *is* its spawn).
- **The `SceneOverlay` wall-mapping resolver leak (a discovered boundary wrinkle).**
  `SceneOverlay` today takes a `wallMappingResolver` backed by
  `InfWallGeometry.wallType(script, …)` — an *importer* dependency used only to
  resolve a runtime `SET_WALL` override's raw index into a wall-set. In M2 (no
  triggers → no `SET_WALL`) it is never called, so `boot` supplies a trivial
  resolver. **But M3 (events) will need this resolvable game-agnostically at boot**
  — i.e. the wall-mapping table must ride in the IR, not be recomputed from the
  script. M2 surfaces this leak and defers it to M3 (a concrete example of the
  boundary doing its job).
- **Game auto-detection** (PAK vs loose) is the proposed default; an explicit
  `--game` flag is the fallback if detection proves fragile.
- **Arc impact (not a re-plan):** M2 front-loads the round-trip machinery at the
  app level. **M3** (EOB2 events) flows through the *same* extract/boot (triggers
  become non-empty; the resolver leak above is closed). **M4** (round-trip + diff vs
  kyra) gets cheaper — it adds byte-stable re-serialize + the kyra diff on top of
  M2's pipeline rather than building it.

## 8. Global constraints (carried from P1/M1)

- **JDK 21**; `mvn spotless:apply` then `mvn verify`; `-pl <m> -am` or root verify.
- **All `CODING_STANDARDS` gates green** (one `return`/method, no `instanceof`,
  cyclomatic ≤ 8, coverage ≥ 0.90 branch+line, NullAway, full Javadoc, every magic
  value a named constant + comment, …). The new module obeys them; the CLI/window
  glue that is genuinely untestable-without-a-window follows the existing
  render-libgdx `gl/**` JaCoCo-exclusion precedent, but `Boot`/`Extract` logic is
  **not** exempt and is covered by the synthetic-IR + real-data tests.
- **New module = allow-listed deps only** (the license gate); `beholder-app` adds no
  new third-party dep beyond what its dependencies already bring.
- **Clean-room** (kyra/eoblib read-only oracles); jCodeMunch local-only; **never
  commit game data** (real ITs skip-safe, artifacts to `target/`); the synthetic IR
  golden is our own content and IS committed.
- **Process:** `writing-plans` → `subagent-driven-development` (fresh implementer +
  independent review per task), TDD, whole-branch review, `--no-ff` merge on the
  maintainer's verdict; conventional branches, no worktrees.

## 9. Checkpoint (MASTER §12)

Maintainer review of: the walkable EOB2 evidence (structural invariants + the live
`play` eyeball), the EOB1 render-identity net (the rewire is behaviour-preserving),
and the extract/boot boundary (is `boot` genuinely IR-only, and is the synthetic-IR
headless test the durable regression pin the maintainer wanted?). On approval, the
engine is proven game-neutral across two Westwood games and the composition root is
IR-pure — the substrate M3 (EOB2 events) and M4 (round-trip + diff) build on.

---

## 10. M2 outcome & M3 carry-forward (post-merge)

**Merged to `main`** (`13db07f`, `--no-ff`): all 8 tasks + one trivial Javadoc fix;
per-task spec+quality reviews + a whole-branch opus review. The load-bearing one-IR
boundary was verified by import scan — only `Eob1Extractor`/`Eob2Extractor` import
Westwood decoders; `Boot`/`BootedGame`/`GameWindow`/`Main` and the render-libgdx main
tree are decoder-free, and `LibGdxLauncher`'s in-process loading is deleted. EOB2
LEVEL1 is walkable booted-from-IR (decorations + door leaf render); the EOB1 rewire is
proven behaviour-preserving by a byte-identical render-identity net; the launcher's
first automated test is a headless boot from a committed synthetic IR (no game data).

**Carry-forward to M3** (from the whole-branch review):
- **Asset-name resolution — generalize when a second look exists.** The extractors
  hardcode LEVEL1's asset file names (EOB2 `DUNG.*`/`BROWN1/2.CPS`/`BROWN.DEC`/
  `DOOR1.CPS`, pack id `dung`; EOB1 `BRICK.*`/`DOOR.CPS`). Deferred per MASTER §8 (with
  only LEVEL1 available, name-derivation code adds branches no test can differentially
  validate). Derive them from `Eob2InfScript.gfxName()`/`.decorationOverlays()`/door
  shapeFiles (EOB1: `InfScript.vcnVmpName()`/overlays) when M3's events or a multi-level
  step brings a second look. Optional cheap hardening meanwhile: a fail-closed
  `gfxName()`-matches-the-hardcoded-look guard.
- **`RuntimeState.initialDoorState` is edge-blind — fix before M3's door work.** It
  filters `events().doors()` by `block` only, ignoring `Door.edge()`, so a two-door
  block resolves to `findFirst()`. Harmless under M2 (base level, all doors CLOSED),
  but a real bug once a trigger opens one door on a multi-door block.
- **The `SceneOverlay` wall-mapping resolver leak (design §7).** M3 must carry the
  wall-mapping table in the IR so a `SET_WALL` override resolves game-agnostically at
  boot (M2 uses an inert identity resolver since it has no triggers).
