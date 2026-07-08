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
