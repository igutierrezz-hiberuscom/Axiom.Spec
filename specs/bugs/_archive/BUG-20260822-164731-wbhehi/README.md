# freeze-empty-engram-query

> **Código**: BUG-20260822-164731-wbhehi
> **Estado**: Archivado
> **Fecha de reporte**: 2026-08-22
> **Severidad**: media — bloquea el lifecycle de incrementos, sin pérdida de datos
> **Prioridad**: alta — impide re-congelar y archivar R-12

## Resumen del defecto

`axiom freeze --increment <id>` falla contra Engram 1.17 antes de escribir `candidate-freeze.json`. `hashCandidateInputs()` consulta la memoria con un `MemoryQuery` vacío; el adaptador lo traduce a `mem_search` con `query: ""`, pero el tool MCP requiere una consulta y Engram la envía a FTS5, que rechaza el literal vacío.

## Contexto conocido

R-12 convirtió Engram en el backend único de memoria y eliminó el fallback JSON. El comportamiento fail-closed es correcto para un error real de Engram, pero la entrada vacía es generada por Axiom y representa la operación pública de carga/consulta sin filtros. No es admisible que el propio adaptador produzca una consulta FTS inválida para ese caso.

## Clasificación funcional

- `flow`: bug
- `route`: sdd
- Repositorio propietario: `Axiom/`.
- No hay acción de emergencia, pérdida de datos ni mutación Git.

## Comportamiento actual

`hashCandidateInputs()` ejecuta `queryMemory(backend, scope, {})`. `createEngramBackend().query()` aplica `query.text ?? ''` y llama a `mem_search`. En el entorno observado, `engram mcp --project=axiom-runtime --tools=agent` devuelve el error: `search: SQL logic error: fts5: syntax error near "" (1)`.

## Comportamiento esperado

Una consulta de memoria vacía debe devolver las entradas Axiom del proyecto hasta el límite aplicable sin emitir una expresión FTS5 inválida. En consecuencia, `axiom freeze --increment <id>` debe calcular y escribir un candidate freeze válido cuando Engram está disponible. Las consultas no vacías, los límites, los filtros cliente y los guards cross-project deben conservar su comportamiento.

## Reproducción

### Precondiciones

- Axiom construido desde el worktree actual.
- Engram 1.17.0 disponible en `PATH`.
- Un incremento activo con README y topology/bindings válidos.

### Pasos

1. Desde `Axiom/`, ejecutar `node apps/cli/dist/index.js freeze --increment INC-20260821-r12-engram-only-memory`.
2. Esperar la fase de hash de memoria.

### Resultado observado

El comando termina con código 1 y `AXIOM_MEMORY_QUERY`; no escribe un freeze nuevo. El mensaje identifica el error FTS5 de la consulta vacía.

## Causa raíz

El adaptador Engram reenvía una cadena vacía a `mem_search`, aunque el schema MCP marca `query` como requerido y la implementación real de Engram no admite el término FTS vacío. El stub hermético aceptaba ese valor y por ello no representaba el contrato real.

## Plan de corrección mínimo

1. Representar la consulta vacía con un término de anclaje seguro presente en todas las entradas Axiom válidas —la clave de frontmatter `rationale`, obligatoria por `validateMemoryEvidence`— en vez de enviar `""` a FTS5.
2. Alinear el stub hermético con Engram rechazando consultas vacías.
3. Añadir una regresión de `queryMemory(..., {})` y actualizar la prueba de `load()` para comprobar que se manda el ancla no vacía.
4. Reejecutar las suites `engram-backend` y `freeze`, build y el freeze real que inició el hallazgo.

## Criterios de aceptación

- `queryMemory(backend, scope, {})` devuelve entradas guardadas con el backend Engram hermético y no envía `query: ""`.
- `loadMemory()` conserva sus resultados y no envía una query vacía.
- El stub devuelve error ante `mem_search` vacío para que la regresión sea efectiva.
- Las consultas textuales, filtros, límites y cross-project existentes siguen pasando.
- `axiom freeze --increment INC-20260821-r12-engram-only-memory` finaliza correctamente contra Engram real.

## Superficie de regresión

- `Axiom/packages/memory/src/engram-backend.ts`
- `Axiom/packages/memory/tests/engram-backend.test.ts`
- `Axiom/packages/memory/tests/fixtures/stub-engram-mcp-server.mjs`
- `Axiom/apps/cli/tests/freeze.test.ts`
- `Axiom/apps/cli/src/commands/freeze.ts` (consumidor de la consulta sin filtros; no se prevé cambio)

## No incluidos

- No se añade backend ni fallback JSON.
- No se modifica el protocolo MCP, el pin `--project`, el schema de Engram ni la semántica de `private`/Knowledge Sync.
- No se cambian o borran memorias históricas.
- No se realiza staging, commit, push, reset ni checkout destructivo.

## Trazabilidad y fuentes

- Reproducción local del 2026-08-22 con `axiom freeze`.
- Schema inspeccionado mediante handshake MCP estándar: `mem_search` requiere `query`.
- `Axiom/packages/memory/src/engram-backend.ts`
- `Axiom/apps/cli/src/commands/freeze.ts`
- `Axiom/packages/memory/tests/{engram-backend,fixtures/stub-engram-mcp-server}.mjs`
- `Axiom/apps/cli/tests/freeze.test.ts`

## Estado de validación humana

Corrección implementada y verificada independientemente.

- `ENGRAM_EMPTY_QUERY_ANCHOR = 'rationale'` representa consultas sin texto o con solo whitespace. La invariante es verificable: `validateMemoryEvidence` exige `rationale` y `encodePhaseMetadata` la serializa en el frontmatter de toda entrada guardada por Axiom.
- El stub hermético ahora rechaza `query: ""` con un error FTS5; por tanto, las regresiones no pueden pasar si el adaptador vuelve a emitir una consulta vacía.
- `npx vitest run packages/memory/tests/engram-backend.test.ts apps/cli/tests/freeze.test.ts --reporter=dot` pasó de forma independiente: 2 archivos y 26 pruebas correctas.
- `npm run build` pasó (`tsc -b` y copia de workflow).
- La reproducción que abrió el defecto ya pasa contra Engram real: `node apps/cli/dist/index.js freeze --increment INC-20260821-r12-engram-only-memory` escribió `candidate-freeze.json` con hash `1fcc15bb1ded74d3b7f6aa6ea8e13edf1fb080e7d556805b1de7f731c21bf015`.
- La revisión confirmó los criterios: vacío/whitespace, `load`, límite, filtros, pin de proyecto y guard cross-project permanecen cubiertos; no se añadió fallback JSON ni se cambió la API pública.

El bug confirmado quedó capturado explícitamente en memoria `project-shared` con id `mt4m4ix3-x8gymkit`; una captura adicional idempotente del worker devolvió `mt4m7byj-ag82wlc6`. No hubo mutaciones Git.

## Integración documental y contexto técnico

No requiere cambio en `specs/00..08` ni `context/**`: la corrección restaura el comportamiento ya declarado de carga/consulta de memoria Axiom y no introduce una capacidad, contrato de usuario ni arquitectura nuevos. Tampoco se creó ni editó un índice manualmente.
