# r11-context-tag-selection

> **Código**: INC-20260820-r11-context-tag-selection
> **Estado**: Gestionado por Axiom/Core; consultar `metadata.yml` para el lifecycle.
> **Fecha de creación**: 2026-08-20
> **Tipo de cambio**: modificar

## Resumen

Convierte la recomendación de contexto técnico en un selector único, determinista y basado en etiquetas explícitas de documentos, compartido por las herramientas MCP de lista y lectura compuesta.

## Contexto y motivación

ACC-049 detectó dos comportamientos incompatibles: `spec.recommendedContextList` ignoraba documentos disponibles y `spec.implementationContextRead` devolvía todos ellos. Ninguno elegía contexto según la tarea. El resultado debe usar etiquetas declarativas, no inferencia probabilística ni análisis libre de texto.

## Alcance

### Incluido

- Metadata YAML frontmatter opcional `tags: [..]` en documentos de `Axiom.Spec/context/**`, con normalización estable y vocabulario pequeño.
- Fallback de etiqueta a la carpeta raíz del documento cuando no hay tags explícitos.
- Generación de tags en el índice derivado mediante `axiom context index`, sin editar índices manualmente.
- Un selector compartido que ordena y deduplica: `mandatory.always`, coincidencias de `mandatory.whenTags` y documentos `available` que compartan al menos una `taskTag`.
- Sin `taskTags`, devolución exclusiva de documentos obligatorios.
- Uso idéntico del selector por `spec.recommendedContextList` y `spec.implementationContextRead`; la lectura compuesta acepta `taskTags` y combina de forma explícita las tags estructuradas del plan/rol.
- Preservación de confinamiento al spec repo, presupuesto y distinción observable entre obligatorio y recomendado.
- Pruebas focalizadas de generación, normalización, fallback, orden, deduplicación, presupuestos, aislamiento y dos callers.

### Excluido

- Scoring, recomendación por IA, búsqueda semántica, análisis de texto libre o cambios manuales a `technical-context/indexes/*.index.yml`.
- Nuevos índices obligatorios, backends externos, MCP nuevo o cambios a la autoridad de la spec.

## Documentos del incremento

Los contratos de selector, shape de índice, criterios y superficie MCP se concretan en 01–04.

## Dudas abiertas

Se decide YAML frontmatter opcional como metadata legible. Solo `tags` es nuevo; otro frontmatter se ignora para este selector.

## Decisiones funcionales cerradas

1. La recomendación es determinista por intersección de tags; no existe ranking.
2. El orden contractual es obligatorio primero y disponible recomendado después, sin duplicados por ruta.
3. La ausencia de `taskTags` no convierte el contexto disponible en recomendado.
4. Plan/rol aportan solo señales estructuradas convertidas explícitamente a tags; nunca se infiere del texto.
5. La lista directa responde `malformed` ante tags inválidas, mientras el bundle compuesto conserva su degradación parcial segura: omite contexto técnico seleccionado y señala `technicalContext.taskTags`, sin fallar las fuentes independientes del bundle.

## Consolidación en la spec general

Al cierre del batch R-11, reconciliar el contrato de contexto técnico y MCP en specs 03/05/06 y `context/integrations/`, junto con ACC-048, sin duplicar el historial.

## Estrategia E2E

Ejecutar tests focalizados de `technical-context`, `mcp-tools`/`mcp-server` y CLI `context index`, después build raíz y review independiente.

## Trazabilidad y fuentes

ACC-049 del plan R-11; `packages/technical-context`, `packages/mcp-tools`, `packages/mcp-server`, `apps/cli/src/commands/context.ts` y `Axiom.Spec/context/**`.

## Estado de validación humana

Implementado, revalidado y revisado independientemente sin blockers de código, presupuesto, confinamiento ni contrato MCP.

- Selector común: orden `mandatory.always` → `mandatory.whenTags` con coincidencia ALL → `available` con coincidencia ANY, deduplicado por path.
- `taskTags` omitido y `taskTags: []` son equivalentes: solo contexto obligatorio. Las señales estructuradas de `plan.taskType` y rol se suman exclusivamente cuando existe alguna tag explícita.
- El generador acepta frontmatter YAML opcional `tags`; sin metadata usa fallback de carpeta o `repo`, y el índice real se regenera mediante `axiom context index`.
- Cobertura focal reejecutada: 8 archivos, 97/97 pruebas PASS (`technical-context`, `mcp-tools`, `mcp-server` y CLI).
- `npm run build`: PASS.
- `git diff --check`: exit 0; solo avisos CRLF preexistentes.

## Consolidación final R-11

La reconciliación única del lote actualiza los contratos estables en `specs/00`, `03`, `04`, `05`, `06` y `08`, además de `context/integrations/01-capabilities-providers-y-toolchain.md`, cubriendo ACC-048 y ACC-049 sin copiar su historia de implementación. Se revisaron también `specs/01`, `02` y `07`: no requerían cambios porque no contienen claims activos afectados por los contratos de intercambio de memoria o selección de contexto. El índice derivado se regenera exclusivamente mediante Axiom/Core tras la edición de contexto.
