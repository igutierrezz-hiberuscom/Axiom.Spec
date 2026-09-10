# baseline launcher execution mode and TUI retirement parity

> **Código**: BUG-20260910-baseline-launcher-execution-mode-parity
> **Estado**: Reportado
> **Fecha de reporte**: 2026-09-10
> **Severidad**: (pendiente de completar)
> **Prioridad**: (pendiente de completar)

## Resumen del defecto

La proyección de comandos del Launcher y la paridad posterior a la retirada de
la TUI no conservan el contrato de `executionMode`: el flag `--worktree` se
emite o se omite de forma incorrecta y el flujo de wizard no se ejecuta.

## Contexto conocido

Fallaron `launcher-execution-mode.test.ts` y `tui-retirement-launcher.test.ts`.
Las aserciones muestran arrays de argv distintos y un resultado
`executed: false` donde el contrato espera ejecución.

## Clasificación funcional

Paridad de launcher/CLI tras la retirada de TUI; no es ownership del helper
externo R13-4.

## Comportamiento actual

La selección de modo y la ruta de confirmación no comparten la misma
normalización entre preview y ejecución.

## Comportamiento esperado

`worktree` añade exactamente el flag esperado, el modo no seleccionado no lo
añade, y el wizard headless conserva options, preview y confirmación con un
resultado ejecutado.

## Reproducción

(pendiente de completar)

### Precondiciones

(pendiente de completar)

### Pasos

(pendiente de completar)

### Resultado observado

Fallos de igualdad de argv y ejecución falsa en las suites de Launcher/TUI.

## Superficie de regresión

`packages/launcher`, builders de comandos y endpoints/flows de workspace wizard.

## Estructura mínima del bug

No reintroducir una TUI ni lógica de negocio duplicada; corregir la proyección
headless compartida.

## Material adicional

Log: `%TEMP%\\axiom-r13-baseline-20260910.log`.

Suites: `launcher-execution-mode.test.ts`, `tui-retirement-launcher.test.ts`.

## Trazabilidad y fuentes

Baseline global del 2026-09-10, fuera del diff de R13-3.

## Estado de validación humana

Pendiente de reproducción con los tres modos y validación de que no reaparece
ownership de TUI.
