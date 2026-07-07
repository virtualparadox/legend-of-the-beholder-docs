# P1 · M3 — EOB1 walkable level (discrete lockstep movement + libGDX front-end) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make **EOB1 level 1 walkable** — discrete lockstep movement (step / turn) with collision against the wall archetypes (`OPEN` = passable, `SOLID` = blocks), driven by a real interactive **libGDX** window that wraps the existing headless `beholder-render-core`.

**Architecture:** Two new modules over the existing IR + headless renderer. `beholder-engine-core` (depends on `beholder-ir` only) holds the party runtime state and the movement/collision engine — a pure, fully-gated headless unit. `beholder-render-libgdx` (depends on `beholder-engine-core` + `beholder-render-core` + `beholder-importer-westwood` + libGDX) is the interactive front-end and composition root: it maps discrete keyboard input to a closed `MovementCommand` vocabulary, drives the engine, renders each discrete state through `FirstPersonRenderer`, and blits the resulting `int[176*120]` ARGB frame to the window aspect-preserving. All decision logic lives in gated units; only the irreducible GL boundary (`…/render/libgdx/gl/**`) carries a narrow, documented JaCoCo exclusion.

**Tech Stack:** Java 21, `eu.virtualparadox:parent`, JUnit Jupiter + AssertJ. libGDX **1.12.1** (`gdx` + `gdx-backend-lwjgl3` + `gdx-platform:natives-desktop`) — the reactor's first UI dependency; darkmoor (libGDX 1.5.3 / LWJGL2 / Java 6) is *not* a coordinate reference. PNG eyeballs use JDK-only `javax.imageio` in test/scratch scope. Build on JDK 21 (Corretto 21; reactor enforces `[21,22)` fail-fast).

> **Status of this document.** Sections up to "Sequencing" are the approved *design* (brainstorming deliverable, committed `3f3e63d`). "Module / File Structure" onward is the *implementation plan* (writing-plans). The design's differential leg #2 is refined below (state-sequence golden instead of a churning frame-hash ratchet — see "Differential correctness").

## Where this sits — the P1 milestone ladder

This is **M3** of the five-milestone P1 EOB1 vertical slice (`PHASES.md` §"Phase 1"). M1 (CPS decode + palette, `d17befb`) and M2 (static first-person render, `d45ad63`) are **done**. M3 adds `beholder-engine-core` (party + lockstep movement + level runtime state) and `beholder-render-libgdx` (the first interactive front-end); the renderer becomes dynamic. M4 adds the `.INF` event graph + one trigger; M5 the JSON lossless round-trip. M3's differential retires the **movement/collision fidelity risk** and proves the render+movement composition is self-consistent frame-by-frame.

**Non-goals (explicit, deferred).** No triggers/events (`.INF` bytecode → M4), no doors/decoration overlays (M4's `0xFB` table), no characters/AD&D stats/combat (arrive when a trigger or combat first needs them), no auto-map, no inventory/UI side-panels, no movement tween, no JSON round-trip (M5), no mouse interaction. M3 is *walking and looking*, nothing more (MASTER §10: vertical slice, no speculative framework).

## Decisions settled in brainstorming (2026-07-07)

- **engine-core scope = position + facing only (YAGNI).** The party's runtime state is a `GridPosition` + a `Facing`; **no characters, no AD&D stats**. The base-vs-runtime split is realised on the **`Level` (immutable base: geometry + tileset) vs `RuntimeState` (mutable: the party)** axis.
- **No movement/turn tween in M3 — instant discrete snap** (ADR-0003 M3 ruling #1). The render layer stays stateless: `FirstPersonRenderer.render(state) → int[]`.
- **Aspect/FoV = pillarbox at original FoV** (ADR-0003 M3 ruling #2). The window shows only the 176×120 viewport, aspect-preserving nearest-neighbour, pillarboxed; no FoV widening.
- **Discrete input = structural** (ADR-0003 M3 ruling #3). Keyboard can only emit a closed `MovementCommand`; the engine exposes no continuous position/orientation/FoV API.
- **Coverage-gate for the irreducible GL leaf = narrow, documented JaCoCo exclusion**, analogous to the existing `**/model/**` exemption. All decision logic stays in-gate.

See `decisions/adr-0003-rendering-fidelity-envelope.md` → "M3 grey-area rulings (2026-07-07)".

## Architecture — two new modules

Current reactor (acyclic): `beholder-ir ← beholder-importer-westwood`, `beholder-ir ← beholder-render-core`. M3 adds two modules; the graph stays acyclic:

```
beholder-ir  ◀── beholder-importer-westwood ──┐
     ▲  ▲                                      │  (importer: production dep of the app/composition root)
     │  └── beholder-render-core ──────────────┤
     └────── beholder-engine-core ─────────────┤
                                               ▼
                                     beholder-render-libgdx
              (engine-core + render-core + importer-westwood + libGDX)
```

`beholder-render-libgdx` is the **app / composition root**: it legitimately orchestrates the importer (PAK → IR) then feeds IR to the engine and renderer. The *engine* (`engine-core`) still only ever sees IR — MASTER §4's "engine eats IR only" invariant holds; the composition root wiring the importer is not the engine.

- **`beholder-engine-core`** — headless, framework-neutral, **fully gated**. Depends on `beholder-ir` only (never on render-core — the engine does not render).
- **`beholder-render-libgdx`** — the interactive front-end, split so almost everything is gate-tested and only `…/render/libgdx/gl/**` is excluded.

## Movement & collision model

- **Turn** (`TURN_LEFT` / `TURN_RIGHT`): `facing = facing.left()` / `right()`; position unchanged; **never blocked**. Turn cycle `N → E → S → W → N` (right), reverse for left.
- **Move** (`FORWARD` / `BACKWARD` / `STRAFE_LEFT` / `STRAFE_RIGHT`): compute the facing-relative grid delta via the existing `Facing.delta(localX, localY)` — `FORWARD` = `delta(0,+1)`, `BACKWARD` = `delta(0,-1)`, `STRAFE_RIGHT` = `delta(+1,0)`, `STRAFE_LEFT` = `delta(-1,0)`. Target cell = `(x + dCol, y + dRow)`.
- **Collision invariant:** blocked iff (a) the target is off the 32×32 grid, **or** (b) the wall archetype crossed in the travel direction is `SOLID`. Travel-direction → edge `dir`: `(0,-1)→N=0`, `(+1,0)→E=1`, `(0,+1)→S=2`, `(-1,0)→W=3`; then test `geometry.archetype(x, y, dir)`. `OPEN` → move; else stay.
- **Authoritative edge (pinned in TDD).** M3 checks the **current cell's** edge in the travel direction. EOB1 stores walls per block *symmetrically* for the `OPEN`/`SOLID` archetypes M3 handles (a wall between two cells appears on both cells' shared edge), so the current-cell edge equals the target-cell's opposite edge; the synthetic-geometry unit tests (Task A) assert the exact behaviour, and the real-`LEVEL1.MAZ` IT (Task C) confirms it holds on real data. (One-sided/illusory walls that could break the symmetry are M4+ archetypes, out of scope.) Oracle for the move deltas: kyra's move table `{-32,+1,+32,-1}` (= `dRow*32 + dCol`), already encoded by `Facing.delta` and verified at M2.
- **Illegal states unrepresentable:** `Party.position` is always a validated `GridPosition`, `facing` always one of four — no continuous position/orientation is ever representable.

## Interaction envelope (render-libgdx v1)

- **Window:** renders only the 176×120 viewport, uploaded to a `Texture` and drawn aspect-preserving (nearest-neighbour, pillarbox/letterbox via `ViewportFit`), no FoV widening. `beholder-render-core` is **unchanged** except for two pure additions (`Rgba.toRgba8888`, `ViewportFit`).
- **Input:** discrete keyboard → `MovementCommand`. Bindings (pinned): `UP`=FORWARD, `DOWN`=BACKWARD, `LEFT`=TURN_LEFT, `RIGHT`=TURN_RIGHT, `Q`=STRAFE_LEFT, `E`=STRAFE_RIGHT. One key-press = one command = one discrete transition.
- **No mouselook / continuous camera** — structurally impossible (no continuous API exists).

## Differential correctness (§8) — the M3 milestone

ScummVM is **not installed** (only source in `~/Workspaces/CPP/scummvm`), so — matching M2's precedent — the M3 differential stands on three legs, primary first:

1. **Movement/collision logic on synthetic geometry (in-gate, deterministic — the primary correctness proof).** Task A's unit tests build hand-crafted `LevelGeometry` where every `OPEN`/`SOLID` edge is known, and assert every transition: step through `OPEN` advances; step into `SOLID` stays; step off the grid edge stays; the full turn cycle; strafe/backward deltas. This is the load-bearing correctness claim, fully covered by the gate with no game data.
2. **Scripted-path self-consistency on real `LEVEL1.MAZ` (skip-safe IT — refined from the design's frame-hash ratchet).** Task C's `WalkableLevelIT` (`assumeTrue`-skips without game data) loads the real level and asserts *structural* invariants that need no hard-coded maze layout and do not churn when rendering fidelity changes later: four `TURN_RIGHT` restore the original facing; the party never leaves the 32×32 grid; walking `FORWARD` until it stops leaves the party on a cell it legally entered; a `SOLID`-blocked step is a no-op. **A frozen pixel-hash frame ratchet stays deferred** (as M2 ruled: it churns when M4 doors/fidelity land — freezing it now buys regression coverage at the cost of guaranteed churn). The *state* sequence is the durable golden.
3. **Maintainer eyeball (human verdict).** The interactive libGDX window (walk level 1) and a rendered **contact-sheet PNG** of the scripted path (Task C), relayed via `SendUserFile`. The M2-style final leg, given no kyra.
4. **Opportunistic (non-blocking):** if ScummVM is later built/installed, the frame sequence can be pixel-diffed against kyra — same deferred posture as M2's kyra golden.

## Sequencing

**First concrete step: `beholder-engine-core` movement/collision (Task A) — headless, TDD, in-gate — before any libGDX.** It is the load-bearing correctness surface and needs no window; libGDX (Tasks B–C) is plumbing once the engine + render-core exist, its correctness eyeball-verified. End: whole-branch **opus review**, `--no-ff` merge to `main`, **no worktree**.

---

## Module / File Structure

```
pom.xml                                  MODIFY (T-A,T-B) — register the two new modules

beholder-ir/src/main/java/eu/virtualparadox/beholder/ir/
  Facing.java                            MODIFY (T-A) — add left()/right() quarter-turn algebra

beholder-engine-core/                    CREATE (T-A) new module
  pom.xml                                CREATE (T-A) — depends on beholder-ir
  src/main/java/eu/virtualparadox/beholder/engine/
    model/Party.java                     CREATE (T-A) — immutable {position, facing}      [coverage-exempt: model/]
    model/Level.java                     CREATE (T-A) — immutable {geometry, tileset}     [coverage-exempt: model/]
    MovementCommand.java                 CREATE (T-A) — closed enum + applyTo (collision)
    RuntimeState.java                    CREATE (T-A) — mutable party holder + applyCommand

beholder-render-core/src/main/java/eu/virtualparadox/beholder/render/
  Rgba.java                              MODIFY (T-B) — add toRgba8888(argb) channel reorder
  ViewportFit.java                       CREATE (T-B) — aspect-preserving letterbox math
  ViewportRect.java                      CREATE (T-B) — immutable {x, y, width, height}

beholder-render-libgdx/                  CREATE (T-B) new module (app / composition root)
  pom.xml                                CREATE (T-B) — engine-core + render-core + importer + libGDX; jacoco gl/ exclude
  src/main/java/eu/virtualparadox/beholder/render/libgdx/
    InputMapper.java                     CREATE (T-B) — libGDX keycode -> Optional<MovementCommand>   [in-gate]
    InputSource.java                     CREATE (T-B) — interface: poll() -> List<MovementCommand>
    FrameSink.java                       CREATE (T-B) — interface: present(int[] frame)
    GameSession.java                     CREATE (T-B) — poll -> apply -> render -> present            [in-gate]
    gl/LibGdxInputSource.java            CREATE (T-C) — Gdx.input.isKeyJustPressed polling            [EXCLUDED]
    gl/GlFrameSink.java                  CREATE (T-C) — Pixmap/Texture/SpriteBatch blit               [EXCLUDED]
    gl/LibGdxLauncher.java               CREATE (T-C) — Lwjgl3Application main() + wiring             [EXCLUDED]
```

Test files mirror each source under `src/test/java/...`; the differential IT is `beholder-render-libgdx/src/test/java/.../render/libgdx/WalkableLevelIT.java`.

## Execution note (workflow)

Branch `feat/p1-m3-walkable-level`; commit per task; `--no-ff` merge to `main` at the end (controller does the merge, no worktree). Prefix every `mvn` with `JAVA_HOME`:

```bash
export JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home
export REPO=~/Workspaces/Java/legend-of-the-beholder
```

- [ ] **Task 0 · Create the branch**
```bash
git -C "$REPO" checkout -b feat/p1-m3-walkable-level
```
Expected: `Switched to a new branch 'feat/p1-m3-walkable-level'`.

Global constraints (all `CODING_STANDARDS.md` gates): package roots `eu.virtualparadox.beholder.engine` and `…render.libgdx`; Palantir format (`mvn spotless:apply` before `verify`); `final` params, exactly one `return`/method, no `instanceof`, no `switch yield`, cyclomatic ≤8 (PMD counts boolean operators — split guards if needed, as M1 did), ≤12 fields / ≤6 params / ≤18 methods per class, no `return null` (use `Optional`/throw), NullAway ERROR, no `System.out`/`Thread.sleep`, Javadoc `doclint=all` on public/protected, **every magic value a named `ALL_CAPS` constant with a what-and-why comment**, JaCoCo BUNDLE ≥0.90 branch AND line, never commit game data (skip-safe ITs; goldens are hashes/derived state only).

---

## Task A: `beholder-engine-core` — party, movement, collision

**Files:**
- Modify: `beholder-ir/.../ir/Facing.java` (add `left()`/`right()`)
- Modify: `pom.xml:22-26` (register `beholder-engine-core`)
- Create: `beholder-engine-core/pom.xml`
- Create: `beholder-engine-core/.../engine/model/Party.java`, `model/Level.java`, `MovementCommand.java`, `RuntimeState.java`
- Test: `beholder-ir/.../ir/FacingTest.java` (extend), `beholder-engine-core/.../engine/model/PartyTest.java`, `model/LevelTest.java`, `MovementCommandTest.java`, `RuntimeStateTest.java`

**Interfaces:**
- Consumes: `Facing.delta(int,int) -> int[]{dCol,dRow}`, `Facing.values()/ordinal()`; `GridPosition.of(int,int)`, `.x()`, `.y()`, `GridPosition.GRID_SIZE=32`; `WallArchetype.SOLID`; `LevelGeometry.archetype(int x,int y,int dir)`; `TileSet`.
- Produces: `Facing.left()`/`right()`; `Party.of(GridPosition,Facing)`, `.position()`, `.facing()`, `.withPosition(GridPosition)`, `.withFacing(Facing)`; `Level.of(LevelGeometry,TileSet)`, `.geometry()`, `.tileset()`; `MovementCommand.{FORWARD,BACKWARD,STRAFE_LEFT,STRAFE_RIGHT,TURN_LEFT,TURN_RIGHT}` with `applyTo(Party,LevelGeometry) -> Party`; `RuntimeState.startingAt(Party)`, `.party()`, `.applyCommand(MovementCommand,Level)`.

- [ ] **Step A1: Write failing `FacingTest` for the quarter-turn algebra**

Append to `beholder-ir/src/test/java/eu/virtualparadox/beholder/ir/FacingTest.java`:
```java
@Test
void rightCyclesClockwiseNorthEastSouthWest() {
    assertThat(Facing.NORTH.right()).isEqualTo(Facing.EAST);
    assertThat(Facing.EAST.right()).isEqualTo(Facing.SOUTH);
    assertThat(Facing.SOUTH.right()).isEqualTo(Facing.WEST);
    assertThat(Facing.WEST.right()).isEqualTo(Facing.NORTH);
}

@Test
void leftCyclesCounterClockwiseNorthWestSouthEast() {
    assertThat(Facing.NORTH.left()).isEqualTo(Facing.WEST);
    assertThat(Facing.WEST.left()).isEqualTo(Facing.SOUTH);
    assertThat(Facing.SOUTH.left()).isEqualTo(Facing.EAST);
    assertThat(Facing.EAST.left()).isEqualTo(Facing.NORTH);
}
```

- [ ] **Step A2: Run it, verify it fails to compile / fails**

Run: `JAVA_HOME=$JAVA_HOME mvn -q -pl beholder-ir test -Dtest=FacingTest`
Expected: compile error — `cannot find symbol: method left()/right()`.

- [ ] **Step A3: Add `left()`/`right()` to `Facing`**

In `Facing.java`, after the abstract `delta(...)` declaration, add the constants and methods (one `return` each, no `switch`, ordinal arithmetic — matching the declaration order `N,E,S,W` which is clockwise):
```java
/** The four facings; rotations are taken modulo this count over the clockwise N,E,S,W order. */
private static final int FACING_COUNT = 4;

/** Ordinal steps for one clockwise (right) quarter-turn: N->E->S->W->N. */
private static final int RIGHT_TURN_STEPS = 1;

/** Ordinal steps for one counter-clockwise (left) quarter-turn: three clockwise steps = one left. */
private static final int LEFT_TURN_STEPS = 3;

/**
 * Returns the facing one clockwise (right) quarter-turn from this one.
 *
 * @return the facing to the party's right
 */
public Facing right() {
    return values()[(ordinal() + RIGHT_TURN_STEPS) % FACING_COUNT];
}

/**
 * Returns the facing one counter-clockwise (left) quarter-turn from this one.
 *
 * @return the facing to the party's left
 */
public Facing left() {
    return values()[(ordinal() + LEFT_TURN_STEPS) % FACING_COUNT];
}
```

- [ ] **Step A4: Run it, verify pass**

Run: `JAVA_HOME=$JAVA_HOME mvn -q -pl beholder-ir test -Dtest=FacingTest`
Expected: PASS (all facing tests green). Then `mvn -q -pl beholder-ir verify` — coverage still met (`left/right` each hit).

- [ ] **Step A5: Commit**
```bash
git -C "$REPO" add beholder-ir/src/main/java/.../ir/Facing.java beholder-ir/src/test/java/.../ir/FacingTest.java
git -C "$REPO" commit -m "feat(ir): add Facing.left()/right() quarter-turn algebra (M3)"
```

- [ ] **Step A5b: Add `GridPosition` value-equality (`equals`/`hashCode`) — needed by every movement test**

`GridPosition` is a value type but currently has no `equals`/`hashCode` (M2 deferred this, anticipating "later golden tests may want it" — M3's movement tests are exactly that; without it `assertThat(pos).isEqualTo(GridPosition.of(x,y))` compares by reference and fails). Write failing `GridPositionTest` cases first:
```java
@Test
void equalsIsValueBasedAndSymmetric() {
    assertThat(GridPosition.of(10, 12)).isEqualTo(GridPosition.of(10, 12));
    assertThat(GridPosition.of(10, 12)).isNotEqualTo(GridPosition.of(12, 10));
}

@Test
void hashCodeAgreesWithEquals() {
    assertThat(GridPosition.of(10, 12)).hasSameHashCodeAs(GridPosition.of(10, 12));
}

@Test
void notEqualToNullOrOtherType() {
    assertThat(GridPosition.of(10, 12)).isNotEqualTo(null).isNotEqualTo("10,12");
}
```
Then add to `GridPosition.java` (`java.util.Objects` import), each method one `return`, no `instanceof` (use `getClass()` comparison per the house no-`instanceof` rule):
```java
@Override
public boolean equals(final Object other) {
    return other != null && getClass() == other.getClass()
            && x == ((GridPosition) other).x && y == ((GridPosition) other).y;
}

@Override
public int hashCode() {
    return Objects.hash(x, y);
}
```
(The cast follows the `getClass()` guard, so it is safe and `instanceof`-free.) Run `mvn -q -pl beholder-ir verify` → PASS + coverage met (equals' branches: null, other-type, x-differs, y-differs, equal — the three tests above hit all).
```bash
git -C "$REPO" add beholder-ir/src/main/java/.../ir/GridPosition.java beholder-ir/src/test/java/.../ir/GridPositionTest.java
git -C "$REPO" commit -m "feat(ir): add GridPosition value-equality for movement/state comparisons (M3)"
```

- [ ] **Step A6: Register the module + create `beholder-engine-core/pom.xml`**

In root `pom.xml`, add inside `<modules>` (after `beholder-render-core`):
```xml
        <module>beholder-engine-core</module>
```
Create `beholder-engine-core/pom.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>eu.virtualparadox.beholder</groupId>
        <artifactId>legend-of-the-beholder</artifactId>
        <version>0.1.0-SNAPSHOT</version>
    </parent>
    <artifactId>beholder-engine-core</artifactId>
    <name>Beholder Engine Core</name>
    <description>Headless engine runtime: party state and discrete lockstep movement/collision over the IR.</description>
    <dependencies>
        <dependency>
            <groupId>eu.virtualparadox.beholder</groupId>
            <artifactId>beholder-ir</artifactId>
            <version>0.1.0-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```
Run: `JAVA_HOME=$JAVA_HOME mvn -q -pl beholder-engine-core validate` → Expected: BUILD SUCCESS (empty module resolves).

- [ ] **Step A7: Write failing `PartyTest` + `LevelTest` (value types, fail-closed guards)**

`model/PartyTest.java`:
```java
@Test
void ofExposesPositionAndFacing() {
    final Party party = Party.of(GridPosition.of(10, 12), Facing.NORTH);
    assertThat(party.position()).isEqualTo(GridPosition.of(10, 12));
    assertThat(party.facing()).isEqualTo(Facing.NORTH);
}

@Test
void withFacingReplacesOnlyFacing() {
    final Party party = Party.of(GridPosition.of(10, 12), Facing.NORTH).withFacing(Facing.EAST);
    assertThat(party.facing()).isEqualTo(Facing.EAST);
    assertThat(party.position()).isEqualTo(GridPosition.of(10, 12));
}

@Test
void withPositionReplacesOnlyPosition() {
    final Party party = Party.of(GridPosition.of(10, 12), Facing.NORTH).withPosition(GridPosition.of(11, 12));
    assertThat(party.position()).isEqualTo(GridPosition.of(11, 12));
    assertThat(party.facing()).isEqualTo(Facing.NORTH);
}

@Test
void ofRejectsNullPosition() {
    assertThatThrownBy(() -> Party.of(null, Facing.NORTH)).isInstanceOf(IllegalArgumentException.class);
}

@Test
void ofRejectsNullFacing() {
    assertThatThrownBy(() -> Party.of(GridPosition.of(0, 0), null)).isInstanceOf(IllegalArgumentException.class);
}
```
(Value-equality on `GridPosition` was added in Step A5b, so `isEqualTo(GridPosition.of(...))` works.) `LevelTest.java` mirrors: `of` exposes `geometry()`/`tileset()`, rejects null geometry and null tileset. Build a tiny `LevelGeometry.of(1,1,new WallArchetype[]{OPEN,OPEN,OPEN,OPEN}, new int[]{0,0,0,0}, "1.0.0")` and a minimal `TileSet` (see Step A11's `syntheticTileSet` helper — extract it to a shared test helper `EngineTestFixtures`).

- [ ] **Step A8: Run, verify fail; implement `Party` and `Level`; run, verify pass**

Create `model/Party.java` (immutable, house pattern; `model/` is coverage-exempt but guards are behavior-tested):
```java
public final class Party {
    private final GridPosition position;
    private final Facing facing;

    private Party(final GridPosition position, final Facing facing) {
        this.position = position;
        this.facing = facing;
    }

    public static Party of(final GridPosition position, final Facing facing) {
        requireNonNull(position, "position");
        requireNonNull(facing, "facing");
        return new Party(position, facing);
    }

    public GridPosition position() { return position; }
    public Facing facing() { return facing; }
    public Party withPosition(final GridPosition newPosition) { return Party.of(newPosition, facing); }
    public Party withFacing(final Facing newFacing) { return Party.of(position, newFacing); }

    private static void requireNonNull(final Object value, final String name) {
        if (value == null) {
            throw new IllegalArgumentException("Party " + name + " must not be null");
        }
    }
}
```
Create `model/Level.java` analogously (`{LevelGeometry geometry, TileSet tileset}`, `of` null-guards each, accessors `geometry()`/`tileset()`). Full Javadoc on all public members. Run `mvn -q -pl beholder-engine-core test` → PASS.

- [ ] **Step A9: Write failing `MovementCommandTest` — turns**
```java
@Test
void turnRightRotatesFacingWithoutMoving() {
    final Party start = Party.of(GridPosition.of(5, 5), Facing.NORTH);
    final Party turned = MovementCommand.TURN_RIGHT.applyTo(start, openGeometry());
    assertThat(turned.facing()).isEqualTo(Facing.EAST);
    assertThat(turned.position()).isEqualTo(start.position());
}

@Test
void turnLeftRotatesFacingWithoutMoving() {
    final Party turned = MovementCommand.TURN_LEFT.applyTo(Party.of(GridPosition.of(5, 5), Facing.NORTH), openGeometry());
    assertThat(turned.facing()).isEqualTo(Facing.WEST);
}
```
where `openGeometry()` builds a 3×3 all-`OPEN` `LevelGeometry` (12×4 = wait: `LevelGeometry.of(3,3, allOpen(3*3*4), new int[3*3*4], "1.0.0")`, `allOpen(n)` = array filled `WallArchetype.OPEN`).

- [ ] **Step A10: Write failing `MovementCommandTest` — moves + collision**
```java
@Test
void forwardThroughOpenEdgeAdvancesNorth() {
    // NORTH forward from (1,1) crosses the north edge to (1,0); north edge OPEN -> moves.
    final Party moved = MovementCommand.FORWARD.applyTo(Party.of(GridPosition.of(1, 1), Facing.NORTH), openGeometry());
    assertThat(moved.position()).isEqualTo(GridPosition.of(1, 0));
    assertThat(moved.facing()).isEqualTo(Facing.NORTH);
}

@Test
void forwardIntoSolidEdgeStaysPut() {
    // Geometry where cell (1,1) north edge is SOLID -> FORWARD facing NORTH is blocked.
    final LevelGeometry geo = geometryWithSolidNorthEdgeAt(1, 1);
    final Party start = Party.of(GridPosition.of(1, 1), Facing.NORTH);
    assertThat(MovementCommand.FORWARD.applyTo(start, geo).position()).isEqualTo(start.position());
}

@Test
void forwardOffTheGridEdgeStaysPut() {
    // NORTH forward from row 0 targets row -1 (off-grid) -> blocked even on all-open geometry.
    final Party start = Party.of(GridPosition.of(1, 0), Facing.NORTH);
    assertThat(MovementCommand.FORWARD.applyTo(start, openGeometry()).position()).isEqualTo(start.position());
}

@Test
void backwardStrafeLeftStrafeRightUseLocalAxisDeltas() {
    final LevelGeometry geo = openGeometry();
    final Party at = Party.of(GridPosition.of(1, 1), Facing.NORTH);
    assertThat(MovementCommand.BACKWARD.applyTo(at, geo).position()).isEqualTo(GridPosition.of(1, 2));     // south
    assertThat(MovementCommand.STRAFE_RIGHT.applyTo(at, geo).position()).isEqualTo(GridPosition.of(2, 1)); // east
    assertThat(MovementCommand.STRAFE_LEFT.applyTo(at, geo).position()).isEqualTo(GridPosition.of(0, 1));  // west
}
```
`geometryWithSolidNorthEdgeAt(x,y)` builds all-`OPEN` then sets `(x,y)` dir 0 to `SOLID` with wallSet 1 (remember `LevelGeometry.of` requires OPEN edges carry wallSet 0, SOLID may carry ≥0). Note the 3×3 grid means use a bigger grid (e.g. 5×5, party at (2,2)) so all four moves land in-range; adjust the openGeometry helper to 5×5 and recentre the party at (2,2) in the move tests to keep every direction on-grid.

- [ ] **Step A11: Implement `MovementCommand`; run, verify pass**
```java
public enum MovementCommand {
    FORWARD {
        @Override public Party applyTo(final Party party, final LevelGeometry geometry) {
            return stepped(party, geometry, LOCAL_ORIGIN, LOCAL_FORWARD);
        }
    },
    BACKWARD {
        @Override public Party applyTo(final Party party, final LevelGeometry geometry) {
            return stepped(party, geometry, LOCAL_ORIGIN, LOCAL_BACKWARD);
        }
    },
    STRAFE_LEFT {
        @Override public Party applyTo(final Party party, final LevelGeometry geometry) {
            return stepped(party, geometry, LOCAL_LEFT, LOCAL_ORIGIN);
        }
    },
    STRAFE_RIGHT {
        @Override public Party applyTo(final Party party, final LevelGeometry geometry) {
            return stepped(party, geometry, LOCAL_RIGHT, LOCAL_ORIGIN);
        }
    },
    TURN_LEFT {
        @Override public Party applyTo(final Party party, final LevelGeometry geometry) {
            return party.withFacing(party.facing().left());
        }
    },
    TURN_RIGHT {
        @Override public Party applyTo(final Party party, final LevelGeometry geometry) {
            return party.withFacing(party.facing().right());
        }
    };

    /** Local-axis magnitudes: forward is +1 on the local Y axis, right is +1 on the local X axis. */
    private static final int LOCAL_ORIGIN = 0;
    private static final int LOCAL_FORWARD = 1;
    private static final int LOCAL_BACKWARD = -1;
    private static final int LOCAL_RIGHT = 1;
    private static final int LOCAL_LEFT = -1;

    /** Added to a unit grid delta (-1..+1) to index EDGE_DIR_BY_DELTA (0..2). */
    private static final int DELTA_ORIGIN = 1;

    /**
     * Maps a unit grid delta {dCol,dRow} to the crossed edge's direction index, addressed as
     * EDGE_DIR_BY_DELTA[dCol+1][dRow+1]. Only the four axis-unit deltas are ever produced by a move
     * command via Facing.delta, so the diagonal/centre cells are unreachable sentinels (-1):
     *   (0,-1)->N=0, (+1,0)->E=1, (0,+1)->S=2, (-1,0)->W=3  (matches LevelGeometry's dir order).
     */
    private static final int UNREACHABLE = -1;
    private static final int[][] EDGE_DIR_BY_DELTA = {
        {UNREACHABLE, 3 /*W*/, UNREACHABLE},
        {0 /*N*/, UNREACHABLE, 2 /*S*/},
        {UNREACHABLE, 1 /*E*/, UNREACHABLE},
    };

    public abstract Party applyTo(Party party, LevelGeometry geometry);

    /** Attempts a one-cell move by a party-local (localX,localY); returns the party unchanged if blocked. */
    private static Party stepped(final Party party, final LevelGeometry geometry, final int localX, final int localY) {
        final int[] delta = party.facing().delta(localX, localY);
        final int targetX = party.position().x() + delta[0];
        final int targetY = party.position().y() + delta[1];
        final boolean blocked = offGrid(targetX, targetY)
                || geometry.archetype(party.position().x(), party.position().y(), edgeDir(delta)) == WallArchetype.SOLID;
        return blocked ? party : party.withPosition(GridPosition.of(targetX, targetY));
    }

    /** True if (x,y) falls outside the fixed 32x32 grid. */
    private static boolean offGrid(final int x, final int y) {
        return x < 0 || x >= GridPosition.GRID_SIZE || y < 0 || y >= GridPosition.GRID_SIZE;
    }

    /** The crossed-edge direction for a unit grid delta (only evaluated on-grid, i.e. for a valid unit delta). */
    private static int edgeDir(final int[] delta) {
        return EDGE_DIR_BY_DELTA[delta[0] + DELTA_ORIGIN][delta[1] + DELTA_ORIGIN];
    }
}
```
Note the short-circuit: `edgeDir(delta)` is only evaluated when `offGrid(...)` is false, so it never indexes a sentinel. `EDGE_DIR_BY_DELTA` is `private static final` (SpotBugs MS_MUTABLE_ARRAY only flags exposed arrays). Run `mvn -q -pl beholder-engine-core test` → PASS. Verify `stepped`/`offGrid`/`edgeDir` each ≤ cyclomatic 8 (offGrid = 4 conditions; stepped = 1 `||` + ternary).

- [ ] **Step A12: Write failing `RuntimeStateTest`; implement `RuntimeState`; pass**
```java
@Test
void applyCommandAdvancesTheHeldParty() {
    final RuntimeState state = RuntimeState.startingAt(Party.of(GridPosition.of(2, 2), Facing.NORTH));
    state.applyCommand(MovementCommand.FORWARD, openLevel());   // 5x5 open Level
    assertThat(state.party().position()).isEqualTo(GridPosition.of(2, 1));
}

@Test
void applyCommandTurnsInPlace() {
    final RuntimeState state = RuntimeState.startingAt(Party.of(GridPosition.of(2, 2), Facing.NORTH));
    state.applyCommand(MovementCommand.TURN_RIGHT, openLevel());
    assertThat(state.party().facing()).isEqualTo(Facing.EAST);
    assertThat(state.party().position()).isEqualTo(GridPosition.of(2, 2));
}

@Test
void startingAtRejectsNullParty() {
    assertThatThrownBy(() -> RuntimeState.startingAt(null)).isInstanceOf(IllegalArgumentException.class);
}
```
`RuntimeState.java`:
```java
public final class RuntimeState {
    private Party party;

    private RuntimeState(final Party party) { this.party = party; }

    public static RuntimeState startingAt(final Party party) {
        if (party == null) {
            throw new IllegalArgumentException("RuntimeState party must not be null");
        }
        return new RuntimeState(party);
    }

    public Party party() { return party; }

    public void applyCommand(final MovementCommand command, final Level level) {
        this.party = command.applyTo(party, level.geometry());
    }
}
```
Run `mvn -q -pl beholder-engine-core verify` → PASS + "coverage checks met" (MovementCommand/RuntimeState 100%; model/ exempt).

- [ ] **Step A13: Full reactor verify + commit**

Run: `JAVA_HOME=$JAVA_HOME mvn -q -Dskip=false spotless:apply && JAVA_HOME=$JAVA_HOME mvn -q verify`
Expected: BUILD SUCCESS across all 4 modules; engine-core coverage met.
```bash
git -C "$REPO" add pom.xml beholder-engine-core
git -C "$REPO" commit -m "feat(engine-core): party state + discrete lockstep movement/collision (M3 Task A)"
```

---

## Task B: render-core additions + `beholder-render-libgdx` in-gate logic + libGDX bring-up

**Files:**
- Modify: `beholder-render-core/.../render/Rgba.java` (+ `RgbaTest`)
- Create: `beholder-render-core/.../render/ViewportFit.java`, `ViewportRect.java` (+ `ViewportFitTest`)
- Modify: `pom.xml` (register `beholder-render-libgdx`)
- Create: `beholder-render-libgdx/pom.xml`, `.../render/libgdx/InputMapper.java`, `InputSource.java`, `FrameSink.java`, `GameSession.java`
- Test: `.../render/libgdx/InputMapperTest.java`, `GameSessionTest.java`, `EngineTestFixtures` (synthetic Level/TileSet helper)

**Interfaces:**
- Consumes: `FirstPersonRenderer.render(LevelGeometry,TileSet,GridPosition,Facing) -> int[176*120]`; `Rgba.pack(...)`; engine-core `Level`, `RuntimeState`, `MovementCommand`, `Party`; libGDX `com.badlogic.gdx.Input.Keys.*`.
- Produces: `Rgba.toRgba8888(int) -> int`; `ViewportFit.fit(int,int,int,int) -> ViewportRect` with `.x()/.y()/.width()/.height()`; `InputMapper.map(int) -> Optional<MovementCommand>`, `.boundKeys() -> Set<Integer>`; `InputSource.poll() -> List<MovementCommand>`; `FrameSink.present(int[])`; `GameSession(Level,RuntimeState,InputSource,FrameSink)`, `.tick()`.

- [ ] **Step B1: libGDX dependency spike (de-risk before writing front-end code)**

Register the module in root `pom.xml` `<modules>`:
```xml
        <module>beholder-render-libgdx</module>
```
Create `beholder-render-libgdx/pom.xml` (libGDX 1.12.1; note the `natives-desktop` classifier; test-time importer dep is *compile* here because the launcher composition root loads levels):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>eu.virtualparadox.beholder</groupId>
        <artifactId>legend-of-the-beholder</artifactId>
        <version>0.1.0-SNAPSHOT</version>
    </parent>
    <artifactId>beholder-render-libgdx</artifactId>
    <name>Beholder Render libGDX</name>
    <description>Interactive libGDX front-end and composition root: discrete input -> engine -> renderer -> window.</description>
    <properties>
        <!-- The reactor's first UI dependency. darkmoor (1.5.3/LWJGL2/Java6) is not a reference; this is LWJGL3/JDK21. -->
        <gdx.version>1.12.1</gdx.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>eu.virtualparadox.beholder</groupId>
            <artifactId>beholder-engine-core</artifactId>
            <version>0.1.0-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>eu.virtualparadox.beholder</groupId>
            <artifactId>beholder-render-core</artifactId>
            <version>0.1.0-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>eu.virtualparadox.beholder</groupId>
            <artifactId>beholder-importer-westwood</artifactId>
            <version>0.1.0-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>com.badlogicgames.gdx</groupId>
            <artifactId>gdx</artifactId>
            <version>${gdx.version}</version>
        </dependency>
        <dependency>
            <groupId>com.badlogicgames.gdx</groupId>
            <artifactId>gdx-backend-lwjgl3</artifactId>
            <version>${gdx.version}</version>
        </dependency>
        <dependency>
            <groupId>com.badlogicgames.gdx</groupId>
            <artifactId>gdx-platform</artifactId>
            <version>${gdx.version}</version>
            <classifier>natives-desktop</classifier>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <!--
                Narrow, documented JaCoCo exclusion for the irreducible GL boundary (render/libgdx/gl/**):
                LibGdxInputSource, GlFrameSink, LibGdxLauncher have no branch logic and cannot run without a
                display/GL context, so they are untestable in the same category as the parent's **/model/**
                exemption. All decision logic (InputMapper, GameSession) stays in-gate. combine.children=append
                preserves the parent's model/dto excludes and adds this one.
            -->
            <plugin>
                <groupId>org.jacoco</groupId>
                <artifactId>jacoco-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <id>report</id>
                        <configuration>
                            <excludes combine.children="append">
                                <exclude>**/render/libgdx/gl/**</exclude>
                            </excludes>
                        </configuration>
                    </execution>
                    <execution>
                        <id>check</id>
                        <configuration>
                            <excludes combine.children="append">
                                <exclude>**/render/libgdx/gl/**</exclude>
                            </excludes>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```
Run: `JAVA_HOME=$JAVA_HOME mvn -q -pl beholder-render-libgdx dependency:resolve`
Expected: libGDX 1.12.1 artifacts resolve (download from Maven Central is a normal dependency fetch — not a code/data leak; permitted). If **offline**, the maintainer pre-fetches once. Then create a throwaway `src/main/java/.../render/libgdx/package-info.java` (Javadoc-only) and run `mvn -q -pl beholder-render-libgdx compile` → Expected: BUILD SUCCESS (libGDX on the classpath, JDK 21). **If a libGDX API used later differs in 1.12.1, adjust the version here — this spike is where a version surprise surfaces.**

- [ ] **Step B2: Write failing `RgbaTest.toRgba8888`; implement; pass**
```java
@Test
void toRgba8888MovesArgbChannelsToRgbaOrder() {
    // 0xAARRGGBB -> 0xRRGGBBAA: alpha moves from the top byte to the bottom byte.
    assertThat(Rgba.toRgba8888(0xFF112233)).isEqualTo(0x112233FF);
    assertThat(Rgba.toRgba8888(0x00000000)).isEqualTo(0x00000000);
    assertThat(Rgba.toRgba8888(0x80ABCDEF)).isEqualTo(0xABCDEF80);
}
```
Add to `Rgba.java`:
```java
/**
 * Converts a packed {@code 0xAARRGGBB} (Java/AWT order, as {@link FirstPersonRenderer} produces) to
 * libGDX {@code Pixmap}'s native {@code 0xRRGGBBAA} (RGBA8888) order: rotate the RGB triple up one
 * byte and drop the alpha to the low byte. Pure integer op so the one bit of real logic in the GL
 * blit is testable in-gate; the excluded GlFrameSink only loops over pixels calling this.
 *
 * @param argb a packed {@code 0xAARRGGBB} pixel
 * @return the same colour packed as {@code 0xRRGGBBAA} (RGBA8888)
 */
public static int toRgba8888(final int argb) {
    return (argb << 8) | (argb >>> 24);
}
```
Run `mvn -q -pl beholder-render-core test -Dtest=RgbaTest` → PASS.

- [ ] **Step B3: Write failing `ViewportFitTest`; implement `ViewportFit` + `ViewportRect`; pass**
```java
@Test
void exactIntegerMultipleFillsFully() {
    final ViewportRect r = ViewportFit.fit(176, 120, 704, 480);
    assertThat(r.x()).isZero();  assertThat(r.y()).isZero();
    assertThat(r.width()).isEqualTo(704);  assertThat(r.height()).isEqualTo(480);
}

@Test
void widerWindowPillarboxesLeftAndRight() {
    final ViewportRect r = ViewportFit.fit(176, 120, 800, 480);  // height-limited (min scale = 4.0)
    assertThat(r.width()).isEqualTo(704);  assertThat(r.height()).isEqualTo(480);
    assertThat(r.x()).isEqualTo(48);  assertThat(r.y()).isZero();
}

@Test
void tallerWindowLetterboxesTopAndBottom() {
    final ViewportRect r = ViewportFit.fit(176, 120, 704, 600);  // width-limited (min scale = 4.0)
    assertThat(r.width()).isEqualTo(704);  assertThat(r.height()).isEqualTo(480);
    assertThat(r.x()).isZero();  assertThat(r.y()).isEqualTo(60);
}

@Test
void rejectsNonPositiveSourceDimensions() {
    assertThatThrownBy(() -> ViewportFit.fit(0, 120, 704, 480)).isInstanceOf(IllegalArgumentException.class);
    assertThatThrownBy(() -> ViewportFit.fit(176, 0, 704, 480)).isInstanceOf(IllegalArgumentException.class);
}
```
`ViewportRect.java` — final class, `{x,y,width,height}` ints, public ctor + 4 accessors, full Javadoc (no `equals`/`hashCode`, matching the house pattern that avoids record-generated coverage gaps). `ViewportFit.java`:
```java
public final class ViewportFit {
    private ViewportFit() {}

    /**
     * Computes the largest aspect-preserving rectangle of {@code source} centred inside {@code target}
     * (pillarbox/letterbox); never widens the source aspect. The limiting axis (smaller scale factor)
     * sets the scale; the other axis is centred.
     */
    public static ViewportRect fit(
            final int sourceWidth, final int sourceHeight, final int targetWidth, final int targetHeight) {
        requirePositive(sourceWidth, "sourceWidth");
        requirePositive(sourceHeight, "sourceHeight");
        final float scale =
                Math.min((float) targetWidth / sourceWidth, (float) targetHeight / sourceHeight);
        final int fittedWidth = Math.round(sourceWidth * scale);
        final int fittedHeight = Math.round(sourceHeight * scale);
        return new ViewportRect((targetWidth - fittedWidth) / 2, (targetHeight - fittedHeight) / 2, fittedWidth, fittedHeight);
    }

    private static void requirePositive(final int value, final String name) {
        if (value <= 0) {
            throw new IllegalArgumentException("ViewportFit " + name + " must be positive: " + value);
        }
    }
}
```
Run `mvn -q -pl beholder-render-core verify` → PASS + coverage met.

- [ ] **Step B4: Write failing `InputMapperTest`; implement `InputMapper`; pass**
```java
@Test
void mapsBoundKeysToCommands() {
    final InputMapper mapper = new InputMapper();
    assertThat(mapper.map(Input.Keys.UP)).contains(MovementCommand.FORWARD);
    assertThat(mapper.map(Input.Keys.DOWN)).contains(MovementCommand.BACKWARD);
    assertThat(mapper.map(Input.Keys.LEFT)).contains(MovementCommand.TURN_LEFT);
    assertThat(mapper.map(Input.Keys.RIGHT)).contains(MovementCommand.TURN_RIGHT);
    assertThat(mapper.map(Input.Keys.Q)).contains(MovementCommand.STRAFE_LEFT);
    assertThat(mapper.map(Input.Keys.E)).contains(MovementCommand.STRAFE_RIGHT);
}

@Test
void unboundKeyMapsToEmpty() {
    assertThat(new InputMapper().map(Input.Keys.SPACE)).isEmpty();
}

@Test
void boundKeysAreTheSixMovementKeys() {
    assertThat(new InputMapper().boundKeys()).hasSize(6)
        .contains(Input.Keys.UP, Input.Keys.DOWN, Input.Keys.LEFT, Input.Keys.RIGHT, Input.Keys.Q, Input.Keys.E);
}
```
`InputMapper.java` (uses `com.badlogic.gdx.Input.Keys` compile-time int constants — no GL context needed to unit-test):
```java
public final class InputMapper {
    /** The fixed EOB-style discrete key bindings; Input.Keys values are compile-time ints (no GL needed). */
    private static final Map<Integer, MovementCommand> BINDINGS = Map.of(
            Input.Keys.UP, MovementCommand.FORWARD,
            Input.Keys.DOWN, MovementCommand.BACKWARD,
            Input.Keys.LEFT, MovementCommand.TURN_LEFT,
            Input.Keys.RIGHT, MovementCommand.TURN_RIGHT,
            Input.Keys.Q, MovementCommand.STRAFE_LEFT,
            Input.Keys.E, MovementCommand.STRAFE_RIGHT);

    public Optional<MovementCommand> map(final int keycode) {
        return Optional.ofNullable(BINDINGS.get(keycode));
    }

    public Set<Integer> boundKeys() {
        return BINDINGS.keySet();
    }
}
```
Run `mvn -q -pl beholder-render-libgdx test -Dtest=InputMapperTest` → PASS.

- [ ] **Step B5: Create the `InputSource` / `FrameSink` interfaces + `EngineTestFixtures`**

`InputSource.java`: `List<MovementCommand> poll();` (Javadoc: "commands issued since the last poll; empty if none"). `FrameSink.java`: `void present(int[] frame);` (Javadoc: "receive one 176*120 ARGB frame to display"). Create test helper `EngineTestFixtures` with `syntheticLevel()` → a `Level` over a small all-`OPEN` `LevelGeometry` (e.g. 32×32 so the render footprint is in-range) + a `syntheticTileSet()`: `TileSet.of(new byte[][]{new byte[64]}, new int[][]{new int[16], new int[16]}, new int[][]{new int[330], new int[431]}, new int[256])` (one blank tile, blank sub-palettes, one blank wall-set, blank palette — renders a plain background frame, no game data). Reuse this helper for `GameSessionTest`.

- [ ] **Step B6: Write failing `GameSessionTest`; implement `GameSession`; pass**
```java
@Test
void tickAppliesPolledCommandsThenRendersCurrentState() {
    final RuntimeState state = RuntimeState.startingAt(Party.of(GridPosition.of(16, 16), Facing.NORTH));
    final FakeInputSource input = new FakeInputSource(List.of(MovementCommand.FORWARD));
    final RecordingFrameSink sink = new RecordingFrameSink();
    final GameSession session = new GameSession(EngineTestFixtures.syntheticLevel(), state, input, sink);

    session.tick();

    assertThat(state.party().position()).isEqualTo(GridPosition.of(16, 15));  // advanced north
    assertThat(sink.lastFrame()).hasSize(FirstPersonRenderer.VIEWPORT_WIDTH * FirstPersonRenderer.VIEWPORT_HEIGHT);
}

@Test
void tickWithNoInputStillRendersTheStaticFrame() {
    final RecordingFrameSink sink = new RecordingFrameSink();
    new GameSession(EngineTestFixtures.syntheticLevel(),
            RuntimeState.startingAt(Party.of(GridPosition.of(16, 16), Facing.NORTH)),
            new FakeInputSource(List.of()), sink).tick();
    assertThat(sink.lastFrame()).isNotNull();
}
```
`FakeInputSource` returns its list once then empty; `RecordingFrameSink` stores the last frame. `GameSession.java`:
```java
public final class GameSession {
    private final Level level;
    private final RuntimeState runtimeState;
    private final InputSource inputSource;
    private final FrameSink frameSink;

    public GameSession(final Level level, final RuntimeState runtimeState,
            final InputSource inputSource, final FrameSink frameSink) {
        this.level = level;
        this.runtimeState = runtimeState;
        this.inputSource = inputSource;
        this.frameSink = frameSink;
    }

    /** One discrete frame: apply all polled commands, then render and present the (possibly unchanged) state. */
    public void tick() {
        for (final MovementCommand command : inputSource.poll()) {
            runtimeState.applyCommand(command, level);
        }
        final Party party = runtimeState.party();
        frameSink.present(FirstPersonRenderer.render(
                level.geometry(), level.tileset(), party.position(), party.facing()));
    }
}
```
Run `mvn -q -pl beholder-render-libgdx verify` → PASS + **coverage met** (InputMapper + GameSession in-gate; `gl/**` empty so far). Confirm the output says coverage met even though libGDX is on the classpath.

- [ ] **Step B7: Full reactor verify + commit**
```bash
JAVA_HOME=$JAVA_HOME mvn -q spotless:apply && JAVA_HOME=$JAVA_HOME mvn -q verify
git -C "$REPO" add pom.xml beholder-render-core beholder-render-libgdx
git -C "$REPO" commit -m "feat(render-libgdx): in-gate front-end logic + render-core Rgba/ViewportFit (M3 Task B)"
```
Expected: BUILD SUCCESS all 5 modules.

---

## Task C: GL leaf + differential IT + eyeball

**Files:**
- Create: `beholder-render-libgdx/.../render/libgdx/gl/LibGdxInputSource.java`, `gl/GlFrameSink.java`, `gl/LibGdxLauncher.java` (all JaCoCo-excluded)
- Create: `beholder-render-libgdx/src/test/java/.../render/libgdx/WalkableLevelIT.java` (skip-safe differential IT + contact-sheet PNG)

**Interfaces:**
- Consumes: `InputMapper`, `InputSource`, `FrameSink`, `GameSession`; `Rgba.toRgba8888`, `ViewportFit.fit`; libGDX `Lwjgl3Application(...)`, `ApplicationAdapter`, `Pixmap`, `Texture`, `SpriteBatch`, `Gdx.input/graphics`; importer `PakArchive.parse`, `Maze.parse`, `LevelInfoHeader.parse`, `VcnAtlas/VmpProjection/VgaPalette.parse`, `WestwoodToIr.geometry/tileSet`.

- [ ] **Step C1: Implement the excluded GL leaf (no gate tests — display-only)**

`gl/LibGdxInputSource.java implements InputSource`: constructed with an `InputMapper`; `poll()` iterates `mapper.boundKeys()`, and for each `Gdx.input.isKeyJustPressed(key)` collects `mapper.map(key)` into a list. (Just-pressed = one command per physical press = discrete lockstep.) `gl/GlFrameSink.java implements FrameSink`: holds a `Pixmap(176,120, Pixmap.Format.RGBA8888)`, a `Texture`, a `SpriteBatch`; `present(int[] frame)` writes each pixel via `pixmap.drawPixel(x, y, Rgba.toRgba8888(frame[y*176+x]))`, `texture.draw(pixmap, 0, 0)`, computes `ViewportFit.fit(176, 120, Gdx.graphics.getWidth(), Gdx.graphics.getHeight())`, then `batch.begin(); batch.draw(texture, rect.x(), rect.y(), rect.width(), rect.height()); batch.end();`; plus a `dispose()`. `gl/LibGdxLauncher.java`: `main(String[])` builds `Lwjgl3ApplicationConfiguration` (title `"Legend of the Beholder — EOB1 L1"`, size 704×480, `setForegroundFPS(60)`), and a `new Lwjgl3Application(adapter, config)` where the `ApplicationAdapter`'s `create()` loads real level 1 (importer chain from the `~/Games/...` PAK; if unreadable, logs via `Gdx.app.error` and `Gdx.app.exit()`), starts the party at `GridPosition.of(10, 12)` facing `NORTH` (the M2 reference cell, a known open corridor), and builds `GameSession(level, state, new LibGdxInputSource(mapper), glFrameSink)`; `render()` clears the screen and calls `session.tick()`; `dispose()` disposes the sink. Full Javadoc; every magic value a named constant. **macOS note (in the class Javadoc + Task run):** on macOS the LWJGL3 window needs the JVM on the first thread — libGDX 1.12's `Lwjgl3Application` auto-relaunches with `-XstartOnFirstThread` via its start-on-first-thread helper; if a manual run deadlocks, add `-XstartOnFirstThread` to the JVM args.

- [ ] **Step C2: Verify the JaCoCo exclusion actually holds**

Run: `JAVA_HOME=$JAVA_HOME mvn -q -pl beholder-render-libgdx verify`
Expected: BUILD SUCCESS with "coverage checks met" **despite** the three new untested `gl/` classes. Then open `beholder-render-libgdx/target/site/jacoco/index.html` (or the `jacoco-report`), and confirm the `…render.libgdx.gl` package/classes are **absent** from the coverage table (excluded), while `InputMapper` + `GameSession` are present at 100%. If the build instead fails on coverage, the `combine.children="append"` merge didn't take — fallback: replace the child excludes with the full explicit list (copy the parent's nine `<exclude>` entries plus `**/render/libgdx/gl/**`, without `combine.children`).

- [ ] **Step C3: Write the differential IT — scripted-path self-consistency on real `LEVEL1.MAZ`**

`WalkableLevelIT.java` (skip-safe, mirrors the importer ITs' `GAME`/`assumeTrue` pattern; loads real level 1 via the importer chain; asserts churn-free structural invariants — no hard-coded maze layout):
```java
private static final Path GAME =
        Path.of(System.getProperty("user.home"), "Games/Eye of the Beholder.app/Contents/Resources/game");

private Level loadLevel1() throws IOException {
    final Path archive = GAME.resolve("EOBDATA3.PAK");
    assumeTrue(Files.isReadable(archive), "EOB1 game data present at " + archive);
    final PakArchive pak = PakArchive.parse(Files.readAllBytes(archive));
    final LevelGeometry geo = WestwoodToIr.geometry(
            Maze.parse(pak.file("LEVEL1.MAZ")), LevelInfoHeader.parse(pak.file("LEVEL1.INF")));
    final TileSet tiles = WestwoodToIr.tileSet(
            VcnAtlas.parse(pak.file("BRICK.VCN")), VmpProjection.parse(pak.file("BRICK.VMP")),
            VgaPalette.parse(pak.file("BRICK.PAL")));
    return Level.of(geo, tiles);
}

@Test
void fourRightTurnsRestoreTheOriginalFacing() throws IOException {
    final Level level = loadLevel1();
    final RuntimeState state = RuntimeState.startingAt(Party.of(GridPosition.of(10, 12), Facing.NORTH));
    for (int i = 0; i < 4; i++) { state.applyCommand(MovementCommand.TURN_RIGHT, level); }
    assertThat(state.party().facing()).isEqualTo(Facing.NORTH);
    assertThat(state.party().position()).isEqualTo(GridPosition.of(10, 12));
}

@Test
void walkingForwardNeverLeavesTheGridAndStopsAtAWall() throws IOException {
    final Level level = loadLevel1();
    final RuntimeState state = RuntimeState.startingAt(Party.of(GridPosition.of(10, 12), Facing.NORTH));
    GridPosition previous = state.party().position();
    for (int step = 0; step < 40; step++) {
        state.applyCommand(MovementCommand.FORWARD, level);
        final GridPosition now = state.party().position();
        assertThat(now.x()).isBetween(0, GridPosition.GRID_SIZE - 1);
        assertThat(now.y()).isBetween(0, GridPosition.GRID_SIZE - 1);
        previous = now;
    }
    // After 40 forward steps in a bounded 32-cell level the party has necessarily hit a wall and
    // stopped (a further FORWARD is a no-op).
    final GridPosition settled = state.party().position();
    state.applyCommand(MovementCommand.FORWARD, level);
    assertThat(state.party().position()).isEqualTo(settled);
}
```
Run: `JAVA_HOME=$JAVA_HOME mvn -q -pl beholder-render-libgdx verify` (with game data present → runs; absent → skipped). Expected: PASS or skipped, never failed.

- [ ] **Step C4: Add the contact-sheet PNG generator (eyeball, scratch-only)**

Add a `@Test` `rendersAScriptedPathContactSheet()` to `WalkableLevelIT` (skip-safe): drive a fixed sequence — e.g. `FORWARD×3, TURN_RIGHT, FORWARD×2, TURN_LEFT, FORWARD×2` — rendering each state via `FirstPersonRenderer.render` to an `int[176*120]`, and tile the frames into one `BufferedImage` (columns of the path), writing it to the scratch dir (`/private/tmp/claude-.../scratchpad/m3-walk-contact-sheet.png`) via `ImageIO.write`. **Never** write into the repo. Convert ARGB `int[]` → `BufferedImage` with `image.setRGB(...)` (AWT is already 0xAARRGGBB — no reorder needed here, unlike the GL path). This test asserts only that the file was written (`assertThat(Files.exists(out)).isTrue()`); the visual verdict is the maintainer's.

- [ ] **Step C5: Full reactor verify + commit**
```bash
JAVA_HOME=$JAVA_HOME mvn -q spotless:apply && JAVA_HOME=$JAVA_HOME mvn -q verify
git -C "$REPO" add beholder-render-libgdx
git -C "$REPO" commit -m "feat(render-libgdx): GL front-end leaf + walkable-level differential IT + eyeball (M3 Task C)"
```

- [ ] **Step C6: Manual eyeball — run the window + relay the contact sheet**

Run the interactive window (game data required): `JAVA_HOME=$JAVA_HOME mvn -q -pl beholder-render-libgdx exec:java -Dexec.mainClass=eu.virtualparadox.beholder.render.libgdx.gl.LibGdxLauncher` (add the `exec-maven-plugin` invocation; on macOS pass `-XstartOnFirstThread` if it deadlocks). Walk with `↑↓←→ Q E`. Relay the contact-sheet PNG to the maintainer via `SendUserFile` for the render+movement verdict (the M2-style human differential leg). **Hold the merge for the maintainer's verdict**, exactly as M2 did.

---

## Self-Review (plan vs. design)

- **Spec coverage.** engine-core (Party/Level/MovementCommand/RuntimeState) → Task A; movement/collision model + `Facing.left/right` → Task A; render-libgdx in-gate logic (InputMapper/GameSession) + render-core additions (Rgba/ViewportFit) → Task B; GL leaf + interaction envelope + differential + eyeball → Task C; coverage-gate exclusion → Task B pom + Task C verify step. Every design section maps to a task.
- **Placeholder scan.** No `TBD`/"add error handling"/"similar to". Every code step shows real code; every command has an expected result. The one deliberate deferral (frozen frame-hash ratchet) is stated with its reason, not a gap.
- **Type consistency.** `MovementCommand.applyTo(Party,LevelGeometry)` consumed identically in RuntimeState (Task A) and referenced in GameSession via `RuntimeState.applyCommand(MovementCommand,Level)` (Task B). `Level.geometry()/tileset()`, `Party.position()/facing()`, `ViewportFit.fit(...) -> ViewportRect.x()/y()/width()/height()`, `Rgba.toRgba8888(int)`, `InputMapper.map(int)->Optional`/`boundKeys()->Set` — names/signatures match across every consuming task.
- **Scope.** One walkable level (MASTER §10 vertical slice); triggers/doors/characters/auto-map/round-trip deferred to owning milestones. YAGNI: `Facing.opposite()` dropped (unused in M3).
- **Ambiguity.** The collision authoritative-edge is pinned (current-cell edge; symmetry holds for M3's OPEN/SOLID; validated by Task A synthetic tests + Task C real-data IT). The libGDX version risk is retired by the Task B spike before front-end code.

---

## Execution outcome (2026-07-07)

Executed subagent-driven (fresh implementer + independent task-review per task, an opus whole-branch review, then a final-review fast-follow batch) on branch `feat/p1-m3-walkable-level`, merged `--no-ff` to `main` (`bf5f809`) and pushed. **EOB1 level 1 is walkable through the real IR**: discrete lockstep movement + collision in `beholder-engine-core`, driven by an interactive `beholder-render-libgdx` front-end wrapping the M2 headless renderer. Merge-verify: all 6 reactor modules `BUILD SUCCESS`, coverage met on every module.

- **Tasks A–C landed**, six commits: `Facing.left/right` (`1bef288`), `GridPosition` value-equality (`b061710`), `beholder-engine-core` (`ce8b5fb`); render-libgdx in-gate logic + render-core `Rgba.toRgba8888`/`ViewportFit` (`0fe4ded`) + a review fix naming the `toRgba8888` shift constants (`0d5ebff`); the excluded GL leaf + `WalkableLevelIT` (`338c852`); and the final-review fast-follow (`1582157`). Test counts on the merge: `beholder-engine-core` 21, `beholder-render-core` 34, `beholder-render-libgdx` 5 in-gate + `WalkableLevelIT` **3/3 against real `LEVEL1.MAZ`, 0 skipped**.
- **Correctness (MASTER §8).** The movement/collision logic is proven by synthetic-geometry unit tests (every command, every collision outcome, off-grid, the full turn cycle) and confirmed on real data by `WalkableLevelIT`'s churn-free structural invariants (four `TURN_RIGHT` restore the facing+cell; 40 `FORWARD` steps never leave the 32×32 grid and settle at a wall; a blocked step is a no-op). The opus whole-branch review hand-traced all six commands × four facings through `MovementCommand.stepped → Facing.delta → EDGE_DIR_BY_DELTA → LevelGeometry.archetype` and found the current-cell travel-direction edge correct throughout; the final-review fast-follow added the E/S/W `SOLID`-collision tests that were missing in-gate — **all passed against the real table (no `EDGE_DIR_BY_DELTA` bug)**. **Maintainer-verified the rendered walk by eye** (contact-sheet PNG of a scripted path); ScummVM is still not installed, so a pixel-exact kyra golden stays deferred (M2 posture).
- **Real-data / plan corrections during execution:**
  - **libGDX is the reactor's first UI dependency and tripped the aggregate license-check.** Fixed in the root `pom.xml` (maintainer-signed-off): a `licenseMerges` entry normalises LWJGL's SPDX `BSD-3-Clause` onto the already-allowed `BSD 3-Clause License` family, and unused LGPL audio transitives (`jlayer`/`jorbis`) are excluded from `gdx-backend-lwjgl3`. Verified (task + opus review) to be dependency hygiene / string normalisation — **not** a gate relaxation (the build-config allowlist and `includedLicenses`/`excludedLicenses` are untouched). darkmoor (libGDX 1.5.3 / LWJGL2 / Java 6) was confirmed **not** a coordinate reference; the module uses libGDX **1.12.1** (LWJGL3).
  - `Facing.left/right` uses ordinal arithmetic and needed `@SuppressWarnings("EnumOrdinal")` (ErrorProne `-Werror`), with a Javadoc documenting the fixed clockwise N,E,S,W order the quarter-turn cycles over.
  - `beholder-engine-core` introduced a project-local `spotbugs-exclude.xml` (via the parent's own documented `vp.spotbugs.excludeFilterFile` override) to allow the deliberate-`null` fail-closed guard tests in `Party`/`Level`.
  - `GridPosition` gained `equals`/`hashCode` (M2 had deferred this) — the movement/state comparisons need value-equality.
  - `WalkableLevelIT` and `LibGdxLauncher` start the party at the M2 reference cell `(10,12)` NORTH; that cell is near a corner, so the party is boxed in after two cells — a plausible real geometry, honestly flagged (a more dynamic free-walk demo is a deferred follow-up).
- **IR/render shape.** New IR: `Facing.left/right`, `GridPosition` value-equality. `beholder-engine-core` (`Party`/`Level` immutable values in `engine.model` — coverage-exempt, guards behavior-tested; `MovementCommand`/`RuntimeState` covered). `beholder-render-core` gained `Rgba.toRgba8888` + `ViewportFit`/`ViewportRect`. `beholder-render-libgdx` is the app/composition root; the display-only GL leaf (`render/libgdx/gl/**`) carries the **only** new, documented JaCoCo exclusion, everything else in-gate. Dependency graph stays an acyclic engine-eats-IR-only DAG (opus-verified).
- **ADR-0003 M3 grey-area rulings** recorded and enforced: no movement tween (instant snap, stateless renderer), pillarbox at the original FoV (no widening), structural discrete input.

**Deferred follow-ups (documented, non-blocking):** a dynamic free-walk demo / interactive-window eyeball run (the automated flow has no display); the frozen pixel-hash frame ratchet (would churn at M4 doors, per M2's ruling — the state sequence is the durable golden); GL-window resize handling (`resize()`/`Viewport` hook; the window is `setResizable(false)` for now); and a few test-completeness polish minors (spotbugs class-scoped test filter, `RuntimeState.applyCommand` null-guard, duplicate test geometry builders, unnamed test literals). **M4 owns** the `.INF` event graph + one working trigger. No game assets committed (skip-safe ITs; the contact-sheet PNG stays in git-ignored `target/`).

**Next:** M4 — `.INF` event graph + one trigger (a pressure plate → door/teleport that fires identically to `kyra`/DOSBox), introducing `engine-core`'s executor + the IR event layer + the deferred `.INF 0xFB` wall-mapping table (doors/decorations).
