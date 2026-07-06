# P0 — EOB1 Westwood PAK Extractor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `beholder-importer-westwood` module with a `PakArchive` reader that lists and extracts files from a Westwood `.PAK` archive, unblocking access to EOB1's packed data (`EOBDATA*.PAK`) for the P1 vertical slice.

**Architecture:** A single-responsibility, framework-neutral binary reader. `PakArchive.parse(byte[])` decodes the directory and returns an immutable name→bytes view. No IR dependency yet (pure bytes → files). Unit-tested with a synthetic in-test archive (deterministic, no copyrighted data); an optional integration test validates against the real `EOBDATA1.PAK` if present.

**Tech Stack:** Java 21, `eu.virtualparadox:parent:1.0.0`, JUnit Jupiter + AssertJ. **Build on JDK 21** (`JAVA_HOME` → Corretto 21; the reactor enforces `[21,22)` fail-fast — see `AGENTS.md`).

**PAK format** — ⚠️ **SUPERSEDED (see §Execution outcome):** the sentinel-based description below was found WRONG during execution and corrected to ScummVM kyra's **position-based interleaved** layout (`off0, name0, off1, …, nameN-1, offN` — N+1 offsets, N names — end-of-directory at `pos+4 == firstOffset`, NO trailing sentinel). The committed `PakArchive` code is authoritative. *Original (incorrect) notes:* the file begins with a directory of entries, each `{ uint32 offset (little-endian, absolute file position of the entry's data); NUL-terminated ASCII filename }`, laid out contiguously from byte 0. The **first entry's offset** equals the total directory length (= where file data begins), so the directory is exactly the entries whose position is `< firstOffset`. The directory ends with a **sentinel entry whose offset == the archive length** (its name is empty); file *i*'s bytes span `[offset[i], offset[i+1])`, the sentinel providing the final upper bound. (Verified: `EOBDATA1.PAK` starts `07 01 00 00` → firstOffset `0x107`, name `ADLIB.DAT`, then `0x1E48 PCSOUND.DAT`, `0x260E PCGAMESN.DAT`, `0x38A3 ADVENTUR.CMP`, …) Note: `EYE.PAK` uses a different/variant (likely compressed) header — OUT OF SCOPE here; this reader targets the `EOBDATA*.PAK` layout only and must fail cleanly (throw) on input it cannot parse.

## Global Constraints

All constraints from `CODING_STANDARDS.md` apply (see the skeleton plan `2026-07-06-p0-project-skeleton.md` for the full verbatim list). The load-bearing ones for this module:

- `<parent>` = the reactor root `eu.virtualparadox.beholder:legend-of-the-beholder:0.1.0-SNAPSHOT`; base package `eu.virtualparadox.beholder.importer.westwood`.
- Palantir format (`spotless:apply`); max line 140; one statement/line; braces on control flow; uppercase `L` longs.
- Method params `final`; locals `final` where applicable; no param reassignment; exactly one `return` per method; no `instanceof`; no reference-equality; cyclomatic ≤8/method; ≤12 fields, ≤18 methods, ≤6 params per class.
- No `return null;` — throw or `Optional`/immutable-empty. NullAway ERROR for `eu.virtualparadox`.
- No `System.out`/`err`; no `printStackTrace`; no `Thread.sleep` (incl. tests).
- Javadoc on public/protected types+methods with `@param`/`@return`/`@throws`; doclint=all.
- JaCoCo ≥90% branch AND line; keep classes out of `**/model/**` / `*Dto` / `*Record` naming so they ARE measured.
- Unit tests `*Test.java`; integration tests `*IT.java` (Failsafe). JUnit Jupiter + AssertJ.
- **NEVER commit game data.** Synthetic test bytes only in unit tests; the `*IT` reads the real PAK from `~/Games/...` if present and is skipped otherwise; real unpacked output goes to scratch, never git.

## Execution note (workflow)

Work on branch `feat/p0-pak-extractor`; commit per task; `--no-ff` merge to `main` at the end (final step of the last task), then push. Build every `mvn` with `JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home`.

## File Structure

```
legend-of-the-beholder/
├── pom.xml                                  MODIFY — add <module>beholder-importer-westwood</module>
└── beholder-importer-westwood/              CREATE
    ├── pom.xml                              CREATE — module POM (parent = reactor root; test deps)
    └── src/
        ├── main/java/eu/virtualparadox/beholder/importer/westwood/
        │   └── PakArchive.java              CREATE (Task 2)
        └── test/java/eu/virtualparadox/beholder/importer/westwood/
            ├── PakArchiveTest.java          CREATE (Task 2) — synthetic archive, full-branch
            └── PakArchiveIT.java            CREATE (Task 3) — real EOBDATA1.PAK, skip-if-absent
```

---

### Task 1: `beholder-importer-westwood` module

**Files:**
- Modify: `pom.xml` (reactor root — add the module)
- Create: `beholder-importer-westwood/pom.xml`
- Create: the `src/main` and `src/test` package directories

**Interfaces:**
- Consumes: the reactor root POM.
- Produces: module `beholder-importer-westwood`, base package `eu.virtualparadox.beholder.importer.westwood`.

- [ ] **Step 1: Create the branch**

Run:
```bash
git -C ~/Workspaces/Java/legend-of-the-beholder checkout -b feat/p0-pak-extractor
```
Expected: `Switched to a new branch 'feat/p0-pak-extractor'`.

- [ ] **Step 2: Register the module in the reactor root `pom.xml`**

In `~/Workspaces/Java/legend-of-the-beholder/pom.xml`, change the `<modules>` block from:
```xml
    <modules>
        <module>beholder-ir</module>
    </modules>
```
to:
```xml
    <modules>
        <module>beholder-ir</module>
        <module>beholder-importer-westwood</module>
    </modules>
```

- [ ] **Step 3: Write the module `pom.xml`**

Create `~/Workspaces/Java/legend-of-the-beholder/beholder-importer-westwood/pom.xml`:
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

    <artifactId>beholder-importer-westwood</artifactId>

    <name>Beholder Importer: Westwood</name>
    <description>Anti-corruption importer for Westwood EOB1/EOB2 original formats into the canonical IR.</description>

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

- [ ] **Step 4: Create the package directories**

Run:
```bash
mkdir -p ~/Workspaces/Java/legend-of-the-beholder/beholder-importer-westwood/src/main/java/eu/virtualparadox/beholder/importer/westwood
mkdir -p ~/Workspaces/Java/legend-of-the-beholder/beholder-importer-westwood/src/test/java/eu/virtualparadox/beholder/importer/westwood
```

- [ ] **Step 5: Verify the reactor still validates (JDK 21)**

Run:
```bash
JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home mvn -q -f ~/Workspaces/Java/legend-of-the-beholder/pom.xml validate
```
Expected: BUILD SUCCESS; reactor now lists three projects (`legend-of-the-beholder`, `beholder-ir`, `beholder-importer-westwood`).

- [ ] **Step 6: Commit**

```bash
git -C ~/Workspaces/Java/legend-of-the-beholder add pom.xml beholder-importer-westwood/pom.xml
git -C ~/Workspaces/Java/legend-of-the-beholder commit -m "build: add beholder-importer-westwood module"
```

---

### Task 2: `PakArchive` reader (TDD, synthetic archive)

**Files:**
- Create: `beholder-importer-westwood/src/main/java/eu/virtualparadox/beholder/importer/westwood/PakArchive.java`
- Test: `beholder-importer-westwood/src/test/java/eu/virtualparadox/beholder/importer/westwood/PakArchiveTest.java`

**Interfaces:**
- Produces:
  - `PakArchive.parse(byte[] data)` → `PakArchive` (throws `IllegalArgumentException` on malformed input).
  - `PakArchive#names()` → `List<String>` (immutable, directory order).
  - `PakArchive#contains(String name)` → `boolean`.
  - `PakArchive#file(String name)` → `byte[]` (a copy; throws `IllegalArgumentException` if absent).
  - `PakArchive#size()` → `int` (number of files, excluding the sentinel).

- [ ] **Step 1: Write the failing test**

Create `.../src/test/java/eu/virtualparadox/beholder/importer/westwood/PakArchiveTest.java`:
```java
package eu.virtualparadox.beholder.importer.westwood;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.io.ByteArrayOutputStream;
import java.nio.charset.StandardCharsets;
import org.junit.jupiter.api.Test;

class PakArchiveTest {

    /**
     * Builds a minimal Westwood-style PAK holding the given name→content pairs:
     * a directory of {uint32 LE offset, NUL-terminated name} entries plus a
     * trailing empty-name sentinel whose offset is the total length, then the
     * concatenated file data.
     */
    private static byte[] buildPak(final String[] names, final byte[][] contents) {
        int dirLen = 4 + 1; // sentinel: uint32 + empty-name NUL
        for (final String name : names) {
            dirLen += 4 + name.getBytes(StandardCharsets.US_ASCII).length + 1;
        }
        final int[] offsets = new int[names.length + 1];
        int running = dirLen;
        for (int i = 0; i < names.length; i++) {
            offsets[i] = running;
            running += contents[i].length;
        }
        offsets[names.length] = running; // sentinel offset = total length
        final ByteArrayOutputStream out = new ByteArrayOutputStream();
        for (int i = 0; i < names.length; i++) {
            writeU32(out, offsets[i]);
            out.writeBytes(names[i].getBytes(StandardCharsets.US_ASCII));
            out.write(0);
        }
        writeU32(out, offsets[names.length]);
        out.write(0); // sentinel empty name
        for (final byte[] content : contents) {
            out.writeBytes(content);
        }
        return out.toByteArray();
    }

    private static void writeU32(final ByteArrayOutputStream out, final int value) {
        out.write(value & 0xFF);
        out.write((value >>> 8) & 0xFF);
        out.write((value >>> 16) & 0xFF);
        out.write((value >>> 24) & 0xFF);
    }

    @Test
    void parseListsNamesInDirectoryOrder() {
        final byte[] pak = buildPak(
                new String[] {"A.DAT", "B.DAT"},
                new byte[][] {"hello".getBytes(StandardCharsets.US_ASCII), "world!".getBytes(StandardCharsets.US_ASCII)});
        final PakArchive archive = PakArchive.parse(pak);
        assertThat(archive.size()).isEqualTo(2);
        assertThat(archive.names()).containsExactly("A.DAT", "B.DAT");
    }

    @Test
    void fileReturnsExactContentBytes() {
        final byte[] pak = buildPak(
                new String[] {"A.DAT", "B.DAT"},
                new byte[][] {"hello".getBytes(StandardCharsets.US_ASCII), "world!".getBytes(StandardCharsets.US_ASCII)});
        final PakArchive archive = PakArchive.parse(pak);
        assertThat(archive.file("A.DAT")).isEqualTo("hello".getBytes(StandardCharsets.US_ASCII));
        assertThat(archive.file("B.DAT")).isEqualTo("world!".getBytes(StandardCharsets.US_ASCII));
    }

    @Test
    void containsReflectsMembership() {
        final byte[] pak = buildPak(
                new String[] {"A.DAT"}, new byte[][] {"x".getBytes(StandardCharsets.US_ASCII)});
        final PakArchive archive = PakArchive.parse(pak);
        assertThat(archive.contains("A.DAT")).isTrue();
        assertThat(archive.contains("MISSING.DAT")).isFalse();
    }

    @Test
    void fileRejectsUnknownName() {
        final byte[] pak = buildPak(
                new String[] {"A.DAT"}, new byte[][] {"x".getBytes(StandardCharsets.US_ASCII)});
        final PakArchive archive = PakArchive.parse(pak);
        assertThatThrownBy(() -> archive.file("MISSING.DAT")).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void parseRejectsTruncatedData() {
        assertThatThrownBy(() -> PakArchive.parse(new byte[] {0x02})).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void fileReturnsADefensiveCopy() {
        final byte[] pak = buildPak(
                new String[] {"A.DAT"}, new byte[][] {"xy".getBytes(StandardCharsets.US_ASCII)});
        final PakArchive archive = PakArchive.parse(pak);
        final byte[] first = archive.file("A.DAT");
        first[0] = 0x7A;
        assertThat(archive.file("A.DAT")).isEqualTo("xy".getBytes(StandardCharsets.US_ASCII));
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run:
```bash
JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home mvn -q -pl beholder-importer-westwood -f ~/Workspaces/Java/legend-of-the-beholder/pom.xml test
```
Expected: FAIL — `PakArchive` does not exist (compilation error).

- [ ] **Step 3: Write the implementation**

Create `.../src/main/java/eu/virtualparadox/beholder/importer/westwood/PakArchive.java`:
```java
package eu.virtualparadox.beholder.importer.westwood;

import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

/**
 * A read-only view over a Westwood {@code .PAK} archive (the {@code EOBDATA*.PAK}
 * layout used by Eye of the Beholder I).
 *
 * <p>The archive begins with a directory of {@code {uint32 little-endian offset;
 * NUL-terminated name}} entries laid out from byte 0; the first entry's offset is
 * where file data starts, and a trailing empty-name sentinel whose offset equals
 * the archive length bounds the last file. This reader targets that layout only
 * and rejects input it cannot parse.
 */
public final class PakArchive {

    private final Map<String, byte[]> filesByName;

    private PakArchive(final Map<String, byte[]> filesByName) {
        this.filesByName = filesByName;
    }

    /**
     * Parses a Westwood PAK archive.
     *
     * @param data the full archive bytes
     * @return the parsed archive
     * @throws IllegalArgumentException if the data is not a parseable PAK
     */
    public static PakArchive parse(final byte[] data) {
        if (data.length < 5) {
            throw new IllegalArgumentException("Too short to be a PAK: " + data.length + " bytes");
        }
        final int firstOffset = readU32(data, 0);
        if (firstOffset < 5 || firstOffset > data.length) {
            throw new IllegalArgumentException("Invalid PAK directory length: " + firstOffset);
        }
        final List<Integer> offsets = new ArrayList<>();
        final List<String> names = new ArrayList<>();
        int pos = 0;
        while (pos < firstOffset) {
            final int offset = readU32(data, pos);
            pos += 4;
            final String name = readCString(data, pos, firstOffset);
            pos += name.getBytes(StandardCharsets.US_ASCII).length + 1;
            offsets.add(offset);
            names.add(name);
        }
        return new PakArchive(slice(data, offsets, names));
    }

    private static Map<String, byte[]> slice(final byte[] data, final List<Integer> offsets, final List<String> names) {
        final Map<String, byte[]> result = new LinkedHashMap<>();
        for (int i = 0; i < names.size(); i++) {
            final String name = names.get(i);
            if (!name.isEmpty()) {
                final int start = offsets.get(i);
                final int end = offsets.get(i + 1);
                if (start < 0 || end > data.length || start > end) {
                    throw new IllegalArgumentException("Corrupt PAK entry '" + name + "': [" + start + ", " + end + ")");
                }
                final byte[] slice = new byte[end - start];
                System.arraycopy(data, start, slice, 0, end - start);
                result.put(name, slice);
            }
        }
        return result;
    }

    private static int readU32(final byte[] data, final int pos) {
        if (pos + 4 > data.length) {
            throw new IllegalArgumentException("Truncated uint32 at " + pos);
        }
        return (data[pos] & 0xFF) | ((data[pos + 1] & 0xFF) << 8) | ((data[pos + 2] & 0xFF) << 16) | ((data[pos + 3] & 0xFF) << 24);
    }

    private static String readCString(final byte[] data, final int start, final int limit) {
        int end = start;
        while (end < limit && data[end] != 0) {
            end++;
        }
        if (end >= limit) {
            throw new IllegalArgumentException("Unterminated name at " + start);
        }
        return new String(data, start, end - start, StandardCharsets.US_ASCII);
    }

    /**
     * Returns the file names in directory order.
     *
     * @return an immutable list of names
     */
    public List<String> names() {
        return List.copyOf(filesByName.keySet());
    }

    /**
     * Reports whether a named file is present.
     *
     * @param name the file name
     * @return {@code true} if present
     */
    public boolean contains(final String name) {
        return filesByName.containsKey(name);
    }

    /**
     * Returns a defensive copy of a file's bytes.
     *
     * @param name the file name
     * @return the file bytes
     * @throws IllegalArgumentException if no such file exists
     */
    public byte[] file(final String name) {
        final byte[] bytes = filesByName.get(name);
        if (bytes == null) {
            throw new IllegalArgumentException("No such file in PAK: " + name);
        }
        return bytes.clone();
    }

    /**
     * Returns the number of files (excluding the directory sentinel).
     *
     * @return the file count
     */
    public int size() {
        return filesByName.size();
    }
}
```

- [ ] **Step 4: Format**

```bash
JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home mvn -q -pl beholder-importer-westwood -f ~/Workspaces/Java/legend-of-the-beholder/pom.xml spotless:apply
```

- [ ] **Step 5: Verify (full reactor, JDK 21)**

```bash
JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home mvn -q -f ~/Workspaces/Java/legend-of-the-beholder/pom.xml verify
```
Expected: BUILD SUCCESS; 6 `PakArchiveTest` tests pass; JaCoCo ≥90% branch+line for `beholder-importer-westwood` (each branch of `parse`/`slice`/`readU32`/`readCString`/`file` is exercised by the six tests: happy list, exact bytes, contains true/false, unknown-name throw, truncated throw, defensive copy).

- [ ] **Step 6: Commit**

```bash
git -C ~/Workspaces/Java/legend-of-the-beholder add beholder-importer-westwood/src
git -C ~/Workspaces/Java/legend-of-the-beholder commit -m "feat(importer): add Westwood PakArchive reader"
```

---

### Task 3: Real-data integration test + operational unpack

**Files:**
- Create: `beholder-importer-westwood/src/test/java/eu/virtualparadox/beholder/importer/westwood/PakArchiveIT.java`

**Interfaces:** consumes `PakArchive` (Task 2). No new production code.

- [ ] **Step 1: Write the integration test (skip-if-absent)**

Create `.../src/test/java/eu/virtualparadox/beholder/importer/westwood/PakArchiveIT.java`:
```java
package eu.virtualparadox.beholder.importer.westwood;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assumptions.assumeThat;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import org.junit.jupiter.api.Test;

class PakArchiveIT {

    private static final Path EOBDATA1 = Path.of(
            System.getProperty("user.home"),
            "Games/Eye of the Beholder.app/Contents/Resources/game/EOBDATA1.PAK");

    @Test
    void parsesRealEobData1Archive() throws IOException {
        assumeThat(Files.isReadable(EOBDATA1))
                .as("EOB1 game data present at %s", EOBDATA1)
                .isTrue();
        final PakArchive archive = PakArchive.parse(Files.readAllBytes(EOBDATA1));
        assertThat(archive.names()).startsWith("ADLIB.DAT", "PCSOUND.DAT", "PCGAMESN.DAT");
        assertThat(archive.size()).isGreaterThan(3);
        assertThat(archive.file("ADLIB.DAT")).isNotEmpty();
    }
}
```

- [ ] **Step 2: Run the integration test (JDK 21)**

```bash
JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home mvn -q -pl beholder-importer-westwood -f ~/Workspaces/Java/legend-of-the-beholder/pom.xml verify
```
Expected: BUILD SUCCESS; `PakArchiveIT` runs under Failsafe and passes against the real `EOBDATA1.PAK` (names start `ADLIB.DAT, PCSOUND.DAT, PCGAMESN.DAT`). If the game data is absent, the test is skipped (assumption), not failed.

- [ ] **Step 3: Operationally unpack EOB1 (scratch only — NEVER commit game data)**

This proves the extractor end-to-end on all EOB1 archives and produces loose files for P1. Do it via a throwaway `main` or a JShell snippet reading each `EOBDATA*.PAK` and writing entries under the scratch dir `/private/tmp/claude-501/-Users-tothp-Workspaces-Java-legend-of-the-beholder/9d16fc35-a0bc-4823-87e6-40291dc8cfb9/scratchpad/eob1-unpacked/`. Report the total file count extracted. Do NOT add any extracted bytes to git.

- [ ] **Step 4: Commit the integration test**

```bash
git -C ~/Workspaces/Java/legend-of-the-beholder add beholder-importer-westwood/src/test
git -C ~/Workspaces/Java/legend-of-the-beholder commit -m "test(importer): integration-test PakArchive against real EOBDATA1.PAK"
```

- [ ] **Step 5: Merge to main (--no-ff) and push**

```bash
cd ~/Workspaces/Java/legend-of-the-beholder
git checkout main
git merge --no-ff feat/p0-pak-extractor -m "Merge branch 'feat/p0-pak-extractor'"
git branch -d feat/p0-pak-extractor
git push
```
(Run one more `JAVA_HOME=<jdk21> mvn verify` on merged `main` before pushing.)

---

## Self-Review

**Spec coverage:** the P0 PAK-extractor workstream (skeleton plan §"Remaining P0 plans" #2) → Tasks 1–3. The `.INF` decompiler spike, AESOP spike, and §11 ADRs are separate plans.
**Placeholder scan:** none — exact paths, complete code, exact commands with expected output. (Task 3 Step 3 is deliberately an operational action, not code, and says so.)
**Type consistency:** `PakArchive.parse/names/contains/file/size` are used identically in `PakArchiveTest`, `PakArchiveIT`, and the implementation.
**Format subtlety:** the PAK directory-end + sentinel handling is pinned by the synthetic-archive unit tests (which build the exact on-disk layout) AND cross-checked by the real-data `*IT`. If the reference implementation's end-detection is off, the unit tests fail first and the implementer corrects to pass both.

## Execution outcome (2026-07-06)

Executed subagent-driven on branch `feat/p0-pak-extractor` (pending `--no-ff` merge to `main`): `beholder-importer-westwood` module + `PakArchive` reader + real-data integration test; all gates green on JDK 21, unit coverage branch/line 100%.

**Format correction (the key lesson).** The reference implementation + synthetic tests above encoded a WRONG directory-end assumption (a trailing empty-name sentinel whose offset = archive length). The synthetic unit tests *passed* because they built archives in that same wrong shape — false confidence. The **real-data integration test caught it**: `PakArchive.parse` threw on all 6 real `EOBDATA*.PAK`. The correct format, derived clean-room from kyra's `ResLoaderPak`, is interleaved `off0, name0, …, offN` (N+1 offsets, N names) with **position-based** end detection (`pos+4 == firstOffset`), no sentinel (the final offset is the real archive size in EOBDATA1/5/6 but junk in 2/3/4). After the fix: **6/6 archives parse** (EOBDATA1..6 → 17/2/40/47/36/44 unique files; duplicate names collapse last-wins, matching kyra), 186 files unpack to scratch.

**Lesson for future importers (P1-relevant):** validate every binary-format parser against REAL game data, not only synthetic fixtures built from the same (possibly wrong) assumption. Add the real-data `*IT` early — for the Westwood/AESOP importers especially.
