# r11-knowledge-sync-hardening

> **Código**: INC-20260820-r11-knowledge-sync-hardening
> **Estado**: Gestionado por Axiom/Core; consultar `metadata.yml` para el estado de lifecycle.
> **Fecha de creación**: 2026-08-20
> **Tipo de cambio**: modificar

## Resumen

Endurece `axiom knowledge sync/pull` para compartir solo memoria explícitamente project-shared, con evidencia completa, previews por defecto, confirmación explícita para mutar, Git seguro sin shell interpolation y marcadores personales fuera del árbol versionable.

## Contexto y motivación

ACC-048 verificó que `pull --increment` era un argumento sin uso; `sync` podía exportar visibilidad ausente, perder `rationale`/`source`, revisar secretos solo en `text` y hacer add/commit/push automáticamente. El marcador `.engram/.imported` era local dentro del árbol compartido y el import ocultaba chunks corruptos o parcialmente fallidos.

## Alcance

### Incluido

- Retirar `--increment` de `knowledge pull`; importar todos los chunks pendientes al confirmar.
- Hacer preview por defecto para sync/pull; exigir `--confirm` para escribir, importar o ejecutar Git; exigir `--push` explícito adicional para enviar remoto.
- Usar APIs Git con argumentos, sin interpolación de shell.
- Exportar solo `visibility: project-shared`, preservar todos los datos estables de `MemoryEntry`, incluidos `rationale` y `source`, y reportar exclusiones privadas/sin visibilidad/secretos.
- Inspeccionar secretos en todo campo textual serializado; validar schema de manifest/chunk/memoria; usar escrituras atómicas e idempotentes.
- Mover los markers de chunks importados a `.axiom-state/<projectKey>/` mediante helpers canónicos, migrando el archivo legacy sin versionarlo.
- Marcar un chunk importado únicamente tras guardar todas sus entradas válidas; hacer visibles y reintentables corrupción y fallos parciales.
- Añadir pruebas focalizadas y documentación operativa de runtime.

### Excluido

- Cambiar el backend JSON local o Engram, la topología Git del usuario o hacer commits/push desde esta sesión.
- Compartir entradas private/ausentes de visibilidad, secretos o datos fuera del proyecto.
- Añadir sync externo, MCP nuevo o dependencias.

## Documentos del incremento

Los requisitos de seguridad, contratos, criterios y ausencia de UI se distribuyen en los documentos 01–04.

## Dudas abiertas

La implementación elige `--confirm` como confirmación explícita consistente con el resto de Axiom; `--dry-run` se conserva como alias de preview cuando sea compatible.

## Decisiones funcionales cerradas

1. `pull` opera sobre todos los chunks y no acepta filtro de incremento.
2. Solo `project-shared` se exporta; privado o ausente nunca es compartible.
3. La ausencia de confirmación nunca escribe, importa, ejecuta Git ni empuja.
4. Un fallo parcial permanece visible y reintentable; nunca se marca éxito completo.

## Consolidación en la spec general

Al cierre, actualizar los contratos estables de conocimiento/memoria en specs 03/05/06 y contexto de integraciones, sin copiar la historia de implementación.

## Estrategia E2E

Pruebas focalizadas de sync/pull con repos Git temporales/fakes para preview, confirmación, push, privacidad, secretos, schemas, atomicidad, fallos parciales, idempotencia y dos miembros; build raíz y revisión independiente.

## Trazabilidad y fuentes

ACC-048 del plan R-11; `apps/cli/src/commands/{knowledge,knowledge-sync}.ts`, `@axiom/memory`, helpers project-scoped, documentación CLI y pruebas.

## Estado de validación humana

Implementado y revisado independientemente. La revisión final validó el contrato de preview/confirmación/push, privacidad opt-in, schema, marker project-scoped, migración legacy recuperable y recuperación de fallos de `git add`/`git commit` sin usar shell.

- `npx vitest run apps/cli/tests/knowledge-sync.test.ts --reporter=dot`: PASS — 18 pruebas.
- `npm run build`: PASS.
- `git diff --check`: PASS; solo avisos CRLF de cambios preexistentes fuera de este incremento.
- Receipt Core de verify: `60c349e6dae8cb06b9646f751d2936048c5506de5996d1a15c52f6b9d16b2a40`.

La consolidación de los claims estables en `specs/03`, `05`, `06` y `context/integrations/` se ejecutará una única vez al cerrar el batch completo R-11, después de ACC-049, para respetar la reconciliación final requerida por el lote.