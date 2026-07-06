# ADR-0001 — IR serialization format

> **Status:** Proposed — firm recommendation, one taste call still open
> ([NEEDS HUMAN CONFIRMATION], see below). Awaiting the Phase 0 human-gated
> checkpoint (MASTER §12; PHASES "Phase 0" checkpoint).
> **Date:** 2026-07-06
> **Deciders:** maintainer (human-gated).
> **Resolves:** MASTER §11 open decision — *"IR serialisation format
> (human-diffable and versionable — favour text/typed schema over opaque
> binary)."*
> **Related:** MASTER §5 (IR design principles), §7.1/§7.2 (fail-closed, strong
> typing), §8 (correctness / golden references); ADR-0002 (event model, whose
> closed vocabulary this format must be able to express).

---

## Context

The IR is the product (MASTER §2). Four independent clients evolve against it —
importers, the engine, the editor, and third-party mods — so the on-disk form is
a real interface, not an implementation detail. MASTER §11 asks for a
**human-diffable, versionable** format and explicitly favours **text / typed
schema over opaque binary**. MASTER §5 adds three non-negotiable properties the
serialization must mechanically support:

1. **Versioned from commit zero** — importer, engine, editor, and mods evolve
   independently against the schema.
2. **Validated fail-closed on load** — unknown or invalid content is *rejected*,
   never best-effort rendered (also §7.1).
3. **Closed, enumerated vocabulary** for tiles, wall-decoration slots, and the
   event/condition/action language — *not* an open property bag (also §7.2:
   "illegal states unrepresentable").

Two further constraints come from the wider project:

- **Golden references** are kept per game and per level and are diffed/ratcheted
  (MASTER §8). A text form that produces small, stable diffs is a direct enabler
  of the correctness strategy, and of the lossless round-trip acceptance test
  (extract → edit → save → play).
- **Every dependency must be license-allowlisted** (CODING_STANDARDS "Dependency
  And License Rules"): Apache-2.0, BSD-2/3-Clause, EPL-2.0, MIT, PostgreSQL.
  GPL/AGPL/SSPL are blacklisted. Any library named here must sit on the
  allowlist.

The IR bodies are **deeply nested** (level → 32×32 grid → cells → 4 walls each →
archetype handles + decoration slots; plus event graphs), and are predominantly
**machine-emitted by the importers**, then diffed and occasionally hand-edited.
That shape favours a nesting-friendly text format bound to immutable typed
values, over a flat config-style format.

## Decision

**The canonical on-disk IR is strict, schema-validated, deterministically
formatted JSON, parsed and produced by Jackson, validated against JSON Schema,
and bound to immutable Java `record` types.**

Concretely:

1. **Format: JSON** (UTF-8, LF line endings, trailing newline per
   CODING_STANDARDS). One canonical text form per IR document.

2. **Binding library: Jackson** (`com.fasterxml.jackson.core:jackson-databind`
   and `jackson-annotations`) — **Apache-2.0, allowlisted.** IR values are
   immutable Java `record`s under `eu.virtualparadox…`. Records are already
   excluded from the JaCoCo coverage gate (CODING_STANDARDS "Coverage":
   `**/*Record.class`, `**/model/**`), so a record-per-IR-node style carries no
   coverage penalty.

3. **Validation library: `com.networknt:json-schema-validator`** (JSON Schema
   Draft 2020-12) — **Apache-2.0, allowlisted.** A machine-checked JSON Schema is
   the source of truth for the closed vocabulary and the required-field
   contract. `additionalProperties: false` at every object is the *mechanical*
   enforcement of MASTER §5's "not an open property bag."

4. **Deterministic serialization.** The writer emits keys in a fixed order
   (`SerializationFeature.ORDER_MAP_ENTRIES_BY_KEYS` + a stable property order on
   records) with `INDENT_OUTPUT` and a project-pinned pretty-printer, so the same
   IR always serializes byte-identically. This makes round-trip losslessness
   testable (MASTER §8) and keeps git diffs minimal — the "human-diffable"
   requirement, delivered by discipline rather than by format.

5. **Fail-closed load pipeline**, in order, aborting on first failure and
   rejecting the document (never partial render, MASTER §7.1):
   1. **Version gate first.** A mandatory top-level `irVersion` (semver string)
      is read before anything else; an unrecognised **major** version is
      rejected.
   2. **Structural parse** (Jackson, `FAIL_ON_UNKNOWN_PROPERTIES = true`).
   3. **Schema validation** (JSON Schema): required fields, `additionalProperties:
      false`, and closed-enum membership. Unknown enum values **hard-fail** —
      `READ_UNKNOWN_ENUM_VALUES_AS_NULL` / `…_USING_DEFAULT_VALUE` are **off**.
   4. **Binding** to records.
   5. **Semantic / referential validation** — cross-references resolve, invariants
      hold (e.g. every wall archetype handle exists, every `Call`-target
      action-sequence is defined — see ADR-0002).

6. **Versioning policy.** `irVersion` is semver. **Major** = breaking schema
   change; the loader rejects majors it does not implement. Minor/patch are
   backward-compatible additions. Importer, engine, editor, and mods each declare
   the IR version they target. Versioned from commit zero (MASTER §5).

7. **[NEEDS HUMAN CONFIRMATION] — JSON vs YAML vs TOML is a taste call.** The
   *engineering* recommendation is firm (typed text + Jackson + JSON Schema +
   deterministic writer + fail-closed loader); the specific **surface syntax** is
   the maintainer's readability/authoring-ergonomics call. JSON is recommended
   because it is unambiguous, footgun-free, and validates cleanly against JSON
   Schema. The two credible alternatives — YAML and TOML, both reachable via
   Jackson dataformat modules (**Apache-2.0, allowlisted**) with the *same*
   record binding and the *same* JSON-Schema validation — trade differently on
   comments and readability (see Alternatives). If chosen, the canonical form
   simply switches dataformat module; everything else in this ADR is unchanged.

## Rationale

- **Typed text over opaque binary** (MASTER §11) — records + Jackson give a typed
  in-memory model *and* a diffable text form, with no bespoke binary reader in
  the engine (which would also risk leaking a format into the runtime, MASTER
  §7.3/§5).
- **Closed vocabulary is enforced twice, structurally** — once by JSON Schema
  (`additionalProperties: false`, enum membership) and once by the Java type
  system (records + enums, "illegal states unrepresentable", MASTER §7.2). This is
  exactly the "constraints, not goodwill" posture of MASTER §7.4.
- **Fail-closed is the default, not a mode** — Jackson's `FAIL_ON_UNKNOWN_*` plus
  hard enum failure plus schema `additionalProperties: false` means an unknown
  construct cannot be silently tolerated (MASTER §7.1). Mods especially are
  untrusted input and are rejected on any unknown construct.
- **Human-diffable is achieved by determinism, not by the format alone** — a
  canonical writer (sorted keys, stable indentation) yields minimal, reviewable
  diffs and byte-stable round-trips, which is what the golden-reference ratchet
  and the lossless round-trip test (MASTER §8) actually need.
- **License-clean** — Jackson and networknt json-schema-validator are both
  Apache-2.0 (allowlisted); no GPL/AGPL/SSPL touch the dependency graph.
- **Ecosystem fit** — Java 21 + Jackson is the maintainer's day-job stack and the
  `darkmoor` reference's neighbourhood; no exotic tooling.

## Consequences

- **Positive.** One typed schema drives all four clients. Diffs are small and
  reviewable; golden references are plain text. Fail-closed validation is
  mechanical and centralised in the `ir` module's loader. Records are
  coverage-exempt, so a fine-grained node-per-record model is free under the
  gate.
- **Cost — canonical formatting must be enforced.** JSON allows many equivalent
  encodings; the round-trip guarantee holds only if the writer is canonical.
  Mitigation: a single pinned `ObjectWriter` in the `ir` module is the *only*
  sanctioned emitter, and a formatting check runs in CI on hand-edited/golden
  files (Spotless can host a JSON formatting step; the deterministic writer is
  the reference).
- **Cost — JSON has no comments.** Machine-emitted bodies do not need them; for
  authored notes, an explicit `description` / `notes` field convention is added
  where it matters (and the editor, MASTER §10 / PHASES P5, is the real authoring
  surface, not a text editor). If comment support is judged essential, that is
  precisely the trigger to reopen the taste call in favour of YAML/TOML.
- **Cost — two schema artefacts to keep in step** (the Java records and the JSON
  Schema). Mitigation: treat the JSON Schema as generated-from or checked-against
  the records in CI, so they cannot drift silently.
- **Neutral.** Palantir Java Format governs the record classes; the JSON text is
  governed by the canonical writer / Spotless JSON step — two formatters, two
  scopes, no conflict.

## Alternatives considered

- **Opaque binary (e.g. Protobuf/FlatBuffers/custom).** Rejected on MASTER §11's
  face ("favour text/typed over opaque binary"): not human-diffable, weak golden
  references, and a binary reader is a format-shaped thing that risks leaking into
  the engine (MASTER §7.3). Note Protobuf is BSD-3-Clause (allowlisted) — so this
  was rejected on *design fit*, not on licensing.
- **YAML** (`jackson-dataformat-yaml`, Apache-2.0, allowlisted; backed by
  SnakeYAML, Apache-2.0). More readable and comment-friendly for nested content,
  but its implicit typing (the "Norway problem", octal/sexagesimal coercions,
  indentation sensitivity) is a hazard for a format that must be deterministic and
  round-trip losslessly. Held as the leading alternative if readability/comments
  outweigh determinism — the [NEEDS HUMAN CONFIRMATION] call.
- **TOML** (`jackson-dataformat-toml`, Apache-2.0, allowlisted). Comment-friendly
  and unambiguous, but awkward for the IR's deep nesting (levels → grid → cells →
  walls). Plausible for top-level campaign/config documents, but a poor single
  canonical form for the nested level bodies.
- **XML.** Verbose, noisy diffs, heavier authoring; no advantage here over JSON.
- **A bespoke DSL / custom text grammar.** Maximum readability, but a hand-written
  parser plus editor tooling is a large surface to build and secure, contradicts
  MASTER §5's "no speculative abstractions ahead of need," and buys nothing that a
  typed JSON schema + editor does not.

## Open questions

- **The surface-syntax taste call** (JSON vs YAML vs TOML) — flagged
  [NEEDS HUMAN CONFIRMATION] above. Everything else in this ADR is format-agnostic
  across the three Jackson dataformats.
- **JSON Schema authoring vs generation** — hand-author the schema, or generate it
  from the Java records (or vice versa)? Decide before the schema and records can
  drift (resolve alongside the `ir` module in PHASES P0/P1).
- **Canonical-form CI enforcement** — exact Spotless/formatter configuration for
  the text files (the deterministic writer is the reference implementation).
- **Compression/packaging for shipped campaigns** — canonical JSON is the *source*
  form; whether distribution bundles are zipped (text preserved inside) is a
  packaging question, out of scope for the IR format itself.

## Decision confirmed (2026-07-06)
Surface syntax = **JSON** (the recommended option), per the maintainer. The record binding, schema validation, and the `additionalProperties:false` closed-vocabulary enforcement are unchanged. **Status: Accepted.**
