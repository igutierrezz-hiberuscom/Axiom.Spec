# 02 Cambios de Modelo

## Objetivo del documento

Centralizar la decisión de QA previa al archive en el límite de transición gobernada.

## Entidades o estructuras afectadas

- `QaArchiveDecision` tipada en `@axiom/workflow`.
- Runner gobernado y sus adaptadores CLI, launcher, MCP e integrate.
- Evidencia de `qa-e2e`, `topology.qaLane` y rol QA requerido por plan.

## Contratos o estados afectados

El contrato distingue `passed`, `failed`, `cancelled` y `pending`. `parallel` permite archive con aviso salvo `passed`, mientras inline o QA requerida sólo permite `passed`; una policy o evidencia requerida no evaluable bloquea. `axiom-qa-e2e` recorre el mismo grafo gobernado también en carril inline (`start → verify → pass`), por lo que puede registrar evidencia `passed` antes del archive. Preview expone la decisión y no persiste.

## Notas de compatibilidad

Se retiraron los helpers locales divergentes. La máquina de estados no adquiere estados nuevos y los adaptadores no pueden rebajar un bloqueo a warning.
