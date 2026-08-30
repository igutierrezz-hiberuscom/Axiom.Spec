# 02 Cambios de Modelo

## Objetivo del documento

Reutilizar la metadata ya existente de `MemoryEntry` para hacer visible la intención de compartición y la evidencia de la captura.

## Entidades o estructuras afectadas

- `MemoryAddArgs`: incorpora `visibility?`, `rationale?` y `source?`.
- CLI `memory add`: incorpora las tres options equivalentes.
- `MemoryEntry`: no gana campos nuevos; se rellenan los existentes `visibility`, `rationale` y `source`.
- Política de curación: se retiran los exports de auto/manual-persistencia y cualquier claim de que el runtime guarda por kind.

## Contratos o estados afectados

`visibility: 'private'` y `visibility: 'project-shared'` conservan su semántica existente para Knowledge Sync. Una entrada sin visibilidad explícita continúa disponible localmente y no es elegible para exportación.

## Notas de compatibilidad

Las llamadas programáticas existentes siguen siendo válidas porque los nuevos campos son opcionales. Los imports de la política retirada se eliminan o dejan de exportarse; no existe caller de producción que deba emular auto-persistencia.
