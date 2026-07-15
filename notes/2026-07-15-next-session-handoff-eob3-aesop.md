# Handoff prompt — next session: go vertical to EOB3 / AESOP

> Paste the block below into a fresh session to continue. (Kept here so it isn't lost;
> the session's auto-loaded `MEMORY.md` already carries the vision, conventions, coding
> standards, env map, package-org rule, and the AESOP oracle stash.)

---

You are continuing the **legend-of-the-beholder** project — a clean-room, one-shared-IR reimplementation of the SSI Eye of the Beholder games. Chat in **Hungarian**, code/docs/Javadoc in **English**. Your auto-loaded `MEMORY.md` carries the vision, conventions, coding standards, env map, the package-org rule, and the AESOP oracle stash — read them; they govern.

## Where we are
- **Phase 1 (EOB1)** and **Phase 2 (EOB2)** are complete and merged to `main`: geometry + doors + decorations + the `.INF` event lift into the shared IR + a byte-stable lossless round-trip, for BOTH games, through the same `beholder-ir` / `beholder-render-core` / `beholder-engine-core` / `beholder-ir-json` / `beholder-render-libgdx`. **Two importers, one IR.**
- Recently: fixed the EOB2 render (a missing hardcoded standard-wall prefill + per-level appearance-pack selection) — real EOB2 levels (incl. the LEVEL4 **forest**) now render and are walkable from the shared IR; the maintainer walked it and found the Darkmoon Temple from memory. Then refactored `beholder-importer-westwood`'s flat package into `westwood.eob1` / `westwood.eob2` sub-packages (shared decoders stay in the parent; the shared parent imports no `eobN` type).
- Durable task history: `.superpowers/sdd/progress.md`. Design docs + plans: `~/Workspaces/Java/legend-of-the-beholder-docs/plans/`. Research notes (article seeds, with RE-community attribution): `.../notes/`.

## The decision (this session): GO VERTICAL — EOB3 / AESOP
EOB1↔EOB2 are the SAME Westwood engine family, so horizontal breadth mostly re-runs proven machinery. **EOB3 uses AESOP** — a different bytecode VM (John Miles, 1992) — and is the project's PRIMARY RISK + the strongest test of the "one IR for all SSI games" thesis. **AESOP powers BOTH EOB3 (`EYE.RES`) AND Dungeon Hack (`HACK.RES`+`OPEN.RES`)**, so cracking it unlocks the rest of the 4-game vision. De-risk it now while the IR is fresh.

## The key resource (this de-risks it): `~/Workspaces/CPP/AESOP/` (see memory `aesop-oracle-stash`)
- **`thirdeye/`** — a **GPL v3 from-scratch C++ reimplementation of AESOP** for EOB3 + Dungeon Hack, by Bret Curtis (the ScummVM/kyra author), v0.87.0 → the **clean-room oracle** for EOB3 + Dungeon Hack (read-only, like kyra: never copy code; isolate GPL reads in subagents; derive facts in your own words). kyra does NOT cover EOB3.
- **`aesop-package/`** — original AESOP interpreter source (John Miles, 1992). **`daesop/`** — an AESOP decompiler. **`docs/`** — the 2009 AESOP RE thread + Dungeon Hack runtime-function docs.
- EOB3 data: `~/Games/Eye of the Beholder III Assault on Myth Drannor.app/Contents/Resources/game/` (`EYE.RES`). Dungeon Hack: `~/Games/Dungeon Hack.app/...`. EOB1/2 wiki: `~/Documents/EOB-DOCS`.

## Immediate task: an EOB3 / AESOP RECON spike (before brainstorming)
Size the vertical path, grounded in thirdeye (oracle) + aesop-package (original) + daesop (decompiler) + the docs + the real `EYE.RES`:
1. What does EOB3 ship? Is the **geometry/tileset** Westwood-like (can `beholder-importer-westwood`'s `Maze`/CPS/VCN/VMP/palette decoders reuse?) or AESOP-native (a new decoder surface)?
2. What is **AESOP** structurally — the `.RES` resource archive format, the bytecode VM (opcodes, stack model), how levels/scripts are packaged? How does thirdeye load a level end-to-end?
3. The thesis test: how would EOB3's events/triggers map onto the **shared** `Action`/`Condition`/`Reason`/`TriggerBinding` IR vocabulary — does AESOP fit, or force a *shared, family-coherent* vocabulary extension (never an EOB3 silo)?
4. Is **Dungeon Hack** close enough (also AESOP) to share the EOB3 importer path?

Produce a recon writeup in the docs repo `notes/`, then run the normal flow: **superpowers brainstorming → writing-plans → subagent-driven-development**. Likely first milestone: EOB3 first-level **geometry + render** (a third game walkable from the shared IR — the visible payoff, mirroring the EOB2 forest), then the AESOP event lift (the real thesis test).

## Process (unchanged)
brainstorming → writing-plans → subagent-driven-development (fresh implementer + independent review per task; whole-branch review; `--no-ff` merge to `main` on the maintainer's verdict). No worktrees, conventional branches. JDK 21 (Corretto; `JAVA_HOME=/Users/tothp/Library/Java/JavaVirtualMachines/corretto-21.0.4/Contents/Home`), ALL `-Werror` coding-standards gates + ≥90% Surefire coverage, full Javadoc, every magic value a named constant + a what/why comment. Clean-room (thirdeye/aesop/kyra are read-only oracles; NEVER commit game data — skip-safe `assumeTrue` ITs; real-level artifacts to git-ignored `target/`). Mandatory tooling: superpowers, RTK, jCodeMunch. Keep research notes for the eventual **articles**, crediting the RE community (thirdeye/Bret Curtis, John Miles, the AESOP thread, the EOB-DOCS wiki contributors).

**Start** by reconning `~/Workspaces/CPP/AESOP/thirdeye` (its `README.md`, how it loads EOB3 `EYE.RES`, its AESOP VM + `.RES` handling) and the real `EYE.RES`, then report the recon findings and propose the first milestone.

---
