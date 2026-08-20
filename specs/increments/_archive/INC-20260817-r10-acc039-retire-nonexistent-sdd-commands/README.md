# R-10 ACC-039 retire nonexistent public SDD commands

> **Código**: INC-20260817-r10-acc039-retire-nonexistent-sdd-commands
> **Estado**: Archivado mediante Core
> **Fecha de creación**: 2026-08-17
> **Tipo de cambio**: eliminar

## Resumen

Retirar promesas activas de comandos SDD públicos no registrados y dirigir las superficies operativas a comandos reales.

## Contexto y motivación

Catálogos y process surfaces aún pueden materializar `axiom sdd advance`, `axiom plan` y `axiom role start`, aunque la CLI pública usa comandos con guion y `axiom state`.

## Alcance

### Incluido

- Retirar las tres formas inexistentes de catálogos, contratos materializables, process surfaces, pruebas y documentación activa.
- Sustituir por `axiom state`, `axiom-increment`, `axiom-bug`, `axiom-plan` y `axiom-role` según la acción real.
- Actualizar tests de catálogo/materialización y manuales activos.

### Excluido

- Retirar los 19 intent commands internos `not-implemented` de `@axiom/orchestrator`.
- Alterar referencias en decisiones o incrementos archivados.

## Decisiones funcionales cerradas

Las formas públicas reales usan comandos con guion; `axiom sdd advance`, `axiom plan` y `axiom role start` no son contratos operativos.

## Consolidación en la spec general

Actualizar interfaces y flujo SDD solo con la lista de comandos públicos realmente alcanzables; preservar historia en artifacts archivados.

## Trazabilidad y fuentes

Plan R-10 ACC-039; catálogos de agentes, templates/materializadores, process surfaces, documentación CLI y tests.

## Estado de validación humana

Sin preguntas abiertas: no se retiran los intents internos en este lote.
## Resultado de cierre

Se retiraron de contratos activos las tres formas SDD inexistentes y las superficies emitidas usan los entrypoints con guion o `axiom state`. La inspección de `@axiom/orchestrator` confirma que los 19 intent stubs `not-implemented` permanecen internos y sin caller CLI. Validación actual: batería focalizada R-10 `21/21` archivos y `238/238` tests, build y suite global `330/330` archivos y `3289/3289` tests, doctor PASS `45/60` (0 fallos) y readiness PASS. Receipt Core de verify: SHA-256 `552276b5fd6eb0f73ba00dca24f66cf3b2938187553ec3085dd6887aa6e23067`.
## Cierre Core

El Core archivó este incremento en `specs/increments/_archive/` mediante la cadena legal `specify → plan → plan-approve → verify → archive`; el receipt `2026-08-18T14-09-02.475Z-increment-archive-success.json` lo confirma. La cifra `3289/3289` anterior es una fotografía previa: la validación global final es `330/330` archivos y `3294/3294` tests, con build, doctor y readiness PASS.