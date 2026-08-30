# 02 Cambios de Modelo

## Objetivo del documento

Precisar el contrato CLI, los chunks compartidos y el estado local de importación.

## Entidades o estructuras afectadas

- Argumentos/resultados de `runKnowledgeSync` y `runKnowledgePull`.
- Wrapper Commander de `knowledge sync` y `knowledge pull`.
- Schema versionado de manifest/chunk y la proyección de `MemoryEntry` compartible.
- Marker project-scoped de chunks importados y migración del legacy `.engram/.imported`.

## Contratos o estados afectados

`knowledge pull --increment` deja de ser válido. Sync/pull sin `--confirm` son previews sin escritura. `--push` no equivale a confirmación y no habilita mutación solo.

## Notas de compatibilidad

Se puede leer el marker legacy para migrarlo al namespace canónico. Los chunks preexistentes sin campos de evidencia deben fallar visiblemente/reintentarse, no inventar texto ni marcarlos importados.
