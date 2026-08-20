# R-10 ACC-042 simplify plan approval

> **Código**: INC-20260817-r10-acc042-simplify-plan-approval
> **Estado**: Archivado mediante Core
> **Fecha de creación**: 2026-08-17
> **Tipo de cambio**: modificar

## Resumen

Hacer que la única operación canónica de cierre de un plan valide requisitos, exija confirmación gobernada y transicione `draft → plan-approved`.

## Alcance

Incluye validación mínima de plan, `axiom-plan approve`, launcher preview/confirm y gate de inicio de roles. Excluye estados nuevos, duplicación de validación en UI y cambios QA/archive.

## Decisiones funcionales cerradas

Un plan permanece `draft` hasta su aprobación; `plan-approved` es el único estado habilitante de roles. No existen estados independientes cerrado/validado/aprobado.

## Criterios resumidos

- La aprobación valida requisitos definidos y requiere confirmación explícita a través del runner ACC-041.
- CLI y launcher delegan la misma operación y preview no escribe.
- Roles solo arrancan con plan `plan-approved`.
- No se introducen estados ni aliases nuevos.

## Consolidación en la spec general

Al final del lote, describir draft → plan-approved como cierre canónico único.

## Estado de validación humana

No hay decisiones abiertas.
## Resultado de cierre

La única aprobación gobernada es `draft → plan-approved`: el runner valida la metadata, el preview no escribe y sólo la confirmación explícita muta. CLI y launcher delegan el mismo límite; role start exige state y metadata de un plan aprobado. Validación actual: focal R-10 `21/21` y `238/238`, build, suite global `330/330` y `3289/3289`, doctor PASS `45/60` (0 fallos) y readiness PASS. Receipt Core de verify: SHA-256 `b99c0b3504a7acebd6244693d9910df912e7c9ec57cfde91fb2104298f1e9ddf`.
## Cierre Core

El Core archivó este incremento en `specs/increments/_archive/` mediante la cadena legal `specify → plan → plan-approve → verify → archive`; el receipt `2026-08-18T14-09-43.192Z-increment-archive-success.json` lo confirma. La cifra `3289/3289` anterior es una fotografía previa: la validación global final es `330/330` archivos y `3294/3294` tests, con build, doctor y readiness PASS.