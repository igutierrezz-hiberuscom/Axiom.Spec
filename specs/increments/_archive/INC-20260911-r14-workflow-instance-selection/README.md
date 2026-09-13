# r14 workflow instance selection

> **Código**: INC-20260911-r14-workflow-instance-selection
> **Estado**: En Borrador
> **Fecha de creación**: 2026-09-11
> **Tipo de cambio**: corrección de contrato del engine de workflow
> **Acción de origen**: `ACC-087`
> **Plan**: `PLAN-INC-20260911-r14-operator-control-surfaces` (posición 1 de 8, desbloquea el lote)

## Resumen

Permitir que convivan varios artefactos en vuelo y que el operador elija de forma explícita sobre cuál opera, eliminando el bloqueo por instancia única del estado de workflow. Es la primera pieza del lote porque, hasta que exista, solo puede avanzar el último artefacto creado.

## Contexto y motivación

Verificado el 2026-09-11 sobre el runtime y en ejecución real:

- `packages/workflow/src/state-store.ts` persiste **un único record por `workflowId`** en `.axiom-state/<projectKey>/workflow-state.json`, con `schemaVersion: 1`. El propio encabezado del módulo documenta ese formato.
- `runGovernedTransition` ya sabe resolver el estado por artefacto: si el id pedido no coincide con el ligado, sintetiza el record desde `metadata.yml#status` del artefacto o desde el `initialState`. La capacidad existe en el runner.
- El wrapper CLI `apps/cli/src/commands/axiom-increment.ts` rechaza antes cualquier subcomando distinto de `create` cuando el id recibido no coincide con el ligado y el estado no es terminal: `[axiom increment] ID mismatch: se recibió 'A', pero workflow-state.json está ligado a 'B'`.
- `create` solo es legal desde `draft`, así que re-ligar un artefacto que ya está en `specifying` se rechaza: `No transition for command 'increment-create' from state 'specifying'`.
- `axiom state` es un inspector read-only y no existe comando soportado de selección o reasignación. `reconcileGovernedWorkflowState` existe en `@axiom/workflow`, pero solo lo consume `axiom-role`.

Consecuencia demostrada al preparar este lote: al crear ocho incrementos, el estado quedó ligado al último y los siete anteriores no pueden avanzar hasta que el ligado alcance estado terminal. El orden de ejecución deja de ser una decisión de planificación y pasa a ser un efecto colateral del orden de creación.

## Alcance

### Incluido

- Estado por instancia identificada (`workflowId` + `metadataId`) en lugar de un único record por `workflowId`.
- Comando soportado de selección/activación explícita del artefacto sobre el que se opera.
- Sustitución del rechazo por `ID mismatch` por un error accionable que indique cómo seleccionar la instancia.
- `axiom state` listando todas las instancias en vuelo con estado y siguiente paso recomendado, sin dejar de ser read-only.
- Migración determinista, atómica e idempotente desde `schemaVersion: 1`, conservando el record vigente y los receipts.
- Alcance de workflows: los que tienen transición identificada por artefacto (`increment`, `bug`, `plan`).

### Excluido

- Los carriles `qa-e2e` y `role`, que conservan su semántica actual.
- Cualquier relajación de legalidad: ninguna transición ilegal debe volverse legal por este cambio.
- Habilitar edición manual del estado o cualquier vía para saltarse los gates de aprobación.
- Ejecución concurrente real de dos transiciones simultáneas: el objetivo es coexistencia y selección, no paralelismo de escritura.

## Dudas abiertas

Ninguna. Cerradas en la fase de decisiones previa (D-06):
- Selección dual: soporte directo de `--id <id>` en todos los subcomandos + comando explícito `select --id <id>`.
- Instancias terminales: purga de las instancias activas al archivar (`workflow-state.json` almacena solo lo que está en vuelo; la verdad inmutable vive en `metadata.yml`).
- Alcance de migración: unificada y simultánea para `increment`, `bug` y `plan` a `schemaVersion: 2`.

## Decisiones funcionales cerradas

- El bloqueo actual no se documenta como comportamiento aceptable: se corrige.
- El runner conserva su comportamiento fail-closed; el cambio es de estado y de selección, no de legalidad.
- `workflow-state.json` migra a `schemaVersion: 2` de forma atómica e idempotente.
- Soporte de subcomando `select` y compatibilidad con `--id <id>`.
- `qa-e2e` y `role` se preservan como records directos en el estado de workflow.

## Consolidación en la spec general

Al cierre, reconciliar las afirmaciones activas sobre el estado de workflow y el ciclo de vida de artefactos: dejan de describir un singleton por workflow y pasan a describir instancias identificadas con selección explícita.

## Estrategia E2E

Dos y tres instancias simultáneas en vuelo; selección explícita y avance de una sin afectar a las otras; id inexistente; instancia terminal; migración desde `schemaVersion: 1` con record previo y sin él; ausencia de herencia de `vars` entre instancias; y regresión de que ninguna transición ilegal pasa a ser legal.

## Trazabilidad y fuentes

Acción `ACC-087` y sesión R-14 del 2026-09-11 en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md`; evidencia en `Axiom/packages/workflow/src/{state-store,governed-transition-runner,recommend-next}.ts`, `Axiom/apps/cli/src/commands/{axiom-increment,state-cmd}.ts` y `Axiom/axiom.config/workflows.yaml`.

## Estado de validación humana

Validado. Criterios AC-087-01 a AC-087-08 verificados:
- `schemaVersion: 2` implementado en `state-store.ts` con migración transparente e idempotente desde `schemaVersion: 1`.
- Coexistencia de múltiples instancias en vuelo para `increment`, `bug` y `plan` comprobada en pruebas unitarias y de integración.
- Subcomando `select` añadido a `axiom-increment`, `axiom-bug` y `axiom-plan`.
- Soporte transparente de `--id <id>` sin rechazo por `ID mismatch`.
- `axiom state` enriquece la vista de workflows mostrando todas las instancias en vuelo, indicando cuál es la activa (`*`), su estado y transiciones recomendadas.
- Purga de instancias terminadas al archivar preservando receipts y `metadata.yml`.
- Evidencia: 51 tests focales PASS (`state-store.test.ts` 15/15, `state-cmd.test.ts` 6/6, `axiom-increment.test.ts` 15/15, `axiom-bug.test.ts` 9/9, `axiom-plan.test.ts` 6/6). Build limpio, doctor PASS (48/61 OK, 0 fallos), readiness PASS, `git diff --check` limpio.
- Receipts emitidos: `apply` (`cf482eb9...`) y `verify` (`22db64af...`).
