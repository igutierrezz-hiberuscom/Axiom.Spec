# Increment: R-00 axiom.spec boundary decision

Status: closed
Date: 2026-08-03

## Goal

Resolve ACC-003 by verifying the purpose of `Axiom/axiom.spec/` and recording whether it is useful product-owned installation infrastructure or an accidental duplicate of the sibling `Axiom.Spec/` repository.

## Context

R-00 identified a naming ambiguity between the uppercase workspace repository `Axiom.Spec/` and the lowercase in-repo directory `Axiom/axiom.spec/`. The current tree contains `increments/`, `plans/`, `target-axiom-agents/`, `target-axiom-skills/`, and `templates/` under `Axiom/axiom.spec/`, while active runtime configuration and topology refer to these surfaces during product self-adoption and materialization.

## Scope

- Inspect the complete current tree under `Axiom/axiom.spec/`.
- Trace its consumers in the runtime catalogs, scaffold/materializer code, and product documentation.
- Record a decision in `Axiom.Spec/decisions/` stating whether the directory is legitimate product-owned baseline content, what it is not, and whether it should be moved, deleted, or renamed.
- Make only the smallest documentation clarification required to prevent readers from treating `Axiom/axiom.spec/` as the canonical workspace spec repository.

## Non-goals

- Do not move, delete, or rename `Axiom/axiom.spec/` or its contents unless the verified consumer trace proves that the current structure is unused and safe to remove.
- Do not change runtime materialization, catalog paths, generated bundles, or package behavior.
- Do not rewrite `Axiom.Spec/specs/00_Resumen_Ejecutivo.md` through `08_Glosario.md` in this increment.
- Do not modify `Axiom.Spec/context/**`; the orchestrator will reconcile any stable clarification after independent verification.
- Do not modify the R-00 review plan until final batch integration.
- Do not run Git mutations.

## Acceptance criteria

- [x] The inventory and consumer trace are recorded with concrete paths and distinguish verified facts from historical claims.
- [x] The decision explicitly concludes whether `Axiom/axiom.spec/` remains, moves, is deleted, or is renamed, with rationale.
- [x] The decision distinguishes `Axiom.Spec/` (canonical workspace specification repository) from `axiom.spec/` (product-owned in-project content) in wording that can be reused by future documentation.
- [x] No runtime behavior or generated baseline changes are introduced.
- [x] The result is independently checked against the live tree and references.

## Open questions

None blocking. If the consumer trace reveals a stale path, record it as a follow-up rather than making an unrelated runtime migration in this documentary increment.

## Assumptions

- Existing active context already treats the two names as distinct; this increment confirms or corrects that statement against the current runtime.
- The conservative outcome is to retain a product-owned directory when active catalogs or materializers consume it.

## Implementation notes

This candidate depends on the ADR ownership increment because its decision belongs in the newly canonical `Axiom.Spec/decisions/` directory. The delegated worker must treat this README as frozen and must not rewrite it after the freeze.

## Validation

La implementación documental y la verificación independiente de la frontera
completaron los siguientes puntos:

- The five `Axiom/axiom.spec/` families were inventoried: `increments/` (2 files), `plans/` (0), `target-axiom-agents/` (14), `target-axiom-skills/` (20), and `templates/` (45).
- Consumers and baseline/scaffolding paths were checked in the workflow artifact store, runtime catalogs, adapters, CLI commands, readiness script, and topology/product contracts.
- `Axiom/axiom.config/topology.yaml` resolves `specRepo` to `../Axiom.Spec`, and `Axiom/axiom.yaml` identifies `Axiom.Spec` as the functional source of truth.
- ADR-0032 records the decision to retain `Axiom/axiom.spec/` without moving, deleting, or renaming it.
- `Axiom/docs/overview.md` now states the boundary explicitly and its Markdown syntax was independently checked.
- `npm run build` from `Axiom/` passed with `tsc -b` (exit code 0).
- `npm run doctor` passed (46/61 OK, 0 failures, 3 warnings, 12 skipped; exit code 0).
- `npm run readiness:first-project` passed (exit code 0).
- La salida determinante y los códigos de salida están persistidos en
	`receipts/2026-08-03-validation.json`.
- La revalidación posterior a los últimos fixes está persistida en
	`receipts/2026-08-03-final-validation.json` y el receipt formal `verify`
	tiene hash íntegro.
- The historical apply receipt `receipts/2026-08-03T13-14-48.311Z-apply-success.json`
	is integrity-checked evidence of an earlier apply only. Its timestamp
	precedes the historical candidate freeze, so it is not treated as a valid
	pre-apply gate for this closeout.

## Governance evidence

The candidate was re-frozen at `14:40:39.726Z` and the idempotent governed apply
completed at `14:58:08.833Z`; both receipts are integrity-checked. The earlier
apply/freeze ordering remains a documented historical deviation, not compliant
evidence.

## Result

Implementation, current validation, governed freeze/apply, independent review,
final R-00 spec/context integration, and authorized physical archive are
complete. The increment is closed and archived.

## General spec integration

Integrated once in `Axiom.Spec/specs/04_Flujos_SDD_y_Ciclo_de_Vida.md`,
`Axiom.Spec/context/TECHNICAL_CONTEXT.md`, and
`Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md`. Stable knowledge is the
distinction between canonical `Axiom.Spec/` and product-owned materializable
`Axiom/axiom.spec/`, plus the current `.axiom-state/` and `axiom.config/`
paths; no product behavior is expected to change.
