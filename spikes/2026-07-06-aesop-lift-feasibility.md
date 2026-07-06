# AESOP lift feasibility spike

> **Date:** 2026-07-06
> **Type:** primary risk-retirement spike for the IR (MASTER §6, PHASES §5).
> **Question:** Does the AESOP object/message/event-queue model lift cleanly into
> the IR's declarative `event → condition → action` form with a *closed* action
> vocabulary — and where does it strain — **before** we freeze the IR?
> **Verdict up front:** **Mostly yes.** Handler bodies are shallow, side-effecting
> `event → guards → actions` sequences that lift almost 1:1. The strain is not
> control flow; it is that AESOP is *object-relative* — handlers read/write other
> objects' state and dispatch messages to them — so the IR must carry an
> **entity/object addressing + typed message** model, or the lift silently
> degrades into "embed a VM." Details and mitigations below.

All raw disassembly stays in scratch (SSI-copyright-derived); this note is our
analysis in our own words with only a few illustrative pseudo-lines.

---

## 1. Toolpath used

**Winner: DAESOP 0.85 C source, compiled natively with clang on macOS arm64.**
Chosen over Thirdeye (`apps/daesop`, GPL — oracle-only, must not seed our code)
and the DOS `daesop.exe`. Two portability fixes were required — both are
instructive for our future importer, so they are recorded here:

1. **`long` width.** DAESOP targeted 32-bit Watcom, where `long == 4 bytes`. The
   on-disk RES structs (`RESGlobalHeader`, `RESDirectoryBlock`, `RESEntryHeader`)
   use `ULONG`/`LONG` for their 4-byte fields. On LP64 (macOS arm64) `long` is
   8 bytes, which doubles every struct and corrupts all directory parsing. Fix:
   redefine the type aliases in `tdefs.h` to fixed-width (`uint32_t`/`int32_t`,
   etc.). **Validation that the fix is correct:** the directory-block stride came
   out to exactly **644 bytes** = `4 + 128 + (128 × 4)`, which is only true when
   `ULONG` is 4 bytes and structs are packed to 1.
2. **DOS-isms.** `#include <malloc.h>` and `#include <dos.h>` don't exist on
   macOS; no DOS symbol is actually *referenced*, so a one-line `malloc.h` shim
   (`#include <stdlib.h>`) and an empty `dos.h` shim let it link. Compiled with
   `clang -I. -fpack-struct=1 -w` (packing-to-1 matches Watcom `-zp1`).

**Validation before touching game data:** ran the built binary on the bundled
public-domain `SAMPLE.RES`. It correctly reported the header, listed 8 resources,
and fully disassembled the `start` code resource (object with `create`/`destroy`
handlers; a `CALL "C:launch"` on an inline `"xxx.exe"` string). Only then did I
run it on `EYE.RES`.

**RES filename quirk (note for the importer):** DAESOP mangles input paths that
contain `.` before the extension (it truncated the `.app` bundle path). Copying
`EYE.RES` into a clean scratch path fixed it. Output is Latin-1, not UTF-8
(re-encode when reading).

**Commands (all output in scratch only):** `daesop /ir EYE.RES …` for the
resource inventory; `daesop /j EYE.RES "<name>" …` to disassemble a named object.

---

## 2. What I disassembled

`EYE.RES` (6.5 MB, `AESOP/16 V1.00`, 1993) contains **376 code resources**
(= AESOP classes/objects), each paired with an `.IMPT` (imports) and `.EXPT`
(exports) resource, plus 749 strings, bitmaps, sound, music, fonts, and 5 special
tables (resource 0..4: names, low-level function table, message-name table).

The code resources are visibly organised as an **object hierarchy of dungeon
features**, with base classes and per-appearance subclasses that inherit from
them. Confirmed base classes: `features` (root of interactive props), and under
it `levers`, `buttons`, `doors`, `locks`, `teleporters`; concrete subclasses e.g.
`mausoleum lever`, `marble teleporter`, `marble door`, `mausoleum door button`,
`marble floor spike trap`, `illusionary wall`, `mausoleum ceiling pit`. This is
*exactly* the `event → condition → action` domain, so the sample is representative.

I fully disassembled and hand-analysed: `levers`, `buttons`, `teleporters`
(base classes), and `marble teleporter`, `mausoleum lever`, `mausoleum door
button`, `marble floor spike trap` (subclasses). Two are hand-lifted in §4.

### The AESOP model, as observed (grounded in the manual + the bytecode)

- A **code resource = a class/state-object.** Its `.EXPT` names the object, its
  **parent** (single inheritance — e.g. `levers member features`), its public
  static variables, and its **message handlers** by `(message-id → byte offset)`.
- A **message handler = one event handler.** Handler = named message + a small
  body of stack-VM bytecode ending in `END` (opcode 0x56).
- **`SEND obj,"msg"` (0x22)** dispatches a named message to an object → runs that
  object's handler and can **return a value** (so a `SEND` is used both as an
  *action* and, when its return is consumed, as a *condition query*).
- **`PASS "msg"` (0x23)** forwards to the parent class's handler (the inheritance
  escape hatch; the child handler *overrides* unless it explicitly PASSes).
- **`CALL n` (0x21)**, preceded by **`RCRS "C:fn"` (0x20)**, invokes a **native
  runtime function** through the import table. Discovered detail: the `CALL`
  operand is the **argument count** (args are pushed and delimited by `PUSH`,
  0x04). Verified on `SAMPLE.RES` (`launch` with 4 args → `CALL #04`) and
  throughout EYE.
- **Event queue.** Objects register interest with `notify(obj,"msg",event,parm)`;
  `post_event(owner,event,parm)` enqueues; the main loop is literally
  `do dispatch_event(); while(!game_over);` — FIFO, with input events re-queued
  behind application events. So event→handler binding is *data* (`notify`
  registrations), not hardwired.
- **State** lives in three places: `auto` locals/params (per-call), `static`
  vars (per-object instance state, e.g. a lever's `stage`), and **`extern` vars
  imported by name from *other* objects** (`W:decflags` from `features`,
  `party_x/party_y` from `kernel`, `x/y/lvl` from `entities`, dungeon render
  state from `dungeon`). Extern access is `LXB/LXW/LXD` / `SXB/SXW/SXD` (0x46–0x4B).
- **Conditions** are ordinary compare/branch ops: `LT LE EQ NE GE GT` (0x15–0x1A),
  `NOT/BAND/BOR` and friends, feeding `BRT`/`BRF`/`BRA` (0x00–0x02).
- **Control flow** in handlers is shallow: forward `if/return` guards, an
  occasional bounded loop, and a `CASE` jump-table (0x03) used mostly to answer
  property queries (a `report` message that switches on a property id).

---

## 3. A tiny pseudo-schema for the lift

To make the mapping concrete I use this throwaway notation (not a proposal for
the IR's concrete syntax — just enough to show the shape):

```
on <EventName>(params…) [for <ObjectType>]:
    when <condition-expr>:            # zero or more guards, short-circuit
        <action>                      # ordered, side-effecting
        …
    else: …
# actions:   SELF.msg(args) | OTHER.msg(args) | SVC.fn(args) | SET var = expr | PASS | RETURN v
# conditions: comparisons over SELF/OTHER queries, extern vars, params, statics
```

Three action kinds fall out of the bytecode: **`msg(...)`** (a lifted `SEND`,
to self or another object), **`SVC.fn(...)`** (a lifted runtime `CALL` — an
enumerated engine service / domain action), and **`SET`** (a lifted store).
`PASS` and `RETURN` are structural.

---

## 4. The two hand-lifts

### 4a. `levers` → message `"missile hit"` (the clean case)

Raw shape (our re-expression of a ~6-instruction handler):

```
LAW THIS ; SEND THIS "disabled" ; NOT ; BRF L
LAW THIS ; SEND THIS "toggle"
L: push 1 ; END        # return 1 = "handled"
```

Lifted:

```
on MissileHit for Lever:
    when not SELF.disabled():        # condition = a self-query SEND, negated
        SELF.toggle()                # action = a self-directed SEND
    return HANDLED
```

- **Event** = the `"missile hit"` message (a spell/projectile struck the wall
  feature).
- **Condition** = `not SELF.disabled()` — note the condition is *itself a
  message send* whose boolean return is consumed by `NOT`+`BRF`.
- **Action** = `SELF.toggle()` — re-dispatch to this object's own `toggle`
  handler. Return value signals "consumed."

This is a textbook lift: one guard, one action, one return. Nothing here tempts
a VM.

**The strain sibling — `levers."toggle"`** (same object, harder handler). In our
words it: reads the shared **`W:decflags`** extern (a packed bitfield holding the
render/animation state of *all* features), masks bit `0x10` to pick pull
direction, queries `SELF."get state"`, runs a tiny **arithmetic state machine**
over a private `staticVar0` (multi-state levers: clamp/wrap the stage counter),
writes the new packed state back into `W:decflags` via `SXW`, then fires three
follow-ups: `post_event(THIS, EV, state)` (enqueue an event for whatever this
lever is wired to), `SELF."sound effect request"`, and `SELF."update"`. Lifted:

```
on Toggle for Lever:
    dir   = (features.decflags & 0x10) ? -1 : 1     # SET, over an extern bitfield
    state = clamp_or_wrap(SELF.getState() + dir, 0, SELF.maxStage)   # small state machine
    features.decflags = (features.decflags & ~0x1F0) | pack(state, dir)  # SET extern
    SVC.post_event(SELF, EV_FEATURE_CHANGED, state)  # enqueue → drives wired doors
    SELF.soundEffectRequest()
    SELF.update()                                    # request redraw
```

Still `event → condition → action`, but it exposes the three real stress points:
(1) it **reads and writes another object's variable** (`features.decflags`); (2)
it does small **arithmetic on instance state**; (3) it **posts an event** rather
than calling the door directly — the lever↔door wiring is *indirect through the
event queue*. All three are absorbable (see §6/§7) but only if the IR carries
addressable object state, a `SET`/expression action, and an "emit event" action.

### 4b. `teleporters` → message `"trigger"` (the guard-heavy, escape-hatch case)

The `teleporters` base class imports party/entity/dungeon state by name
(`party_x`, `party_y` from `kernel`; `x`, `y`, `lvl`, `place` from `entities`;
`scrn`, `view`, `fade_request` from `dungeon`) and the graphics/interface
services `pixel_fade`, `wipe_window`, `hide_mouse`, `show_mouse`, `draw_bitmap`,
plus `post_event`/`notify`. Its `trigger` handler receives the **triggering
entity as an object-reference parameter** and is essentially a long guard
sequence followed by the teleport. In our words:

```
on Trigger(mover) for Teleporter:
    when mover.isNull() or mover == SELF or SELF.disabled():
        return                                   # ignore self / disabled
    kind = mover.report(WHAT_ARE_YOU)            # cross-object query (SEND "report")
    when kind == PARTY and mover.offBoard(): return
    when SELF.dest_lvl == mover.lvl and dungeon.impedance(mover, dest_x, dest_y) blocked:
        mover.collide(); return                  # blocked → bump, abort
    # --- the actual actions ---
    SET mover.x = SELF.dest_x ; mover.y = SELF.dest_y ; mover.lvl = SELF.dest_lvl
    SET mover.facing = SELF.dest_fdir            # (party dir only when specified)
    when mover == PARTY:
        SVC.pixel_fade(); SVC.wipe_window(); SET dungeon.fade_request = …   # screen transition
    SVC.post_event(SELF, EV_TELEPORTED, mover)
```

- **Event** = `"trigger"` with an **object-reference param** (`mover`).
- **Conditions** = a stack of guards mixing self-state, param identity, a
  **cross-object query** (`mover.report(...)`, `dungeon.impedance(...)`), and
  extern reads.
- **Actions** = write the mover's position externs (`SET`), then — for the party
  — a cluster of **engine-service** calls (`pixel_fade`/`wipe_window`/fade
  request) that are pure presentation, then emit an event.

The teleport *logic* is trivial (copy destination coords into the mover). What's
heavy is (a) the guard chain and (b) the presentation escape hatches. Both lift,
but (b) is where the closed-vocabulary rule is genuinely tested — see §6.

Also noted: `marble teleporter` overrides only `sound effect`, `timer` (a state
flip + `update`), and a `report` **`CASE` jump-table** that returns per-property
constants — i.e. subclasses are thin data/appearance overrides that **`PASS`**
everything structural to `teleporters`. This is the dominant subclass pattern.

---

## 5. Feasibility verdict

**Mostly yes — the model lifts.** Across every handler examined, a handler is a
shallow, terminating `event → (guards) → ordered side-effects → return`. There is
**no general computation, no unbounded recursion, no self-modifying dispatch**
that would force an interpreter. The object/message/event-queue model maps onto
`event → condition → action` with three named action kinds (`SEND`→message,
`CALL`→service, store→`SET`).

**The single biggest strain point is not control flow — it is object-relative
state and dispatch.** AESOP handlers routinely read/write *other objects'*
variables by name and send messages to other objects (and to a shared
event queue that other objects listen on). A trigger model that only knows
"this tile" and a flat blackboard cannot express `mover.x = SELF.dest_x` or
`lever posts an event that a wired door consumes`. If the IR lacks **entity
addressing + typed inter-object messages + a small event bus**, the importer
will be pushed toward smuggling a mini-interpreter back in to fake them. That is
the one place the "no VM" rule is actually at risk.

Estimated fit against MASTER's ≥95% target: comfortably met **for the logic**,
provided the vocabulary explicitly includes (a) inter-object messages/queries and
(b) named engine services for the presentation/IO calls. The long tail (save,
automap, DH file I/O) is not "script logic" at all and should be engine services,
not action-vocabulary entries.

---

## 6. Strain points, concretely

**Inheritance / `PASS` (override chains).** Real and pervasive: subclasses
override a handful of handlers and `PASS` the rest; base classes hold the
structural logic. **Absorb by lift-time flattening**, not runtime chains: the
importer resolves each `(objectType, message)` to a concrete handler by walking
`member` parents, and inlines `PASS` as "continue into the parent's lifted
handler." The IR sees a resolved handler per event; it never needs a live
inheritance graph. (This mirrors how our lift already must resolve names via the
import/export tables.) Watch item: a `PASS` in the *middle* of a handler (child
does X, PASSes, does Y) means the parent body inlines between the child's
pre/post actions — the lifter must preserve ordering, not just prepend/append.

**Arbitrary control flow (`BRT/BRF/BRA/CASE/JSR/RTS`).** Not arbitrary in
practice. Branches form reducible `if/guard/return` and small bounded loops →
lift to `when/else`, early-`return`, and (rarely) a bounded `repeat`. `CASE`
(0x03) is a jump table → lift to a `switch`/match, and in most sightings it backs
a **`report` property query** that is better lifted as a *data table* (property →
value) than as code. `JSR/RTS` (0x24/0x25) are intra-handler subroutines → inline
them. The one thing to build in the importer is a **structuring pass**
(basic-block + dominator reconstruction of `if/loop/switch` from the branch
soup); if that pass ever hits genuinely irreducible flow we fall back to a
labelled-goto action *inside a single handler only* — never a general VM.

**The ~178 runtime functions — closed vocabulary vs. escape hatches.** This is
the crux for the "closed vocabulary" rule, and it splits cleanly. The 128
documented AESOP/32 functions group by source module, plus ~46 Dungeon-Hack-only
functions:

| Group (module) | ~count | Lift target |
|---|---|---|
| `GRAPHICS.C` (draw_bitmap, pixel_fade, wipe_window, blit…) | 37 | **Engine services** (presentation) — *not* action vocabulary |
| `EYE.C` game-domain (change_level, step_X/Y/FDIR, distance, seek_direction, spell_request, magic_field, do_ice/do_dots, save_game, suspend/resume…) | 22 | **Named domain actions** — the real action vocabulary |
| `RTCODE.C` (string/math/util: string_compare, force_lower…) | 19 | **Pure expression built-ins** (used inside conditions/`SET`) |
| `INTRFACE.C` (mouse/window/cursor) | 12 | **Engine services** (UI) |
| `SOUND.C` (sound_effect, play_sequence, load_music…) | ~6–8 | **Engine services** (audio) — one `PlaySound`-ish action |
| `EVENT.C` (post_event, notify, dispatch, flush, peek, cancel) | 6–8 | **The event substrate itself** — becomes IR primitives (`emit`, `subscribe`), not user actions |
| `RTOBJECT.C` (create_object/destroy_object, create_program, cache) | 5 | **Spawn/despawn actions** + engine object-lifecycle |
| DH delta (open/close/create_file, save/explode_save, draw_walls, draw_auto_square, build_clipping, automap…) | ~46 | **Engine services** (persistence, dungeon/automap rendering) — never script vocabulary |

Takeaways: (1) The functions that look scary — rendering, automap, save/load,
file I/O — are **not** logic; they are **engine services** invoked by name. They
must exist as a *closed set of named service calls*, but they are the engine's
job, not the campaign author's action palette. (2) The genuine **action
vocabulary** the author sees is small and mostly `EYE.C` + object lifecycle:
change level, move/teleport, spell/field effect, spawn/destroy, play sound,
show text, gate on item, set flag, emit event. (3) `EVENT.C` and `RTCODE.C`
should **not** appear as "actions" at all — the former is the IR's own event
plumbing, the latter is expression-language built-ins.

**Cross-object state (`SEND` to others, extern vars).** The hardest part (see
§5). Handlers read/write `features.decflags`, `kernel.party_x`, `entities.x/y/lvl`
by name and send `report`/`impedance`/`collide`/`toggle` to other objects. Lift
requires: (a) **stable object/entity identity** in the IR (so `mover`, `SELF`,
`the wired door` are addressable); (b) **typed queries** (a `report`-style
read that returns a property — often lift the *target's* `report` to a data
table so the query becomes a field read); (c) a small **shared blackboard** for
the genuinely global bitfields like `decflags`, or better, per-feature state
fields the engine owns. Extern *writes* to shared engine state (party position,
fade_request) become `SET engine.<field>` service-mediated actions, not raw
memory pokes.

**Where someone is tempted to embed a VM, and how we avoid it.**
1. *`report`/`CASE` property handlers* look like "code to run." → Lift to
   **property tables**; don't execute them.
2. *`PASS` chains* look like they need a live dispatch engine. → **Flatten at
   import**.
3. *`decflags` bit-twiddling and small state machines* (multi-state levers) look
   like arbitrary arithmetic. → Model feature state as a **typed enum/counter
   field** with `advance/clamp/wrap` actions; the bitfield packing is an
   engine-internal detail of the original, not something the IR must reproduce.
4. *The event queue + `notify`* looks like it needs a scheduler VM. → It is just
   a **FIFO event bus with subscriptions**; implement it as an engine primitive
   (`emit`, `on`) — that is the IR's event layer, which we were building anyway.
   The temptation is to reimplement AESOP's dispatcher; instead we implement *an*
   event bus and lift `notify` registrations into IR subscriptions.

The recurring anti-pattern to resist: because handler bodies are literally
bytecode, the lazy path is "keep the bytecode, ship an interpreter." Every strain
above has a **declarative** answer; none requires a general evaluator.

---

## 7. What the IR event model MUST carry to absorb AESOP

These are the load-bearing requirements this spike surfaced (feed into IR design
before P1/P3):

1. **Entity/object identity & addressing.** Events, conditions, and actions must
   be able to name `SELF`, the **event source/param object** (`mover`, the
   triggering entity), and other referenced features. Without this, teleport,
   lever→door wiring, and every `report`/`collide` query are inexpressible.
2. **Typed inter-object messages / queries**, distinct from engine services. A
   `SEND` lifts to either `TARGET.message(args)` (fire) or a query
   (`TARGET.property` read — prefer resolving to a data field). Model both;
   keep them separate from `SVC.*`.
3. **An event bus primitive** (`emit(event, params)` + subscription binding),
   because AESOP wiring is indirect through `post_event`/`notify`, not direct
   calls. Event **parameters carry a typed payload**, including object refs and
   a small scalar (matches `post_event(owner,event,parm)` and `notify`'s two
   recipient variables).
4. **Ordered, side-effecting action lists with early-return and guarded
   branches** (`when/else`, `return`) — not a single flat "then." Handlers are
   sequences with mid-body guards.
5. **A `SET`/expression action** over addressable state (self statics, engine
   fields, shared flags) with **integer arithmetic + bitwise ops** in the
   expression language (levers/decflags need `& | << clamp`). Keep it an
   *expression* language (total, no loops), not a Turing-complete body.
6. **A closed, enumerated action vocabulary in two tiers:** (a) authoring-level
   **domain actions** (change level, teleport/move, spawn/destroy, spell/field,
   text, sound, set-flag, gate-on-item, emit-event); (b) **engine services**
   (render/automap/fade/mouse/save/file-IO) that logic may *invoke by name* but
   that are implemented by the engine and are **not** part of the campaign
   author's palette. Both are closed sets; distinguishing the tiers is what keeps
   the author-facing vocabulary small while still absorbing all ~178 functions.
7. **Feature/instance state as typed fields** (enum stage, boolean disabled,
   destination coords), so multi-state levers and teleporter destinations are
   data, not packed bit arithmetic to be re-executed.
8. **Import-time name resolution + flattening** is assumed: the importer resolves
   `.IMPT`/`.EXPT` names, walks `member` inheritance, inlines `PASS`/`JSR`, and
   runs a control-flow structuring pass. The IR receives resolved, flattened
   handlers — it must be *expressible enough* to hold their output, which is what
   items 1–7 guarantee.

**Closed-vocabulary risk flags.** (i) If we forget tier (b), authors of engine
services, presentation calls will look "un-enumerable" and invite a property bag
or a VM — enumerate them explicitly as services. (ii) The DH runtime delta (save
formats, automap, feature-file IO) is an *engine* surface; if it leaks into the
action vocabulary the closed set explodes. Keep it engine-side. (iii) A
`report`/`CASE` handler that we lift as *code* instead of a *table* is a small VM
in disguise — prefer tables.

---

## 8. Open questions

1. **Structuring-pass coverage.** All handlers seen were reducible. Do *any* of
   the 376 objects (esp. `kernel`, `dungeon`, combat/AI, spell resolution) have
   irreducible flow or deep loops that resist `if/loop/switch` reconstruction?
   Need a sweep across all code resources counting branch-graph reducibility
   before committing to "no goto action."
2. **How wide is the true action vocabulary?** This spike sampled feature
   interaction (levers/buttons/doors/teleporters). Combat, spellcasting
   (`spell_request`, `magic_field`, `do_ice`, `do_dots`), monster AI, and party
   management (`change_level`, save/resume) will add domain actions. Enumerate
   the full runtime-function usage across EYE.RES (and HACK.RES/OPEN.RES for DH)
   to size the closed set precisely.
3. **`decflags` semantics.** Confirm the exact bit layout and who owns it
   (`features` object) so we can decide between "typed per-feature state field"
   vs. "shared flag store." Affects requirement §7.5/§7.7.
4. **Event taxonomy.** Enumerate the application event types (`NAMES.H` in the
   original; recoverable from `notify`/`post_event` call sites and the message
   table, resource 4) so the IR's event enum is closed and complete, not guessed.
5. **`notify` timing/ordering guarantees.** AESOP re-queues input behind app
   events and dispatches FIFO. Does any logic depend on that exact ordering
   (e.g. a lever's `post_event` being processed before the next input)? If so the
   IR event bus must replicate the FIFO + re-queue discipline, not just "emit."
6. **DH generator surface.** Dungeon Hack adds ~46 functions and is procedurally
   generated; confirm none of them imply *runtime* generation logic that must be
   scripted (vs. import-time generation), which would be a different risk.

---

*Toolpath, disassembly, and doc extracts for this spike live in scratch only
(SSI-copyright-derived); reproduce with the built DAESOP as described in §1.*
