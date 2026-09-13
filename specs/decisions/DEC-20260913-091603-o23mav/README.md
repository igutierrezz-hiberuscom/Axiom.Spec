<!-- AXIOM:MIGRATED — source: Axiom.Spec/decisions/0019-operator-control-plane-runtime.md; migrated for INC-20260912-r15-canonical-roots-dedup via Core. -->

# Incremento 0019 — Operator Control Plane Runtime

> **Estado**: closed (2026-06-30)

Contenido preservado del documento legacy `decisions/0019-operator-control-plane-runtime.md`.

## Resumen ejecutivo

El runtime MVP de Axiom obtiene una superficie operativa comparable a GentleAI, manteniendo configuración declarativa versionada en `axiom.config/*.yaml`, estado mutable project-scoped en `.axiom-state/<project>/` y proyección por adapter target. `opencode` es el primer target completo; el resto usa fallback explícito.

## Comandos y paquetes

Se añadieron `axiom upgrade`, `axiom tui`, `axiom model show/set/unset/reset/validate`, `axiom components list/show/install/uninstall/restore` y `axiom skills list/refresh/drift`, junto con los paquetes de versioning, TUI, CLI commands, model routing, components y skills.

## Support por target

`opencode` soporta multi-mode; `claude-code` single-mode; `copilot-vscode` y otros targets usan fallback a `medium` con razón visible.

## Métricas

**681/681 tests verde**, **40/0 doctor estable** y **`tsc -b` exit 0**.
