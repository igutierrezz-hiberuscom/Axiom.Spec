<!-- AXIOM:MIGRATED — source: Axiom.Spec/decisions/0026-integration-hardening-and-target-parity.md; migrated for INC-20260912-r15-canonical-roots-dedup via Core. -->

# Incremento 0026 — Integration Hardening and Target Parity

> **Estado**: archived 2026-06-30

Contenido preservado del documento legacy `decisions/0026-integration-hardening-and-target-parity.md`.

## Resumen ejecutivo

Se cerró la paridad operativa de integraciones project-scoped, se resolvió el flake de `tool-routing`, se integró routing real con el adapter de opencode, se hizo idempotente `axiom skills apply` y se añadieron integraciones TUI para `model validate` y `components show`.

## Lotes

- `routeTool` y emitters derivan `emittedAt` de `requestedAt`.
- `axiom skills apply` materializa `skills-pending.json` sin tocar lockfile.
- Opencode consume `.opencode/model-routing.json` tolerando ausencia y malformación.
- TUI añade `model-validate` y `components-show`.
- `SUPPORT_MATRIX` cubre cinco targets MVP, incluidos `antigravity` y `visual-studio-2026`.

## Métricas

**963/963 tests verde**, **`tsc -b` verde** y GATE 0026 honrada.
