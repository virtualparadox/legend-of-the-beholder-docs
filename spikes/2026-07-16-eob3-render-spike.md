# EOB3 render spike — proving the vertical (walkable + faithful) on real bytes

*Date: 2026-07-16. Throwaway spikes (code lived in a scratch dir, never committed; no game-art
images committed — SSI assets stay out of the repo per the §9 legal model). This note preserves the
hard-won technical facts so the production implementation inherits them.*

## What was proven

Two throwaway spikes, both run end-to-end against the maintainer's real `EYE.RES`:

1. **Spike 1 — EOB3 geometry on the existing (Westwood) projection.** A minimal clean-room `.RES`
   reader pulled a real 32×32 Mausoleum occupancy grid; an occupancy→per-edge derivation produced a
   shared-IR `LevelGeometry`; it rendered through the **existing** `FirstPersonRenderer` with a
   **borrowed EOB1 `brick` `AppearancePack`**. Result: a coherent, walkable first-person dungeon
   (EOB1-looking walls, correct EOB3 layout). Confirms: `.RES` reader works, occupancy→edge works,
   the shared IR + engine + render pipeline absorb EOB3 with **zero engine changes** and a borrowed
   look.

2. **Spike 2 — real EOB3 art with an EOB3-native projection.** Decoded the real Mausoleum wall
   sprites (`1.10` VFX) + palette, and composed a first-person frame with a **reconstruction of
   EOB3's own whole-sprite projection**. Result: an authentic EOB3 Mausoleum view (grey stone,
   receding side walls, front walls at three depths, tiled floor). Answers the open fear ("EOB3's
   different projection will look weird") — **it looks right, not weird.**

**Net:** the whole vertical is de-risked end-to-end — container, geometry, walkability, the `1.10`
VFX + `PAL` decoders, and the EOB3-native wall projection all work on real bytes.

## Technical facts confirmed on the real `EYE.RES` (for the production importer)

### `.RES` container (matches the recon; verified on real bytes)
36-byte header (`u32 first_directory_block @0x18` = `0x24`), linked 644-byte directory blocks (`u32
next@0`, `u8 attr[128]@4` with `==1`=free, `u32 entry_header_index[128]@0x84`), 12-byte entry
headers (`u32 size@8`), verbatim little-endian payloads. Resource number = `block*128 + slot`.
`data_attributes & 0x10000000` = placeholder (no payload). 2449 resources total.

### ROED (resource 0) name→number
`u16 bucketCount@0`, `bucketCount × u32` chain offsets (relative to blob start), each non-zero chain
a run of length-prefixed strings (`u16 len` **including the trailing NUL**, then `len` bytes),
**alternating key,value**, terminated by `len==0`. **The value is the resource number as ASCII
decimal text** (`std::stoi`). This is how a level's assets are resolved by name.

### Mausoleum resources (parsed from ROED)
- Map "Mausoleum 1" = **#5** (1024-byte occupancy grid, values `{0,1,255}`; `0xff`=floor,
  `0x00`/`0x01`=solid). The 14 maps are #5–#18.
- "Mausoleum walls" = **#253** (`1.10` VFX, 19 shapes). "Mausoleum palette" = **#386** (`PAL`, 16
  stone-grey colours). "Fixed palette" = **#335** (256 colours). The four wall sets: Mausoleum #253,
  Forest #228, Ruins #236, Marble #297. Naming convention: `"<Area> walls"` / `"<Area> palette"`;
  beware `"<Area> view palette"` is a *different* (cutscene) palette.

### `1.10` VFX shape codec (confirmed, colours correct)
Magic `'1.10'`(4) + `u32 shapeCount` + directory of `shapeCount × {u32 offset, u32 color}`. **The
directory offset is relative to the START OF THE RESOURCE**; each shape has a **24-byte subpicture
header** (`u16 boundsY=height-1 @0`, `u16 boundsX=width-1 @2`; the other 20 bytes are ignorable
duplicate-bounds padding, **no** hotspot/origin — placement comes from the draw call). Pixel stream
starts at `offset+24`, per-scanline RLE:
- `marker==0` → end of line; `marker==1` → next byte = transparent-skip count;
- `marker` even `≥2` → run of `marker>>1` of the next byte; `marker` odd `≥3` → literal of
  `marker>>1` following bytes.
- **Pixel index 0 = transparent.**

### `PAL` + DAC banking (the colour crux)
`u16 numColors@0`, `u16 colorArrayOffset@2` (=26), 11 fade-index `u16`s, **RGB at offset 26**, 3
bytes/colour, **6-bit VGA → 8-bit via `<<2`**. **Wall-shape pixel bytes are ABSOLUTE DAC indices**,
not 0-based: the wall palette (#386, 16 colours) is banked at **DAC 0xB0** (`kFirstColor = {0x00,
0xB0, 0xC0, 0xE0, 0xB0}`, region 1 = walls). So build a 256-entry palette = Fixed #335 at 0x00,
overlaid by walls #386 at 0xB0; sprite byte → that absolute index directly (0 = don't draw; a lone
`0x0A` = fixed-palette black outline). Getting this base wrong makes every wall pixel land on an
empty palette slot (transparent) — the first colorised dump was blank until the 0xB0 base was
applied.

### Mausoleum wall-set inventory (#253, 19 shapes; w×h)
Backdrop **176×120** (floor+ceiling, the "empty view", shape 18). Front walls **128×96 / 80×59 /
48×37** (near/mid/far; two art variants each). Side walls **24×120 / 24×95 / 16×59 / 8×35** (nearest
→ farthest; left/right variants; `3/9` are byte-identical, others differ). Viewport = **176×120**
(the left region of the 320×200 screen; HUD ≥ x176).

### EOB3-native projection — reconstruction (the one piece not in the oracle)
The exact `(cell,depth,side)→shapeIndex` and screen `x,y` tables live in **EOB3 SOP bytecode (the
dungeon-draw class, res 2409)** — `thirdeye` executes the bytecode and has no static tables; DH by
contrast draws walls via *native* functions. thirdeye gives the primitives: `draw_bitmap(page,
table, number, x, y, scale, flip, fade_table, fade_level)`; scale 256=1:1/128=½/64=¼; flip bit-1 =
X-mirror (right-hand walls); painter far-to-near; per-column clip. A working reconstruction that
looks authentic: draw backdrop (18) at (0,0); front wall at the nearest solid cell ahead —
128×96@x24,y7 / 80×59@x48,y20 / 48×37@x64,y26 (centred, bottom on floor-lines {119,103,79,63}); side
walls per depth at left-x {0,24,48,64} (widths {24,24,16,8}), right = mirror, drawn when the lateral
neighbour is solid, deepest→nearest. **Production needs the exact tables** — either RE the dungeon
draw handler once and bake `(cell→shape,x,y,scale,flip)` into IR/pack data, or run a mini AESOP
interpreter. The reconstruction is good enough for a walkable first-cut and to eyeball-tune against
DOSBox ground truth.

## Milestone implications
- **M-EOB3.1** (container reader + occupancy→IR geometry + `Eob3Extractor` → walkable third game) is
  low-risk; walkability is free once per-edge geometry is emitted.
- **M-EOB3.2** (faithful EOB3 render) is a **new EOB3-native whole-sprite projection path** (as in
  spike 2), **not** the big render-core parameterisation — a clean parallel renderer fed by the
  `1.10` VFX + `PAL` decoders, with the placement tables baked from the dungeon draw handler.
- **M-EOB3.3** — the AESOP event lift (the real "one IR" thesis test).

*Credits (for the article): the projection/codec facts were extracted clean-room from Bret Curtis'
`thirdeye`; format authority cross-checked against John Miles' original AESOP source, Mirek Luza's
DAESOP, and Darkstar's ReWiki pages.*
