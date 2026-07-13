# Increment: `from-context` ingest — adopt an existing technical context

Status: archived
Date: 2026-07-13
Type: New capability
Owner: migration-engineer

> Implements **Phase 3** of the parent design/plan
> `specs/increments/INC-20260711-spec-sdd-migration-plan/README.md`
> (§Proposed design (3) New `from-context` / ingest-context mode).

## Goal

Add `axiom bootstrap from-context <path>`: point Axiom at an **existing
technical context** (an `ARCHITECTURE.md`, a `docs/**` tree, an `adr/` set)
that predates Axiom, and adopt it into a navigable, **MCP-queryable**
`technical-context/` tree plus a draft `TechnicalContextIndex`.

The command mirrors the shape of `bootstrap-from-code/` (ingest → map →
index) but sources from **existing docs** instead of mechanical facts, and
reuses `buildTechnicalContextIndex` from `bootstrap-from-code/index-builder.ts`
**verbatim** plus the exact from-code write plumbing (`writeGuardedFile` +
`technicalContextIndexPath`). Because the generated index conforms to
`@axiom/technical-context`'s `validateTechnicalContextIndex`,
`spec.technicalContextIndexRead` (MCP) serves it immediately — MCP-queryability
is free, no new MCP surface needed.

## Scope

- A new `apps/cli/src/bootstrap-from-context/` module:
  - **ingest** (replaces `analyzer.ts`): discover existing context docs in the
    source path — `ARCHITECTURE.md`, README architecture sections, `docs/**`
    trees, `adr/`/`docs/adr/` — and classify each by convention into the
    `technical-context-template.md` subfolder taxonomy (architecture /
    conventions / operations / testing / integrations / references). Respects
    ignore lists (mirrors `analyzer.ts`'s `IGNORED_DIR_NAMES`) and bounds the
    scan depth.
  - **map** (replaces `drafter.ts`): copy each discovered doc into
    `technical-context/<category>/<slug>.md`, prepending an `AXIOM:MIGRATED`
    provenance banner naming the source path (distinct from from-code's
    `AXIOM:DRAFT`). Produces one `DraftedDocument`-shaped entry per doc.
  - **index** (reuses `buildTechnicalContextIndex` verbatim): `available`-only,
    `mandatory.always`/`whenTags` empty, `status: 'draft'`.
- The `from-context` subcommand in `commands/bootstrap.ts`, resolving the spec
  scope exactly like `from-code` (`resolveSpecScopeAbsolutePath`), writing via
  `writeGuardedFile` + `technicalContextIndexPath`, and supporting `--dry-run`.

## Non-goals

- **No spec-artifact migration.** Migrating foreign increments/bugs/plans/adr/
  decisions into canonical `specs/**` artifacts is subject A / the detector
  layer, handled by the separate spec-adopter work (INC-MIG-SPEC / Phases 1-2),
  not here.
- **No idempotency / source→artifact provenance map.** Re-runs are made safe by
  a simple non-existence gate (skip-if-exists) only; the provenance map + true
  idempotent reconciliation is **Phase 4**.
- **No install-time adoption UX.** Wiring `from-context` into `axiom workspace
  setup` / the init wizard is **Phase 5**.
- **No semantic/architectural understanding pass.** `from-context` ingests
  *existing docs* verbatim (with a provenance banner); it does not re-derive
  architecture from source code.
- **No auto-approval.** Everything lands `status: draft`, `available`-only.

## Acceptance (copied from the parent plan's Phase 3)

- Pointing at a repo with `ARCHITECTURE.md` + `docs/**` produces
  `technical-context/<role>/*.md` + a valid `TechnicalContextIndex`
  (`status: draft`, `available`-only, `mandatory.*` empty) that
  `spec.technicalContextIndexRead` serves over MCP.
- Every generated doc carries the `AXIOM:MIGRATED` banner (distinct from
  from-code's `AXIOM:DRAFT`).
- Re-run does not clobber human-edited docs.

### Corner cases

- No context docs found → no-op + explicit message (not a failure).
- A doc that fails to read → recorded as a failure, scan **continues** (mirrors
  `scanLegacyRepo`'s failure posture).
- Doc name collisions → deterministic slug + skip-if-exists (guarded write).

## Result

**Delivered and green.** Shipped `axiom bootstrap from-context <path>
[--role <role>] [--dry-run]`.

- New module `apps/cli/src/bootstrap-from-context/`:
  - `ingest.ts` (mirrors `analyzer.ts`): discovers `ARCHITECTURE.md`, a root
    README architecture section, and `docs/`/`doc/`/`adr/` trees; classifies
    each by convention into architecture / conventions / operations / testing /
    integrations / references; ignore-list (mirrored from `analyzer.ts`) and
    depth-bounded; read-failures recorded, scan continues.
  - `map.ts` (mirrors `drafter.ts`): copies each doc into
    `technical-context/<category>/<slug>.md` with an `AXIOM:MIGRATED` banner
    (distinct from from-code's `AXIOM:DRAFT`) naming the source path;
    deterministic, injective slug.
  - `index-builder.ts`: **re-exports `buildTechnicalContextIndex` from
    `bootstrap-from-code/index-builder.ts` VERBATIM** (a test asserts
    referential identity) → `available`-only, `mandatory.*` empty,
    `status: draft`.
- `commands/bootstrap.ts`: `runBootstrapFromContext` resolves the spec scope
  with the same `resolveSpecScopeAbsolutePath` helper as from-code and writes
  via `writeGuardedFile` + `technicalContextIndexPath`; per-doc and per-index
  non-existence gate (skip-if-exists, never clobber); `--dry-run` previews.

### Acceptance — met

- `ARCHITECTURE.md` + `docs/**` + `adr/` → `technical-context/<category>/*.md`
  + a valid `TechnicalContextIndex` (`status: draft`, `available`-only,
  `mandatory.*` empty). Verified read-back through the actual MCP handler
  `getTechnicalContextIndex` (backing `spec.technicalContextIndexRead`).
- Every generated doc carries `AXIOM:MIGRATED` (asserted distinct from
  `AXIOM:DRAFT`).
- Re-run does not clobber human-edited docs (verified via CLI e2e).
- Corner cases: no docs → no-op message; unreadable doc → failure recorded,
  scan continues; slug collision / pre-existing target → skip.

### Gate

- `npm run build`: PASS
- `npm test`: 2681 passing (2658 baseline + 23 new), 0 failures
- `npm run typecheck`: PASS
- `npm run doctor`: PASS (0 failures)

### Deferred (per Non-goals)

- Idempotent source→artifact provenance map — Phase 4.
- Install-time adoption UX (`axiom workspace setup` branch) — Phase 5.
- Foreign spec-artifact migration + detector layer — INC-MIG-SPEC / Phases 1-2.
