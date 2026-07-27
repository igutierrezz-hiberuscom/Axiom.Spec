# Increment: Topología de repo único gestionado (`<project>.axiom`)

Status: closed
Date: 2026-07-24

## Goal

Evolve the `@axiom/topology` schema so Axiom supports a SINGLE managed
`<project>.axiom` repo model (the target topology for installed projects),
able to coexist with — and eventually supersede — the current
`sddRepo`+`specRepo` split, while staying fully backward compatible with
every existing `topology.yaml` (schemaVersion 1).

This is **INC-A1** of Cluster A in the Axiom evolution plan
(`giggly-imagining-moore.md`, "Arquitectura objetivo"): the base increment
that the rest of Cluster A (adoption flow, provenance, export, unified MCP)
builds on. It is an explicit, user-approved graduation to the full product
lifecycle (per `Axiom.SDD/AGENTS.md`'s "Explicit Bootstrap Limits" — the
managed single-repo model is the deliberate exception this plan requests).

## Context

Today, `packages/topology/src/types.ts` + `axiom.config/topology.yaml` only
support `sddRepo`/`specRepo`/`roleCodeRepositories` (no `kind`/`mode`
discriminators on `RepoRef`, no concept of a single control+knowledge repo,
no concept of a read-only legacy-source bucket). `TopologyManifest` is
`schemaVersion: 1`-only. Validation (`validateTopology`) checks repo-id
uniqueness, `roleCodeRepositories` coverage by `assignments[]`, and
`unknown-role`/`overlap-warning`. The CLI surface `axiom topology
show|validate` (`apps/cli/src/commands/topology.ts`) is a thin wrapper over
`loadTopology`/`validateTopology`.

The target architecture ("Arquitectura objetivo" in the plan) replaces the
`sddRepo`+`specRepo` pair with **one** `<project>.axiom` repo — the whole
"brain" (`spec/`, `increments/`, `bugs/`, `adr/`, `technical-context/`,
`logs/` + `skills/`, `adapters/`, `commands/`, `rules/`) — plus `codeRepos[]`
(role-owned code repositories) and `legacyRepos[]` (pre-existing project
source repos Axiom never writes to). This increment introduces the SCHEMA
for that model; it does not implement the adoption flow that actually
creates a `<project>.axiom` repo on disk (that is INC-A2), nor provenance
metadata (INC-A3), export/rollback (INC-A4), or the unified MCP (INC-A5).

This workspace's own `axiom.config/topology.yaml` (`sddRepo: ../Axiom.SDD`,
`specRepo: ../Axiom.Spec`) is intentionally left on schemaVersion 1 — the
plan explicitly defers dogfooding this migration to the user; the managed
model is for projects Axiom installs into, not Axiom's own repos.

## Scope

- `packages/topology/src/types.ts`: add `RepoKind`
  (`'axiom' | 'code' | 'legacy'`) and `RepoMode` (`'read-only-source'`)
  discriminators on `RepoRef`; add `axiomRepo?`, `codeRepos?`,
  `legacyRepos?` to `TopologyManifest`; bump `schemaVersion` to `1 | 2`; add
  `TopologyManifestInput` (the pre-normalization shape); add
  `'invalid-repo-kind'` and `'deprecated-legacy-shape'` to `TopologyFinding`.
- `packages/topology/src/normalize.ts` (new): `normalizeTopologyManifest`
  (mirrors the legacy shape ↔ managed shape so both are always populated)
  and `isLegacyTopologyShape`.
- `packages/topology/src/loader.ts`: schemaVersion-aware shape guard
  (`isManifestLike`); accept `schemaVersion: 1 | 2`; call the normalizer
  before returning; route the two default-manifest builders
  (`defaultSingleRepoManifest`, `defaultInstalledMultiRepoManifest`) through
  the normalizer too.
- `packages/topology/src/validate.ts`: `invalid-repo-kind` error (kind
  doesn't match its bucket), `deprecated-legacy-shape` warning (schemaVersion
  1), `legacyRepos[]` duplicate-id checks.
- `packages/topology/src/index.ts`: export the new types/functions.
- `apps/cli/src/commands/topology.ts`: `topology show` surfaces
  `schemaVersion`, `axiom-repo`, `legacy-repos`, and a legacy-shape note.
- `axiom.config/topology.yaml`: documentation-only comment (no structural
  change — see Context).
- Tests: `packages/topology/tests/{normalize,managed-model}.test.ts` (new),
  targeted fixes to pre-existing assertions in
  `packages/topology/tests/topology.test.ts` and
  `apps/cli/tests/topology.test.ts` that the new `deprecated-legacy-shape`
  warning affects, plus new CLI-level `topology show` scenarios for the
  managed shape.

## Non-goals

- The adoption flow that creates a real `<project>.axiom` repo on disk
  (`workspace-adopt.ts`, `bootstrap-from-legacy-sdd/*`) — INC-A2.
- Provenance/lifecycle metadata, migration manifest — INC-A3.
- Rollback/export ("leaving Axiom") — INC-A4.
- The unified `<project>.axiom` MCP broker — INC-A5.
- Migrating Axiom's own `Axiom.SDD`/`Axiom.Spec` repos to the managed model
  (explicitly deferred to the user; see Context).
- A schema for `topology.yaml` inside `@axiom/config-validation` (see
  Assumptions — none exists today for `topology.yaml`; not introduced here).
- Any change to `roles.ts`/`bindings.ts` (they consume
  `sddRepo`/`specRepo`/`roleCodeRepositories`, which are unchanged and
  always populated post-normalization — no edit needed).

## Acceptance criteria

- [x] New schema accepted: a schemaVersion 2 document with `axiomRepo` (+
      optionally `codeRepos[]`/`legacyRepos[]`) loads and validates.
- [x] Legacy `sddRepo`/`specRepo` (schemaVersion 1) config still validates,
      is auto-mapped to the managed model's fields, and
      `validateTopology` emits a non-blocking `deprecated-legacy-shape`
      warning.
- [x] `kind`/`mode` discriminators exist on `RepoRef`, are shape-validated
      (closed enums), and cross-checked against their bucket
      (`invalid-repo-kind`) by `validateTopology`.
- [x] Mixed/invalid configs are rejected with clear errors: schemaVersion 2
      without `axiomRepo`; unsupported schemaVersion; invalid `kind`/`mode`
      enum values; mismatched `kind` for a bucket; duplicate ids.
- [x] Edge cases covered: config with only `axiomRepo`; legacy
      `sddRepo`+`specRepo` (both the lossless same-ref case and the
      genuinely split case); missing/relative/absolute refs; `legacyRepos`
      default to `mode: 'read-only-source'`.
- [x] `npm run build` (tsc -b) passes.
- [x] Targeted `vitest run` (topology, config-validation, CLI topology
      tests) passes; every failure classified pre-existing vs new.
- [x] `axiom topology show` surfaces the managed-model fields.
- [x] `@axiom/cli-commands` single-ownership preserved (no relative imports
      from `apps/cli` into package source) — N/A this increment touched no
      files re-exported through that package; verified by grep.
- [x] `@axiom/tui` stayed generic — not touched (its `TopologyFinding.kind`
      handling was already a generic `string`, so it needed no change for
      the two new finding kinds).

## Assumptions

Each is a deliberate, narrowest-change resolution of an ambiguity the task
left open (per the "resolve ambiguity yourself" guardrail) — see the
executor's final report for the one-line reasoning behind each:

1. `sddRepo`/`specRepo`/`roleCodeRepositories`/`assignments` stay REQUIRED
   (non-optional) on `TopologyManifest`; the new managed fields
   (`axiomRepo`/`codeRepos`/`legacyRepos`) are ADDITIVE and optional. This
   is what let every existing consumer (`roles.ts`, `bindings.ts`,
   `@axiom/tui`, `@axiom/doctor`, `@axiom/mcp-server`, `@axiom/memory`,
   `@axiom/workflow`) compile unchanged.
2. `codeRepos[]` entries are associated with a role via the EXISTING
   `assignments[]` mechanism (`repoId` → `roleId`) — no inline `role` field
   duplicating that source of truth.
3. `axiomRepo` is derived from a legacy `sddRepo`/`specRepo` pair only when
   lossless (`sddRepo.ref === specRepo.ref` — the common single-repo case);
   a genuinely split pair (e.g. this workspace's own `../Axiom.SDD` +
   `../Axiom.Spec`) is left `undefined` rather than fabricating a merged
   path. The `deprecated-legacy-shape` warning fires either way.
4. The `deprecated-legacy-shape` warning fires for EVERY `schemaVersion: 1`
   manifest uniformly (explicit file or built-in default) — simplest,
   single code path; the message is worded to be accurate/non-alarmist for
   both.
5. `axiom.config/topology.yaml` (this workspace's own config) is left
   structurally on schemaVersion 1 (documentation comment only) — changing
   its shape would constitute migrating Axiom's own repos, explicitly out
   of scope.
6. No new schema was added to `@axiom/config-validation` for
   `topology.yaml` — none exists there today (topology validation is
   fully self-contained in `@axiom/topology`), and nothing wires the
   zod-based validator package into the topology load path; adding one
   would be unused/speculative code.

## Implementation notes

Files changed (all in `Axiom/`, the product monorepo):

- `packages/topology/src/types.ts` — `RepoKind`, `RepoMode`, `RepoRef.kind`/
  `.mode`, `TopologyManifest.axiomRepo`/`.codeRepos`/`.legacyRepos`,
  `schemaVersion: 1 | 2`, `TopologyManifestInput`, `TopologyFinding`
  `'invalid-repo-kind'` / `'deprecated-legacy-shape'`.
- `packages/topology/src/normalize.ts` (new) — `normalizeTopologyManifest`,
  `isLegacyTopologyShape`, `AXIOM_REPO_DEFAULT_ID`.
- `packages/topology/src/loader.ts` — schemaVersion-aware `isManifestLike`;
  `VALID_REPO_KINDS`/`VALID_REPO_MODES` enum checks inside `isRepoRefLike`;
  `loadTopology` accepts `1 | 2` and normalizes before returning; both
  default-manifest builders route through the normalizer.
- `packages/topology/src/validate.ts` — `legacyRepos[]` dup-id checks
  (§1b), `invalid-repo-kind` via a `checkBucketKind` helper (§5, checks
  `roleCodeRepositories` only — NOT also `codeRepos`, to avoid double-
  reporting the same mismatch once each mirror after normalization),
  `deprecated-legacy-shape` warning (§6).
- `packages/topology/src/index.ts` — barrel exports for the above.
- `apps/cli/src/commands/topology.ts` — `formatTopologyShow` additive
  lines (`schemaVersion`, `axiom-repo`, `legacy-repos`, legacy-shape note);
  all pre-existing lines preserved verbatim.
- `axiom.config/topology.yaml` — comment only.
- `packages/topology/tests/normalize.test.ts` (new).
- `packages/topology/tests/managed-model.test.ts` (new).
- `packages/topology/tests/topology.test.ts` — one pre-existing assertion
  updated (see Assumption 4).
- `apps/cli/tests/topology.test.ts` — one pre-existing assertion updated;
  two new scenarios (schemaVersion 2 `topology show`, legacy-shape note).

Design note on the mirroring contract: `codeRepos[]` and
`roleCodeRepositories[]` are kept as two mirrors of the SAME list
post-normalization (by id), never independently validated for duplicates
against each other (that would false-positive on every valid manifest).
`sddRepo`/`specRepo` similarly mirror `axiomRepo` when derived from it (with
conventional, distinct ids `sdd-repo`/`spec-repo` — never reusing
`axiomRepo`'s own id — to avoid a false "duplicate id" from the
pre-existing cross-list dup-scan).

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0.
- Targeted `npx vitest run packages/topology apps/cli/tests/topology.test.ts
  apps/cli/tests/bindings.test.ts apps/cli/tests/roles.test.ts
  packages/config-validation packages/doctor/tests/topology.test.ts
  packages/tui/tests/topology.test.ts` — **105/105 tests passed** across 10
  files (includes the 2 new test files: `normalize.test.ts` 10 tests,
  `managed-model.test.ts` 11 tests).
- The task brief's "known pre-existing baseline"
  (`packages/skills/tests/catalog.test.ts`) was checked explicitly: it
  currently **passes** (11/11) — the product repo now has its own
  `axiom.config/skills-catalog.yaml`, so that baseline note no longer
  reproduces (fixed by an earlier, unrelated increment). Not a regression
  risk either way.
- Full-suite `npx vitest run` (all 287 files, 2894 tests) as an extra,
  broader regression check beyond the required targeted scope: **285/287
  files, 2891/2894 tests passed.** The 3 failures were ALL
  `Test timed out in 5000ms` (a timeout, not an assertion failure) in
  `apps/cli/tests/context.test.ts` (1) and
  `apps/cli/tests/workspace-setup.test.ts` (2) — two files this increment
  never touched. Re-running those exact 2 files in isolation:
  **50/50 tests passed, 0 failures** — confirming the timeouts were
  parallel-execution resource contention from the 287-file run (that file
  took 182s in the full run vs 28s alone), not a regression from this
  increment.

## Result

Implemented. All acceptance criteria met. The managed single-repo schema
(schemaVersion 2: `axiomRepo`/`codeRepos`/`legacyRepos`, `kind`/`mode`
discriminators) is live in `@axiom/topology`, fully backward compatible
with schemaVersion 1 (`sddRepo`/`specRepo`/`roleCodeRepositories`, still
validates, now with an explicit, non-blocking deprecation warning), and
surfaced in `axiom topology show`. No adoption/provenance/export/MCP work
was touched (deferred to INC-A2..A5 per Scope). Axiom's own
`axiom.config/topology.yaml` was left structurally on schemaVersion 1 by
design (see Assumptions).

## General spec integration

Integrado en la spec canónica al cierre del batch INC-20260724-* (integración
única de toda la tanda). Ficheros que recibieron el conocimiento de este
incremento:

- `03_Modelo_Operativo_y_Datos.md` — topología `schemaVersion: 2`
  (`axiomRepo`/`codeRepos`/`legacyRepos`, discriminadores `kind`/`mode`,
  auto-map + warning `deprecated-legacy-shape` del shape legacy) como modelo
  objetivo que supersede el `schemaVersion: 1`.
- `05_Interfaces_Operativas.md` — `axiom topology show` con los campos del
  modelo gestionado.
- `08_Glosario.md` — `<project>.axiom`, `axiomRepo`/`codeRepos`/`legacyRepos`,
  `kind`/`mode`.
- `00_Resumen_Ejecutivo.md` — eje "modelo de repo único `<project>.axiom`".
- `01_Requisitos_Funcionales.md` — RF-AXM-040.
- `02_Requisitos_No_Funcionales.md` — NFR-AXM-019 (evolución de schema aditiva).
