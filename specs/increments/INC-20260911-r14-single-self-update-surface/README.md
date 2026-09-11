# r14 single self update surface

> **Código**: INC-20260911-r14-single-self-update-surface
> **Estado**: En Borrador
> **Fecha de creación**: 2026-09-11
> **Tipo de cambio**: consolidación y eliminación de código muerto
> **Acción de origen**: `ACC-083`
> **Plan**: `PLAN-INC-20260911-r14-operator-control-surfaces` (posición 5 de 8)

## Resumen

Dejar una sola superficie de self-update alcanzable desde la CLI y eliminar la otra por completo, separando «instalar» de «actualizar» si el análisis concluye que ambos verbos son necesarios.

## Contexto y motivación

Verificado el 2026-09-11:

- `apps/cli/src/index.ts` solo invoca `registerSelfUpdateContract(program)`.
- `apps/cli/src/commands/self-update.ts` (27,7 KB) exporta `registerSelfUpdate`, pero ningún módulo de producción lo importa. Su único consumidor es `apps/cli/tests/self-update.test.ts`, con 16 pruebas que pasan.
- Ambas generaciones convivieron desde el commit que introdujo el motor transaccional (`feat(cli): add transactional self-update engine and dedicated spec-repo workspace support`, 2026-09-10): la antigua no se retiró al aterrizar la nueva.
- `docs/cli/self-update.md` describe únicamente el contrato nuevo (`status`/`check`/`plan`/`apply`/`recover`), así que la implementación antigua tampoco está documentada como vigente.
- La generación antigua se apoya en `scripts/install-global.mjs`, `runInstallGlobal`, backup/restore del shim y `node --test scripts/install-global.test.mjs`.

Es código muerto sostenido por sus propias pruebas, y una segunda ruta de actualización que puede confundir a quien lea el árbol.

## Alcance

### Incluido

- Ejecución de la decisión `D-01`: qué generación sobrevive y qué se borra.
- Eliminación completa de la descartada, incluidos tipos, helpers, pruebas, referencias y documentación asociada.
- Si `D-01` concluye que la instalación inicial del shim sigue siendo un camino soportado, separarla explícitamente del verbo «actualizar», con nombre y superficie que no se confundan.
- Migración de los escenarios que hoy solo cubren las pruebas antiguas y que deban conservarse.

### Excluido

- Reabrir el contrato transaccional cerrado por `ACC-077`..`ACC-080`: este incremento lo consume.
- Cambiar el canal de entrega o publicar el CLI como package npm.

## Dudas abiertas

`D-01` del plan, que debe cerrarse antes de empezar: si la instalación inicial es un camino soportado, si queda como comando propio o como paso interno del contrato, y qué escenarios de las 16 pruebas antiguas se migran antes de borrar.

## Decisiones funcionales cerradas

- No se mantienen dos rutas de actualización.
- Ninguna función de producción debe quedar sostenida únicamente por sus propias pruebas.

## Consolidación en la spec general

Al cierre, dejar una sola afirmación activa sobre cómo se actualiza el CLI global y, si procede, una afirmación separada sobre cómo se instala por primera vez.

## Estrategia E2E

Regresión de que la ruta borrada ya no es alcanzable; conservación o migración demostrada de la cobertura de los escenarios que sobreviven; verificación herméticamente aislada, sin tocar el home ni el PATH reales.

## Trazabilidad y fuentes

Acción `ACC-083` y sesión R-14 del 2026-09-11 en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md`.

## Estado de validación humana

Pendiente. Bloqueado hasta que `INC-20260911-r14-workflow-instance-selection` permita seleccionar instancia y hasta que `D-01` esté cerrada.
