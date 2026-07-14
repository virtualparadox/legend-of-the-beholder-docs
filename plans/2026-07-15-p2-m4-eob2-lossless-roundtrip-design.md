# P2 · M4 — EOB2 lossless round-trip + kyra differential — Design

> **Status:** Design (awaiting maintainer review before the implementation plan
> is written). Fourth and final milestone of Phase 2 (PHASES §"Phase 2 — EOB2");
> **closes Phase 2**.
> **Date:** 2026-07-15
> **Grounding:** MASTER §2 (the IR is the product — *one* IR for all SSI games),
> §8 (differential correctness + the lossless round-trip as the primary
> machine-checkable acceptance test), §12 (the phase-close maintainer checkpoint);
> **ADR-0001** (IR serialization — JSON / deterministic writer / fail-closed
> loader). Realises the P2 design's M4 (`2026-07-09-p2-eob2-design.md` §4 "M4 —
> lossless round-trip + diff", §5 differential-evidence posture). Builds on M1
> (merged `e8c1a4a`), M2 (`13db07f`), and **M3** (merged `0e8e598` — the EOB2
> `.INF` event lift, which introduced the one new IR node kind this phase: the
> `WallMappingTable`).
> **Template:** the P1 M5 EOB1 acceptance test `LosslessRoundTripIT`
> (`beholder-render-libgdx`) — its three legs (equality, byte-stability,
> render-identity) are mirrored here for EOB2.

---

## 1. Goal

Close Phase 2 with its acceptance gate: prove that a **real, unmodified EOB2
`LEVEL1`**'s canonical on-disk IR form is **lossless**, **byte-stable**, and
**renders identically** after a full write→reload round-trip through the *reused,
unchanged* `beholder-ir-json` codec — and consolidate the phase's accumulated
differential evidence into the maintainer's Phase-2-close verdict: **did the one
IR absorb a second real Westwood game without distortion?**

This milestone writes almost no production code. Its value is the *proof*: the
same codec that served EOB1 (P1 M5) round-trips EOB2 unchanged (the game-neutrality
of the serde), including the one IR node kind M3 added (`WallMappingTable`). Success
is a green real-`LEVEL1` round-trip IT plus the maintainer's eyeball of the
before/after render — the machine-checkable half of MASTER §8 — with the pixel-exact
`kyra` golden remaining deferred (documented, non-blocking; see §4).

## 2. Scope

**In scope:**
- **`Eob2LosslessRoundTripIT`** — a real-`LEVEL1`, skip-safe round-trip IT
  mirroring EOB1's `LosslessRoundTripIT`, with all three legs (equality,
  byte-stability, render-identity), building `IR_a` directly from the EOB2
  in-memory decode via `Eob2Extractor.extract`.
- **The committed synthetic golden's `WallMappingTable` made non-trivial** — the
  one always-run regression anchor for M3's new IR node kind, populated with a
  real (non-zero) table instead of the current all-zeros placeholder.
- **Differential-evidence consolidation** — the design/plan records the phase's
  accumulated differential evidence (the round-trip + the M1 geometry-vs-`eob2_dos.h`
  and M3 trigger-vs-`kyra`-script oracle cross-checks) as the Phase-2-close answer.

**Out of scope (deferred / YAGNI — MASTER §10):**
- **The DOSBox pixel ground-truth capture** — a real DOS EOB2 screenshot
  pixel-diffed against our render. The P2 design (§5) carved this out as a
  separate, later task; the maintainer reaffirmed it stays deferred at M4's
  brainstorm. The round-trip + oracle cross-checks are the established, sufficient
  acceptance posture for all of P1+P2.
- **New codec / serde** — none. M3 already added the `WallMappingTable` serde
  (serializer/deserializer/`Nodes` allow-list/schema/back-compat). M4 adds no new
  IR node kind, so no new serde.
- **New shared vocabulary** — none forced this milestone.
- Monsters, items, combat, spells; runtime/save-state serialization; multi-level /
  campaign packaging; anything from Phases 3–5.

**The load-bearing invariant — the codec is UNCHANGED (game-neutral).** The whole
point of leg (b)/(c) is that `beholder-ir-json` serves EOB2 with zero EOB2-specific
serde. If M4 finds itself needing to touch the codec, that is a signal the IR node
was EOB1-shaped — a finding to surface, not to paper over.

## 3. Architecture

### 3.1 The parallel per-game round-trip IT (decided)
A new `Eob2LosslessRoundTripIT` sits alongside EOB1's `LosslessRoundTripIT`,
matching this codebase's parallel-per-game IT convention (as `Eob2BootWalkIT`
parallels the EOB1 walk IT). It is **cleaner** than the EOB1 template: because the
`GameExtractor` abstraction (M2) exposes `IrDocument extract(Path gameFolder,
String level)` — the in-memory decode, before any disk write — `IR_a` is obtained
directly:

```
IrDocument irA = new Eob2Extractor().extract(GAME_FOLDER, "LEVEL1"); // in-memory decode
IrJson.writeDocument(dir, "level1", irA);                            // JSON + assets, git-ignored target/
IrDocument irB = IrJson.readDocument(dir, "level1");                 // fail-closed reload
```

*(Rejected alternatives: (i) reusing the `Extract.extract` disk path for `IR_a`
would give only reload-vs-reload, weakening leg (a) — it would not prove the fresh
decode survives the first write; (ii) a game-parametric single IT coupling both
games' fixtures runs against the parallel-per-game convention.)*

### 3.2 The three legs (mirroring `LosslessRoundTripIT`, P1 M5 §6)
- **(a) Equality:** `assertThat(irB).isEqualTo(irA)` across the whole IR —
  geometry, events (trigger bindings, doors, decorations), the `WallMappingTable`,
  and the `AppearancePack`.
- **(b) Byte-stability:** `IrJson.toCanonicalJson(irB.level())` equals the on-disk
  `level1.json` text, and a second `IrJson.writeDocument(dir, "level1", irB)`
  reproduces byte-identical files — the canonical JSON plus every asset-folder
  file (`snapshotFiles` before/after, key-set + per-file byte equality).
- **(c) Render-identity:** the EOB2 entry viewpoint — `(col=5, row=4)` facing
  `NORTH`, the interior entry cell `Eob2BootWalkIT`/`Eob2LevelOneRenderIT` already
  use — rendered from `IR_a` and from `IR_b` through the IR-fed render path (the
  same `GameSession#tick()` wiring the other render ITs drive, base door state
  all-CLOSED) is pixel-for-pixel identical. The render's wall resolver is built
  from the **document's own `WallMappingTable`** (`i -> doc.wallMappings()
  .wallSetForIndex(i)`), keeping the render purely IR-sourced — nothing is read
  behind the IR's back. The resolver is **inert for this override-free static probe**
  (no `SET_WALL` fires, so nothing is resolved through it), exactly as EOB1's
  `LosslessRoundTripIT` documents its identity resolver; the `WallMappingTable`'s own
  losslessness is proven by leg (a), not by this leg. A before/after contact-sheet
  PNG is written to git-ignored `target/` for the maintainer eyeball.

Because `IR_a.equals(IR_b)`, the renders are trivially identical; the point (as in
P1 M5) is proving the **reload alone is a sufficient render input**.

### 3.3 The non-trivial synthetic golden (M3's new node, always-run anchor)
The committed synthetic golden (`synthetic-level.json` / `SyntheticGoldenTest`)
currently carries `wallMappings` as 256 zeros — M3 added the field but left the
always-run anchor trivial for the new node. M4 populates it with a real, non-zero
`WallMappingTable` (a handful of distinct wall-set indices at distinct wall-index
slots), so the **data-independent** test suite genuinely anchors the new node's
serialization (the real round-trip in §3.2 is skip-safe; this is its
runs-without-game-data complement). This is a fixture + expected-golden edit, not a
new artifact — the goldens ratchet, never regress (P2 design §5).

## 4. Differential-evidence & correctness posture (MASTER §8)

- **The round-trip IT is the primary machine-checkable acceptance test:** real
  `LEVEL1` equality + byte-stability + render-identity, no game data committed.
- **The `kyra` differential for Phase 2 is the accumulated oracle cross-checks**,
  consolidated here: M1's geometry/projection eyeballed against `eob2_dos.h`
  (dsc/door tables) + `darkmoon.cpp`; M3's fired trigger's effect cross-checked
  against the shared `EoBInfProcessor` script semantics (including the byte-identical
  `((flags&0xFFF8)>>3)|0xE0` reason decode). These, plus the round-trip and the
  maintainer eyeball, are the differential evidence.
- **The pixel-exact `kyra` golden stays deferred** (no built ScummVM; documented,
  non-blocking). The EOB2 `.app` ships DOSBox, so a pixel ground-truth capture
  remains available as a later, separate task (§2, out of scope).
- **Clean-room:** `kyra`/`eob2_dos.h` are read-only oracles, never copied.

## 5. Global constraints (carried from P1/P2)

- **JDK 21** (`JAVA_HOME` = Corretto 21); `mvn spotless:apply` then `mvn verify`;
  single-module builds use `-pl <m> -am` **or** root verify (the stale `~/.m2`
  upstream-jar trap).
- **All `CODING_STANDARDS` gates green** (one `return`/method, no `instanceof`,
  cyclomatic ≤ 8, NPath ≤ 120, field/method/param caps, no `return null`, `final`
  params, braces, ≤ 140-char lines, no `System.out`, NullAway `ERROR`, full
  Javadoc, every magic value a named `ALL_CAPS` constant + what/why comment).
- **Coverage: JaCoCo ≥ 0.90 branch AND line** (Surefire/unit only — skip-safe ITs
  do not count). M4's real-`LEVEL1` round-trip is a skip-safe **IT**, so it carries
  no jacoco burden; the always-run golden edit is a unit-level `SyntheticGoldenTest`.
  No new production logic is introduced, so no new synthetic-fixture coverage debt.
- **Clean-room; never commit game data:** the real-data IT is `assumeTrue`-skip-safe;
  the real-level JSON/assets/PNG go to git-ignored `target/`. The synthetic golden is
  our own content and IS committed.
- **Process:** `writing-plans` → `subagent-driven-development` (fresh implementer +
  independent review per task), TDD per task, whole-branch review, `--no-ff` merge
  on the maintainer's verdict; conventional branches/commits, **no worktrees**.

## 6. Open questions (plan-level)

- **The EOB2 entry viewpoint for leg (c).** `(5,4)` NORTH is the established
  interior entry cell (`Eob2BootWalkIT`/`Eob2LevelOneRenderIT`); the plan confirms
  it renders a non-degenerate frame (a wall in view) so render-identity is a
  meaningful pixel comparison, not an all-black frame.
- **The synthetic golden's populated table shape.** Which wall-index → wall-set
  entries to seed (a small, readable, distinct set) and whether to edit the existing
  `synthetic-level.json` in place or add a sibling golden — a plan-level call; the
  design's intent is the *minimal* change that makes the always-run anchor
  non-trivial for the new node.
- **`Eob2Extractor` instantiation.** Whether the IT constructs `Eob2Extractor`
  directly or via the `Extract`/`GameExtractor` resolution path — a plan detail;
  either yields the same in-memory `IrDocument`.

## 7. Checkpoint (MASTER §12) — Phase-2 close

Maintainer review of: the real-`LEVEL1` round-trip's three green legs (equality +
byte-stability + the before/after render contact sheet); confirmation the codec was
**not** touched (game-neutral serde); and the consolidated differential evidence.
The governing question is the Phase-2 thesis: **did the one IR absorb a second real
Westwood game without distortion?** On approval, **Phase 2 is complete** — the IR is
proven against two Westwood games (EOB1 + EOB2), the precondition for the Phase 3
AESOP lift (the project's primary risk), which stresses the same event substrate
hardest.
