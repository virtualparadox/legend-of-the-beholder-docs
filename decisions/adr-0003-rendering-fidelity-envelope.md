# ADR-0003 — Rendering-fidelity envelope

> **Status:** Proposed — firm principle and a concrete envelope; the final
> grey-area line is the maintainer's authenticity taste call
> ([NEEDS HUMAN CONFIRMATION], see below). Awaiting the Phase 0 human-gated
> checkpoint (MASTER §12; PHASES "Phase 0" checkpoint).
> **Date:** 2026-07-06
> **Deciders:** maintainer (human-gated).
> **Resolves:** MASTER §11 open decision — *"Rendering fidelity envelope —
> exactly how much QoL (resolution, input) is allowed before it violates §7.5.
> Draw this line explicitly and defend it."*
> **Related:** MASTER §3 (out of scope), §4 ("uniform QoL"), §7.4 (authenticity
> in constraints), §7.5 ("do not drift into a remake"), §7.2 (illegal states
> unrepresentable); PHASES §3.1 (the three separable IR layers).

---

## Context

MASTER §7.5 is blunt: "The moment free mouselook / continuous movement / hi-res
sprites creep in 'to look nicer,' the reason the project exists is dead. Grimrock
exists for that. The value here is authentic scarcity." §3 puts the same items
out of scope. Yet §4 lists, as *uniform QoL for all four games*, "auto-map,
save-anywhere, **hi-res rendering of period art**, mouse marking." So the source
documents contain a **real, deliberate tension** — some modernisation is a
feature, some is fatal — and §7.5 demands the line be drawn explicitly and
defended. That is this ADR's job.

**The enabling recon finding (PHASES §3.1):** the IR is three separable layers,
and *appearance* is fully separable from *logic*. The tileset/appearance pack —
**VCN 8×8 tile atlas + VMP projection LUT + palette + shading** — is decoupled
from the geometry/behaviour and event layers. "This is what makes higher-fidelity
/ QoL rendering feasible without touching level logic (MASTER §7.5)." So the
question is **not** *can* we render hi-fi (we can, cleanly) — it is **how much is
allowed** before presentation crosses into remake.

## Decision

### The governing rule (the line, stated once)

> **A modernisation is ALLOWED if and only if it changes only how the fixed
> game-model is *presented* (the appearance/render layer, PHASES §3.1). It is
> FORBIDDEN if it changes what the player can *perceive or do per turn* — the
> interaction model: the discrete grid, lockstep movement, information scarcity,
> and discrete input.**

Presentation may improve without limit; the **interaction model is frozen at the
originals'**. Appearance is a swappable pack; the interaction model is not a knob.

### Reconciling "hi-res period art" (§4, allowed) with "no hi-res sprites" (§3/§7.5, forbidden)

The two clauses are not in conflict once the distinction is named:

- **ALLOWED — faithful high-resolution *presentation of the period art itself*.**
  The render pipeline may output at any resolution and may even redraw the VCN
  atlas at higher fidelity **in the original palettised style and content**. It
  still looks like Eye of the Beholder, only crisp.
- **FORBIDDEN — substituting *new, modern-style* art** (PBR, normal-mapped
  lighting, photoreal or re-imagined sprites) that replaces the period look "to
  look nicer." That is the "hi-res/PBR art" of §3 and the "hi-res sprites" of
  §7.5.

The test: *does it still read as EOB, just cleaner (allowed) — or does it read as
Grimrock (forbidden)?*

### The concrete envelope

**ALLOWED** (appearance layer; does not touch the interaction model):

- **Arbitrary render/output resolution** of the period art (crisp scaling; optional
  higher-fidelity redraw of the atlas *in the same style*). Same view frustum,
  same cells visible, same reveal distances as the original.
- **Auto-map** (MASTER §4) — recording what the party has *actually seen*.
- **Save-anywhere** (MASTER §4).
- **Mouse marking** (MASTER §4) — mouse for discrete UI: inventory, clicking wall
  buttons/levers, selecting targets the original allowed.
- **Index-colour / palettised tile banks preserved** as a constraint (MASTER §7.4).
- **Smooth sprite/effect animation and frame rate** (presentation only).
- **Optional cosmetic filters** (CRT/scanline, clean integer scaling), off by
  default.

**FORBIDDEN** (changes the interaction model → remake, MASTER §3/§7.5):

- **Free mouselook** / continuous camera aiming.
- **Continuous / free movement** breaking grid lockstep.
- **PBR / modern 3D / normal-mapped lighting / new photoreal or re-imagined art.**
- **Real-time free-flight camera** in place of the stepped, cell-aligned views.
- **Widening the field of view** (more cells visible = more information = an
  interaction-model change, not presentation).

**GREY AREAS** — enumerated with a recommended default; the precise final line is
[NEEDS HUMAN CONFIRMATION] (see below):

- **Render resolution.** *Recommended:* unrestricted output resolution; the
  *game-visible information* stays exactly what the original showed (same frustum,
  same A–O cell visibility, same reveal distance). Resolution is presentation.
- **Aspect ratio.** *Recommended:* preserve the original view's aspect/FoV;
  pillarbox/letterbox or a decorative border to fill modern screens. Do **not**
  widen the FoV to fill the screen (that adds information).
- **Movement / turn animation (input smoothing).** *Recommended:* a short
  *tween* between grid cells (step/turn interpolation) is allowed as pure
  presentation **iff** it does not change timing-dependent logic and does not let
  the player act mid-step — still one command per cell, still lockstep. This is
  the sharpest grey line and the one most likely to drift; flagged below.
- **Auto-map reveal policy.** *Recommended:* reveal only what the party has
  actually explored; do **not** reveal hidden/illusory walls the party has not
  discovered, and do **not** full-reveal the level. The map records what you saw;
  it is not an oracle — preserving information scarcity (MASTER §7.4/§7.5).
- **Mouse usage boundary.** *Recommended:* mouse may select/click **discrete**
  targets the original permitted (inventory, buttons, marking); it may **not**
  provide **continuous** aim or camera control. The discrete/continuous split is
  the operative line.

### Defend the line structurally, not by policy (MASTER §7.4 / §7.2)

The envelope is enforced by **architecture**, not goodwill:

- The renderer consumes **only** the separable appearance pack (VCN + VMP +
  palette + shading, PHASES §3.1) plus the **fixed, discrete grid model**. It has
  **no API** to widen the FoV or free the camera, because the game model does not
  expose continuous position or orientation.
- Party **position and facing are discrete enum/typed values** (cell + one of four
  facings), so free mouselook and continuous movement are **unrepresentable**
  (MASTER §7.2, "illegal states unrepresentable") — not merely discouraged.
- Because appearance is a swappable pack, higher-fidelity *presentation* is a pack
  change that provably cannot reach level logic (PHASES §3.1) — the render/state
  differential tests (MASTER §8) stay valid across resolutions.

### [NEEDS HUMAN CONFIRMATION]

The *principle* and the ALLOWED/FORBIDDEN lists are firm. The **final placement
of the grey-area line** is the maintainer's authenticity taste call and is
marked [NEEDS HUMAN CONFIRMATION] — specifically:

1. Whether **movement/turn tweening** is permitted at all, and if so its exact
   constraint (must remain strictly lockstep, no mid-step action, no logic-timing
   effect).
2. **Aspect/FoV handling** on widescreen: pillarbox at original FoV (recommended)
   vs. a configurable decorative border, with FoV widening firmly excluded.
3. **Auto-map reveal policy** — confirm "explored-only, no oracle" as the scarcity
   line.
4. **Input smoothing** in general — confirm the discrete-input boundary (discrete
   clicks/steps allowed; continuous aim/movement forbidden).

## Rationale

- **The separability recon makes this a *permission* question, not a *feasibility*
  one** (PHASES §3.1). Hi-fi presentation is clean to build; the discipline is
  choosing where to stop, which is exactly what §7.5 asks.
- **"Presentation vs interaction model" is the only line that reconciles §4 with
  §3/§7.5.** It classifies every item the source docs list — auto-map,
  save-anywhere, mouse marking, hi-res period art (presentation, allowed);
  mouselook, continuous movement, PBR/new sprites (interaction/replacement,
  forbidden) — without special-casing.
- **Authenticity as a constraint, not goodwill (MASTER §7.4).** Making the
  interaction model *unrepresentable* in the type system means correct-era
  behaviour is the only thing the engine can produce — the tooling enforces the
  envelope, matching the §7.4 posture and §7.2 typing invariant.
- **The differential-correctness strategy stays intact (MASTER §8).** Because the
  frustum, cell visibility, and reveal distances are unchanged, render/state diffs
  against `kyra` and the DOSBox originals remain meaningful even at higher output
  resolution.

## Consequences

- **Positive.** The renderer is a thin client of the appearance pack + fixed grid
  model; QoL that MASTER §4 promises (auto-map, save-anywhere, crisp period art,
  mouse marking) ships without endangering authenticity. The line is defensible
  and, for the hard cases, structural.
- **Constraint on the render module.** It must expose no continuous
  position/orientation/FoV API. Any future contributor request for "just a little
  mouselook / a wider view" is answered by the type system, not a debate.
- **Grey areas need a one-time ruling** (the [NEEDS HUMAN CONFIRMATION] list),
  then they too become fixed constraints. Until ruled, the *recommended* defaults
  above hold, all on the conservative (authentic) side.
- **Tween animation, if allowed, needs a guard** that it never affects
  logic-timing or enables mid-step action — otherwise it silently becomes
  "continuous movement" (the §7.5 failure). Cheapest guard: animation is a pure
  view interpolation over a completed discrete state transition, never an input
  state of its own.

## Alternatives considered

- **Full remaster (hi-res/PBR art, free camera, smooth movement).** Rejected by
  MASTER §3/§7.5 on the project's face — "Grimrock exists for that." It deletes
  the reason the project exists (authentic scarcity).
- **Pixel-exact only (original resolution, no scaling, no QoL).** Rejected: it
  contradicts MASTER §4's explicitly promised uniform QoL (auto-map,
  save-anywhere, hi-res period art, mouse marking) and needlessly forgoes clean
  presentation that does not touch the interaction model.
- **Policy-document-only envelope (rules a contributor may violate).** Rejected in
  favour of a *structural* envelope (MASTER §7.4/§7.2): make the forbidden states
  unrepresentable rather than merely discouraged, so the line cannot erode "to
  look nicer" (the precise §7.5 failure mode).
- **Per-game envelopes.** Rejected: one runtime, one uniform QoL layer for all
  four games (MASTER §4); a shared envelope is simpler and matches the one-engine
  thesis.

## Open questions

- The four [NEEDS HUMAN CONFIRMATION] grey-area rulings above (movement tween,
  aspect/FoV, auto-map reveal, input-smoothing boundary).
- **Original per-game viewport specifics** (exact FoV / aspect / A–O cell
  visibility) to pin "same information" precisely per game — captured during the
  PHASES P1 render work and its golden references (MASTER §8).
- Whether **cosmetic filters** (CRT/scanline) and **higher-fidelity atlas redraws**
  are shipped or left to appearance-pack mods (they are architecturally mod-shaped
  under PHASES §3.1 either way).
