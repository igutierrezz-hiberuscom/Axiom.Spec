# R-10 ACC-038 lifecycle documentation alignment

> **Código**: INC-20260817-r10-acc038-lifecycle-docs
> **Estado**: En especificación
> **Fecha de creación**: 2026-08-17
> **Tipo de cambio**: documentar

## Resumen

Alinear dos afirmaciones documentales del lifecycle con el comportamiento ya ejecutable, sin cambiar la lógica funcional.

## Contexto y motivación

`axiom upgrade` puede inicializar `managed-state.json` en su primer uso. Además, los gates del orchestrator sí pueden consultar el filesystem para comprobar precondiciones. Ambos hechos se presentaban de forma contradictoria en superficies activas.

## Alcance

### Incluido

- Corregir el manual de `axiom upgrade` para describir la inicialización de `managed-state.json`.
- Corregir el comentario de lifecycle de gates para reconocer comprobaciones locales de filesystem.
- Actualizar o añadir únicamente pruebas documentales directamente asociadas.

### Excluido

- Cambiar el algoritmo de upgrade, los archivos que persiste o los gates existentes.
- Cambiar políticas de I/O, estado o lifecycle.

## Decisiones funcionales cerradas

- El primer `axiom upgrade` puede inicializar `managed-state.json`.
- Los gates de orchestrator sí pueden leer filesystem.
- No hay cambio funcional de upgrade ni gates.

## Consolidación en la spec general

No requiere cambio en `specs/00..08`: `03_Modelo_Operativo_y_Datos.md` ya declara que upgrade genera `managed-state.json` y no conserva el claim de preexistencia. La reconciliación de referencias lifecycle próximas se tratará en ACC-044.

## Trazabilidad y fuentes

- Plan R-10, ACC-038.
- `packages/cli-commands/src/commands/upgrade.ts`, `docs/cli/upgrade.md`.
- `packages/orchestrator/src/gates.ts`, `state-machine.ts` y `gates.test.ts`.

## Resultado y validación

Implementado: el manual declara la inicialización posible en la primera ejecución real y distingue el dry-run; el comentario de gates declara lecturas locales read-only de filesystem. No hubo cambios de lógica funcional.

Validación independiente: `git diff --check && npx vitest run packages/orchestrator/tests/gates.test.ts packages/versioning/tests/upgrade.test.ts` pasó con 2 archivos y 19 tests. Review independiente: conforme en alcance; señaló únicamente el `tsconfig.tsbuildinfo` generado fuera de alcance y divergencias próximas reservadas para ACC-044.

## Estado de validación humana

No requiere decisión humana adicional: las dos correcciones están cerradas por el lote.
