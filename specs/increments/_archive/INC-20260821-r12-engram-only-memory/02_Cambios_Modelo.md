# 02 Cambios de Modelo

## Objetivo del documento

Eliminar la implementación alternativa de persistencia y mantener el contrato de backend único.

## Entidades o estructuras afectadas

- Se retiran el constructor JSON y sus helpers de path/persistencia local.
- `resolveMemoryBackend` deja de seleccionar un `kind: 'json'`, de aceptar `forceJson` o de presentar un note de fallback.
- Los helpers genéricos `loadMemory`, `saveMemory`, `queryMemory` y `saveMemorySessionSummary` permanecen si siguen siendo adaptadores puros sobre `MemoryBackend`.
- Doctor incorpora un check de disponibilidad de Engram dentro de la categoría `memory`.

## Contratos o estados afectados

Los callers reciben el backend Engram o un error tipado proveniente de su operación. No cambia `MemoryEntry`, `MemoryQuery`, los tipos de visibilidad ni el protocolo de Knowledge Sync.

## Notas de compatibilidad

Se elimina una API de fallback intencionalmente. Los tests y callers deben inyectar/usar stubs de backend o el stub de Engram; no deben depender de la existencia de un JSON local. Los archivos JSON históricos quedan sin consumir.
