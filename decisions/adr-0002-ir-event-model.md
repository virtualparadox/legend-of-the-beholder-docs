# ADR-0002 — IR event model

> **Status:** Proposed — **firm**; the core is *determined* by the two Phase 0
> feasibility spikes, not an open taste call. Only the empirically-open
> sub-points listed under "Open questions" remain, and they are resolved by
> further data sweeps, not by human preference. Awaiting the Phase 0 human-gated
> checkpoint (MASTER §12; PHASES "Phase 0" checkpoint).
> **Date:** 2026-07-06
> **Deciders:** maintainer (human-gated).
> **Resolves:** MASTER §11 open decision — *"Scripting model, final call:
> declarative `event → condition → action` … with the explicit rule that a miss
> means extend the vocabulary, not embed a VM."*
> **Grounded in:**
> - AESOP lift feasibility spike (`spikes/2026-07-06-aesop-lift-feasibility.md`)
>   — EOB3 / Dungeon Hack; the object/message/event-queue VM.
> - `.INF` lift feasibility spike (`spikes/2026-07-06-inf-lift-feasibility.md`)
>   — EOB1 / EOB2; the Westwood block-script VM.
> **Related:** MASTER §4.1 (data vs behaviour; no embedded interpreter), §5 (IR
> principles), §6 (AESOP lift is the primary risk), §7.1/§7.2 (fail-closed,
> strong typing); ADR-0001 (serialization that must express this vocabulary).

---

## Context

MASTER §4.1 draws a non-negotiable line: triggers and bytecode are *behaviour*,
and must be **lifted** into a declarative `event → condition → action` model with
a **closed, enumerated action vocabulary** — **not** run by an embedded
interpreter. A miss means *extend the vocabulary*, never *add a VM*. AESOP is the
stress test (MASTER §5, §6): if the object/message model fits cleanly, the rest
follows.

Two Phase 0 spikes hand-lifted real game logic to test this before the IR is
frozen. Their verdicts:

- **`.INF` (Westwood) lifts cleanly — more easily than AESOP.** ~30 opcodes,
  ~26 shared EOB1/EOB2; control flow is shallow and *statically* resolvable
  (`Jump`/`Call`/`Return` targets are literal offsets); the event taxonomy
  (~12 trigger reasons) and the flag/query vocabulary are already closed on the
  original's own terms (`.INF` spike §5).
- **AESOP lifts — mostly.** Handlers are shallow `event → guards → ordered
  side-effects → return` sequences; no general computation, no unbounded
  recursion, no self-modifying dispatch that would force an interpreter
  (AESOP spike §5).

**The single most important finding is a convergence.** Both spikes,
independently and against two unrelated bytecode VMs, hit the *same* hardest
problem and pointed to the *same* fix. The hard problem is **hidden state
threaded between a condition and a later, syntactically separate action**:

- AESOP: object-relative state — handlers read/write *other objects'* variables
  by name and dispatch messages to them; `SEND` is used both as an action and,
  when its return is consumed, as a **query**; wiring is indirect through an
  **event queue** (`post_event`/`notify`), not direct calls (AESOP spike §5, §6).
- `.INF`: EOB2's side-effecting RPN predicates select a specific party member
  into the hidden interpreter register `_activeCharacter`, which a *later*
  action reads back as its implicit target; `dialogue`'s `_dlgResult` behaves the
  same way (`.INF` spike §6).

If the IR's condition/action split is naïvely "pure," it silently drops this
hidden channel — and the lazy fix is to smuggle a mini-interpreter (a mutable
register, a live dispatch engine) back in. That is the one place the "no VM"
rule is actually at risk (AESOP spike §5; `.INF` spike §6, risk flag ii). Both
spikes converge on the fix: the event model must carry **explicit entity
addressing + a bindable "subject" + typed inter-object messages + an event
bus** (`.INF` spike §7.3 + §9; AESOP spike §7.1–§7.3). This convergence is the
spine of this ADR and the reason **one** IR event model serves all four games.

## Decision

Adopt a declarative `event → condition → action` model with a **closed,
enumerated action vocabulary and no embedded VM**, extended with the four
convergence primitives the spikes proved are load-bearing. All control flow and
inheritance are resolved **at import time**; the IR only ever holds resolved,
structured, flattened handlers.

### D1 — Handler shape

A handler is:

```
on <Event> [as <subject>]:
    <ordered, side-effecting action list, with mid-body guards and early return>
```

Action lists are **ordered and side-effecting**, with **mid-body guards**
(`when <cond>: … / else: …`) and **early `return`** — *not* a single flat "then."
Both spikes require this identically (AESOP spike §7.4; `.INF` spike §7.2).

### D2 — Closed action vocabulary, in two tiers

One closed vocabulary unifies **the ~30 Westwood opcodes** and **AESOP's small
author-facing action set**:

- **Tier (a) — author-facing domain actions** (the campaign author's palette):
  change level, teleport/move, turn/face, spawn/destroy entity, spell/field
  effect, damage/heal, give XP, new/consume/steal item, gate-on-item, set/clear
  flag, show text/message, play sound, set-wall/change-wall, open/close door,
  encounter, wait, identify, **emit-event**, and **call a named action-sequence**.
  The `.INF` opcodes map **~1:1** here (`.INF` spike §5); AESOP's `EYE.C`
  game-domain functions map here too (AESOP spike §6 table).
- **Tier (b) — engine services** (invoked *by name* from logic, but implemented
  by the engine and **not** in the author's palette): rendering, automap,
  screen fades, mouse/window/cursor, audio mixing, save/load, and Dungeon-Hack
  file-I/O. AESOP's `GRAPHICS.C`, `INTRFACE.C`, `SOUND.C`, `RTOBJECT.C` lifecycle,
  and the ~46-function DH delta live here (AESOP spike §6 table, §7.6).

Both tiers are **closed sets**. Distinguishing them is what keeps the
author-facing vocabulary small while still absorbing all ~178 AESOP runtime
functions (AESOP spike §7.6). Two AESOP groups are deliberately **not** actions:
`RTCODE.C` string/math utilities become **pure expression built-ins** (D3), and
`EVENT.C` becomes the IR's own event substrate (D6), not user actions (AESOP
spike §6, takeaway 3).

### D3 — A pure expression language (conditions + `SET`)

Conditions and the `SET` action share a **total, loop-free expression language**:
integer arithmetic and bitwise ops (`& | << clamp` — levers/`decflags` need
them), comparisons, and booleans, over: trigger reason, `SELF`/subject/other
queries, scoped flags, item/monster counts, dice, party facing, race/class
membership, wall ids, and typed feature-state fields. It is an **expression**
language (total, no loops), never a Turing-complete body (AESOP spike §7.5;
`.INF` spike §5). AESOP `RTCODE.C` utilities live here as built-ins, not actions.

### D4 — Entity addressing (convergence primitive 1)

Every event, condition, and action can name **`SELF`**, the event's **subject**
(the triggering entity / `mover` / held item / active character), and other
referenced entities or features, via **stable IR identity**. For `.INF` this is
nearly trivial — "this trigger's own block" plus a flat flag/party namespace —
because Westwood levels have no object graph; for AESOP it is a real object
graph. The primitive is designed **once** and used at both scales: proven cheap
against `.INF` first, then stress-tested at scale against AESOP (`.INF` spike §9;
AESOP spike §5, §7.1). Without it, teleport (`mover.x = SELF.dest_x`), lever→door
wiring, and every `report`/`collide`/`impedance` query are inexpressible.

### D5 — Bindable subject: `when … as subject` (convergence primitive 2 — the spine)

A predicate/query may **both test and bind a name** in one construct:

```
when <predicate> as <subject>:
    … Message(speaker=subject) …
    … Damage(target=subject) …
```

This is the shared fix both spikes reached independently (`.INF` spike §6, §7.3;
AESOP spike §7.2). It absorbs:

- **AESOP** `SEND`-as-query — a `SEND` whose boolean/value return is consumed
  (`not SELF.disabled()`, `mover.report(WHAT_ARE_YOU)`) — as a condition/query
  that yields a bound value threaded into later actions.
- **EOB2** side-effecting predicates — `party.memberWithClass(CLERIC) as subject`
  — replacing the hidden `_activeCharacter` register. The bound `subject` is then
  explicitly named by later actions, instead of a spooky side channel. This is a
  *strictly more disciplined* version of what the original did — it removes the
  action-at-a-distance without losing expressiveness (`.INF` spike §6).
- **`dialogue`'s `_dlgResult`** — `RunDialogue(id) -> result` reuses the same
  primitive (a query-producing action, D9).

**Design it once**; both lifts consume the same primitive (`.INF` spike §9,
convergence point 1). This is what keeps hidden interpreter registers out of the
engine.

### D6 — Typed inter-object messages + event bus (convergence primitives 3 & 4)

- **Typed inter-object messages/queries**, kept distinct from engine services
  (tier b): a `SEND` lifts either to `TARGET.message(args)` (fire) or to a query
  (`TARGET.property` read — *preferably resolved to a data-field read*, see D7).
  Model both; keep them separate from `SVC.*` (AESOP spike §7.2).
- **An event bus primitive** — `emit(event, payload)` plus subscription binding —
  because AESOP wiring is indirect through `post_event`/`notify` (a lever posts an
  event that a wired door consumes), not direct calls. The payload is **typed**
  (object refs + a small scalar), matching `post_event(owner, event, parm)` and
  `notify`'s two recipient variables (AESOP spike §7.3). We implement *an* event
  bus and lift `notify` registrations into IR subscriptions — we do **not**
  re-implement AESOP's dispatcher (AESOP spike §6, "how we avoid it" 4). The
  `EVENT.C` functions become this substrate, never user actions.

### D7 — Resolve control flow, calls, and inheritance at IMPORT time (never at runtime)

The IR receives already-structured, already-flattened handlers. The importer runs:

- **A control-flow structuring pass** (basic-block + dominator reconstruction).
  `.INF` `Conditional`/`Jump` and AESOP `BRT/BRF/BRA` reduce to `if/else`; chains
  of mutually-exclusive comparisons over one value (the 4-way facing check) become
  a `switch`; converging forward jumps become a shared exit / early `return`. All
  samples in both spikes were reducible; no loops, no back-edges, no computed
  jumps seen (`.INF` spike §6; AESOP spike §6). **Fallback:** if a sweep ever
  finds genuinely irreducible flow, emit a labelled-goto action **inside a single
  handler only** — never a general VM (AESOP spike §6; `.INF` spike §6).
- **`CASE` / `report` property handlers → property tables, not code.** AESOP's
  `report`-style `CASE` jump-tables answer property queries; lift them as
  **data tables** (property → value) so the query becomes a field read, not
  executed code (AESOP spike §6, §7.2 — "a `report`/`CASE` handler lifted as code
  instead of a table is a small VM in disguise").
- **Call / inheritance flattening.** `.INF` `Call`/`Return` (static literal
  offsets, bounded by the real 10-deep stack) and AESOP `PASS`/`JSR` (dynamic,
  single-inheritance `member` chain) **both flatten at import** — inline, or emit
  **one named shared action-sequence** referenced by each call site. AESOP
  inheritance is resolved by walking `member` parents; a mid-handler `PASS`
  inlines the parent body **between** the child's pre/post actions, preserving
  ordering (AESOP spike §6, §7.8). The IR never holds a live inheritance graph or
  a runtime call stack. `.INF`'s static `Call` output and AESOP's flattened `PASS`
  output become the **same** downstream IR construct: "this handler's actions
  include another named handler's actions" (`.INF` spike §9, convergence point 3).

### D8 — Feature/instance state as typed fields; flags as a small closed namespace

Feature and instance state (enum `stage`, boolean `disabled`, destination coords)
are **typed fields**, so multi-state levers and teleporter destinations are
*data*, not packed bit-arithmetic to be re-executed. Flags are a **small, closed,
enumerated namespace** with three scopes — **level, global/campaign,
per-monster** — as typed boolean/counter fields, **not** raw bitfields. Both
spikes converge on this almost exactly (AESOP spike §7.7; `.INF` spike §7.5). The
original `decflags` bit layout is an engine-internal detail, not something the IR
reproduces (AESOP spike §6, "how we avoid it" 3).

### D9 — Opaque escape hatches modelled as named actions or engine services

- **`.INF` `sequence(id)`** → `PlayCutscene(id: enum)` — a fire-and-forget named
  domain action; the cutscene content is an engine/importer asset, opaque to the
  condition/action language (`.INF` spike §6).
- **`.INF` `dialogue(...)`** → `RunDialogue(id) -> result` — a **query-producing**
  action reusing D5's bindable-subject primitive, so its result is a bound value,
  not a hidden register (`.INF` spike §6, §7.6).
- **`.INF` `specialEvent(id)`** → an **explicitly bespoke, per-level named hook**
  (`SpecialEvent("level10_lightning_puzzle")`), hand-authored as ordinary IR
  content at import time where feasible, and an **engine service** (tier b) only
  where it is truly presentation (e.g. the lightning screen effect). It is
  **not** a closed content table; every sighting is **flagged for manual review
  at import** (`.INF` spike §6, §7.6, risk flag iii). This is a *process*
  requirement on the importer/editor, not a new IR construct.
- **AESOP graphics / save / automap / mouse / file-I/O** → **engine services**
  (tier b), invoked by name, engine-implemented (AESOP spike §6, §7.6).

### D10 — Closed event taxonomy, with trigger-reason as an ordinary condition operand

- **Westwood:** an event binds to **`(block, reason)`**. The reason enum is
  closed (~12 disk values; engine call-site reasons: step onto/off block, click
  wall, timer fire, step-counter fire, use/drop item, monster step, monster
  death, spell hit, dialogue-result). Crucially, `trigger.reason` is an
  **ordinary enumerable condition operand** (`when trigger.reason == BUMP`), so
  "one physical trigger multiplexing two logical events" is expressible without a
  magic dispatch-key primitive (`.INF` spike §4a, §6, §7.1, risk flag i).
- **AESOP:** events bind to **object identity + message**; the application-event
  enum is closed by enumerating `notify`/`post_event` call sites and the
  message-name table (resource 4) (AESOP spike §7.3, §8.4).

## Rationale

- **The spikes determine this, not taste.** Two unrelated bytecode VMs, hand-lifted
  on real data, both reduced to `event → guards → ordered side-effects → return`
  with no need for a general evaluator, and both surfaced the *same* four
  convergence requirements. That independent agreement is the strongest possible
  signal these belong in the core schema rather than being lift-specific bolt-ons
  (`.INF` spike §9; AESOP spike §7).
- **Every "tempted to embed a VM" path has a declarative answer** (AESOP spike §6;
  `.INF` spike §6): `report`/`CASE` → tables; `PASS`/`Call` → import-time
  flattening; `decflags` arithmetic → typed state fields; the event queue → an
  engine event bus; hidden registers → bindable subjects. None requires an
  interpreter — satisfying MASTER §4.1 by construction.
- **One IR across four games.** The addressing primitive is present but nearly
  trivial for EOB1/2 (self + flat flags) and load-bearing for EOB3/DH (object
  graph). Building it cheap against `.INF` first and stress-testing scale against
  AESOP is exactly MASTER §10's Westwood-first, AESOP-as-risk sequencing — for a
  reason beyond convenience: `.INF` is a cheap smoke test of the *shape* of the
  binding/addressing primitive before AESOP tests its *scale* (`.INF` spike §9).
- **Closed vocabulary + strong typing** deliver MASTER §7.2/§7.4: the action set
  is enumerated, illegal constructs are unrepresentable, and authenticity lives in
  the constraint, not in goodwill.

## Consequences

- **The importer is where the intelligence lives.** It must run a structuring pass,
  name resolution, `PASS`/`Call` flattening, and `CASE`→table lifting *before*
  emitting IR (AESOP spike §7.8; `.INF` spike §7.7). This is deliberate: the
  engine stays a simple, closed-vocabulary executor (MASTER §4.1) — the "lift"
  competency exists from the first Westwood importer (MASTER §6; PHASES §3.3).
- **The event bus is an engine primitive**, with defined `emit`/`on` semantics and
  (pending OQ4) a possibly FIFO/re-queued dispatch discipline. It is *the* IR
  event layer we were building anyway, not a re-hosted AESOP dispatcher.
- **Two enumerated tiers to maintain** (domain actions vs engine services). If tier
  (b) is forgotten, presentation/IO calls look "un-enumerable" and invite a
  property bag or a VM — so engine services are enumerated *explicitly* (AESOP
  spike §7.6, risk flag i).
- **`specialEvent`/`dialogue` sightings gate import** — each is a manual-review
  checkpoint, not a blind translation (`.INF` spike §6). This is a known,
  bounded, per-occurrence authoring cost, not an open-ended one.
- **The vocabulary is closed *in principle* from day one, but its final membership
  grows as real games land** (combat/spell/AI/party-management actions are not yet
  fully enumerated) — matching MASTER §10 ("let the schema grow from real games")
  and §5 (no speculative abstractions ahead of two real games). "Closed" is a
  property of the type (a sealed enum), not a claim that every member is already
  known.

## Alternatives considered

- **Embed / port the original VMs (AESOP stack machine; `.INF` interpreter).**
  Rejected head-on by MASTER §4.1 and by both spikes: it re-hosts original-style
  logic in the clean engine, defeats moddability (MASTER §2/§7.6), and is the
  precise anti-pattern ("keep the bytecode, ship an interpreter") the spikes warn
  against (AESOP spike §6; `.INF` spike §6). Feasibility showed it is *unnecessary*
  — the logic is declarative-shaped.
- **Pure condition/action split with no bindable subject.** Rejected: it silently
  drops the hidden channel (`_activeCharacter`, `SEND`-as-query, `_dlgResult`) both
  spikes identified, forcing a mutable interpreter register back in — a VM in
  disguise (`.INF` spike §6, risk flag ii; AESOP spike §5).
- **Runtime inheritance / dynamic dispatch (a live `PASS`/`SEND` engine).**
  Rejected: `PASS` chains and `SEND` resolution flatten cleanly at import; a live
  dispatch engine is unnecessary state in the runtime (AESOP spike §6, §7.8).
- **Per-id opcode for every `specialEvent`.** Rejected on principle: `specialEvent`
  is *by design* "we didn't want to enumerate this" in the original engine;
  enumerating it as a closed table is wrong, not merely impractical. Author it as
  reviewed IR content per occurrence instead (`.INF` spike §6, risk flag iii).
- **Open property bag for actions/state.** Rejected by MASTER §5/§7.2 — the
  vocabulary is closed and enumerated, illegal states unrepresentable.

## Open questions

*(Empirical — resolved by wider data sweeps, not by human preference; carried
from the spikes' §8/§10.)*

1. **Structuring-pass coverage.** All sampled handlers were reducible. Sweep all
   376 AESOP objects (esp. `kernel`, `dungeon`, combat/AI, spell resolution) and
   all 44 `.INF` levels for any irreducible flow / computed jump before *dropping*
   the labelled-goto fallback (AESOP §8.1; `.INF` §6, §10.1).
2. **Final action-vocabulary membership.** Enumerate full runtime-function usage
   across `EYE.RES` (and `HACK.RES`/`OPEN.RES`) plus `.INF` combat/spell/AI/party
   actions to size the closed set precisely (AESOP §8.2; `.INF` §10.4).
3. **`decflags` / feature-state ownership.** Confirm exact bit layout and owner to
   finalise "typed per-feature field" vs "shared flag store" (AESOP §8.3; §7.7).
4. **Event-bus ordering semantics.** Does any logic depend on AESOP's FIFO +
   input-re-queue discipline? If so, the bus must replicate it, not just "emit"
   (AESOP §8.5). Also: EOB2 per-block *multiple concurrent* `(block, reason)`
   bindings, and per-monster-scoped flags (`.INF` §7.1, §10.5).
5. **Escape-hatch census.** Count distinct `dialogue` sub-opcodes and
   `specialEvent` ids across all levels; promote any *recurring* pattern (e.g. a
   class/race rune door) to a real named domain action rather than an authored
   escape hatch (`.INF` §10.4).
6. **Dungeon Hack generator surface.** Confirm none of the ~46 DH functions imply
   *runtime* generation logic that must be scripted (vs. import-time generation) —
   a different risk class (AESOP §8.6).
