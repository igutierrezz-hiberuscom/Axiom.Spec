# 02 Cambios de Modelo

## Objetivo del documento

Detallar los cambios estructurales en el monorepo Axiom.

## Entidades o estructuras afectadas

- Eliminación del workspace `packages/tui`.
- Depuración de `package-lock.json` (se remueven las referencias a `"packages/tui"` y al junction).
- Ausencia de `@axiom/tui` en el catálogo de paquetes.

## Contratos o estados afectados

- No hay impacto en contratos de runtime: `@axiom/tui` ya no formaba parte de las dependencias de producción.
