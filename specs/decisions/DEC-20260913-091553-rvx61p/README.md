<!-- AXIOM:MIGRATED — source: Axiom.Spec/decisions/0015-cavekit-discipline-post-mvp.md; migrated for INC-20260912-r15-canonical-roots-dedup via Core. -->

# Incremento 0015 — Cavekit Discipline and Optional GGA Adoption (post-MVP)

> **Estado**: archived 2026-06-30

Contenido preservado del documento legacy `decisions/0015-cavekit-discipline-post-mvp.md`.

## Resumen ejecutivo

Incremento 0015 adopta la disciplina seleccionada de Cavekit como **capability contracts nativos del runtime de Axiom**, con backprop workflow declarativo, check workflow read-only y contrato opcional de GGA.

## Mini-lotes

- **F1**: `@axiom/cavekit-discipline`, invariants y predicados helpers.
- **F2**: `backpropFromFailure`, con bug-create para severidad alta/crítica y spec-update para media/baja.
- **F3**: `checkDrift`, puro y case-insensitive.
- **F4**: `GgaContract` opcional con política `advisory-first` por defecto.

## Métricas

- **1029/1029 tests verde** (106/106 files), +14 tests.
- **`tsc -b` verde**.
- **GATE 0015 honrada**: capability contracts nativos, no copia de tooling externo.

## Siguiente paso

Cierre final del backlog post-0025; todos los incrementos estaban cerrados al consolidarse este registro.
