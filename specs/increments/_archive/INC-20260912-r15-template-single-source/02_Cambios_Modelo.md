# 02 Cambios de Modelo

## Objetivo del documento

Detallar qué estructuras y contratos cambian al colapsar las copias de plantillas en una sola fuente.

## Estructuras afectadas

- `Axiom/axiom.spec/templates/` (45 archivos): pasa a ser fuente única. Recibe el contenido rescatado y las correcciones de vocabulario.
- `Axiom.Spec/templates/` (45 archivos): se retira por completo tras el rescate.
- `Axiom.Spec/README.md`: se reconcilia la estructura declarada del repositorio canónico.

## Código afectado

- `packages/workflow/src/artifact-skeleton.ts`: comentarios de procedencia y alcance del test dorado. Las copias bundleadas siguen existiendo; cambia su fuente de verdad declarada y la comprobación que las protege.
- `packages/cli-commands/src/commands/workspace-adapter-templates.ts`: comentarios de procedencia de `AGENTS_MD_TEMPLATE` y descripción del override on-disk.
- `packages/workflow/tests/artifact-skeleton.test.ts`: el test dorado pasa de comparar títulos de sección de unos pocos archivos a comparar la fuente única completa.
- `packages/adapters/{opencode,claude-code,codex,antigravity}/src/generator.ts` y sus `types.ts`: no cambia `DEFAULT_TEMPLATE_PATH`; se revisan los comentarios que describen de dónde procede la plantilla.

## Contratos o estados afectados

- Cambia el contrato documental de procedencia: la fuente de verdad de plantillas es el runtime, no el repositorio canónico.
- No cambia el contrato de resolución en tiempo de ejecución: el override on-disk sigue teniendo precedencia sobre la copia bundleada.
- No cambia ningún esquema, estado persistido ni salida de comando.

## Riesgo estructural

Retirar plantillas del repositorio canónico afecta a la frontera declarada por ADR-0032. No se reabre la decisión: se documenta que las plantillas son material de producto y viven con el producto.
