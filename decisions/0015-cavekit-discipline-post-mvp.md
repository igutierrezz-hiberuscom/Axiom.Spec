# Incremento 0015 — Cavekit Discipline and Optional GGA Adoption (post-MVP)

> **Estado**: archived 2026-06-30
> **Plan**: `axiom.spec/plans/PLAN-INC-0015-cavekit-discipline-post-mvp.md`
> **Archive**: `openspec/changes/2026-07-01-0015-cavekit-discipline/`

## Resumen ejecutivo

Incremento 0015 adopta la disciplina seleccionada de Cavekit como
**capability contracts nativos del runtime de Axiom** (no copia
de tooling externo), con backprop workflow declarativo, check
workflow read-only y contrato opcional de GGA.

## Mini-lotes

### F1 — Capability contracts de invariants

- `@axiom/cavekit-discipline` package nuevo.
- `Invariants<T>` con `id`, `level`, `message`, `predicate` puro.
- `evaluateInvariant` y `evaluateInvariants`.
- Predicates helpers: `exactLength`, `minLength`, `inRange`.

### F2 — Backprop workflow (declarativo)

- `backpropFromFailure(failure) → BackpropProposal`.
- Severity `critical`/`high` → `kind: 'bug-create'`.
- Severity `medium`/`low` → `kind: 'spec-update'`.
- Mensaje vacío → `kind: 'noop'`.

### F3 — Check workflow read-only

- `checkDrift(sources, expectedKeywords) → DriftReport`.
- Detecta keywords ausentes en alguna de las 3 fuentes.
- Pura; case-insensitive.

### F4 — GGA opcional

- `GgaContract` con `mode: 'advisory-first' | 'strict'`,
  `optional: true`, `rollback.policy`.
- `DEFAULT_GGA_CONTRACT` con `advisory-first`.
- `applyGgaGate` retorna el set original en `advisory-first`.

## Métricas

- **1029/1029 tests verde** (106/106 files). +14 tests.
- **`tsc -b` verde**.
- **GATE 0015 honrada**: capability contracts nativos, no copia.
- **GGA explícitamente opcional**.

## Siguiente paso

Cierre final del backlog post-0025. Estado del backlog operativo
reducido a 0 (todos los incrementos cerrados).
