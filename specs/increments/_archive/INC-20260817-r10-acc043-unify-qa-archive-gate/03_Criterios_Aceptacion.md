# 03 Criterios de Aceptación

## Criterios de aceptación

- [x] Las dos implementaciones de `checkQaArchiveGate` desaparecen y no queda un helper duplicado con la misma política.
- [x] Existe un único contrato QA tipado, invocado por el runner gobernado antes de cualquier archive mutante.
- [x] El contrato distingue explícitamente `passed`, `failed`, `cancelled` y `pending`, sin tratar `failed` o `cancelled` como equivalentes a éxito.
- [x] `qaLane: parallel` permite archive para cada estado QA y muestra el estado/aviso correspondiente de forma consistente.
- [x] QA inline o requerida bloquea archive para `pending`, `failed` y `cancelled`; con `passed` permite continuar.
- [x] Increment, bug, integrate, launcher y MCP reciben la misma decisión; ningún adaptador rebaja un bloqueo a warning.
- [x] Un preview muestra la disposición QA y no escribe workflow-state, metadata, receipt ni mueve el artefacto.
- [x] Las pruebas cubren la matriz de policy (parallel/inline/requerida) por estado QA y al menos un camino de archivo y bloqueo por cada familia de superficie afectada.

### Happy path

Un archive con QA requerido y evidencia `passed` se completa por el runner; un archive paralelo comunica su estado QA y se completa sin alterar el resultado según el canal de entrada.

### Validaciones y errores

Una policy QA requerida ilegible o evidencia no evaluable bloquea antes de persistir. Los errores explican el modo y estado que causaron el resultado.

### Permisos y visibilidad

La policy QA no sustituye la confirmación declarada. La decisión QA se incluye en el resultado/previsualización visible al operador.

### Estados y efectos observables

En bloqueos no hay cambio de state, metadata, `integration.status`, movimiento a `_archive` ni receipt de éxito. En archives permitidos el estado QA comunicado coincide con la evaluación canónica.

## Evidencia de verificación

- Corrección de QA inline: `npm run build` y
  `npx --no-install vitest run packages/workflow/tests/governed-transition-runner.test.ts apps/cli/tests/axiom-increment.test.ts apps/cli/tests/axiom-qa-e2e.test.ts apps/cli/tests/functional-verify.test.ts`
  pasaron con `4` archivos y `77` tests.
- Smoke y evidencia Core: el preview de `axiom-qa-e2e start` proyectó
  `draft → verifying` sin persistir. La secuencia real
  `axiom-qa-e2e start`, `axiom-qa-e2e verify --run-validation` y
  `axiom-qa-e2e pass` terminó en `qa-e2e: archived`; el estado archivado es la
  evidencia `passed` que el gate inline requiere antes de archive.
- La validación global y la re-revisión final del lote se registran antes del
  archivado; ningún status o metadata se modificó manualmente.