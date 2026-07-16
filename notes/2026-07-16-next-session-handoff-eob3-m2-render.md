# Handoff prompt — next session: EOB3 M2 (EOB3-native render)

> Paste the block below into a fresh session to continue. (Kept here so it isn't lost; the session's
> auto-loaded `MEMORY.md` already carries the vision, conventions, coding standards, env map, the
> package-org rule, and the AESOP oracle stash.)

---

You are continuing the **legend-of-the-beholder** project — a clean-room, one-shared-IR
reimplementation of the SSI Eye of the Beholder games. Chat in **Hungarian**, code/docs/Javadoc in
**English**. Your auto-loaded `MEMORY.md` carries the vision, conventions, coding standards, env map,
the package-org rule, and the AESOP oracle stash — read them; they govern.

## Where we are
- **Phase 1 (EOB1)** and **Phase 2 (EOB2)** complete and merged (Westwood engine): geometry + doors +
  decorations + the `.INF` event lift + a byte-stable lossless round-trip, both games, one IR.
- **Phase 3 M1 (EOB3 / AESOP importer + walkable) is COMPLETE and merged to `main` (`c737052`,
  pushed to origin).** A **THIRD game (EOB3, the AESOP engine, `EYE.RES` container) is now walkable
  from the SAME shared IR**, with **placeholder walls**. New module `beholder-importer-aesop`
  (`ResArchive` `.RES` reader, `Roed` name→number dict, `OccupancyMaze` 32×32 grid, `AesopToIr`
  occupancy→per-edge `LevelGeometry`, `PlaceholderAppearancePack`, `SyntheticRes` test builder) +
  `beholder-app` wiring (`Eob3Extractor`, `GameKind.EOB3`, `GameDetector` `EYE.RES` marker, `Extract`)
  + skip-safe real-`EYE.RES` ITs (walkable, lossless round-trip, wall-render). **`beholder-ir` and
  `beholder-engine-core` are UNTOUCHED** — the one-IR boundary held; walkability came free. Root
  `mvn verify` green (955 tests, 0 skipped). Two importer modules (westwood + aesop), one IR, three
  games.
- Durable task history: `.superpowers/sdd/progress.md` (all task outcomes + the deferred-minor list).
  Recon: `notes/2026-07-15-eob3-aesop-recon.md`. **Render spike writeup (the M2 seed):**
  `spikes/2026-07-16-eob3-render-spike.md`. M1 design+plan: `plans/2026-07-16-p3-m1-eob3-importer-*`.

## The next milestone: M-EOB3.2 — the EOB3-NATIVE render (real Mausoleum walls)
M1 renders placeholder (flat) walls through the *existing* EOB1 projection. **M2 makes EOB3 look like
EOB3**: decode the real wall art and draw it with EOB3's own whole-sprite projection. Two throwaway
spikes already **PROVED this end-to-end on real bytes** (see `spikes/2026-07-16-eob3-render-spike.md`)
— the reconstructed EOB3-native view of the Mausoleum looks authentic, so this is de-risked
execution, not research. The spike code itself lived in the session scratchpad (gone); **re-derive
the decoders clean-room from the writeup's facts** (that's how the spike was built).

**The render fork is already resolved (maintainer decision):** M2 is a **clean EOB3-native
whole-sprite render path** (its own view frustum + sprite blitter + placement tables), **NOT** the
big render-core parameterization refactor.

**The confirmed facts to build from (all in the spike writeup, verified on the real `EYE.RES`):**
- Resources (resolve by name via `Roed`): **Mausoleum walls = #253** (`1.10` VFX shape table, 19
  shapes), **Mausoleum palette = #386** (`PAL`, banked at **DAC 0xB0**), **Fixed palette = #335** (at
  DAC 0x00). The four wall sets: Mausoleum #253, Forest #228, Ruins #236, Marble #297.
- **`1.10` VFX codec:** magic `'1.10'`+`u32 count`+directory of `{u32 offset(from resource start),
  u32 color}`; each shape a 24-byte subpicture header (`u16 boundsY=h-1`, `u16 boundsX=w-1`) then
  per-scanline RLE (0=EOL, 1=transparent-skip, even≥2=run, odd≥3=literal; **pixel index 0 =
  transparent**). **Wall pixel bytes are ABSOLUTE DAC indices** → build a 256-palette = #335 at 0x00
  overlaid by #386 at 0xB0.
- **`PAL`:** `u16 numColors@0`, RGB at offset 26, 6-bit VGA (`<<2`).
- **Wall-set structure (#253, 19 shapes):** backdrop 176×120 (floor+ceiling); front walls 128×96 /
  80×59 / 48×37 (near/mid/far); side walls 24×120 / 24×95 / 16×59 / 8×35 (near→far, L/R). Viewport =
  **176×120**.
- **Projection reconstruction (spike, looks authentic):** front walls centred at x {24,48,64},
  bottom on floor-lines {119,103,79,63}; side walls at left-x {0,24,48,64} widths {24,24,16,8},
  right = mirror; painter far→near; backdrop first.
- **THE GAP (the one real RE task):** the exact per-cell `(cell,side,depth)→shapeIndex` and screen
  `(x,y,scale,flip)` tables live in **EOB3 SOP bytecode (the dungeon-draw class, resource 2409)** —
  NOT in the `thirdeye` oracle (which runs the bytecode). M2 must either **(a) reverse the dungeon
  draw handler once and bake those tables into IR/pack data** (pragmatic), or **(b)** run a mini
  AESOP interpreter. The spike's reconstruction is close enough for a first cut; tune against DOSBox
  pixel ground-truth (the EOB3 `.app` is DOSBox-runnable).

## Immediate task
Run the normal flow: **superpowers brainstorming → writing-plans → subagent-driven-development**.
Scope M2: (1) `beholder-importer-aesop` decoders for `1.10` VFX + `PAL` (+ their DAC banking);
(2) an EOB3-native whole-sprite render path fed by those decoders + the placement tables (route (a)
above); (3) wire it so an EOB3 level renders its REAL Mausoleum walls (replace the placeholder pack).
Likely first visible payoff: **the real Mausoleum walkable with its real walls**. Then **M-EOB3.3 =
the AESOP event lift** (triggers/doors/decorations → the shared Action/Condition/Reason/TriggerBinding
vocabulary) — the real "one IR for all SSI games" thesis test.

Deferred M1 follow-ups (M2 tickets, in the ledger's DEFER list): SyntheticRes public-main-scope (or a
test-jar); a couple of test/doc nits already triaged as non-blocking.

## Process (unchanged)
brainstorming → writing-plans → subagent-driven-development (fresh implementer + independent review
per task; whole-branch review on the most-capable model; `--no-ff` merge to `main` on the maintainer's
verdict; branch kept). No worktrees, conventional branches. JDK 21 (Corretto;
`JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home`); ALL `-Werror`
coding-standards gates (one-return/method, no-instanceof, ≤5 params, full Javadoc + doclint, named
constants + what/why comment) + ≥90% branch+line coverage; use `-am` or a root verify (reactor stale-
`~/.m2` trap). Clean-room: `thirdeye` / original AESOP source / DAESOP are read-only fact oracles —
**never copy their code; isolate GPL reads in subagents; derive facts in your own words**; NEVER
commit game data or game-art images (skip-safe `assumeTrue` ITs; artifacts to git-ignored `target/`).
Mandatory tooling: superpowers, RTK, jCodeMunch. Keep research notes for the eventual **articles**,
crediting the RE community (thirdeye/Bret Curtis, John Miles, Mirek Luza/DAESOP, Darkstar/ReWiki, the
2009 AESOP thread).

**Start** by reading `spikes/2026-07-16-eob3-render-spike.md` (the M2 facts) and
`.superpowers/sdd/progress.md` (the M1 outcomes + deferred list), then run brainstorming for M-EOB3.2.

---
