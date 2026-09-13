# 02 Cambios de Modelo

## Objetivo del documento

Detallar la remoción de módulos en `apps/cli`.

## Estructuras y archivos afectados

- `apps/cli/src/commands/self-update.ts` eliminado.
- `apps/cli/tests/self-update.test.ts` eliminado.
- Superficie canónica: `apps/cli/src/commands/self-update-contract.ts` (exporta `registerSelfUpdateContract`, `runSelfUpdateContract`).

## Contratos o estados afectados

- `axiom self-update` mantiene envelope `schemaVersion: 1` sin alteraciones en su API pública.
- Las funciones legacy (`runSelfUpdateCheck`, `runSelfUpdateTarget`, `runSelfUpdateApply`, etc.) dejan de existir.
