# P0 — Project Skeleton & Gate-Proof Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the `legend-of-the-beholder` Maven multi-module reactor under the `eu.virtualparadox:parent` quality gates, with one real, fully-tested IR primitive proving every gate (including ≥90 % branch+line coverage) is green from commit zero.

**Architecture:** A reactor-root aggregator POM (`packaging: pom`) inherits `eu.virtualparadox:parent:1.0.0` and lists modules. This plan creates only the first module, `beholder-ir` (the canonical IR schema layer), containing a single `GridPosition` value type grounded in the reconnaissance finding that EOB levels are a fixed 32×32 block grid (`blockIndex = y*32 + x`). Further modules (`beholder-importer-westwood`, `beholder-engine-core`, `beholder-render-libgdx`, `beholder-app`) are added just-in-time by their own plans (YAGNI — no empty modules).

**Tech Stack:** Java 21, Maven 3.9.6+, `eu.virtualparadox:parent:1.0.0` (Palantir format, Checkstyle, PMD, SpotBugs, NullAway, JaCoCo), JUnit Jupiter + AssertJ (test).

**Naming (confirmed):** groupId `eu.virtualparadox.beholder`, root artifactId `legend-of-the-beholder`, module `beholder-ir`, base package `eu.virtualparadox.beholder`. The product working title remains TBD (MASTER §1); these Maven coordinates are the confirmed choice for the build.

**Target repository:** the code repo `~/Workspaces/Java/legend-of-the-beholder` (currently only `README.md` + `AGENTS.md`).

## Global Constraints

Every task's requirements implicitly include these (verbatim from `CODING_STANDARDS.md` / the parent). Full values live in `CODING_STANDARDS.md`; the load-bearing ones here:

- Java 21 (`maven.compiler.release=21`); Maven 3.9.6+.
- `<parent>` = `eu.virtualparadox:parent:1.0.0` with `<relativePath/>`.
- All consumer code under package prefix `eu.virtualparadox.beholder.*` (NullAway annotated prefix).
- Palantir Java Format (`mvn spotless:apply` writes it, `spotless:check` verifies); max line length 140; no tabs; every file ends with a newline; no wildcard imports; one statement per line; braces on all control flow; uppercase `L` for long literals.
- Method parameters `final`; local variables `final` where the rule applies; do not reassign parameters.
- Exactly one `return` per method (`OnlyOneReturn`); no `instanceof`; no switch `yield`; no reference-equality object comparison; cyclomatic complexity ≤8/method, ≤40/class; NPath ≤120; ≤12 fields, ≤18 methods, ≤6 parameters per class; coupling ≤10.
- `return null;` forbidden — use `Optional`, immutable empty collections, or throw. NullAway runs at ERROR for `eu.virtualparadox` packages.
- No `System.out`/`System.err` (use SLF4J); no `printStackTrace()`; no `System.exit`/`Runtime.exec`; **no `Thread.sleep(...)` (including tests)**.
- Javadoc required on public/protected types and methods (methods annotated `@Override`/`@Test`/`@BeforeEach`/… are exempt); `doclint=all` — every Javadoc warning fails the build. Include `@param`/`@return`/`@throws` tags.
- Coverage gate: JaCoCo ≥90 % **branch** and ≥90 % **line** at bundle level (data carriers under `**/model/**`, `*Dto`, `*Entity`, `*Record`, etc. are excluded — keep `GridPosition` OUT of those names/paths so it IS measured).
- Unit test files named `*Test.java`; JUnit Jupiter + AssertJ.
- Local routine before every commit: `mvn spotless:apply` then `mvn verify`.

## Execution note (project workflow)

Per the project workflow (conventional branches, `--no-ff` merges, never worktrees): do all work on a branch `feat/p0-skeleton`, commit per task, and merge to `main` with `--no-ff` at the end (final step of Task 2). Commit trailers as configured for this session.

## File Structure

```
legend-of-the-beholder/                     (existing git repo; branch feat/p0-skeleton)
├── pom.xml                                  CREATE — reactor-root aggregator (packaging: pom)
├── .gitignore                               CREATE — target/, IDE, .DS_Store
└── beholder-ir/                                 CREATE — the IR schema module
    ├── pom.xml                              CREATE — module POM (parent = reactor root)
    └── src/
        ├── main/java/eu/virtualparadox/beholder/ir/
        │   └── GridPosition.java            CREATE (Task 2) — 32×32 grid cell value type
        └── test/java/eu/virtualparadox/beholder/ir/
            └── GridPositionTest.java        CREATE (Task 2) — full-branch tests
```

---

### Task 1: Reactor root + `beholder-ir` module scaffold

**Files:**
- Create: `pom.xml` (reactor root)
- Create: `.gitignore`
- Create: `beholder-ir/pom.xml`
- Create: `beholder-ir/src/main/java/eu/virtualparadox/beholder/ir/` (empty dir, kept by Task 2's file)

**Interfaces:**
- Consumes: `eu.virtualparadox:parent:1.0.0` (resolved from `~/.m2`).
- Produces: reactor coordinates `eu.virtualparadox.beholder:legend-of-the-beholder:0.1.0-SNAPSHOT`; module artifact `beholder-ir`; base package `eu.virtualparadox.beholder.ir`.

- [ ] **Step 1: Create the feature branch**

Run:
```bash
git -C ~/Workspaces/Java/legend-of-the-beholder checkout -b feat/p0-skeleton
```
Expected: `Switched to a new branch 'feat/p0-skeleton'`.

- [ ] **Step 2: Write the reactor-root `pom.xml`**

Create `~/Workspaces/Java/legend-of-the-beholder/pom.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>eu.virtualparadox</groupId>
        <artifactId>parent</artifactId>
        <version>1.0.0</version>
        <relativePath/>
    </parent>

    <groupId>eu.virtualparadox.beholder</groupId>
    <artifactId>legend-of-the-beholder</artifactId>
    <version>0.1.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <name>Legend of the Beholder</name>
    <description>Grid-Crawler Engine Platform: one clean IR-driven engine for EOB I/II/III and Dungeon Hack.</description>

    <modules>
        <module>beholder-ir</module>
    </modules>

    <properties>
        <!-- Keep shared suppression files resolvable from the reactor root across all submodules. -->
        <vp.suppressions.dir>${maven.multiModuleProjectDirectory}</vp.suppressions.dir>
    </properties>
</project>
```

- [ ] **Step 3: Write `.gitignore`**

Create `~/Workspaces/Java/legend-of-the-beholder/.gitignore`:
```gitignore
target/
*.class
.DS_Store
.idea/
*.iml
*~
```

- [ ] **Step 4: Write the `beholder-ir` module `pom.xml`**

Create `~/Workspaces/Java/legend-of-the-beholder/beholder-ir/pom.xml`:
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

    <artifactId>beholder-ir</artifactId>

    <name>Beholder IR</name>
    <description>Canonical intermediate representation: the versioned, fail-closed content schema all clients consume.</description>

    <dependencies>
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

- [ ] **Step 5: Create the main source directory**

Run:
```bash
mkdir -p ~/Workspaces/Java/legend-of-the-beholder/beholder-ir/src/main/java/eu/virtualparadox/beholder/ir
mkdir -p ~/Workspaces/Java/legend-of-the-beholder/beholder-ir/src/test/java/eu/virtualparadox/beholder/ir
```

- [ ] **Step 6: Verify the reactor is valid**

Run:
```bash
mvn -q -f ~/Workspaces/Java/legend-of-the-beholder/pom.xml validate
```
Expected: `BUILD SUCCESS`, reactor lists `legend-of-the-beholder` and `beholder-ir`, no POM errors. (If it fails resolving `eu.virtualparadox:parent:1.0.0`, install it first: `mvn -f ~/Workspaces/Java/parent install -DskipTests`.)

- [ ] **Step 7: Commit the scaffold**

Run:
```bash
git -C ~/Workspaces/Java/legend-of-the-beholder add pom.xml .gitignore beholder-ir/pom.xml
git -C ~/Workspaces/Java/legend-of-the-beholder commit -m "build: scaffold reactor + beholder-ir module"
```

---

### Task 2: `GridPosition` IR primitive (TDD, proves all gates)

**Files:**
- Create: `beholder-ir/src/main/java/eu/virtualparadox/beholder/ir/GridPosition.java`
- Test: `beholder-ir/src/test/java/eu/virtualparadox/beholder/ir/GridPositionTest.java`

**Interfaces:**
- Consumes: nothing (leaf type).
- Produces:
  - `GridPosition.of(int x, int y)` → `GridPosition` (validates `0 ≤ x,y < 32`, throws `IllegalArgumentException` otherwise).
  - `GridPosition#x()` → `int`, `GridPosition#y()` → `int`.
  - `GridPosition#blockIndex()` → `int` (`y*32 + x`, range 0..1023 — the EOB block id).
  - `GridPosition.GRID_SIZE` = `32` (public constant).

- [ ] **Step 1: Write the failing test**

Create `~/Workspaces/Java/legend-of-the-beholder/beholder-ir/src/test/java/eu/virtualparadox/beholder/ir/GridPositionTest.java`:
```java
package eu.virtualparadox.beholder.ir;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import org.junit.jupiter.api.Test;

class GridPositionTest {

    @Test
    void ofStoresCoordinatesAndComputesBlockIndex() {
        final GridPosition pos = GridPosition.of(3, 5);
        assertThat(pos.x()).isEqualTo(3);
        assertThat(pos.y()).isEqualTo(5);
        assertThat(pos.blockIndex()).isEqualTo(5 * 32 + 3);
    }

    @Test
    void ofAcceptsBothCornersOfTheGrid() {
        assertThat(GridPosition.of(0, 0).blockIndex()).isEqualTo(0);
        assertThat(GridPosition.of(31, 31).blockIndex()).isEqualTo(1023);
    }

    @Test
    void ofRejectsNegativeX() {
        assertThatThrownBy(() -> GridPosition.of(-1, 0))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void ofRejectsXAtOrAboveGridSize() {
        assertThatThrownBy(() -> GridPosition.of(32, 0))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void ofRejectsNegativeY() {
        assertThatThrownBy(() -> GridPosition.of(0, -1))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void ofRejectsYAtOrAboveGridSize() {
        assertThatThrownBy(() -> GridPosition.of(0, 32))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run:
```bash
mvn -q -pl beholder-ir -f ~/Workspaces/Java/legend-of-the-beholder/pom.xml test
```
Expected: FAIL — compilation error, `GridPosition` does not exist.

- [ ] **Step 3: Write the minimal implementation**

Create `~/Workspaces/Java/legend-of-the-beholder/beholder-ir/src/main/java/eu/virtualparadox/beholder/ir/GridPosition.java`:
```java
package eu.virtualparadox.beholder.ir;

/**
 * An immutable cell coordinate on the fixed 32x32 EOB level grid.
 *
 * <p>The engine's levels are a 32x32 block grid; a block's linear id is
 * {@code y * 32 + x} (range 0..1023), matching the original Westwood layout.
 */
public final class GridPosition {

    /** The fixed edge length of an EOB level grid. */
    public static final int GRID_SIZE = 32;

    private final int x;
    private final int y;

    private GridPosition(final int x, final int y) {
        this.x = x;
        this.y = y;
    }

    /**
     * Creates a validated grid position.
     *
     * @param x the column, {@code 0 <= x < 32}
     * @param y the row, {@code 0 <= y < 32}
     * @return the position
     * @throws IllegalArgumentException if {@code x} or {@code y} is outside the grid
     */
    public static GridPosition of(final int x, final int y) {
        if (x < 0 || x >= GRID_SIZE || y < 0 || y >= GRID_SIZE) {
            throw new IllegalArgumentException("Coordinate out of 0.." + (GRID_SIZE - 1) + " range: (" + x + ", " + y + ")");
        }
        return new GridPosition(x, y);
    }

    /**
     * Returns the column.
     *
     * @return the x coordinate
     */
    public int x() {
        return x;
    }

    /**
     * Returns the row.
     *
     * @return the y coordinate
     */
    public int y() {
        return y;
    }

    /**
     * Returns the linear block id ({@code y * 32 + x}, range 0..1023).
     *
     * @return the block index
     */
    public int blockIndex() {
        return y * GRID_SIZE + x;
    }
}
```

- [ ] **Step 4: Format the code**

Run:
```bash
mvn -q -pl beholder-ir -f ~/Workspaces/Java/legend-of-the-beholder/pom.xml spotless:apply
```
Expected: BUILD SUCCESS (reformats to Palantir style if needed).

- [ ] **Step 5: Run the full verify to confirm all gates + coverage pass**

Run:
```bash
mvn -q -f ~/Workspaces/Java/legend-of-the-beholder/pom.xml verify
```
Expected: BUILD SUCCESS. All 6 tests pass; Checkstyle/PMD/SpotBugs/NullAway/Javadoc clean; JaCoCo reports ≥90 % branch and line for `beholder-ir` (all branches of the `of(...)` validation are exercised, and `x()`/`y()`/`blockIndex()` are covered).

- [ ] **Step 6: Commit**

Run:
```bash
git -C ~/Workspaces/Java/legend-of-the-beholder add beholder-ir/src
git -C ~/Workspaces/Java/legend-of-the-beholder commit -m "feat(ir): add GridPosition 32x32 grid cell primitive"
```

- [ ] **Step 7: Merge the branch to main (--no-ff) and push**

Run:
```bash
cd ~/Workspaces/Java/legend-of-the-beholder
git checkout main
git merge --no-ff feat/p0-skeleton -m "Merge branch 'feat/p0-skeleton'"
git branch -d feat/p0-skeleton
git push
```
Expected: fast-forward-free merge commit on `main`; push succeeds.

---

## Self-Review

**Spec coverage (against the P0 §Phase-0 scope in `PHASES.md`):**
- "Project structure — Maven multi-module under `eu.virtualparadox:parent` (Java 21, gates green, ≥90 % coverage)" → Tasks 1–2. ✅ (Modules beyond `beholder-ir` are deferred to their own plans by design — YAGNI, no empty modules.)
- "Resolve §11 open decisions in writing" → NOT in this plan — deferred to the P0-decisions/ADR plan.
- "AESOP lift spike / `.INF` decompiler spike / PAK extractor" → NOT in this plan — each is its own P0 plan (see below).

**Placeholder scan:** none — every step has exact paths, complete file contents, exact commands, and expected output.

**Type consistency:** `GridPosition.of(int,int)`, `x()`, `y()`, `blockIndex()`, `GRID_SIZE` are used identically in the test (Task 2 Step 1) and the implementation (Task 2 Step 3).

## Remaining P0 plans (written next, on request — sequenced)

1. **This plan** — reactor skeleton + `beholder-ir` gate-proof.
2. **EOB1 PAK extractor** — `beholder-importer-westwood` module; TDD a Westwood PAK reader against the real `~/Games/Eye of the Beholder.app/…/EOBDATA1.PAK`; unpack EOB1 for later use.
3. **`.INF` decompiler spike** — prototype lifting one EOB1 level script into `event → condition → action`, using `eoblib`'s `Script.java` as a reference; deliverable = a feasibility note + a tiny prototype (research-flavoured, not strict TDD).
4. **AESOP lift spike** — run DAESOP on `EYE.RES`, hand-lift 1–2 message handlers into `event → condition → action`; deliverable = a feasibility note (the MASTER §6 risk, retired before the IR is frozen).
5. **§11 decision ADRs** — IR serialisation format and the rendering-fidelity envelope, informed by spikes 3–4.

## Execution outcome (2026-07-06)

Executed subagent-driven. Merged to `main` as `4e65596` (`--no-ff`), all gates green on JDK 21. Commits: `269cbbb` scaffold → `71cadb3` GridPosition → `02a84b3` JDK-21 fail-fast enforcer + build docs.

**Environment finding:** the build requires **JDK 21.x** — the machine default (JDK 25/26) passes the parent's `requireJavaVersion [21,)` but then crashes Palantir Java Format (spotless). Fixed fail-fast: the reactor root enforces `requireJavaVersion [21,22)` with a clear message, and the JDK-21 requirement is documented in the code-repo `README.md` and `AGENTS.md`.

**Deferred follow-ups (from the final whole-branch review — carry into P1):**
- **`GridPosition` `equals`/`hashCode`/`toString`:** deferred (YAGNI), but the **first task that uses `GridPosition` as a `HashMap` key / `HashSet` member MUST add `equals`/`hashCode` (+ tests)** — identity semantics would otherwise silently produce wrong results.
- **PMD `AtLeastOneConstructor` on test classes:** every `*Test` class currently needs a dummy no-arg constructor. Exclude this rule from test sources in the parent (`eu.virtualparadox:parent`) so it stops recurring.
- **Value-type convention:** decide project-wide whether IR primitives are hand-written `final class` (as here, chosen to stay JaCoCo-measured and avoid the `*Record` coverage exclusion) or Java `record`s; standardise `equals`/`hashCode`/`toString`.
- **Minor niceties:** message-content assertions on the rejection tests; `package-info.java` (`@NullMarked`) for `eu.virtualparadox.beholder.ir` as the package grows.
- **Optional hardening:** move the JDK-21 pin (toolchains / capped enforcer) into the parent so every `eu.virtualparadox` consumer benefits.
