# r14 build artifact hygiene

> **Código**: INC-20260911-r14-build-artifact-hygiene
> **Estado**: En Borrador
> **Fecha de creación**: 2026-09-11
> **Tipo de cambio**: higiene estructural y gate anti-regresión
> **Acción de origen**: `ACC-084`
> **Plan**: `PLAN-INC-20260911-r14-operator-control-surfaces` (posición 4 de 8)

## Resumen

Retirar del control de versiones los artefactos compilados que viven dentro del código fuente, decidir el destino de los `dist/` de adapters, tapar el hueco de exclusión que los permite y añadir un gate que impida su reaparición.

## Contexto y motivación

Verificado el 2026-09-11 con `git ls-files`:

- 19 artefactos compilados versionados bajo rutas `src/`: `packages/model-routing/src/{assignments,mutate,slots,types}.{js,js.map,d.ts,d.ts.map}` (16) y `apps/cli/src/commands/{mcp,repair,toolchain}.d.ts.map` (3).
- 148 ficheros versionados bajo `packages/adapters/*/dist/`.
- `.gitignore` declara `packages/*/dist/`, `apps/*/dist/`, `packages/*/*.tsbuildinfo` y `apps/*/*.tsbuildinfo`. Los `dist/` de adapters están un nivel más abajo (`packages/adapters/<target>/dist/`), fuera del patrón, y nada excluye emisiones dentro de `src/`.
- `packages/model-routing/src/types.js` comparte commit con su `.ts` (`17416d2`), así que no hay evidencia de desincronización. El problema es de higiene y de ambigüedad de fuente, no una divergencia demostrada.

## Alcance

### Incluido

- Retirada de los 19 artefactos compilados versionados bajo `src/`.
- Ejecución de la decisión `D-02` sobre los 148 ficheros bajo `packages/adapters/*/dist/`.
- Extensión de `.gitignore` a salidas anidadas (`packages/*/*/dist/`) y a cualquier `.js`/`.d.ts`/`.map` emitido dentro de `src/`.
- Gate automático que falle ante la reaparición de artefactos compilados versionados.

### Excluido

- Cambiar la estrategia de compilación o el layout de salidas de los paquetes, que pertenece al ámbito de `ACC-004`.
- Reorganizar los adapters o su contrato de generación.

## Dudas abiertas

Ninguna. Cerradas en D-02 (`DEC-20260911-223523-jii9h2`):
- Los 19 artefactos bajo `src/` y los 148 bajo `packages/adapters/*/dist/` se desversionan.
- `.gitignore` se extiende para cubrir salidas anidadas y archivos compilados en `src/`.
- `TC-009` distingue honestamente entre adapter ausente (FAIL) y falta de build (WARN).
- Se añade gate anti-regresión automatizado.

## Decisiones funcionales cerradas

- El código fuente no contiene artefactos compilados.
- Las salidas anidadas quedan formalmente excluidas en `.gitignore`.
- `TC-009` no bloquea diagnósticos en clones limpios no compilados.

## Consolidación en la spec general

Al cierre, dejar registrado qué se versiona y qué no, y el comportamiento de `TC-009` sobre un árbol sin build.

## Estrategia E2E

Gate anti-regresión ejercitado comprobando ausencia de artefactos compilados en `src/` y exclusión de `dist/` anidados; validación del comportamiento de `TC-009` compilado y no compilado; `npm run build`, doctor, readiness y `git diff --check`.

## Trazabilidad y fuentes

Acción `ACC-084`, decisión `DEC-20260911-223523-jii9h2` y sesión R-14 del 2026-09-11 en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md`.

## Estado de validación humana

Validado. Criterios AC-084-01 a AC-084-05 verificados:
- 19 artefactos compilados bajo `src/` eliminados y desversionados de Git.
- 148 archivos compilados bajo `packages/adapters/*/dist/` desversionados del índice de Git.
- `.gitignore` extendido para cubrir `packages/*/*/dist/`, `packages/*/*/*.tsbuildinfo` y emisiones de `.js`, `.d.ts` y `.map` en `src/`.
- `TC-009` en `checks.ts` refinado para emitir `WARN` informativo ante adapters no construidos en clones limpios y `FAIL` solo ante adapters o generadores ausentes.
- Gate automatizado anti-regresión `apps/cli/tests/artifact-hygiene.test.ts` implementado y pasando 4/4 tests.
- `npm run build` limpio, `npm run doctor` PASS (48/61 OK, 0 fallos), readiness PASS, `git diff --check` limpio.
