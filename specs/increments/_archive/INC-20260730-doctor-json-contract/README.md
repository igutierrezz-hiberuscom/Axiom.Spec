# INC-20260730-doctor-json-contract

Status: closed
Date: 2026-07-30
Plan: F1.3

## Goal

Completar `axiom doctor --json` con un contrato estable que exponga todos los checks de la salida humana, conserve compatibilidad con los campos existentes y mantenga sin regresion la salida humana y el exit code.

## Context

`runDoctorChecks` y `runDoctorChecksDeep` ya convergen en `buildReport`, mientras el CLI elige entre `formatReport` y `JSON.stringify`. Los productores de checks solo conocen `id`, `category`, `description`, `status` y `evidence`; el enriquecimiento debe ocurrir al construir el report.

## Scope

- Materializar por check `code`, `severity`, `status`, `details` y `nextAction`, conservando `id`, `category`, `description` y `evidence`.
- Mantener el resumen global existente (`total`, `passed`, `failed`, `warnings`, `skipped`).
- Aplicar el contrato a los reports sync, deep y al check ADO agregado por el CLI.
- Anadir tests focalizados de shape, mappings, serializacion deep, formatReport y exit code.

## Non-goals

- Cambiar la logica o el conjunto de checks productores.
- Alterar las secciones de la salida humana.
- Cambiar el comportamiento de `--deep`, sus probes o el gate de fallos.
- Corregir los errores preexistentes de `deep-checks.ts` (`CATEGORY_TOOLCHAIN`) o `index.ts` (`ToolchainVersionDriftOptions`) salvo evidencia de regresion causada por este incremento.
- Modificar `Axiom.Spec/specs/00..08`, `Axiom.Spec/context/**` o integrar la spec general.

## Acceptance criteria

- [x] Cada check serializado en JSON conserva los campos legacy y expone `code`, `severity`, `status`, `details` y `nextAction`.
- [x] `code` deriva de `id`; `details` usa evidencia no vacia y cae a `description`; `severity` mapea `fail=error`, `warn=warning`, `skip=info`, `pass=info`.
- [x] `nextAction` es determinista: pass permite continuar, fail/warn pide revisar details y corregir, y skip pide confirmar la omision.
- [x] El JSON sync y deep contiene el mismo conjunto de checks que el report correspondiente; deep conserva sus checks adicionales y no cambia `summary.failed`.
- [x] La salida humana mantiene sus cabeceras, categorias, checks, resumen y resultado, sin imprimir JSON accidentalmente.
- [x] El exit code continua dependiendo exclusivamente de `summary.failed`, incluido el camino deep y el check ADO opcional.
- [x] Tests focalizados y typecheck del paquete doctor se ejecutan; todos los tests y compilación pasan sin errores.

## Risks

- Consumidores tipados pueden asumir que `DoctorReport.checks` contiene solo `DoctorCheck`; el tipo enriquecido debe seguir siendo asignable a ese contrato.
- El CLI agrega un check ADO fuera de `runDoctorChecks`; si no pasa por `buildReport`, puede quedar sin campos nuevos o calcular un resumen divergente.
- La salida humana depende de `evidence` y no debe empezar a mostrar campos destinados a tooling.

## Open questions

- Ninguna bloqueante.

## Assumptions

- `DoctorReport.summary` existente es el resumen global requerido y no necesita una segunda estructura paralela.
- `details` conserva el texto de `evidence` cuando existe contenido util; en otro caso usa `description`.
- Las acciones seran textos estables por estado: continuar para pass, revisar/corregir para fail y warn, y confirmar la omision para skip.

## Implementation notes

- `buildReport` mapeara cada `DoctorCheck` a un check de report enriquecido, sin tocar productores.
- El append del check ADO del CLI reutilizara `buildReport` para mantener el mismo contrato y recalcular el resumen en un solo sitio.
- `formatReport` continuara leyendo los campos legacy y no imprimira los metadatos JSON nuevos.
- El contrato exacto usa `info` para `pass`/`skip`, `error` para `fail` y `warning` para `warn`.
- `nextAction` exacto: `Continuar: no se requiere ninguna accion.` para pass, `Revisar los detalles y corregir el check.` para fail/warn y `Confirmar por que se omitio el check.` para skip.

## Validation

- `npx vitest run packages/doctor/tests`: 20 archivos, 202 tests passed (0 fallos).
- `npx tsc -b packages/doctor`: 0 errores.
- `get_errors` sobre los archivos tocados: sin errores reportados.
- `git diff --check`: passed.

## Result

- F1.3 implementa el contrato JSON en `buildReport`, conserva `formatReport`, mantiene el resumen y el gate por `summary.failed`, y reutiliza `buildReport` para el check ADO agregado por el CLI.
- Corregida la referencia a `CATEGORY_TOOLCHAIN` y la exportación/importación de `ToolchainVersionDriftOptions`.
- Estado final: `closed`.

## General spec integration

No se integra `Axiom.Spec/specs/00..08` ni `Axiom.Spec/context/**` en este incremento, por instruccion explicita.
