# 02 Cambios de Modelo

## Objetivo del documento

Concentrar en un boundary gobernado las decisiones y escrituras de una
transición de workflow, sin convertir la máquina de estados pura en un
servicio de I/O.

## Entidades o estructuras afectadas

- `GovernedTransitionInput`, decisión, resultado aplicado/preview y receipt en
  `@axiom/workflow`.
- `WorkflowStateRecord`, metadata de incrementos/bugs/planes, efectos locales
  declarados y evidencia QA de archive.
- `GovernedWorkflowStateReconciliationInput`: seam compare-and-save limitado
  para bookkeeping o compensación post-transición.

## Contratos o estados afectados

El runner resuelve la configuración efectiva fail-closed, valida legalidad y
plan approval, expone preview sin persistir y exige confirmación explícita
para `requiresApproval`. Para archive/integrate coordinado de incrementos y
bugs, metadata, efectos locales soportados, move a `_archive` y workflow state
terminan coherentes; en un fallo recuperable restaura el estado previo y, si no
puede hacerlo completamente, devuelve inconsistencia explícita.

## Notas de compatibilidad

Los entrypoints CLI, launcher y MCP conservan sus contratos de entrada, pero
delegan su mutación en el runner. Los gates propios de role (affinity, review y
verificación funcional) se ejecutan antes del boundary; `--force`,
`--no-review` y `--no-verify` no equivalen a `confirmed: true`. Las acciones
git opt-in continúan fuera de los efectos que el runner gobierna.