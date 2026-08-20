# 03 Criterios de Aceptación

## Criterios de aceptación

- [x] CLI, launcher y MCP invocan un mismo runner para transiciones mutantes.
- [x] Preview no escribe y presenta legalidad, confirmación, efectos y QA.
- [x] Todas las transiciones `requiresApproval` bloquean sin confirmación explícita, incluso con force/no-verify.
- [x] Archive/integrate deja estado, metadata, integración y ubicación coherentes o recupera/declara inconsistencia.
- [x] Receipts se emiten desde la ruta común cuando corresponden.
- [x] El efecto tracker inexistente deja de estar en el grafo activo y tracker general permanece.
- [x] Pruebas cubren CLI, launcher, MCP, preview, confirmación, recovery y error de I/O.

### Happy path

La misma transición aprobada produce el mismo resultado visible desde CLI,
launcher o MCP.

### Validaciones y errores

Un fallo de persistencia/move deja el artefacto recuperado o marca una
inconsistencia explícita; nunca éxito parcial silencioso.

### Permisos y visibilidad

La confirmación es visible y explícita en cada canal de mutación. Los bypass de
review o verificación no sustituyen aprobación de lifecycle.

### Estados y efectos observables

El runner persiste solo después de confirmar una transición legal y aplica los
efectos declarados soportados.

## Evidencia de verificación

- Validación dirigida histórica: `npm run build` y la suite de transiciones de
  CLI, launcher, MCP y runner pasaron con `9` archivos y `135` tests; la
  corrección posterior de expectativas stale de preview pasó con `3` archivos y
  `58` tests.
- Corrección final de identidad: `npm run build` y
  `npx --no-install vitest run packages/workflow/tests/governed-transition-runner.test.ts apps/cli/tests/axiom-increment.test.ts apps/cli/tests/axiom-qa-e2e.test.ts apps/cli/tests/functional-verify.test.ts`
  pasaron con `4` archivos y `77` tests. El smoke Core
  `axiom-increment specify --id INC-20260817-r10-acc043-unify-qa-archive-gate --preview --json`
  proyectó `specifying → planned` con el `metadataId` de ACC-043, no el estado
  terminal de otro artefacto.
- Revisión independiente `axiom-review`: primero detectó y se reconciliaron los
  claims documentales stale sobre receipts; la re-revisión confirmó los siete
  criterios técnicos sin blockers, incluido el preview launcher de archive.
- Receipt Core histórico: `receipts/2026-08-18T09-34-21.756Z-verify-success.json`;
  hash interno SHA-256 recalculado y coincidente:
  `a98bc8eaa9014633ed9e0307098a5383201b2bfe7cd527253b591616e0fd9f1b`.
- El cierre y archivado se ejecutan exclusivamente con transiciones Core después
  de la validación global y la revisión final; no se edita metadata ni estado a
  mano.
