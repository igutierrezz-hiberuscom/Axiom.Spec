# Requisitos

## RQ-001. Superficie Copilot

Axiom.SDD debe ofrecer el workflow `axiom-autopilot` mediante una skill, un custom agent orquestador y un prompt `/axiom-autopilot` descubribles desde GitHub Copilot.

## RQ-002. Delegación controlada

El agente orquestador debe poder delegar la ejecución de incrementos y la revisión independiente a workers con límites explícitos, manteniendo la integración final en el orquestador principal.

## RQ-003. Reconciliación normativa

La consolidación final debe comparar los claims afectados con el estado resultante, modificar o eliminar afirmaciones activas obsoletas y conservar la historia únicamente bajo una marca histórica explícita. `SUPERSEDE` no basta por sí solo.

## RQ-004. Contexto técnico

La consolidación debe revisar `Axiom.Spec/context/**`, actualizar sus documentos con fuentes verificables cuando cambie el estado técnico de `Axiom/` y declarar que no aplica cuando el incremento sea tooling-only.

## RQ-005. Trazabilidad

El incremento archivado debe registrar la validación ejecutada, los archivos normativos integrados y el resultado de contexto técnico.
