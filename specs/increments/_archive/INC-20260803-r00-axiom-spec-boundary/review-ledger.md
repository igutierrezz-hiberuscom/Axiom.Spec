# Independent review ledger

- Increment: `INC-20260803-r00-axiom-spec-boundary`
- Review mode: `axiom-review`, read-only except this ledger.
- Review date: `2026-08-03`.
- Scope: `Axiom/axiom.spec/`, runtime/catalog consumers, adapters, readiness, artifact store, topology, `axiom.yaml`, overview, ADR-0032, `specs/00..08`, `context/**`, README, metadata, acceptance criteria, freeze, receipt, and declared validation evidence.
- Sweep budget: 6 focused sweeps completed. The final two sweeps covered active workflow guidance, materializable templates, catalog hashes, and post-fix validation.
- Revalidation date: 2026-08-03.
- Mutations: the orchestrator applied the documented remediation outside this ledger; no Git index, commit, branch, or destructive checkout mutation was performed.

## Findings

| id | lens | location | severity | status | evidence |
| --- | --- | --- | --- | --- | --- |
| REVIEW-001 | review | `receipts/2026-08-03T14-40-39.726Z-freeze-success.json`; `receipts/2026-08-03T14-58-08.833Z-apply-success.json`; `Axiom.SDD/.agents/skills/axiom-autopilot/SKILL.md` | CRITICAL | resolved | The historical `13:14` apply remains explicitly non-governed because it predates the historical freeze. A new freeze receipt at `14:40:39.726Z` precedes the idempotent apply receipt at `14:58:08.833Z`; both hashes recompute correctly and the later verify receipt is present. |
| REVIEW-002 | review | `Axiom/docs/overview.md`; `Axiom/scripts/verify-first-project-readiness.mjs`; `Axiom/axiom.config/skills-catalog.yaml`; `Axiom/packages/workflow/src/artifact-store.ts` | WARNING | resolved | The overview now states that `Axiom/axiom.spec/` is product-owned baseline content consumed as materializable source, while `Axiom.Spec/` remains the canonical workspace repository. |
| REVIEW-003 | review | `Axiom/axiom.yaml`; `Axiom.Spec/context/TECHNICAL_CONTEXT.md` | WARNING | resolved | The runtime description and technical context now use the verified current figures: 43 packages, 81 CLI command files, doctor 46/61 with 0 failures, and the characterized test baseline. |
| REVIEW-004 | review | `Axiom.Spec/specs/07_Gobierno_y_Seguridad.md:88`; `Axiom.Spec/context/TECHNICAL_CONTEXT.md:52` | WARNING | resolved | The 2026-07-30 `TC-011` failure and 3425/3427 test count are explicitly labeled as historical; the active statement records doctor/readiness PASS and the current characterized baseline. |
| REVIEW-005 | review | `Axiom/axiom.spec/templates/**`; `Axiom/apps/cli/src/commands/workspace-adapter-templates.ts`; active runtime comments and diagnostics | WARNING | resolved | Active guidance no longer points to absent `axiom.spec/capabilities`, `integrations`, `policies`, `adapters`, `decisions`, or `scripts`; it now points to `.axiom-state/`, `axiom.config/`, `Axiom.Spec/`, or the actual validation commands. |
| REVIEW-006 | review | `receipts/2026-08-03-final-validation.json`; `receipts/2026-08-03T15-23-22.372Z-verify-success.json` | WARNING | resolved | Dedicated validation evidence now records build, doctor, readiness, and the expected no-match legacy-route sweep after the final corrections. The formal verify receipt was emitted after those checks and its SHA-256 recomputes correctly. |
| REVIEW-007 | review | `03_Criterios_Aceptacion.md`; `README.md`; `metadata.yml`; `_archive/` directory | CRITICAL | resolved | Final spec/context integration completed; the README and criteria are closed and the metadata is archived with `integration.status: integrated`. |
| NEW-001 | review | `Axiom.SDD/.agents/**`; `Axiom.SDD/.github/prompts/**`; `Axiom.SDD/.claude/**`; `Axiom.SDD/.opencode/**` | WARNING | resolved | Active workflow surfaces no longer refer to the removed `general-spec.md`; they now name the canonical numbered `Axiom.Spec/specs/00..08` files. |
| NEW-002 | review | `Axiom/axiom.spec/templates/**`; `Axiom/apps/cli/src/commands/workspace-adapter-templates.ts`; `Axiom/axiom.config/skills-catalog.yaml` | WARNING | resolved | Materializable guidance now uses `.axiom-state/`, `axiom.config/`, and current canonical spec/context paths. The bundled template is synchronized and `TC-010` passes with the recalculated skill hash. |

## Cumplimiento

| criterio | resultado | evidencia observada |
| --- | --- | --- |
| Inventario y trazabilidad de consumidores | compliant in substance | Cinco familias comprobadas: `increments/` (2), `plans/` (0), `target-axiom-agents/` (14), `target-axiom-skills/` (20), `templates/` (45). Catálogos, readiness, adapters, artifact store y topología coinciden con el árbol vivo. |
| Decisión sobre conservar, mover, borrar o renombrar | compliant | ADR-0032 conserva `Axiom/axiom.spec/` sin moverla, borrarla ni renombrarla; coincide con los consumidores verificados. |
| Distinción entre `Axiom.Spec/` y `Axiom/axiom.spec/` | compliant | Ownership y papel materializable están descritos en overview, topología, contexto y ADR-0032. |
| Sin cambios de comportamiento runtime ni baseline generada | compliant for reviewed scope | No se observó cambio de lógica; las referencias legacy son comentarios, documentación o diagnóstico y las fuentes catalogadas existen con hashes correctos. |
| Specs y contexto sin claims contradictorios | compliant for reviewed scope | `specs/07`, contexto, `axiom.yaml` y las referencias activas revisadas reflejan el estado vigente; los snapshots históricos están etiquetados como históricos. |
| Revisión independiente registrada | compliant after this ledger | Este ledger registra alcance, evidencia y hallazgos abiertos. |
| Integración final y archivo | compliant | README y criterios están cerrados, metadata está `archived` con `integration.status: integrated`, y la carpeta está bajo `_archive/`. |

## Desviaciones

- El apply histórico de las `13:14` quedó explícitamente fuera del gobierno porque precede al freeze histórico; la secuencia reestablecida de `14:40` freeze y `14:58` apply sí cumple el orden requerido.
- Los claims de materialización, métricas, estado de doctor y rutas legacy fueron reconciliados; los snapshots históricos permanecen etiquetados como históricos.
- La checklist de aceptación, el README, metadata y el archivado final quedaron reconciliados después de la integración canónica.

## Riesgos

- No queda riesgo bloqueante en el alcance revisado. La desviación histórica de freeze/apply se conserva como historia y no se reinterpreta como evidencia gobernada.

## Validación observada

- Inventario filesystem: `increments/` 2 archivos, `plans/` 0, `target-axiom-agents/` 14, `target-axiom-skills/` 20 y `templates/` 45; total 81 archivos.
- Catálogos: 14 agentes y 20 skills; 0 fuentes ausentes y 0 `bundleHash` discrepantes.
- Freeze: el hash del README coincide con `specsHash`; la recomputación con backend JSON reprodujo `memoryHash`, `specsHash` y `combinedHash`. La consulta de memoria por defecto no pudo resolver `sdd-repo` (`unknown_project`), por lo que se usó el backend JSON explícito.
- Receipt apply: el SHA-256 almacenado `d9bd40077d35b4cd901947e6fe72412bf6c11c757a5c9d0b9a642a4254c5ff84` coincide con el hash recalculado del payload.
- Familias legacy bajo `Axiom/axiom.spec/`: `capabilities/`, `integrations/`, `policies/`, `adapters/`, `decisions/`, `scripts/` y `config/` no existen.
- `get_errors` encontró errores preexistentes de tipos Node en `apps/cli/src/commands/start.ts`; `workspace-adapter-templates.ts` y `packages/memory/src/types.ts` no tienen diagnósticos. `tsc -b` pasa.
- `npm run build`, `npm run doctor` y `npm run readiness:first-project` se rerun desde `Axiom/` después de los fixes; el resultado está persistido en `receipts/2026-08-03-final-validation.json`.
- No se ejecutó ningún comando Git mutante de índice, commit, branch o checkout.

## Acciones documentales para el orquestador

1. No queda acción documental bloqueante. Mantener junto al incremento archivado los receipts de freeze/apply/verify/knowledge y la validación final.
2. No reinterpretar el apply histórico de `13:14` como gobernado.

## Recomendación

`closed`: la frontera, las rutas activas, la validación, la evidencia gobernada, la integración canónica, el cierre README/metadata y el archive están verificados.

## Mensaje de commit sugerido

`docs: clarify axiom.spec product boundary and freeze evidence`
