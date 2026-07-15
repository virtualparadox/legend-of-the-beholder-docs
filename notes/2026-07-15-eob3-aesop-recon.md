# EOB3 / AESOP recon spike

*Date: 2026-07-15. Author: clean-room recon, four parallel agents cross-checking three
independent descriptions of the same artifacts.*

> **Method (clean-room).** Four isolated recon agents read the oracle stash so the
> coordinator never ingested GPL / original-source expression; each reported facts (formats,
> offsets, enums, algorithms) in prose. The four vantages deliberately overlapped on the
> critical questions (`.RES` layout, event model, geometry) so that the **GPL reimplementation**
> (`thirdeye`, Bret Curtis), the **original Miles Design source** (`aesop-package`, the DAESOP
> public-domain disassembler), and the **real shipped bytes** (`EYE.RES`, `HACK.RES`, `OPEN.RES`,
> `SAMPLE.RES`) cross-validate each other. Where they disagreed, the empirical bytes and the
> original source win over a single reading. Agreement was near-total.

---

## Bottom line up front

1. **The `.RES` container is one format across the whole AESOP family** — EOB3, both Dungeon
   Hack archives, and the DAESOP sample all parse with a single reader (36-byte header +
   linked 644-byte directory blocks + 12-byte entry headers + verbatim payloads; little-endian,
   packed, no archive-level compression, no content-type tag). Empirically: 0 out-of-bounds
   entries, 0 payload overlaps across 5,614 resources in four files. **Low-risk, highly tractable.**

2. **EOB3 geometry is AESOP-native, not Westwood** — but the *maze grid keeps EOB's 32×32
   dimensions*. Levels are 14 `MAP` resources of exactly 1024 bytes (32×32×1), a **single scalar
   per cell** occupancy grid (not per-face wall ids like a Westwood `MAZ`). **No `VCN`/`VMP`, no
   `CPS`/Format80, no `FORM`/`ILBM`** anywhere in the geometry path. Wall/sprite art is an
   AESOP `1.10` VFX shape-table RLE; palettes are AESOP `PAL` resources banked into DAC windows.
   Of our Westwood decoder surface, **only the CPS/LCW decompressor is reusable**, and only for
   the char-gen backdrops — never for in-game dungeon art.

3. **The render projection is code, not data — this is the headline risk.** For EOB1/2 we decode
   `MAZ`+`VCN`+`VMP` and *we* project the 3D view. EOB3 ships no viewport-map data: the wall-slice
   placement, depth scaling (three tiers 256/128/64 = full/½/¼), and left/right mirroring live
   inside the game's own SOP **bytecode**, executed by the AESOP VM through a generic
   `draw_bitmap` primitive. **Walkability is nearly free** (a 1-byte occupancy grid with
   `coord & 31` wrap); **wall-render fidelity is the cost**.

4. **AESOP fits the shared IR at the event/message seam, but forces a family-coherent extension.**
   `notify(object, message, event_type, parameter)` ≈ `TriggerBinding`; the match modes
   (equals / `≥` for timers / `-1` wildcard) ≈ `Condition`/`Reason`; `RT_execute(client, message)`
   ≈ `Action`. **But** EOB1/2 dungeon logic is *declarative* (INF trigger tables), whereas EOB3
   dungeon logic is *imperative bytecode* over grid arrays — spatial triggers (step on a tile,
   pull a lever) are fused into handler code, with no declarative trigger table. The IR must be
   rich enough to hold **both** a declarative row (EOB1/2) and a lifted/handler-referencing action
   (EOB3), never an EOB3 silo.

5. **Dungeon Hack shares the path down to the codec layer.** Same container (one reader), same VM,
   same object model. It diverges at: name resolution (EOB3 embeds a name dict; DH uses external
   `HACK.TBL`/`OPEN.TBL`), bitmap codec (different RLE), music (EOB3 has XMI, DH none), and —
   crucially — **resource 3, the native-function table**: DH renders via *native* functions
   (`draw_walls`, `init_viewspace`, `build_clipping`) and has **no stored maze grids** (procedural,
   ships `MAZE.EXE`). Swapping resource 3 retargets the engine between games.

---

## Q1 — Is EOB3's geometry/tileset Westwood-like or AESOP-native?

**Verdict: AESOP-native storage with EOB's 32×32 grid dimensions. A new decoder surface, not a
reuse of the Westwood decoders (except CPS for char-gen).**

### The maze grid
- `EYE.RES` declares **14 `MAP` resources**, info strings `map:art\maps\<name>.lbm,32,32,58,43,23,17`:
  `mauslvl1/2`, `magelvl1-4`, `frstrail`, `ghqrtr`, `templvl1-4`, `tmplqrtr`, `graveyrd`.
- The trailing numbers are the **`mapcomp` (Miles' AESOP Cartesian-map compiler) sampling spec**:
  `xsize=32, ysize=32, org_x=58, org_y=43, cell_x=23, cell_y=17` — i.e. the level is authored as a
  256-colour DPaint `.LBM`, and the compiler samples it on a grid starting at pixel (58,43) with
  cell pitch (23,17), writing **one byte per cell = (sampled palette index − 1)**. *(This resolves
  the empirical agent's guess that the trailing numbers were wall/decoration counts — the
  compiler source is authoritative: they are LBM sampling origin + pitch.)*
- Corroborated empirically: `EYE.RES` contains **exactly 14 resources of exactly 1024 bytes**
  (= 32×32×1), low-cardinality per-cell (e.g. values {0, 1, 255}). Stored **raw, no header** —
  dimensions are convention (`LVL_X = LVL_Y = 32`, `coord & 31` wrap in the movement code).
- Semantics: a **scalar occupancy grid** (`0x00 = wall`, `0xff = open floor`). This is
  *fundamentally simpler* than a Westwood `MAZ`: there are **no per-face wall ids** — a cell is
  wall-or-not, one wall tileset per level, and every per-face feature (doors, buttons, alcoves,
  decorations, monsters, items) is externalised into separate SOP **objects** linked into a
  parallel `lvlobj` grid (three planes indexed `plane*2048 + y*64 + x*2` = monsters / floor items
  / wall features).

### Tileset / bitmap codec
- **No `VCN` (wall tile-blocks) and no `VMP` (viewport block-map) exist** — confirmed by byte scan
  (`VCN`=0, `CPS`=0, `ILBM`=0 across the archives). The 132 `FORM` hits are IFF **XMI music**, not
  images.
- In-game art is an **AESOP `1.10` VFX shape table** (312 resources in `EYE.RES`, matching the 312
  `.BMP` info strings, each beginning with the ASCII tag `1.10`): a `u32 shapeCount`, a directory
  of `{u32 offset, u32 color}`, then per-shape a 24-byte subpicture header (`height-1`, `width-1`)
  and a **per-scanline RLE** (end-of-line / transparent-skip / run / literal tokens; palette index
  0 = transparent). This is **not** Format80/LCW and **not** raw.
- Palettes are an **AESOP `PAL` resource** (`u16 numColors`, `u16 colorArrayOffset`, ~12 fade-index
  array offsets, RGB at offset 26, **6-bit VGA** 0–63 shifted `<<2`). Each level loads **three**
  palette regions banked into fixed DAC windows (`PAL_WALLS=0xB0`, `PAL_M1=0xC0`, `PAL_M2=0xE0`),
  plus precomputed **fade-index tables** used for torch-distance shading — a genuinely different
  dimming mechanism from EOB1/2.

### Reuse assessment for `beholder-importer-westwood`

| Westwood decoder | Reuse for EOB3? |
|---|---|
| `MAZ` (per-face wall grid) | **No** — replace with a trivial raw 32×32×1-byte occupancy reader. *Simpler* than MAZ. |
| `VCN` (wall tile-blocks) | **No** — wall art is AESOP `1.10` VFX shape tables → new codec. The *concept* (indexed wall slices) survives; the container does not. |
| `VMP` (viewport block-map) | **N/A — no such data.** The projection is bytecode (see Q3 / risk). |
| `CPS` / Format80 (LCW) | **Yes, but only** for char-gen backdrops (`CHARGEN.CPS`), a Westwood-catalogue holdover — never dungeon art. |
| Palette | **Partial** — 6-bit VGA concept reused; new header parser + fade tables needed. |

There is also a **second, unrelated container** for cutscenes: `GFFI` (`*.GFF`, IFF/GFF-style
tagged blocks `PAL`/`BMP`/`BMA`/`LSE`) — not on the dungeon path, only relevant if we ever import
intro/finale assets.

---

## Q2 — What is AESOP structurally?

A bytecode virtual machine (John Miles / Miles Design, 1992) with an object/message runtime, a
resource archive, and a native-function library. EOB3's entire game — level load, movement,
combat, rendering — is SOP bytecode running on this VM; the engine only supplies primitives.

### The `.RES` archive (confirmed: original source ≡ DAESOP ≡ real bytes)

**Global header — 36 bytes @ 0x00** *(empirically confirmed: `first_directory_block` = 0x24 = 36)*:

| Off | Type | Field | Note |
|---|---|---|---|
| 0x00 | char[16] | `signature` | `AESOP/16 V1.00\0` + 1 pad. **Constant even for the 32-bit VM and for DH** — not an engine discriminator. |
| 0x10 | u32 | `file_size` | == on-disk size exactly (validated in all four files). |
| 0x14 | u32 | `lost_space` | dead bytes from in-place patching (resources are never physically removed). |
| 0x18 | u32 | `first_directory_block` | file offset of first dir block; **0x24 in all four files**. |
| 0x1C | u32 | `create_time` | DOS packed date/time. |
| 0x20 | u32 | `modify_time` | DOS packed date/time. |

**Directory block — 644 bytes (0x284), 128 entries, singly-linked chain:**

| Off | Type | Field |
|---|---|---|
| +0x000 | u32 | `next_directory_block` (file offset; **0 = end of chain**) |
| +0x004 | u8[128] | `data_attributes[slot]` (**==1 → free/unused slot**) |
| +0x084 | u32[128] | `entry_header_index[slot]` (file offset of the 12-byte entry header; **0 = empty**) |

**Resource number → offset:** `block = N/128`, `slot = N%128`; walk the chain `block` times; absent
if free/zero, else seek to the entry header. (EYE walks 20 blocks; HACK 24.)

**Entry header — 12 bytes, payload follows immediately:**

| Off | Type | Field |
|---|---|---|
| +0 | u32 | `storage_time` |
| +4 | u32 | `data_attributes` (`DA_*` cache/placement flags; **`DA_PLACEHOLDER 0x10000000` ⇒ no payload**) |
| +8 | u32 | `data_size` |

**No on-disk content-type tag.** `data_attributes` is a cache/placement flag field (empirically a
resident-vs-purgeable distinction), not a category. **Type is inferred** from name suffix
(`.IMPT`/`.EXPT` → code dictionaries; a resource with a companion `<name>.EXPT` is a code class),
from info-string prefix (`map:`, `palette:`, `file:...\X.BMP`, `.SND`, `.XMI`, `.FNT`), or from a
payload magic (`S:` = string resource).

**Five reserved special resources (numbers 0–4)** — the engine dictionaries:
- **0 `ROED`** — Resource Ordinal Entry Directory: **name → number** (the master name table).
- **1 `RDES`** — source-file names / descriptions (the `map:`/`palette:`/`file:` info strings).
- **2 `RDEP`** — resource dependency table.
- **3 `CRFD`** — Code Resource Function Directory: **native-function name → slot** (the engine's
  low-level-function table; **the per-game retarget lever**).
- **4 `MSGD`** — Message Name Directory: **message name → number**.

*(The 2009 forum loosely calls resource 4 the `start` object; the source-cross-checked answer is
`MSGD`. The `start`/root code class is a separately named resource, not one of the five
dictionaries — worth confirming against a real disassembly during implementation.)*

**Dictionary blob format** (also reused for each class's `.IMPT`/`.EXPT`): `u16 bucketCount`,
`bucketCount × u32` chain offsets, each chain a run of length-prefixed strings (`u16 len` incl.
NUL) **alternating key,value**, terminated by a zero length.

### The bytecode VM

- **32-bit stack machine (AESOP/32 for EOB3 R2.0, Oct-1993).** One contiguous byte-addressed stack
  growing downward; 4-byte value slots; three cursors (`Sp`, `Fptr`, `PC`); 16 KB stack. No
  register file. Little-endian operands. AESOP/16 and /32 share **identical opcodes and
  encodings** — only pointer width differs; the `.RES` signature stays `AESOP/16 V1.00`.
- **Code resource (SOP) = a class.** 14-byte program header (`static_size` u16, `imports` u32,
  `exports` u32, `parent` u32 with `0xFFFFFFFF` = none). A message handler is reached via its
  `.EXPT` `M:<msgnum>` offset; at that offset sits a 2-byte `auto_size` (local+param frame) before
  the opcode stream. Locals are zero-initialised on entry; assignment is expression-valued
  (stores leave the value on the stack).
- **88 opcodes (`0x00`–`0x57`)** — a *generic* instruction set (no game-specific opcodes; all game
  behaviour lives in bytecode + the native library). Categories: control flow
  (`BRT/BRF/BRA/CASE/JSR/RTS/END`; branch targets are **absolute 16-bit** code offsets; `CASE` is a
  jump table), stack (`PUSH` reserves+zeros a slot — also the CALL/SEND arg delimiter; `DUP`),
  arithmetic/bitwise/comparison, constants (`SHTC/INTC/LNGC` overwrite TOS in place), variable
  access in four spaces — **auto** (local/param), **static** (instance fields), **extern**
  (cross-object via import ref), and **code tables** — with load/store/effective-address and
  array variants, and the call family `RCRS` (resolve native fn), `CALL` (native call),
  `SEND <argc>,<msg>` (message send), `PASS` (delegate to parent).
- **Objects & messages.** An object is an instance of a class; classes form a **single-parent
  hierarchy (≤16 deep)**. Instance state is a raw byte block, **base-class first**, fields
  addressed by numeric offset (only `B:/W:/L:` name tags in `.EXPT`; no per-field type metadata).
  **Dispatch is by numeric message selector**, resolved up the parent chain (lowest class with a
  handler wins); unhandled messages return −1; `PASS` continues the same message at the parent.
  System messages: `MSG_CREATE=0`, `MSG_DESTROY=1`, `MSG_RESTORE=2` (sent automatically). Runtime
  builds a **thunk** (message-vector table + static/extern base descriptors) and patches import
  slots at instance creation — these fixups are **runtime-only, never on disk**.
- **Native runtime functions (~123, `CRFD`-resolved).** A generic kernel band (math/RNG, strings,
  object mgmt, event system, windowing, graphics raster incl. `draw_bitmap`/`set_palette`/fades,
  text, sound, save/level-streaming) plus an **EOB3-specific band** (`step_X/Y/FDIR`,
  `step_square_*`, `seek_direction`, `spell_request/list`, `player_attrib`, `item_attrib`,
  `change_level`, `magic_field`, `do_dots`, `do_ice`, `create_initial_binary_files`).

### The event model

- **Event** = `{type:i32, owner:i32, parameter:i32}`; a single 128-entry circular FIFO. Types 0–31
  are system/input (`SYS_TIMER=1`, `SYS_MOUSEMOVE=2`, region enter/leave 3/4, click/release and
  region-scoped variants 5–16, `SYS_KEYDOWN=17`); 32–255 are application-defined.
- **Notify request (the binding)** = `{client, message, parameter, status, next, prev}`, pooled
  (768), one linked chain per event type. The script primitive
  **`notify(object, message, event_type, parameter)`** registers *"when an event of `event_type`
  whose parameter matches `parameter` fires, `SEND message` to `object`."* `cancel(...)` removes
  it; destroying an object cancels all its bindings.
- **`dispatch_event()`** pops one event and, for each binding whose `match_parameter` holds,
  `SEND`s the message with `{parameter, owner}` as args. Match modes: `-1` = wildcard;
  `SYS_TIMER` matches when `eventParam ≥ requestParam` (heartbeat threshold — how timed behaviour
  works); all others match on equality. System/input events defer behind pending application
  events so input can't pre-empt an in-progress response.

### How thirdeye loads a level (end-to-end)
1. **Boot mode** from a DOS inter-app cell (`peekmem`): `INTR` (menu) / `CINE` (into game from
   save) / `CHGN` (char-gen transfer first).
2. **`resume_level`**: create the `entities` singleton, register PCs, seed party position
   (default graveyard **LVL03 (7,24) facing east**).
3. **`load_resource(dest, resNum)`** copies raw resource bytes into a SOP static buffer — the maze
   into the dungeon object's `lvlmap` (32×32), the wall set (a `1.10` VFX shape table) into
   `wallset`. Resources are resolved **by name** from area keywords (`"Mausoleum 1"` →
   `"Mausoleum walls"` / `"Mausoleum palette"`).
4. **`loadLevelObjects`** streams the level's temp file and creates every interactive object
   (door / lever / stairs / decoration / monster) as its SOP class, linking each into the `lvlobj`
   grid; each record **stores its SOP class number directly** (2208=door, 2247=lever, …), so no
   type→class table is needed.
5. **Area singletons** run their own bytecode, which calls `set_palette(region, pal)` three times.
6. **Rendering** is the dungeon class's draw handler walking an 18-cell view frustum
   (`view_X/view_Y/visible`), reading `lvlmap`/`lvlobj`, issuing one `draw_bitmap(page, wallset,
   shapeIdx, x, y, scale, mirror)` per visible cell. The C++ only executes the blit.

---

## Q3 — The thesis test: mapping EOB3 events onto the shared IR

The shared IR vocabulary is `Action` / `Condition` / `Reason` / `TriggerBinding`. AESOP's closest
primitives map cleanly at the **event/message seam**:

| Shared IR | AESOP equivalent |
|---|---|
| `TriggerBinding` | a `notify(client, message, event_type, parameter)` registration → `(event_type, parameter) ⇒ (object, message)` |
| `Condition` / `Reason` | the `match_parameter` predicate + in-handler bytecode tests; three match modes (equals / `≥` for timers / `-1` wildcard) |
| `Action` | the `SEND` / `CALL` / `PASS` a handler performs (`RT_execute(client, message)`) |

**The tension (the real thesis test).** This is the *opposite shape* from EOB1/2:

- **EOB1/2 (Westwood):** dungeon triggers are **declarative data** — INF trigger rows and wall-flag
  tables our importer already lifts directly into the IR.
- **EOB3 (AESOP):** dungeon logic is **imperative, Turing-complete bytecode**. A lever-opens-door
  or step-teleports is a *message handler mutating grid arrays*, not a table row. Worse:
  - The `notify` bindings are **runtime state, not static metadata** — built imperatively inside
    `MSG_CREATE`/`MSG_RESTORE`, and cancellable/re-armable. There is no static manifest of
    "what reacts to what"; recovering the binding graph means running (or symbolically executing)
    the creation handlers.
  - **Spatial triggers aren't events at all** — tile-entry / wall-button semantics are discovered
    *imperatively* by handler code walking `lvlobj`/`lvlmap` during `step`/`move`. So there are two
    layers: a generic `(event_type, param) ⇒ (object, message)` table for input/timer (snapshot-able
    ≈ TriggerBinding), and hand-written bytecode for spatial dungeon logic (opaque to a
    declarative row).
  - State is **untyped, offset-addressed**, keyed on **numeric message selectors**, and behaviour
    is the **composition of several classes' handlers** via inheritance + `PASS`.

**Implication for the IR (feeds brainstorming).** AESOP *fits* the shared vocabulary at the
event/message seam, but it **forces a family-coherent extension**: the IR needs to hold **both** a
declarative trigger row (EOB1/2) **and** a lifted / handler-referencing action (EOB3) under the
same `Action`/`Condition`/`Reason`/`TriggerBinding` types — never an EOB3-only silo. Two realistic
strategies to get there, to be weighed in brainstorming:
- **(a) Bytecode-lifting pass** — project each class's handler bytecode into the shared Action /
  Condition primitives (a real decompilation/lifting effort; highest fidelity, highest cost).
- **(b) Model the declarative layer + reference the imperative** — snapshot the `notify` bindings
  as `TriggerBinding`s, represent spatial dungeon logic as a *reference to* an AESOP handler
  (opaque action) executed by an embedded mini-interpreter, and lift only the common cases
  (door/lever/teleport/text) into declarative rows. Pragmatic; degrades gracefully.

**Geometry/walkability, by contrast, maps onto the existing IR trivially** — the occupancy grid is
strictly *simpler* than what the IR already carries for EOB1/2 (doors/decorations as entities is
exactly `lvlobj`). So the *geometry* thesis is already proven; the *event* thesis is the open one.

---

## Q4 — Is Dungeon Hack close enough to share the EOB3 path?

**Share everything down to the codec layer; split there.**

**Shared (one implementation):**
- The **`.RES` container reader** — proven byte-identical on `HACK.RES`, `OPEN.RES`, `SAMPLE.RES`,
  and `EYE.RES` (same magic, 36-byte header, 644-byte chained dir blocks, 12-byte entry headers).
- The **AESOP VM** (88 opcodes, object/message model, thunks, inheritance) and the **event model**.

**Per-game (split point):**
- **Name resolution** — EOB3 embeds a name dictionary in the archive; DH's in-archive names are
  encoded and it relies on **external `HACK.TBL`/`OPEN.TBL`** tables.
- **Bitmap codec** — EOB3 uses the `1.10`-tagged VFX shape table (312 resources); DH uses a
  size-prefixed subpicture + copy/fill RLE (632 resources). Zero overlap.
- **Music** — EOB3 ships 66 XMI (IFF `FORM`); DH ships none.
- **Native-function table (resource 3 `CRFD`)** — the retarget lever. DH renders its dungeon via
  **native functions** (`draw_walls`, `init_viewspace`, `build_clipping`, `draw_auto_square`) and
  adds file-I/O, feature-file, and map/visibility natives; EOB3 draws in *bytecode* via the generic
  `draw_bitmap` and has `create_initial_binary_files` (DH aborts on it).
- **Geometry** — **EOB3 = 14 fixed hand-designed 32×32 `MAP` resources**; **DH = procedural**
  (0 stored maze grids, ships `MAZE.EXE`). So DH contributes the object/script/bytecode model and
  a procedural-maze path, **not** stored levels.

**Conclusion:** Dungeon Hack rides the same container + VM + object model — so building the AESOP
foundation for EOB3 directly de-risks DH — but its render path is *native-function-driven* (a
different mechanism from EOB3's bytecode-driven `draw_bitmap`), and its content is procedural. DH is
a **second consumer of the AESOP foundation, not a drop-in of the EOB3 importer**. The 4-game "one
IR" thesis holds at the container/VM/object level; each game keeps its codec + native-table + level
source.

---

## Difficulty & risk assessment

| Work item | Risk | Notes |
|---|---|---|
| `.RES` container reader (shared) | **Low** | Fully spec'd, cross-validated on real bytes; DAESOP proves a pure reader works. Traps: everything is by resource-number via dictionaries; no type tag (infer); `DA_PLACEHOLDER` = no payload; trust the directory, not linear scan (stale `lost_space`). |
| 32×32 occupancy grid → IR geometry → **walkable** | **Low** | Raw 1024-byte grid, `coord & 31` wrap; movement geometry already spelled out (`step_X/Y/FDIR`, `step_square_*`). Simpler than EOB1/2 MAZ. |
| `1.10` VFX shape-table + `PAL` decoders | **Medium** | New but self-contained scanline-RLE + 6-bit palette with fade tables. |
| **Wall-render fidelity** | **High** | The projection is bytecode, not data. Either **(a)** embed an AESOP SOP interpreter (large), or **(b)** reverse the dungeon draw handler *once* and **bake its frustum tables** (per-view-cell shape index / screen x,y / scale tier / mirror) into the IR as static data. Route (b) is the pragmatic one and is genuine RE work. **Open design fork:** can EOB3 walls render through our existing render-core projection (25 Westwood wall slots) with AESOP art mapped in, or does EOB3's 18-cell / 3-scale-tier frustum differ enough to require its own baked projection? → brainstorming. |
| Event lift into shared IR | **High** | The real thesis test (Q3): imperative bytecode vs declarative rows; runtime-built bindings; spatial triggers fused into handlers. |
| DOS-memory permissiveness | **Medium** | The original relied on zeroed heap slack (OOB static/extern reads return 0). An interpreter/executor must reproduce "OOB reads as zero, writes sink" or real game logic diverges. |

---

## Proposed first milestone

Mirror how EOB2 landed (geometry first → the walkable "forest" payoff → then render-fidelity
fixes). Split the vertical into two milestones so the visible payoff arrives before the hard RE:

- **M-EOB3.1 — AESOP `.RES` reader + first level walkable from the shared IR.**
  Build the shared, cross-game `.RES` container reader (skip-safe ITs against the user's own
  `EYE.RES`; never commit game data). Lift the 14 32×32 `MAP` occupancy grids into the existing
  shared geometry IR (occupancy → walls; doors/decorations as entities, initially from the map
  grid alone). Get **a third game walkable** in `beholder-engine-core` / `beholder-render-libgdx`,
  even with placeholder wall art. This de-risks the container + IR-geometry fit with **zero VM
  work** and delivers the visible payoff.

- **M-EOB3.2 — Wall-render fidelity.** Decode the `1.10` VFX wall shapes + `PAL` palettes, resolve
  the open design fork (reuse render-core projection vs bake AESOP's frustum tables), and render
  EOB3 walls with correct art. Then the **AESOP event lift** (M-EOB3.3) as the true thesis test.

**Recommendation:** proceed to `brainstorming` scoped to **M-EOB3.1 + the render fork**, since the
render strategy (Q1/Q3 headline) is the one genuine architectural decision and it shapes the plan.

---

## RE-community credits (for the eventual article)

- **John Miles / Miles Design** — the AESOP engine and toolchain (`ARC`/`sopcomp`/`mapcomp`/
  `palcomp`, `INTERP`/`ARUN`), released with docs as public domain (the games were not).
- **Bret Curtis** — `thirdeye`, the GPL-v3 from-scratch AESOP reimplementation (our clean-room
  render/VM oracle; also the ScummVM/kyra author).
- **Mirek Luza** — `DAESOP`, the public-domain `.RES` analyser / disassembler / decompiler our
  parser is validated against.
- **Darkstar (Darke)** — the ReWiki `AESOP/16` / `.RES` / `EYE.RES` format pages.
- The **2009 VOGONS AESOP thread** participants (`vasyl`, `keropi`, `WolverineDK`, `DosFreak`,
  `takis76`, `Hierophant`, `Dominus`, `wd`, `Qbix`) who published and cross-checked the release
  and tooling.

*Cross-check asset: for any disputed on-disk field, the Miles compiler (`arc`) is the period-original
**writer** and `thirdeye` is an independent **reader** — two independent descriptions of the same
format, ideal for clean-room confirmation.*
