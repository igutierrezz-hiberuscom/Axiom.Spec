# INC-20260729-knowledge-phase-metadata

## Goal

Extender `MemoryEntry` con campos opcionales de metadata de fase SDD para que las memorias guardadas por agentes de fase lleven trazabilidad suficiente para el Knowledge Harvest.

## Scope

- Añadir campos opcionales a `MemoryEntry` en `@axiom/memory/types.ts`: `increment?`, `phase?`, `actorRole?`, `knowledgeKind?`, `stability?`, `visibility?`, `sourceArtifact?`
- Actualizar `engram-backend.ts` para mapear estos campos a/desde el contenido estructurado de Engram (metadata YAML-like embebida en `content`)
- Actualizar `store.ts` (backend in-memory) para preservar los nuevos campos
- Añadir tipos cerrados: `SddPhase`, `ActorRole`, `KnowledgeKind`, `Stability`, `Visibility`
- Back-compat total: los campos son opcionales, una memoria sin ellos sigue siendo válida

## Non-goals

- No modificar la superficie MCP de `memory.*`
- No crear nuevos comandos CLI
- No modificar `learning.ts`

## Acceptance Criteria

1. `MemoryEntry` acepta los nuevos campos opcionales sin romper entradas existentes
2. El backend engram mapea los campos correctamente en round-trip (save → load)
3. El backend in-memory preserva los campos
4. Build + tests en verde
5. Los tipos cerrados están documentados con JSDoc

## Implementation Notes

### Archivos modificados

| Archivo | Cambio |
|---|---|
| `Axiom/packages/memory/src/types.ts` | Añadidos 5 tipos cerrados (`SddPhase`, `ActorRole`, `KnowledgeKind`, `Stability`, `Visibility`) con JSDoc + 7 campos opcionales a `MemoryEntry` |
| `Axiom/packages/memory/src/phase-metadata.ts` | **Nuevo módulo** con `encodePhaseMetadata` y `decodePhaseMetadata` para serializar/deserializar metadata como frontmatter YAML-like en `content` |
| `Axiom/packages/memory/src/engram-backend.ts` | `save`: `encodePhaseMetadata` antes de enviar a `mem_save`. `load`/`query`: `buildEntryFromSearchResult` obtiene contenido completo vía `mem_get_observation`, decodifica metadata y rehidrata campos. Se añadió stripping del header del stub server (`#<id> [<type>] <title>\n`) para que `decodePhaseMetadata` reciba contenido limpio. |
| `Axiom/packages/memory/src/store.ts` | `saveMemory`: spread de los 7 nuevos campos opcionales desde el draft. El backend in-memory serializa/deserializa el objeto completo vía JSON, así que los preserva automáticamente. |
| `Axiom/packages/memory/src/index.ts` | Export de los 5 nuevos tipos + `encodePhaseMetadata`/`decodePhaseMetadata` |

### Estrategia de encoding

El frontmatter YAML-like se antepone al `text` solo si al menos un campo de metadata de fase está presente. Si no hay metadata, `encodePhaseMetadata` devuelve `entry.text` sin modificar — back-compat total.

Al leer, `decodePhaseMetadata` usa un regex para detectar el bloque `---\n...\n---\n` al inicio del contenido. Si no existe, devuelve `metadata: {}` y el texto intacto.

### Decisión: `mem_get_observation` en load/query

Para recuperar la metadata en round-trip, `load()` y `query()` ahora llaman a `mem_get_observation` por cada resultado de `mem_search`. Esto añade 1 llamada MCP extra por entry, pero es necesario porque `mem_search` solo devuelve `title` (truncado a 300 chars en engram real), no el `content` completo donde va embebido el frontmatter. El fallback si `mem_get_observation` falla es usar solo el `title` (sin metadata), igual que antes.

## Validation

- **Build**: `npm run build` (tsc -b) — OK, sin errores
- **Tests**: `npx vitest run --reporter=verbose packages/memory` — **55/55 passed** (4 test files)
  - `memory.test.ts`: 15/15 ✓
  - `engram-backend.test.ts`: 15/15 ✓
  - `learning.test.ts`: 16/16 ✓
  - `recall.test.ts`: 9/9 ✓

## Result

### Criterios cumplidos

| # | Criterio | Estado |
|---|---|---|
| 1 | `MemoryEntry` acepta nuevos campos opcionales sin romper entradas existentes | ✓ Los 55 tests existentes pasan sin modificación |
| 2 | Backend engram mapea campos en round-trip (save → load) | ✓ `encodePhaseMetadata`/`decodePhaseMetadata` + `buildEntryFromSearchResult` con `mem_get_observation` |
| 3 | Backend in-memory preserva los campos | ✓ `saveMemory` hace spread de los 7 campos; JSON serialization los preserva automáticamente |
| 4 | Build + tests en verde | ✓ `tsc -b` limpio, 55/55 tests |
| 5 | Tipos cerrados documentados con JSDoc | ✓ Cada tipo tiene docstring con enumeración de valores y su significado |

### Fallos clasificados

- **Preexistentes**: Ninguno detectado
- **Introducidos por este incremento**: Ninguno

## General Spec Integration

- `00_Resumen_Ejecutivo.md`: añadida sección "Knowledge Harvest: memoria viva entre fases SDD (2026-07-29)"
- `03_Modelo_Operativo_y_Datos.md`: añadida subsección "Metadata de fase en MemoryEntry" con los 7 campos y 5 tipos cerrados
- `04_Flujos_SDD_y_Ciclo_de_Vida.md`: añadida sección "Memoria viva entre fases y Knowledge Harvest"
- `06_Integraciones_y_Capacidades.md`: añadida subsección "Metadata de fase en memoria y Knowledge Harvest"
- `08_Glosario.md`: añadidos 6 términos (`SddPhase`, `ActorRole`, `KnowledgeKind`, `Stability`, `Visibility`, `Metadata de fase`)

### Context technical result

`context/**`: sin cambios requeridos. El módulo `phase-metadata.ts` es interno a `@axiom/memory` (ya inventariado). Los nuevos tipos no añaden packages ni cambian la arquitectura.

## Status

**closed** — Todos los criterios de aceptación cumplidos, build y tests en verde, sin fallos introducidos.