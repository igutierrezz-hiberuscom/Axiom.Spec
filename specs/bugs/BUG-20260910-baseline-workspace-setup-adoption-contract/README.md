# baseline workspace setup adoption and scaffolding contract

> **Código**: BUG-20260910-baseline-workspace-setup-adoption-contract
> **Estado**: Reportado
> **Fecha de reporte**: 2026-09-10
> **Severidad**: (pendiente de completar)
> **Prioridad**: (pendiente de completar)

## Resumen del defecto

La baseline global falla en setup/adoption porque los fixtures y consumidores de
workspace no convergen con el contrato actual de `WorkspaceSetupSpec` y del
scaffold multi-repo. Se observaron fallos en `init.test.ts`,
`inc-20260727-adoption-config-scaffolding.test.ts`, `workspace-adopt.test.ts`,
`workspace-incremental.test.ts` y el E2E de schema v2.

## Contexto conocido

La ejecución `npm test` del 2026-09-10 terminó con 28 archivos y 65 tests
fallidos. El error dominante fue `WorkspaceSetupSpec.repos debe incluir
EXACTAMENTE un repo kind='axiom' (encontrados: 0)`; `init` también observó
`legacy` donde los tests esperan `sdd`. Esta clasificación es independiente de
R13-3 y no se considera preexistente sin reproducirla después del arreglo.

## Clasificación funcional

Contrato de setup/adoption/scaffolding y normalización de topología; no es un
defecto del actualizador transaccional.

## Comportamiento actual

Los inputs de setup que representan un workspace multi-repo son rechazados o
normalizados hacia el modo legacy. Los flujos de adoption e incremental no
materializan consistentemente el repo `axiom`, archivos derivados y estado v2.

## Comportamiento esperado

El parser acepta el shape canónico con exactamente un repo `kind: axiom`,
`init` emite el modo/schema vigente, y setup/adoption/incremental son
best-effort, no-clobber e idempotentes con topology, registry y scaffolding
convergentes.

## Reproducción

(pendiente de completar)

### Precondiciones

(pendiente de completar)

### Pasos

(pendiente de completar)

### Resultado observado

Fallos de aserción por `kind` incorrecto, repo `axiom` ausente, scaffolding no
creado y timeout en la suite de operaciones incrementales.

## Superficie de regresión

`Axiom/apps/cli/src/commands/workspace.ts`, `workspace-incremental.ts`,
`init.ts`, validadores de topology y los handlers de adoption/setup.

## Estructura mínima del bug

No mezclar este arreglo con `packages/core/src/self-update/**` ni con los gates
AC-078/079. La prueba debe usar un fixture de workspace canónico y comprobar
que una segunda ejecución no añade archivos ni entradas duplicadas.

## Material adicional

Log: `%TEMP%\\axiom-r13-baseline-20260910.log`.

Suites observadas: `inc-20260727-adoption-config-scaffolding.test.ts`,
`init.test.ts`, `workspace-adopt.test.ts`, `workspace-incremental.test.ts` y
`schemaversion2-e2e.test.ts`.

## Trazabilidad y fuentes

Baseline completa de `Axiom` ejecutada el 2026-09-10; no se usa como evidencia
de GO para R13-3.

## Estado de validación humana

Pendiente: reproducir los casos aislados, aplicar la corrección fuera del
alcance R13-3 y ejecutar las suites de setup/adoption más la baseline completa.
