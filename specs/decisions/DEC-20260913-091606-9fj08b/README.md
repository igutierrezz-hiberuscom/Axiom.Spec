<!-- AXIOM:MIGRATED — source: Axiom.Spec/decisions/0029-memory-recall-and-context-repair.md; migrated for INC-20260912-r15-canonical-roots-dedup via Core. -->

# Incremento 0029 — Memory Recall and Context Repair

> **Estado**: archived 2026-06-30

Contenido preservado del documento legacy `decisions/0029-memory-recall-and-context-repair.md`.

## Resumen ejecutivo

Se incorporó recall útil de memoria en workflows, reparación operativa de MCP y surfaces CLI/TUI de inventory.

## Lotes

- `rankEntries` y `buildRecallResult` aplican match, recencia y boost por kind, con explicación humana por hit.
- `axiom mcp repair --id <mcpId>` valida bindings y escribe estado project-scoped de forma atómica.
- `axiom memory inventory` y `axiom mcp inventory` exponen inventario.
- TUI añade `memory-inventory` y `mcp-inventory`.
- Recall queda opt-in; el wire-up al state machine permanece fuera de alcance.

## Métricas

**992/992 tests verde**, **`tsc -b` verde** y GATE 0024 honrada.
