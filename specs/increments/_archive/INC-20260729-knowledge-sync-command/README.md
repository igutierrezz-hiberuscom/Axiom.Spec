# INC-20260729-knowledge-sync-command

## Goal

Crear los comandos `axiom knowledge sync` y `axiom knowledge pull` que sincronizan la memoria de Engram entre miembros del equipo usando Git en el repo `<project>.axiom`.

## Scope

- **`axiom knowledge sync --increment <id> --phase <phase>`**: exporta memorias del proyecto activo a `.engram/` (chunks versionables), git add/commit/push en `<project>.axiom`
- **`axiom knowledge pull --increment <id>`**: git pull en `<project>.axiom`, importa chunks nuevos de `.engram/` a la BD local de Engram vía MCP
- El sync usa el backend MCP de Engram (`mem_search` para leer, `mem_save` para escribir) para exportar/importar
- Los artefactos en `.engram/` son JSON estructurados (un archivo por chunk, append-only, sin merge conflicts)
- `.engram/engram.db` está gitignored (solo se versionan los chunks + manifest)

## Non-goals

- No modifica el backend de Engram (`engram-backend.ts`)
- No implementa `engram sync` como CLI de Go (eso es responsabilidad de Engram)
- No implementa sync cloud (solo Git local)
- No modifica `phase-metadata.ts` ni `knowledge.ts` (harvest)

## Acceptance Criteria

1. `axiom knowledge sync --increment INC-xxx --phase architecture` exporta memorias a `.engram/chunks/` y hace git add/commit/push
2. Si no hay memorias nuevas, termina OK sin crear commits vacíos
3. `axiom knowledge pull --increment INC-xxx` hace git pull e importa chunks nuevos
4. Si no hay chunks nuevos, termina OK sin errores
5. Si hay conflicto de Git, lo reporta claramente sin ocultarlo
6. `.engram/engram.db` está en `.gitignore`
7. Build + tests en verde

## Implementation Notes

### Estrategia de sync

El sync de Engram (`internal/sync`) escribe chunks comprimidos `.jsonl.gz` + `manifest.json` en `.engram/`. Como el sync no está expuesto como CLI aún, `axiom knowledge sync` implementa un export/import propio usando el backend MCP:

**Export (sync)**:
1. `resolveMemoryBackend` → `queryMemory` con filtro `increment=<id>`
2. Serializar las memorias como JSON (un archivo `.json` por chunk, nombrado por timestamp)
3. Escribir en `.engram/chunks/<timestamp>-<hash>.json`
4. Actualizar `.engram/manifest.json` (append-only)
5. `git add .engram/ && git commit && git push`

**Import (pull)**:
1. `git pull --rebase`
2. Leer `.engram/manifest.json`
3. Para cada chunk no importado, leer el archivo y hacer `mem_save` por cada memoria
4. Actualizar `.engram/.imported` (tracking de chunks ya importados)

### Archivos a crear/modificar

- **NUEVO**: `Axiom/apps/cli/src/commands/knowledge-sync.ts` — `axiom knowledge sync` y `axiom knowledge pull`
- **MODIFICAR**: `Axiom/apps/cli/src/commands/knowledge.ts` — añadir subcomandos `sync` y `pull`

### Seguridad

- Validar que no se exportan secretos/tokens/credenciales (chequeo básico de patrones)
- Solo memorias con `visibility: 'project-shared'` (o sin visibility = default compartido)
- Nunca exportar memorias con `visibility: 'private'`

### Conflictos

- Append-only por diseño: cada sync crea un chunk NUEVO, nunca modifica chunks viejos
- `manifest.json` es pequeño y mergeable (array de entries)
- Si Git detecta conflicto en `manifest.json`, reportar y abortar (no hacer merge automático)

## Validation

- **Build**: `npm run build` (tsc -b) — OK, sin errores
- **Tests**: `npx vitest run --reporter=verbose packages/memory` — **55/55 passed** (4 test files)

## Result

### Criterios cumplidos

| # | Criterio | Estado |
|---|----------|--------|
| 1 | `axiom knowledge sync` exporta a `.engram/chunks/` y hace git add/commit/push | ✅ |
| 2 | Sin memorias nuevas → OK sin commits vacíos | ✅ `isEmpty: true` |
| 3 | `axiom knowledge pull` hace git pull e importa chunks | ✅ |
| 4 | Sin chunks nuevos → OK sin errores | ✅ |
| 5 | Conflicto de Git → reportado sin ocultar | ✅ |
| 6 | `.engram/engram.db` en `.gitignore` | ✅ |
| 7 | Build + tests en verde | ✅ |

### Fallos

- **Preexistentes**: Ninguno
- **Introducidos**: Ninguno

## General Spec Integration

- `00_Resumen_Ejecutivo.md`: corregida decisión #1 (ya no "sin Git sync", ahora "Sync vía .engram/ chunks + Git"), añadida decisión #4 (validación anti-secretos), añadida mención a sync/pull
- `03_Modelo_Operativo_y_Datos.md`: añadida subsección "Comandos axiom knowledge sync y axiom knowledge pull"
- `04_Flujos_SDD_y_Ciclo_de_Vida.md`: añadida subsección "Sync de memoria entre miembros del equipo"
- `06_Integraciones_y_Capacidades.md`: corregida afirmación "sin Git sync" → ahora describe sync/pull + Git
- `08_Glosario.md`: añadidos 3 términos (`axiom knowledge sync`, `axiom knowledge pull`, `Chunk de memoria`)

### Context technical result

`context/**`: sin cambios requeridos. El nuevo módulo `knowledge-sync.ts` incrementa el conteo CLI en +1. No se añaden packages nuevos.

## Status

closed