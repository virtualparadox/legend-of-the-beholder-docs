# P1 · M3 — EOB1 walkable level (discrete lockstep movement + libGDX front-end) — Design

> **Status:** design (brainstorming deliverable). The detailed, TDD task-by-task
> implementation plan is produced *against this design* by `superpowers:writing-plans`
> and appended below as "Tasks" once the maintainer approves this design. This document
> fixes the module structure, the movement/collision model, the interaction envelope, the
> differential-correctness strategy, and the coverage strategy — the decisions that shape
> the code, before any is written.

**Goal.** Make **EOB1 level 1 walkable**: discrete lockstep movement (step / turn) with
collision against the wall archetypes (`OPEN` = passable, `SOLID` = blocks), driven by a real
interactive **libGDX** window that wraps the existing headless renderer. This is the milestone
where the engine gains party/level *runtime state* and where the renderer becomes *dynamic*
(a frame per discrete state along a walked path), differentially checked as a frame sequence.

**Non-goals (explicit, deferred).** No triggers/events (`.INF` bytecode → M4), no doors/decoration
overlays (M4's `0xFB` table), no characters/AD&D stats/combat (arrive when a trigger or combat first
needs them), no auto-map, no inventory/UI side-panels, no movement tween, no JSON round-trip (M5),
no mouse interaction. M3 is *walking and looking*, nothing more (MASTER §10: vertical slice, no
speculative framework).

## Where this sits — the P1 milestone ladder

This is **M3** of the five-milestone P1 EOB1 vertical slice (`PHASES.md` §"Phase 1"). M1 (CPS decode
+ palette, `d17befb`) and M2 (static first-person render, `d45ad63`) are **done**. M3 adds the
`beholder-engine-core` (party + lockstep movement + level runtime state) and `beholder-render-libgdx`
(the first interactive front-end); the renderer becomes dynamic. M4 adds the `.INF` event graph + one
trigger; M5 the JSON lossless round-trip. M3's differential retires the **movement/collision fidelity
risk** (does our step/turn/collision match the original's move table) and proves the
render+movement composition is self-consistent frame-by-frame.

## Decisions settled in brainstorming (2026-07-07)

- **engine-core scope = position + facing only (YAGNI).** The party's runtime state is a
  `GridPosition` + a `Facing`; **no characters, no AD&D stats**. "Just enough AD&D 2e to walk" is,
  literally, no rules — walking needs none. Characters/AD&D content arrive at the milestone where a
  trigger or combat first requires them (M4+). The base-vs-runtime split is realised on the
  **`Level` (immutable base: geometry + tileset) vs `RuntimeState` (mutable: the party)** axis.
- **No movement/turn tween in M3 — instant discrete snap** (ADR-0003 M3 ruling #1). The render layer
  stays stateless: `FirstPersonRenderer.render(state) → int[]`. A guarded presentation-only tween is a
  later appearance-QoL, explicitly out of scope here.
- **Aspect/FoV = pillarbox at original FoV** (ADR-0003 M3 ruling #2). The window shows only the
  176×120 viewport, aspect-preserving nearest-neighbour, pillarboxed; no FoV widening.
- **Discrete input = structural** (ADR-0003 M3 ruling #3). Keyboard can only emit a closed
  `MovementCommand`; the engine exposes no continuous position/orientation/FoV API. Illegal
  (continuous) states are unrepresentable (MASTER §7.2).
- **Coverage-gate for the irreducible GL leaf = narrow, documented JaCoCo exclusion**, analogous to
  the existing `**/model/**` exemption. All *decision logic* stays in-gate (see Coverage strategy).

See `decisions/adr-0003-rendering-fidelity-envelope.md` → "M3 grey-area rulings (2026-07-07)" for the
authoritative envelope record.

## Architecture — two new modules

Current reactor (acyclic): `beholder-ir ← beholder-importer-westwood`, `beholder-ir ←
beholder-render-core`. M3 adds two modules; the graph stays acyclic:

```
beholder-ir  ─────────────┬──────────────┬───────────────┐
                          │              │               │
              beholder-importer-westwood │               │
                                 beholder-render-core     │
                          beholder-engine-core ───────────┤
                                                          │
                                     beholder-render-libgdx
                     (depends on: engine-core + render-core + libGDX)
```

- **`beholder-engine-core`** — headless, framework-neutral, **fully gated**. Depends on
  `beholder-ir` only (never on render-core — the engine does not render).
  - `MovementCommand` (enum): `FORWARD, BACKWARD, STRAFE_LEFT, STRAFE_RIGHT, TURN_LEFT, TURN_RIGHT`
    — the closed input vocabulary.
  - `Party` (immutable value): `GridPosition position` + `Facing facing`.
  - `Level` (immutable base): `LevelGeometry geometry` + `TileSet tileset` — the appearance pack
    bundled with its geometry; the immutable base the runtime never mutates in M3.
  - `RuntimeState` (mutable overlay): holds the current `Party`. In M3 the party is the *only* thing
    that mutates. Shaped so M4 can add a level-mutation overlay (opened doors, moved walls) without
    reshaping this boundary.
  - `MovementEngine.apply(MovementCommand) → RuntimeState`/new `Party`: the lockstep transition,
    enforcing collision against `Level.geometry`. The load-bearing correctness surface.
  - **IR addition:** `Facing.left()` / `right()` / `opposite()` — pure facing algebra that belongs
    next to the existing `Facing.delta(...)` (a reviewed, closed-enum schema change, not a property
    bag). Turn commands use these; move commands use `delta`.

- **`beholder-render-libgdx`** — the interactive front-end. Depends on `beholder-engine-core` +
  `beholder-render-core` + libGDX (LWJGL3). Split so almost everything is gate-tested and only the
  irreducible GL boundary is excluded:
  - `InputMapper` (pure, no GL): libGDX `Input.Keys` int keycode → `Optional<MovementCommand>`.
    Uses libGDX's static int key constants but needs no GL context → plain-JUnit testable, **in-gate**.
  - `GameSession` (pure, injected seams): owns the `RuntimeState` + `Level`; `onCommand(command)`
    applies it via `MovementEngine` and re-renders the frame via `FirstPersonRenderer.render(...)`;
    `currentFrame() → int[176*120]`. Driven through an injected `InputSource` + `FrameSink` so the
    poll→command→apply→render→present loop is fully covered with fakes, **in-gate**.
  - **Irreducible GL leaf (JaCoCo-excluded, documented):** `LibGdxInputSource` (polls
    `Gdx.input.isKeyJustPressed`), `GlFrameSink` (uploads the `int[]` to a `Pixmap`→`Texture`, draws
    it aspect-preserving via `ViewportFit`), and `Main` (the `Lwjgl3Application` launcher). No branch
    logic — the untestable-without-a-display boundary.
  - `ViewportFit` (pure aspect/letterbox math) lives in **`beholder-render-core`** (it is
    presentation, headless, needs no libGDX): `(windowW, windowH, 176, 120) → dest rect`,
    aspect-preserving, no widening. **In-gate** in render-core.

## Movement & collision model (the load-bearing bit)

- **Turn** (`TURN_LEFT` / `TURN_RIGHT`): `facing = facing.left()` / `right()`; position unchanged;
  **never blocked**. Turn cycle is `N → E → S → W → N` (right) and the reverse (left).
- **Move** (`FORWARD` / `BACKWARD` / `STRAFE_LEFT` / `STRAFE_RIGHT`): compute the facing-relative grid
  delta from the party's local axes via the existing `Facing.delta(localX, localY)`:
  - `FORWARD` = `delta(0, +1)`, `BACKWARD` = `delta(0, -1)`,
    `STRAFE_RIGHT` = `delta(+1, 0)`, `STRAFE_LEFT` = `delta(-1, 0)`.
  - Target cell = `(party.x + dCol, party.y + dRow)`.
- **Collision invariant:** the move is **blocked** iff (a) the target is off the 32×32 grid, **or**
  (b) the wall archetype crossed in the travel direction is `SOLID`. Travel-direction → edge `dir`:
  `(0,-1)→N=0`, `(+1,0)→E=1`, `(0,+1)→S=2`, `(-1,0)→W=3`; then test `geometry.archetype(x, y, dir)`.
  `OPEN` → move; `SOLID` (or off-grid) → stay.
- **The one thing to pin in TDD against the oracle:** *which* edge is authoritative — the current
  cell's edge in the travel direction, the target cell's opposite edge, or both must agree. EOB stores
  walls per block and they are usually symmetric, but the collision rule must match the original. Pin
  it against **kyra `scene_rpg.cpp`** movement (`calcNewBlockPosition` / `blockPosTable`) and
  cross-check against **eoblib**, exercised on the real `LEVEL1.MAZ`. The invariant above ("SOLID in
  travel direction blocks") is fixed; the authoritative-edge detail is a TDD decision, not a design
  open question.
- **Oracle for the move deltas:** kyra's move table `{-32, +1, +32, -1}` for N/E/S/W (= `dRow*32 +
  dCol`), which `Facing.delta` already encodes (verified at M2). M3 adds only the collision-edge
  mapping and the turn algebra.
- **Illegal states unrepresentable:** `Party.position` is always a validated `GridPosition` (factory
  rejects out-of-range), `facing` is always one of four — no continuous position/orientation is ever
  representable (ADR-0003 structural enforcement).

## Interaction envelope (render-libgdx v1)

- **Window:** renders only the 176×120 first-person viewport, uploaded to a `Texture` and drawn
  aspect-preserving (nearest-neighbour, pillarbox/letterbox), no FoV widening. Same frustum / A–O
  visibility as M2 — `beholder-render-core` is **unchanged** (M3 only *drives* it dynamically).
- **Input:** discrete keyboard → `MovementCommand` (proposed default: `↑`=forward, `↓`=backward,
  `←`/`→`=turn-left/right, and two strafe keys, e.g. `,`/`.` or `Q`/`E`; exact bindings pinned in the
  plan). One key-press = one command = one discrete transition; no continuous glide.
- **No mouselook / continuous camera** — structurally impossible (no continuous API exists).

## Differential correctness (§8) — the M3 milestone

ScummVM is **not installed** (only source in `~/Workspaces/CPP/scummvm`), so — matching M2's precedent
— the M3 differential stands on three legs, primary first:

1. **Scripted-path state test (in-gate, deterministic — the primary proof).** A fixed
   `MovementCommand` sequence over the real `LEVEL1.MAZ` (an `*IT` that `assumeTrue`-skips without
   game data, like the M2 goldens) drives `MovementEngine`; assert the resulting `(position, facing)`
   sequence equals a hand-computed expected sequence derived from the kyra move-table oracle —
   including the collision cases: step into a `SOLID` wall → position unchanged; step through an
   `OPEN` passage → advances; full turn cycle `N→E→S→W→N`; a grid-edge step → blocked. This *state*
   differential is the load-bearing correctness claim and runs entirely under the coverage gate on
   pure logic.
2. **Self-render frame-sequence ratchet (regression guard).** Render each frame of that path headless
   via `FirstPersonRenderer` → freeze a **SHA-256 sequence golden** (the self-render ratchet M2
   deferred until fidelity settled — M3 is a natural place to introduce it as a *regression* pin, not
   a ground-truth claim). Locks the render+movement composition against silent regressions.
3. **Maintainer eyeball (human verdict).** The interactive libGDX window (walk level 1) and/or a
   rendered **frame-strip / contact-sheet** of the fixed path relayed via `SendUserFile` for verdict —
   the M2-style final leg, given no kyra reference.
4. **Opportunistic (non-blocking):** if ScummVM is later built/installed from source, the frame
   sequence can be pixel-diffed against kyra — same deferred posture as M2's kyra golden.

## Coverage strategy (the JaCoCo-gate detail)

The `CODING_STANDARDS` ≥90% branch+line gate applies per bundle. Plan:

- **`beholder-engine-core`** — 100% pure logic; every branch (each command, each collision outcome,
  each turn) is unit-tested. Fully in-gate.
- **`beholder-render-core`** — `ViewportFit` added here is pure math; every branch (portrait vs
  landscape window, integer vs fractional scale, exact-fit) tested. In-gate.
- **`beholder-render-libgdx`** — `InputMapper` (plain-JUnit: every keycode arm + the unmapped-key
  empty case) and `GameSession` (fakes for `InputSource`/`FrameSink`: apply-command, re-render,
  present) are in-gate. The irreducible GL leaf (`LibGdxInputSource`, `GlFrameSink`, `Main`) gets a
  **narrow, documented JaCoCo `<exclude>`** — the same category as `**/model/**`: no behaviour,
  untestable without a display. The exclusion list is minimal and commented with *why* each class is
  on it (the maintainer's magic-value/why-comment discipline extended to build config). Confirm the
  exact JaCoCo `check` mechanism (per-module thresholds + `excludes`) against the parent pom during
  planning, and that libGDX/LWJGL3 builds and runs on **JDK 21** (a small bring-up spike is Task B's
  first step).

## Sequencing — first concrete step + task-shape sketch

**First concrete step (recommended): `beholder-engine-core` movement/collision first — headless, TDD,
in-gate — before any libGDX.** It is the load-bearing correctness surface, the primary §8 differential
(a state test) needs the engine not the window, it is fully gate-testable, and it retires the one real
risk (collision-edge semantics vs kyra) up front. The libGDX window is *plumbing* once engine +
render-core exist, and its correctness is eyeball, not gate.

Task shape (calibrated to risk, per the maintainer's process — light/inline for mechanical,
subagent-driven for risky/novel):

- **Task A — `beholder-engine-core`: movement + collision** (*medium risk*: the collision-edge-vs-kyra
  semantics is the oracle cross-check; the rest is mechanical). Inline TDD or one fresh implementer +
  independent review. Delivers: `Facing.left/right/opposite` (IR), `MovementCommand`, `Party`,
  `Level`, `RuntimeState`, `MovementEngine`, and the **scripted-path state IT + self-render ratchet**.
- **Task B — libGDX bring-up + `beholder-render-libgdx`** (*fiddly integration, low correctness risk,
  novel coverage strategy*): first a small spike confirming libGDX/LWJGL3 on JDK 21 + the JaCoCo
  exclusion mechanism, then `InputMapper`, `GameSession`, `ViewportFit` (in render-core), and the GL
  leaf + `Main`. One subagent.
- **Task C — differential harness + eyeball**: assemble the frame-strip/contact-sheet from the scripted
  path, relay via `SendUserFile` for the maintainer's verdict; wire the interactive walkthrough.
- End: whole-branch **opus review**, `--no-ff` merge to `main`, **no worktree** (conventional
  branch/commit).

## Open items to pin during planning / TDD (not design open questions)

1. Authoritative collision edge (current-cell vs target-cell vs both-agree) — pin vs kyra + eoblib on
   real `LEVEL1.MAZ`.
2. Exact keyboard bindings for the six commands.
3. JaCoCo `check` mechanism + the precise `<exclude>` entries; libGDX/LWJGL3-on-JDK-21 confirmation.
4. libGDX + LWJGL3 dependency coordinates/version (cross-check `darkmoor` as the stack reference; do
   **not** copy its hardcoded pixel/scale tables — anti-pattern, MASTER §9/§11).
5. Whether the self-render ratchet is a single concatenated-sequence SHA-256 or per-frame hashes
   (regression-guard ergonomics).

## Self-Review (design)

- **Placeholders:** none — every section is concrete; the items above are explicitly *implementation*
  pins, not design gaps.
- **Consistency:** the module dependency graph is acyclic (engine-core and render-core both depend only
  on ir; render-libgdx depends on both + libGDX). `ViewportFit` in render-core and `InputMapper` in
  render-libgdx are consistent with "engine-core never renders / render-core is framework-neutral".
  The stateless-renderer decision (no tween) is consistent with the ADR-0003 ruling and the §8
  frame-sequence differential.
- **Scope:** focused on *one walkable level* (MASTER §10 vertical slice); triggers/doors/characters/
  auto-map/round-trip are explicitly deferred to their owning milestones.
- **Ambiguity:** the single genuine ambiguity (collision edge authority) is named and assigned to TDD
  against a concrete oracle, with the fixed invariant stated so the ambiguity cannot widen scope.

---

**Next:** on the maintainer's approval of this design, `superpowers:writing-plans` expands it into the
TDD task-by-task implementation plan (Tasks A–C above, each with RED→GREEN steps, `mvn verify`
evidence, and per-task review), appended to this document.
