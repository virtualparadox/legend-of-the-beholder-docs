# Grid-Crawler Engine Platform — Master Specification (Project Spine)

> **Status:** canonical spine / single source of truth.
> **Purpose of this document:** to be used both as a human reference and as a *prompt seed* for later, phase-by-phase planning. It fixes the intent, the architecture, the invariants, and the sequencing philosophy. It deliberately does **not** contain a phase breakdown — that is produced later, against this document.
> **Working title:** TBD.

---

## 1. One-paragraph pitch

A single, clean, extensible engine that plays all four SSI/Westwood grid-based dungeon crawlers — *Eye of the Beholder I*, *II*, *III*, and *Dungeon Hack* — by importing each game's original data into one **canonical intermediate representation (IR)**, then running that IR. The engine never touches original game files; importers do. The IR is a real content format, so the same pipeline that revives the four originals also lets anyone author new campaigns and generators. The product is not a remake of any one game — it is the unification, plus the platform that unification makes possible.

## 2. Core thesis

- **The IR is the product.** The engine, the importers, and the editor are all clients of one canonical schema. The reverse-engineering knowledge already largely exists (see §9); the value we add is *coherent synthesis* onto one spine — which nobody has done for these games.
- **Not a remake, a platform.** Once the engine only eats IR, moddability and new-content authoring fall out for free. That is what outlives the "four old games" scope.
- **The synthesis is the hard part, not the RE.** Reading a documented format is well-defined work. Drawing the IR boundary — where the data/behaviour line sits, how bytecode is lifted, how all four games fit without distortion — is open design work. That is where this project succeeds or degrades into a retrofit-hacked EOB1 viewer.

## 3. Scope

**In scope (the four "official campaigns"):**

| Game | Engine | Nature |
|---|---|---|
| Eye of the Beholder I | Westwood | Authored, hand-built levels |
| Eye of the Beholder II | Westwood (same family) | Authored |
| Eye of the Beholder III | SSI **AESOP** bytecode VM | Authored |
| Dungeon Hack | SSI **AESOP** (improved variant) | **Procedurally generated** |

**Out of scope (explicitly):**
- Turning any of these into a modern game — no free mouselook, no continuous movement, no hi-res/PBR art. Authentic scarcity is a feature (see §7).
- Shipping any original game assets. Engine only; users bring their own original data (see §9, legal model).

## 4. Architecture

```
        ┌───────────┐   ┌───────────┐   ┌────────────────┐
originals│ importers │──▶│ canonical │──▶│  clean engine  │
(4 games)│ (per src) │   │    IR     │   │ (eats IR only) │
        └───────────┘   └───────────┘   └────────────────┘
                              ▲   ▲
                        ┌─────┘   └──────┐
                    ┌────────┐      ┌────────┐
                    │ editor │      │  mods  │
                    └────────┘      └────────┘
```

- **Importers = anti-corruption layer.** All the mess of the original formats (Westwood `.INF`/CPS/PAK/VCN/VMP; AESOP RES/bytecode; the DH generator) is quarantined here. The engine never sees an original file, only IR.
- **One runtime for all four**, with uniform QoL (auto-map, save-anywhere, hi-res rendering of period art, mouse marking). Today each game needs a different launch ritual (ScummVM / DOSBox+fanpatch / etc.); one runtime replacing all four is itself a thing you can't get in one place today.
- **Editor and mods are additional clients of the same IR** — not separate codebases.

### 4.1 Two hard separations (non-negotiable)

**Data vs behaviour.** Triggers and bytecode are *behaviour*, not static data. They must be **lifted** into a declarative event model — `event → condition → action` — with a **closed, enumerated action vocabulary** (pressure plate, teleport, spawn, text, item-gate, …). Do **not** embed an interpreter to run original-style logic inside the clean engine. If something does not fit the vocabulary, that is a signal the vocabulary is incomplete — not a licence to add a VM.

**Rules vs content.** The AD&D 2e ruleset (THAC0, saving throws, spell memorisation, resolution logic) is **engine-side** and **shared across all four games**. The IR carries *content* (monster stats, item definitions, encounters, level geometry, triggers) — never the resolution rules. Dungeon Hack's single-character mode is just a `party_size` parameter, **not** a rules fork.

### 4.2 Content is authored **or** generated (first-class)

The IR models a level as **either** an authored instance **or** a **generator-spec** (seed + parameters + tile/room banks + encounter tables). This distinction exists from day zero, typed and fail-closed-validated like everything else.

- Generation runs **at load time**, in one clean step, emitting *authored-like IR*. The engine cannot tell whether a level was hand-built or generated — so rendering, differential tests, and save logic are identical for both.
- Dungeon Hack's exposed knobs (dungeon size, level count, monster difficulty, item rarity, quest-item placement, permadeath, auto-map/light) **are** the schema of the moddable generator. Formalise them.

## 5. The IR — design principles

The IR is the heart. It must:

- Be **versioned from commit zero** (importer, engine, editor, and mods all evolve independently against it).
- Be **validated fail-closed on load** — unknown/invalid content is rejected, never "best-effort" rendered.
- Use a **closed, enumerated vocabulary** for tiles, wall-decoration slots, and the event/condition/action language — not an open property bag.
- Be **designed against all four games at once.** Do **not** shape the IR around EOB1 (the gentlest source). **AESOP's event/trigger model is the stress test** — if it fits cleanly, the rest follows.
- Carry **content, not rules** (see §4.1).
- Distinguish **authored vs generated** content (see §4.2).

It must **not**:
- Leak original formats into the engine (if you can't build a new campaign purely via IR, a format is still leaking).
- Encode AD&D resolution logic.
- Grow speculative "universal" abstractions ahead of two real games proving they're needed.

## 6. Primary risk (address first, not last)

**Lifting AESOP bytecode into the declarative IR.** EOB3/Dungeon Hack logic *is* compiled AESOP code — the "data-is-actually-code" tension in its hardest form (the Westwood `.INF` triggers are lighter by comparison — though reconnaissance showed they are themselves a small imperative bytecode VM with jumps, a call stack and an RPN condition evaluator, so the *lift* competency must exist from the first Westwood importer, not only here; AESOP is a much larger VM with its own object/message language). This lift is the research core of the whole project and the thing most likely to blow up the IR design.

**Mitigation:** prototype the AESOP → declarative lift **early**, on real EOB3 data, before freezing the IR. You are not decoding blind — there is a disassembler, a reference reimplementation, and the original language manual (see §9).

## 7. Design invariants (the character of the thing)

1. **Fail-closed everywhere** — importer, IR loader, mod loader, editor. Untrusted input (mods especially) is rejected on any unknown construct, not tolerated.
2. **Strong typing / invariants in the type system** — enumerated vocabularies, immutable IR values, illegal states unrepresentable.
3. **The engine eats IR and nothing else.** No original-format code path reaches the runtime.
4. **Period authenticity lives in constraints, not goodwill.** Index-colour tile banks, fixed grid, lockstep movement, enumerated decoration slots, closed action vocabulary. The tooling should make correct-era content the *only* thing you can produce — not merely the recommended thing.
5. **Do not drift into a remake.** The moment free mouselook / continuous movement / hi-res sprites creep in "to look nicer," the reason the project exists is dead. Grimrock exists for that. The value here is authentic scarcity.
6. **Moddability is emergent, not bolted on.** It is a consequence of invariant #3, not a feature.
7. **Correctness is proven, not asserted** (see §8).

## 8. Correctness strategy

Differential testing against ground truth, applied to level/render/state — the "karantén" reflex, at the data level:

- **Ground truth sources:** ScummVM's GPL `kyra` engine (Westwood EOB1/2), the original runtime, and AESOP's built-in diagnostics (`SET AESOP_DIAG=1` enables walk-through-walls, jump-to-coords, etc.).
- **Method:** render the same level via our engine and diff (pixel and/or state) against the reference. A reimplementation that provably matches the original is a quality claim none of the current scattered solutions make.
- **Round-trip as acceptance test:** extractor output must load in the editor and re-export to IR the engine plays identically (extract → edit → save → play → back). A lossless round-trip proves the IR is faithful and is simultaneously the best tutorial for modders (they learn on the Westwood levels).
- Keep **golden references** per game and per level; ratchet, don't regress.

## 9. Prior art, resources, and legal model

**Westwood (EOB1/2):**
- ScummVM **`kyra`** engine — a GPL, readable, reverse-engineered reimplementation of the Westwood engine; documents CPS/PAK/VCN/VMP and the `.INF`/`.DAT` level scripting. Primary reference and ground truth.
- Archived original EOB2 Westwood source exists (Internet Archive) — **legally radioactive to copy from**. Clean-room via the GPL `kyra` source is the safe path. Do not read it if you intend a non-GPL clean-room implementation.

**AESOP (EOB3 + Dungeon Hack):**
- **John Miles**, AESOP's original author, released the AESOP *package* (language, 32-bit compiler, manual) as effectively **public domain** — but deliberately withheld the runtime interpreter because it was too entangled with SSI-owned code. So the language semantics come cleanly from the source; the original runtime does not.
- **DAESOP** (Mirek Luza) — resource extractor/replacer + **bytecode disassembler** (symbolic names for locals/params) + resource-type/string inspection + EOB3→AESOP/32 conversion. Effectively the seed of our AESOP importer.
- **Thirdeye** (psi29a / mindwerks, OpenMW-adjacent) — GPL AESOP reimplementation targeting EOB3 + Dungeon Hack. **Dormant** (~2015, small repo), but a real, readable reference for opcode / system-call behaviour. GPL: read to learn, do not copy into a non-GPL codebase.
- **VOGONS AESOP thread** (`vogons.org`, topic 20601) — the RE hub: tool backups, John Miles' posts, and `ADDITIONAL_DH_RUNTIME_FUNCTIONS.TXT` (the Dungeon Hack function delta over AESOP/32 — the DH generator's documented extra surface).
- James Lacey's abandoned ScummVM AESOP subengine attempt, and **The Cutting Room Floor** (RES internals, unused content), as secondary references.

**Remake references (half-finished — read for approach, not to depend on):**
- **darkmoor** — Java/libGDX EOB2 remake (relevant: our likely stack, see §11).
- **Dungeon Eye** — C# EOB2 remake with a resource system + editor.

**Legal model:** reimplement the engine; the user brings their own original data (the DevilutionX / OpenMW model). Ship no assets. This keeps the project clean regardless of the SSI/D&D (Wizards/Hasbro) rights tangle.

### 9.1 Local environment paths (this workstation)

Concrete local paths for the oracles and ground-truth this project builds against. The
GPL/original sources are **read-only oracles** — clean-room, never copied into the engine.

- **ScummVM / `kyra` source** (read-only clean-room oracle for the Westwood algorithms —
  render projection, `.INF`/`.DAT`, CPS/VCN/VMP): `~/Workspaces/CPP/ScummVM` (case-insensitive
  APFS, so `~/Workspaces/CPP/scummvm` also resolves); the kyra engine at
  `~/Workspaces/CPP/ScummVM/engines/kyra` (e.g. `scene_rpg.cpp`'s `generateBlockDrawingBuffer`,
  `staticres_*`'s `_vmpOffsets*`/`_dsc*` tables). **Source only — no built binary** (building
  ScummVM would be a separate task).
- **Original games** (the user's own data — MASTER §9 legal model; the engine ships none) under
  `~/Games/`: EOB1 `Eye of the Beholder.app`, EOB2 `Eye of the Beholder II The Legend of
  Darkmoon.app`, EOB3 `Eye of the Beholder III Assault on Myth Drannor.app`, `Dungeon Hack.app`.
- **EOB1 data + runnable pixel ground-truth**: `~/Games/Eye of the Beholder.app/Contents/Resources/game/`
  (`EOBDATA1..6.PAK`, `EYE.PAK`). The bundle ships **DOSBox**
  (`~/Games/Eye of the Beholder.app/Contents/Resources/dosbox`, sources under `…/Extras/dosbox/`),
  so the **original EOB1 is runnable for pixel-exact ground-truth capture** — the missing oracle
  the M2–M4 render goldens were "maintainer-eyeball" for. Use it to pixel-verify the first-person
  render (VMP slot geometry, wall/door projection) against the real game.
- **Engine repo**: `~/Workspaces/Java/legend-of-the-beholder`. **Docs repo** (this document,
  ADRs, plans, PHASES): `~/Workspaces/Java/legend-of-the-beholder-docs`. **Build-config parent**
  (`eu.virtualparadox:parent`, the enforced gates): `~/Workspaces/Java/parent`.
- **EOB RE wiki** (EOB1/2 format specs + decompiled level scripts, graphics-only): see PHASES §2
  (`~/Documents/EOB-unzip/eob.phpwiki`). **AESOP tooling** (EOB3/DH, later phases):
  `~/Workspaces/AESOP/` (Thirdeye, DAESOP, the AESOP/32 manuals — PHASES §2).

## 10. Sequencing philosophy

Governs *how* phases are planned later. Not a phase list.

- **Vertical slice before any framework.** First target: **EOB1, one level, end-to-end through the real IR** — importer → IR → engine renders it → one working trigger. No speculative abstraction.
- **Then EOB2** — cheap; same Westwood engine, validates the IR against a second real game.
- **Then the AESOP lift** (EOB3) — the risk (§6); it will tell you where the IR must stretch. Let the schema grow from real games, never top-down.
- **Then Dungeon Hack** — the generator; proves the authored-vs-generated split (§4.2) and that the platform is not a museum.
- **Editor last** — only after the IR is frozen and proven lossless. UI is ~10× slower to build than engine core and never "done"; building it against a moving IR means writing it twice.
- **Human-gated checkpoints** between phases. Each phase ends at a reviewable, demonstrable state with differential evidence, not a vibe.

## 11. Open decisions (resolve in Phase 0)

- **Stack.** Natural lean: **Java (21) + libGDX** — matches the maintainer's ecosystem and the existing `darkmoor` reference. Confirm vs alternatives before committing. (This is the only place the maintainer's day-job stack should bias the design; the architecture itself is language-agnostic.)
- **Scripting model, final call:** declarative `event → condition → action` (target: ≥95% of cases) with the explicit rule that a miss means *extend the vocabulary*, not *embed a VM*. Confirm the vocabulary is expressive enough by prototyping the AESOP lift (§6) first.
- **IR serialisation format** (human-diffable and versionable — favour text/typed schema over opaque binary).
- **Rendering fidelity envelope** — exactly how much QoL (resolution, input) is allowed before it violates §7.5. Draw this line explicitly and defend it.

## 12. How to use this document (for phase planning)

> The concrete, recon-grounded phase breakdown produced against this spine lives
> in **`PHASES.md`** — a living companion document. This spine stays phase-free by
> design (§1); `PHASES.md` holds the phases and evolves as real games shape the IR.

When planning phases against this spine:
1. Treat §§3–8 as **fixed constraints**, not suggestions. Any proposed phase that violates an invariant in §7 or the separations in §4.1 is wrong by construction.
2. Respect the ordering discipline in §10: **vertical slice first, editor last, AESOP lift early as the risk-retirement step.**
3. Each phase must define: its demonstrable end state, its differential-correctness evidence (§8), and its human-gated checkpoint.
4. Do not introduce a universal abstraction until two real games have demonstrated the need.
5. Resolve §11 in Phase 0 before writing engine or IR code.
6. The deliverable of phase planning is a sequence of small, demonstrable, independently reviewable phases — each one a working vertical increment on the real IR, never a horizontal framework layer built ahead of use.