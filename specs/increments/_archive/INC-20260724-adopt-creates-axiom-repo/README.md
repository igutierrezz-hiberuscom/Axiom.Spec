# Increment: Adopción crea un `<project>.axiom` nuevo (no in-place)

Status: closed
Date: 2026-07-24

## Goal

Change `axiom workspace setup`'s adoption mode (`--adopt-spec`/`--adopt-sdd`/
`--ingest-context`) so that, instead of migrating **in-place into the legacy
spec/SDD repo**, it **creates a NEW managed `<project>.axiom` repo** (sibling,
`../${projectName}.axiom`) and migrates the legacy spec/context content INTO
it (Axiom format), leaving the legacy repos **completely untouched** (never
written to). Record the legacy sources in the project's topology as
`legacyRepos[]` (`mode: read-only-source`, INC-A1 schema, schemaVersion 2).

This is **INC-A2** of Cluster A in the Axiom evolution plan
(`giggly-imagining-moore.md`, "Arquitectura objetivo"), building directly on
**INC-A1** (`INC-20260724-topology-single-axiom-repo`, closed), which added the
`axiomRepo`/`codeRepos`/`legacyRepos` schema (schemaVersion 2) this increment
now actually POPULATES via a real workflow. Explicit, user-approved graduation
to the full product lifecycle (`Axiom.SDD/AGENTS.md`'s "Explicit Bootstrap
Limits" exception, per the plan).

## Context

Today, adoption (`runWorkspaceAdopt` in `apps/cli/src/commands/
workspace-adopt.ts`, a pure orchestrator over `runWorkspaceSetup` +
`runBootstrapFromLegacySdd`/`runBootstrapFromContext`) treats `--adopt-spec
<path>`/`--adopt-sdd <path>` as the ACTUAL destination: `specPath`/`controlPath`
in the constructed `WorkspaceSetupSpec` are set directly to the foreign path
(`create:false`), and the migration destination (`resolveSpecScopeAbsolutePath`
in `bootstrap.ts`/`_spec-scope.ts`) resolves to that SAME foreign repo —
adopt-in-place. Existing FP-1 tests (`apps/cli/tests/e2e/
adoption-breadth.e2e.test.ts:200`, `Axiom.Pruebas/verificar-adopcion.ps1:139`)
explicitly assert this in-place behavior (no `axiom.spec` nesting, artifacts
land directly at `<specRepo>/increments`).

The target architecture ("Arquitectura objetivo" in the plan) says legacy
repos must stay intact — Axiom never writes into what the project already
had — and a single managed `<project>.axiom` repo is the "whole brain" (spec/
increments/bugs/adr/technical-context + skills/adapters/commands). INC-A1
built the SCHEMA for this (`axiomRepo`/`codeRepos`/`legacyRepos`,
`kind`/`mode` discriminators) but explicitly deferred the actual adoption-flow
change ("This increment introduces the SCHEMA for that model; it does not
implement the adoption flow that actually creates a `<project>.axiom` repo on
disk (that is INC-A2)").

## Scope

- `apps/cli/src/commands/workspace-adopt.ts`: `WorkspaceAdoptArgs` gains
  `adoptSddSourcePath?: string` (replaces the old `adoptSdd: boolean` — the
  legacy SDD/control content source, now READ-ONLY, never additively
  parametrized in place); a new exported `defaultAxiomRepoSiblingPath(anchorDir,
  projectName)` helper (mirrors `@axiom/topology`'s `../${projectName}.spec`
  sibling convention → `../${projectName}.axiom`); a new
  `recordLegacyReposInTopology` step that upserts `legacyRepos[]` (+
  `axiomRepo`, schemaVersion 2) into the destination's `topology.yaml` after
  `runWorkspaceSetup` runs; conformance-report wording updated (no longer
  implies the legacy repo itself was "adopted in place").
- `apps/cli/src/commands/workspace.ts`: `handleWorkspaceSetup` computes the
  `<project>.axiom` destination (sibling of cwd, or `--control-path` as an
  explicit override) when adopting, feeds it as BOTH the `control` and `spec`
  `WorkspaceRepoSpec` entries (co-located, "one-axiom-per-product" mode,
  already supported by `runWorkspaceSetup`), and threads
  `adoptSpecSourcePath`/`adoptSddSourcePath` as separate, read-only source
  paths (no longer conflated with the destination). CLI help text updated.
- `apps/cli/src/commands/workspace-setup.ts`: ONE targeted correctness fix —
  `created` (whether a repo's directory existed before this run) is now
  pre-computed ONCE per unique resolved path (not per repo entry), so that TWO
  `WorkspaceRepoSpec` entries sharing the same destination path (control +
  spec collapsed into one `<project>.axiom`) both correctly report
  `created: true` on a fresh run. Without this, the SECOND entry (spec) would
  see the directory as already-existing (created moments earlier by the
  first/control entry's own `mkdir`) and wrongly skip `scaffoldSpecRepoBase`.
  `writeOneRepo` gains an optional `dirExistedOverride` parameter (backward
  compatible — `workspace-incremental.ts`'s calls are unaffected).
- Tests rewritten to the new destination model: `apps/cli/tests/
  workspace-adopt.test.ts`, `apps/cli/tests/e2e/workspace-adopt.e2e.test.ts`,
  `apps/cli/tests/e2e/adopt-upgrade.e2e.test.ts`; light-touch rewrite of
  `apps/cli/tests/e2e/adoption-breadth.e2e.test.ts` (explicit legacy-unchanged
  assertion + terminology). New e2e file: `apps/cli/tests/e2e/
  adopt-creates-axiom-repo.e2e.test.ts` (writer/reader convergence proof).
- `Axiom.Pruebas/verificar-adopcion.ps1`: detection + assertions updated to
  the new destination model (read-only text edit, not executed).

## Non-goals

- Richer provenance/lifecycle states (`migrated`/`migrated-and-modified`/
  `axiom-native`) and the migration manifest — INC-A3. This increment keeps
  today's `AXIOM:MIGRATED` banner + `.migration-provenance.yml` mechanism
  working, just pointed at the new destination.
- Export/rollback ("leaving Axiom") — INC-A4.
- The unified `<project>.axiom` MCP broker — INC-A5. `buildWorkspaceMcpServers`
  already degrades gracefully for a co-located control+spec repo by emitting
  BOTH `sdd-mcp-server`/`spec-mcp-broker` aliases pointed at the same
  directory (pre-existing code, unchanged) — that dual-alias transitional
  state is accepted as-is here.
- Migrating Axiom's OWN `Axiom.SDD`/`Axiom.Spec` repos to the managed model
  (explicitly deferred to the user).
- A full de-duplication of `runWorkspaceSetup`'s per-repo-kind writes when
  control/spec paths coincide (topology.yaml/AGENTS.md/process-surfaces get
  written twice, redundantly-but-harmlessly, once per kind) — only the ONE
  genuine correctness bug (`created` flag divergence blocking
  `scaffoldSpecRepoBase`) is fixed; other redundant re-writes are idempotent
  no-ops, not addressed (would widen the blast radius on a heavily-used
  engine for no functional benefit).
- Recording `--ingest-context`'s source path as a `legacyRepos[]` entry — the
  task/plan explicitly names "legacy spec/sdd repos"; a context source is
  typically a doc subfolder, not a distinct repo (see Assumptions).

## Acceptance criteria

- [x] `axiom workspace setup --adopt-spec <path>` (and/or `--adopt-sdd
      <path>`) creates a NEW `../${projectName}.axiom` repo (or an
      explicit `--control-path` override) instead of writing into the
      legacy path.
- [x] The legacy repo(s) named by `--adopt-spec`/`--adopt-sdd` are BYTE-FOR-BYTE
      unchanged after adoption (no `axiom.yaml`, no migrated artifacts, no
      writes of any kind).
- [x] Migrated spec/context content (increments/bugs/adr/technical-context)
      lands under the NEW `<project>.axiom` repo, in Axiom format, with the
      existing `AXIOM:MIGRATED` banner + `.migration-provenance.yml`
      idempotency mechanism intact.
- [x] The new repo's `topology.yaml` (schemaVersion 2) declares `axiomRepo`
      (self) and `legacyRepos[]` (one entry per adopted legacy source,
      `kind: 'legacy'`, `mode: 'read-only-source'`).
- [x] Writer/reader convergence: content written by the migration
      (`runBootstrapFromLegacySdd`/`runBootstrapFromContext`, driven from
      `resolveSpecScopeAbsolutePath`) is visible to CLI readers
      (`axiom-increment`/`axiom-bug` `list`, driven from
      `resolveSpecArtifactRelPath`/`resolveSpecRelPathForScope`) — both
      resolve to the SAME `<project>.axiom` root, with NO `axiom.spec`
      subfolder nesting.
- [x] `--dry-run` performs zero writes (including zero topology/legacyRepos
      writes); confirm gate (`-y`) unchanged; re-run is idempotent (no
      duplicate artifacts, no duplicate `legacyRepos[]` entries — upsert by
      conventional id).
- [x] Best-effort scaffolding: a missing/invalid legacy source path never
      throws; it's simply skipped (not recorded in `legacyRepos[]`), same
      posture as the rest of the adoption flow.
- [x] `npm run build` (tsc -b) passes.
- [x] Targeted `vitest run` (adoption/bootstrap tests in `apps/cli/tests` +
      `packages/workflow` + `packages/topology`) passes; every failure
      classified pre-existing vs new.
- [x] Rewritten tests assert the NEW semantics (see Scope); the FP-1
      assertion in `adoption-breadth.e2e.test.ts` is reformulated (that test
      operates at the lower-level migration-primitive layer, not through
      `runWorkspaceAdopt` — see Assumptions).
- [x] `@axiom/cli-commands` single-ownership preserved; `@axiom/tui` untouched
      (adoption has no TUI surface today — out of scope to add one).

## Open questions

None blocking — the brief's "FINAL decisions" section explicitly bakes in
destination/legacy-untouched/topology-population; the few remaining
implementation-level choices are resolved narrowly under Assumptions below
(existing-precedent-first, per the executor's guardrails).

## Assumptions

Each is a deliberate, narrowest-change resolution of an ambiguity the task
left open — see the executor's final report for one-line reasoning:

1. The merged `<project>.axiom` repo is fed into `runWorkspaceSetup` as TWO
   `WorkspaceRepoSpec` entries (`kind: 'control'` and `kind: 'spec'`, same
   `path`) rather than changing `runWorkspaceSetup`'s hard "exactly one
   control, one spec" contract — this is the ALREADY-EXISTING
   "one-axiom-per-product" mode (`buildRoleAwareAxiomYaml`'s `mode` string
   literal predates this increment), so it introduces NO new write primitive.
2. Because `spec.repos` is processed in order (`[control, spec, ...roles]`)
   and both entries share a path, the SECOND (`spec`) write wins for
   `axiom.yaml#role` — the merged repo deterministically ends up `role:
   'spec'`. Verified as the CORRECT (not just tolerable) outcome: it is what
   `resolveSpecRelPathForScope`/`packages/mcp-server/src/context.ts` require
   for the dedicated (`'.'`, no `axiom.spec` nesting) rel-path, what
   `_repo-affinity.ts`'s `spec-repo` expectation allows immediately, and what
   `upgrade.ts`'s cross-repo fan-out gate (`role === 'sdd'`-only) correctly
   AVOIDS triggering for a self-contained merged repo.
3. `--adopt-sdd <path>`'s meaning changes from "the control repo IS this
   foreign path, additively parametrize it in place" (writing into it) to "a
   legacy SDD source recorded read-only in `legacyRepos[]`" — consistent with
   `--adopt-spec`'s equivalent change and the plan's explicit "legacy spec/sdd
   repos" wording.
4. `--control-path` is repurposed, ONLY in adopting mode, as an explicit
   override for the `<project>.axiom` destination directory (instead of being
   ignored or erroring) — a low-cost escape hatch reusing an existing flag
   rather than adding a new one. `--spec-path` is ignored during adoption
   (destination is always the same merged repo as control) — undocumented
   behavior change accepted since no existing test exercises the CLI-flag
   layer for adoption (`workspace.ts` has zero direct test coverage for its
   adoption branch; all existing tests call `runWorkspaceAdopt` directly).
5. `legacyRepos[]` entries use fixed conventional ids (`legacy-spec-source`,
   `legacy-sdd-source`) and are upserted by id on re-run (never duplicated).
   If `--adopt-spec`/`--adopt-sdd` happen to resolve to the SAME physical
   directory, two distinct entries are still recorded (different semantic
   origin) — cheap and not flagged as a duplicate by `validateTopology`
   (which checks by `id`, not `ref`).
6. `adoption-breadth.e2e.test.ts`'s T4 scenario already operates at the
   `runBootstrapFromLegacySdd`/`runBootstrapFromContext` primitive layer
   (never through `runWorkspaceAdopt`), against a manually-built "dedicated
   spec repo" fixture that already IS what the new model calls
   `<project>.axiom` (role: spec, artifacts at its root). Its existing
   `axiom.spec`-absence assertion is preserved; the rewrite adds an EXPLICIT
   byte-for-byte "foreign source unchanged" assertion and renames identifiers
   for clarity, rather than restructuring the whole fixture.
7. `--ingest-context <path>`'s source is NOT recorded in `legacyRepos[]`
   (unlike `--adopt-spec`/`--adopt-sdd`) — it is typically a doc subfolder,
   not a distinct repo, and the plan/task explicitly name only "spec/sdd"
   repos for this bucket. It remains read-only exactly as before (unaffected
   either way).

## Implementation notes

Files changed (all in `Axiom/`, the product monorepo, except this spec):

- `apps/cli/src/commands/workspace-adopt.ts` — `WorkspaceAdoptArgs.adoptSdd:
  boolean` → `adoptSddSourcePath?: string`; new `defaultAxiomRepoSiblingPath`;
  new `buildLegacySourceRef`/`recordLegacyReposInTopology` (upsert-by-id,
  schemaVersion 2, best-effort); conformance report header wording updated;
  new report lines naming the `<project>.axiom` destination + the read-only
  legacy sources.
- `apps/cli/src/commands/workspace.ts` — `handleWorkspaceSetup`: computes the
  axiom-repo destination via `defaultAxiomRepoSiblingPath`
  (or `--control-path` override) when adopting; passes it as both
  control/spec paths; threads `adoptSpecSourcePath`/`adoptSddSourcePath`
  (resolved from `--adopt-spec`/`--adopt-sdd`) separately; `--spec-path`
  requirement scoped to the non-adopting branch only; CLI option help text
  updated.
- `apps/cli/src/commands/workspace-setup.ts` — `writeOneRepo` gains optional
  trailing `dirExistedOverride?: boolean`; `runWorkspaceSetup`'s step-1 loop
  pre-computes a `Map<resolvedPath, boolean>` of pre-run existence ONCE and
  passes the per-path value through, fixing the co-located-repo `created`
  divergence (see Scope).
- `apps/cli/tests/workspace-adopt.test.ts` — rewritten to the new contract
  (destination ≠ legacy source; `adoptSddSourcePath` instead of `adoptSdd`).
- `apps/cli/tests/e2e/workspace-adopt.e2e.test.ts` — rewritten.
- `apps/cli/tests/e2e/adopt-upgrade.e2e.test.ts` — rewritten (merged
  destination path; `adoptSddSourcePath` field rename).
- `apps/cli/tests/e2e/adoption-breadth.e2e.test.ts` — light-touch: explicit
  legacy-source-unchanged assertion added; identifiers/comments updated.
- `apps/cli/tests/e2e/adopt-creates-axiom-repo.e2e.test.ts` (new) — the
  increment's centerpiece proof: adopt a sandbox legacy project → legacy
  repos byte-unchanged → `<project>.axiom` created → migrated artifacts
  readable via the `axiom-increment`/`axiom-bug` list reader
  (writer/reader convergence).
- `Axiom.Pruebas/verificar-adopcion.ps1` — text-only edits (destination
  detection + assertions); not executed.

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0.
- Targeted `npx vitest run` (adoption/bootstrap surface): 8 files, 87 tests
  passed (`workspace-adopt.test.ts`, `e2e/workspace-adopt.e2e.test.ts`,
  `e2e/adopt-upgrade.e2e.test.ts`, `e2e/adoption-breadth.e2e.test.ts`,
  `e2e/adopt-creates-axiom-repo.e2e.test.ts` (new), `workspace-setup.test.ts`,
  `workspace-command.test.ts`, `workspace-incremental.test.ts`).
- Targeted `npx vitest run` (bootstrap/migration primitives +
  `packages/workflow` + `packages/topology`, per the task's required scope):
  37 files, 418 tests passed.
- Targeted `npx vitest run` (all `apps/cli/tests/e2e/*` + topology/doctor/tui
  + repo-affinity + upgrade + remaining `workspace-*` unit suites): 23 files,
  196 tests passed.
- Full-suite `npx vitest run` (all 288 files, 2896 tests) as an extra,
  broader regression check (this touched `workspace-setup.ts`, a widely-used
  core engine): **288/288 files, 2896/2896 tests passed, zero failures** —
  no pre-existing or new failures this run (no timeout flakiness reproduced
  either, though `workspace-setup.test.ts`/`context.test.ts` are documented
  as occasionally flaky under full-suite parallel load per prior increments).
- PowerShell script (`Axiom.Pruebas/verificar-adopcion.ps1`): NOT executed
  (read-only sandbox verification tool, requires a real adopted project) —
  parse-only syntax check via
  `[System.Management.Automation.Language.Parser]::ParseFile` confirmed zero
  syntax errors.

## Result

Implemented. All acceptance criteria met. Adoption now creates a NEW
`<project>.axiom` repo (sibling of cwd, `../${projectName}.axiom`) instead of
migrating in-place; the legacy spec/SDD sources named by
`--adopt-spec`/`--adopt-sdd` are read-only inputs, never written to (proven
byte-for-byte in tests), and are recorded as `legacyRepos[]`
(schemaVersion 2) in the new repo's `topology.yaml`. Migrated content
(increments/bugs/adr/technical-context) lands directly under the new repo
(no `axiom.spec` nesting) and is readable through the same CLI reader
primitives (`axiom-increment`/`axiom-bug list`) the migration writer
converges with. A genuine pre-existing-shaped bug was found and fixed along
the way: `runWorkspaceSetup`'s per-repo `created` flag diverged when two
`WorkspaceRepoSpec` entries (control + spec) share the same destination path
(the new co-located model), silently skipping `scaffoldSpecRepoBase` for a
brand-new repo — fixed by pre-computing directory pre-existence once per
unique resolved path. Deferred per Scope: richer provenance lifecycle
(INC-A3), export/rollback (INC-A4), unified MCP broker (INC-A5).

## General spec integration

Integrado en la spec canónica al cierre del batch INC-20260724-* (integración
única de toda la tanda). Ficheros que recibieron el conocimiento de este
incremento:

- `03_Modelo_Operativo_y_Datos.md` — adopción crea un `<project>.axiom` nuevo,
  legacy intacto, `legacyRepos[]` poblados (read-only-source), contenido
  migrado en la raíz del repo (sin anidamiento `axiom.spec/`).
- `04_Flujos_SDD_y_Ciclo_de_Vida.md` — flujo de adopción (crea repo axiom,
  legacy byte-unchanged, writer/reader convergen en la raíz).
- `07_Gobierno_y_Seguridad.md` — legacy intacto por construcción.
- `08_Glosario.md` — término "Adopción (nuevo sentido)".
- `00_Resumen_Ejecutivo.md` — eje "modelo de repo único `<project>.axiom`".
- `01_Requisitos_Funcionales.md` — RF-AXM-040.
- `02_Requisitos_No_Funcionales.md` — NFR-AXM-017 (legacy intacto).
