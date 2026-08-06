# Review ledger — INC-20260806-unify-project-state-paths

Date: 2026-08-06
Flow: increment
Route: sdd
Scope: canonical state paths, legacy migration, checkpoint restore, toolchain, provider selection, documentation and archive closure.

## Alcance

Revisión independiente de la convención `.axiom-state/<projectKey>/`,
precedencia/migración legacy, persistence/isolation, workflow/memory/MCP,
checkpoints, toolchain, providers de worktree, doctor, documentación y cierre
formal.

## Hallazgos

| id | lens | location | severity | status | evidence |
|---|---|---|---|---|---|
| REVIEW-002 | runtime | `Axiom/packages/toolchain/src/detect.ts`; `Axiom/packages/toolchain/src/probe.ts`; `Axiom/packages/toolchain/src/repair.ts`; `Axiom/packages/toolchain/tests/repair-add-gitignore.test.ts` | CRITICAL | resolved | Detect/probe/repair aceptan aliases; repair migra al canonical y elimina todos los aliases restantes, incluso con canonical preexistente. |
| REVIEW-005 | runtime | `packages/versioning/src/checkpoints.ts`; `packages/versioning/tests/checkpoints.test.ts` | CRITICAL | resolved | Restore busca raíces direct/config legacy y remapea destinos de manifest a `.axiom-state/<projectKey>/` antes de limpiar el origen. |
| REVIEW-006 | isolation | `apps/cli/src/commands/workspace-worktree-provision.ts`; `apps/cli/tests/workspace-worktree-provision.test.ts` | CRITICAL | resolved | Providers se leen con `Execution.projectId` y aliases; regresión v2 cubre `projectId != name`. |
| REVIEW-008 | governance | `Axiom.Spec/specs/increments/_archive/INC-20260806-unify-project-state-paths/candidate-freeze.json`; `Axiom.Spec/specs/increments/_archive/INC-20260806-unify-project-state-paths/receipts/2026-08-06T23-18-32.264Z-freeze-success.json` | BLOCKER | resolved | Freeze final archivado verificado con `checkCandidateFreeze`; hash y receipt recomputables. |
| REVIEW-009 | governance | `Axiom.Spec/specs/increments/_archive/INC-20260806-unify-project-state-paths/candidate-freeze.json`; `Axiom.Spec/specs/increments/_archive/INC-20260806-unify-project-state-paths/receipts/2026-08-06T23-18-32.264Z-freeze-success.json` | BLOCKER | resolved | Freeze final archivado verificado después de la última README y del cierre estructurado. |
| REVIEW-010 | closure | `Axiom.Spec/specs/increments/_archive/INC-20260806-unify-project-state-paths/01_Requisitos.md`; `Axiom.Spec/specs/increments/_archive/INC-20260806-unify-project-state-paths/02_Cambios_Modelo.md`; `Axiom.Spec/specs/increments/_archive/INC-20260806-unify-project-state-paths/03_Criterios_Aceptacion.md`; `Axiom.Spec/specs/increments/_archive/INC-20260806-unify-project-state-paths/04_Interacciones_UI.md`; `Axiom.Spec/specs/increments/_archive/INC-20260806-unify-project-state-paths/context/README.md`; `Axiom.Spec/specs/increments/_archive/INC-20260806-unify-project-state-paths/metadata.yml`; `Axiom.Spec/specs/increments/_archive/INC-20260806-unify-project-state-paths/review-ledger.md` | BLOCKER | resolved | Artefactos completos, metadata `archived`/`integrated` y ledger conforme. |
| REVIEW-DOC-001 | documentation | `Axiom/docs/cli/configure.md`; `Axiom/docs/cli/upgrade.md`; `Axiom/docs/configuration/project-structure.md`; `Axiom/axiom.spec/templates/onboarding-member-template.md`; `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md`; `Axiom.Spec/context/architecture/02-modelo-de-datos-y-configuracion.md` | WARNING | resolved | Claims activos usan `projectKey`; local y executions conservan fronteras correctas. |
| REVIEW-GEN-001 | generated-output | `Axiom/apps/cli/dist/commands/gateway.d.ts.map` | WARNING | resolved | Source map stale eliminado; el source gateway sigue retirado. |
| REVIEW-FINAL-001 | runtime | `packages/toolchain/src/repair.ts`; `packages/toolchain/tests/repair-add-gitignore.test.ts` | CRITICAL | resolved | Se limpian todos los markers legacy cuando existe canonical y se reportan `removedPaths`. Regresión nueva: canonical + dos aliases. |
| REVIEW-FINAL-002 | closure | `Axiom.Spec/specs/increments/_archive/INC-20260806-unify-project-state-paths/context/README.md` | BLOCKER | resolved | Placeholder eliminado y contexto técnico completado. |
| REVIEW-FINAL-003 | documentation | `Axiom/axiom.spec/templates/onboarding-member-template.md` | WARNING | resolved | La plantilla separa members project-scoped de topology bindings local-only. |
| REVIEW-FINAL-004 | governance | `Axiom.Spec/specs/increments/_archive/INC-20260806-remove-legacy-resolution-modes/review-ledger.md`; `Axiom.Spec/specs/increments/_archive/INC-20260806-unify-project-state-paths/review-ledger.md` | WARNING | resolved | Ledgers normalizados al contrato `id/lens/location/severity/status/evidence`. |
| REVIEW-FINAL-005 | coverage | `Axiom/packages/toolchain/tests/repair-add-gitignore.test.ts` | WARNING | resolved | La prueba persistida cubre migración sin canonical con dos aliases y limpieza del segundo alias, además del caso canonical preexistente. |

## Evidencia independiente

- Suite enfocada de blockers: 4 archivos, 52 tests verdes; repair final añade
  una regresión y deja 14 tests en su archivo.
- Suite secundaria doctor/member install: 6 archivos, 100 tests verdes.
- Suite completa: 3326 tests en 327 archivos; los dos fallos de la ejecución
  paralela fueron interferencias no deterministas en tests no relacionados y
  pasan aislados.
- `npm run build`: pasa.
- `npm run doctor`: PASS, 45/60 OK, 0 fallos, 4 advertencias.
- `npm run readiness:first-project`: PASS.
- `get_errors` no reporta errores en los archivos modificados.

## Decisiones

- Canonical siempre gana; la migración legacy es lazy, atómica, idempotente y
  produce warning cuando hay conflicto.
- `config` permanece como label de API compatible, no como segmento físico.
- `.axiom-state/local/` queda reservado a estado repo/operador-local; no se
  usa como fallback project-bound salvo archivos legacy explícitamente conocidos.
- `executions/<executionId>/` mantiene su frontera independiente.

## Cierre

No quedan hallazgos bloqueantes en el alcance funcional. El incremento está
`closed`, `integration.status: integrated` y archivado; el freeze final y los
receipts se conservan en la carpeta histórica.
