# AGENTS.md — Grid-Crawler Engine Platform (canonical agent instructions)

> **Single source of truth for how agents work on this project.**
> The code repository (`legend-of-the-beholder`) carries only a thin pointer to
> this file. Read this before doing anything, then read the two companion
> documents in this same docs repository:
>
> - **`MASTER.md`** — the project spine: architecture, invariants, sequencing. Treat §§3–8 as fixed constraints.
> - **`CODING_STANDARDS.md`** — the build gates enforced by `eu.virtualparadox:parent` (Java 21, Palantir format, one `return` per method, no `instanceof`, ≥90 % branch *and* line coverage, no `System.out`/`System.exit`/`Thread.sleep`, etc.).

---

## 1. Mandatory tooling

These three are **required**, not optional. Use them by default.

- **superpowers (skills).** Invoke the relevant skill *before* acting. Process
  skills set the approach first: `brainstorming` before any creative/design
  work, `systematic-debugging` before diagnosing a bug, `test-driven-development`
  before writing implementation code, `writing-plans` / `executing-plans` for
  multi-step work. If a skill plausibly applies, use it.
- **RTK (Rust Token Killer).** Dev shell commands are auto-rewritten through the
  Claude Code hook (e.g. `git status` → `rtk git status`), transparently. Use
  `rtk` directly for its meta commands (`rtk gain`, `rtk discover`, `rtk proxy`).
- **jCodeMunch.** Code-intelligence MCP for navigation, impact/blast-radius,
  references, refactor planning, and repo maps. Prefer it over blind grepping
  for structural questions. **See the hard rule in §2.**

## 2. Hard rule: jCodeMunch is local-only

- **jCodeMunch runs LOCAL ONLY.** No project source, data, embeddings, or
  indexes may be sent to jCodeMunch's internet services, ever. This is the one
  hard restriction.
- **git is fine anytime** — committing and pushing to the project's own
  repositories (e.g. `github.com/virtualparadox/legend-of-the-beholder-docs`) is
  expected and allowed.
- **Do not leak project code/data to third-party AI/analysis/indexing or
  cloud-review services.** The project is a public clean-room engine; the concern
  is uncontrolled analysis pipelines, not the user's own git remotes.
- **Reading public references is always allowed** — ScummVM `kyra`, the EOB
  reverse-engineering wiki, the VOGONS AESOP thread, DAESOP / Thirdeye, the AESOP
  language manual, etc.

## 3. Communication language

- **Interactive communication with the maintainer: Hungarian.**
- **English** for: all documentation (this file included), Javadoc, and source
  code (identifiers and comments). Everything committed reads in English;
  the conversation happens in Hungarian.

## 4. Local environment map

The maintainer keeps `~/Workspaces` organized **by language / technology**
(`Java/`, `CPP/`, `Rust/`, `Go/`, `Python/`, `ZIL/`, `Kaitai/`, `GHidra/`, …).
Projects follow a `<name>` + `<name>-docs` sibling-repository convention, both
checked out side by side under the same parent directory.

**This project**

| Purpose | Path |
|---|---|
| Code | `~/Workspaces/Java/legend-of-the-beholder` |
| Docs (this repo, source of truth) | `~/Workspaces/Java/legend-of-the-beholder-docs` |

**Ground truth & references (already present locally)**

| What | Path | Relevance |
|---|---|---|
| ScummVM (`kyra` engine) | `~/Workspaces/CPP/scummvm` | Westwood EOB1/2 ground truth (MASTER §8/§9) |
| `darkmoor` | `~/Workspaces/Java/darkmoor` | libGDX EOB2 remake — closest stack reference; its A–O visibility + far-to-near draw order are reusable, but its hardcoded pixel/scale tables are an anti-pattern (MASTER §9/§11) |
| `eoblib` | `~/Workspaces/Java/eoblib` | clean-room EOB1 binary importer (CPS/VCN/VMP/MAZ/INF/DAT) + `.INF` bytecode disassembler — the model for our importer→IR layer |
| `wallsetviewer` | `~/Workspaces/Java/wallsetviewer` | VCN/VMP → perspective wall-image assembly (VGA/EGA/Amiga) |
| `parent` | `~/Workspaces/Java/parent` | Maven parent behind `CODING_STANDARDS.md` |
| EOB RE wiki dump | `~/Documents/EOB-unzip/eob.phpwiki/mainSpace` | Concrete EOB1/2 format specs + 12 decompiled EOB1 scripts (no AESOP/EOB3 logic docs) |

**Original game data** (user's own — MASTER §9 legal model; engine ships no assets)

- `~/Games/<Game>.app/Contents/Resources/game/`. EOB1 packed (`EOBDATA*.PAK`,
  `EYE.PAK`); EOB2 loose (`LEVEL*.INF/.MAZ`, `ITEM.DAT`); EOB3 `EYE.RES`
  (+`.GFF` cutscenes); Dungeon Hack `HACK.RES`, `OPEN.RES`. The `.app` bundles
  include DOSBox → the originals are runnable for ground-truth capture.

**Reverse-engineering tooling directories**

- `~/Workspaces/GHidra`, `~/Workspaces/Kaitai`, `~/Workspaces/ZIL`.

**AESOP tooling (cloned under `~/Workspaces/AESOP/`)**

- `thirdeye/` — GPL C++ AESOP reimpl (EOB3 + Dungeon Hack). **Oracle only — read
  to learn, do NOT copy into this non-GPL codebase.** Useful RE docs live in
  `thirdeye/docs/`.
- `daesop/` — public-domain resource extractor + **bytecode disassembler**;
  `abc_list.def` is the adoptable opcode spec.
- `aesop-package/` — AESOP/32 compiler + runtime + language manuals (public
  domain, John Miles).
- `docs/` — the Dungeon-Hack runtime-function delta + the original VOGONS thread
  archive.
