# 03 Criterios de Aceptación

> **Estado documental**: satisfecho y archivado.
> **Evidencia principal**: `npx vitest run packages/memory/tests/engram-backend.test.ts apps/cli/tests/freeze.test.ts --reporter=dot` — 2 archivos y 26 pruebas correctas; `npm run build` correcto; freeze real correcto contra Engram 1.17.

## Criterios de aceptación del bug

1. Una consulta de memoria sin texto o con solo whitespace devuelve las entradas Axiom del proyecto sin enviar `query: ""` a `mem_search`.
2. `loadMemory()` conserva las entradas del proyecto y utiliza una consulta no vacía.
3. El stub MCP hermético rechaza explícitamente una query vacía con el error FTS5 equivalente, para impedir que la regresión quede ocultada por el double de prueba.
4. Las consultas textuales, límites, filtros cliente, pin `--project=<projectId>` y guards cross-project conservan su comportamiento previo.
5. `axiom freeze --increment INC-20260821-r12-engram-only-memory` calcula y escribe un `candidate-freeze.json` válido contra Engram real.

## Corrección principal

El backend normaliza una consulta vacía o whitespace al ancla `rationale`. La invariante es segura para entradas Axiom: `validateMemoryEvidence` exige `rationale` y `encodePhaseMetadata` la serializa en el frontmatter antes de guardar en Engram. No se modifica el protocolo MCP, no se añade fallback JSON y no se cambia la API pública.

## No regresión

La prueba hermética cubre `load`, query vacía y whitespace, límite por defecto y máximo de Engram, filtros por kind/tags, metadata de observaciones, compatibilidad raw, upsert, session summary, errores de binario y aislamiento cross-project. La reproducción real que abrió el bug pasó y escribió el freeze con hash `1fcc15bb1ded74d3b7f6aa6ea8e13edf1fb080e7d556805b1de7f731c21bf015`.

## Permisos, validaciones o side effects

La corrección es de lectura de memoria project-scoped. Mantiene el proceso Engram fijado por `--project`, no envía un payload `project` redundante y propaga errores MCP como `MemoryError` visible. No escribe datos fuera del `candidate-freeze.json` solicitado por el comando `freeze`, no toca memorias históricas y no ejecuta mutaciones Git.
