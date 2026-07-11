# P0 — Core reconciliation: canonical generator + declared side-effects + guided next-step + lifecycle map

> **Código**: INC-20260711-sdd-launcher-p0-core-generator
> **Estado**: Archivado (gate verde; pendiente de mover a `_archive/` por el orquestador)
> **Implementa**: Phase **P0** of the parent epic `INC-20260711-sdd-launcher-core-port` (see that increment's `PLAN.md` §P0 and `RECONCILIATION.md`).
> **Fecha de creación**: 2026-07-11
> **Tipo de cambio**: Nueva capacidad (extend `@axiom/workflow` + `@axiom/core`)

## Goal

Make `@axiom/workflow` (+ `@axiom/core`) the canonical **sdd-core** by adding, additively and green-preserving:

1. One **canonical structure generator** that emits the FULL template-shaped artifact skeleton (not `# title`).
2. **Declared per-transition side-effects** (local-YAML mutation + optional named tracker hook) with a pure runner.
3. A **guided next-step** pure function (`recommendNext`).
4. A static **lifecycle vocabulary mapping** table (sdd-launcher Spanish literals → Axiom `WorkflowState` / integration axis).

This EXTENDS `@axiom/workflow`; it does NOT create a parallel `@axiom/sdd-core` (decision D-001, per `RECONCILIATION.md` Q-001).

## Scope (P0.1–P0.4, P0.5 optional)

- **P0.1 — Canonical structure generator.** Extend `packages/workflow/src/artifact-store.ts` so creating an
  increment/bug/plan emits the full template skeleton: README with all template sections (headings +
  `(pendiente de completar)` prose placeholders + a filled header block), the conditional docs
  (`02_Cambios_Modelo`, `04_Interacciones_UI`) when applicable to the kind, the obligatory sub-docs
  (`01_Requisitos`, `03_Criterios_Aceptacion`), the `context/` folder (readme from
  `context-readme-template.md`), and — for plans — `plan.metadata.yml` (from `plan-metadata-template.yaml`)
  plus one `role-<slug>.md` per role (from `role-plan-template.md`). `metadata.yml` keeps being produced by
  the existing `makeInitial*Metadata` factories (already full). Idempotent / no-clobber per file.
- **P0.2 — Declared per-transition side-effects.** Extend the `WorkflowTransition` shape with an OPTIONAL
  `effects` field: `{ localYaml?, tracker? }` where `localYaml` declares a local-`metadata.yml` mutation and
  `tracker` is an OPTIONAL named hook (string only; no ADO impl — that is P2). Add a pure runner
  `applyTransitionEffects`. Additive & NO-OP when a transition declares nothing.
- **P0.3 — Guided next-step.** Pure `recommendNext(config, state) → transition[]` over the workflow config.
- **P0.4 — Lifecycle vocabulary mapping.** Static, pure, non-breaking table mapping sdd-launcher's
  emoji-prefixed Spanish literals onto Axiom's 9 `WorkflowState`s, with `Integrado` mapped to the separate
  `integration.status: integrated` axis (NOT a `WorkflowState`).
- **P0.5 (optional) — `FileSystem` port** in `@axiom/core` + `NodeFs` impl.

## Non-goals

- No Azure DevOps client / tracker implementation (that is **P2**).
- No front-end, local server, or transport shim (that is **P3**).
- No new CLI subcommands (`scaffold`/`normalize`/`integrate`/`validate`/`state` are **P1**).
- No migration of the existing lockstep side-effects out of the CLI wrappers (P0.2 only establishes the
  declarative mechanism + runner; full migration is a **P1** follow-up).

## Acceptance criteria (copied from PLAN.md §P0)

1. `axiom-increment/bug/plan create` emit the **full** template skeleton (all structural keys present, prose
   placeholders only); a golden-file test asserts the generated tree matches `Axiom.Spec/templates/*`.
2. `recommendNext` and per-transition side-effects are unit-tested against `DEFAULT_WORKFLOWS`; existing
   `@axiom/workflow` tests still pass.
3. No editor dependency introduced (dogfooding boundary check **DF-001** stays green).

## Template-source decision (Q-001, resolved)

Skeletons are built from **bundled TS descriptors** in `@axiom/workflow` (`artifact-skeleton.ts`) — the only
production-safe option because `tsc -b` does not copy non-TS assets into `dist` (the same reason
`apps/cli/src/commands/workspace-spec-base.ts` bundles the spec-base docs as TS string constants, and the same
reason `@axiom/workflow` ships `DEFAULT_WORKFLOWS` as a constant). To honor "prefer the project templates/ dir"
the generator ALSO accepts a project `<specRoot>/templates/` override: when a matching template file exists
there it wins (verbatim, H1 replaced by the artifact title), else the bundled skeleton is emitted — mirroring
the established `workflows.yaml` ↔ `DEFAULT_WORKFLOWS` present-file-wins pattern. This avoids template
triplication: the bundled descriptors are the ONLY code copy of the artifact-template structure
(`workspace-spec-base.ts` covers the disjoint spec-base doc set), and a golden test keeps them faithful to
`Axiom.Spec/templates/*`.

## Result (per sub-item)

- **P0.1 — DONE.** `packages/workflow/src/artifact-skeleton.ts` (bundled descriptors + template resolution +
  builders), `scaffoldArtifact()` added to `artifact-store.ts`, `ensureArtifactReadme` now emits the skeleton
  README. CLI `axiom-increment`/`axiom-bug`/`axiom-plan create` wired to the generator. Golden test:
  `packages/workflow/tests/artifact-skeleton.test.ts`.
- **P0.2 — DONE (mechanism + runner + declared effects on the two `archive` transitions).** `effects` field on
  `WorkflowTransition` (`types.ts`), pure `applyTransitionEffects` runner
  (`packages/workflow/src/transition-effects.ts`), declared on `increment`/`bug` `archive` in both
  `DEFAULT_WORKFLOWS` and `axiom.config/workflows.yaml`. Full migration of CLI-wrapper lockstep side-effects
  into the declared graph is a **P1 follow-up**. Unit test:
  `packages/workflow/tests/transition-effects.test.ts`.
- **P0.3 — DONE.** `recommendNext` in `packages/workflow/src/recommend-next.ts`. Unit test:
  `packages/workflow/tests/recommend-next.test.ts`.
- **P0.4 — DONE.** `packages/workflow/src/lifecycle-vocabulary.ts`. Unit test:
  `packages/workflow/tests/lifecycle-vocabulary.test.ts`.
- **P0.5 — DEFERRED.** Adding a `FileSystem` port and refactoring `artifact-store.ts`'s raw `fs` calls to use
  it balloons scope and risks the green gate for no P0 behavior change; deferred as a follow-up (the port can
  land as its own additive increment).

## Consolidación en la spec general

Documentos canónicos a actualizar al integrar el epic completo (P0–P4): ver `INC-20260711-sdd-launcher-core-port`.
Este P0 no integra por sí solo.

## Estado de validación humana

- Gate: `npm run build && npm test && npm run typecheck && npm run doctor` (ver Result / reporte).
