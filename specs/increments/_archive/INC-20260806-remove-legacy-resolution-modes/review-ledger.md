# Review ledger — INC-20260806-remove-legacy-resolution-modes

Date: 2026-08-06
Flow: increment
Route: sdd
Scope: resolver contract, v1/v2 compatibility, active claims, freeze, receipts and archive closure.

## Alcance

Revisión independiente de `ProjectMode`, normalización v1/v2, consumidores de
`ProjectResolution.mode`, compatibilidad raw legacy, pruebas, freeze y cierre
estructural.

## Hallazgos

| id | lens | location | severity | status | evidence |
|---|---|---|---|---|---|
| REVIEW-008 | governance | `Axiom.Spec/specs/increments/_archive/INC-20260806-remove-legacy-resolution-modes/candidate-freeze.json`; `Axiom.Spec/specs/increments/_archive/INC-20260806-remove-legacy-resolution-modes/receipts/2026-08-06T23-18-25.861Z-freeze-success.json` | BLOCKER | resolved | El freeze histórico quedó stale al cambiar la README. Se reemitió el freeze archivado final y `checkCandidateFreeze` devuelve `ok: true`; el receipt SHA-256 es recomputable. |
| REVIEW-010 | closure | `Axiom.Spec/specs/increments/_archive/INC-20260806-remove-legacy-resolution-modes/01_Requisitos.md`; `Axiom.Spec/specs/increments/_archive/INC-20260806-remove-legacy-resolution-modes/02_Cambios_Modelo.md`; `Axiom.Spec/specs/increments/_archive/INC-20260806-remove-legacy-resolution-modes/03_Criterios_Aceptacion.md`; `Axiom.Spec/specs/increments/_archive/INC-20260806-remove-legacy-resolution-modes/04_Interacciones_UI.md`; `Axiom.Spec/specs/increments/_archive/INC-20260806-remove-legacy-resolution-modes/context/README.md`; `Axiom.Spec/specs/increments/_archive/INC-20260806-remove-legacy-resolution-modes/metadata.yml` | BLOCKER | resolved | Se eliminaron placeholders, se completó la matriz de aceptación, se añadió ledger y metadata quedó `archived`/`integrated` mediante `axiom integrate`. |
| REVIEW-DOC-001 | documentation | `Axiom/docs/cli/configure.md`; `Axiom/docs/cli/upgrade.md`; `Axiom/docs/configuration/project-structure.md`; `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md`; `Axiom.Spec/context/architecture/02-modelo-de-datos-y-configuracion.md` | WARNING | resolved | Las guías activas usan `projectKey` y local-only; las referencias históricas conservan su contexto explícito. |
| REVIEW-DOC-002 | generated-output | `Axiom/apps/cli/dist/commands/gateway.d.ts.map` | WARNING | resolved | Se retiró el source map stale del comando gateway eliminado; el source runtime no fue reintroducido. |
| REVIEW-FINAL-002 | closure | `Axiom.Spec/specs/increments/_archive/INC-20260806-unify-project-state-paths/context/README.md` | BLOCKER | resolved | Se eliminó el placeholder residual y se añadió la frontera de restore/toolchain/providers. |
| REVIEW-FINAL-003 | documentation | `Axiom/axiom.spec/templates/onboarding-member-template.md` | WARNING | resolved | `join` ahora documenta `.axiom-state/<projectKey>/members.yaml`; los topology bindings permanecen en `local/`. |
| REVIEW-FINAL-004 | governance | `Axiom.Spec/specs/increments/_archive/INC-20260806-remove-legacy-resolution-modes/review-ledger.md`; `Axiom.Spec/specs/increments/_archive/INC-20260806-unify-project-state-paths/review-ledger.md` | WARNING | resolved | Ambos ledgers adoptan las columnas y estados `id/lens/location/severity/status/evidence` del contrato de review. |

## Evidencia independiente

- `npm run build`: pasa.
- `npx vitest run packages/project-resolution/tests/resolver.test.ts packages/doctor/tests/checks.test.ts`: pasa.
- Suites dirigidas de resolver/doctor y freeze archivado pasan; la suite global
  en modo paralelo mostró interferencia en dos tests, pero ambos pasan aislados.
- `npm run doctor`: PASS, 45/60 OK, 0 fallos.
- `npm run readiness:first-project`: PASS.
- La declaración compilada de `ProjectMode` queda limitada a `local-only`.

## Decisiones

- `gateway` y `hybrid` se conservan solo como entrada raw compatible o historia
  explícita; no son estados efectivos ni valores del union público.
- El estado del incremento se puede cerrar porque el código, las pruebas, la
  review y la integración canónica están completas; el archivado conserva este
  ledger junto con freeze y receipts.

## Cierre

No quedan hallazgos bloqueantes en el alcance. El incremento está `closed`,
`integration.status: integrated` y archivado; el freeze final y los receipts
se conservan en la carpeta histórica.
