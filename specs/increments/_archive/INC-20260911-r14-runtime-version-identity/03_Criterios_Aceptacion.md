# 03 Criterios de Aceptación

## Criterios de aceptación

### AC-085-01: Desaparición del literal manual
`packages/versioning/src/version.ts` no contiene el literal cableado a mano `RUNTIME_VERSION = '0.1.0'` sin procedencia; consume el módulo generado en build.

### AC-085-02: Generación en build
`npm run build` ejecuta `scripts/generate-cli-build-metadata.mjs` y genera sincronizadamente `generated-build-metadata.ts` y `packages/versioning/src/generated-version.ts`.

### AC-085-03: Coherencia de identidades
`RUNTIME_VERSION`, `axiom --version`, `ManagedState.runtime.version` y el valor por defecto de `--target-version` en `axiom upgrade` resuelven al mismo SemVer de producto.

### AC-085-04: Compatibilidad de estados 0.1.0
Los estados `managed-state.json` existentes con versión `0.1.0` se leen y operan sin errores ni migraciones espurias.

### AC-085-05: Validación general
`npm run build`, pruebas de `@axiom/versioning`, doctor y readiness en estado verde.
