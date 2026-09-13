# 03 Criterios de Aceptación

## Criterios de aceptación

### AC-083-01: Eliminación física de archivos legacy
`apps/cli/src/commands/self-update.ts` y `apps/cli/tests/self-update.test.ts` no existen en el filesystem ni en Git.

### AC-083-02: Superficie única de self-update
`apps/cli/src/index.ts` registra exclusivamente `registerSelfUpdateContract`. `axiom self-update` soporta `status`, `check`, `plan`, `apply`, `recover`.

### AC-083-03: Rechazo de flags legacy
Invocaciones con `--apply`, `--check`, `--target-version` o `--dry-run` sin subcomando canónico son rechazadas con error explicativo indicando el uso de subcomandos.

### AC-083-04: Scripts de instalación preservados
`scripts/install-global.mjs` y sus tests `scripts/install-global.test.mjs` se mantienen intactos y funcionales (18/18 tests PASS).

### AC-083-05: Validación general
`npm run build`, suites de pruebas de `apps/cli/tests/self-update*.test.ts`, doctor y readiness en verde.
