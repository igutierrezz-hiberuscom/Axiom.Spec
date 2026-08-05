# Independent review ledger

- Increment: `INC-20260803-r00-adr-ownership-docs`
- Review mode: `axiom-review`, read-only except this ledger.
- Review date: `2026-08-03`.
- Scope: eight ADR, ownership docs, active links, runtime comments/config, `specs/00..08`, `context/**`, and declared freeze/receipt/build evidence.
- Sweep budget: 6 sweeps completed. The final two sweeps covered active workflow guidance, materializable templates, catalog hashes, and post-fix validation.
- Revalidation date: 2026-08-03.
- Mutations: the orchestrator applied the documented remediation outside this ledger; no Git mutation was performed.

## Findings

| id | lens | location | severity | status | evidence |
| --- | --- | --- | --- | --- | --- |
| REVIEW-001 | review | `receipts/2026-08-03T14-40-27.037Z-freeze-success.json`; `receipts/2026-08-03T14-52-11.736Z-apply-success.json`; `Axiom.SDD/.agents/skills/axiom-autopilot/SKILL.md` | CRITICAL | resolved | The historical `13:07` apply remains explicitly non-governed because it predates the historical freeze. A new freeze receipt at `14:40:27.037Z` precedes the idempotent apply receipt at `14:52:11.736Z`; both hashes recompute correctly and the later verify receipt is present. |
| REVIEW-002 | review | `Axiom.Spec/specs/00_Resumen_Ejecutivo.md:112` | SUGGESTION | verified | Previous inventory finding resolved. The current overview names `Axiom.Spec/decisions/0015-...`, `0019-...`, `0026-...` through `0031-...` as eight migrated ADRs and separately names `0032`; no current inventory gap remains. |
| REVIEW-003 | review | `receipts/2026-08-03-final-validation.json`; `receipts/2026-08-03T15-23-17.694Z-verify-success.json` | WARNING | resolved | Dedicated validation evidence now records build, doctor, readiness, and the expected no-match legacy-route sweep after the final corrections. The formal verify receipt was emitted after those checks and its SHA-256 recomputes correctly. |

| REVIEW-004 | review | `03_Criterios_Aceptacion.md`; `README.md`; `metadata.yml`; absent `_archive/` directory | CRITICAL | pending-close | The checklist and README/metadata still require final integration and archive. This is intentionally left open until the orchestrator completes those gates. |
| NEW-001 | review | `Axiom.SDD/.agents/**`; `Axiom.SDD/.github/prompts/**`; `Axiom.SDD/.claude/**`; `Axiom.SDD/.opencode/**` | WARNING | resolved | Active workflow surfaces no longer refer to the removed `general-spec.md`; they now name the canonical numbered `Axiom.Spec/specs/00..08` files. |
| NEW-002 | review | `Axiom/axiom.spec/templates/**`; `Axiom/apps/cli/src/commands/workspace-adapter-templates.ts`; `Axiom/axiom.config/skills-catalog.yaml` | WARNING | resolved | Materializable guidance now uses `.axiom-state/`, `axiom.config/`, and current canonical spec/context paths. The bundled template is synchronized and `TC-010` passes with the recalculated skill hash. |
## Compliance summary

| acceptance criterion | result | observed evidence |
| --- | --- | --- |
| `Axiom.Spec/decisions/` contains all eight named ADR | compliant | All eight destination paths exist. |
| Byte equivalence and old-source absence | compliant | Each destination blob equals the historical `Axiom/docs/<ADR>.md` blob from Git; all eight current `Axiom/docs/<ADR>.md` paths are absent. |
| Active links and references use canonical paths | compliant | No old ADR path was found in the requested active docs, specs, context, runtime source, or YAML. The four checked relative links resolve to existing destination files. Historical/archive references remain scoped as history. |
| Ownership docs describe the real structure and omit obsolete `general-spec.md` | compliant | Root `AGENTS.md`, `Axiom.SDD/AGENTS.md`, and `Axiom.Spec/README.md` describe `decisions/`, `bugs/`, `increments/`, `plans/`, `templates/`, and `technical-context/`; no obsolete standalone `general-spec.md` claim was found. |
| No R-00 runtime behavior change | compliant for reviewed diff | Runtime changes tied to the migration are path/comment updates; no logic change was observed. `get_errors` returned no errors. |
| Build/doctor/readiness state | compliant | `receipts/2026-08-03-final-validation.json` records exit code 0 for all three commands; the expected no-match route sweep and formal verify receipt are also recorded. |
| R-00 integration in `specs/00..08` and `context/**` | compliant for reviewed paths | Active ownership/path statements are coherent; the previous `specs/00` inventory gap is verified resolved by REVIEW-002. |
| Final integration and archive | pending | `03_Criterios_Aceptacion.md` marks the criterion complete, but README/metadata and the missing `_archive` directory show that final integration/archive is not complete; see REVIEW-004. |

## Validation observed

- Historical Git blob comparison: all eight ADR pairs returned `equal=True`.
- Path sweep: eight destinations `True`; eight old direct sources `False`.
- Structural sweep: `decisions/`, `bugs/`, `increments/`, `plans/`, `templates/`, and `technical-context/` all exist.
- Receipt hash recomputation: historical apply evidence and the new `verify` receipt hash recomputed successfully.
- Freeze recomputation via the built CLI module returned the declared `memoryHash`, `specsHash`, and combined hash; the README SHA-256 is `8808af84db41c1772f6e4a81bbf5a6b0de5bc104f721c56258c7ccf1cc7f8932`, and the combined hash is `a81076590c545f9f89b102f81a80e4864b84813561f2469755fa175c15e5b863`.
- Relative-link checks for the two `Axiom/README.md` links and the two `Axiom/docs` links resolve to existing canonical destinations.
- `get_errors` found no errors in `workspace-adapter-templates.ts` or `memory/src/types.ts`; the editor reported pre-existing missing Node type declarations in `start.ts`, while `tsc -b` passed.
- `Axiom/package.json` exposes `npm run build` as `tsc -b`; build, doctor, and readiness were rerun from `Axiom/` after the final fixes.
- The current operational results are persisted in `receipts/2026-08-03-final-validation.json`.
- No mutating Git command was executed.

## Deviations and risks

- The migration and active ownership references are compliant.
- The historical freeze/apply ordering remains a documented deviation; the re-established `14:40` freeze and later apply provide valid closeout evidence.
- The current increment README remains `in-progress`, which is consistent with the unresolved review and the required final integration/archive steps.

## Documentary actions for the orchestrator

1. Complete final `specs/00..08` integration and archive; do not reinterpret the historical `13:07` apply as governed.
2. Keep the new validation and verify receipts with the increment when it is archived.

## Recommendation

`pending-close`: migration, validation, and governed freeze/apply evidence are verified; only final canonical integration, README/metadata closure, and archive remain.

## Suggested commit message

`docs: migrate R-00 ADR ownership to Axiom.Spec`