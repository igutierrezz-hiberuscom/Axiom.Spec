# r14 runtime version identity

> **Código**: INC-20260911-r14-runtime-version-identity
> **Estado**: En Borrador
> **Fecha de creación**: 2026-09-11
> **Tipo de cambio**: unificación de fuente de identidad
> **Acción de origen**: `ACC-085`
> **Plan**: `PLAN-INC-20260911-r14-operator-control-surfaces` (posición 6 de 8)

## Resumen

Sustituir la versión de runtime escrita a mano por una identidad derivada de una única fuente, para que la versión del CLI, el manifest instalado, el estado del proyecto y el default de `--target-version` no puedan divergir.

## Contexto y motivación

Verificado el 2026-09-11:

- `packages/versioning/src/version.ts` exporta el literal `RUNTIME_VERSION = '0.1.0'`, con un comentario que reconoce el problema: se mantiene ahí «para evitar leer el `package.json` en runtime» y deja la sincronización como iteración futura.
- Esa constante alimenta el `runtime.version` que persiste el `ManagedState` y el default de `--target-version` de `axiom upgrade`.
- En paralelo, `apps/cli/src/generated-build-metadata.ts` ya es generado por `scripts/generate-cli-build-metadata.mjs` y expone `version`, `releaseTag`, `commit`, `buildId` y `dirty`. Existe un productor de identidad; el paquete de versionado no lo consume.

## Alcance

### Incluido

- Derivación de la versión de runtime desde la identidad de producto que gobierna la release del CLI.
- Coherencia verificable entre `axiom --version`, el manifest instalado, `ManagedState.runtime.version` y el default de `--target-version`.
- Derivación determinista, funcional en test y bajo bundling, sin leer `package.json` en caliente.
- Fallo explícito cuando la identidad no está disponible, en lugar de inventar un valor por defecto.
- Política de migración de los estados ya persistidos con `0.1.0`.

### Excluido

- Colapsar los dos ejes que R-13.5-A separó: la versión del CLI instalado y la versión de compatibilidad/migración project-scoped siguen siendo conceptos distintos, aunque compartan fuente.
- Reimplementar la convención de tags ni el gate de release de `ACC-080`.

## Dudas abiertas

Ninguna. Cerradas en D-03 (`DEC-20260911-223524-dfl08s`):
- Fuente única: `scripts/generate-cli-build-metadata.mjs` genera `packages/versioning/src/generated-version.ts` en tiempo de build.
- Sin release publicada: se utiliza el SemVer de `package.json` marcando `releaseTag: null` y `isDev: true`.
- Estados 0.1.0: compatibilidad total garantizada.
- `--target-version` por defecto: mantiene la versión de runtime activa.

## Decisiones funcionales cerradas

- Desaparece el literal escrito a mano en `version.ts`.
- La derivación es determinista y no requiere lectura en caliente de `package.json` en runtime.
- Se mantiene la separación conceptual entre versión de distribución del CLI y compatibilidad del proyecto.

## Consolidación en la spec general

Al cierre, una sola afirmación activa sobre de dónde sale cada versión que Axiom muestra o persiste.

## Estrategia E2E

Coherencia entre `axiom --version`, `ManagedState.runtime.version` y default de `--target-version`; pruebas de `@axiom/versioning` (`managed-state.test.ts`, `upgrade.test.ts`); `npm run build`, doctor, readiness y `git diff --check`.

## Trazabilidad y fuentes

Acción `ACC-085`, decisión `DEC-20260911-223524-dfl08s` y sesión R-14 del 2026-09-11 en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md`.

## Estado de validación humana

Validado. Criterios AC-085-01 a AC-085-05 verificados:
- Literal estático eliminado de `packages/versioning/src/version.ts`; ahora importa de `./generated-version`.
- `scripts/generate-cli-build-metadata.mjs` genera `packages/versioning/src/generated-version.ts` en sincronía con la identidad del CLI en cada build.
- Coherencia demostrada entre `axiom --version`, `ManagedState.runtime.version`, y valor por defecto de `--target-version`.
- Compatibilidad demostrada con estados 0.1.0 (42/42 tests PASS en `@axiom/versioning` y `upgrade-fanout.test.ts`).
- `npm run build` limpio, `npm run doctor` PASS (48/61 OK, 0 fallos), readiness PASS, `git diff --check` limpio.
