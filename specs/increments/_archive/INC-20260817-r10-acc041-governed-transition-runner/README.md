# R-10 ACC-041 governed transition runner

> **Código**: INC-20260817-r10-acc041-governed-transition-runner
> **Estado**: Archivado mediante Core
> **Fecha de creación**: 2026-08-17
> **Tipo de cambio**: modificar

## Resumen

Crear un ejecutor común de transiciones gobernadas, basado en el resolvedor de ACC-045, para CLI, launcher y MCP.

## Contexto y motivación

Las rutas actuales persisten estado, metadata, efectos, QA, confirmación y archive de forma diferente. `requiresApproval` es declarativo pero no se aplica uniformemente; archive/integrate puede dejar combinaciones divergentes.

## Alcance

### Incluido

- Runner único que centraliza legalidad, preview, confirmación declarada, efectos locales, gate QA, persistencia y receipts aplicables.
- Coordinación transaccional/compensable de archive/integrate: workflow-state, metadata, `integration.status`, move `_archive` y recuperación.
- CLI, launcher y MCP delegan al runner; máquina de estados permanece pura.
- `requiresApproval` se exige coherentemente; `--force`/`--no-verify` no la omiten.
- Retirar el efecto activo `tracker: close-external-work-item` sin retirar tracker general.

### Excluido

- Simplificar el cierre de plan (ACC-042) y contrato QA definitivo (ACC-043), aunque consumen el runner.
- Inventar dispatch externo de tracker, automatización externa o estados nuevos.

## Decisiones funcionales cerradas

I/O vive fuera de la máquina pura. Archive/integrate se ejecuta como operación gobernada única y recuperable. Approval es confirmación explícita de mutación, no una flag que `force` pueda saltar.

## Consolidación en la spec general

Al final del lote, describir un runner canónico de transición y eliminar las rutas paralelas como contrato activo.

## Trazabilidad y fuentes

Plan R-10 ACC-041; ACC-045; workflow state/effects; CLI, launcher, MCP, integrate, metadata/receipts.

## Estado de validación humana

No quedan decisiones abiertas: todos los comportamientos de confirmación y tracker están definidos por el lote.
## Evidencia actual de cierre

La validación previa queda sustituida por evidencia actual: focal R-10 `21/21` archivos y `238/238` tests; build y suite global `330/330` archivos y `3289/3289` tests; doctor PASS `45/60` (0 fallos) y readiness PASS. La revisión final debe contrastar el runner común y la compensación contra este receipt Core de verify: SHA-256 `160b55f8cc525b2a0221debcae4527a50cfeb256c14782f39363def754ad8425`.
## Cierre Core

El Core archivó este incremento en `specs/increments/_archive/` mediante la cadena legal `specify → plan → plan-approve → verify → archive`; el receipt `2026-08-18T14-09-29.983Z-increment-archive-success.json` lo confirma. La cifra `3289/3289` anterior es una fotografía previa: la validación global final es `330/330` archivos y `3294/3294` tests, con build, doctor y readiness PASS.