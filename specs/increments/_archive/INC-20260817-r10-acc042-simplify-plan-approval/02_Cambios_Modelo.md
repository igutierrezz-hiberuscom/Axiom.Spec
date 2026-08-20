# 02 Cambios de Modelo

## Objetivo del documento

Reducir el cierre de un plan a una única transición gobernada y verificable.

## Entidades o estructuras afectadas

- Workflow `plan`, metadata de planes y `workflow-state`.
- Operación CLI/launcher de aprobación y gate previo al inicio de roles.

## Contratos o estados afectados

Un plan redactado permanece `draft`; sólo `plan-approve` confirmado, validado por el runner común, produce `plan-approved`. El inicio de roles exige a la vez state y metadata `plan-approved` para un plan existente.

## Notas de compatibilidad

El preview es no mutante. No se añadieron aliases ni estados de aprobación alternativos, y los consumers CLI y launcher conservan el mismo contrato observable.
