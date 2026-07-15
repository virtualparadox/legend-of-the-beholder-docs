# P2 · M4 — EOB2 lossless round-trip + kyra differential — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close Phase 2 by proving a real, unmodified EOB2 `LEVEL1`'s canonical on-disk IR is lossless, byte-stable, and renders identically through the *unchanged* `beholder-ir-json` codec, and by making the reviewable synthetic golden's `WallMappingTable` (M3's one new IR node) non-trivial.

**Architecture:** Two test-only tasks, zero production code. Task 1 adds `Eob2LosslessRoundTripIT` in `beholder-app` (where `Eob2Extractor` lives; `beholder-render-libgdx`, which holds EOB1's `LosslessRoundTripIT`, cannot depend on `beholder-app`), mirroring the EOB1 template's three legs (equality, byte-stability, render-identity) but building `IR_a` directly from `Eob2Extractor.extract`'s in-memory decode. Task 2 populates the hand-authored `SyntheticLevel` document's wall table and regenerates its committed golden.

**Tech Stack:** Java 21, Maven multi-module reactor, JUnit 5 + AssertJ, JaCoCo, the reused `beholder-ir-json` codec (`IrJson.writeDocument`/`readDocument`/`toCanonicalJson`), the reused IR-fed render path (`GameSession`/`SceneOverlay`/`IrDecorationScene`/`IrDoorLeafScene`).

## Global Constraints

- **JDK 21**: `export JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home`; `mvn spotless:apply` then `mvn verify`; single-module builds use `-pl <m> -am` **or** root verify (the stale `~/.m2` upstream-jar trap).
- **All `CODING_STANDARDS` gates green (all `-Werror`)**: one `return`/method (local-var accumulation, no early returns), **no `instanceof`**, no `switch`-yield, cyclomatic ≤ 8, NPath ≤ 120, ≤ 5 params, ≤ 18 methods & ≤ 12 fields per class, CBO/coupling ≤ 10 (whole-file), no `return null`, `final` params, braces everywhere, ≤ 140-char lines, **no `System.out`** (write PNGs via `ImageIO`, never print), full Javadoc on public/protected only, every magic value a named `ALL_CAPS` constant + a what/why comment. An end-to-end IT that spans many seams uses `@SuppressWarnings("PMD.CouplingBetweenObjects")` with an inline justification (as `LosslessRoundTripIT`/`Eob2BootWalkIT` do).
- **Coverage:** JaCoCo ≥ 0.90 branch AND line, **Surefire (unit) only** — skip-safe Failsafe ITs do NOT count. Task 1 is a skip-safe **IT** (no jacoco burden). Task 2 touches a unit-level fixture + golden (`SyntheticGoldenTest` is Surefire) but adds no production logic.
- **Clean-room; never commit game data:** the real-data IT is `assumeTrue`-skip-safe; every real-`LEVEL1` artifact (`level1.json`, `level1-assets/`, the before/after PNG) goes to git-ignored `target/`. The synthetic golden is our own content and IS committed.
- **Codec must stay UNCHANGED (the milestone's point):** if either task needs to touch `beholder-ir-json` production code, STOP — that is a game-neutrality finding to surface, not to patch.
- **Process:** TDD per task (write test → run → commit), fresh implementer + independent review per task, whole-branch review, `--no-ff` merge on the maintainer's verdict; conventional branches, no worktrees.

---

## File Structure

| File | Responsibility | Task |
|---|---|---|
| `beholder-app/src/test/java/eu/virtualparadox/beholder/app/Eob2LosslessRoundTripIT.java` (**create**) | The EOB2 real-`LEVEL1` lossless round-trip IT: three legs (equality, byte-stability, render-identity), skip-safe, artifacts to `target/`. | 1 |
| `beholder-ir-json/src/test/java/eu/virtualparadox/beholder/ir/json/SyntheticLevel.java` (**modify** — line ~89) | Give the hand-authored reviewable document a non-trivial `WallMappingTable`. | 2 |
| `beholder-ir-json/src/test/resources/golden/synthetic-level.json` (**modify** — regenerate) | The committed byte-exact golden; regenerated after the fixture change. | 2 |

**Interfaces this plan consumes (verified present — exact signatures):**
- `Eob2Extractor` — `public Eob2Extractor() {}`, `public IrDocument extract(final Path gameFolder, final String level)` (the in-memory decode → `IrDocument`, before any disk write).
- `IrJson` (all `public static`) — `void writeDocument(Path dir, String baseName, IrDocument doc) throws IOException`, `LevelDocument readDocument(Path dir, String baseName) throws IOException` returning an `IrDocument`… **NOTE:** verify `readDocument`'s return type in the source; the EOB1 `LosslessRoundTripIT` assigns it to `IrDocument irB = IrJson.readDocument(dir, BASE_NAME);` — use that exact call shape. Also `String toCanonicalJson(LevelDocument level)`.
- `IrDocument` — `record IrDocument(LevelDocument level, AppearancePack pack)`; accessors `.level()`, `.pack()`.
- `LevelDocument` — `record LevelDocument(String irVersion, LevelGeometry geometry, LevelEvents events, String appearancePackId, WallMappingTable wallMappings)`; accessors `.geometry()`, `.events()`, `.wallMappings()`.
- `WallMappingTable` — `static WallMappingTable of(int[] wallSetByIndex /*len 256*/)`, `int wallSetForIndex(int index)`, `static WallMappingTable empty()`.
- Render wiring (verbatim from `LosslessRoundTripIT.renderEntryFrame`): `Level.of(geometry, tileSet, events)`, `RuntimeState.startingAt(Party.of(pos, facing))`, `new IrDecorationScene(events.decorations(), pack)`, `new IrDoorLeafScene(events.doors(), doorClosedTest, pack.doorLeafSheet(), pack.tileSet().paletteArgb())`, `new SceneOverlay(decorations, doorLeaves, resolver)`, `new GameSession(level, state, inputSource, frameSink, sceneOverlay).tick()`, `FirstPersonRenderer.VIEWPORT_WIDTH`/`VIEWPORT_HEIGHT`.
- Skip-safe EOB2 access (verbatim from `Eob2BootWalkIT`): `GAME_FOLDER = Path.of(System.getProperty("user.home"), "Games/Eye of the Beholder II The Legend of Darkmoon.app/Contents/Resources/game")`, `MAZE_FILE = "LEVEL1.MAZ"`, `assumeTrue(Files.isReadable(GAME_FOLDER.resolve(MAZE_FILE)), "…")`.

---

### Task 1: `Eob2LosslessRoundTripIT` — the EOB2 real-`LEVEL1` lossless round-trip

**Files:**
- Create: `beholder-app/src/test/java/eu/virtualparadox/beholder/app/Eob2LosslessRoundTripIT.java`

**Interfaces:**
- Consumes: `Eob2Extractor().extract(GAME_FOLDER, "LEVEL1") -> IrDocument`; `IrJson.writeDocument/readDocument/toCanonicalJson`; the render wiring above.
- Produces: nothing (a leaf acceptance IT).

**Nature of this task (read first):** This is a **verification IT over already-working code** — no production code is added. The codec + render path already serve EOB1; this proves they serve EOB2 unchanged (game-neutrality). So the test should go **GREEN on first run** against the local EOB2 install. If any leg goes RED, that is a genuine game-neutrality finding — do NOT patch the codec to make it pass; stop and report the exact mismatch (it means an EOB2 IR value does not round-trip, which is the milestone's job to surface).

- [ ] **Step 1: Write the IT**

Create `beholder-app/src/test/java/eu/virtualparadox/beholder/app/Eob2LosslessRoundTripIT.java`:

```java
package eu.virtualparadox.beholder.app;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assumptions.assumeTrue;

import eu.virtualparadox.beholder.engine.MovementCommand;
import eu.virtualparadox.beholder.engine.RuntimeState;
import eu.virtualparadox.beholder.engine.model.Level;
import eu.virtualparadox.beholder.engine.model.Party;
import eu.virtualparadox.beholder.ir.Facing;
import eu.virtualparadox.beholder.ir.GridPosition;
import eu.virtualparadox.beholder.ir.json.IrDocument;
import eu.virtualparadox.beholder.ir.json.IrJson;
import eu.virtualparadox.beholder.ir.model.AppearancePack;
import eu.virtualparadox.beholder.ir.model.LevelDocument;
import eu.virtualparadox.beholder.render.DecorationScene;
import eu.virtualparadox.beholder.render.DoorClosedTest;
import eu.virtualparadox.beholder.render.DoorLeafScene;
import eu.virtualparadox.beholder.render.FirstPersonRenderer;
import eu.virtualparadox.beholder.render.IrDecorationScene;
import eu.virtualparadox.beholder.render.IrDoorLeafScene;
import eu.virtualparadox.beholder.render.libgdx.FrameSink;
import eu.virtualparadox.beholder.render.libgdx.GameSession;
import eu.virtualparadox.beholder.render.libgdx.InputSource;
import eu.virtualparadox.beholder.render.libgdx.SceneOverlay;
import java.awt.image.BufferedImage;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.Map;
import java.util.TreeMap;
import java.util.function.IntUnaryOperator;
import java.util.stream.Stream;
import javax.imageio.ImageIO;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

/**
 * P2 M4 acceptance test (design &sect;3.2), the EOB2 mirror of EOB1's {@code LosslessRoundTripIT}:
 * proves a real, unmodified Eye of the Beholder II {@code LEVEL1}'s canonical on-disk IR is
 * <em>lossless</em>, <em>byte-stable</em>, and renders <em>identically</em> after a full
 * write&rarr;reload round-trip through the reused, unchanged {@code beholder-ir-json} codec &mdash;
 * the game-neutrality of the serde, and Phase 2's machine-checkable close.
 *
 * <p><b>What it round-trips.</b> {@code IR_a} is the in-memory decode straight from {@link
 * Eob2Extractor#extract} (no disk write yet); it is written under git-ignored {@code target/} and read
 * back as {@code IR_b}. The three legs mirror EOB1 M5 design &sect;6: (a) whole-IR value equality; (b)
 * byte-stability (re-serialize + full re-write reproduce byte-identical files); (c) render-identity of
 * the entry viewpoint. The render's wall resolver is built from the document's own {@link
 * LevelDocument#wallMappings()} (purely IR-sourced); it is inert for this override-free static probe
 * (no {@code SET_WALL} fires), and the table's own losslessness is proven by leg (a).
 *
 * <p>Lives in {@code beholder-app} (not beside EOB1's {@code LosslessRoundTripIT} in {@code
 * beholder-render-libgdx}) because it consumes {@link Eob2Extractor}, and {@code beholder-render-libgdx}
 * cannot depend on {@code beholder-app} (reactor direction).
 *
 * <p>Skip-safe: the EOB2 install is not part of the repository, so this test is skipped (not failed)
 * when it is absent, mirroring {@code Eob2BootWalkIT}. No game data is committed: every artifact goes to
 * git-ignored {@code target/}.
 *
 * <p>{@code @SuppressWarnings("PMD.CouplingBetweenObjects")}: an end-to-end acceptance proof necessarily
 * spans the extractor, the IR, the JSON/asset codec, and every IR-fed render seam &mdash; the same
 * rationale {@code LosslessRoundTripIT}/{@code Eob2BootWalkIT} document for this identical suppression.
 */
@SuppressWarnings("PMD.CouplingBetweenObjects")
class Eob2LosslessRoundTripIT {

    /** Root folder of the locally-installed EOB2 game data (never part of the repository). */
    private static final Path GAME_FOLDER = Path.of(
            System.getProperty("user.home"),
            "Games/Eye of the Beholder II The Legend of Darkmoon.app/Contents/Resources/game");

    /** One of the level-specific files {@link Eob2Extractor} reads; presence gates this test. */
    private static final String MAZE_FILE = "LEVEL1.MAZ";

    /** The level {@link Eob2Extractor} decodes. */
    private static final String LEVEL = "LEVEL1";

    /** The artifact base name shared by the JSON file and the sibling asset folder. */
    private static final String BASE_NAME = "level1";

    /** Extension of the canonical JSON file the codec writes next to {@link #BASE_NAME}. */
    private static final String JSON_SUFFIX = ".json";

    /** Image format string for the before/after contact sheet. */
    private static final String PNG_FORMAT = "png";

    /** Scratch-only output directory (module {@code target/}, git-ignored). */
    private static final Path OUT_DIR = Path.of("target");

    /** File name of the before/after (IR_a vs IR_b) contact-sheet PNG, written under {@link #OUT_DIR}. */
    private static final String BEFORE_AFTER_PNG = "eob2-m4-roundtrip-before-after.png";

    /**
     * The EOB2 interior entry viewpoint reused verbatim from {@code Eob2BootWalkIT}: {@code (col=5,
     * row=4)}, whose north edge is a door (a non-degenerate frame, so render-identity is a meaningful
     * pixel comparison rather than an all-black view).
     */
    private static final GridPosition ENTRY = GridPosition.of(5, 4);

    /** The party's starting facing: north, dead ahead of {@link #ENTRY}'s door. */
    private static final Facing ENTRY_FACING = Facing.NORTH;

    /**
     * The round-trip's base door state: every door tests CLOSED (its leaf is drawn). The reload has no
     * live runtime to consult, so leg (c) pins a constant all-closed test, making both renders
     * deterministic (mirrors EOB1's {@code LosslessRoundTripIT}).
     */
    private static final DoorClosedTest ALL_DOORS_CLOSED = (block, edge) -> true;

    /** Empty-byte sentinel so a byte-comparison lookup stays non-null; the key-set equality guarantees presence. */
    private static final byte[] NO_BYTES = new byte[0];

    Eob2LosslessRoundTripIT() {}

    @Test
    void realEob2Level1RoundTripsLosslessByteStableAndPixelIdentical(@TempDir final Path dir) throws IOException {
        assumeTrue(Files.isReadable(GAME_FOLDER.resolve(MAZE_FILE)), "EOB2 game data present at " + GAME_FOLDER);

        final IrDocument irA = new Eob2Extractor().extract(GAME_FOLDER, LEVEL);
        IrJson.writeDocument(dir, BASE_NAME, irA);
        final IrDocument irB = IrJson.readDocument(dir, BASE_NAME);

        // Leg (a) -- losslessness: whole-IR value equality (geometry, events, doors, decorations,
        // the WallMappingTable, and the appearance pack).
        assertThat(irB).isEqualTo(irA);
        // Leg (b) -- byte-stability: the reload re-serializes to the exact on-disk bytes.
        assertByteStability(dir, irB);
        // Leg (c) -- render-identity: the entry view renders pixel-identically from IR_a and IR_b.
        assertRenderIdentity(irA, irB);
    }

    /**
     * Asserts leg (b): the reloaded {@code IR_b} re-serializes to the exact on-disk {@code level1.json}
     * text, and a second full write of {@code IR_b} reproduces byte-identical files (the canonical JSON
     * plus every asset-folder file).
     *
     * @param dir the folder the artifact was written to
     * @param irB the reloaded document
     * @throws IOException if any artifact file cannot be read or re-written
     */
    private static void assertByteStability(final Path dir, final IrDocument irB) throws IOException {
        final String onDiskJson = Files.readString(dir.resolve(BASE_NAME + JSON_SUFFIX), StandardCharsets.UTF_8);
        assertThat(IrJson.toCanonicalJson(irB.level())).isEqualTo(onDiskJson);

        final Map<String, byte[]> before = snapshotFiles(dir);
        IrJson.writeDocument(dir, BASE_NAME, irB);
        final Map<String, byte[]> after = snapshotFiles(dir);

        assertThat(after.keySet()).isEqualTo(before.keySet());
        assertSameBytes(before, after);
    }

    /** Asserts every file present before the re-write has byte-identical content after it. */
    private static void assertSameBytes(final Map<String, byte[]> before, final Map<String, byte[]> after) {
        for (final Map.Entry<String, byte[]> entry : before.entrySet()) {
            final byte[] rewritten = after.getOrDefault(entry.getKey(), NO_BYTES);
            assertThat(rewritten).as("file %s", entry.getKey()).isEqualTo(entry.getValue());
        }
    }

    /** Snapshots every regular file under {@code dir} as a deterministic relative-path &rarr; bytes map. */
    private static Map<String, byte[]> snapshotFiles(final Path dir) throws IOException {
        final Map<String, byte[]> files = new TreeMap<>();
        try (Stream<Path> walk = Files.walk(dir)) {
            final List<Path> regularFiles = walk.filter(Files::isRegularFile).sorted().toList();
            for (final Path file : regularFiles) {
                files.put(dir.relativize(file).toString(), Files.readAllBytes(file));
            }
        }
        return files;
    }

    /**
     * Asserts leg (c): the entry viewpoint renders pixel-for-pixel identically from {@code IR_a} and
     * {@code IR_b}, and writes a before/after contact sheet to git-ignored {@code target/} for a human
     * eyeball.
     *
     * @param irA the freshly-decoded document
     * @param irB the reloaded document
     * @throws IOException if the contact-sheet PNG cannot be written
     */
    private static void assertRenderIdentity(final IrDocument irA, final IrDocument irB) throws IOException {
        final int[] frameA = renderEntryFrame(irA);
        final int[] frameB = renderEntryFrame(irB);

        Files.createDirectories(OUT_DIR);
        ImageIO.write(tile(List.of(frameA, frameB)), PNG_FORMAT, OUT_DIR.resolve(BEFORE_AFTER_PNG).toFile());

        assertThat(frameB).isEqualTo(frameA);
    }

    /**
     * Renders the entry viewpoint from one document through the IR-fed render path (the same {@link
     * GameSession#tick()} wiring the app renders through). The render {@link Level} is rebuilt purely
     * from the document's own geometry, events, and pack tile set; the wall resolver reads the
     * document's own {@link LevelDocument#wallMappings()} &mdash; nothing outside the IR.
     */
    private static int[] renderEntryFrame(final IrDocument ir) {
        final LevelDocument document = ir.level();
        final AppearancePack pack = ir.pack();
        final IntUnaryOperator resolver = index -> document.wallMappings().wallSetForIndex(index);
        final Level level = Level.of(document.geometry(), pack.tileSet(), document.events());
        final RuntimeState state = RuntimeState.startingAt(Party.of(ENTRY, ENTRY_FACING));
        final DecorationScene decorations = new IrDecorationScene(document.events().decorations(), pack);
        final DoorLeafScene doorLeaves = new IrDoorLeafScene(
                document.events().doors(), ALL_DOORS_CLOSED, pack.doorLeafSheet(), pack.tileSet().paletteArgb());
        final RecordingFrameSink sink = new RecordingFrameSink();
        new GameSession(level, state, NoOpInput.INSTANCE, sink, new SceneOverlay(decorations, doorLeaves, resolver))
                .tick();
        return sink.lastFrame();
    }

    /** Tiles a sequence of {@code VIEWPORT_WIDTH}x{@code VIEWPORT_HEIGHT} ARGB frames side by side into one image. */
    private static BufferedImage tile(final List<int[]> frames) {
        final int width = FirstPersonRenderer.VIEWPORT_WIDTH;
        final int height = FirstPersonRenderer.VIEWPORT_HEIGHT;
        final BufferedImage sheet = new BufferedImage(width * frames.size(), height, BufferedImage.TYPE_INT_ARGB);
        for (int i = 0; i < frames.size(); i++) {
            sheet.setRGB(i * width, 0, width, height, frames.get(i), 0, width);
        }
        return sheet;
    }

    /** Test double: an {@link InputSource} that never queues a command (this probe never moves the party). */
    private enum NoOpInput implements InputSource {
        INSTANCE;

        @Override
        public List<MovementCommand> poll() {
            return List.of();
        }
    }

    /** Test double: records the last frame it was asked to present. */
    private static final class RecordingFrameSink implements FrameSink {

        private int[] lastFrame = new int[0];

        @Override
        public void present(final int[] frame) {
            this.lastFrame = frame;
        }

        int[] lastFrame() {
            return lastFrame;
        }
    }
}
```

- [ ] **Step 2: Verify accessor/return-type shapes against the real sources before running**

The code above mirrors `LosslessRoundTripIT` (render wiring + `snapshotFiles`/`assertByteStability`) and `Eob2BootWalkIT` (skip-safe + `GAME_FOLDER`/`ENTRY`). Before running, open both templates and confirm each call matches verbatim: `IrJson.readDocument(dir, BASE_NAME)` returns an `IrDocument` (EOB1 test assigns exactly this); `pack.doorLeafSheet()`, `pack.tileSet().paletteArgb()`, `Level.of(...)`, `new GameSession(level, state, input, sink, overlay)`, `new IrDecorationScene(decorations, pack)`, `new IrDoorLeafScene(doors, closedTest, doorLeafSheet, paletteArgb)`. Fix any signature that has drifted (e.g. an extra param) to match the template exactly. Do NOT invent new APIs.

- [ ] **Step 3: Compile + run the IT (expect GREEN against the local EOB2 install)**

```bash
export JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home
mvn -q spotless:apply
mvn -q -pl beholder-app -am verify -Dit.test=Eob2LosslessRoundTripIT -Dfailsafe.failIfNoSpecifiedTests=false
```

Expected: `Eob2LosslessRoundTripIT` runs (real EOB2 data present) and **passes 1/1** (Skipped: 0). Confirm via the failsafe report:

```bash
cat beholder-app/target/failsafe-reports/eu.virtualparadox.beholder.app.Eob2LosslessRoundTripIT.txt
```
Expected: `Tests run: 1, Failures: 0, Errors: 0, Skipped: 0`.

**If any leg fails:** do NOT touch the codec. Capture the exact failure (which leg; for leg (a) the AssertJ diff of `IR_a` vs `IR_b`; for leg (b) the mismatching file/JSON) and report it — it is a real game-neutrality finding (an EOB2 IR value that does not round-trip losslessly), which is precisely what this milestone exists to surface.

- [ ] **Step 4: Confirm the whole module + gates are green (nothing else regressed)**

```bash
mvn -q -pl beholder-app -am verify
```
Expected: BUILD SUCCESS — all Surefire + Failsafe green, checkstyle/PMD/CPD/SpotBugs/NullAway clean, JaCoCo ≥ 0.90 (the new IT adds no production code, so no coverage movement).

- [ ] **Step 5: Eyeball the artifact + commit**

Open `beholder-app/target/eob2-m4-roundtrip-before-after.png` — the two tiles (IR_a vs IR_b) must be visually identical (a closed door dead ahead). Confirm `git status` shows only the new `.java` file (the `target/` artifacts are git-ignored — verify none are staged).

```bash
git add beholder-app/src/test/java/eu/virtualparadox/beholder/app/Eob2LosslessRoundTripIT.java
git commit -m "test(p2-m4): EOB2 real-LEVEL1 lossless round-trip IT (equality + byte-stable + render-identity)"
```

---

### Task 2: Make the reviewable synthetic golden's `WallMappingTable` non-trivial

**Files:**
- Modify: `beholder-ir-json/src/test/java/eu/virtualparadox/beholder/ir/json/SyntheticLevel.java:89` (the `WallMappingTable.empty()` in `document()`)
- Modify (regenerate): `beholder-ir-json/src/test/resources/golden/synthetic-level.json`

**Interfaces:**
- Consumes: `WallMappingTable.of(int[])`.
- Produces: nothing (a self-contained fixture + golden edit).

**Nature of this task (read first):** `SyntheticGoldenTest` pins `SyntheticLevel.document()` against the committed golden `golden/synthetic-level.json` from both sides (byte-exact write + round-trip read). Today the fixture's wall table is `WallMappingTable.empty()` (256 zeros), so the *reviewable* canonical golden — the "is this the IR we want as the product?" artifact — misleadingly shows an all-zeros wall table. This task makes it a small, realistic, non-zero table so the reviewable golden honestly represents a level with wall mappings (M3's new IR node). **Scope note:** this is a *reviewability* improvement to the byte-exact golden; the wall table's *round-trip correctness* is already covered by `IrDocumentRoundTripTest`'s populated-table case (M3 Task 6), so a reviewer should not over-weight this task. Only `SyntheticLevel` / `synthetic-level.json` change; the maximal-vocabulary fixture `SyntheticLevelDocuments` / `level-document.canonical.json` is intentionally left as-is.

- [ ] **Step 1: Change the fixture to a populated table (RED — golden goes stale)**

In `beholder-ir-json/src/test/java/eu/virtualparadox/beholder/ir/json/SyntheticLevel.java`, add a named constant for the populated table and use it in `document()`. Add the import `import eu.virtualparadox.beholder.ir.model.WallMappingTable;` (already present). Insert the constant near the other `static final` fields:

```java
    /**
     * A small, hand-authored non-trivial wall-mapping table so the reviewable golden honestly shows a
     * populated {@link WallMappingTable} (M3's new IR node) rather than an all-zeros placeholder. The
     * seeded slots use readable EOB-flavoured indices (a teleporter id 44, a wall-of-force id 74) mapped
     * to distinct wall-set values; every other index stays 0.
     */
    private static final int[] SYNTHETIC_WALL_SETS = synthetic256WallSets();

    /** The fixed number of wall-index slots a {@link WallMappingTable} holds (its {@code of(int[])} guards this length). */
    private static final int WALL_INDEX_COUNT = 256;

    /** Builds the 256-entry wall-set table with a handful of distinct non-zero slots. */
    private static int[] synthetic256WallSets() {
        final int[] table = new int[WALL_INDEX_COUNT];
        table[1] = 1;
        table[5] = 3;
        table[44] = 18;
        table[69] = 5;
        table[74] = 2;
        return table;
    }
```

(`WallMappingTable`'s own `WALL_INDEX_COUNT` is private, so this fixture declares its own named constant — matching the codebase's existing per-class-constant convention for this format fact.)

Then change the `document()` return (line ~89) from:

```java
        return new LevelDocument(IR_VERSION, geometry(), events, PACK_ID, WallMappingTable.empty());
```
to:
```java
        return new LevelDocument(IR_VERSION, geometry(), events, PACK_ID, WallMappingTable.of(SYNTHETIC_WALL_SETS));
```

- [ ] **Step 2: Run the golden test to confirm it goes RED (stale golden)**

```bash
export JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home
mvn -q -pl beholder-ir-json test -Dtest=SyntheticGoldenTest
```
Expected: FAIL — `serializesToTheCommittedGoldenByteForByte` mismatches (the golden still holds the old all-zeros `wallMappings`); `roundTripsTheGoldenBackToTheSameDocument` also fails (the golden's document no longer equals the fixture). This confirms the golden is genuinely pinned and now stale.

- [ ] **Step 3: Regenerate the committed golden**

Regenerate `golden/synthetic-level.json` from the updated fixture with a one-off writer. Create a throwaway test `RegenerateGoldenScratch` in `beholder-ir-json/src/test/java/eu/virtualparadox/beholder/ir/json/`:

```java
package eu.virtualparadox.beholder.ir.json;

import java.nio.file.Files;
import java.nio.file.Path;
import org.junit.jupiter.api.Test;

class RegenerateGoldenScratch {
    @Test
    void regenerate() throws Exception {
        Files.writeString(
                Path.of("src/test/resources/golden/synthetic-level.json"),
                IrJson.toCanonicalJson(SyntheticLevel.document()));
    }
}
```

Run it, then DELETE it:

```bash
mvn -q -pl beholder-ir-json test -Dtest=RegenerateGoldenScratch
rm beholder-ir-json/src/test/java/eu/virtualparadox/beholder/ir/json/RegenerateGoldenScratch.java
```

Inspect the diff — the ONLY change to `synthetic-level.json` must be inside the `wallMappings` array (slots 1, 5, 44, 69, 74 now non-zero; everything else unchanged):

```bash
git diff beholder-ir-json/src/test/resources/golden/synthetic-level.json
```
Expected: only five array entries change from `0` to their new values; no other field moves.

- [ ] **Step 4: Confirm the golden test is GREEN + the module verifies**

```bash
mvn -q spotless:apply
mvn -q -pl beholder-ir-json -am verify
```
Expected: BUILD SUCCESS — `SyntheticGoldenTest` 2/2 green (byte-exact + round-trip), all other ir-json tests still green (`IrJsonWriteTest`/`IrJsonReadTest`/`SchemaValidationTest` use the *separate* `level-document.canonical.json`, untouched), gates + JaCoCo green. Confirm no `RegenerateGoldenScratch.java` remains (`git status`).

- [ ] **Step 5: Commit**

```bash
git add beholder-ir-json/src/test/java/eu/virtualparadox/beholder/ir/json/SyntheticLevel.java \
        beholder-ir-json/src/test/resources/golden/synthetic-level.json
git commit -m "test(p2-m4): populate the reviewable synthetic golden's WallMappingTable (M3 new node)"
```

---

## Self-Review

**1. Spec coverage.** Design §2 "in scope": (i) the `Eob2LosslessRoundTripIT` (3 legs) → Task 1 ✓; (ii) the populated synthetic golden `WallMappingTable` → Task 2 ✓; (iii) differential-evidence consolidation → captured in the design doc §4 (no code) and surfaced at the §7 checkpoint (the whole-branch review + maintainer verdict), not a code task ✓. Design §2 "out of scope" (DOSBox pixel capture, new codec/serde, new vocabulary) → nothing in this plan adds them ✓. Design §3.2 three legs → Task 1's three asserts map 1:1 ✓. Design §3.3 "only `SyntheticLevel`/`synthetic-level.json` change; `SyntheticLevelDocuments` left as-is" → Task 2 scope note ✓.

**2. Placeholder scan.** No "TBD"/"handle edge cases"/"similar to". Task 1 carries the full IT verbatim; Task 2 carries the exact fixture edit + the regeneration procedure. The one deliberate branch — `WallMappingTable.WALL_INDEX_COUNT` vs literal `256` — is resolved inline with a concrete fallback, not left open. Step 2 of Task 1 ("verify accessor shapes against the templates") is a real safeguard against signature drift, not a placeholder — the code is complete; the step just guards the copy against the live sources.

**3. Type consistency.** `IrDocument.level()`/`.pack()`, `LevelDocument.geometry()`/`.events()`/`.wallMappings()`, `WallMappingTable.of(int[])`/`wallSetForIndex(int)`, `IrJson.writeDocument`/`readDocument`/`toCanonicalJson`, and the render wiring names are all used identically to the two templates (`LosslessRoundTripIT`, `Eob2BootWalkIT`) and to M3's merged `WallMappingTable`/`LevelDocument`. `ENTRY = (5,4)` NORTH matches `Eob2BootWalkIT.ENTRY`. No name defined in one task is used differently in another.

**Note for the executor:** Task 1 is a verification IT over unchanged production code — a GREEN first run is the expected, and desired, outcome (it proves game-neutrality). A RED leg is a real finding to surface, never a cue to modify the codec.
