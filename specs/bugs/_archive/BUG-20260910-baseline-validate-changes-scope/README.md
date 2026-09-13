# baseline validate changes repo affinity and scope

> **Código**: BUG-20260910-baseline-validate-changes-scope
> **Estado**: Reportado
> **Fecha de reporte**: 2026-09-10
> **Severidad**: (pendiente de completar)
> **Prioridad**: (pendiente de completar)

## Resumen del defecto

`validate changes` no reconoce correctamente un cambio dentro del repo asignado
al role de un plan ni clasifica de forma estable un repo no declarado. La
baseline devuelve exit 1 en ambos casos.

## Contexto conocido

Fallaron los dos casos de role-split en `apps/cli/tests/validate-changes.test.ts`:
el cambio dentro de scope no obtiene exit 0 y el cambio fuera de scope no
produce el diagnóstico `undeclared-repo` esperado.

## Clasificación funcional

Resolución de allowedWriteScope y afinidad repo/plan; separado de los wrappers
que solo reportan `repo-affinity`.

## Comportamiento actual

La validación compara el repo equivocado o no carga la asignación role-split
antes de calcular el scope permitido.

## Comportamiento esperado

Un cambio dentro del repo asignado pasa; uno en un repo no declarado falla con
`undeclared-repo`, sin mutar metadata ni ampliar el scope implícitamente.

## Reproducción

(pendiente de completar)

### Precondiciones

(pendiente de completar)

### Pasos

(pendiente de completar)

### Resultado observado

Ambos escenarios terminan con exit 1 y aserciones de resultado ausentes o
incorrectas.

## Superficie de regresión

`apps/cli/src/commands/validate-changes.ts` y los helpers de workflow/plan que
resuelven repos y allowedWriteScope.

## Estructura mínima del bug

Cubrir la resolución desde un plan role-split y conservar la prohibición de
escribir fuera del scope.

## Material adicional

Log: `%TEMP%\\axiom-r13-baseline-20260910.log`.

Suite: `Axiom/apps/cli/tests/validate-changes.test.ts`.

## Trazabilidad y fuentes

Baseline global del 2026-09-10; no es evidencia contra R13-3.

## Estado de validación humana

Pendiente: reproducir ambos escenarios con repos temporales y validar salida,
exit code y ausencia de escrituras.
