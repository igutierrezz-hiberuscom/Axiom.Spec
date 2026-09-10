# baseline provider selection and worktree project state

> **Código**: BUG-20260910-baseline-provider-worktree-state
> **Estado**: Reportado
> **Fecha de reporte**: 2026-09-10
> **Severidad**: (pendiente de completar)
> **Prioridad**: (pendiente de completar)

## Resumen del defecto

La lectura de estado de workspace usada por provider selection y provisioning
de worktree no acepta de forma consistente el schema vigente: rechaza
`workspace.json#projectId` y pierde providers como `cmm` al resolver otro
`projectKey`.

## Contexto conocido

Fallaron cinco casos de `packages/doctor/tests/provider-selection.test.ts` con
`WorkspaceStateError: workspace.json.projectId debe ser string` y un caso de
`workspace-worktree-provision.test.ts` donde el resultado no incluye `cmm`.

## Clasificación funcional

Schema de estado/provider y resolución project-scoped; separado de setup HTTP y
del actualizador global.

## Comportamiento actual

El lector valida un shape distinto al que escriben los fixtures/registry v2 y
provisioning consulta el proyecto equivocado cuando el nombre v2 difiere.

## Comportamiento esperado

Los fixtures con providers vacío, code-intel o Engram cargan y producen pass/warn
sin `fail`; provisioning consulta el `projectKey` de la ejecución y conserva
todos los providers habilitados.

## Reproducción

(pendiente de completar)

### Precondiciones

(pendiente de completar)

### Pasos

(pendiente de completar)

### Resultado observado

Excepción `invalid-type` en doctor y lista de providers vacía en provisioning.

## Superficie de regresión

`packages/providers`, `packages/doctor` y el resolver de project/worktree
provisioning.

## Estructura mínima del bug

Corregir la lectura/normalización del schema y cubrir aislamiento por
`projectKey`; no tocar self-update.

## Material adicional

Log: `%TEMP%\\axiom-r13-baseline-20260910.log`.

Suites: `provider-selection.test.ts`, `workspace-worktree-provision.test.ts`.

## Trazabilidad y fuentes

Baseline global del 2026-09-10, clasificada como causa independiente.

## Estado de validación humana

Pendiente: reproducir los cinco shapes de provider y el caso de projectKey
distinto antes de cambiar el lector.
