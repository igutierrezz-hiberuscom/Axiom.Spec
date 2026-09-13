# 03 Criterios de Aceptación

## Criterios de aceptación

### AC-084-01: Ausencia de compilados en `src/`
Ningún archivo con extensión `.js`, `.d.ts` o `.map` existe bajo directorios `src/` en `packages/` o `apps/`.

### AC-084-02: Exclusión de `dist/` anidados en adapters
`packages/adapters/*/dist/` no contiene archivos versionados en Git y queda cubierto por `.gitignore`.

### AC-084-03: Honestidad de `TC-009`
En un árbol sin compilar, `TC-009` emite una advertencia honesta `WARN` que solicita ejecutar `npm run build`, sin emitir un falso `FAIL` de adapter ausente. Al compilarse, emite `PASS`.

### AC-084-04: Gate anti-regresión
La suite de pruebas incluye una verificación automática que falla si se detectan artefactos compilados versionados en ubicaciones prohibidas.

### AC-084-05: Validación integral
`npm run build`, `npm run doctor`, `npm run readiness:first-project` y `git diff --check` devuelven estado verde.
