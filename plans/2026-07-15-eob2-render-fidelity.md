# EOB2 render-fidelity fix — standard-wall prefill + per-level pack selection

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development. Steps use checkbox (`- [ ]`) syntax.

**Goal:** Make real EOB2 levels render their walls (they currently draw as fragments / floor+ceiling only), and let non-`dung` levels (e.g. `LEVEL4`/forest) load at all — by adding the hardcoded standard-wall prefill the EOB2 decode omits, and selecting the appearance pack per level instead of hardcoding `dung`.

**Architecture:** Two decode-side fixes, no render-layer change (the render is already wall-set-count-agnostic — confirmed: `DUNG.VMP`=6 wall-sets, `FOREST.VMP`=2). (1) Seed `Eob2InfScript`'s wall-mapping table with the 25-entry standard-wall defaults (indices 0..24) before the INF `0xFB` entries override it — mirroring EOB1's `InfScript.buildDefaultWallMappings()`. (2) Make `Eob2Extractor` derive asset names + pack id from `script.gfxName()`/`decorationOverlays()`/`doors()` instead of `DUNG`/`BROWN`/`DOOR1` literals. Plus a **correctness-oracle render test** that would have caught the bug (Phase 2's structural + round-trip tests were all green while the render was wrong — round-trip proves determinism, not correctness).

**Tech Stack:** Java 21, Maven reactor, JUnit5+AssertJ, JaCoCo. Detailed root cause + exact tables + file:line gaps: **`.superpowers/sdd/eob2-render-fix-spec.md`** (the implementer brief). Background + attribution: `legend-of-the-beholder-docs/notes/2026-07-15-eob2-wall-render-root-cause.md`.

## Global Constraints

- **JDK 21**: `export JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home`; `mvn spotless:apply` then `mvn verify`; `-pl <m> -am` or root verify (stale `~/.m2` `-am` trap).
- **All `CODING_STANDARDS` gates green (`-Werror`)**: one `return`/method (no early returns), no `instanceof`, no `switch`-yield, cyclomatic ≤8, NPath ≤120, ≤5 params, ≤18 methods/≤12 fields per class, CBO ≤10, no `return null`, `final` params, braces, ≤140-char lines, no `System.out`, full Javadoc on public/protected, every magic value a named `ALL_CAPS` constant + a what/why comment.
- **Coverage: JaCoCo ≥ 0.90 branch AND line, Surefire (unit) only** — new decode logic (the prefill) is NOT exempt and must be branch-covered by unit tests; skip-safe ITs don't count.
- **Clean-room; never commit game data:** real-data ITs are `assumeTrue`-skip-safe; real-level artifacts (JSON/assets/PNG) go to git-ignored `target/`. Facts derive from `~/Documents/EOB-DOCS` (clean wiki) + kyra (read-only oracle); credit the wiki contributors in notes, never copy code.
- **Render layer must NOT change** (`beholder-render-core`/`FirstPersonRenderer`/`VmpProjection`): the fix is decode-side only. If a task finds itself editing the render, stop — that is a finding.
- **Process:** TDD per task, fresh implementer + independent review per task, whole-branch review, `--no-ff` merge on the maintainer's verdict (this milestone: the maintainer tests the forest first). Branch `feat/eob2-render-fidelity`, no worktrees.

---

## File Structure

| File | Change | Task |
|---|---|---|
| `beholder-importer-westwood/.../westwood/Eob2InfScript.java` | Seed `WallDecoScan.wallMappingTable` (line ~726) with the 25 standard-wall defaults before INF `0xFB` entries override; mirror EOB1 `InfScript.buildDefaultWallMappings()` (531-581) with `Eob2WallMapping`'s 4-field shape. | 1 |
| `beholder-importer-westwood/.../westwood/Eob2InfScriptTest.java` (or a new peer) | Unit-cover the prefill: index 1→vmp1, 2→vmp2, 3..22→vmp3, 23→vmp4, 24→vmp5; an INF `0xFB` entry overrides its index. | 1 |
| `beholder-app/.../app/Eob2Extractor.java` | Replace hardcoded `DUNG`/`BROWN1`/`BROWN2`/`BROWN.DEC`/`DOOR1`/`"dung"` literals with `script.gfxName()`/`decorationOverlays()`/`doors()`-derived names; handle zero-doors; fix the stale `UNMAPPED_WALL_SET` Javadoc (lines 110-116). | 2 |
| `beholder-app/.../app/Eob2ExtractorTest.java` + `Eob2ExtractIT.java` | LEVEL1 still → `dung`; LEVEL4 → `forest` and now extracts without throwing. | 2 |
| `beholder-app/.../app/Eob2WallRenderIT.java` (**create**) | The correctness oracle (skip-safe, real data): render a SOLID-wall-facing viewpoint of a real level and assert **wall pixels are present** (would fail on the pre-fix empty wall-mapping). Cover LEVEL1 (dung) and LEVEL4 (forest). | 3 |

**Interfaces (verified in the fix spec):** `Eob2InfScript.hasWallMapping(int)`/`wallMapping(int) -> Eob2WallMapping(vmpIndex, decIndex, specialType, flags)`; `gfxName()` (→ e.g. `"forest"`), `decorationOverlays()`, `doors()`; `NO_DECORATION_DEC_INDEX` sentinel (-1). EOB1 template: `InfScript.buildDefaultWallMappings()` + `DEFAULT_WALL_TYPE`/`DEFAULT_EVENT_MASK`/`DEFAULT_FLAGS`.

---

### Task 1: Standard-wall prefill in `Eob2InfScript` (the primary fix)

**Read first:** `.superpowers/sdd/eob2-render-fix-spec.md` §A.2 (the exact 25-entry table + why), §B.2, §C.1 (the exact gap + fix shape). Read EOB1's `InfScript.buildDefaultWallMappings()` (lines 531-581) + its `DEFAULT_WALL_TYPE` array (182) as the template to MIRROR (not reuse — `Eob2WallMapping`'s shape differs).

**The exact prefill (25 entries, indices 0..24):** vmpIndex = `{0, 1, 2, then 3 for indices 3..22 (20×), 4 for 23, 5 for 24}`; `decIndex = NO_DECORATION_DEC_INDEX` (-1); `specialType`/`flags` = a documented `0` placeholder (nothing downstream reads them yet — fix-spec §E.3). Seed these into `WallDecoScan.wallMappingTable` (line ~726, currently a bare `new HashMap<>()`), then let the existing `assignWall`/`parseWallDecoEntry` `put()`s (738-743) override any index an INF `0xFB` entry defines (last-write-wins, exactly like EOB1). Index 0 may be seeded or omitted (both call sites special-case `wallIndex==0` before consulting the map — §C.1).

- [ ] **Step 1 (RED):** unit test on `Eob2InfScript` (synthetic or a small fixture): after parse, `wallMapping(1).vmpIndex()==1`, `wallMapping(2).vmpIndex()==2`, `wallMapping(3).vmpIndex()==3`, `wallMapping(22).vmpIndex()==3`, `wallMapping(23).vmpIndex()==4`, `wallMapping(24).vmpIndex()==5`; and a synthetic INF `0xFB` entry at some index (e.g. 27) resolves to ITS vmpIndex, and an INF entry that overrides a defaulted index (e.g. 3) wins. Assert it fails today (empty map → `hasWallMapping(1)==false`).
- [ ] **Step 2:** implement the prefill seed (mirror EOB1's `buildDefaultWallMappings`), with a named `DEFAULT_WALL_MAPPING_COUNT = 25` + the vmp-index default array (each magic value a named constant/table + what/why comment citing the fix spec / kyra `resetWallData`).
- [ ] **Step 3:** run the unit test → GREEN; `mvn -q -pl beholder-importer-westwood -am verify` green (jacoco covers the new prefill branch).
- [ ] **Step 4:** also correct `Eob2Extractor.java`'s stale `UNMAPPED_WALL_SET` Javadoc (lines 110-116) which asserts the wrong "EOB2 carries no built-in defaults" premise. Commit.

### Task 2: Per-level appearance-pack selection in `Eob2Extractor` (so LEVEL4/forest loads)

**Read first:** fix-spec §C.2 (the exact literals to replace + the accessors to consume + the on-disk casing note). LEVEL4.INF's `gfxName()` already returns `"forest"`; on-disk files are upper-case (`FOREST.VCN/VMP/PAL/CPS/DEC`), pack id is lower-case (`"dung"`/`"forest"`).

- [ ] **Step 1 (RED):** extend `Eob2ExtractorTest`/`Eob2ExtractIT`: LEVEL1 → pack id `"dung"` (unchanged); LEVEL4 → pack id `"forest"` AND `extract` does NOT throw (today it throws `decoration overlayId 'forest' absent from pack 'dung'`).
- [ ] **Step 2:** replace the hardcoded `PALETTE_FILE`/`TILE_ATLAS_FILE`/`TILE_PROJECTION_FILE`/`DECORATION_SHEET_*`/`DECORATION_DAT_FILE`/`DOOR_SHEET_FILE`/`APPEARANCE_PACK_ID` literals with names derived from `script.gfxName()` (→ `FOREST.VCN`/`FOREST.VMP`/`FOREST.PAL`, id `forest`), `script.decorationOverlays()` (the level's real `(cps, dec)` pairs), and `script.doors()` (door sheet name(s)). **Handle a level that declares zero doors** (LEVEL4 may — fix-spec §E.4): skip loading any door sheet rather than erroring. Keep the `AppearancePackBuilder#build` call shape.
- [ ] **Step 3:** `mvn -q -pl beholder-app -am verify` (root-ish — the extractor is downstream). LEVEL1 unchanged, LEVEL4 extracts. Commit.

### Task 3: The correctness-oracle render test (`Eob2WallRenderIT`) — would have caught the bug

**Read first:** fix-spec §D (render is unchanged) + the existing `Eob2BootWalkIT`/`Eob2TriggerIT` for the skip-safe real-data + render wiring (`GameSession.tick()` / `FirstPersonRenderer`, `RecordingFrameSink`, the `GAME_FOLDER`/`assumeTrue` guard).

**Intent:** the test that Phase 2 lacked — a *correctness* oracle, not a determinism one. Render a viewpoint whose facing edge is `SOLID` (a wall directly ahead) for a real level, and assert the frame **contains wall pixels** — i.e. it would FAIL on the pre-fix all-zero wall-mapping (floor+ceiling only). Pick a robust, non-vacuous metric (e.g. the central wall band has a meaningful count of pixels that are neither the ceiling nor the floor fill, or the frame differs from a synthetic all-zero-wall-mapping render of the same view). Do it for **LEVEL1** (dung) and **LEVEL4** (forest — proves both fixes end-to-end). Write before/after-style PNGs to git-ignored `target/` for the maintainer eyeball.

- [ ] **Step 1:** write `Eob2WallRenderIT` (skip-safe): for LEVEL1 and LEVEL4, extract → boot/render a solid-wall-facing viewpoint → assert wall pixels present (a metric that is ~0 pre-fix). Confirm it RUNS (real data present) and passes.
- [ ] **Step 2:** `mvn -q verify` (root). Confirm the new IT runs (not skipped) and passes; whole suite green. Write the PNGs. Commit (no `target/` artifacts staged).

---

## Self-Review

- **Spec coverage:** primary prefill (fix-spec §C.1) → Task 1; per-level pack (§C.2) → Task 2; render unchanged (§D) → respected (no render edit); the correctness-oracle gap (the meta-lesson) → Task 3. Follow-ups (§E.1 EOB1 truncated prefill, §E.2 door-band gap, §E.3 specialType/flags) → out of scope, tracked in the research note.
- **Placeholder scan:** the exact prefill table + the exact literals-to-replace live in the fix spec (the implementer brief); each task names its fix-spec section. `specialType`/`flags`=0 is a *documented* placeholder, not a vague one.
- **Type consistency:** `Eob2WallMapping(vmpIndex, decIndex, specialType, flags)`, `gfxName()`, `decorationOverlays()`, `doors()`, `hasWallMapping`/`wallMapping` used consistently with the fix spec + existing code.
