# Incremento 0027 — Toolchain Provider Expansion and Repair

> **Estado**: archived 2026-06-30
> **Plan**: `axiom.spec/plans/PLAN-INC-0027-toolchain-provider-expansion-and-repair.md`
> **Archive**: `openspec/changes/2026-07-01-0027-toolchain-provider-expansion/`

## Resumen ejecutivo

Incremento 0027 completa el Lote B de `0023` para que la toolchain
gestionada cubra las herramientas P1 (CodeGraph, Graphify,
Headroom/RTK, Caveman, Autoskills), soporte selección por repo,
repair y políticas locales por tool.

## Mini-lotes

### B1 — Tools P1 operativas

- 5 tools P1 ahora con `mvp: true`, `detectionPaths`, `mcpServer`,
  `gitignoreEntries`.
- Bug pre-existente de `resolveDetectionPath` corregido (paths
  relativos al `projectRoot`, no al cwd del proceso).

### B2 — Repair

- Sub-comando `axiom toolchain repair [--id <id>]`. Idempotente.

### B3 — Selección por repo

- `axiom toolchain add --id <id> --path <repoId>`. Valida contra
  `topology.yaml#roleCodeRepositories`.

### B4 — `.gitignore` per-tool

- Sub-comando `axiom toolchain gitignore [--write <file>]`. Output
  ordenado y deduplicado.

### B5 — Docs y openspec

- Roadmap y plan actualizados; openspec
  `2026-07-01-0027-toolchain-provider-expansion/`.

## Métricas

- **983/983 tests verde** (100/100 files). +20 tests nuevos vs 963.
- **`tsc -b` verde**.
- **GATE 0023 y 0027 honradas**.

## Siguiente paso

Iniciar incremento 0029 (memory recall and context repair).
