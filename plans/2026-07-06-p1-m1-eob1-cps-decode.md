# P1 · M1 — EOB1 CPS decode + palette (pixel-verified) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a clean-room EOB1 **CPS image decoder** (Westwood *Format80* LZ decompression) plus a **VGA palette** loader to `beholder-importer-westwood`, emitting framework-neutral output (palette indices + ARGB), and prove it byte-exact against real EOB1 game data via a committed SHA-256 golden.

**Architecture:** Three small, single-responsibility, framework-neutral classes in the existing importer module — `Format80Decompressor` (the LZ decruncher), `CpsImage` (CPS container parse → dispatch → indices), `VgaPalette` (`.PAL`/`.COL` → ARGB, 6-bit→8-bit). No IR module, no engine, no renderer, no libGDX (those are M2+). Output is raw `byte[]` indices + `int[]` ARGB — never `BufferedImage`/`Sprite`. Correctness is proven the way P0 taught us: synthetic vectors for branch coverage and fail-closed edges, and a **real-data integration test** as the load-bearing correctness gate (synthetic fixtures alone give false confidence — the P0 PAK bug slipped through green synthetic tests and was caught only on real data).

**Tech Stack:** Java 21, `eu.virtualparadox:parent`, JUnit Jupiter + AssertJ. PNG eyeball + SHA-256 use JDK-only APIs (`javax.imageio.ImageIO`, `java.security.MessageDigest`) in test/scratch scope — **no new production dependency**. Build on JDK 21 (`JAVA_HOME` → Corretto 21; the reactor enforces `[21,22)` fail-fast).

## Where this sits — the P1 milestone ladder

P1 (MASTER §10, PHASES "Phase 1") is decomposed into demonstrable, individually differential-testable milestones. **This plan is M1 only.**

| # | Milestone | Differential evidence | New surface |
|---|---|---|---|
| **M1** | **CPS decode + palette** *(this plan)* | Byte-exact SHA-256 of decoded indices vs real EOB1 data; visual match of a palette-paired screen | `importer-westwood`: CPS/Format80/palette |
| M2 | Static first-person render | Golden diff of one fixed (cell,facing) view vs kyra/DOSBox | `beholder-ir` geometry+tileset layers; headless renderer |
| M3 | Walkable level | Frame sequence along a fixed path matches ground truth | `engine-core` party+movement; dynamic render |
| M4 | `.INF` event graph + one trigger | The trigger fires identically to kyra/DOSBox | `engine-core` executor; IR event layer |
| M5 | Lossless round-trip | extract → IR (JSON) → render/play identically; byte-stable re-serialize | differential harness |

## Global Constraints

All `CODING_STANDARDS.md` gates apply (see the P0 plans for the full verbatim list). The load-bearing ones for this module:

- `<parent>` = reactor root `eu.virtualparadox.beholder:legend-of-the-beholder:0.1.0-SNAPSHOT`; base package `eu.virtualparadox.beholder.importer.westwood`.
- Palantir format (`spotless:apply`); max line 140; one statement/line; braces on control flow; uppercase `L` longs.
- Method params `final`; locals `final` where applicable; **no param reassignment**; **exactly one `return` per method**; **no `instanceof`**; **no switch `yield`**; no reference-equality; cyclomatic ≤8/method; ≤12 fields, ≤18 methods, ≤6 params per class.
- No `return null;` — throw or `Optional`/immutable-empty. NullAway ERROR for `eu.virtualparadox`.
- No `System.out`/`err`; no `printStackTrace`; no `Thread.sleep` (incl. tests). Use SLF4J if logging is needed (none is, here).
- Javadoc on public/protected types+methods with `@param`/`@return`/`@throws`; `doclint=all`, every warning fails.
- JaCoCo ≥90% branch AND line at bundle level. These are logic classes (not `**/model/**`/`*Record`) → they **are** measured; unit tests must exercise every branch.
- Unit tests `*Test.java` (Surefire); integration tests `*IT.java` (Failsafe). JUnit Jupiter + AssertJ.
- **NEVER commit game data or anything derived from it that reconstructs the art.** Synthetic bytes only in unit tests; the `*IT` reads the real PAK from `~/Games/...` if present and is skipped otherwise; the committed golden is a **SHA-256 hash only** (32 bytes, not a derivative work); rendered PNGs go to scratch, never git.

## Format facts (clean-room, grounded in eoblib + the EOB RE wiki)

Derived from `~/Workspaces/Java/eoblib/src/main/java/eoblib/Decruncher.java` (clean-room Java, mirrors the public `uncps.cpp`), the wiki pages `eob.cps` / `Format 80` / `eob.pal`, and cross-checked against `wallsetviewer`'s VGA palette math. **All 47 EOB1 CPS files verified on the real PAKs are `compressionType=4` (Format80), `uncompressedSize=64000` (320×200), `paletteSize=0`.**

**CPS header (10 bytes, little-endian):**

| Offset | Size | Field | Note |
|---|---|---|---|
| 0 | u16 | `fileSize` | informational; not validated here |
| 2 | u16 | `compressionType` | `0`=raw `1`=lzw12 `2`=lzw14 `3`=rle `4`=Format80. EOB1 = always `4`. |
| 4 | u32 | `uncompressedSize` | image size; EOB1 = `64000` |
| 8 | u16 | `paletteSize` | bytes of embedded palette after header; EOB1 = `0` |
| 10 + `paletteSize` | … | compressed image data | Format80 stream |

**Format80 decode** — read one command byte `code`, dispatch (checked **in this order**):
1. `code == 0xFE` — fill: `count`=u16, `color`=u8 → write `color` `count` times.
2. `code >= 0xC0` — absolute copy from dest: if `code==0xFF` then `count`=u16 else `count=(code&0x3F)+3`; then `pos`=u16; copy `count` bytes from `dest[pos..]` (absolute).
3. `code >= 0x80` — `0x80` = **terminator (stop)**; else copy-as-is: `count=code&0x3F` bytes verbatim from source.
4. `code < 0x80` — relative copy from dest: `count=(code>>4)+3`; `low`=u8; `offset=((code&0x0F)<<8)|low`; copy `count` bytes from `dest[dp-offset..]`.

**Critical:** back-references copy **byte-by-byte** (the source region grows as you write — offset 1 is an RLE run). A well-formed stream decodes to exactly `uncompressedSize` bytes and ends at `0x80`.

**Palette (`.PAL`/`.COL`):** 768 bytes = 256 × (R,G,B), each channel **6-bit VGA (0..63)**; scale to 8-bit with `round(v*255/63)` (integer: `(v*255+31)/63`). For EOB1 the palette is a **separate** PAK member, paired by filename convention (`BLUE.CPS↔BLUE.PAL`, `XANATHA.CPS↔XANATHA.PAL`, `BRICK{1,2,3}.CPS↔BRICK.PAL`, …).

## File Structure

```
beholder-importer-westwood/
└── src/
    ├── main/java/eu/virtualparadox/beholder/importer/westwood/
    │   ├── Format80Decompressor.java   CREATE (Task 1) — Westwood Format80 LZ decruncher
    │   ├── CpsImage.java               CREATE (Task 2) — CPS container → palette indices
    │   └── VgaPalette.java             CREATE (Task 3) — .PAL/.COL → ARGB (6→8 bit)
    └── test/java/eu/virtualparadox/beholder/importer/westwood/
        ├── Format80DecompressorTest.java  CREATE (Task 1) — synthetic command vectors + fail-closed
        ├── CpsImageTest.java              CREATE (Task 2) — synthetic headers + fail-closed
        ├── VgaPaletteTest.java            CREATE (Task 3) — synthetic palettes + scaling
        └── CpsGoldenIT.java               CREATE (Task 4) — real EOB1 CPS → SHA-256 golden (skip-if-absent)
```

## Execution note (workflow)

Work on branch `feat/p1-m1-cps-decode`; commit per task; `--no-ff` merge to `main` at the end. Prefix every `mvn` with `JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home`. Reusable shell vars:

```bash
export JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home
export REPO=~/Workspaces/Java/legend-of-the-beholder
```

- [ ] **Task 0 · Step 1: Create the branch**

```bash
git -C "$REPO" checkout -b feat/p1-m1-cps-decode
```
Expected: `Switched to a new branch 'feat/p1-m1-cps-decode'`.

---

### Task 1: `Format80Decompressor` (TDD, synthetic vectors)

**Files:**
- Create: `beholder-importer-westwood/src/main/java/eu/virtualparadox/beholder/importer/westwood/Format80Decompressor.java`
- Test: `beholder-importer-westwood/src/test/java/eu/virtualparadox/beholder/importer/westwood/Format80DecompressorTest.java`

**Interfaces:**
- Produces: `Format80Decompressor.decompress(byte[] src, int srcOffset, int expectedSize)` → `byte[]` of length `expectedSize` (throws `IllegalArgumentException` on malformed/overrun/underrun/size-mismatch input).

- [ ] **Step 1: Write the failing test**

Create `.../test/.../Format80DecompressorTest.java`:
```java
package eu.virtualparadox.beholder.importer.westwood;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import org.junit.jupiter.api.Test;

class Format80DecompressorTest {

    Format80DecompressorTest() {}

    private static byte[] bytes(final int... values) {
        final byte[] out = new byte[values.length];
        for (int i = 0; i < values.length; i++) {
            out[i] = (byte) values[i];
        }
        return out;
    }

    @Test
    void copyAsIsFromSource() {
        // 0x83 = copy 3 verbatim from source; 0x80 terminator.
        final byte[] src = bytes(0x83, 0x41, 0x42, 0x43, 0x80);
        assertThat(Format80Decompressor.decompress(src, 0, 3)).containsExactly(0x41, 0x42, 0x43);
    }

    @Test
    void relativeBackReferenceRepeatsOverlapping() {
        // Write "AB", then relative copy count=3 offset=1 -> "ABBBB".
        final byte[] src = bytes(0x82, 0x41, 0x42, 0x00, 0x01, 0x80);
        assertThat(Format80Decompressor.decompress(src, 0, 5)).containsExactly(0x41, 0x42, 0x42, 0x42, 0x42);
    }

    @Test
    void absoluteBackReferenceShortForm() {
        // "ABCD", then 0xC0 (count=3) pos=0 -> "ABCDABC".
        final byte[] src = bytes(0x84, 0x41, 0x42, 0x43, 0x44, 0xC0, 0x00, 0x00, 0x80);
        assertThat(Format80Decompressor.decompress(src, 0, 7)).containsExactly(0x41, 0x42, 0x43, 0x44, 0x41, 0x42, 0x43);
    }

    @Test
    void absoluteBackReferenceLongForm() {
        // "ABCD", then 0xFF count=3 pos=1 -> "ABCDBCD".
        final byte[] src = bytes(0x84, 0x41, 0x42, 0x43, 0x44, 0xFF, 0x03, 0x00, 0x01, 0x00, 0x80);
        assertThat(Format80Decompressor.decompress(src, 0, 7)).containsExactly(0x41, 0x42, 0x43, 0x44, 0x42, 0x43, 0x44);
    }

    @Test
    void fillRun() {
        // 0xFE count=4 color=0x5A -> "ZZZZ".
        final byte[] src = bytes(0xFE, 0x04, 0x00, 0x5A, 0x80);
        assertThat(Format80Decompressor.decompress(src, 0, 4)).containsExactly(0x5A, 0x5A, 0x5A, 0x5A);
    }

    @Test
    void honoursSourceOffset() {
        final byte[] src = bytes(0xEE, 0xEE, 0x83, 0x41, 0x42, 0x43, 0x80);
        assertThat(Format80Decompressor.decompress(src, 2, 3)).containsExactly(0x41, 0x42, 0x43);
    }

    @Test
    void rejectsEarlyTerminatorBeforeExpectedSize() {
        final byte[] src = bytes(0x81, 0x41, 0x80);
        assertThatThrownBy(() -> Format80Decompressor.decompress(src, 0, 5)).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rejectsSourceUnderrun() {
        final byte[] src = bytes(0x83, 0x41);
        assertThatThrownBy(() -> Format80Decompressor.decompress(src, 0, 3)).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rejectsOutputOverrun() {
        final byte[] src = bytes(0x84, 0x41, 0x42, 0x43, 0x44, 0x80);
        assertThatThrownBy(() -> Format80Decompressor.decompress(src, 0, 2)).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rejectsBackReferenceOutOfRange() {
        // relative offset points before the start of the buffer.
        final byte[] src = bytes(0x00, 0x05, 0x80);
        assertThatThrownBy(() -> Format80Decompressor.decompress(src, 0, 4)).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rejectsNonPositiveExpectedSize() {
        assertThatThrownBy(() -> Format80Decompressor.decompress(bytes(0x80), 0, 0)).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rejectsOffsetOutOfRange() {
        assertThatThrownBy(() -> Format80Decompressor.decompress(bytes(0x80), 5, 1)).isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
JAVA_HOME=$JAVA_HOME mvn -q -pl beholder-importer-westwood -f "$REPO/pom.xml" test
```
Expected: FAIL — `Format80Decompressor` does not exist (compilation error).

- [ ] **Step 3: Write the implementation**

Create `.../main/.../Format80Decompressor.java`:
```java
package eu.virtualparadox.beholder.importer.westwood;

import java.util.Arrays;

/**
 * Decompresses a Westwood <em>Format80</em> (compression type 4) byte stream, the
 * LZ scheme used by Eye of the Beholder I CPS images.
 *
 * <p>The stream is a sequence of command bytes. A command either copies bytes
 * verbatim from the source, fills a run of one value, or back-references bytes
 * already written to the destination (relative to the write cursor, or at an
 * absolute destination position). Back-references are copied one byte at a time
 * so that overlapping runs (offset&nbsp;1 repeats the previous byte) expand
 * correctly. A well-formed stream produces exactly {@code expectedSize} bytes and
 * ends with the terminator {@code 0x80}.
 *
 * <p>This reader is fail-closed: source underrun, destination overrun, an
 * out-of-range back-reference, or a decoded length that does not match
 * {@code expectedSize} all throw {@link IllegalArgumentException}. Reimplemented
 * independently from the format documentation; no code is copied.
 */
public final class Format80Decompressor {

    private static final int TERMINATOR = 0x80;
    private static final int FILL = 0xFE;
    private static final int ABSOLUTE_LONG = 0xFF;
    private static final int ABSOLUTE_MIN = 0xC0;

    private final byte[] src;
    private final byte[] dest;
    private int sp;
    private int dp;

    private Format80Decompressor(final byte[] src, final byte[] dest) {
        this.src = src;
        this.dest = dest;
        this.sp = 0;
        this.dp = 0;
    }

    /**
     * Decompresses a Format80 stream.
     *
     * @param src the buffer containing the compressed stream
     * @param srcOffset the offset within {@code src} at which the stream begins
     * @param expectedSize the exact decompressed length (e.g. 64000 for a 320x200 CPS)
     * @return a fresh array of exactly {@code expectedSize} decoded bytes
     * @throws IllegalArgumentException if the arguments or the stream are invalid
     */
    public static byte[] decompress(final byte[] src, final int srcOffset, final int expectedSize) {
        if (expectedSize <= 0) {
            throw new IllegalArgumentException("expectedSize must be positive: " + expectedSize);
        }
        if (srcOffset < 0 || srcOffset > src.length) {
            throw new IllegalArgumentException("srcOffset out of range: " + srcOffset);
        }
        final byte[] source = Arrays.copyOfRange(src, srcOffset, src.length);
        final byte[] dest = new byte[expectedSize];
        final Format80Decompressor decoder = new Format80Decompressor(source, dest);
        decoder.run(expectedSize);
        return dest;
    }

    private void run(final int expectedSize) {
        boolean terminated = false;
        while (!terminated && dp < expectedSize) {
            terminated = step();
        }
        if (dp != expectedSize) {
            throw new IllegalArgumentException("Format80 decoded " + dp + " bytes, expected " + expectedSize);
        }
    }

    private boolean step() {
        final int code = readByte();
        boolean terminate = false;
        if (code == FILL) {
            fill();
        } else if (code >= ABSOLUTE_MIN) {
            absoluteCopy(code);
        } else if (code >= TERMINATOR) {
            terminate = highCopyOrTerminate(code);
        } else {
            relativeCopy(code);
        }
        return terminate;
    }

    private boolean highCopyOrTerminate(final int code) {
        boolean terminate = false;
        if (code == TERMINATOR) {
            terminate = true;
        } else {
            copyFromSource(code & 0x3F);
        }
        return terminate;
    }

    private void fill() {
        final int count = readWord();
        final int color = readByte();
        for (int i = 0; i < count; i++) {
            emit(color);
        }
    }

    private void copyFromSource(final int count) {
        for (int i = 0; i < count; i++) {
            emit(readByte());
        }
    }

    private void absoluteCopy(final int code) {
        final int count = absoluteCount(code);
        final int pos = readWord();
        copyFromDest(pos, count);
    }

    private int absoluteCount(final int code) {
        final int count = (code == ABSOLUTE_LONG) ? readWord() : ((code & 0x3F) + 3);
        return count;
    }

    private void relativeCopy(final int code) {
        final int count = (code >>> 4) + 3;
        final int low = readByte();
        final int offset = ((code & 0x0F) << 8) | low;
        copyFromDest(dp - offset, count);
    }

    private void copyFromDest(final int fromStart, final int count) {
        int from = fromStart;
        for (int i = 0; i < count; i++) {
            if (from < 0 || from >= dest.length) {
                throw new IllegalArgumentException("Format80 back-reference out of range: " + from);
            }
            emit(dest[from] & 0xFF);
            from++;
        }
    }

    private int readByte() {
        if (sp >= src.length) {
            throw new IllegalArgumentException("Format80 source underrun at " + sp);
        }
        final int value = src[sp] & 0xFF;
        sp++;
        return value;
    }

    private int readWord() {
        final int lo = readByte();
        final int hi = readByte();
        return lo | (hi << 8);
    }

    private void emit(final int value) {
        if (dp >= dest.length) {
            throw new IllegalArgumentException("Format80 output overrun at " + dp);
        }
        dest[dp] = (byte) value;
        dp++;
    }
}
```

- [ ] **Step 4: Format, then run the tests to verify they pass**

```bash
JAVA_HOME=$JAVA_HOME mvn -q -pl beholder-importer-westwood -f "$REPO/pom.xml" spotless:apply
JAVA_HOME=$JAVA_HOME mvn -q -pl beholder-importer-westwood -f "$REPO/pom.xml" test
```
Expected: PASS — 12 `Format80DecompressorTest` tests green. (Every branch of `step`/`highCopyOrTerminate`/`absoluteCount`/`copyFromDest`/`readByte`/`emit`/`run` is hit: the five command forms, source-offset, and the five fail-closed cases.)

- [ ] **Step 5: Commit**

```bash
git -C "$REPO" add beholder-importer-westwood/src
git -C "$REPO" commit -m "feat(importer): add Westwood Format80 decompressor"
```

---

### Task 2: `CpsImage` (TDD, synthetic headers)

**Files:**
- Create: `beholder-importer-westwood/src/main/java/eu/virtualparadox/beholder/importer/westwood/CpsImage.java`
- Test: `beholder-importer-westwood/src/test/java/eu/virtualparadox/beholder/importer/westwood/CpsImageTest.java`

**Interfaces:**
- Consumes: `Format80Decompressor.decompress(byte[], int, int)` (Task 1).
- Produces: `CpsImage.parse(byte[] data)` → `CpsImage` (throws `IllegalArgumentException` on malformed/unsupported input); `CpsImage#pixels()` → `byte[]` (a copy of the palette indices; length = `uncompressedSize`); `CpsImage#size()` → `int`.

- [ ] **Step 1: Write the failing test**

Create `.../test/.../CpsImageTest.java`:
```java
package eu.virtualparadox.beholder.importer.westwood;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.io.ByteArrayOutputStream;
import org.junit.jupiter.api.Test;

class CpsImageTest {

    CpsImageTest() {}

    /** Builds a CPS: 10-byte header (fileSize, compressionType, uncompressedSize, paletteSize) + stream. */
    private static byte[] cps(final int compressionType, final int uncompressedSize, final int paletteSize, final int... stream) {
        final ByteArrayOutputStream out = new ByteArrayOutputStream();
        writeU16(out, 0); // fileSize (ignored)
        writeU16(out, compressionType);
        writeU32(out, uncompressedSize);
        writeU16(out, paletteSize);
        for (int i = 0; i < paletteSize; i++) {
            out.write(0);
        }
        for (final int value : stream) {
            out.write(value & 0xFF);
        }
        return out.toByteArray();
    }

    private static void writeU16(final ByteArrayOutputStream out, final int value) {
        out.write(value & 0xFF);
        out.write((value >>> 8) & 0xFF);
    }

    private static void writeU32(final ByteArrayOutputStream out, final int value) {
        writeU16(out, value & 0xFFFF);
        writeU16(out, (value >>> 16) & 0xFFFF);
    }

    @Test
    void decodesFormat80Payload() {
        final byte[] data = cps(4, 3, 0, 0x83, 0x41, 0x42, 0x43, 0x80);
        final CpsImage image = CpsImage.parse(data);
        assertThat(image.size()).isEqualTo(3);
        assertThat(image.pixels()).containsExactly(0x41, 0x42, 0x43);
    }

    @Test
    void skipsEmbeddedPaletteBytesBeforeData() {
        final byte[] data = cps(4, 3, 2, 0x83, 0x41, 0x42, 0x43, 0x80);
        assertThat(CpsImage.parse(data).pixels()).containsExactly(0x41, 0x42, 0x43);
    }

    @Test
    void pixelsReturnsADefensiveCopy() {
        final CpsImage image = CpsImage.parse(cps(4, 3, 0, 0x83, 0x41, 0x42, 0x43, 0x80));
        image.pixels()[0] = 0x7A;
        assertThat(image.pixels()).containsExactly(0x41, 0x42, 0x43);
    }

    @Test
    void rejectsUnsupportedCompressionType() {
        assertThatThrownBy(() -> CpsImage.parse(cps(3, 3, 0, 0x00)))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("compression type");
    }

    @Test
    void rejectsHeaderTooShort() {
        assertThatThrownBy(() -> CpsImage.parse(new byte[9])).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rejectsNonPositiveUncompressedSize() {
        assertThatThrownBy(() -> CpsImage.parse(cps(4, 0, 0, 0x80))).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rejectsImplausiblyLargeUncompressedSize() {
        assertThatThrownBy(() -> CpsImage.parse(cps(4, 0x20000, 0, 0x80))).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rejectsPaletteSizeBeyondData() {
        assertThatThrownBy(() -> CpsImage.parse(cps(4, 3, 4, 0x83)))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
JAVA_HOME=$JAVA_HOME mvn -q -pl beholder-importer-westwood -f "$REPO/pom.xml" test
```
Expected: FAIL — `CpsImage` does not exist.

- [ ] **Step 3: Write the implementation**

Create `.../main/.../CpsImage.java`:
```java
package eu.virtualparadox.beholder.importer.westwood;

/**
 * A decoded Westwood CPS image: a rectangle of palette indices (one byte per
 * pixel), decompressed from a CPS container. Eye of the Beholder I CPS files are
 * Format80-compressed, 64000-byte (320x200) images whose palette lives in a
 * separate {@code .PAL}/{@code .COL} member (see {@link VgaPalette}).
 *
 * <p>The CPS header is 10 bytes, little-endian: {@code fileSize} (u16, ignored),
 * {@code compressionType} (u16), {@code uncompressedSize} (u32), {@code paletteSize}
 * (u16). Image data begins at {@code 10 + paletteSize}. Only {@code compressionType}
 * 4 (Format80) is supported; anything else is rejected fail-closed.
 *
 * <p>This decoder carries no width/height — the CPS format does not store them; a
 * 64000-byte image is 320x200 by convention, applied by the consumer. Output is a
 * raw index array, never a {@code BufferedImage}/{@code Sprite}.
 */
public final class CpsImage {

    private static final int HEADER_SIZE = 10;
    private static final int FORMAT80 = 4;
    private static final int MAX_UNCOMPRESSED = 0x10000;

    private final byte[] pixels;

    private CpsImage(final byte[] pixels) {
        this.pixels = pixels;
    }

    /**
     * Parses and decompresses a CPS image.
     *
     * @param data the full CPS file bytes
     * @return the decoded image
     * @throws IllegalArgumentException if the header is malformed or the compression type is unsupported
     */
    public static CpsImage parse(final byte[] data) {
        if (data.length < HEADER_SIZE) {
            throw new IllegalArgumentException("CPS shorter than header: " + data.length);
        }
        final int compressionType = readU16(data, 2);
        if (compressionType != FORMAT80) {
            throw new IllegalArgumentException("Unsupported CPS compression type: " + compressionType);
        }
        final int uncompressedSize = readI32(data, 4);
        if (uncompressedSize <= 0 || uncompressedSize > MAX_UNCOMPRESSED) {
            throw new IllegalArgumentException("Invalid CPS uncompressedSize: " + uncompressedSize);
        }
        final int paletteSize = readU16(data, 8);
        final int dataStart = HEADER_SIZE + paletteSize;
        if (dataStart > data.length) {
            throw new IllegalArgumentException("CPS paletteSize runs past end of file: " + paletteSize);
        }
        return new CpsImage(Format80Decompressor.decompress(data, dataStart, uncompressedSize));
    }

    /**
     * Returns a defensive copy of the palette indices (length = uncompressedSize).
     *
     * @return the pixel index bytes
     */
    public byte[] pixels() {
        return pixels.clone();
    }

    /**
     * Returns the number of pixels (index bytes).
     *
     * @return the pixel count
     */
    public int size() {
        return pixels.length;
    }

    private static int readU16(final byte[] data, final int pos) {
        return (data[pos] & 0xFF) | ((data[pos + 1] & 0xFF) << 8);
    }

    private static int readI32(final byte[] data, final int pos) {
        return (data[pos] & 0xFF)
                | ((data[pos + 1] & 0xFF) << 8)
                | ((data[pos + 2] & 0xFF) << 16)
                | ((data[pos + 3] & 0xFF) << 24);
    }
}
```

- [ ] **Step 4: Format, then run the tests to verify they pass**

```bash
JAVA_HOME=$JAVA_HOME mvn -q -pl beholder-importer-westwood -f "$REPO/pom.xml" spotless:apply
JAVA_HOME=$JAVA_HOME mvn -q -pl beholder-importer-westwood -f "$REPO/pom.xml" test
```
Expected: PASS — 8 `CpsImageTest` tests green; every `parse` branch exercised.

- [ ] **Step 5: Commit**

```bash
git -C "$REPO" add beholder-importer-westwood/src
git -C "$REPO" commit -m "feat(importer): add CPS container decoder (Format80)"
```

---

### Task 3: `VgaPalette` (TDD, synthetic palettes)

**Files:**
- Create: `beholder-importer-westwood/src/main/java/eu/virtualparadox/beholder/importer/westwood/VgaPalette.java`
- Test: `beholder-importer-westwood/src/test/java/eu/virtualparadox/beholder/importer/westwood/VgaPaletteTest.java`

**Interfaces:**
- Produces: `VgaPalette.parse(byte[] data)` → `VgaPalette` (requires exactly 768 bytes; throws otherwise); `VgaPalette#argb(int index)` → `int` (0xAARRGGBB, alpha 0xFF) for index 0..255.

- [ ] **Step 1: Write the failing test**

Create `.../test/.../VgaPaletteTest.java`:
```java
package eu.virtualparadox.beholder.importer.westwood;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import org.junit.jupiter.api.Test;

class VgaPaletteTest {

    VgaPaletteTest() {}

    private static byte[] palette768() {
        return new byte[768];
    }

    @Test
    void scalesSixBitChannelsToEightBit() {
        final byte[] data = palette768();
        // entry 1: R=63 (->255), G=32 (->130), B=1 (->4)
        data[3] = 63;
        data[4] = 32;
        data[5] = 1;
        final VgaPalette palette = VgaPalette.parse(data);
        assertThat(palette.argb(1)).isEqualTo(0xFFFF8204);
    }

    @Test
    void blackHasFullAlpha() {
        assertThat(VgaPalette.parse(palette768()).argb(0)).isEqualTo(0xFF000000);
    }

    @Test
    void masksToSixBits() {
        final byte[] data = palette768();
        data[0] = (byte) 0xFF; // low 6 bits = 63 -> 255
        assertThat(VgaPalette.parse(data).argb(0)).isEqualTo(0xFFFF0000);
    }

    @Test
    void rejectsWrongLength() {
        assertThatThrownBy(() -> VgaPalette.parse(new byte[767])).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rejectsIndexOutOfRange() {
        final VgaPalette palette = VgaPalette.parse(palette768());
        assertThatThrownBy(() -> palette.argb(256)).isInstanceOf(IllegalArgumentException.class);
        assertThatThrownBy(() -> palette.argb(-1)).isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
JAVA_HOME=$JAVA_HOME mvn -q -pl beholder-importer-westwood -f "$REPO/pom.xml" test
```
Expected: FAIL — `VgaPalette` does not exist.

- [ ] **Step 3: Write the implementation**

Create `.../main/.../VgaPalette.java`:
```java
package eu.virtualparadox.beholder.importer.westwood;

/**
 * A 256-colour VGA palette loaded from a Westwood {@code .PAL}/{@code .COL} file:
 * 768 bytes = 256 x (R, G, B), each channel a 6-bit VGA value (0..63). Channels
 * are scaled to 8-bit with {@code round(v * 255 / 63)} and exposed as packed
 * 0xAARRGGBB integers (alpha always 0xFF).
 *
 * <p>Framework-neutral: the palette is plain {@code int} ARGB values, not an
 * {@code IndexColorModel} or any UI-toolkit type.
 */
public final class VgaPalette {

    private static final int COLOR_COUNT = 256;
    private static final int PALETTE_BYTES = COLOR_COUNT * 3;

    private final int[] argb;

    private VgaPalette(final int[] argb) {
        this.argb = argb;
    }

    /**
     * Parses a 768-byte VGA palette.
     *
     * @param data the palette bytes (must be exactly 768)
     * @return the palette
     * @throws IllegalArgumentException if {@code data} is not 768 bytes
     */
    public static VgaPalette parse(final byte[] data) {
        if (data.length != PALETTE_BYTES) {
            throw new IllegalArgumentException("VGA palette must be " + PALETTE_BYTES + " bytes, got " + data.length);
        }
        final int[] colors = new int[COLOR_COUNT];
        for (int i = 0; i < COLOR_COUNT; i++) {
            final int r = scale(data[i * 3] & 0x3F);
            final int g = scale(data[i * 3 + 1] & 0x3F);
            final int b = scale(data[i * 3 + 2] & 0x3F);
            colors[i] = 0xFF000000 | (r << 16) | (g << 8) | b;
        }
        return new VgaPalette(colors);
    }

    private static int scale(final int sixBit) {
        return (sixBit * 255 + 31) / 63;
    }

    /**
     * Returns the packed ARGB colour at a palette index.
     *
     * @param index the palette index, 0..255
     * @return 0xAARRGGBB with alpha 0xFF
     * @throws IllegalArgumentException if {@code index} is outside 0..255
     */
    public int argb(final int index) {
        if (index < 0 || index >= COLOR_COUNT) {
            throw new IllegalArgumentException("Palette index out of range: " + index);
        }
        return argb[index];
    }
}
```

- [ ] **Step 4: Format, then run the FULL reactor verify (JDK 21)**

```bash
JAVA_HOME=$JAVA_HOME mvn -q -pl beholder-importer-westwood -f "$REPO/pom.xml" spotless:apply
JAVA_HOME=$JAVA_HOME mvn -q -f "$REPO/pom.xml" verify
```
Expected: BUILD SUCCESS; all unit tests green; JaCoCo ≥90% branch+line for `beholder-importer-westwood` (the three new classes are fully exercised by synthetic vectors — **no game data needed for the gate**).

- [ ] **Step 5: Commit**

```bash
git -C "$REPO" add beholder-importer-westwood/src
git -C "$REPO" commit -m "feat(importer): add VGA palette loader (.PAL/.COL, 6-bit to 8-bit)"
```

---

### Task 4: Real-data golden IT + eyeball (the correctness gate)

**Files:**
- Create: `beholder-importer-westwood/src/test/java/eu/virtualparadox/beholder/importer/westwood/CpsGoldenIT.java`

**Interfaces:** consumes `PakArchive` (P0), `CpsImage` (Task 2), `VgaPalette` (Task 3). No new production code.

**Why this task exists (P0 lesson).** Synthetic vectors prove the *code paths*, not that our reading of the format is *correct*. The correctness proof must run on real data and be cross-checked against an independent oracle. The committed artefact is a **SHA-256 hash of the decoded indices** — legally clean (not a derivative work; the art is not reconstructable from it) and a permanent regression ratchet.

- [ ] **Step 1: Establish the golden — decode real CPS, cross-check, render an eyeball PNG**

This is an operational verification step (like P0's operational unpack), done in scratch — **no game data or PNG is committed.**

1. Write a throwaway `scratchpad/inspect/Golden.java` that, using the built `beholder-importer-westwood` classes on the classpath, for each of these `(pak, cps)` pairs decodes the CPS and prints `name + SHA-256(pixels)`:
   - `EOBDATA3.PAK` → `INVENT.CPS` (small, 2481 B compressed)
   - `EOBDATA3.PAK` → `INTRO.CPS` (large, 16479 B — exercises many commands)
   - `EOBDATA5.PAK` → `XANATH1.CPS` (a known size-bound stress case)
   - `EOBDATA4.PAK` → `XDEATH3.CPS` (smallest, 1087 B — another stress case)
   For each, assert `pixels().length == 64000`.
2. **Cross-check against the independent eoblib decoder** (clean-room Java, `~/Workspaces/Java/eoblib/.../Decruncher.java`): extract each CPS to a scratch file, compile+run eoblib's `Decruncher` on it, and byte-compare eoblib's output to ours. They must be identical. (Two independent clean-room decoders agreeing is a strong correctness signal.)
3. **Eyeball:** render a name-paired image to PNG in scratch — decode `EOBDATA4.PAK/BLUE.CPS` with `EOBDATA4.PAK/BLUE.PAL` (or `EOBDATA5.PAK/XANATHA.CPS` + `XANATHA.PAL`), map indices through `VgaPalette.argb(...)` into a 320x200 `BufferedImage`, write PNG via `ImageIO`. Send it to the maintainer to confirm it reads as authentic EOB1.
4. Record the four SHA-256 hex strings printed in (1) — these become the golden constants in Step 2.

Run:
```bash
# build first so the classes exist on the classpath
JAVA_HOME=$JAVA_HOME mvn -q -pl beholder-importer-westwood -f "$REPO/pom.xml" test-compile
# then compile+run the scratch Golden.java against target/classes (see the scratchpad)
```
Expected: all four decode to exactly 64000 bytes, eoblib byte-matches ours on all four, the PNG reads as EOB1. If eoblib and ours disagree, STOP and debug (do not freeze a golden from an unverified decode).

- [ ] **Step 2: Write the golden integration test (skip-if-absent) with the frozen hashes**

Create `.../test/.../CpsGoldenIT.java` — fill each `EXPECTED_*` with the verified hash from Step 1:
```java
package eu.virtualparadox.beholder.importer.westwood;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assumptions.assumeTrue;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import org.junit.jupiter.api.Test;

/**
 * Byte-exact golden test of the CPS decoder against real, unmodified Eye of the
 * Beholder I game data. The archives are not part of the repository, so each case
 * is skipped (not failed) when the data is absent. Only SHA-256 hashes of the
 * decoded output are committed here — never the copyrighted image bytes.
 */
class CpsGoldenIT {

    CpsGoldenIT() {}

    private static final Path GAME = Path.of(
            System.getProperty("user.home"), "Games/Eye of the Beholder.app/Contents/Resources/game");

    // Frozen from a decode cross-checked against the independent eoblib decoder (Task 4, Step 1).
    private static final String EXPECTED_INVENT = "<sha256-hex-from-step-1>";
    private static final String EXPECTED_INTRO = "<sha256-hex-from-step-1>";
    private static final String EXPECTED_XANATH1 = "<sha256-hex-from-step-1>";
    private static final String EXPECTED_XDEATH3 = "<sha256-hex-from-step-1>";

    @Test
    void inventCpsMatchesGolden() throws IOException {
        assertCpsGolden("EOBDATA3.PAK", "INVENT.CPS", EXPECTED_INVENT);
    }

    @Test
    void introCpsMatchesGolden() throws IOException {
        assertCpsGolden("EOBDATA3.PAK", "INTRO.CPS", EXPECTED_INTRO);
    }

    @Test
    void xanath1CpsMatchesGolden() throws IOException {
        assertCpsGolden("EOBDATA5.PAK", "XANATH1.CPS", EXPECTED_XANATH1);
    }

    @Test
    void xdeath3CpsMatchesGolden() throws IOException {
        assertCpsGolden("EOBDATA4.PAK", "XDEATH3.CPS", EXPECTED_XDEATH3);
    }

    private static void assertCpsGolden(final String pak, final String cps, final String expectedHex) throws IOException {
        final Path archive = GAME.resolve(pak);
        assumeTrue(Files.isReadable(archive), "EOB1 game data present at " + archive);
        final PakArchive pakArchive = PakArchive.parse(Files.readAllBytes(archive));
        final CpsImage image = CpsImage.parse(pakArchive.file(cps));
        assertThat(image.size()).isEqualTo(64000);
        assertThat(sha256Hex(image.pixels())).isEqualTo(expectedHex);
    }

    private static String sha256Hex(final byte[] data) {
        final MessageDigest digest = newSha256();
        final byte[] hash = digest.digest(data);
        final StringBuilder hex = new StringBuilder(hash.length * 2);
        for (final byte b : hash) {
            hex.append(Character.forDigit((b >> 4) & 0xF, 16));
            hex.append(Character.forDigit(b & 0xF, 16));
        }
        return hex.toString();
    }

    private static MessageDigest newSha256() {
        try {
            return MessageDigest.getInstance("SHA-256");
        } catch (final NoSuchAlgorithmException e) {
            throw new IllegalStateException("SHA-256 unavailable", e);
        }
    }
}
```

- [ ] **Step 3: Run the integration test (JDK 21)**

```bash
JAVA_HOME=$JAVA_HOME mvn -q -pl beholder-importer-westwood -f "$REPO/pom.xml" verify
```
Expected: BUILD SUCCESS; `CpsGoldenIT` runs under Failsafe — 4 cases pass against the real archives (or are skipped if the game data is absent). Unit coverage remains ≥90%.

- [ ] **Step 4: Commit**

```bash
git -C "$REPO" add beholder-importer-westwood/src/test
git -C "$REPO" commit -m "test(importer): byte-exact CPS golden vs real EOB1 data (SHA-256)"
```

- [ ] **Step 5: Merge to main (--no-ff) and push**

```bash
JAVA_HOME=$JAVA_HOME mvn -q -f "$REPO/pom.xml" verify   # green on the branch first
git -C "$REPO" checkout main
git -C "$REPO" merge --no-ff feat/p1-m1-cps-decode -m "Merge branch 'feat/p1-m1-cps-decode': EOB1 CPS decode + palette (M1)"
git -C "$REPO" branch -d feat/p1-m1-cps-decode
git -C "$REPO" push
```

---

## Self-Review

**Spec coverage.** M1 goal = "decompression + palette, pixel-verified against real data": Task 1 (Format80 decompression) + Task 2 (CPS container) + Task 3 (palette) + Task 4 (real-data SHA-256 golden + eoblib cross-check + eyeball). All three IR-layer-agnostic; no engine/render/IR surface leaks in (M2+). Framework-neutral output (byte[]/int[], never BufferedImage in production) — the only `ImageIO`/`BufferedImage` use is the scratch eyeball (Task 4 Step 1), not committed.

**Placeholder scan.** All code is complete. The one deliberately deferred value is the four `EXPECTED_*` golden hashes — these are *derived outputs* that can only be produced by decoding the real data, and Task 4 Step 1 specifies exactly how they are computed and cross-checked before Step 2 freezes them (the same "operational step" pattern as the P0 plan's real-data unpack). This is a golden-establishment procedure, not a vague TODO.

**Type consistency.** `Format80Decompressor.decompress(byte[], int, int)` is called identically in `CpsImage.parse` and the tests. `CpsImage.parse`/`pixels`/`size` and `VgaPalette.parse`/`argb` match across their tests, `CpsGoldenIT`, and the Task 4 eyeball. `PakArchive.parse`/`file` reused from P0 unchanged.

**Gate compliance.** Every method has one `return`; no `instanceof`/`switch yield`; params `final`; `src`/`dest` fields hold internally-created arrays (`Arrays.copyOfRange` + `new byte[]`) so no external mutable state is stored (avoids SpotBugs EI_EXPOSE_REP2); no `System.out` (the scratch `Golden.java` is throwaway, outside the module). Cyclomatic per method ≤ ~5.

**Format subtlety (the P0 reflex).** The Format80 command grammar is pinned by synthetic vectors *and* re-proven on real data with an independent decoder (eoblib) — so a wrong-but-self-consistent reading (the P0 failure mode) cannot pass: the eoblib cross-check and the four real-file 64000-byte decodes would diverge before any golden is frozen.
