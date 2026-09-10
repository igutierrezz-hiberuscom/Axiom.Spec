# baseline workflow command guards infer state and ids

> **Código**: BUG-20260910-baseline-workflow-command-guards
> **Estado**: Reportado
> **Fecha de reporte**: 2026-09-10
> **Severidad**: (pendiente de completar)
> **Prioridad**: (pendiente de completar)

## Resumen del defecto

Varios comandos de workflow rechazan antes de evaluar su guardia funcional
porque exigen `--id` explícito aunque los tests y el contrato permitan inferir
el artefacto desde el estado persistido. Esto oculta `repo-affinity`, `evidence`
y el resultado de functional verify.

## Contexto conocido

La baseline reportó fallos en `functional-verify.test.ts`,
`qa-archive-gate.test.ts`, `repo-affinity.test.ts` y `scaffold.test.ts`, con
mensajes como `--id es obligatorio` en lugar de la guardia esperada.

## Clasificación funcional

Orden y resolución de argumentos en wrappers CLI gobernados; no pertenece al
runner de self-update.

## Comportamiento actual

`axiom-increment`, `axiom-plan` y comandos relacionados terminan con usage
error antes de cargar `workflow-state.json` o comprobar la afinidad del repo.

## Comportamiento esperado

Cuando el estado identifica un único artefacto, el wrapper puede inferir el ID;
cuando no puede hacerlo, devuelve el error semántico correspondiente. Las
guardias de affinity, QA y functional verify deben ejecutarse en el orden del
contrato y no quedar tapadas por parsing prematuro.

## Reproducción

(pendiente de completar)

### Precondiciones

(pendiente de completar)

### Pasos

(pendiente de completar)

### Resultado observado

Los tests esperan `repo-affinity`, `evidence=pending` o transición bloqueada y
reciben `--id es obligatorio`; varios casos terminan con exit 1.

## Superficie de regresión

`Axiom/apps/cli/src/commands/axiom-increment.ts`, `axiom-plan.ts`, wrappers de
scaffold y validación de transiciones.

## Estructura mínima del bug

Separar la resolución de identidad del artefacto de la validación de la
transición. No modificar el workflow de R13-3 ni sus receipts.

## Material adicional

Log: `%TEMP%\\axiom-r13-baseline-20260910.log`.

Fallos: `functional-verify.test.ts`, `qa-archive-gate.test.ts`,
`repo-affinity.test.ts` y `scaffold.test.ts`.

## Trazabilidad y fuentes

Baseline global del runtime Axiom, 2026-09-10. La clasificación se mantiene
separada del alcance self-update.

## Estado de validación humana

Pendiente: reproducir cada wrapper con y sin ID, verificar que no haya escrituras
antes de la guardia y cerrar por Core solo con las suites afectadas verdes.
