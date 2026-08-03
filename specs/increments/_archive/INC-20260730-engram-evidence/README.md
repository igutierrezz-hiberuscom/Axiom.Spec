# INC-20260730-engram-evidence: Evidencia de Engram

## Metadata

- **ID**: INC-20260730-engram-evidence
- **Status**: closed
- **Goal**: Forzar, en tiempo de ejecución y de forma fail-closed, que todo guardado de memoria persistente (`saveMemory`, y cualquier `MemoryBackend.save()` invocado directamente) requiera un `rationale` (por qué se toma la decisión) y un `source` (de dónde proviene la información o directiva) con contenido real, no vacío ni de solo espacios en blanco.
- **Scope**: `@axiom/memory` (`packages/memory/src/{types,store,engram-backend,evidence,index}.ts`); actualización de callers y tests existentes que dejaban de cumplir el nuevo contrato (`packages/memory/tests/{memory,engram-backend,learning}.test.ts`, nuevo `packages/memory/tests/evidence.test.ts`). Los callers de CLI (`apps/cli/src/commands/{memory,knowledge-sync}.ts`) ya pasaban evidencia real y no requirieron cambios de comportamiento.
- **Non-goals**: No se verifica en tiempo de ejecución si el `rationale`/`source` es semánticamente "suficientemente bueno" (spec explícito). Solo se valida presencia + tipado estricto + un piso mínimo de longitud tras `trim()`. No se toca el path de lectura/reconstrucción (`searchResultToEntry`/`buildEntryFromSearchResult` en `engram-backend.ts`), que puede legítimamente no tener evidencia disponible. No se toca `saveSessionSummary` (concepto distinto, sin campos `rationale`/`source` en su propio tipo `MemorySessionSummary`).

## Acceptance Criteria

1. `saveMemory()` rechaza, con un `Result` de error tipado (`MemoryError.kind === 'missing-evidence'`, con `field: 'rationale' | 'source'`), cualquier draft cuyo `rationale` o `source` esté ausente, sea `undefined`, no sea `string`, o cuyo contenido (tras `trim()`) no supere `MIN_EVIDENCE_LENGTH`.
2. Un string de solo espacios en blanco (`"   "`, `"\t\n"`) cuenta como evidencia ausente (se mide longitud tras `trim()`, no longitud cruda).
3. La validación se ejecuta ANTES de tocar cualquier I/O real (escritura a filesystem del backend en memoria, o spawn del proceso `engram mcp` del backend de Engram) — un save inválido nunca llega a persistir nada.
4. La validación no puede evadirse eligiendo un backend distinto: `createInMemoryBackend().save()` y `createEngramBackend().save()` repiten el mismo chequeo (`validateMemoryEvidence`) de forma independiente, además del chequeo en `saveMemory()`.
5. Un save con `rationale`/`source` válidos (no vacíos, no whitespace-only, longitud suficiente) sigue funcionando exactamente igual que antes (round-trip save→load intacto).
6. El path de lectura/recall (`load`/`query`/`queryMemory`/`buildRecallResult`) sigue funcionando sin cambios de comportamiento, incluyendo la reconstrucción de entries desde resultados de búsqueda de Engram que no traen evidencia propia.
7. Todos los callers internos existentes (`apps/cli/src/commands/memory.ts`, `apps/cli/src/commands/knowledge-sync.ts`, `packages/memory/src/learning.ts`, el auto-save de session summary en `store.ts`) siguen guardando exitosamente, con evidencia real y significativa (no placeholders vacíos).
8. Build (`tsc -b`) y la suite de tests de `packages/memory` y `apps/cli` pasan, salvo el fallo pre-existente ya documentado y fuera de alcance en `engram-backend.test.ts`.

## Implementation Plan

### Fase 1: Modelo de error y validador puro

1. Extender el union `MemoryError` (`types.ts`) con la variante `{ kind: 'missing-evidence', field: 'rationale' | 'source', message: string }`.
2. Crear `packages/memory/src/evidence.ts`: constante `MIN_EVIDENCE_LENGTH` y función pura `validateMemoryEvidence(entry): Result<void, MemoryError>` — sin I/O, reutilizable desde cualquier call site.
3. Exportar `MIN_EVIDENCE_LENGTH` y `validateMemoryEvidence` desde el barrel (`index.ts`).

### Fase 2: Enforcement fail-closed en el camino de escritura

4. `saveMemory()` (`store.ts`): invocar `validateMemoryEvidence(entry)` inmediatamente después de construir la `entry` y ANTES de llamar a `backend.save()`. Rechazo rápido, sin I/O.
5. `createInMemoryBackend().save()` (`store.ts`): invocar el mismo validador como paso 3 (tras las dos validaciones de `projectId` ya existentes, antes del upsert por `topicKey` y de la escritura atómica a disco).
6. `createEngramBackend().save()` (`engram-backend.ts`): invocar el mismo validador tras las validaciones de `projectId`, ANTES de construir `encodedContent`/`args` y de spawnear `engram mcp`.

### Fase 3: Callers y tests

7. Auditar todos los call sites reales de `saveMemory`/`backend.save()` en el repo (CLI, `learning.ts`, tests) y confirmar/actualizar evidencia real donde faltaba.
8. Actualizar `packages/memory/tests/memory.test.ts` (`makeEntry()` con defaults de evidencia + dos calls de `saveMemory` sin evidencia), `packages/memory/tests/engram-backend.test.ts` (11 calls de `saveMemory`/`backend.save()` sin evidencia + 1 nuevo test de defensa en profundidad) y `packages/memory/tests/learning.test.ts` (1 call de `backend.save()` sin evidencia).
9. Crear `packages/memory/tests/evidence.test.ts` con cobertura dedicada del validador puro y del camino `saveMemory`/`backend.save()`.

## Decisiones de implementación

- **Umbral elegido — `MIN_EVIDENCE_LENGTH = 3`**: el campo, tras `trim()`, debe superar los 3 caracteres (mínimo 4 caracteres no-whitespace). Es un piso deliberadamente pequeño y defendible, no un juicio semántico: alcanza para rechazar el string vacío, strings de solo whitespace, y los placeholders más cortos que un agente podría usar para "cumplir" el requisito de forma trivial (`"-"`, `"x"`, `"ok"`, `"na"`), sin opinar sobre el CONTENIDO de un string más largo (el propio non-goal del incremento excluye explícitamente ese juicio semántico). No se intentó filtrar placeholders algo más largos como `"n/a"` o `"tbd"` — eso ya sería juicio de calidad, fuera de alcance.
- **Whitespace-only cuenta como ausente**: la validación mide la longitud DESPUÉS de `trim()`, no la longitud cruda del string. Un `rationale: "   "` es rechazado igual que `rationale: ""` o `rationale: undefined`.
- **Dónde vive el chequeo — en `saveMemory()` Y en cada backend (defensa en profundidad), no solo en uno**: `MemoryBackend` es una interfaz exportada y duck-typed (GATE 0024 §3.4) cuyo método `save()` es parte de la superficie pública — varios tests existentes ya invocaban `backend.save()` directamente, saltándose `saveMemory()`. Poner la validación únicamente en `saveMemory()` habría dejado ese camino sin protección. Se centralizó la lógica en una única función pura (`validateMemoryEvidence`, en `evidence.ts`) para no duplicar reglas, y se la invoca desde tres call sites: `saveMemory()` (rechazo rápido sin tocar I/O, el camino principal y recomendado), `createInMemoryBackend().save()` y `createEngramBackend().save()` (cada uno, defensa en profundidad, ANTES de su propio I/O — escritura a disco o spawn de `engram mcp` respectivamente).
- **Orden del chequeo dentro de cada backend**: la validación de evidencia se ejecuta DESPUÉS de las validaciones de `projectId` (cross-project) ya existentes, y ANTES de cualquier I/O real. Esto preserva el comportamiento de los tests de GATE 0024 existentes (que asertan `cross-project-blocked` con entries que tampoco traían evidencia) sin necesidad de tocarlos, porque el chequeo de proyecto sigue disparando primero cuando ambos fallan a la vez.
- **Session summary (`saveSessionSummary`) queda deliberadamente fuera del enforcement de evidencia**: `MemorySessionSummary` es un tipo separado (solo `projectId`/`content`/`sessionId`, sin `rationale`/`source`) que refleja el concepto distinto `mem_session_summary` de Engram. El propio `saveSessionSummary` de cada backend construye internamente una `MemoryEntry` con literales fijos (`rationale: 'Session summary auto-save'`, `source: 'session-summary'`) que el caller no controla y que ya superan el umbral — no hay draft externo que pueda inventar o vaciar esos campos, así que exigir evidencia ahí sería redundante y, peor, acoplaría un concepto de auto-resumen de sesión a un contrato pensado para decisiones/hechos puntuales justificables. Se documenta la decisión en vez de añadir un chequeo sin caller que lo pueda violar.
- **Path de lectura (`searchResultToEntry`, `buildEntryFromSearchResult` en `engram-backend.ts`) queda sin cambios**: ambas funciones reconstruyen una `MemoryEntry` a partir de resultados de búsqueda/observación de Engram que genuinamente pueden no traer `rationale`/`source` propios (código legado escrito antes de este incremento, o memoria creada directamente vía Engram fuera de Axiom). Hacer que el path de lectura lance o rechace habría roto `load()`/`query()`/`recall` para cualquier entry pre-existente sin evidencia — el propio spec de esta tarea advierte explícitamente contra esto. La ausencia de evidencia en una entry LEÍDA es un dato de hecho sobre el pasado, no una violación de un contrato de escritura nuevo; el fail-closed se aplica únicamente al momento de ESCRIBIR.
- **`isEntryLike()` (shape guard de `createInMemoryBackend`, usado al cargar el JSON persistido) no se modificó**: no exige `rationale`/`source` para considerar una entry válida al leerla del disco. Esto preserva compatibilidad hacia atrás con archivos `.axiom-state/local/memory/<projectId>.json` escritos antes de este incremento (sin esos campos, o con backends/versiones previas) — se siguen cargando y recuperando normalmente. Igual que con el path de Engram, el fail-closed rige el momento de guardar, no el de leer lo ya guardado.
- **Tests actualizados con evidencia real, no placeholders**: todo call site de test que dejó de compilar semánticamente (aunque los tests no se type-checkean — ver hallazgo de auditoría inicial) fue actualizado con un `rationale`/`source` real y específico al escenario que ese test ejercita (p. ej. "Sesiones stateless requeridas para escalar horizontalmente el API" / "ADR-0012 auth architecture review"), no con un string corto arbitrario, seaún la instrucción explícita de no usar "placeholder junk".

## Validación y review

- `npm run build` (desde `C:\repos\Axiom Workspace\Axiom`): **pasa** (`tsc -b`, sin errores).
- `npx vitest run packages/memory apps/cli`: **134 archivos pasan, 1 archivo con 1 fallo — 1311 tests pasan, 1 falla, de un total de 1312 tests.**
  - El único fallo es `packages/memory/tests/engram-backend.test.ts > createEngramBackend > query() with an active kind filter requests the server max (20) but still caps returned entries to the caller's own limit after filtering`, con `Error: Test timed out in 5000ms`. Este es el fallo **pre-existente** documentado explícitamente como fuera de alcance en el brief de este incremento (confirmado antes de empezar). No fue introducido ni "arreglado" por este cambio: los 6 `saveMemory()` de ese mismo test ya incluyen evidencia real tras la actualización de este incremento (por lo que la validación de evidencia no es la causa del timeout); el test sigue fallando por el mismo motivo pre-existente (timeout contra el proceso stub de `engram mcp`), no relacionado con `rationale`/`source`. Se deja tal cual, sin reescribirlo ni intentar "arreglarlo" — es responsabilidad de un ticket de bug futuro, fuera del AC de este incremento.
  - `packages/install-profiles/tests/composer.test.ts` (5 tests, fallo pre-existente reportado en el brief) **no** corrió en este comando (el comando de validación se acotó explícitamente a `packages/memory apps/cli`, por instrucción directa); no se reporta su estado porque no se ejecutó, consistente con el alcance pedido.
  - Conteo por archivo relevante: `packages/memory/tests/evidence.test.ts` (nuevo) 13/13 ok; `packages/memory/tests/memory.test.ts` 15/15 ok; `packages/memory/tests/engram-backend.test.ts` 15/16 ok (1 fallo pre-existente, ver arriba); `packages/memory/tests/learning.test.ts` 16/16 ok; `packages/memory/tests/recall.test.ts` 9/9 ok; `apps/cli/tests/memory.test.ts` 6/6 ok.
- Review contra el AC: los 8 criterios de aceptación fueron verificados manualmente contra el código final y los tests añadidos — AC 1/2/5 cubiertos por `evidence.test.ts` (rechazo de rationale/source ausente y whitespace-only, aceptación de evidencia válida, `error.kind === 'missing-evidence'` con `field` correcto); AC 3/4 cubiertos por el test de defensa en profundidad en `evidence.test.ts` (confirma que la entry rechazada nunca se persiste) y el nuevo test GATE en `engram-backend.test.ts` (confirma que `backend.save()` de Engram rechaza ANTES de intentar la llamada MCP); AC 6 cubierto por que `recall.test.ts` (funciones puras de ranking, no toca `saveMemory`) y los tests de `load`/`query` de `engram-backend.test.ts`/`memory.test.ts` siguen en verde sin cambios; AC 7 verificado por lectura directa de `apps/cli/src/commands/{memory,knowledge-sync}.ts` y `packages/memory/src/learning.ts` (ya pasaban evidencia real, sin necesidad de cambios) y por `store.ts`'s `saveSessionSummary` (literales fijos, documentado en Decisiones); AC 8 verificado por la corrida real reportada arriba.
- No se ejecutó una review adversarial independiente (`axiom-review`/`judgment-day`) para este incremento individual — queda como parte del pase de review a nivel de batch, si corresponde, dado que este incremento es ejecutado como parte de un flujo `/axiom-autopilot`.

## General spec integration

**Realizada** en la pasada única de integración a nivel de lote (2026-08-02), junto con los otros cinco incrementos `INC-20260730-*`. Se tocaron los nueve ficheros canónicos:

- `Axiom.Spec/specs/00_Resumen_Ejecutivo.md`
- `Axiom.Spec/specs/01_Requisitos_Funcionales.md`
- `Axiom.Spec/specs/02_Requisitos_No_Funcionales.md`
- `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md`
- `Axiom.Spec/specs/04_Flujos_SDD_y_Ciclo_de_Vida.md`
- `Axiom.Spec/specs/05_Interfaces_Operativas.md`
- `Axiom.Spec/specs/06_Integraciones_y_Capacidades.md`
- `Axiom.Spec/specs/07_Gobierno_y_Seguridad.md`
- `Axiom.Spec/specs/08_Glosario.md`

Lo aportado por ESTE incremento quedó en: RF-AXM-058 (`01`), NFR-AXM-023 §1 (`02`), evidencia requerida en `MemoryEntry` + la nota sobre `"include": ["src/**/*"]` y el typecheck de tests (`03`), gate de evidencia (`07`), términos `MIN_EVIDENCE_LENGTH`/`missing-evidence` (`08`), resumen de tanda (`00`).

### Contexto técnico (`Axiom.Spec/context/**`)

**Sí aplicó.** Documentos actualizados por este incremento: `references/01-inventario-de-packages.md` (fila `@axiom/memory`), `references/02-historial-de-incrementos.md`, `references/03-riesgos-y-brechas-conocidas.md` (el gate no juzga calidad; la ruta de lectura sigue permisiva).

El pase de contexto no fue solo aditivo: se corrigió el punto 5 de `context/TECHNICAL_CONTEXT.md`, que declaraba `TC-011` como bloqueo vigente y citaba "3425/3427 tests" — `npm run doctor` da hoy `PASS` (46/61 OK, 0 fallos) y la suite vigente es 3489 tests / 3483 verdes / 6 rojos preexistentes.

## Estado de cierre

El incremento se marca `closed`: el goal es claro, los criterios de aceptación están definidos y fueron verificados uno a uno contra el resultado final, los cambios de código están implementados (`evidence.ts` nuevo; `types.ts`, `store.ts`, `engram-backend.ts`, `index.ts` modificados; tests actualizados/añadidos), la validación disponible se ejecutó y se reportó con números reales, y la review contra el AC quedó documentada arriba. La única salvedad — la integración en `Axiom.Spec/general-spec.md`/`specs/00-08`/`context/**` y el archivado de la carpeta — queda explícitamente diferida al pase a nivel de batch por instrucción directa del orquestador (no por omisión): esto replica el patrón ya establecido en incrementos previos ejecutados vía `/axiom-autopilot`, donde el cierre a nivel de incremento individual precede a un pase de integración+archivado consolidado sobre todo el batch. El fallo de `Test timed out` en `engram-backend.test.ts` (`query() with an active kind filter...`) es pre-existente, reproducido igual que en el baseline reportado antes de empezar, y no bloquea el cierre de este incremento por estar fuera de su alcance y AC.
