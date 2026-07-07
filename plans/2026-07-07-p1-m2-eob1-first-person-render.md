# P1 · M2 — EOB1 static first-person render (kyra-differential) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Render **one fixed `(cell, facing)` first-person view** of EOB1 level 1 through the real IR — decode VCN/VMP/MAZ/INF, assemble them into in-memory IR (geometry + tileset), and produce the 176×120 3D viewport as a headless `int[]` ARGB image — then prove it against the shipping renderer via a **kyra `set_position` screenshot pixel-differential**.

**Architecture:** Three modules. `beholder-importer-westwood` gains the Westwood tileset/level decoders (a shared `WestwoodContainer` decompressor extracted from `CpsImage`, plus `VcnAtlas`, `VmpProjection`, `Maze`, `LevelInfoHeader`), each verified byte-exact against the independent clean-room `eoblib` decoder. `beholder-ir` gains the two separable IR layers as versioned immutable records — geometry (`LevelGeometry`: a 32×32 grid of cells, 4 wall archetypes each) and tileset/appearance (`TileSet`: VCN atlas + VMP projection + palette) — plus assemblers that map importer POJOs → IR. A new headless `beholder-render-core` module turns `(LevelGeometry, TileSet, party position, facing)` into a 176×120 `int[]` ARGB frame (18-cell far-to-near draw model, no libGDX — that arrives at M3). No JSON/on-disk IR yet (deferred to M5 round-trip).

**Tech Stack:** Java 21, `eu.virtualparadox:parent`, JUnit Jupiter + AssertJ. PNG eyeballs + SHA-256 use JDK-only APIs (`javax.imageio.ImageIO`, `java.security.MessageDigest`) in test/scratch scope. **No new production dependency.** Build on JDK 21 (Corretto 21; reactor enforces `[21,22)` fail-fast).

## Where this sits — the P1 milestone ladder

This is **M2** of the five-milestone P1 EOB1 vertical slice (`PHASES.md` §"Phase 1", ladder recorded there). M1 (CPS decode + palette) is **done** (`d17befb`). M2 introduces the `beholder-ir` geometry + tileset layers and a headless renderer; M3 adds movement + `beholder-render-libgdx`; M4 the `.INF` event graph + one trigger; M5 the JSON round-trip. M2's differential retires the **VMP-projection fidelity risk** — the real render risk of P1.

## Global Constraints

All `CODING_STANDARDS.md` gates apply. The load-bearing ones:

- `<parent>` = reactor root `eu.virtualparadox.beholder:legend-of-the-beholder:0.1.0-SNAPSHOT`. Packages: `eu.virtualparadox.beholder.importer.westwood`, `eu.virtualparadox.beholder.ir`, `eu.virtualparadox.beholder.render`.
- Palantir format (`spotless:apply`); max line 140; one statement/line; braces on control flow; uppercase `L` longs.
- Method params `final`; locals `final` where applicable; **no param reassignment**; **exactly one `return` per method**; **no `instanceof`**; **no `switch yield`**; no reference-equality; cyclomatic ≤8/method; ≤12 fields, ≤18 methods, ≤6 params per class.
- No `return null;` — throw or `Optional`/immutable-empty. NullAway ERROR for `eu.virtualparadox`.
- No `System.out`/`err`; no `printStackTrace`; no `Thread.sleep` (incl. tests). Fail-closed: reject malformed input with `IllegalArgumentException`.
- Javadoc on public/protected types+methods with `@param`/`@return`/`@throws`; `doclint=all`.
- JaCoCo ≥90% branch AND line at bundle level. IR records are coverage-exempt (`**/*Record.class` / `**/model/**`); parsers/renderers are measured → exercise every branch.
- **DOCUMENT EVERY MAGIC VALUE.** No bare literal in logic. Every binary-format offset, field size, magic number, bitmask, and enum disk-value MUST be a named `ALL_CAPS` constant **with a comment stating what it is and why** (its source in the format spec / recon). This is a hard review criterion for this plan, not optional polish — the reverse-engineered *reason* a constant exists is the knowledge this project produces. Example: `// VMP 16-bit codes are little-endian on PC; eoblib's VMP.java reads big-endian, which is WRONG for PC data (verified against real .vmp files).`
- **NEVER commit game data or art-reconstructing derivatives.** Real `.PAK` data read from `~/Games/...` if present, else the `*IT` skips. Golden references committed as SHA-256 hashes only; kyra/DOSBox reference PNGs and rendered PNGs stay local/scratch, never git.

## Format facts (clean-room, grounded in eoblib + wallsetviewer + the EOB RE wiki + kyra as read-only oracle)

Verified against the real EOB1 PAKs. **Level 1's full asset set lives in `EOBDATA3.PAK`:** `LEVEL1.MAZ` (4102 B), `LEVEL1.INF` (1960 B), `BRICK.VCN` (34847 B), `BRICK.VMP` (5834 B), `BRICK.PAL` (768 B). All of VCN/VMP/CPS are wrapped in the **same Westwood container** and compressed (usually Format80).

**Westwood container header (10 bytes, little-endian)** — shared by CPS/VCN/VMP:
`fileSize u16 @0` (informational) · `compressionType u16 @2` (0=raw, 3=RLE, 4=Format80) · `uncompressedSize u32 @4` · `paletteSize u16 @8`; payload begins at `10 + paletteSize`. (This is exactly M1's `CpsImage` header; Task 1 extracts it into `WestwoodContainer`.)

**VCN (decompressed payload)** — the 8×8 wall/background tile atlas:
`tileCount u16 LE @0` · `backdropSubPalette[16] @2` (16 bytes; each = an index into the 256-color VGA palette) · `wallSubPalette[16] @18` (16 bytes) · then `tileCount × 32` bytes of tile data @34. Each tile = 8×8 pixels at **4bpp packed**, 2 pixels/byte (`byte>>4` = left pixel, `byte&0xF` = right pixel), row-major → 64 nibble values 0..15. A nibble is **not** a color: it indexes one of the two 16-entry sub-palettes (backdrop tiles use `backdropSubPalette`, wall tiles use `wallSubPalette`), whose entry is the real 256-VGA index. Nibble **0 = transparent**. There is **no runtime distance shading** — near/far look is baked into which tiles the VMP selects.

**VMP (decompressed payload)** — the projection map:
`codeCount u16 LE @0` · then `codeCount × u16 LE` codes. **CRITICAL: little-endian on PC.** eoblib's `VMP.java` reads big-endian, which is wrong for PC `.vmp` files (verified: LE gives clean integer wall-set counts, BE does not). Each 16-bit code: `bit15 = limit` (wall/floor seam-blend flag; ignore for M2 plain walls), `bit14 = flip` (draw the tile horizontally mirrored), `bits13..0 = tileIndex` into the VCN atlas (mask `0x3FFF`). Layout: `codes[0..329]` = the **background** (floor/ceiling) layer, 22×15 row-major (330 = 22×15); then `nrWallSets` blocks of **431 codes each** starting at index 330 (`nrWallSets = (codeCount - 330) / 431`). All real `.vmp` are 5834 bytes → `codeCount = 2916` → `nrWallSets = (2916-330)/431 = 6`.

**MAZ** — the level grid:
`width u16 LE @0` · `height u16 LE @2` · `nof u16 LE @4` (=4, bytes per cell) · then `width*height` cells of 4 bytes each @6, row-major (`cellIndex = y*width + x`). **EOB1 is 32×32** (`4102 = 6 + 32*32*4`). Each cell's 4 bytes are the wall values for directions **N, E, S, W** (clockwise from north — eoblib's order; the wiki's "N,S,W,E" prose is WRONG, disproven empirically by shared-wall agreement and corroborated by kyra's move table `{-32,+1,+32,-1}`). Each byte is a `wallMappingIndex` (0..255): **0 = open/air**; **1,2 = the two plain-wall graphic variants**; 3..22 = predefined door/special archetypes; any index may be redefined per-level by `.INF` `0xFB` commands.

**`.INF` header (for M2 we need only the container + names + wall-mapping, NOT the trigger bytecode — that's M4):** after the maze filename and a small fixed header, the `.INF` carries the paired `vcnVmpName` (e.g. `brick`) and a wall-mapping table (defaults for indices 0..22, plus `0xFB wallMappingIndex,wallType,decorationID,eventMask,flags` overrides). `wallType` (1-based) selects which VMP wall-set to draw; `decorationID` (0xFF=none) is a separate overlay (out of scope for M2's plain-corridor render).

**First-person view model** (kyra `scene_rpg.cpp` `generateBlockDrawingBuffer`, read-only oracle):
- **Viewport = 176×120 px** (22×15 blocks of 8×8), drawn at screen offset **(0,0)** (top-left of the 320×200 frame).
- **18 visible cells** = 17 lettered (A..Q) + the party cell, a 7-5-3-2 diamond over 4 depth rows. Local axes: `+ly` = forward (away from party), `+lx` = party's right. Rows: `ly=3`→A..G (`lx=-3..+3`), `ly=2`→H..L (`lx=-2..+2`), `ly=1`→M..O (`lx=-1..+1`), `ly=0`→P(`lx=-1`), party(`0`), Q(`lx=+1`).
- **Facing → grid delta** `(Δcol, Δrow)` from local `(lx, ly)`: N=`(lx, -ly)`, E=`(ly, lx)`, S=`(-lx, ly)`, W=`(-ly, -lx)`. (North forward = −row, per kyra `blockPosTable[0]=-32`.)
- **Draw order: strictly far row → near row** (painter's algorithm; nearer slots overwrite farther). Each visible wall maps to one of **25 on-screen slots** in `WALL_RENDER_DATA` (below), which name a rectangle in the 22×15 block grid and a base offset into the current wall-set's 431-code block. Right-side ("west") slots are drawn mirrored (`flip`).

**`WALL_RENDER_DATA[25]`** (from the `eob.vmp` wiki; **illustrative/approximate — verified and adjusted against the kyra golden in Task 10**). Fields: `baseOffset` (signed, into the wall-set's 431-code block), `viewportOffset` (`x = v%22, y = v/22`, in 8×8 blocks), `w`,`h` (blocks), `skip` (row skip), `flip`:

```
name       base  vpOff  w   h  skip flip
A-east        3     66   5   1   2   0
B-east        1     68   5   3   0   0
C-east       -4     74   5   1   0   0
E-west       -4     79   5   1   0   1
F-west        1     83   5   3   0   1
G-west        3     87   5   1   2   1
B-south      32     66   5   2   4   0
C-south      28     68   5   6   0   0
D-south      28     74   5   6   0   0
E-south      28     80   5   6   0   0
F-south      28     86   5   2   4   0
H-east       16     66   6   2   0   0
I-east      -20     50   8   2   0   0
K-west      -20     58   8   2   0   1
L-west       16     86   6   2   0   1
I-south      62     44   8   6   4   0
J-south      58     50   8  10   0   0
K-south      58     60   8   6   4   0
M-east      -56     25  12   3   0   0
O-west      -56     38  12   3   0   1
M-south     151     22  12   3  13   0
O-south     138     41  12   3  13   0
N-south     138     25  12  16   0   0
P-east     -101      0  15   3   0   0
Q-west     -101     19  15   3   0   1
```

## Module / File Structure

```
beholder-importer-westwood/src/main/java/eu/virtualparadox/beholder/importer/westwood/
  WestwoodContainer.java   CREATE (T1) — shared container header parse + Format80/raw decompress
  VcnAtlas.java            CREATE (T2) — VCN payload -> tiles + 2 sub-palettes
  VmpProjection.java       CREATE (T3) — VMP payload -> background + wall-set code blocks (+ WALL_RENDER_DATA)
  Maze.java                CREATE (T4) — MAZ -> 32x32 wall-index grid
  LevelInfoHeader.java     CREATE (T5) — .INF names + wall-mapping (index -> wallType)
  CpsImage.java            MODIFY (T1) — delegate decompression to WestwoodContainer
beholder-ir/src/main/java/eu/virtualparadox/beholder/ir/
  Facing.java              CREATE (T6) — N/E/S/W enum + local->grid delta
  WallArchetype.java       CREATE (T6) — closed enum (OPEN, SOLID; extensible)
  model/LevelGeometry.java CREATE (T6) — record: 32x32 cells x 4 wall archetypes (+ wall-set id)   [coverage-exempt: model/]
  model/TileSet.java       CREATE (T6) — record: VCN atlas + VMP projection + VgaPalette
  WestwoodToIr.java        CREATE (T6) — assemble importer POJOs -> IR records
beholder-render-core/      CREATE (T7) new module
  src/main/java/eu/virtualparadox/beholder/render/
    Rgba.java              CREATE (T7) — small ARGB helpers (packing)
    TileRasterizer.java    CREATE (T7) — 8x8 tile -> ARGB (4bpp + sub-palette + flip + transparency)
    ViewFootprint.java     CREATE (T8) — 18 visible cells + facing transform
    FirstPersonRenderer.java CREATE (T9) — compose 176x120 int[] ARGB frame
```

## Execution note (workflow)

Branch `feat/p1-m2-first-person-render`; commit per task; `--no-ff` merge to `main` at the end (controller does the merge). Prefix every `mvn` with `JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home`. Reusable vars:

```bash
export JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home
export REPO=~/Workspaces/Java/legend-of-the-beholder
```

- [ ] **Task 0 · Step 1: Create the branch**
```bash
git -C "$REPO" checkout -b feat/p1-m2-first-person-render
```
Expected: `Switched to a new branch 'feat/p1-m2-first-person-render'`.

---

### Task 1: Format80 hardening + `WestwoodContainer` extraction

**Files:**
- Modify: `beholder-importer-westwood/.../Format80DecompressorTest.java` (add 2 hardening tests)
- Create: `beholder-importer-westwood/.../WestwoodContainer.java`
- Modify: `beholder-importer-westwood/.../CpsImage.java` (delegate decompression)
- Test: `beholder-importer-westwood/.../WestwoodContainerTest.java`

**Interfaces:**
- Consumes: `Format80Decompressor.decompress(byte[], int, int)` (M1).
- Produces: `WestwoodContainer.decompress(byte[] data) -> byte[]` — parses the 10-byte container header, dispatches on `compressionType`, returns the decompressed payload; throws `IllegalArgumentException` fail-closed. `CpsImage.parse` becomes a thin wrapper (its `pixels()`/`size()` contract is unchanged).

- [ ] **Step 1: Add the two carried-forward Format80 hardening tests (close the M1-deferred branch arms)**

Add to `Format80DecompressorTest.java` (uses its existing `bytes(int...)` helper):
```java
    @Test
    void rejectsNegativeSourceOffset() {
        // Closes the srcOffset<0 arm of the offset guard (M1 deferred Minor).
        assertThatThrownBy(() -> Format80Decompressor.decompress(bytes(0x80), -1, 1)).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rejectsAbsoluteBackReferenceWalkingOffEnd() {
        // Absolute copy count=3 from pos=1 into a 2-byte output walks past dest end -> must throw.
        // (Closes the from>=dest.length arm of copyFromDest's guard, M1 deferred Minor.)
        final byte[] src = bytes(0x82, 0x41, 0x42, 0xFF, 0x03, 0x00, 0x01, 0x00, 0x80);
        assertThatThrownBy(() -> Format80Decompressor.decompress(src, 0, 2)).isInstanceOf(IllegalArgumentException.class);
    }
```

- [ ] **Step 2: Write the failing `WestwoodContainer` test**

Create `WestwoodContainerTest.java`:
```java
package eu.virtualparadox.beholder.importer.westwood;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.io.ByteArrayOutputStream;
import org.junit.jupiter.api.Test;

class WestwoodContainerTest {

    WestwoodContainerTest() {}

    /** Builds a Westwood container: 10-byte header (fileSize, compressionType, uncompressedSize, paletteSize) + payload. */
    private static byte[] container(final int compressionType, final int uncompressedSize, final int paletteSize, final int... payload) {
        final ByteArrayOutputStream out = new ByteArrayOutputStream();
        writeU16(out, 0); // fileSize: informational, ignored by the decoder
        writeU16(out, compressionType);
        writeU16(out, uncompressedSize & 0xFFFF);
        writeU16(out, (uncompressedSize >>> 16) & 0xFFFF);
        writeU16(out, paletteSize);
        for (int i = 0; i < paletteSize; i++) {
            out.write(0);
        }
        for (final int b : payload) {
            out.write(b & 0xFF);
        }
        return out.toByteArray();
    }

    private static void writeU16(final ByteArrayOutputStream out, final int value) {
        out.write(value & 0xFF);
        out.write((value >>> 8) & 0xFF);
    }

    @Test
    void decompressesFormat80Payload() {
        // 0x83 = copy 3 literal bytes from source, then 0x80 terminator.
        final byte[] data = container(4, 3, 0, 0x83, 0x41, 0x42, 0x43, 0x80);
        assertThat(WestwoodContainer.decompress(data)).containsExactly(0x41, 0x42, 0x43);
    }

    @Test
    void copiesRawPayloadWhenUncompressed() {
        // compressionType 0 = the payload is the raw image, copied verbatim.
        final byte[] data = container(0, 3, 0, 0x41, 0x42, 0x43);
        assertThat(WestwoodContainer.decompress(data)).containsExactly(0x41, 0x42, 0x43);
    }

    @Test
    void skipsEmbeddedPaletteBeforePayload() {
        final byte[] data = container(4, 3, 2, 0x83, 0x41, 0x42, 0x43, 0x80);
        assertThat(WestwoodContainer.decompress(data)).containsExactly(0x41, 0x42, 0x43);
    }

    @Test
    void rejectsHeaderTooShort() {
        assertThatThrownBy(() -> WestwoodContainer.decompress(new byte[9])).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rejectsUnsupportedCompressionType() {
        // Type 3 (RLE) is a real Westwood type but not needed by EOB1 tilesets; reject fail-closed until a real file needs it.
        assertThatThrownBy(() -> WestwoodContainer.decompress(container(3, 3, 0, 0x00))).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rejectsNonPositiveUncompressedSize() {
        assertThatThrownBy(() -> WestwoodContainer.decompress(container(4, 0, 0, 0x80))).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rejectsImplausiblyLargeUncompressedSize() {
        // Format80 absolute back-refs are u16-limited to 64KB; reject larger as malformed (OOM guard).
        assertThatThrownBy(() -> WestwoodContainer.decompress(container(4, 0x20000, 0, 0x80))).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rejectsPaletteRunningPastEnd() {
        assertThatThrownBy(() -> WestwoodContainer.decompress(container(4, 3, 4, 0x83))).isInstanceOf(IllegalArgumentException.class);
    }
}
```
(The `container(4,3,4,0x83)` case: header claims paletteSize but the file is too short — analogous to M1's `validateDataStart`; build a truly-too-short case if needed so `dataStart > length` trips.)

- [ ] **Step 3: Run the tests to verify they fail**
```bash
JAVA_HOME=$JAVA_HOME mvn -q -pl beholder-importer-westwood -f "$REPO/pom.xml" test
```
Expected: FAIL — `WestwoodContainer` does not exist.

- [ ] **Step 4: Write `WestwoodContainer`, and refactor `CpsImage` to delegate**

Create `WestwoodContainer.java`. Move the container-header logic out of `CpsImage` (constants + validators), adding `RAW` support:
```java
package eu.virtualparadox.beholder.importer.westwood;

import java.util.Arrays;

/**
 * Decompresses a Westwood container file (the wrapper shared by EOB1 CPS, VCN and VMP
 * files inside the {@code EOBDATA*.PAK} archives).
 *
 * <p>The container is a 10-byte little-endian header followed by the (usually compressed)
 * payload: {@code fileSize u16 @0} (informational), {@code compressionType u16 @2},
 * {@code uncompressedSize u32 @4}, {@code paletteSize u16 @8}; the payload begins at
 * {@code 10 + paletteSize}. Only the compression types EOB1 actually uses are supported:
 * {@code 0} (raw/verbatim) and {@code 4} (Format80 LZ). Anything else is rejected
 * fail-closed rather than best-effort decoded.
 */
public final class WestwoodContainer {

    /** Header is 10 bytes: fileSize(u16) + compressionType(u16) + uncompressedSize(u32) + paletteSize(u16). */
    private static final int HEADER_SIZE = 10;

    private static final int COMPRESSION_TYPE_OFFSET = 2;
    private static final int UNCOMPRESSED_SIZE_OFFSET = 4;
    private static final int PALETTE_SIZE_OFFSET = 8;

    /** compressionType 0: payload is the raw image, no decompression. */
    private static final int RAW = 0;
    /** compressionType 4: payload is a Format80 LZ stream. */
    private static final int FORMAT80 = 4;

    /** Format80 absolute back-references use a u16 position, so a payload cannot exceed 64 KiB; larger is malformed. */
    private static final int MAX_UNCOMPRESSED = 0x10000;

    private WestwoodContainer() {}

    /**
     * Decompresses a Westwood container file to its raw payload bytes.
     *
     * @param data the full container file bytes
     * @return the decompressed payload (length = the header's {@code uncompressedSize})
     * @throws IllegalArgumentException if the header is malformed or the compression type is unsupported
     */
    public static byte[] decompress(final byte[] data) {
        if (data.length < HEADER_SIZE) {
            throw new IllegalArgumentException("Westwood container shorter than 10-byte header: " + data.length);
        }
        final int uncompressedSize = readI32(data, UNCOMPRESSED_SIZE_OFFSET);
        if (uncompressedSize <= 0 || uncompressedSize > MAX_UNCOMPRESSED) {
            throw new IllegalArgumentException("Invalid Westwood uncompressedSize: " + uncompressedSize);
        }
        final int dataStart = HEADER_SIZE + readU16(data, PALETTE_SIZE_OFFSET);
        if (dataStart > data.length) {
            throw new IllegalArgumentException("Westwood paletteSize runs past end of file");
        }
        return decodePayload(data, dataStart, readU16(data, COMPRESSION_TYPE_OFFSET), uncompressedSize);
    }

    private static byte[] decodePayload(final byte[] data, final int dataStart, final int compressionType, final int uncompressedSize) {
        final byte[] payload;
        if (compressionType == FORMAT80) {
            payload = Format80Decompressor.decompress(data, dataStart, uncompressedSize);
        } else if (compressionType == RAW) {
            payload = rawCopy(data, dataStart, uncompressedSize);
        } else {
            throw new IllegalArgumentException("Unsupported Westwood compression type: " + compressionType);
        }
        return payload;
    }

    private static byte[] rawCopy(final byte[] data, final int dataStart, final int uncompressedSize) {
        if (dataStart + uncompressedSize > data.length) {
            throw new IllegalArgumentException("Raw Westwood payload truncated");
        }
        return Arrays.copyOfRange(data, dataStart, dataStart + uncompressedSize);
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
Then edit `CpsImage.parse` so its body becomes: validate `data.length >= HEADER_SIZE`, require `compressionType == 4` (CPS is always Format80 — keep that specific check for a clear message), then `return new CpsImage(WestwoodContainer.decompress(data));`. Keep `CpsImage`'s own `MAX`/header constants comment-documented or delegate them; the existing `CpsImageTest` (8 tests) and `CpsGoldenIT` (4 real-data hashes) MUST stay green unchanged — that is the refactor's safety net.

- [ ] **Step 5: Format, run the module tests, verify green**
```bash
JAVA_HOME=$JAVA_HOME mvn -q -pl beholder-importer-westwood -f "$REPO/pom.xml" spotless:apply
JAVA_HOME=$JAVA_HOME mvn -q -pl beholder-importer-westwood -f "$REPO/pom.xml" verify
```
Expected: BUILD SUCCESS; the 2 new Format80 tests + 8 WestwoodContainer tests pass; **`CpsImageTest` and `CpsGoldenIT` unchanged and green** (proves the refactor preserved behavior); coverage ≥90%.

- [ ] **Step 6: Commit**
```bash
git -C "$REPO" add beholder-importer-westwood/src
git -C "$REPO" commit -m "refactor(importer): extract WestwoodContainer decompressor; harden Format80 coverage"
```

---

### Task 2: `VcnAtlas` decoder (+ eoblib byte-exact + atlas eyeball)

**Files:**
- Create: `beholder-importer-westwood/.../VcnAtlas.java`
- Test: `beholder-importer-westwood/.../VcnAtlasTest.java`, `.../VcnAtlasIT.java`

**Interfaces:**
- Consumes: `WestwoodContainer.decompress(byte[])` (T1).
- Produces: `VcnAtlas.parse(byte[] compressedVcn) -> VcnAtlas`; `tileCount() -> int`; `tile(int i) -> byte[]` (64 nibble values 0..15, row-major); `backdropSubPalette() -> int[]` / `wallSubPalette() -> int[]` (each 16 VGA indices).

- [ ] **Step 1: Write the failing unit test** (synthetic VCN: 1 tile, known nibbles)
```java
package eu.virtualparadox.beholder.importer.westwood;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.io.ByteArrayOutputStream;
import org.junit.jupiter.api.Test;

class VcnAtlasTest {

    VcnAtlasTest() {}

    /** Wraps a raw VCN payload in an uncompressed (type 0) Westwood container so parse() can decode it. */
    private static byte[] uncompressedContainer(final byte[] payload) {
        final ByteArrayOutputStream out = new ByteArrayOutputStream();
        writeU16(out, 0); // fileSize
        writeU16(out, 0); // compressionType 0 = raw
        writeU16(out, payload.length & 0xFFFF);
        writeU16(out, (payload.length >>> 16) & 0xFFFF);
        writeU16(out, 0); // paletteSize
        out.writeBytes(payload);
        return out.toByteArray();
    }

    private static void writeU16(final ByteArrayOutputStream out, final int value) {
        out.write(value & 0xFF);
        out.write((value >>> 8) & 0xFF);
    }

    private static byte[] oneTilePayload() {
        // tileCount=1 (u16 LE), backdropSubPalette[16], wallSubPalette[16], then 32 bytes of one 8x8 4bpp tile.
        final ByteArrayOutputStream out = new ByteArrayOutputStream();
        writeU16(out, 1);
        for (int i = 0; i < 16; i++) {
            out.write(100 + i); // backdrop sub-palette: distinct VGA indices
        }
        for (int i = 0; i < 16; i++) {
            out.write(200 + i); // wall sub-palette
        }
        // 32 bytes: each byte packs two pixels; first byte 0x12 -> pixels (1,2), rest 0.
        out.write(0x12);
        for (int i = 1; i < 32; i++) {
            out.write(0);
        }
        return out.toByteArray();
    }

    @Test
    void parsesHeaderAndOneTile() {
        final VcnAtlas atlas = VcnAtlas.parse(uncompressedContainer(oneTilePayload()));
        assertThat(atlas.tileCount()).isEqualTo(1);
        assertThat(atlas.backdropSubPalette()).startsWith(100, 101);
        assertThat(atlas.wallSubPalette()).startsWith(200, 201);
        // 4bpp unpack: byte 0x12 -> left pixel = 0x1, right pixel = 0x2.
        assertThat(atlas.tile(0)[0]).isEqualTo((byte) 0x1);
        assertThat(atlas.tile(0)[1]).isEqualTo((byte) 0x2);
    }

    @Test
    void rejectsTruncatedTileData() {
        final byte[] payload = oneTilePayload();
        final byte[] truncated = java.util.Arrays.copyOf(payload, payload.length - 4);
        assertThatThrownBy(() -> VcnAtlas.parse(uncompressedContainer(truncated))).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void rejectsIndexOutOfRange() {
        final VcnAtlas atlas = VcnAtlas.parse(uncompressedContainer(oneTilePayload()));
        assertThatThrownBy(() -> atlas.tile(1)).isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run to verify fail.** `... -pl beholder-importer-westwood ... test` → FAIL (no `VcnAtlas`).

- [ ] **Step 3: Implement `VcnAtlas`** — full header/tile decode with every offset a documented constant:
```java
package eu.virtualparadox.beholder.importer.westwood;

/**
 * The VCN 8x8 wall/background tile atlas for one EOB1 dungeon look (e.g. {@code BRICK.VCN}).
 *
 * <p>Decompressed payload layout (little-endian): {@code tileCount u16 @0};
 * {@code backdropSubPalette[16] @2} and {@code wallSubPalette[16] @18} — two 16-entry tables
 * each mapping a tile's 4-bit pixel value (0..15) to an index into the 256-colour VGA palette;
 * then {@code tileCount} tiles of 32 bytes each @34. Each tile is 8x8 pixels at 4 bits-per-pixel,
 * two pixels per byte (high nibble = left pixel, low nibble = right pixel), decoded row-major into
 * 64 values 0..15. Value 0 is transparent. There is no runtime distance shading — near/far look is
 * chosen by the VMP, not computed here.
 */
public final class VcnAtlas {

    /** tileCount is a u16 at payload offset 0. */
    private static final int TILE_COUNT_OFFSET = 0;
    /** Two 16-entry sub-palettes follow the count: backdrop @2, wall @18. */
    private static final int SUB_PALETTE_SIZE = 16;
    private static final int BACKDROP_SUB_PALETTE_OFFSET = 2;
    private static final int WALL_SUB_PALETTE_OFFSET = 18;
    /** Tile data starts after the count (2) + two 16-byte sub-palettes = offset 34. */
    private static final int TILE_DATA_OFFSET = 34;
    /** Each 8x8 tile is 32 bytes (64 pixels at 4bpp = 2 pixels/byte). */
    private static final int BYTES_PER_TILE = 32;
    private static final int PIXELS_PER_TILE = 64;
    /** A nibble high/low mask and shift for the 4bpp two-pixels-per-byte packing. */
    private static final int NIBBLE_MASK = 0x0F;
    private static final int HIGH_NIBBLE_SHIFT = 4;

    private final int tileCount;
    private final int[] backdropSubPalette;
    private final int[] wallSubPalette;
    private final byte[][] tiles;

    private VcnAtlas(final int tileCount, final int[] backdropSubPalette, final int[] wallSubPalette, final byte[][] tiles) {
        this.tileCount = tileCount;
        this.backdropSubPalette = backdropSubPalette;
        this.wallSubPalette = wallSubPalette;
        this.tiles = tiles;
    }

    /**
     * Decompresses and parses a VCN file.
     *
     * @param compressedVcn the full VCN file bytes (Westwood container)
     * @return the decoded atlas
     * @throws IllegalArgumentException if the payload is malformed
     */
    public static VcnAtlas parse(final byte[] compressedVcn) {
        final byte[] p = WestwoodContainer.decompress(compressedVcn);
        if (p.length < TILE_DATA_OFFSET) {
            throw new IllegalArgumentException("VCN payload shorter than 34-byte header: " + p.length);
        }
        final int count = readU16(p, TILE_COUNT_OFFSET);
        if (TILE_DATA_OFFSET + count * BYTES_PER_TILE > p.length) {
            throw new IllegalArgumentException("VCN tile data truncated for tileCount " + count);
        }
        final int[] backdrop = readSubPalette(p, BACKDROP_SUB_PALETTE_OFFSET);
        final int[] wall = readSubPalette(p, WALL_SUB_PALETTE_OFFSET);
        return new VcnAtlas(count, backdrop, wall, unpackTiles(p, count));
    }

    private static int[] readSubPalette(final byte[] p, final int offset) {
        final int[] table = new int[SUB_PALETTE_SIZE];
        for (int i = 0; i < SUB_PALETTE_SIZE; i++) {
            table[i] = p[offset + i] & 0xFF;
        }
        return table;
    }

    private static byte[][] unpackTiles(final byte[] p, final int count) {
        final byte[][] out = new byte[count][PIXELS_PER_TILE];
        for (int t = 0; t < count; t++) {
            final int base = TILE_DATA_OFFSET + t * BYTES_PER_TILE;
            for (int b = 0; b < BYTES_PER_TILE; b++) {
                final int value = p[base + b] & 0xFF;
                out[t][b * 2] = (byte) ((value >>> HIGH_NIBBLE_SHIFT) & NIBBLE_MASK); // left pixel
                out[t][b * 2 + 1] = (byte) (value & NIBBLE_MASK); // right pixel
            }
        }
        return out;
    }

    private static int readU16(final byte[] p, final int pos) {
        return (p[pos] & 0xFF) | ((p[pos + 1] & 0xFF) << 8);
    }

    /**
     * Returns the number of 8x8 tiles.
     *
     * @return the tile count
     */
    public int tileCount() {
        return tileCount;
    }

    /**
     * Returns a tile's 64 pixel values (0..15), row-major.
     *
     * @param index the tile index, 0..tileCount-1
     * @return a copy of the 64 nibble values
     * @throws IllegalArgumentException if index is out of range
     */
    public byte[] tile(final int index) {
        if (index < 0 || index >= tileCount) {
            throw new IllegalArgumentException("VCN tile index out of range: " + index);
        }
        return tiles[index].clone();
    }

    /**
     * Returns the 16-entry backdrop sub-palette (tile value -> VGA index).
     *
     * @return a copy of the backdrop sub-palette
     */
    public int[] backdropSubPalette() {
        return backdropSubPalette.clone();
    }

    /**
     * Returns the 16-entry wall sub-palette (tile value -> VGA index).
     *
     * @return a copy of the wall sub-palette
     */
    public int[] wallSubPalette() {
        return wallSubPalette.clone();
    }
}
```

- [ ] **Step 4: Run unit tests → green; format.**

- [ ] **Step 5: Write the eoblib byte-exact IT + atlas eyeball (establishment step)**

Create `VcnAtlasIT.java`: skip-if-absent; load `BRICK.VCN` from `EOBDATA3.PAK` via `PakArchive`; parse; assert `tileCount() == 1575` (the real BRICK.VCN count — **verify this exact number during execution by printing it once**, then pin it), and assert a **SHA-256 of the flattened tiles + both sub-palettes** equals a frozen golden. Establish the golden by cross-checking against **eoblib's `VCN`** decoder (compile+run eoblib on the same file, byte-compare our tiles/sub-palettes) — same independent-oracle method as M1's CPS/eoblib check; freeze the hash only after eoblib matches. Also render an **atlas contact-sheet PNG** (all tiles laid out in a grid, mapped through `wallSubPalette` + `BRICK.PAL` via `VgaPalette`) to scratch and report its path (controller relays it as the eyeball).

- [ ] **Step 6: Run module `verify` → green; commit.**
```bash
git -C "$REPO" add beholder-importer-westwood/src
git -C "$REPO" commit -m "feat(importer): add VCN tile-atlas decoder (eoblib byte-exact)"
```

---

### Task 3: `VmpProjection` decoder

**Files:** Create `VmpProjection.java`; Test `VmpProjectionTest.java`, `VmpProjectionIT.java`.

**Interfaces:**
- Produces: `VmpProjection.parse(byte[] compressedVmp) -> VmpProjection`; `wallSetCount() -> int`; `backgroundCode(int x, int y) -> int` (0..329 grid, 22×15); `wallCode(int wallSet, int slotOffset) -> int` (into the 431-block); plus a public static `WALL_RENDER_DATA` table (25 `WallSlot` records). A `code` is `{tileIndex = code & 0x3FFF, flip = (code & 0x4000)!=0}`.

- [ ] **Step 1: Failing unit test** — synthetic VMP (background 330 codes + 1 wall-set of 431 codes = 761 codes; **little-endian**), assert `wallSetCount()==1`, a known `backgroundCode`/`wallCode`, and the flip/index bit decode. Include an assertion that would FAIL if read big-endian (e.g. a code `0x0102` stored LE as bytes `02 01` must decode to `0x0102`, not `0x0201`).

- [ ] **Step 2: Run → fail.**

- [ ] **Step 3: Implement `VmpProjection`** — with the `WALL_RENDER_DATA[25]` table (from Format facts above) as documented constants, the LE reader (with a comment: *eoblib's VMP.java reads big-endian — WRONG for PC; we read little-endian, verified against real .vmp files*), the `330`/`431` partitioning as named constants (`BACKGROUND_CODE_COUNT = 22*15`, `CODES_PER_WALL_SET = 431`, `VIEWPORT_BLOCKS_WIDE = 22`), and the `0x3FFF`/`0x4000`/`0x8000` bit masks documented (`TILE_INDEX_MASK`, `FLIP_BIT`, `LIMIT_BIT` with "seam-blend, unused for plain walls").

- [ ] **Step 4: Unit green; format.**

- [ ] **Step 5: `VmpProjectionIT`** — load `BRICK.VMP` (EOBDATA3.PAK); assert `codeCount == 2916` and `wallSetCount() == 6`; freeze a SHA-256 of all codes. Cross-check the decoded **code values** against eoblib's `VMP` **only after byte-swapping eoblib's output to LE** (or compare structurally — eoblib's BE read means its raw values differ; the reliable cross-check is that our LE decode yields `wallSetCount==6` cleanly, which BE does not, plus the wallsetviewer net-LE behaviour). Document this caveat in the test.

- [ ] **Step 6: Commit** `feat(importer): add VMP projection decoder (little-endian; 25-slot wall table)`.

---

### Task 4: `Maze` decoder

**Files:** Create `Maze.java`; Test `MazeTest.java`, `MazeIT.java`.

**Interfaces:**
- Produces: `Maze.parse(byte[] mazBytes) -> Maze`; `width()`, `height()`; `wall(int x, int y, Direction dir) -> int` where `Direction` is an enum `N,E,S,W` (or reuse `beholder-ir`'s `Facing` if already available — but `Maze` is in the importer, which does not depend on `ir`, so define a local `Direction` here). Actually **use int constants** `N=0,E=1,S=2,W=3` documented, to avoid a cross-module enum dependency.

- [ ] **Step 1: Failing unit test** — synthetic 2×2 MAZ (header `width=2,height=2,nof=4`, 6-byte header + 16 wall bytes); assert `width()==2`, and that cell (1,0)'s N/E/S/W bytes are read in that clockwise order (put distinct values 10,11,12,13 and assert `wall(1,0,N)==10 ... wall(1,0,W)==13`). Include a comment: *byte order is N,E,S,W (eoblib) — the wiki's N,S,W,E is wrong.*

- [ ] **Step 2: Run → fail.**

- [ ] **Step 3: Implement `Maze`** — documented constants: `HEADER_SIZE=6` (`width u16 + height u16 + nof u16`), `BYTES_PER_CELL=4`, direction indices `N=0,E=1,S=2,W=3` (`// clockwise from north; kyra move table {-32,+1,+32,-1} confirms this order`), `cellIndex = y*width + x`. Fail-closed on wrong length / out-of-range coords.

- [ ] **Step 4: Unit green; format.**

- [ ] **Step 5: `MazeIT`** — load `LEVEL1.MAZ` (EOBDATA3.PAK); assert `width()==32 && height()==32`; freeze a SHA-256 of the full wall grid; cross-check byte-exact against **eoblib's `Maze`** (same N,E,S,W order). Establish the hash only after eoblib matches.

- [ ] **Step 6: Commit** `feat(importer): add MAZ 32x32 grid decoder (eoblib byte-exact)`.

---

### Task 5: `LevelInfoHeader` + wall-mapping

**Files:** Create `LevelInfoHeader.java`; Test `LevelInfoHeaderTest.java`, `LevelInfoHeaderIT.java`.

**Interfaces:**
- Produces: `LevelInfoHeader.parse(byte[] infBytes) -> LevelInfoHeader`; `mazeName() -> String`; `vcnVmpName() -> String`; `wallType(int wallMappingIndex) -> int` (resolves the default table + `0xFB` overrides → the 1-based VMP wall-set, or 0 for open).

**⚠️ This task parses ONLY the `.INF` header + wall-mapping table, NOT the trigger bytecode (that is M4).** Read eoblib's `Inf.java:82-126` for the exact header layout and the `0xFB` command; document every offset. If the exact `.INF` header layout proves fiddly during execution (EOB1 vs the recon's notes), **STOP and escalate** rather than guessing — this is the one importer piece not fully pinned by the recon.

- [ ] **Steps** (TDD as above): synthetic `.INF` header with a known maze/vcnVmp name and one `0xFB` override → assert names + `wallType`. Implement with documented offsets. IT: load `LEVEL1.INF` (EOBDATA3.PAK) → assert `vcnVmpName()` equals `"brick"` (case per the real bytes — verify during execution) and `mazeName()` contains `level1`. Commit `feat(importer): add .INF header + wall-mapping parser`.

---

### Task 6: IR geometry + tileset records + assemblers

**Files (in `beholder-ir`):** Create `Facing.java`, `WallArchetype.java`, `model/LevelGeometry.java`, `model/TileSet.java`, `WestwoodToIr.java`; Test `FacingTest.java`, `WestwoodToIrTest.java`. **`beholder-ir` must gain a test-scoped dependency on `beholder-importer-westwood`** for the assembler (add it to `beholder-ir/pom.xml`) — OR put `WestwoodToIr` in the importer module to avoid the dependency direction issue. **Decision: put `WestwoodToIr` in `beholder-importer-westwood`** (it already depends on nothing problematic and naturally produces IR); `beholder-ir` depends on nothing. So: records + `Facing` + `WallArchetype` in `beholder-ir`; `WestwoodToIr` in `beholder-importer-westwood` (which gains a dependency on `beholder-ir`).

**Interfaces:**
- `Facing` enum `{NORTH, EAST, SOUTH, WEST}` with `delta(int localX, int localY) -> int[]{dCol, dRow}` per the facing→grid formulas.
- `WallArchetype` enum `{OPEN, SOLID}` (closed, extensible; `OPEN` = passable/no wall, `SOLID` = a drawn plain wall).
- `LevelGeometry` record: `int width, int height, Cell[] cells` where `Cell` is a nested record `{WallArchetype[] walls (length 4, N/E/S/W), int[] wallSet (length 4)}`; plus `irVersion` (String, e.g. `"1.0.0"`). `cell(x,y)` accessor.
- `TileSet` record: `{int tileCount, byte[][] tiles, int[] backdropSubPalette, int[] wallSubPalette, int[] backgroundCodes, int[][] wallSetCodes, int[] paletteArgb}` — a fully framework-neutral appearance pack (VCN + VMP + palette flattened).
- `WestwoodToIr.geometry(Maze, LevelInfoHeader) -> LevelGeometry` and `WestwoodToIr.tileSet(VcnAtlas, VmpProjection, VgaPalette) -> TileSet`.

- [ ] **Steps:** TDD `Facing.delta` against all 4 facings × sample local coords (assert against the recon's formulas: NORTH.delta(1,3)={1,-3}; EAST.delta(1,3)={3,1}; etc.). TDD `WestwoodToIr.geometry` mapping a 2×2 synthetic Maze+wall-mapping → `LevelGeometry` (index 0→OPEN, 1/2→SOLID). Records are coverage-exempt; `Facing`/`WestwoodToIr` are measured. Document `irVersion` semantics (semver; M2 = "1.0.0"). Commit `feat(ir): add geometry + tileset IR records and Westwood assembler`.

---

### Task 7: `beholder-render-core` module + `TileRasterizer`

**Files:** Create the module `beholder-render-core/pom.xml` (parent = reactor root; deps: `beholder-ir`, test junit+assertj); register it in the reactor `pom.xml`; create `render/Rgba.java`, `render/TileRasterizer.java`; Test `TileRasterizerTest.java`. Add `<module>beholder-render-core</module>` to the root pom.

**Interfaces:**
- `Rgba.pack(int r,int g,int b) -> int` (0xFF-alpha ARGB); `TRANSPARENT = 0`.
- `TileRasterizer.raster(byte[] tile64, int[] subPalette, int[] paletteArgb, boolean flip) -> int[]` (64 ARGB pixels; nibble 0 → fully transparent 0x00000000; else `paletteArgb[subPalette[nibble]]`; `flip` mirrors x per row).

- [ ] **Steps:** TDD a 2-tile raster: a tile with nibble 0 (transparent) and nibble n (opaque via sub-palette→palette), assert ARGB and transparency; assert `flip` mirrors column order (pixel (row,0)↔(row,7)). Document the 8-wide row constant (`TILE_WIDTH=8`). Commit `feat(render): add render-core module + tile rasterizer`.

---

### Task 8: `ViewFootprint`

**Files (in `beholder-render-core`):** Create `render/ViewFootprint.java`; Test `ViewFootprintTest.java`.

**Interfaces:**
- `ViewFootprint.visibleCells(GridPosition party, Facing facing) -> List<VisibleCell>` where `VisibleCell` = `{String label (A..Q or PARTY), int col, int row, int localX, int localY}`, in **far-to-near draw order** (row `ly=3` first … `ly=0` last). Off-grid cells (col/row outside 0..31) are included but flagged (the renderer treats an off-grid or open cell as "no wall").

- [ ] **Steps:** Define the 18-cell local layout (`ly,lx` per the Format facts) as a documented static table. TDD: for party at (5,5) facing NORTH, assert cell D (`lx=0,ly=3`) maps to `(col=5, row=2)`; cell party maps to `(5,5)`; for facing EAST assert the rotation. Assert the list is ordered far→near. Every letter's `(lx,ly)` is a documented constant. Commit `feat(render): add 18-cell view footprint + facing transform`.

---

### Task 9: `FirstPersonRenderer` (+ PNG eyeball)

**Files (in `beholder-render-core`):** Create `render/FirstPersonRenderer.java`, `render/WallSlotRasterizer.java`; Test `FirstPersonRendererTest.java`.

**Interfaces:**
- `FirstPersonRenderer.render(LevelGeometry geo, TileSet tiles, GridPosition party, Facing facing) -> int[]` — a `176*120` ARGB frame (`VIEWPORT_WIDTH=176`, `VIEWPORT_HEIGHT=120`, `VIEWPORT_BLOCKS_WIDE=22`, `VIEWPORT_BLOCKS_HIGH=15`, `TILE=8`, all documented). Algorithm (from the Format facts / kyra oracle):
  1. Build a 22×15 **tile-code buffer** (init 0 = open). Fill the **background** codes (`tiles.backgroundCodes`) into a parallel floor/ceiling buffer.
  2. For each visible cell far→near (`ViewFootprint`), for each of its walls the party can see, resolve the wall's `wallSet` (from `LevelGeometry`); if `SOLID`, look up the wall-set's codes for the matching `WALL_RENDER_DATA` slot and write them into the tile-code buffer (non-zero overwrite = painter's occlusion), honoring the slot's `flip`.
  3. Rasterize the 22×15 buffer to the `176*120` ARGB image via `TileRasterizer`: for each block, if the wall buffer has a non-zero code draw that wall tile, else fall back to the background/floor tile at that block.
- `WallSlotRasterizer` encapsulates step 2's per-slot blit (the `WALL_RENDER_DATA` geometry loop) so `FirstPersonRenderer` stays ≤8 cyclomatic per method.

**⚠️ This is the algorithmic core and the WALL_RENDER_DATA geometry is "illustrative" per the wiki — expect to ITERATE this task's exact slot math against the kyra golden in Task 10.** Implement the algorithm above via TDD with small synthetic geometries (e.g. a single SOLID front wall directly ahead → assert the N-south slot's blocks are non-transparent and the frame is 176×120), and produce a **PNG of the real level-1 viewpoint** (see Task 10 for the chosen `(cell, facing)`) to scratch as the eyeball. The pixel-exactness gate is Task 10.

- [ ] **Steps:** TDD the frame dimensions + a single-wall placement + background fallback; implement `WallSlotRasterizer` + `FirstPersonRenderer`; render the real level-1 view → scratch PNG; report the path. Commit `feat(render): add headless first-person renderer (176x120)`.

---

### Task 10: kyra pixel-golden differential (the M2 correctness gate)

**Files:** Create `beholder-render-core/.../FirstPersonGoldenIT.java`.

**Why:** the decoders are eoblib-verified, but there is **no clean-room automated oracle for the assembled first-person view** — the ground truth is the shipping renderer. This task pins our render against a **kyra `set_position` screenshot** of one deterministic plain-corridor viewpoint in level 1.

- [ ] **Step 1: Choose + capture the reference (maintainer-assisted operational step).** Pick a level-1 `(block, facing)` where all visible walls are plain brick (indices 1/2) and no doors/decorations/items are in view (a straight corridor). In ScummVM kyra: open the debugger, `set_position 1 0 <block>`, turn to `<facing>`, `show_position` to confirm, then Alt+S to save a PNG. Crop the **176×120** viewport (top-left) from the screenshot. Store it **locally, not in git** (copyrighted), path via `-Deob1.kyra.ref=<path>`. (If ScummVM's Alt+S captures a scaled surface, downscale to native 320×200 first; verify the crop is exactly 176×120.) Record the chosen `(block, facing)` in the test as documented constants.

- [ ] **Step 2: Pin palette + slot fidelity.** Render our frame for the same `(block, facing)`; compare to the kyra reference. Resolve the two known fidelity risks: (a) **palette scaling** — if colors are off, test `(v*255+31)/63` vs `v<<2` for the VGA 6→8-bit conversion and pick the one matching kyra (document the choice + why); (b) **WALL_RENDER_DATA slot geometry** — adjust any slot whose blocks are misplaced against the reference, documenting each adjustment with the observed-vs-expected reason. Iterate `FirstPersonRenderer`/`WallSlotRasterizer` until the frames match.

- [ ] **Step 3: Write `FirstPersonGoldenIT`.** Skip-if-absent on both the game data (`~/Games/...`) and the kyra reference (`-Deob1.kyra.ref`). It: builds level-1 IR (PakArchive → decoders → WestwoodToIr), renders the chosen `(block, facing)`, and asserts the render **matches the kyra reference** — first attempt an **exact 176×120 pixel match**; if a small number of pixels differ due to documented, understood boundary effects (the 0x8000 seam-blend bit), fall back to a tight tolerance (≥99.5% exact pixels AND max per-channel Δ ≤ 4) with the exact count logged and justified. Also commit a **SHA-256 of our render** as a regression ratchet (freeze after the reference match is established). Relay the render PNG + a side-by-side to the maintainer.

- [ ] **Step 4: `verify` green; commit** `test(render): kyra pixel-golden for one EOB1 first-person view`.

- [ ] **Step 5: Merge** — controller runs the final review + `--no-ff` merge (do not merge in this task).

---

## Self-Review

**Spec coverage.** M2 goal (render one EOB1 first-person view through the real IR, kyra-differential): T1 (shared container + Format80 hardening) + T2–T5 (VCN/VMP/MAZ/INF decoders, eoblib-verified) + T6 (IR geometry + tileset records + assembler) + T7–T9 (render-core: rasterizer, footprint, renderer) + T10 (kyra pixel-golden). Three modules per the architecture; in-memory IR records only (no JSON — M5); `render-core` headless (no libGDX — M3). Every task documents its magic numbers per the Global Constraints.

**Placeholder scan.** T1–T8 carry complete code or exact synthetic-test specs. T9–T10 are deliberately specified as **algorithm + exact data tables + oracle-driven TDD** rather than fabricated pixel-loop code, because (a) the WALL_RENDER_DATA geometry is "illustrative" per the wiki and must be tuned against the kyra golden, and (b) fabricating unverified pixel math would bake in bugs the differential exists to catch. The exact tables, viewport constants, algorithm, and verification method are all concrete — this is a specification with the load-bearing data present, not a "TODO." Two real-data numbers (BRICK.VCN `tileCount`, the exact `.INF` name casing) and the golden hashes are derived-during-execution outputs, established and pinned per the named procedures (same pattern as M1).

**Type consistency.** `WestwoodContainer.decompress(byte[])` is consumed identically by `CpsImage`/`VcnAtlas`/`VmpProjection`. `Facing`/`WallArchetype`/`LevelGeometry`/`TileSet` flow from T6 into T8–T9 with matching shapes. `VIEWPORT_*`/`WALL_RENDER_DATA` constants are defined once (T3 for the data, T9 for the viewport) and referenced consistently.

**Dependency direction.** `beholder-ir` depends on nothing; `beholder-importer-westwood` depends on `beholder-ir` (for `WestwoodToIr`); `beholder-render-core` depends on `beholder-ir`. No cycle. `WestwoodToIr` lives in the importer to keep `beholder-ir` dependency-free.

**Risk callouts (for the reviewer/executor).** (1) T5 `.INF` header is the least recon-pinned piece — escalate if fiddly. (2) T3 VMP endianness — must be LE, not eoblib's BE. (3) T9–T10 renderer fidelity is the real M2 risk and will iterate against kyra; the plan front-loads all the exact data so the iteration is bounded. (4) T10 depends on a maintainer-captured kyra reference (the one human-in-the-loop step, by the nature of a differential-vs-original test).
