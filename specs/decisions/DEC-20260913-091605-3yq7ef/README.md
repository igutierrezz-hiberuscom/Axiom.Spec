<!-- AXIOM:MIGRATED — source: Axiom.Spec/decisions/0027-toolchain-provider-expansion-and-repair.md; migrated for INC-20260912-r15-canonical-roots-dedup via Core. -->

# Incremento 0027 — Toolchain Provider Expansion and Repair

> **Estado**: archived 2026-06-30

Contenido preservado del documento legacy `decisions/0027-toolchain-provider-expansion-and-repair.md`.

## Resumen ejecutivo

La toolchain gestionada cubre las herramientas P1, soporta selección por repo, repair y políticas locales por herramienta.

## Lotes

- CodeGraph, Graphify, Headroom/RTK, Caveman y Autoskills quedan operativos con detección, MCP y gitignore.
- `axiom toolchain repair --id <id>` es idempotente.
- `axiom toolchain add --id <id> --path <repoId>` valida contra `topology.yaml`.
- `axiom toolchain gitignore [--write <file>]` produce salida ordenada y deduplicada.

## Métricas

**983/983 tests verde**, **`tsc -b` verde** y GATE 0023/0027 honradas.
