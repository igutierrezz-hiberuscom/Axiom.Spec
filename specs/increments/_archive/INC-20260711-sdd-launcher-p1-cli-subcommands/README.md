# P1 — CLI generator subcommands (scaffold / normalize / integrate / validate / state)

> **Código**: INC-20260711-sdd-launcher-p1-cli-subcommands
> **Estado**: archived (gate verde; pendiente de mover a `_archive/` por el orquestador)
> **Implementa**: Phase **P1** of the parent epic `INC-20260711-sdd-launcher-core-port` (see that increment's `PLAN.md` §P1).
> **Depende de**: P0 (`INC-20260711-sdd-launcher-p0-core-generator`, archivado): `scaffoldArtifact`, `applyTransitionEffects`, `recommendNext`, `lifecycle-vocabulary`.
> **Fecha de creación**: 2026-07-11
> **Tipo de cambio**: Nueva capacidad (CLI wrappers delgados sobre el core P0)

## Goal

Expose the P0 generator and state model as first-class, script-invocable CLI subcommands so the
AI **invokes** structure and never invents it. Five subcommands, all thin wrappers over the
already-tested `@axiom/workflow` core:

1. `axiom scaffold <increment|bug|plan|e2e>` — canonical structure generator.
2. `axiom normalize` — pure, idempotent metadata normalizer.
3. `axiom integrate` — deterministic archive + terminal transition.
4. `axiom validate transition` — validate a requested transition against the declared graph.
5. `axiom state` — inspector over `workflow-state.json` + per-artifact `metadata.yml`.

## Scope (P1)

- **scaffold** — `axiom scaffold increment|bug|plan` delegate to the existing create run functions
  (`runIncrementSubcommand` / `runBugSubcommand` / `runPlanCreate`), which already call the P0
  `scaffoldArtifact()`. No generator logic is duplicated; `create` and `scaffold` share one path.
  `--dry-run` previews the file tree (via `buildSkeletonFiles`) without writing.
- **normalize** — pure `normalizeArtifactMetadata()` in `@axiom/workflow` canonicalizes `status` to
  one of Axiom's 9 `WorkflowState`s, using the P0.4 `lifecycle-vocabulary` table for emoji-ES interop
  (`✅ Implementado` → `archived`, `📦 Integrado` → `archived` + `integration.status: integrated`, …).
  Idempotent: already-canonical metadata is left byte-identical, so a second run is a no-op.
  `--dry-run` reports intended changes without writing.
- **integrate** — applies the terminal archive transition (via `applyTransition` on the declared
  graph) + sets `integration.status: integrated` + physically archives the folder via the existing
  `archiveArtifactDir`. Refuses (typed error) if closure preconditions are unmet (artifact absent,
  no legal terminal transition from the current state, or already archived). `--dry-run` previews.
- **validate transition** — EXTENDS `validate-changes.ts`: given `--workflow` + `--command`, validates
  the transition against the declared workflow graph using `applyTransition`. Rejects an illegal
  transition with the typed `invalid-transition` error and lists the legal commands. The existing
  `validate changes` (write-scope) is untouched.
- **state** — inspector over `loadWorkflowState` + per-artifact `metadata.yml`; prints current state,
  available transitions, and P0 `recommendNext` recommendations.

## Non-goals

- **No Azure DevOps / tracker** (that is **P2**). The `tracker` hook stays a named string only.
- **No front-end / local server / transport shim** (that is **P3**).
- **No MCP action tools** (P2/P3 cross-cutting).
- **No prompt engine / routing table** (that is **P4**).
- No `e2e` artifact kind is invented: `axiom scaffold e2e` is DEFERRED (see Q-001) because qa-e2e is a
  runtime workflow lane, not a folder-per-instance artifact.

## Acceptance criteria (from PLAN.md §P1 + corner cases)

1. Each subcommand has a `--dry-run`/preview that prints the intended change WITHOUT writing
   (consistent with `axiom app`'s preview contract).
2. `axiom validate transition` rejects an illegal transition with the typed `invalid-transition`
   error and lists the legal commands.
3. Round-trip: `scaffold → normalize → integrate` on a fixture increment yields a schema-valid
   ARCHIVED artifact; `@axiom/doctor` reports no new failures.
4. `axiom state` on a fresh vs. mid-lifecycle vs. terminal artifact prints correct
   current/available/recommended.
5. Corner cases: normalize is idempotent; integrate refuses if closure preconditions unmet; validate
   on an already-terminal state lists no legal commands; scaffold no-clobber on re-run.

## Result (per sub-item)

- **scaffold — DONE.** `apps/cli/src/commands/scaffold.ts` (`runScaffold` + `registerScaffold`).
  `increment`/`bug`/`plan` delegate to `runIncrementSubcommand`/`runBugSubcommand`/`runPlanCreate`
  (which already call P0 `scaffoldArtifact`) — no generator logic duplicated. `--dry-run` previews the
  tree via `buildSkeletonFiles`. `e2e` — **DEFERRED** (Q-001): prints a pointer to `axiom-qa-e2e start`
  (qa-e2e is a runtime lane, not an artifact kind). Tests: `apps/cli/tests/scaffold.test.ts` (7).
- **normalize — DONE.** Pure core `normalizeArtifactMetadata` in `packages/workflow/src/normalize.ts`
  (canonicalizes `status` to a `WorkflowState` via the P0.4 lifecycle-vocabulary table; axis-splits
  `Integrado`); CLI `apps/cli/src/commands/normalize-cmd.ts` (`runNormalize` + `registerNormalize`).
  Idempotent (writes only on change). `--dry-run` reports without writing. Tests:
  `packages/workflow/tests/normalize.test.ts` (10), `apps/cli/tests/normalize-cmd.test.ts` (4).
- **integrate — DONE.** `apps/cli/src/commands/integrate.ts` (`runIntegrate` + `registerIntegrate`).
  Validates the `→ archived` transition against the declared graph (`applyTransition`), sets
  `integration.status: integrated`, and REUSES `archiveArtifactDir` + `checkQaArchiveGate`. Refuses
  (typed) when the precondition is unmet (not-found / no legal terminal transition). `--dry-run`
  previews. Tests: `apps/cli/tests/integrate.test.ts` (4).
- **validate transition — DONE.** EXTENDED `apps/cli/src/commands/validate-changes.ts`
  (`runValidateTransition` + a new `validate transition` subcommand; `validate changes` untouched).
  Rejects an illegal transition with the typed `invalid-transition` error + legal-command list.
  Tests: `apps/cli/tests/validate-transition.test.ts` (4).
- **state — DONE.** `apps/cli/src/commands/state-cmd.ts` (`runState` + `registerState`). Workflow mode
  (`workflow-state.json`) and artifact mode (`metadata.yml`); prints current / available / recommended
  (`recommendNext`) + integration axis. Tests: `apps/cli/tests/state-cmd.test.ts` (5).

Supporting core changes (`@axiom/workflow`): additive optional `integration` axis on the artifact-store
metadata (`ArtifactIntegration`, parsed/serialized only when set — back-compat); pure
`pickWorkflowConfig(config, id)` converter in `workflows-loader.ts` (de-duplicates the inline
`WorkflowsConfig → WorkflowConfig` mapping that `validate`/`state`/`integrate` now share).

**Gate (in `Axiom/`)**: `npm run build` PASS · `npm test` **2409 passed** (2375 baseline + 34 new, 0
failures) · `npm run typecheck` PASS · `npm run doctor` PASS (0 failures; DF-001 dogfooding boundary +
IX-001 metadata-parse green). Real CLI e2e round-trip `scaffold → (advance) → normalize → integrate`
executed in a scratch project: produced a schema-valid ARCHIVED artifact under `_archive/` carrying
`status: archived` + `integration.status: integrated`.

## Consolidación en la spec general

Documentos canónicos a actualizar al integrar el epic completo (P0–P4): ver
`INC-20260711-sdd-launcher-core-port`. Este P1 no integra por sí solo.

## Estado de validación humana

- Gate: `npm run build && npm test && npm run typecheck && npm run doctor` en `Axiom/` (ver Result).
