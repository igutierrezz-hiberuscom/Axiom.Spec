<!-- AXIOM:MIGRATED — source: Axiom.Spec/decisions/0031-adr-cmm-replaces-graphify-and-codegraph.md; migrated for INC-20260912-r15-canonical-roots-dedup via Core. -->

# ADR-0031 — `cmm` replaces `graphify` and `codegraph` in the canonical provider set

> **Estado**: accepted, implemented 2026-07-24

Contenido preservado del documento legacy `decisions/0031-adr-cmm-replaces-graphify-and-codegraph.md`.

## Decisión

`codegraph` y `graphify` se eliminan del set seleccionable y `cmm` (`codebase-memory-mcp`) pasa a ser el único proveedor de inteligencia estructural, cubriendo `code.knowledgeGraph` y `code.structureAnalysis`. El set canónico queda en seis IDs: `filesystem`, `axiom-gateway`, `serena`, `cmm`, `engram` y `generated-snapshots`.

El fallback es `cmm -> filesystem`; `serena -> filesystem` permanece igual. `cmm` usa una marca de sincronización por proyecto para freshness y auto-sync best-effort. Los kinds legacy se conservan solo en enums por compatibilidad histórica.

## Consecuencias y alternativas

Se actualizaron guards, catálogo y wiring de toolchain; no se migran automáticamente strings antiguos en `workspace.json`. Se rechazó mantener tres proveedores estructurales o reutilizar el kind `knowledge-graph`.
