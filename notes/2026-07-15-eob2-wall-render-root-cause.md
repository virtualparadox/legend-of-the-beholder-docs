# EOB2 wall-render root cause — the missing hardcoded standard-wall table

> **Running research note (article seed).** Part of the reverse-engineering log for the
> shared-IR EOB engine. Findings here are destined for the write-up; community
> contributors are credited inline and must be credited in any article.
> **Date:** 2026-07-15. **Status:** root cause identified; fix pending (next milestone).

## How we got here (the diagnostic journey)

Phase 2 (EOB2 validation) closed green: geometry + events + a byte-stable lossless
round-trip, all through the *shared* IR/engine/render. But Phase 2's EOB2 **render
fidelity** was only ever "maintainer eyeball" — the pixel-exact `kyra` golden was
**deferred** (no built ScummVM). Playtesting immediately exposed what that gap hid.

Booting real EOB2 levels from IR (`extract → IR → boot → GameWindow`) revealed:
1. Every level rendered as a **brown dungeon**, and the walls came out as **fragments**
   (EOB2 `LEVEL1`) or **nothing but floor + ceiling** (EOB2 `LEVEL4`), with
   **invisible walls** (collision blocked where no wall drew).
2. A top-down ASCII map dumped from the IR showed a **coherent maze** (corridors,
   rooms, doors, ~80–90% surrounding solid rock) — so the geometry *layout* decodes;
   the breakage is in the **wall tile** layer, not missing geometry.
3. The player-facing "first level is a *forest*" turned out to be **`LEVEL4`**, not
   `LEVEL1` (`LEVEL1` genuinely *is* a dungeon → its `dung` tileset is correct). The
   game folder has dedicated `FOREST.VCN/VMP/PAL/CPS` but **no `FOREST.MAZ`**, so the
   forest maze is `LEVEL4.MAZ` rendered with the `FOREST` tile set.
4. Attempting to extract `LEVEL4` **threw**: `decoration overlayId 'forest' is absent
   from pack 'dung'` — because `Eob2Extractor` **hardcodes the `dung` appearance pack
   for every level** (the deliberately-deferred M2 Task 4 "asset-name hardcoding"; the
   deferral's own rationale — "no differential test with only LEVEL1" — predicted this
   exact break the moment a second-look level arrived).

The smoking-gun stat: on EOB2 wall edges, the per-edge `wallSet` index is almost
entirely `0` — `LEVEL1`: `{0: 3323, 1: 49}`, `LEVEL4`: `{0: 3742, 1: 6}`. So the
"fragments" *are* the handful of non-zero walls; everything mapped to `0` drew nothing.

## Root cause (confirmed from the EOB Revealed wiki)

`~/Documents/EOB-DOCS` (the "EOB Revealed" wiki archive) named it exactly. From the
**`wall mapping index`** thread on the `EOB.MAZ` page — a discussion literally about
the **EOB2 `LEVEL4` forest** MAZ:

- **inigomartinez:** wall-mapping index `0` = no wall, `1–5` = normal walls, `>5` =
  special walls defined in the INF's `0xFB` table.
- **JackAsser:** the low indices are **hardcoded static tables in the engine code**,
  **not** in the INF — *"Only values `>0x17` are defined by the INF-file."* The
  standard mapping: `00 = air`, `01 = solid wall 1`, `02 = solid wall 2`,
  `03–07 / 08–0c / 0d–11 / 12–16` = the four door variants (5 opening stages each).
  His INF-reader **prefills** before reading the INF:
  ```
  wallTypeInit[]      = {0,1,2,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3,3};  // idx 0..0x16
  wallEventMaskInit[] = {0,0,0,1,1,1,1,1,0,0,0,0,0,1,1,1,1,1,0,0,0,0,0};
  wallFlagsInit[]     = {1,4,4,0x2c,0x2c,0x2c,0x2c,0x19, ... };
  ```
- **The OP (badut):** *"most cells have 1's or 2's"* in the `LEVEL4` MAZ — i.e. the
  overwhelming majority of walls are the **hardcoded standard walls (index 1 / 2)**.

**Our bug:** the EOB2 decode (`Eob2InfScript` / `Eob2InfGeometry` → `WallMappingTable`)
reads **only** the INF `0xFB` table (indices `>0x17`) and **omits the hardcoded
standard-wall prefill** (`0..0x17`). So MAZ indices `1` and `2` — the bulk of every
level's walls — resolve to wall-set `0` and **draw nothing**. `LEVEL1` still shows the
few INF-defined special walls (49 → "fragments"); `LEVEL4`, being almost all standard
walls, shows essentially nothing ("floor + ceiling only"). The `SOLID` archetype is
still derived (collision works) → the walls are *there but invisible*.

## The fix (two parts, next milestone)

1. **Primary — add the hardcoded standard-wall prefill (0..0x17)** to the EOB2
   wall-mapping decode, *before* applying the INF `0xFB` overrides (per JackAsser's
   `wallTypeInit` / event-mask / flags tables). Confirm the exact EOB2 table sizes,
   door variants, and the VMP wall-type count against `kyra`'s `DarkMoonEngine` +
   `eob.vmp`/`eob.inf` (the wiki notes "at least applicable to EoB I"; verify EOB2).
   Cross-check EOB1's own decode — it may share the same gap or already prefill.
2. **Secondary — per-level appearance-pack selection.** Replace `Eob2Extractor`'s
   hardcoded `dung` pack with the level's own tile-set/decoration names from the INF
   (`gfxName()` / `decorationOverlays()`), so `LEVEL4` → `FOREST`, `LEVEL1` → `dung`.

**Validation references (clean-room, non-GPL):** `eob.vmp` (wall projection: the
16-bit tile code `limit:1 / flipped:1 / blockIndex:14`, 25 wall positions), `EOB.MAZ`,
`eob.inf`, and the per-level annotated decodes `files/levelN.txt` (incl. `level4.txt`,
the forest). `kyra` is secondary confirmation.

## Credits (for the article)

The root cause and the exact standard-wall tables came from the **EOB Revealed** wiki
(`~/Documents/EOB-DOCS`) — specifically the `wall mapping index` and `limit flag in the
tile code` threads. Community contributors to cite: **JackAsser** (the hardcoded
standard-wall tables + INF-reader prefill; the `limit` flag being a no-op), **inigomartinez**
(the depth-4 wall-visibility scan + the 25 wall-draw positions + the index semantics),
and **badut** (the EOB2 `LEVEL4` MAZ questions that framed it).

## Meta-lesson (for the article too)

Phase 2's structural + round-trip tests were all green while the render was badly
wrong — because "the reload renders *identically* to the fresh decode" proves
*determinism*, not *correctness*, and the correctness oracle (a pixel/layout reference)
was deferred. A consistently-wrong render passes every test we had. The layout
reference we lacked existed all along in the wiki's `files/levelN.txt` + the community's
map knowledge; wiring a real golden against it is the durable fix for the test gap.

## Outcome (2026-07-15) — fixed + validated live

Fixed on `feat/eob2-render-fidelity`, merged to `main` at `2354408`:
- **`Eob2InfScript`** now seeds the 25-entry standard-wall prefill (indices 0..24 → vmp
  `0,1,2, 3×20, 4, 5`; kyra `resetWallData`-confirmed, EOB Revealed wiki) before the INF
  `0xFB` entries override it — so MAZ indices 1/2 (the bulk of walls) resolve to real
  wall-sets and draw.
- **`Eob2Extractor`** selects the appearance pack PER LEVEL from `gfxName()`/
  `decorationOverlays()`/`doors()` — so `LEVEL4`/forest loads with the `FOREST` tileset
  (it previously threw), plus a clear guard for a level with zero decoration overlays.
- **`Eob2WallRenderIT`** — the correctness oracle Phase 2 lacked: it renders a real
  SOLID-wall-facing view and asserts wall pixels are present (pixel-diff vs a
  wall-set-zeroed twin through the *same* render path, so decoration/door pixels cancel).
  A consistently-wrong render can no longer pass green. The render layer was unchanged;
  EOB1 was untouched.

**Live validation — the "map-as-golden" vindicated.** The EOB2 forest (`LEVEL4`) boots from
the *shared* IR and renders correctly with the `FOREST` tileset; the maintainer walked it and
navigated **from memory** to the **Darkmoon Temple** (marker `A` on the community map). The
forest's geometry layout matches the real game. The correctness reference Phase 2's deferred
pixel-golden lacked was there all along in the community's map knowledge + `files/level4.txt` —
exactly the gap this milestone closed with `Eob2WallRenderIT`.

**Deferred follow-ups:** EOB1's own prefill is short by 2 entries (indices 23/24; §E.1 —
kyra applies 25 to both games, our `InfScript` carries 23; likely benign, no known EOB1 level
hits those indices); the zero-doors door-leaf placeholder duplicates the first decoration sheet
on export (m2 — inert, now exercised end-to-end by the LEVEL4 render IT and proven harmless);
`specialType`/`flags` presets for future door-knob/switch interactivity (§E.3).
