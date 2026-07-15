# EOB3 AESOP importer + walkable third game — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make a real EOB3 level walkable from the shared IR by reading `EYE.RES`, lifting its 32×32 occupancy grid into `LevelGeometry`, and booting it in the existing libGDX front-end (placeholder walls; real EOB3 render is M2).

**Architecture:** A new `beholder-importer-aesop` module (the AESOP decoder quarantine, mirroring `beholder-importer-westwood`) owns the clean-room `.RES` reader, the ROED name→number resolver, the occupancy-maze reader, the occupancy→per-edge geometry mapping, and a synthetic placeholder `AppearancePack`. `beholder-app` gets `Eob3Extractor` + `GameKind.EOB3` + a `GameDetector` marker; the shared `beholder-ir`/`beholder-engine-core` stay untouched, so walkability comes for free.

**Tech Stack:** Java 21 (Corretto), Maven reactor, JUnit 5 + AssertJ, the existing `beholder-ir` / `beholder-render-core` / `beholder-engine-core` / `beholder-render-libgdx` stack. Pure-JDK decoders (no new runtime deps).

**Design spec:** `plans/2026-07-16-p3-m1-eob3-importer-walkable-design.md`. **Recon/spike evidence:** `notes/2026-07-15-eob3-aesop-recon.md`, `spikes/2026-07-16-eob3-render-spike.md`.

## Global Constraints

Every task's code and tests must satisfy these (verbatim from `CODING_STANDARDS.md` / the parent poms). They are implicit in every task.

- **JDK 21 only.** `export JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home` before any `mvn`. The machine default JDK breaks Palantir spotless.
- **Build/verify:** `mvn spotless:apply` then `mvn verify`. Because `beholder-app` depends on the new module, verify downstream with `-am` or a root `mvn verify` (single-module builds resolve a stale `~/.m2` upstream jar — memory `maven-reactor-am-trap`).
- **Format (spotless, Palantir):** final params; final locals where applicable; max line length 140; no wildcard/unused imports; braces on all control flow; one statement/one variable per line; uppercase `L` long literals.
- **One return per method** (PMD `OnlyOneReturn`) — structure guards as nested `if`/computed booleans, not early `return`. **No `instanceof`.** No param reassignment. ≤8 cyclomatic/method, ≤5 effective parameters, ≤12 fields, ≤18 methods/class, coupling ≤10.
- **Javadoc** on every public/protected type AND method (`@param`/`@return`/`@throws`); `@Override`/`@Test`/`@BeforeEach` etc. are exempt; Javadoc doclint runs `all` and **every warning fails the build**.
- **No `return null;`** (throw or return an empty immutable collection). NullAway ERROR for `eu.virtualparadox.*`.
- **No `System.out`/`System.err`/`printStackTrace()`/`Thread.sleep`** (forbidden-apis) — use SLF4J if logging is needed (none is, here).
- **Named constants + what/why comment:** every offset/magic/bitmask is a `private static final` with a Javadoc saying what it is and why (clean-room-format rule; see `Maze.java`/`VgaPalette.java`).
- **Coverage (JaCoCo):** ≥90% branch AND ≥90% line at bundle level. `beholder-ir/.../ir/model/**` is coverage-exempt (pure data), but **importer and app logic classes are not** — decoders/extractor need thorough unit tests.
- **Clean-room:** the AESOP `thirdeye`/original-source oracles are GPL/reference — **never copy their code**; implement from the facts in the recon/spike notes. **Never commit game data or game-art images** (skip-safe `assumeTrue` ITs; artifacts to git-ignored `target/`).
- **House pattern** for every decoder value class (see `beholder-importer-westwood/.../Maze.java`): `public final class`, private constructor, `public static X parse(byte[])` factory, fail-closed `require*` guards (each throwing `IllegalArgumentException` with a message), scalar accessors that never expose internal arrays, explicit constructor even in tests.

---

## File Structure

**New module `beholder-importer-aesop`** (package root `eu.virtualparadox.beholder.importer.aesop`):
- `ResArchive.java` — the `.RES` container reader (by-number payload lookup). *Parent package.*
- `Roed.java` — resource-0 dictionary parser (name→number). *Parent package.*
- `eob3/OccupancyMaze.java` — the 32×32 occupancy grid (open/solid). *`eob3` sub-package.*
- `eob3/AesopToIr.java` — occupancy→per-edge `LevelGeometry` (the anti-corruption mapper). *`eob3` sub-package.*
- `eob3/PlaceholderAppearancePack.java` — a self-contained flat `AppearancePack` (no game assets). *`eob3` sub-package.*
- `src/test/.../aesop/SyntheticRes.java` — test helper: builds a minimal in-memory `.RES` (header + one dir block + entries), incl. a ROED blob and a map grid.
- Tests: `ResArchiveTest`, `RoedTest`, `eob3/OccupancyMazeTest`, `eob3/AesopToIrTest`, `eob3/PlaceholderAppearancePackTest`.

**`beholder-app`:**
- Create `Eob3Extractor.java`. Modify `GameKind.java`, `GameDetector.java`, `Extract.java`, `pom.xml` (add module dep).
- Tests: create `Eob3ExtractorTest`; modify `GameDetectorTest`, `ExtractTest`, `SyntheticGameFolders.java` (add `writeEob3Assets`).

**`beholder-render-libgdx` (test tree):**
- Create `RealEob3EyeRes.java` (skip-safe locator), `Eob3WalkableIT.java`, `Eob3LosslessRoundTripIT.java`.

**Modify root `pom.xml`** (register the new module).

---

## Task 1: Module scaffold + `ResArchive` + `SyntheticRes`

**Files:**
- Create: `beholder-importer-aesop/pom.xml`
- Modify: `pom.xml` (root `<modules>`)
- Create: `beholder-importer-aesop/src/main/java/eu/virtualparadox/beholder/importer/aesop/ResArchive.java`
- Create (test-support, **main scope + public**): `beholder-importer-aesop/src/main/java/eu/virtualparadox/beholder/importer/aesop/SyntheticRes.java`
- Test: `beholder-importer-aesop/src/test/java/eu/virtualparadox/beholder/importer/aesop/ResArchiveTest.java`

**Interfaces:**
- Produces: `ResArchive.parse(byte[]) -> ResArchive`; `boolean ResArchive.contains(int number)`; `byte[] ResArchive.resource(int number)` (clone of the payload; throws `IllegalArgumentException` if absent). `SyntheticRes` — a **public, main-scope** documented test-support builder (it must be reachable from this module's `eob3` sub-package tests AND from `beholder-app`'s tests, so it cannot be a package-private test class; main-scope avoids a test-jar and a CPD-duplicated `.RES` builder; alternatively a Maven test-jar): `public static byte[] of(Map<Integer,byte[]> payloadsByNumber)` builds a valid `.RES`; `public static byte[] roed(Map<String,Integer> names)` builds a ROED blob; `public static byte[] occupancy(boolean[][] open32x32)` builds a 1024-byte grid.

- [ ] **Step 1: Register the module and create the pom.**

Add to root `pom.xml` `<modules>`, right after `beholder-importer-westwood`:
```xml
    <module>beholder-importer-aesop</module>
```
Create `beholder-importer-aesop/pom.xml` (mirror `beholder-importer-westwood/pom.xml` verbatim; only artifactId/name/description differ):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>eu.virtualparadox.beholder</groupId>
        <artifactId>legend-of-the-beholder</artifactId>
        <version>0.1.0-SNAPSHOT</version>
    </parent>
    <artifactId>beholder-importer-aesop</artifactId>
    <name>Beholder Importer: AESOP</name>
    <description>Anti-corruption importer for the AESOP EYE.RES (EOB3) container into the canonical IR.</description>
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

- [ ] **Step 2: Write the failing `ResArchiveTest`.**

`ResArchiveTest.java` (uses `SyntheticRes`, written in Step 4 — write both, test first):
```java
package eu.virtualparadox.beholder.importer.aesop;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.util.Map;
import org.junit.jupiter.api.Test;

class ResArchiveTest {

    ResArchiveTest() {}

    @Test
    void readsPayloadsByResourceNumberAcrossTheDirectoryChain() {
        final byte[] res = SyntheticRes.of(Map.of(5, new byte[] {1, 2, 3}, 200, new byte[] {9}));
        final ResArchive archive = ResArchive.parse(res);
        assertThat(archive.contains(5)).isTrue();
        assertThat(archive.resource(5)).containsExactly(1, 2, 3);
        assertThat(archive.contains(200)).isTrue(); // number 200 lives in the 2nd directory block
        assertThat(archive.resource(200)).containsExactly(9);
    }

    @Test
    void reportsAbsentResourcesAndRejectsLookupOfThem() {
        final byte[] res = SyntheticRes.of(Map.of(5, new byte[] {1}));
        final ResArchive archive = ResArchive.parse(res);
        assertThat(archive.contains(6)).isFalse();
        assertThatThrownBy(() -> archive.resource(6)).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rejectsBytesTooShortForAHeader() {
        assertThatThrownBy(() -> ResArchive.parse(new byte[8])).isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 3: Run it — expect FAIL** (`ResArchive`/`SyntheticRes` undefined).

Run: `export JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home && mvn -q -pl beholder-importer-aesop test`
Expected: compile failure (symbols not found).

- [ ] **Step 4: Write `SyntheticRes` (test helper) and `ResArchive`.**

`SyntheticRes.java` — builds a valid `.RES` from the confirmed layout (36-byte header, 644-byte dir blocks, 12-byte entry headers). Multiple resource numbers spread across blocks (number `N` → block `N/128`, slot `N%128`). Keep it minimal but correct:
```java
package eu.virtualparadox.beholder.importer.aesop;

import java.io.ByteArrayOutputStream;
import java.nio.ByteBuffer;
import java.nio.ByteOrder;
import java.nio.charset.StandardCharsets;
import java.util.Map;
import java.util.TreeMap;

/**
 * Public test-support builder for a minimal, valid AESOP {@code .RES} byte image (and its ROED /
 * occupancy-grid payloads). Lives in main scope (not a test class) so this module's cross-package
 * {@code eob3} tests and downstream modules' tests (e.g. {@code beholder-app}) can reuse it without
 * a Maven test-jar and without duplicating the {@code .RES} builder (which the CPD gate forbids).
 */
public final class SyntheticRes {

    /** Byte length of the global header (signature[16] + 5 u32 fields). */
    private static final int HEADER_LENGTH = 36;
    /** File offset of the first directory block (immediately after the header). */
    private static final int FIRST_DIR_OFFSET = HEADER_LENGTH;
    /** Slots per directory block. */
    private static final int DIR_ENTRIES = 128;
    /** Directory block length: u32 next + u8[128] attrs + u32[128] indices. */
    private static final int DIR_BLOCK_LENGTH = 4 + DIR_ENTRIES + DIR_ENTRIES * 4;
    /** Entry header length: u32 storage_time + u32 data_attributes + u32 data_size. */
    private static final int ENTRY_HEADER_LENGTH = 12;
    /** 32x32 occupancy grid byte length. */
    private static final int GRID_LENGTH = 1024;
    /** Occupancy value meaning "open floor". */
    private static final int OPEN = 0xFF;

    private SyntheticRes() {}

    /**
     * Builds a {@code .RES} image whose directory maps each given resource number to its payload.
     *
     * @param payloadsByNumber resource number → payload bytes
     * @return a valid {@code .RES} byte image
     */
    public static byte[] of(final Map<Integer, byte[]> payloadsByNumber) {
        final TreeMap<Integer, byte[]> sorted = new TreeMap<>(payloadsByNumber);
        final int blockCount = sorted.isEmpty() ? 1 : sorted.lastKey() / DIR_ENTRIES + 1;
        final int dirRegion = FIRST_DIR_OFFSET + blockCount * DIR_BLOCK_LENGTH;
        // Layout: header | dir blocks | entry headers+payloads. Compute entry offsets first.
        final int[] blockNext = new int[blockCount];
        for (int b = 0; b < blockCount; b++) {
            blockNext[b] = b + 1 < blockCount ? FIRST_DIR_OFFSET + (b + 1) * DIR_BLOCK_LENGTH : 0;
        }
        final ByteArrayOutputStream body = new ByteArrayOutputStream();
        final TreeMap<Integer, Integer> entryOffsetByNumber = new TreeMap<>();
        int cursor = dirRegion;
        for (final Map.Entry<Integer, byte[]> e : sorted.entrySet()) {
            entryOffsetByNumber.put(e.getKey(), cursor);
            writeU32(body, 0);
            writeU32(body, 0);
            writeU32(body, e.getValue().length);
            body.writeBytes(e.getValue());
            cursor += ENTRY_HEADER_LENGTH + e.getValue().length;
        }
        final byte[] out = new byte[cursor];
        final ByteBuffer buf = ByteBuffer.wrap(out).order(ByteOrder.LITTLE_ENDIAN);
        System.arraycopy("AESOP/16 V1.00\0".getBytes(StandardCharsets.US_ASCII), 0, out, 0, 15);
        buf.putInt(0x10, out.length);
        buf.putInt(0x18, FIRST_DIR_OFFSET);
        for (int b = 0; b < blockCount; b++) {
            final int base = FIRST_DIR_OFFSET + b * DIR_BLOCK_LENGTH;
            buf.putInt(base, blockNext[b]);
            for (int slot = 0; slot < DIR_ENTRIES; slot++) {
                final int number = b * DIR_ENTRIES + slot;
                final boolean used = entryOffsetByNumber.containsKey(number);
                out[base + 4 + slot] = (byte) (used ? 0 : 1);
                buf.putInt(base + 0x84 + slot * 4, used ? entryOffsetByNumber.get(number) : 0);
            }
        }
        System.arraycopy(body.toByteArray(), 0, out, dirRegion, body.size());
        return out;
    }

    /**
     * Builds a ROED dictionary blob (one bucket) mapping each name to its resource number as ASCII decimal.
     *
     * @param names resource name → resource number
     * @return a ROED blob (the resource-0 payload)
     */
    public static byte[] roed(final Map<String, Integer> names) {
        final ByteArrayOutputStream chain = new ByteArrayOutputStream();
        for (final Map.Entry<String, Integer> e : new TreeMap<>(names).entrySet()) {
            writeLenPrefixed(chain, e.getKey());
            writeLenPrefixed(chain, Integer.toString(e.getValue()));
        }
        chain.write(0);
        chain.write(0); // u16 zero-length terminator
        final int headerLen = 2 + 4; // u16 bucketCount + one u32 chain offset
        final ByteArrayOutputStream out = new ByteArrayOutputStream();
        final ByteBuffer head = ByteBuffer.allocate(headerLen).order(ByteOrder.LITTLE_ENDIAN);
        head.putShort((short) 1);
        head.putInt(headerLen);
        out.writeBytes(head.array());
        out.writeBytes(chain.toByteArray());
        return out.toByteArray();
    }

    /**
     * Builds a 1024-byte occupancy grid: {@code open[y][x]} true -> {@code 0xFF}, false -> {@code 0x00}.
     *
     * @param open a 32×32 open/solid mask
     * @return the 1024-byte grid payload
     */
    public static byte[] occupancy(final boolean[][] open) {
        final byte[] grid = new byte[GRID_LENGTH];
        for (int y = 0; y < 32; y++) {
            for (int x = 0; x < 32; x++) {
                grid[y * 32 + x] = (byte) (open[y][x] ? OPEN : 0x00);
            }
        }
        return grid;
    }

    private static void writeLenPrefixed(final ByteArrayOutputStream out, final String s) {
        final byte[] ascii = s.getBytes(StandardCharsets.US_ASCII);
        final int len = ascii.length + 1; // includes trailing NUL
        out.write(len & 0xFF);
        out.write((len >> 8) & 0xFF);
        out.writeBytes(ascii);
        out.write(0);
    }

    private static void writeU32(final ByteArrayOutputStream out, final int v) {
        out.write(v & 0xFF);
        out.write((v >> 8) & 0xFF);
        out.write((v >> 16) & 0xFF);
        out.write((v >> 24) & 0xFF);
    }
}
```

`ResArchive.java` — the reader (adapt the spike; **one return per method**, guards as nested `if`, named constants + Javadoc):
```java
package eu.virtualparadox.beholder.importer.aesop;

import java.nio.ByteBuffer;
import java.nio.ByteOrder;
import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Clean-room reader for the AESOP {@code .RES} resource container (the EOB3 {@code EYE.RES} format).
 *
 * <p>Layout (little-endian, verified on real bytes): a 36-byte header whose {@code u32} at {@code
 * 0x18} is the file offset of the first directory block; a singly-linked chain of 644-byte
 * directory blocks (128 slots each: {@code u32 next@0}, {@code u8 attr[128]@4} where {@code 1} marks
 * a free slot, {@code u32 entryHeaderOffset[128]@0x84}); each non-free slot points at a 12-byte
 * entry header ({@code u32 storageTime}, {@code u32 dataAttributes}, {@code u32 dataSize}) whose
 * payload follows immediately. Resource number = {@code block * 128 + slot}. An entry whose
 * {@code dataAttributes} carries the placeholder bit has no payload and is skipped.
 */
public final class ResArchive {

    /** Header field: {@code u32} file offset of the first directory block. */
    private static final int HEADER_FIRST_DIR_OFFSET = 0x18;
    /** Directory slots per block. */
    private static final int DIR_ENTRIES = 128;
    /** Directory field: start of the per-slot storage-attribute bytes. */
    private static final int DIR_ATTR_OFFSET = 4;
    /** Directory field: start of the per-slot {@code u32} entry-header offsets. */
    private static final int DIR_INDEX_OFFSET = 0x84;
    /** Total directory block length ({@code u32 next} + attrs + indices). */
    private static final int DIR_BLOCK_LENGTH = 644;
    /** Entry-header length preceding each payload. */
    private static final int ENTRY_HEADER_LENGTH = 12;
    /** Entry-header field: {@code u32 dataAttributes}. */
    private static final int ENTRY_ATTR_OFFSET = 4;
    /** Entry-header field: {@code u32 dataSize}. */
    private static final int ENTRY_SIZE_OFFSET = 8;
    /** {@code dataAttributes} bit meaning "created by reference, no payload". */
    private static final int PLACEHOLDER_FLAG = 0x10000000;
    /** Storage-attribute value marking an unused directory slot. */
    private static final int FREE_SLOT = 1;
    /** Minimum bytes to hold the header. */
    private static final int MIN_LENGTH = 0x24;

    private final Map<Integer, byte[]> payloadByNumber;

    private ResArchive(final Map<Integer, byte[]> payloadByNumber) {
        this.payloadByNumber = payloadByNumber;
    }

    /**
     * Parses a {@code .RES} image into a resource-number → payload map.
     *
     * @param resBytes the full {@code .RES} file bytes; must be at least header-length
     * @return the parsed archive
     * @throws IllegalArgumentException if {@code resBytes} is too short to hold the header
     */
    public static ResArchive parse(final byte[] resBytes) {
        requireHeader(resBytes);
        final ByteBuffer buf = ByteBuffer.wrap(resBytes).order(ByteOrder.LITTLE_ENDIAN);
        final Map<Integer, byte[]> payloads = new LinkedHashMap<>();
        long dirOffset = u32(buf, HEADER_FIRST_DIR_OFFSET);
        int block = 0;
        while (dirOffset != 0 && dirOffset + DIR_BLOCK_LENGTH <= resBytes.length) {
            final int base = (int) dirOffset;
            for (int slot = 0; slot < DIR_ENTRIES; slot++) {
                collectSlot(resBytes, buf, base, slot, block, payloads);
            }
            dirOffset = u32(buf, base);
            block++;
        }
        return new ResArchive(payloads);
    }

    /** Adds one directory slot's payload to {@code payloads} when the slot is used and in-bounds. */
    private static void collectSlot(
            final byte[] resBytes,
            final ByteBuffer buf,
            final int base,
            final int slot,
            final int block,
            final Map<Integer, byte[]> payloads) {
        final int attr = resBytes[base + DIR_ATTR_OFFSET + slot] & 0xFF;
        final long entryOffset = u32(buf, base + DIR_INDEX_OFFSET + slot * 4);
        final boolean headerInBounds = entryOffset != 0 && entryOffset + ENTRY_HEADER_LENGTH <= resBytes.length;
        if (attr != FREE_SLOT && headerInBounds) {
            final long dataAttr = u32(buf, (int) entryOffset + ENTRY_ATTR_OFFSET);
            final long size = u32(buf, (int) entryOffset + ENTRY_SIZE_OFFSET);
            final boolean hasPayload =
                    (dataAttr & PLACEHOLDER_FLAG) == 0 && size > 0 && entryOffset + ENTRY_HEADER_LENGTH + size <= resBytes.length;
            if (hasPayload) {
                final byte[] payload = new byte[(int) size];
                System.arraycopy(resBytes, (int) (entryOffset + ENTRY_HEADER_LENGTH), payload, 0, (int) size);
                payloads.put(block * DIR_ENTRIES + slot, payload);
            }
        }
    }

    /**
     * Returns whether a resource with {@code number} is present.
     *
     * @param number the resource number
     * @return {@code true} if a payload exists for {@code number}
     */
    public boolean contains(final int number) {
        return payloadByNumber.containsKey(number);
    }

    /**
     * Returns a defensive copy of a resource's payload.
     *
     * @param number the resource number
     * @return a clone of the payload bytes
     * @throws IllegalArgumentException if no resource with {@code number} is present
     */
    public byte[] resource(final int number) {
        requirePresent(number);
        return payloadByNumber.get(number).clone();
    }

    private static void requireHeader(final byte[] resBytes) {
        if (resBytes.length < MIN_LENGTH) {
            throw new IllegalArgumentException("ResArchive input too short for a header: " + resBytes.length);
        }
    }

    private void requirePresent(final int number) {
        if (!payloadByNumber.containsKey(number)) {
            throw new IllegalArgumentException("ResArchive has no resource number " + number);
        }
    }

    private static long u32(final ByteBuffer buf, final int off) {
        return buf.getInt(off) & 0xFFFFFFFFL;
    }
}
```

- [ ] **Step 5: Run tests — expect PASS.** `mvn -q -pl beholder-importer-aesop test`. Expected: 3 green.

- [ ] **Step 6: Format + commit.**
```bash
export JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home
mvn -q -pl beholder-importer-aesop spotless:apply
git add pom.xml beholder-importer-aesop
git commit -m "feat(aesop): beholder-importer-aesop module + ResArchive .RES reader"
```

---

## Task 2: `Roed` — resource-0 name→number dictionary

**Files:**
- Create: `beholder-importer-aesop/src/main/java/.../importer/aesop/Roed.java`
- Test: `beholder-importer-aesop/src/test/java/.../importer/aesop/RoedTest.java`

**Interfaces:**
- Consumes: `SyntheticRes.roed(Map<String,Integer>)` (Task 1).
- Produces: `Roed.parse(byte[]) -> Roed`; `boolean Roed.contains(String name)`; `int Roed.number(String name)` (throws `IllegalArgumentException` if absent).

- [ ] **Step 1: Failing `RoedTest`.**
```java
package eu.virtualparadox.beholder.importer.aesop;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.util.Map;
import org.junit.jupiter.api.Test;

class RoedTest {

    RoedTest() {}

    @Test
    void resolvesNamesToResourceNumbers() {
        final Roed roed = Roed.parse(SyntheticRes.roed(Map.of("Mausoleum 1", 5, "Mausoleum walls", 253)));
        assertThat(roed.contains("Mausoleum 1")).isTrue();
        assertThat(roed.number("Mausoleum 1")).isEqualTo(5);
        assertThat(roed.number("Mausoleum walls")).isEqualTo(253);
    }

    @Test
    void rejectsUnknownNames() {
        final Roed roed = Roed.parse(SyntheticRes.roed(Map.of("Mausoleum 1", 5)));
        assertThat(roed.contains("Forest walls")).isFalse();
        assertThatThrownBy(() -> roed.number("Forest walls")).isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run — expect FAIL** (`Roed` undefined).

- [ ] **Step 3: Implement `Roed`** (dictionary blob: `u16 bucketCount`, `u32` chain offsets, chains of `{u16 len (incl. NUL), bytes}` alternating key/value, value = ASCII decimal; one return per method):
```java
package eu.virtualparadox.beholder.importer.aesop;

import java.nio.charset.StandardCharsets;
import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Parser for the AESOP ROED dictionary (resource 0): the master resource-name → resource-number
 * table. The blob is {@code u16 bucketCount}, then {@code bucketCount} {@code u32} chain offsets
 * (relative to the blob start); each non-zero chain is a run of length-prefixed strings ({@code u16
 * length} including a trailing NUL, then the bytes), alternating key (name) and value (the resource
 * number as ASCII decimal text), terminated by a zero-length string.
 */
public final class Roed {

    /** Blob field: {@code u16} number of hash buckets. */
    private static final int BUCKET_COUNT_OFFSET = 0;
    /** Blob field: start of the {@code u32} per-bucket chain offsets. */
    private static final int CHAIN_TABLE_OFFSET = 2;
    /** Bytes per chain offset. */
    private static final int CHAIN_OFFSET_SIZE = 4;
    /** Bytes of the {@code u16} length prefix on each string. */
    private static final int LENGTH_PREFIX_SIZE = 2;

    private final Map<String, Integer> numberByName;

    private Roed(final Map<String, Integer> numberByName) {
        this.numberByName = numberByName;
    }

    /**
     * Parses a ROED blob into a name → number map.
     *
     * @param blob the resource-0 payload
     * @return the parsed dictionary
     * @throws IllegalArgumentException if {@code blob} is too short to hold the bucket count
     */
    public static Roed parse(final byte[] blob) {
        requireBucketHeader(blob);
        final Map<String, Integer> map = new LinkedHashMap<>();
        final int bucketCount = u16(blob, BUCKET_COUNT_OFFSET);
        for (int bucket = 0; bucket < bucketCount; bucket++) {
            final int offsetPos = CHAIN_TABLE_OFFSET + bucket * CHAIN_OFFSET_SIZE;
            if (offsetPos + CHAIN_OFFSET_SIZE <= blob.length) {
                walkChain(blob, (int) u32(blob, offsetPos), map);
            }
        }
        return new Roed(map);
    }

    /** Reads alternating key/value strings from a chain, decimal-parsing each value into {@code map}. */
    private static void walkChain(final byte[] blob, final int start, final Map<String, Integer> map) {
        int p = start;
        boolean more = start > 0 && start < blob.length;
        while (more) {
            final int keyLen = u16(blob, p);
            more = keyLen != 0 && p + LENGTH_PREFIX_SIZE + keyLen <= blob.length;
            if (more) {
                final String key = ascii(blob, p + LENGTH_PREFIX_SIZE, keyLen);
                p += LENGTH_PREFIX_SIZE + keyLen;
                final int valLen = u16(blob, p);
                more = p + LENGTH_PREFIX_SIZE + valLen <= blob.length;
                if (more) {
                    final String value = ascii(blob, p + LENGTH_PREFIX_SIZE, valLen);
                    p += LENGTH_PREFIX_SIZE + valLen;
                    putDecimal(map, key, value);
                }
            }
        }
    }

    /** Stores {@code key -> parseInt(value)} when {@code value} is a decimal number; ignores non-numeric values. */
    private static void putDecimal(final Map<String, Integer> map, final String key, final String value) {
        final String trimmed = value.trim();
        final boolean numeric = !trimmed.isEmpty() && trimmed.chars().allMatch(Roed::isDigit);
        if (numeric) {
            map.put(key, Integer.parseInt(trimmed));
        }
    }

    private static boolean isDigit(final int c) {
        return c >= '0' && c <= '9';
    }

    /**
     * Returns whether {@code name} resolves to a resource number.
     *
     * @param name the resource name
     * @return {@code true} if present
     */
    public boolean contains(final String name) {
        return numberByName.containsKey(name);
    }

    /**
     * Resolves a resource name to its number.
     *
     * @param name the resource name
     * @return the resource number
     * @throws IllegalArgumentException if {@code name} is absent
     */
    public int number(final String name) {
        requirePresent(name);
        return numberByName.get(name);
    }

    private static void requireBucketHeader(final byte[] blob) {
        if (blob.length < CHAIN_TABLE_OFFSET) {
            throw new IllegalArgumentException("Roed blob too short: " + blob.length);
        }
    }

    private void requirePresent(final String name) {
        if (!numberByName.containsKey(name)) {
            throw new IllegalArgumentException("Roed has no name '" + name + "'");
        }
    }

    private static String ascii(final byte[] blob, final int off, final int len) {
        int real = len;
        while (real > 0 && blob[off + real - 1] == 0) {
            real--;
        }
        return new String(blob, off, real, StandardCharsets.US_ASCII);
    }

    private static int u16(final byte[] blob, final int off) {
        return (blob[off] & 0xFF) | ((blob[off + 1] & 0xFF) << 8);
    }

    private static long u32(final byte[] blob, final int off) {
        return (blob[off] & 0xFFL) | ((blob[off + 1] & 0xFFL) << 8) | ((blob[off + 2] & 0xFFL) << 16) | ((blob[off + 3] & 0xFFL) << 24);
    }
}
```

- [ ] **Step 4: Run — expect PASS** (`mvn -q -pl beholder-importer-aesop test`).

- [ ] **Step 5: Format + commit.**
```bash
mvn -q -pl beholder-importer-aesop spotless:apply
git add beholder-importer-aesop
git commit -m "feat(aesop): Roed resource-name to number dictionary"
```

---

## Task 3: `OccupancyMaze` — the 32×32 grid

**Files:**
- Create: `beholder-importer-aesop/src/main/java/.../importer/aesop/eob3/OccupancyMaze.java`
- Test: `beholder-importer-aesop/src/test/java/.../importer/aesop/eob3/OccupancyMazeTest.java`

**Interfaces:**
- Consumes: `SyntheticRes.occupancy(boolean[][])` (Task 1) — importer test tree can reference the parent-package helper (same module).
- Produces: `OccupancyMaze.parse(byte[]) -> OccupancyMaze`; `int OccupancyMaze.size()` (=32); `boolean OccupancyMaze.isOpen(int x, int y)` (throws on out-of-range). Constant `OccupancyMaze.GRID_SIZE = 32`.

- [ ] **Step 1: Failing `OccupancyMazeTest`.**
```java
package eu.virtualparadox.beholder.importer.aesop.eob3;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import eu.virtualparadox.beholder.importer.aesop.SyntheticRes;
import org.junit.jupiter.api.Test;

class OccupancyMazeTest {

    OccupancyMazeTest() {}

    @Test
    void readsOpenAndSolidCells() {
        final boolean[][] open = new boolean[32][32];
        open[3][7] = true; // (x=7, y=3) open
        final OccupancyMaze maze = OccupancyMaze.parse(SyntheticRes.occupancy(open));
        assertThat(maze.size()).isEqualTo(32);
        assertThat(maze.isOpen(7, 3)).isTrue();
        assertThat(maze.isOpen(0, 0)).isFalse();
    }

    @Test
    void rejectsWrongLengthAndOutOfRangeAccess() {
        assertThatThrownBy(() -> OccupancyMaze.parse(new byte[100])).isInstanceOf(IllegalArgumentException.class);
        final OccupancyMaze maze = OccupancyMaze.parse(new byte[1024]);
        assertThatThrownBy(() -> maze.isOpen(32, 0)).isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run — expect FAIL.**

- [ ] **Step 3: Implement `OccupancyMaze`.**
```java
package eu.virtualparadox.beholder.importer.aesop.eob3;

/**
 * The EOB3 dungeon maze as a fixed 32×32 occupancy grid — one byte per cell, where {@code 0xFF}
 * marks open floor and any other value marks solid rock. Interactive features (doors, levers,
 * monsters) are separate AESOP objects and are not part of this grid (M1 reads geometry only).
 */
public final class OccupancyMaze {

    /** The fixed EOB3 maze edge length (matches {@code GridPosition.GRID_SIZE}). */
    public static final int GRID_SIZE = 32;

    /** Total grid byte length ({@code GRID_SIZE * GRID_SIZE}, one byte per cell). */
    private static final int GRID_LENGTH = GRID_SIZE * GRID_SIZE;

    /** Occupancy byte value meaning "open floor"; any other value is solid. */
    private static final int OPEN = 0xFF;

    private final byte[] cells;

    private OccupancyMaze(final byte[] cells) {
        this.cells = cells;
    }

    /**
     * Parses a raw occupancy grid.
     *
     * @param mapBytes the 1024-byte grid payload
     * @return the parsed maze
     * @throws IllegalArgumentException if {@code mapBytes} is not exactly {@value #GRID_LENGTH} bytes
     */
    public static OccupancyMaze parse(final byte[] mapBytes) {
        if (mapBytes.length != GRID_LENGTH) {
            throw new IllegalArgumentException("OccupancyMaze grid must be " + GRID_LENGTH + " bytes, got " + mapBytes.length);
        }
        return new OccupancyMaze(mapBytes.clone());
    }

    /**
     * Returns the grid edge length.
     *
     * @return {@value #GRID_SIZE}
     */
    public int size() {
        return GRID_SIZE;
    }

    /**
     * Returns whether the cell at {@code (x, y)} is open floor.
     *
     * @param x the column, {@code 0..31}
     * @param y the row, {@code 0..31}
     * @return {@code true} if open, {@code false} if solid
     * @throws IllegalArgumentException if {@code x} or {@code y} is out of range
     */
    public boolean isOpen(final int x, final int y) {
        requireInRange(x, "x");
        requireInRange(y, "y");
        return (cells[y * GRID_SIZE + x] & 0xFF) == OPEN;
    }

    private static void requireInRange(final int value, final String axis) {
        if (value < 0 || value >= GRID_SIZE) {
            throw new IllegalArgumentException("OccupancyMaze " + axis + " out of range: " + value);
        }
    }
}
```

- [ ] **Step 4: Run — expect PASS.**

- [ ] **Step 5: Format + commit** (`git commit -m "feat(aesop): OccupancyMaze 32x32 grid reader"`).

---

## Task 4: `AesopToIr` — occupancy → per-edge `LevelGeometry`

**Files:**
- Create: `beholder-importer-aesop/src/main/java/.../importer/aesop/eob3/AesopToIr.java`
- Test: `beholder-importer-aesop/src/test/java/.../importer/aesop/eob3/AesopToIrTest.java`

**Interfaces:**
- Consumes: `OccupancyMaze` (Task 3); `LevelGeometry`, `WallArchetype` (`beholder-ir`).
- Produces: `AesopToIr.geometry(OccupancyMaze maze, int wallSet) -> LevelGeometry` — a 32×32 `LevelGeometry` where an edge is `SOLID` (carrying `wallSet`) iff exactly one of the two cells it separates is open (off-grid counts as solid), else `OPEN` (carrying `0`). Direction order matches `LevelGeometry`: `0=N,1=E,2=S,3=W`.

- [ ] **Step 1: Failing `AesopToIrTest`** (the occupancy→edge truth table):
```java
package eu.virtualparadox.beholder.importer.aesop.eob3;

import static org.assertj.core.api.Assertions.assertThat;

import eu.virtualparadox.beholder.ir.WallArchetype;
import eu.virtualparadox.beholder.ir.model.LevelGeometry;
import org.junit.jupiter.api.Test;

class AesopToIrTest {

    /** Direction constants matching LevelGeometry: N=0, E=1, S=2, W=3. */
    private static final int NORTH = 0;
    private static final int EAST = 1;

    AesopToIrTest() {}

    @Test
    void solidWhereOpenCellAbutsSolidOrBoundary_openWhereBothOpen() {
        // A 2-wide open corridor at row 5, columns 10 and 11; everything else solid.
        final boolean[][] open = new boolean[32][32];
        open[5][10] = true;
        open[5][11] = true;
        final OccupancyMaze maze = OccupancyMaze.parse(occupancyOf(open));
        final LevelGeometry geo = AesopToIr.geometry(maze, 1);

        // Between the two open cells (10,5)->E and (11,5)->W: OPEN, wallSet 0.
        assertThat(geo.archetype(10, 5, EAST)).isEqualTo(WallArchetype.OPEN);
        assertThat(geo.wallSet(10, 5, EAST)).isZero();
        // (10,5) north neighbour is solid -> SOLID wall carrying wallSet 1.
        assertThat(geo.archetype(10, 5, NORTH)).isEqualTo(WallArchetype.SOLID);
        assertThat(geo.wallSet(10, 5, NORTH)).isEqualTo(1);
        // A solid cell's edge toward the open corridor is SOLID too (blocks entry).
        assertThat(geo.archetype(10, 4, 2)).isEqualTo(WallArchetype.SOLID); // (10,4) south -> toward open (10,5)
        // Boundary: open cell forced against the grid edge would be SOLID; interior solid-solid is OPEN.
        assertThat(geo.archetype(0, 0, NORTH)).isEqualTo(WallArchetype.OPEN); // solid-solid + boundary both -> both solid -> OPEN
        assertThat(geo.width()).isEqualTo(32);
    }

    private static byte[] occupancyOf(final boolean[][] open) {
        return eu.virtualparadox.beholder.importer.aesop.SyntheticRes.occupancy(open);
    }
}
```
*(Note: `AesopToIrTest` references the parent-package `SyntheticRes` via its fully qualified name to avoid an import-visibility question across the `eob3` sub-package; either the FQN above or a same-package re-export helper is fine.)*

- [ ] **Step 2: Run — expect FAIL.**

- [ ] **Step 3: Implement `AesopToIr`.**
```java
package eu.virtualparadox.beholder.importer.aesop.eob3;

import eu.virtualparadox.beholder.ir.WallArchetype;
import eu.virtualparadox.beholder.ir.model.LevelGeometry;

/**
 * Anti-corruption mapper: derives a shared-IR {@link LevelGeometry} from an EOB3 {@link
 * OccupancyMaze}. EOB3 stores a single occupancy scalar per cell, whereas the IR is per-edge; a wall
 * stands on an edge exactly when it separates a walkable cell from solid rock (or the grid
 * boundary). This derivation is consistent for the two cells sharing an edge, so it blocks entry
 * into solid cells and the engine's collision works with no engine change.
 */
public final class AesopToIr {

    /** Edges per cell (N, E, S, W), matching {@link LevelGeometry}'s {@code dir} order. */
    private static final int DIRS_PER_CELL = 4;
    /** Per-direction column deltas for N, E, S, W. */
    private static final int[] DX = {0, 1, 0, -1};
    /** Per-direction row deltas for N, E, S, W (north = row-1). */
    private static final int[] DY = {-1, 0, 1, 0};
    /** Wall-set sentinel for an OPEN edge (draws nothing). */
    private static final int OPEN_WALL_SET = 0;

    private AesopToIr() {}

    /**
     * Builds the per-edge geometry for {@code maze}.
     *
     * @param maze the EOB3 occupancy grid
     * @param wallSet the wall-set index every SOLID edge carries (a valid index into the level's
     *     appearance pack's tile set)
     * @return a 32×32 {@link LevelGeometry}: SOLID (carrying {@code wallSet}) where an open cell
     *     abuts solid rock or the boundary, OPEN (carrying {@code 0}) otherwise
     */
    public static LevelGeometry geometry(final OccupancyMaze maze, final int wallSet) {
        final int n = maze.size();
        final WallArchetype[] archetypes = new WallArchetype[n * n * DIRS_PER_CELL];
        final int[] wallSets = new int[n * n * DIRS_PER_CELL];
        for (int y = 0; y < n; y++) {
            for (int x = 0; x < n; x++) {
                fillCell(maze, x, y, wallSet, archetypes, wallSets);
            }
        }
        return LevelGeometry.of(n, n, archetypes, wallSets, LevelGeometry.CURRENT_IR_VERSION);
    }

    /** Fills one cell's four edges into the flattened archetype/wall-set arrays. */
    private static void fillCell(
            final OccupancyMaze maze,
            final int x,
            final int y,
            final int wallSet,
            final WallArchetype[] archetypes,
            final int[] wallSets) {
        final int n = maze.size();
        final boolean openHere = maze.isOpen(x, y);
        for (int dir = 0; dir < DIRS_PER_CELL; dir++) {
            final int nx = x + DX[dir];
            final int ny = y + DY[dir];
            final boolean neighbourOpen = nx >= 0 && nx < n && ny >= 0 && ny < n && maze.isOpen(nx, ny);
            final boolean solidEdge = openHere != neighbourOpen;
            final int index = (y * n + x) * DIRS_PER_CELL + dir;
            archetypes[index] = solidEdge ? WallArchetype.SOLID : WallArchetype.OPEN;
            wallSets[index] = solidEdge ? wallSet : OPEN_WALL_SET;
        }
    }
}
```

- [ ] **Step 4: Run — expect PASS.**

- [ ] **Step 5: Format + commit** (`git commit -m "feat(aesop): AesopToIr occupancy to per-edge LevelGeometry"`).

---

## Task 5: `PlaceholderAppearancePack` — self-contained flat pack

**Files:**
- Create: `beholder-importer-aesop/src/main/java/.../importer/aesop/eob3/PlaceholderAppearancePack.java`
- Test: `beholder-importer-aesop/src/test/java/.../importer/aesop/eob3/PlaceholderAppearancePackTest.java`

**Interfaces:**
- Consumes: `AppearancePack`, `TileSet`, `PixelSheet`, `DecorationTable` (`beholder-ir`).
- Produces: `PlaceholderAppearancePack.build(String id) -> AppearancePack` — a valid pack with one wall-set (so wall-set index 1 renders), empty decorations, a 1×1 door leaf. Constant `PlaceholderAppearancePack.STANDARD_WALL_SET = 1`.

Reference for valid flat data: `beholder-ir-json/src/test/.../SyntheticAppearancePacks.java` (do not depend on it — it is test-scope; reproduce the shape).

- [ ] **Step 1: Failing `PlaceholderAppearancePackTest`.**
```java
package eu.virtualparadox.beholder.importer.aesop.eob3;

import static org.assertj.core.api.Assertions.assertThat;

import eu.virtualparadox.beholder.ir.model.AppearancePack;
import org.junit.jupiter.api.Test;

class PlaceholderAppearancePackTest {

    PlaceholderAppearancePackTest() {}

    @Test
    void buildsAValidFlatPackWithOneWallSetAndNoDecorations() {
        final AppearancePack pack = PlaceholderAppearancePack.build("aesop-placeholder");
        assertThat(pack.id()).isEqualTo("aesop-placeholder");
        assertThat(pack.tileSet().wallSetCount()).isGreaterThanOrEqualTo(PlaceholderAppearancePack.STANDARD_WALL_SET);
        assertThat(pack.tileSet().backgroundCodes()).hasSize(330);
        assertThat(pack.tileSet().wallSetCodes(PlaceholderAppearancePack.STANDARD_WALL_SET)).hasSize(431);
        assertThat(pack.decorationOverlayIds()).isEmpty();
        assertThat(pack.tileSet().paletteArgb()).hasSize(256);
    }
}
```

- [ ] **Step 2: Run — expect FAIL.**

- [ ] **Step 3: Implement `PlaceholderAppearancePack`** (valid flat `TileSet` with a visible palette so the placeholder walls are not black; empty decorations; minimal door leaf):
```java
package eu.virtualparadox.beholder.importer.aesop.eob3;

import eu.virtualparadox.beholder.ir.model.AppearancePack;
import eu.virtualparadox.beholder.ir.model.DecorationTable;
import eu.virtualparadox.beholder.ir.model.PixelSheet;
import eu.virtualparadox.beholder.ir.model.TileSet;
import java.util.Map;

/**
 * Builds a self-contained, flat placeholder {@link AppearancePack} for EOB3 M1 — valid data that
 * renders simple (non-black) walls through the existing projection, needing no game assets. The
 * real EOB3 whole-sprite render (decoded {@code 1.10} VFX walls + palette) is M2; this exists only
 * to make an EOB3 level walkable and visible from the shared IR now.
 */
public final class PlaceholderAppearancePack {

    /** The single wall-set index a SOLID edge uses (the pack carries exactly one wall-set block). */
    public static final int STANDARD_WALL_SET = 1;

    /** Nibble values per 8×8 VCN tile. */
    private static final int TILE_NIBBLE_COUNT = 64;
    /** Entries per 16-colour sub-palette. */
    private static final int SUB_PALETTE_SIZE = 16;
    /** Background code count (22×15 viewport). */
    private static final int BACKGROUND_CODE_COUNT = 330;
    /** Codes per wall-set block. */
    private static final int CODES_PER_WALL_SET = 431;
    /** ARGB palette entry count. */
    private static final int PALETTE_COLOR_COUNT = 256;
    /** Opaque-alpha mask for a synthetic ARGB palette. */
    private static final int OPAQUE = 0xFF000000;

    private PlaceholderAppearancePack() {}

    /**
     * Builds the placeholder pack.
     *
     * @param id the appearance-pack id the level document will reference; must be non-blank
     * @return a valid flat {@link AppearancePack} with one wall-set and no decorations
     */
    public static AppearancePack build(final String id) {
        return AppearancePack.of(id, tileSet(), emptyDecorationTable(), Map.of(), PixelSheet.of(1, 1, new byte[1]));
    }

    private static TileSet tileSet() {
        final byte[] tile = new byte[TILE_NIBBLE_COUNT];
        for (int i = 0; i < TILE_NIBBLE_COUNT; i++) {
            tile[i] = (byte) (i % SUB_PALETTE_SIZE);
        }
        final int[] backdropSub = new int[SUB_PALETTE_SIZE];
        final int[] wallSub = new int[SUB_PALETTE_SIZE];
        for (int i = 0; i < SUB_PALETTE_SIZE; i++) {
            backdropSub[i] = i;
            wallSub[i] = i;
        }
        final int[] background = new int[BACKGROUND_CODE_COUNT];
        final int[] wallSet = new int[CODES_PER_WALL_SET];
        final int[] palette = new int[PALETTE_COLOR_COUNT];
        for (int i = 0; i < PALETTE_COLOR_COUNT; i++) {
            palette[i] = OPAQUE | (i << 16) | (i << 8) | i; // grey ramp: visible, not black
        }
        return TileSet.of(new byte[][] {tile}, new int[][] {backdropSub, wallSub}, new int[][] {background, wallSet}, palette);
    }

    private static DecorationTable emptyDecorationTable() {
        return DecorationTable.of(0, new int[][] {{}, {}, {}}, new int[][] {{}, {}}, new DecorationTable.Rect[0]);
    }
}
```
*(If `TileSet.of`/`DecorationTable.of`/`PixelSheet.of` reject any of the minimal inputs at construction, the test fails loudly — adjust to the exact invariants in `beholder-ir`, which the spec records: tiles rows length 64, sub-palettes 2×16, codeBlocks row0=330 + wall rows=431, palette=256, decorationCount 0 legal.)*

- [ ] **Step 4: Run — expect PASS.**

- [ ] **Step 5: Format + commit** (`git commit -m "feat(aesop): PlaceholderAppearancePack flat self-contained pack"`).

---

## Task 6: `Eob3Extractor` + synthetic-EYE.RES test folder

**Files:**
- Modify: `beholder-app/pom.xml` (add dependency on `beholder-importer-aesop`)
- Create: `beholder-app/src/main/java/.../app/Eob3Extractor.java`
- Modify (test helper): `beholder-app/src/test/java/.../app/SyntheticGameFolders.java` (add `writeEob3Assets`)
- Test: `beholder-app/src/test/java/.../app/Eob3ExtractorTest.java`

**Interfaces:**
- Consumes: `ResArchive`, `Roed`, `OccupancyMaze`, `AesopToIr`, `PlaceholderAppearancePack` (importer-aesop); `IrDocument`, `LevelDocument`, `LevelEvents`, `LevelGeometry`, `WallMappingTable` (ir/ir-json); `GameExtractor` (app).
- Produces: `Eob3Extractor implements GameExtractor` — `extract(Path gameFolder, String level) -> IrDocument`, reading `gameFolder/EYE.RES`, resolving `level` (a map name, e.g. `"Mausoleum 1"`) via `Roed`, decoding the `OccupancyMaze`, and assembling an `IrDocument` with empty events + the placeholder pack. `SyntheticGameFolders.writeEob3Assets(Path, String levelName, int mapNumber)` writes a minimal `EYE.RES` the extractor can read.

- [ ] **Step 1: Add the module dependency to `beholder-app/pom.xml`** (next to the existing `beholder-importer-westwood` dependency):
```xml
        <dependency>
            <groupId>eu.virtualparadox.beholder</groupId>
            <artifactId>beholder-importer-aesop</artifactId>
            <version>0.1.0-SNAPSHOT</version>
        </dependency>
```

- [ ] **Step 2: Write the failing `Eob3ExtractorTest`** (drives a synthetic `EYE.RES`):
```java
package eu.virtualparadox.beholder.app;

import static org.assertj.core.api.Assertions.assertThat;

import eu.virtualparadox.beholder.ir.json.IrDocument;
import eu.virtualparadox.beholder.ir.model.LevelDocument;
import java.io.IOException;
import java.nio.file.Path;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

class Eob3ExtractorTest {

    Eob3ExtractorTest() {}

    @Test
    void extractsAWalkableEob3LevelFromASyntheticEyeRes(@TempDir final Path folder) throws IOException {
        SyntheticGameFolders.writeEob3Assets(folder, "Mausoleum 1", 5);
        final IrDocument doc = new Eob3Extractor().extract(folder, "Mausoleum 1");
        final LevelDocument level = doc.level();
        assertThat(level.geometry().width()).isEqualTo(32);
        assertThat(level.appearancePackId()).isEqualTo(doc.pack().id());
        assertThat(level.events().triggers()).isEmpty();
        assertThat(level.events().doors()).isEmpty();
        assertThat(level.events().decorations()).isEmpty();
    }
}
```

- [ ] **Step 3: Add `writeEob3Assets` to `SyntheticGameFolders`** (writes an `EYE.RES` with a ROED naming the level + a map grid):
```java
    /**
     * Writes a minimal EOB3 {@code EYE.RES} into {@code folder}: a ROED (resource 0) naming {@code
     * levelName} → {@code mapNumber}, and a 1024-byte occupancy grid at {@code mapNumber} (a single
     * open cell so geometry is non-trivial). Uses the importer's own {@code SyntheticRes} builder.
     *
     * @param folder the game folder to write into
     * @param levelName the map resource name the extractor resolves
     * @param mapNumber the resource number the map grid is stored at
     * @throws IOException if the file cannot be written
     */
    static void writeEob3Assets(final Path folder, final String levelName, final int mapNumber) throws IOException {
        final boolean[][] open = new boolean[32][32];
        open[16][16] = true;
        open[16][17] = true;
        final byte[] roed = eu.virtualparadox.beholder.importer.aesop.SyntheticRes.roed(java.util.Map.of(levelName, mapNumber));
        final byte[] grid = eu.virtualparadox.beholder.importer.aesop.SyntheticRes.occupancy(open);
        final byte[] eyeRes = eu.virtualparadox.beholder.importer.aesop.SyntheticRes.of(java.util.Map.of(0, roed, mapNumber, grid));
        java.nio.file.Files.write(folder.resolve("EYE.RES"), eyeRes);
    }
```
*(`SyntheticRes` is already a **public main-scope** class in `beholder-importer-aesop` (Task 1), so `beholder-app`'s test classpath sees it via the normal compile dependency added in Step 1 — no test-jar needed. The fully-qualified references above resolve directly.)*

- [ ] **Step 4: Run — expect FAIL** (`Eob3Extractor` undefined).

- [ ] **Step 5: Implement `Eob3Extractor`** (mirror `Eob1Extractor`'s shape):
```java
package eu.virtualparadox.beholder.app;

import eu.virtualparadox.beholder.importer.aesop.ResArchive;
import eu.virtualparadox.beholder.importer.aesop.Roed;
import eu.virtualparadox.beholder.importer.aesop.eob3.AesopToIr;
import eu.virtualparadox.beholder.importer.aesop.eob3.OccupancyMaze;
import eu.virtualparadox.beholder.importer.aesop.eob3.PlaceholderAppearancePack;
import eu.virtualparadox.beholder.ir.event.LevelEvents;
import eu.virtualparadox.beholder.ir.json.IrDocument;
import eu.virtualparadox.beholder.ir.model.AppearancePack;
import eu.virtualparadox.beholder.ir.model.LevelDocument;
import eu.virtualparadox.beholder.ir.model.LevelGeometry;
import eu.virtualparadox.beholder.ir.model.WallMappingTable;
import java.io.IOException;
import java.io.UncheckedIOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;

/**
 * {@link GameExtractor} for Eye of the Beholder III (P3 M1): reads the AESOP {@code EYE.RES}
 * container from a game folder, resolves the requested level's map resource by name via the ROED
 * dictionary, lifts its 32×32 occupancy grid into the shared per-edge {@link LevelGeometry}, and
 * assembles a walkable base-level {@link IrDocument} with empty events and a self-contained
 * placeholder appearance pack. The real EOB3 whole-sprite render is M2; the AESOP event lift is M3.
 */
public final class Eob3Extractor implements GameExtractor {

    /** The AESOP resource container file inside an EOB3 game folder. */
    private static final String RES_ARCHIVE = "EYE.RES";
    /** Resource number of the ROED (name → number) dictionary. */
    private static final int ROED_RESOURCE = 0;
    /** The appearance-pack id the placeholder pack and level document share. */
    private static final String PLACEHOLDER_PACK_ID = "aesop-placeholder";

    /** Creates the extractor. */
    public Eob3Extractor() {}

    /**
     * Extracts an EOB3 level's base IR document from an {@code EYE.RES} install.
     *
     * @param gameFolder the EOB3 install's data folder, holding {@value #RES_ARCHIVE}
     * @param level the map resource name (e.g. {@code "Mausoleum 1"}) to lift
     * @return the freshly-decoded base-level IR document (walkable geometry, empty events,
     *     placeholder pack)
     * @throws UncheckedIOException if {@value #RES_ARCHIVE} cannot be read
     * @throws IllegalArgumentException if {@code level} names no map resource, or the resource is
     *     not a valid 32×32 grid
     */
    @Override
    public IrDocument extract(final Path gameFolder, final String level) {
        final ResArchive archive = readArchive(gameFolder);
        final Roed roed = Roed.parse(archive.resource(ROED_RESOURCE));
        final OccupancyMaze maze = OccupancyMaze.parse(archive.resource(roed.number(level)));
        final AppearancePack pack = PlaceholderAppearancePack.build(PLACEHOLDER_PACK_ID);
        final LevelGeometry geometry = AesopToIr.geometry(maze, PlaceholderAppearancePack.STANDARD_WALL_SET);
        final LevelEvents events = new LevelEvents(List.of(), List.of(), List.of());
        final LevelDocument document =
                new LevelDocument(LevelGeometry.CURRENT_IR_VERSION, geometry, events, pack.id(), WallMappingTable.empty());
        return new IrDocument(document, pack);
    }

    private static ResArchive readArchive(final Path gameFolder) {
        try {
            return ResArchive.parse(Files.readAllBytes(gameFolder.resolve(RES_ARCHIVE)));
        } catch (final IOException e) {
            throw new UncheckedIOException("Eob3Extractor could not read " + RES_ARCHIVE + " from " + gameFolder, e);
        }
    }
}
```

- [ ] **Step 6: Run — expect PASS** (`mvn -q -pl beholder-app -am test`; `-am` rebuilds the new upstream module).

- [ ] **Step 7: Format + commit.**
```bash
mvn -q -pl beholder-importer-aesop,beholder-app spotless:apply
git add beholder-app beholder-importer-aesop
git commit -m "feat(aesop): Eob3Extractor + synthetic EYE.RES test folder"
```

---

## Task 7: Wire EOB3 into detection + extraction dispatch

**Files:**
- Modify: `beholder-app/src/main/java/.../app/GameKind.java`, `GameDetector.java`, `Extract.java`
- Modify: `beholder-app/src/test/java/.../app/GameDetectorTest.java`, `ExtractTest.java`

**Interfaces:**
- Consumes: `Eob3Extractor` (Task 6), `SyntheticGameFolders.writeEob3Assets` (Task 6).
- Produces: `GameKind.EOB3`; `GameDetector.detect` returns `EOB3` when `EYE.RES` is present; `Extract` routes `EOB3` to `Eob3Extractor`.

- [ ] **Step 1: Add `GameKind.EOB3`.**
```java
    /** Eye of the Beholder III: the AESOP {@code EYE.RES} container (see {@link Eob3Extractor}). */
    EOB3
```
(Add after `EOB2`, with a trailing comma on `EOB2`.)

- [ ] **Step 2: Write the failing `GameDetectorTest` case.**
```java
    @Test
    void detectsEob3ByEyeResMarker(@TempDir final Path folder) throws IOException {
        Files.write(folder.resolve("EYE.RES"), new byte[] {0});
        assertThat(GameDetector.detect(folder)).isEqualTo(GameKind.EOB3);
    }
```

- [ ] **Step 3: Run — expect FAIL** (detect returns EOB2 or throws).

- [ ] **Step 4: Add the EOB3 branch to `GameDetector.detect`** (before the EOB2 `LEVEL1.INF` check; keep the single-`return` style):
```java
    /** The AESOP container filename that marks an EOB3 install. */
    private static final String EOB3_MARKER_FILE = "EYE.RES";
```
and in `detect`:
```java
        final GameKind kind;
        if (hasEob1Archive(gameFolder)) {
            kind = GameKind.EOB1;
        } else if (Files.isRegularFile(gameFolder.resolve(EOB3_MARKER_FILE))) {
            kind = GameKind.EOB3;
        } else if (Files.isRegularFile(gameFolder.resolve(EOB2_MARKER_FILE))) {
            kind = GameKind.EOB2;
        } else {
            throw new IllegalArgumentException("GameDetector: no known game data (EOBDATA*.PAK, "
                    + EOB3_MARKER_FILE + ", or " + EOB2_MARKER_FILE + ") found in " + gameFolder);
        }
        return kind;
```

- [ ] **Step 5: Add the EOB3 entry to `Extract.EXTRACTORS`.**
```java
    private static final Map<GameKind, GameExtractor> EXTRACTORS = Map.of(
            GameKind.EOB1, new Eob1Extractor(),
            GameKind.EOB2, new Eob2Extractor(),
            GameKind.EOB3, new Eob3Extractor());
```

- [ ] **Step 6: Add the `ExtractTest` EOB3 round-trip case** (mirrors the EOB1/EOB2 cases — extract via facade equals direct extract):
```java
    @Test
    void extractsEob3ThroughTheFacadeMatchingDirectExtraction(@TempDir final Path folder, @TempDir final Path irOut)
            throws IOException {
        SyntheticGameFolders.writeEob3Assets(folder, "Mausoleum 1", 5);
        Extract.extract(folder, "Mausoleum 1", irOut, "mausoleum1");
        final IrDocument reloaded = IrJson.readDocument(irOut, "mausoleum1");
        final IrDocument direct = new Eob3Extractor().extract(folder, "Mausoleum 1");
        assertThat(reloaded).isEqualTo(direct);
    }
```

- [ ] **Step 7: Run — expect PASS** (`mvn -q -pl beholder-app -am test`).

- [ ] **Step 8: Format + commit** (`git commit -m "feat(app): detect + dispatch EOB3 (GameKind.EOB3, EYE.RES marker)"`).

---

## Task 8: Real-data skip-safe ITs (walkable + lossless round-trip)

**Files:**
- Create: `beholder-render-libgdx/src/test/java/.../render/libgdx/RealEob3EyeRes.java`
- Create: `beholder-render-libgdx/src/test/java/.../render/libgdx/Eob3WalkableIT.java`
- Create: `beholder-render-libgdx/src/test/java/.../render/libgdx/Eob3LosslessRoundTripIT.java`
- Modify: `beholder-render-libgdx/pom.xml` if it does not already depend on `beholder-importer-aesop`/`beholder-app` for the extractor (it already depends on `beholder-app`; confirm and add `beholder-importer-aesop` only if the IT references importer types directly — it should go through `Eob3Extractor` to stay at the app boundary).

**Interfaces:**
- Consumes: `Eob3Extractor` (via `beholder-app`), `IrJson`, `Boot`/`FirstPersonRenderer`/`RuntimeState`/`MovementCommand`, `ContactSheet` (test tree).
- Produces: skip-safe ITs asserting EOB3 walkability invariants + IR round-trip byte-stability/render-identity against the maintainer's real `EYE.RES`.

- [ ] **Step 1: `RealEob3EyeRes` skip-safe locator** (mirror `RealEob2Level1`):
```java
package eu.virtualparadox.beholder.render.libgdx;

import static org.junit.jupiter.api.Assumptions.assumeTrue;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

/** Skip-safe locator for the maintainer's real EOB3 install (never commits game data). */
final class RealEob3EyeRes {

    /** The EOB3 game-data folder (the maintainer's own install; absent on CI → tests skip). */
    private static final Path GAME = Path.of(System.getProperty("user.home"),
            "Games/Eye of the Beholder III Assault on Myth Drannor.app/Contents/Resources/game");

    /** The map resource name M1 renders. */
    static final String LEVEL = "Mausoleum 1";

    private RealEob3EyeRes() {}

    /**
     * Returns the readable EOB3 game folder, skipping the calling test when it is absent.
     *
     * @return the game folder path
     */
    static Path assumeReadable() {
        assumeTrue(Files.isReadable(GAME.resolve("EYE.RES")), "EOB3 game data present at " + GAME);
        return GAME;
    }
}
```

- [ ] **Step 2: `Eob3WalkableIT`** (structural invariants, mirror `WalkableLevelIT`):
```java
package eu.virtualparadox.beholder.render.libgdx;

import static org.assertj.core.api.Assertions.assertThat;

import eu.virtualparadox.beholder.app.Eob3Extractor;
import eu.virtualparadox.beholder.engine.MovementCommand;
import eu.virtualparadox.beholder.engine.RuntimeState;
import eu.virtualparadox.beholder.engine.model.Level;
import eu.virtualparadox.beholder.engine.model.Party;
import eu.virtualparadox.beholder.ir.Facing;
import eu.virtualparadox.beholder.ir.GridPosition;
import eu.virtualparadox.beholder.ir.json.IrDocument;
import java.nio.file.Path;
import org.junit.jupiter.api.Test;

class Eob3WalkableIT {

    Eob3WalkableIT() {}

    @Test
    void fourRightTurnsRestoreTheOriginalFacing() {
        final Path folder = RealEob3EyeRes.assumeReadable();
        final Level level = buildLevel(folder);
        final RuntimeState state = RuntimeState.startingAt(Party.of(GridPosition.of(16, 16), Facing.NORTH));
        for (int i = 0; i < 4; i++) {
            state.applyCommand(MovementCommand.TURN_RIGHT, level);
        }
        assertThat(state.party().facing()).isEqualTo(Facing.NORTH);
        assertThat(state.party().position()).isEqualTo(GridPosition.of(16, 16));
    }

    @Test
    void walkingForwardStaysOnTheGrid() {
        final Path folder = RealEob3EyeRes.assumeReadable();
        final Level level = buildLevel(folder);
        final RuntimeState state = RuntimeState.startingAt(Party.of(GridPosition.of(16, 16), Facing.NORTH));
        for (int i = 0; i < 40; i++) {
            state.applyCommand(MovementCommand.FORWARD, level);
            assertThat(state.party().position().x()).isBetween(0, GridPosition.GRID_SIZE - 1);
            assertThat(state.party().position().y()).isBetween(0, GridPosition.GRID_SIZE - 1);
        }
    }

    private static Level buildLevel(final Path folder) {
        final IrDocument doc = new Eob3Extractor().extract(folder, RealEob3EyeRes.LEVEL);
        return Level.of(doc.level().geometry(), doc.pack().tileSet(), doc.level().events());
    }
}
```
*(Starting cell `(16,16)` is a documented central fallback; if it is solid in the real Mausoleum, pick any open cell — a helper that scans `doc.level().geometry()` for an OPEN-bounded cell is acceptable, but the two invariants above hold regardless of open/solid start since collision simply blocks a forward step into a wall.)*

- [ ] **Step 3: `Eob3LosslessRoundTripIT`** (mirror `LosslessRoundTripIT` legs a+b: value-equal + byte-stable — this is the milestone's IR-thesis check that the codec is game-neutral for EOB3):
```java
package eu.virtualparadox.beholder.render.libgdx;

import static org.assertj.core.api.Assertions.assertThat;

import eu.virtualparadox.beholder.app.Eob3Extractor;
import eu.virtualparadox.beholder.ir.json.IrDocument;
import eu.virtualparadox.beholder.ir.json.IrJson;
import java.io.IOException;
import java.nio.file.Path;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

class Eob3LosslessRoundTripIT {

    Eob3LosslessRoundTripIT() {}

    @Test
    void realEob3LevelRoundTripsLosslessly(@TempDir final Path dir) throws IOException {
        final Path folder = RealEob3EyeRes.assumeReadable();
        final IrDocument irA = new Eob3Extractor().extract(folder, RealEob3EyeRes.LEVEL);
        IrJson.writeDocument(dir, "mausoleum1", irA);
        final IrDocument irB = IrJson.readDocument(dir, "mausoleum1");
        assertThat(irB).isEqualTo(irA); // leg (a): whole-IR value equality after a disk round-trip
    }
}
```

- [ ] **Step 4: Run the ITs** (they run if the maintainer's `EYE.RES` is present, else skip):
```bash
export JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home
mvn -q -pl beholder-render-libgdx -am verify
```
Expected: the three EOB3 ITs pass (or skip cleanly if the data is absent). No game data or PNG committed.

- [ ] **Step 5: Full root verify + commit.**
```bash
mvn -q verify   # root reactor: all gates (spotless/checkstyle/PMD/SpotBugs/forbiddenapis/JaCoCo ≥90%)
git add beholder-render-libgdx
git commit -m "test(aesop): skip-safe EOB3 walkable + lossless round-trip ITs on real EYE.RES"
```

---

## Final verification (whole milestone)

- [ ] `export JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home && mvn spotless:apply && mvn verify` — full reactor green (all gates, ≥90% branch+line coverage).
- [ ] Manual walk (maintainer eyeball): `mvn -q -pl beholder-app -am exec:java` style `extract` + `play`, or the project's `/run` path — extract `"Mausoleum 1"` from the real `EYE.RES`, then `play` and walk the placeholder-walled Mausoleum. (A third game walkable from the shared IR.)
- [ ] Update `.superpowers/sdd/progress.md` with the M1 branch + per-task outcomes as the work proceeds.

## Self-Review notes (author)
- **Spec coverage:** every §3 component (ResArchive/Roed/OccupancyMaze/AesopToIr/PlaceholderPack/Eob3Extractor/GameKind/GameDetector/Extract) has a task; §5 tests (unit + skip-safe ITs + round-trip) are Tasks 1–8; §3.3 occupancy→edge rule is Task 4's truth table.
- **Out-of-scope confirmed absent:** no `1.10` VFX / `PAL` decoder, no EOB3-native render, no AESOP VM, no event lift — all deferred to M2/M3 per the maintainer's milestone-boundary choice.
- **`SyntheticRes` scope (resolved):** a public main-scope test-support builder in `beholder-importer-aesop` (Task 1), reused by this module's `eob3` tests and `beholder-app`'s tests via the normal compile dependency — avoids a test-jar and a CPD-duplicated `.RES` builder. A Maven test-jar is the alternative if the maintainer prefers keeping it out of main scope.
- **Type consistency check:** decoder factories (`ResArchive.parse`/`Roed.parse`/`OccupancyMaze.parse`), `AesopToIr.geometry(maze, wallSet)`, `PlaceholderAppearancePack.build(id)`/`STANDARD_WALL_SET`, and `Eob3Extractor.extract(folder, level)` names/signatures are used identically across Tasks 4/6/7/8 — verified consistent.
