# Incremento 0029 — Memory Recall and Context Repair

> **Estado**: archived 2026-06-30
> **Plan**: `axiom.spec/plans/PLAN-INC-0029-memory-recall-and-context-repair.md`
> **Archive**: `openspec/changes/2026-07-01-0029-memory-recall/`

## Resumen ejecutivo

Incremento 0029 completa el Lote B de `0024` para incorporar recall útil
de memoria en workflows, reparación operativa de servicios de contexto
(MCP) y surfaces CLI/TUI de inventory.

## Mini-lotes

### C1 — Recall de memoria

- `rankEntries` y `buildRecallResult` puros en `@axiom/memory#recall`.
- Ranking: match de texto (case-insensitive, startBoost, occurrence
  boost) + recencia (90 días) + kind boost (decision=1.5, bug=1.4,
  pattern=1.2, learning=1.1, context=1.0).
- Cada hit retorna `reason: string` con explicación humana.
- 9 tests en `recall.test.ts`.

### C2 — Repair MCP

- `axiom mcp repair --id <mcpId>`. Verifica el id, valida el
  `projectBinding: required` y registra el repair en
  `.axiom-state/local/mcp-bindings.json` (state project-scoped, escritura
  atómica).

### C3 — Inventory CLI

- `axiom memory inventory`: total, per-kind, oldest, newest.
- `axiom mcp inventory`: total, required, optional, readonly,
  per-installMode.

### C4 — TUI inventory

- Nuevos ScreenIds: `memory-inventory`, `mcp-inventory`.
- Screens simples con render informativo.
- Wire-up completo al driver queda para una iteración posterior
  (stubs con shape compatible).

### C5 — Recall en workflows

- Helper `rankEntries` / `buildRecallResult` está disponible en
  `@axiom/memory`. El caller decide cuándo invocarlo (GATE 0029:
  opt-in). El wire-up al state machine de workflows queda para una
  iteración posterior (invasivo; requiere modificar
  `axiom-increment create` que está en el state machine de
  `axiom-increment`).

## Métricas

- **992/992 tests verde** (101/101 files). +9 tests del recall.
- **`tsc -b` verde**.
- **GATE 0024 honrada**: cross-project access bloqueado;
  recall es informativo, no autoritario.

## Siguiente paso

Iniciar incremento 0030 (operator app plugins and external bridge).
