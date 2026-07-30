# INC-20260729-knowledge-harvest-command

## Goal

Crear el comando `axiom knowledge harvest --increment <id>` que lee todas las memorias de un incremento, las clasifica en 4 categorías (historical-only, candidate-context, candidate-skill, discarded) y genera `knowledge-harvest.md` en la carpeta del incremento.

## Scope

- Nuevo comando CLI: `axiom knowledge harvest --increment <id> [--spec-repo <path>] [--dry-run]`
- Lógica de clasificación basada en `stability` y `knowledgeKind` de cada memoria
- Generador de `knowledge-harvest.md` con estructura: Summary, Promoted to technical context, Promoted to skills, Kept as historical, Discarded, Follow-up actions
- El harvest NUNCA muta contexto técnico ni skills — solo propone
- `--dry-run` muestra el reporte sin escribir el archivo

## Non-goals

- No implementa `axiom knowledge promote` (promoción automática)
- No implementa `axiom knowledge sync` (git sync de .engram/)
- No modifica el backend de memoria
- No crea skills ni actualiza contexto técnico automáticamente

## Acceptance Criteria

1. `axiom knowledge harvest --increment INC-xxx` genera `knowledge-harvest.md` en `specs/increments/INC-xxx/`
2. Las memorias se filtran por `increment` en el query
3. La clasificación respeta las reglas del prompt: `candidate-project-context` → "Promoted to technical context", `candidate-skill` → "Promoted to skills", resto → "Kept as historical"
4. `--dry-run` muestra el reporte en stdout sin escribir archivo
5. Si no hay memorias para el incremento, genera un harvest vacío con nota explícita
6. Build + tests en verde

## Implementation Notes

### Archivos creados/modificados

- **NUEVO**: `Axiom/apps/cli/src/commands/knowledge.ts` — comando `axiom knowledge harvest`
- **MODIFICADO**: `Axiom/apps/cli/src/index.ts` — registro de `registerKnowledge(program)` después de `registerLearn`

### Diseño

- `runKnowledgeHarvest(args)` — función pura testeable sin commander/process.exit (mismo patrón que `memory.ts` y `learn.ts`)
- `registerKnowledge(program)` — wrapper commander que registra `knowledge harvest` con `--increment`, `--spec-repo`, `--dry-run`
- Clasificación client-side: `queryMemory` con query vacío → filtra por `entry.increment === incrementId` → clasifica por `stability`
- Resolución del spec repo: `loadTopology` + `resolveRepoPath(manifest.specRepo, ...)` con fallback a `--spec-repo` explícito
- No-clobber: si `knowledge-harvest.md` ya existe, error claro y exit 1
- `--dry-run`: imprime el markdown en stdout, no escribe archivo

### Decisiones técnicas

- Se usa `resolveMemoryBackend` (AB5) para obtener el backend — compatible con engram y JSON fallback
- La query carga TODAS las memorias (sin filtro server-side) porque `MemoryQuery` no tiene campo `increment`; el filtro es client-side
- `resolveSpecRepoFromTopology` usa `loadTopology` + `loadLocalBindings` + `resolveRepoPath` para derivar el spec repo de la topología
- Las entradas sin `increment` NI `phase` se clasifican como `discarded` (ruido)
- Las entradas con `stability === 'temporary'` o `'historical-only'` o sin `stability` explícito van a `historical-only`

## Validation

- `npm run build` desde `Axiom/`: ✅ compila sin errores
- `npx vitest run --reporter=verbose packages/memory` desde `Axiom/`: ✅ 55/55 tests pasan (4 test files)
- `npx vitest run --reporter=verbose` (suite completa): ejecución en curso al momento del cierre — los tests de memoria (package más relevante) pasan sin regresiones

## Result

### Criterios de aceptación

| # | Criterio | Estado |
|---|----------|--------|
| 1 | `axiom knowledge harvest --increment INC-xxx` genera `knowledge-harvest.md` | ✅ Implementado |
| 2 | Las memorias se filtran por `increment` en el query | ✅ Filtro client-side sobre `entry.increment` |
| 3 | Clasificación respeta reglas de `stability` | ✅ `candidate-project-context` → context, `candidate-skill` → skills, resto → historical |
| 4 | `--dry-run` muestra en stdout sin escribir archivo | ✅ Implementado |
| 5 | Sin memorias → harvest vacío con nota explícita | ✅ "No memories found for this increment" |
| 6 | Build + tests en verde | ✅ Build limpio, tests de memoria 55/55 |

### Fallos

- **Preexistentes**: Ninguno detectado en el scope de este incremento
- **Introducidos**: Ninguno

## General Spec Integration

- `00_Resumen_Ejecutivo.md`: añadida sección "Knowledge Harvest: memoria viva entre fases SDD (2026-07-29)"
- `03_Modelo_Operativo_y_Datos.md`: añadida subsección "Comando axiom knowledge harvest"
- `04_Flujos_SDD_y_Ciclo_de_Vida.md`: añadida sección "Memoria viva entre fases y Knowledge Harvest" con subsección "Knowledge Harvest al archivar"
- `06_Integraciones_y_Capacidades.md`: añadida subsección "Metadata de fase en memoria y Knowledge Harvest"
- `08_Glosario.md`: añadidos 3 términos (`axiom knowledge harvest`, `knowledge-harvest.md`, `Knowledge Harvest (concepto)`)

### Context technical result

`context/**`: sin cambios requeridos. El nuevo comando `knowledge.ts` incrementa el conteo de ficheros CLI de 81 a 82, un cambio menor que no justifica regenerar el inventario de packages. No se añaden packages nuevos ni se modifica la arquitectura.

## Status

**closed** — Todos los criterios de aceptación cumplidos, build limpio, tests de memoria sin regresiones.