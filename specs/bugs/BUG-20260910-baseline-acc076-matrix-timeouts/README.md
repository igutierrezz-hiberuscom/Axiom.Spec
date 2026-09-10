# baseline R13 ACC-076 matrix timeout and evidence

> **Código**: BUG-20260910-baseline-acc076-matrix-timeouts
> **Estado**: Reportado
> **Fecha de reporte**: 2026-09-10
> **Severidad**: (pendiente de completar)
> **Prioridad**: (pendiente de completar)

## Resumen del defecto

La matriz ejecutable ACC-076 pierde dos casos por timeout aunque no tenga
aserciones FAIL: el resumen termina con `PASS: 33, TIMEOUT: 2`, incumpliendo el
contrato de evidencia completa.

## Contexto conocido

Fallaron dos casos de `apps/cli/tests/r13-acc-076-matrix.test.ts`, incluidos
`ACC-072-06` y `ACC-072-09`, con timeout de 15000 ms. El resumen esperaba 35
casos PASS y obtuvo dos TIMEOUT.

## Clasificación funcional

Determinismo/aislamiento de la matriz R13 ACC-076; no guarda relación con
self-update.

## Comportamiento actual

Los casos que consultan catálogos/roles/Git/ADO no terminan dentro del límite
de la matriz y su evidencia no se serializa como PASS completo.

## Comportamiento esperado

Cada caso finaliza en tiempo acotado con resultado PASS o FAIL explícito, sin
promesas colgadas ni dependencia de estado global entre casos; el resumen es
machine-readable y cuenta 35/35.

## Reproducción

(pendiente de completar)

### Precondiciones

(pendiente de completar)

### Pasos

(pendiente de completar)

### Resultado observado

`ACC-076 case timeout after 15000ms`; resumen `{ PASS: 33, FAIL: 0, TIMEOUT: 2 }`.

## Superficie de regresión

Harness y fixtures de `r13-acc-076-matrix.test.ts`, catálogos visibles y checks
de roles/Git/ADO.

## Estructura mínima del bug

Aislar cada caso, liberar timers/servidores y conservar la evidencia de timeout;
no convertir TIMEOUT en PASS.

## Material adicional

Log: `%TEMP%\\axiom-r13-baseline-20260910.log`.

Suite: `Axiom/apps/cli/tests/r13-acc-076-matrix.test.ts`.

## Trazabilidad y fuentes

Baseline global del 2026-09-10; este bug es un gate de calidad separado de
AC-078/079.

## Estado de validación humana

Pendiente: reproducir los dos casos con timeout instrumentado y ejecutar la
matriz completa sin ampliar su timeout para ocultar fugas.
