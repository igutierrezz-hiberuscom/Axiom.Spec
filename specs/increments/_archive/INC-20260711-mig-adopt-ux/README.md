# Increment: Install-time adoption UX for existing spec/sdd repos

Status: archived
Date: 2026-07-13
Type: New capability
Owner: migration-engineer

> Implements **Phase 5** (the LAST phase) of the parent design/plan
> `specs/increments/INC-20260711-spec-sdd-migration-plan/README.md`
> (§Proposed design (4) Install-time integration + §B Foreign SDD/tooling repo
> → Axiom-conformant control repo). Phases 0-4 are shipped and archived:
> spec-scope routing, the pluggable detector layer + structural conversion
> (INC-MIG-SPEC), `from-context` ingest (INC-MIG-CONTEXT), and the idempotent
> source→artifact provenance map (INC-MIG-IDEMPOTENCY).

## Goal

At install time, let a user point Axiom at an **existing spec repo** and/or an
**existing SDD/control repo** that predate Axiom, see a **dry-run preview**
(counts, detected format(s), per-item resolved status, and a subject-B
**conformance report**), **confirm**, and end with content that is **migrated
and MCP-queryable** with the same guarantees as natively-created Axiom content.

The adoption flow is an **orchestrator** over the already-shipped primitives —
it introduces **no new write primitive**:

- **Subject B (SDD/control repo)** — *additive parametrization, NOT conversion*:
  run `runWorkspaceSetup` in adopt mode (`create: false`), relying on its
  existing no-clobber guards, then emit a **conformance report** (present /
  added / must-reconcile-by-hand) over the plan's subject-B table.
- **Subject A (spec repo)**: after `scaffoldSpecRepoBase` establishes the base
  (only when the spec repo is freshly created — never clobbering an existing
  base), invoke the detector-driven migration (`runBootstrapFromLegacySdd`) and,
  if requested, the `from-context` ingest (`runBootstrapFromContext`).

## Scope

- A new adoption entry point exposed as **additive flags on `axiom workspace
  setup`** (Q-005): `--adopt-spec <path>`, `--adopt-sdd <path>`,
  `--ingest-context <path>`, `--dry-run`, and `-y/--yes` (confirmation).
- A pure orchestrator `runWorkspaceAdopt` (`apps/cli/src/commands/workspace-adopt.ts`)
  that:
  1. **Collects** the foreign repo path(s) and whether each is a spec repo, an
     SDD/control repo, or both. `--adopt-sdd <path>` makes the control repo
     that (foreign) path (`create: false`); `--adopt-spec <path>` makes the spec
     repo that (foreign) path (`create: false`) and marks its content as the
     subject-A migration source.
  2. Builds a **read-only preview**: subject-B conformance prediction
     (filesystem probes), subject-A scan (`scanLegacyRepo` — detected format(s),
     item counts, per-item resolved status via `resolveDisplayStatusForMigration`),
     and context ingest preview (`ingestContextDocs`).
  3. Gates writes on **dry-run-first + confirmation**: `--dry-run` = zero writes,
     full preview; no `--yes` (and not `--dry-run`) = preview + "confirmation
     required" + zero writes (this is the decline path); `--yes` = execute.
  4. On confirmation: `runWorkspaceSetup` (subject B) → subject-A migration →
     context ingest, resolving the spec scope from the (now-parametrized)
     control repo. Emits the actual conformance report by diffing a pre-setup
     snapshot against a post-setup snapshot.
- The subject-B **conformance report** covers the plan's six concerns
  (AGENTS.md, axiom.yaml v2, topology.yaml, skills-index, `.axiom/mcp.yml`,
  architect declarations) with status `present` | `added` |
  `must-reconcile-by-hand`.

## Non-goals

- **No new migration engine.** Detection, conversion, context ingest, and
  provenance/idempotency are Phases 1-4 (shipped). Phase 5 only wires them into
  the install-time UX.
- **No change to `runWorkspaceSetup`'s public signature.** Adoption is additive;
  the normal (no-adoption-flag) setup path is unchanged, keeping the existing
  setup suites green.
- **No auto-approval.** Every migrated artifact/context doc lands draft/proposed
  with an `AXIOM:MIGRATED` banner, exactly as the underlying commands produce.
- **No interactive TTY prompt.** `axiom workspace setup` is headless; the
  confirmation gate is expressed by `--yes` (present = confirmed) so the flow is
  fully scriptable and deterministically testable.

## Acceptance (from the parent plan's Phase 5)

- From a fresh install, a user can point Axiom at an existing spec repo and/or
  SDD repo, see a dry-run summary, confirm, and end with **migrated +
  MCP-queryable** content.
- All four **preserved guarantees** verified by tests:
  - **No-clobber**: never overwrite an existing artifact, a foreign `axiom.yaml`
    of another project, or an existing technical-context doc/index.
  - **Provenance**: every migrated README/context doc carries `AXIOM:MIGRATED`;
    re-run is idempotent (already-migrated skipped).
  - **Dry-run**: zero writes; reports exactly what would be created, at what
    status, plus the conformance report.
  - **Collision-skip**: one bad/colliding entry is skipped with a clear reason;
    never aborts the batch.
- A **partially-Axiom** repo is adopted without clobbering its existing Axiom
  parts (only foreign/non-conforming items migrated).

### Corner cases

- Adopting a repo that is **already a valid Axiom project** → idempotent no-op +
  report (no clobber).
- User **declines** at the confirmation step (no `--yes`) → zero writes.
- **Empty repo** → no-op + clear message.

## Preserved guarantees (how they are enforced)

The adoption flow does not re-implement any guarantee; it **composes** the
already-guaranteed primitives and adds a write gate on top:

- **No-clobber** is enforced by `writeOneRepo` (foreign `axiom.yaml` of another
  project → skip-with-warning), `migrateLegacyItems` (`artifactExists` → skip),
  and `runBootstrapFromContext`'s per-doc/per-index non-existence gate. The
  orchestrator additionally **preserves a hand-written `.axiom/mcp.yml`**
  (restores it if setup's regeneration would change it) and flags it
  `must-reconcile-by-hand`.
- **Provenance** is enforced by the migrator's `AXIOM:MIGRATED` banner + the
  Phase-4 source→artifact provenance map (`technical-context/.migration-provenance.yml`),
  which both subject A and context ingest thread; re-runs skip already-migrated
  sources.
- **Dry-run** is enforced by building the entire preview from read-only probes
  and only invoking the (writing) `runWorkspaceSetup` after an explicit
  confirmation.
- **Collision-skip** is the migrator's/ingest's existing posture (a colliding
  entry is skipped with a reason, the batch continues).

## Result

**Delivered and green.** Shipped install-time adoption as additive flags on
`axiom workspace setup`: `--adopt-spec <path>`, `--adopt-sdd <path>`,
`--ingest-context <path>`, `--dry-run`, `-y/--yes`.

- New orchestrator `apps/cli/src/commands/workspace-adopt.ts` (`runWorkspaceAdopt`):
  builds a read-only preview (subject-B conformance prediction + subject-A
  `scanLegacyRepo` with per-item resolved status + `ingestContextDocs` preview),
  then on confirmation runs `runWorkspaceSetup` (subject B) →
  `runBootstrapFromLegacySdd` (subject A) → `runBootstrapFromContext` (context),
  resolving the spec scope from the parametrized control repo. Emits a
  present/added/must-reconcile-by-hand conformance report over the six subject-B
  concerns by diffing a pre-setup vs post-setup control snapshot.
- `apps/cli/src/commands/workspace.ts`: `handleWorkspaceSetup` routes to
  `runWorkspaceAdopt` when any adoption flag is present; the normal setup path is
  unchanged. `--spec-path` relaxed from `requiredOption` to `option` (still
  required unless `--adopt-spec` is given; validated in the handler).
  `runWorkspaceSetup`'s public signature is unchanged.
- `apps/cli/src/bootstrap-from-legacy-sdd/migrator.ts`: added
  `isMigrationOutputBody` (exported) + a guard so `migrateLegacyItems` never
  re-migrates its own prior output (a body carrying the `AXIOM:MIGRATED`
  banner). This is what makes **adopt-in-place** (foreign spec repo === resolved
  spec scope, so the canonical `<specScope>/{increments,adr,…}/` folders are
  re-scanned by the axiom-native detector on a re-run) idempotent. The dry-run
  preview (`commands/bootstrap.ts`) and the adoption preview honor the same
  guard.

### Acceptance — met

- Fresh install adopting an existing spec repo (+ `--ingest-context`): dry-run →
  confirm → migrated content; the technical-context index READS BACK through
  `loadTechnicalContextIndex` (the loader backing `spec.technicalContextIndexRead`).
  Verified by a real scratch e2e (built CLI) and by `tests/e2e/workspace-adopt.e2e.test.ts`.
- All four preserved guarantees verified by `tests/workspace-adopt.test.ts`:
  no-clobber (foreign `axiom.yaml` of another project preserved + reconcile;
  hand-written `.axiom/mcp.yml` preserved + reconcile; pre-existing
  technical-context doc not clobbered), provenance (`AXIOM:MIGRATED` banner +
  idempotent re-run), dry-run (zero writes), collision-skip (skip-if-exists).
- Corner cases: already-Axiom repo → no-op + report; decline (no `--yes`) → zero
  writes; empty repo → no-op.

### Gate

- `npm run build`: PASS
- `npm test`: 2716 passing (2706 baseline + 10 new), 0 failures
- `npm run typecheck`: PASS
- `npm run doctor`: PASS (0 failures)
