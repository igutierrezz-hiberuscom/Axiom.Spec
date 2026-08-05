# Increment: R-00 ADR ownership and workspace documentation

Status: closed
Date: 2026-08-03

## Goal

Make `Axiom.Spec/decisions/` the canonical home for the eight architectural decision records currently stored under `Axiom/docs/`, and align the workspace ownership documentation with the repository tree verified during R-00.

## Context

R-00 recorded ACC-001 and ACC-002 after verifying that `Axiom.Spec/decisions/` is absent, the eight ADR files live in `Axiom/docs/`, and the current canonical increment and bug trees live under `Axiom.Spec/specs/`. The runtime topology already points `Axiom` at `../Axiom.Spec` as its spec repository.

## Scope

- Create `Axiom.Spec/decisions/`.
- Move these files byte-for-byte into that directory:
  - `0015-cavekit-discipline-post-mvp.md`
  - `0019-operator-control-plane-runtime.md`
  - `0026-integration-hardening-and-target-parity.md`
  - `0027-toolchain-provider-expansion-and-repair.md`
  - `0028-workflow-ux-and-archive-safety-completion.md`
  - `0029-memory-recall-and-context-repair.md`
  - `0030-operator-app-plugins-and-external-bridge.md`
  - `0031-adr-cmm-replaces-graphify-and-codegraph.md`
- Remove the old copies from `Axiom/docs/` after verifying the destination content.
- Update live links and source/config references that point to the old ADR paths so they resolve to the canonical decision paths.
- Update `AGENTS.md`, `Axiom.SDD/AGENTS.md`, and `Axiom.Spec/README.md` to document the actual `Axiom.Spec` structure, including `decisions/`, `bugs/`, `increments/`, `plans/`, `templates/`, and `technical-context/`.

## Non-goals

- Do not change runtime behavior, package source logic, generated artifacts, or YAML semantics.
- Do not rewrite `Axiom.Spec/specs/00_Resumen_Ejecutivo.md` through `08_Glosario.md` in this increment.
- Do not change `Axiom.Spec/context/**`; the orchestrator will reconcile source references after independent verification.
- Do not modify the R-00 review plan until the final batch integration step.
- Do not run Git mutations.

## Acceptance criteria

- [x] `Axiom.Spec/decisions/` exists and contains all eight named ADR files.
- [x] Each destination ADR is byte-equivalent to its source before deletion, and no source ADR remains in `Axiom/docs/`.
- [x] Live documentation and code/config references no longer point to the old `Axiom/docs/<ADR>.md` locations, except explicitly historical prose that identifies the former location as history.
- [x] The root `AGENTS.md`, `Axiom.SDD/AGENTS.md`, and `Axiom.Spec/README.md` describe the verified repository ownership and folder structure without the obsolete standalone `general-spec.md` claim.
- [x] No product/runtime source behavior changes; the Axiom build remains green.

## Open questions

None. The canonical destination is fixed by the R-00 action and the existing `Axiom.Spec` ownership rules.

## Assumptions

- The eight ADR bodies are authoritative historical decision records and should be preserved unchanged.
- Relative links from runtime documentation may cross the sibling repository when that is the only direct link to the canonical decision.
- Historical references may retain old path text only when clearly marked as historical; current-state references must use `Axiom.Spec/decisions/`.

## Implementation notes

The candidate is intentionally documentary and cross-repository. The delegated worker must treat this README as frozen and must not rewrite it after the freeze; validation evidence will be recorded by the orchestrator after independent verification.

## Validation

La implementación documental y la verificación independiente de la migración
completaron los siguientes puntos:

- The eight ADRs were copied byte-for-byte into `Axiom.Spec/decisions/` and the eight old `Axiom/docs/` paths are absent.
- Active searches found no old ADR paths outside explicitly excluded historical/context areas.
- The documented `Axiom.Spec` structural directories exist.
- `npm run build` from `Axiom/` passed with `tsc -b` (exit code 0).
- `npm run doctor` passed (46/61 OK, 0 failures, 3 warnings, 12 skipped; exit code 0).
- `npm run readiness:first-project` passed (exit code 0).
- La salida determinante y los códigos de salida están persistidos en
  `receipts/2026-08-03-validation.json`.
- La revalidación posterior a los últimos fixes está persistida en
  `receipts/2026-08-03-final-validation.json` y el receipt formal `verify`
  tiene hash íntegro.
- The historical apply receipt `receipts/2026-08-03T13-07-39.874Z-apply-success.json`
  is integrity-checked evidence of an earlier apply only. Its timestamp
  precedes the historical candidate freeze, so it is not treated as a valid
  pre-apply gate for this closeout.

## Governance evidence

The candidate was re-frozen at `14:40:27.037Z` and the idempotent governed apply
completed at `14:52:11.736Z`; both receipts are integrity-checked. The earlier
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
canonical ownership of `Axiom.Spec/decisions/`, the absence of the eight old
active ADR sources, and the numbered `Axiom.Spec/specs/00..08` integration
route; no product behavior is expected to change.
