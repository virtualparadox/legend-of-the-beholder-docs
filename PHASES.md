# Grid-Crawler Engine Platform — Phase Plan (Roadmap)

> **Status:** living companion to `MASTER.md`. The spine deliberately stays
> phase-free (MASTER §1, §12); this document holds the concrete breakdown
> produced *against* it, and evolves as real games teach us where the IR must
> stretch.
> **Grounding:** this plan is not top-down speculation — it is grounded in a
> clean-room source-reconnaissance pass over the reference engines, tools, and
> the original game data (see §2).

---

## 1. How to read this plan

- **Fidelity gradient.** Near phases (P0, P1) are detailed with concrete tasks.
  P2 is medium. P3–P5 are **intentional sketches**, elaborated just-in-time from
  the previous phase's learnings. Detailing the AESOP/Dungeon-Hack/editor phases
  now would violate MASTER §5 ("no speculative universal abstractions ahead of
  two real games proving they're needed") and §10 ("let the schema grow from
  real games, never top-down"). This is by design, not omission.
- **Vertical slice per game, not horizontal layers.** The engine and the IR are
  born *with* EOB1 (P1) and grow per game. There is no "build all importers
  first" phase — that is the anti-pattern MASTER §2/§10 warns against (an IR no
  engine has consumed is an unvalidated, likely-wrong IR).
- **Every phase defines:** a goal, scope/tasks, a demonstrable end state,
  differential-correctness evidence (MASTER §8), and a human-gated checkpoint
  (MASTER §12).

## 2. Recon basis (what grounds this plan)

A reconnaissance pass studied, clean-room, the following. All reading was to
*learn and map*, never to copy code; GPL sources are oracles only.

- **ScummVM `kyra`** (`~/Workspaces/CPP/scummvm`, GPL) — the authoritative
  reverse-engineered Westwood EOB1/2 engine: data formats, the first-person
  render pipeline, the `.INF` level-script VM, and the shared AD&D 2e rules.
- **Local Java references** — `eoblib` (a clean-room EOB1 binary importer with a
  full `.INF` bytecode disassembler), `wallsetviewer` (VCN/VMP → wall-image
  assembly), `darkmoor` (a libGDX EOB2 remake; our closest stack reference).
  (`crawler` was found to be an unrelated Vaadin web crawler — not EOB.)
- **EOB reverse-engineering wiki** (`~/Documents/EOB-unzip/eob.phpwiki`) —
  concrete EOB1/2 format specifications and 12 decompiled EOB1 level scripts.
  Contains **no** AESOP/EOB3 logic documentation (graphics only).
- **AESOP tooling** (`~/Workspaces/AESOP/`) — Thirdeye (GPL C++ AESOP reimpl,
  oracle only), DAESOP (public-domain resource extractor + **bytecode
  disassembler** driven by `abc_list.def`, the adoptable opcode spec), the
  AESOP/32 compiler + runtime + language manuals (public domain, John Miles),
  and the Dungeon-Hack runtime-function delta.
- **Original game data** (`~/Games/<Game>.app/Contents/Resources/game/`, the
  user's own — MASTER §9 legal model; the engine ships no assets). EOB1 packed
  (`EOBDATA*.PAK`, `EYE.PAK`); EOB2 loose (`LEVEL*.INF/.MAZ`, `ITEM.DAT`); EOB3
  `EYE.RES`; Dungeon Hack `HACK.RES`, `OPEN.RES`. The `.app` bundles include
  DOSBox, so the originals are runnable for ground-truth capture.

## 3. Cross-cutting findings that shape the phases

1. **The IR is three separable layers.**
   - *Geometry/behaviour:* a 32×32 grid of cells, 4 walls each (N/E/S/W); a wall
     is a **closed enumerated archetype** (solid / passage / door(+subtype) /
     illusory / switch / niche / teleporter / force-wall …) — a handle, never
     geometry. Plus a decoration layer and a base-vs-runtime state split
     (levels mutate and are save-persisted).
   - *Swappable tileset (appearance):* VCN 8×8 tile atlas + VMP projection LUT +
     palette + shading. Fully separable from behaviour — **this is what makes
     higher-fidelity/QoL rendering feasible without touching level logic**
     (MASTER §7.5).
   - *Event graph:* `event → condition → action` with a closed action vocabulary.
2. **EOB1/2 format decoding is largely a clean-room port, not fresh RE.** `eoblib`
   already parses CPS/VCN/VMP/MAZ/INF/DAT and disassembles `.INF`; `wallsetviewer`
   assembles walls. This substantially de-risks P1.
3. **"Lift" competency starts at P1, not P3.** The Westwood `.INF` system is
   itself a small imperative bytecode VM (jumps, a 10-deep call stack, an RPN
   condition evaluator with evaluation side-effects) — *not* a flat table. So the
   first Westwood importer already needs a decompilation pass (control-flow-graph
   reconstruction → condition AST), and three opcodes (`sequence`, `dialogue`,
   `specialEvent`) must be modelled as opaque named-cutscene actions rather than
   decomposed. (This corrects MASTER §6's "simple/tabular" characterisation.)
4. **Rules-vs-content (MASTER §4.1) is validated by ground truth.** `kyra` is one
   shared AD&D 2e engine plus per-game *data packs*: the AD&D tables (THAC0
   progression, save matrices, ability modifiers, spell slots, HP/level, XP
   curves) are engine-side; per-entity stats (monster AC/THAC0/dice, item
   dice/bonus, spell flags) are IR content. Content schemas (monster/item/spell/
   character) are essentially already mapped. Dungeon Hack single-character mode
   is `party_size = 1`, a parameter — not a rules fork (confirmed).
5. **AESOP is the real risk, but we are not decoding blind (MASTER §6).** It is
   an "Extensible State-Object Processor": objects with message handlers, an
   explicit event queue (`post_event`/`dispatch_event`), 88 stack-VM opcodes, and
   ~178 runtime functions (128 core + ~50 Dungeon-Hack delta). The
   object/message/event-queue model *is* an `event → condition → action`
   substrate; the ~178 runtime functions are the action vocabulary to enumerate
   and map (many engine-specific: rendering/save/automap → named actions or
   engine services). We hold DAESOP's disassembler + opcode spec, the language
   manual, and Thirdeye as an oracle.

## 4. Phases

### Phase 0 — Foundation & risk-retirement  *(detailed)*

**Goal.** Stand up the project skeleton under the enforced build gates, record
the MASTER §11 open decisions, and retire the two lift risks *early* on real data
before any IR is frozen.

**Scope / tasks.**
- **Project structure.** Maven multi-module under `eu.virtualparadox:parent`
  (Java 21, all CODING_STANDARDS gates green from commit zero, ≥90% branch+line
  coverage). Initial module sketch: `ir` (IR schema + fail-closed loader),
  `importer-westwood`, `engine-core` (AD&D rules + runtime state), `render-libgdx`,
  `app`. (`importer-aesop` and `editor` come later.)
- **Resolve MASTER §11 open decisions, in writing:** stack = Java 21 + libGDX
  (confirmed); IR serialisation = human-diffable typed text (format TBD in P0);
  scripting model = declarative `event → condition → action`, closed vocabulary,
  no embedded VM; rendering-fidelity envelope = draw the explicit line (MASTER
  §7.5) — what resolution/input QoL is allowed before it becomes a remake.
- **AESOP lift spike (risk §6).** On real EOB3 data
  (`~/Games/…Myth Drannor.app/…/game/EYE.RES`): run DAESOP's disassembler
  (`abc_list.def`), pick 1–2 message handlers, hand-lift them into
  `event → condition → action`. Deliverable: a feasibility note on whether the IR
  event model absorbs the AESOP object/message pattern cleanly — *before* the IR
  is frozen.
- **`.INF` decompiler spike (Westwood).** Using `eoblib`'s `Script.java` as a
  starting reference, prototype lifting one EOB1 level script into
  `event → condition → action`, exercising the control-flow reconstruction.
- **PAK extraction spike (EOB1).** A small Westwood PAK extractor (format
  documented) so EOB1's `EOBDATA*.PAK` become loose files for P1.

**Demonstrable end state.** Green build with all gates; an IR draft v0; the
rendering-envelope decision written down; two lift-feasibility notes (Westwood +
AESOP); EOB1 data unpacked.
**Differential evidence.** N/A yet (no rendering); spikes judged by whether the
hand-lifts round-trip conceptually.
**Checkpoint.** Review the two spike notes and the §11 decisions before writing
any engine/IR code in earnest.

### Phase 1 — EOB1 vertical slice  *(detailed)*

**Goal.** One EOB1 level, end-to-end through the real IR: importer → IR → engine
renders it → one working trigger. No speculative framework (MASTER §10).

**Scope / tasks.**
- **`importer-westwood` (EOB1):** a clean-room port of `eoblib`'s parsers
  (CPS + Westwood LZ decruncher, VCN, VMP, MAZ, INF, DAT), emitting
  **framework-neutral** data (raw `int[]`/palettes, POJOs) — never `Sprite`/
  `BufferedImage`.
- **IR v1 (all three layers):** geometry/behaviour (cells, wall archetypes,
  decorations, base-vs-runtime); tileset (VCN atlas + VMP projection + palette +
  shading) as an appearance pack; event graph produced by the `.INF` mini-
  decompiler (from the P0 spike).
- **`engine-core` v1:** enough AD&D 2e rules and party/level state to walk a
  level and fire a trigger; AD&D tables live engine-side.
- **`render-libgdx` v1:** reproduce the EOB1 first-person view (faithful to the
  original VMP projection *for differential testing*), but keep the projection in
  the tileset pack, not in level logic — so a parameterised renderer can replace
  it later. Reuse `darkmoor`'s A–O visibility model + far-to-near draw order
  (game-invariant); avoid its hardcoded pixel/scale tables (anti-pattern).
- **Fail-closed IR loader** and the versioned schema from commit zero (MASTER §5).

**Milestone breakdown (M1–M5).** P1 is executed as five demonstrable,
individually differential-testable milestones — each grows the IR from real data
and ends with its own golden/differential evidence, never a horizontal layer
built ahead of use (MASTER §10). M1's detailed plan and execution record live in
`plans/2026-07-06-p1-m1-eob1-cps-decode.md`; later milestones are elaborated
just-in-time from the previous one's learnings.

| # | Milestone | Differential evidence | New surface | Status |
|---|---|---|---|---|
| **M1** | CPS decode + palette | Byte-exact SHA-256 of decoded indices vs real EOB1 data, cross-checked against the independent `eoblib` decoder; palette-paired eyeball | `importer-westwood`: CPS / Format80 / palette | **done** (`d17befb`) |
| **M2** | Static first-person render | Renders a correct EOB1 first-person view through the full real-IR pipeline (decoders eoblib-byte-exact; projection kyra-verified); maintainer-verified by eye. Pixel-exact kyra golden deferred (ScummVM not installed) | `beholder-ir` geometry + tileset layers; headless `beholder-render-core` | **done** (`d45ad63`) |
| M3 | Walkable level | Scripted-path structural invariants hold on real `LEVEL1.MAZ` (turn-cycle identity, never-off-grid, settle-at-wall, blocked-step no-op) + maintainer eyeball of the rendered walk; collision hand-verified across all four facings; pixel-exact kyra golden deferred (ScummVM not installed) | `engine-core` party + lockstep movement; `render-libgdx` (dynamic render) | **done** (`bf5f809`) |
| **M4** | `.INF` event graph + one trigger | One real lifted trigger (`SET_WALL`) fires end-to-end through the import-time lift (`event → condition → action`, not a VM) + a visitor executor; doors (real `DOOR.CPS` leaves, collision passable-iff-open) + `0xEC` decorations render on real `LEVEL1` (incl. the canonical collapsed-corridor entry); maintainer-verified by eye. Pixel-exact kyra golden deferred (ScummVM not installed); the door-open demo half is synthetic (real door triggers need item-count RPN, outside the minimal M4 vocab) | `beholder-engine-core` executor + `RuntimeState`; IR `event`/`Door` layers; render-core doors+decorations; composition-root `.INF`/asset bridges | **done** (`d5d85f4`) |
| M5 | Lossless round-trip | extract → IR (JSON) → render/play identically; byte-stable re-serialize | differential harness | next |

`render-libgdx` (the "render-libgdx v1" task above) enters at **M3**, when
interactivity is first needed; **M1–M2 render headless** to `int[]`/PNG so the
differential goldens stay framework- and windowing-free and run under the
coverage gate.

**Demonstrable end state.** EOB1's first level is walkable; **one working
trigger** (e.g. a pressure plate → door/teleport).
**Differential evidence (MASTER §8).** Render/state diff of that level against
`kyra` (and/or the DOSBox-run original); a lossless round-trip (extract → IR →
render). Golden references captured per level.
**Checkpoint.** Human review of the diff evidence and the IR v1 shape.

### Phase 2 — EOB2  *(medium detail)*

**Goal.** Validate the IR against a second real Westwood game — cheaply, same
engine family.

**Scope / tasks.** A second Westwood importer path for EOB2's diverging `.INF`
(marker bytes `0xEC`/`0xEA`/`0xFF`, 13-byte name slots, the block_A/B/C model,
2-byte block flags, different magic wall ids). Extend the wall-archetype enum and
the event vocabulary **only where a real game forces it** (never ahead of need).
Extend `engine-core` (EOB2 monster-property parsing, spell lists). Two importers,
one IR.
**Demonstrable end state.** An EOB2 level is playable through the same IR/engine.
**Differential evidence.** Diff vs `kyra` EOB2; round-trip.
**Checkpoint.** Human review; confirm the IR absorbed a second game without
distortion.

### Phase 3 — EOB3 / AESOP lift  *(sketch — elaborate JIT after P0 spike + P2)*

**Goal.** Retire the project's primary risk (MASTER §6): lift compiled AESOP
bytecode into the declarative IR.

**Scope (indicative).** `importer-aesop`: DAESOP-style disassembly → a lifter →
IR event graph. Map the 88 opcodes + the object/message/inheritance model into
`event → condition → action`; enumerate the ~178 runtime functions into the
action vocabulary (engine-specific ones — rendering/save/automap — become named
actions or engine services). **The IR will stretch here** — let it, from real
data. `abc_list.def` as the opcode spec; Thirdeye as an oracle; `EYE.RES` as
input.
**Demonstrable end state.** An EOB3 level is playable through the IR.
**Differential evidence.** Diff against Thirdeye / the DOSBox-run original (AESOP
`SET AESOP_DIAG=1` diagnostics as a probe).
**Checkpoint.** Human review; the IR event model is now proven against the
hardest source. *(Full task breakdown produced after the P0 spike and P2.)*

### Phase 4 — Dungeon Hack  *(sketch)*

**Goal.** Prove the authored-vs-generated split (MASTER §4.2): the platform is
not a museum.

**Scope (indicative).** Model the generator-spec (seed + parameters + tile/room
banks + encounter tables) as first-class IR that emits authored-like IR at load
time; map the Dungeon-Hack runtime-function delta (file I/O, dungeon rendering,
automap); `party_size = 1`, permadeath, and the exposed knobs (size, level count,
difficulty, item rarity, quest-item placement, light/auto-map) as the moddable
generator schema.
**Demonstrable end state.** A generated dungeon is playable; the engine cannot
tell it from an authored level.
**Differential evidence.** Same seed → reproducible dungeon; render/state parity
of generated levels with the original generator where feasible.
**Checkpoint.** Human review; the authored/generated boundary holds.

### Phase 5 — Editor  *(sketch)*

**Goal.** Author new content directly on the IR — only after it is frozen and
proven lossless (MASTER §10: editor last).

**Scope (indicative).** An editor as another IR client; the extract → edit →
save → play round-trip as the acceptance test (and the best modder tutorial:
learn on the Westwood levels).
**Demonstrable end state.** An edited/authored level is playable.
**Checkpoint.** Human review; round-trip is lossless.

## 5. Sequencing principles (from MASTER §10)

- Vertical slice before any framework; **editor last**; the AESOP lift **spiked
  early** (P0) as the risk-retirement step, with its full phase (P3) after two
  Westwood games have shaped the IR.
- No universal abstraction until two real games demonstrate the need.
- Each phase ends at a reviewable, demonstrable state with differential evidence,
  not a vibe.

## 6. Open decisions & risks

- **IR serialisation format** — resolve in P0 (human-diffable typed text).
- **Rendering-fidelity envelope** — draw the explicit line in P0 (MASTER §7.5).
- **AESOP action-vocabulary size** — the ~178 runtime functions may not all map
  to clean declarative actions; the engine-specific ones are the pressure point
  (candidates that tempt an embedded VM — resist per MASTER §4.1).
- **Differential-testing harness** — how to capture golden references from `kyra`
  and the DOSBox-run originals; define once, reuse per game (P1 sets it up).
- **Clean-room discipline** — GPL sources (`kyra`, Thirdeye) are read-only
  oracles; never copy. The original Westwood EOB2 source is off-limits. AESOP
  package + DAESOP are public domain (safe to derive; adopt `abc_list.def`).
