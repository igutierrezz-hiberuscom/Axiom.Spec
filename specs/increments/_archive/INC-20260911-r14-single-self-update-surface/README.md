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

Ninguna. Cerradas en D-01 (`DEC-20260911-223522-6ox3vz`):
- Se conserva exclusivamente `self-update-contract.ts`.
- Se elimina `self-update.ts` y sus 16 tests tras asegurar cobertura en `self-update-contract.test.ts`.
- `scripts/install-global.mjs` se mantiene como instalador de bootstrap fuera del CLI.

## Decisiones funcionales cerradas

- No se mantienen dos rutas de actualización.
- Ninguna función de producción queda sostenida únicamente por sus propias pruebas.
- La separación entre "instalar shim por primera vez" y "actualizar producto" queda consolidada.

## Consolidación en la spec general

Al cierre, dejar una sola afirmación activa sobre cómo se actualiza el CLI global y una afirmación separada sobre cómo se instala por primera vez.

## Estrategia E2E

Regresión de que la ruta borrada ya no es alcanzable; conservación de la cobertura de rechazo de flags legacy en `self-update-contract.test.ts`; `scripts/install-global.test.mjs` 18/18 PASS; `npm run build`, doctor, readiness y `git diff --check`.

## Trazabilidad y fuentes

Acción `ACC-083`, decisión `DEC-20260911-223522-6ox3vz` y sesión R-14 del 2026-09-11 en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md`.

## Estado de validación humana

Validado. Criterios AC-083-01 a AC-083-05 verificados:
- `apps/cli/src/commands/self-update.ts` y `apps/cli/tests/self-update.test.ts` eliminados físicamente y desversionados de Git.
- `apps/cli/tests/self-update-contract.test.ts` enriquecido con prueba de rechazo de flags legacy (`--check`, `--apply`, `--target-version`, `--dry-run`) y aserción de ausencia del módulo legacy (15/15 PASS).
- `scripts/install-global.mjs` y `scripts/install-global.test.mjs` preservados e intactos (18/18 PASS en `node:test`).
- `self-update-status.test.ts` pasando 28/28 tests.
- `npm run build` limpio, `npm run doctor` PASS (48/61 OK, 0 fallos), readiness PASS, `git diff --check` limpio.
