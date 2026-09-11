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

`D-03` del plan, que debe cerrarse antes de empezar: la fuente única exacta, el comportamiento cuando no hay release publicada, el trato de los estados persistidos con `0.1.0` y si el default de `--target-version` sigue siendo la versión de runtime o pasa a ser la identidad de producto.

## Decisiones funcionales cerradas

- Desaparece el literal escrito a mano.
- La ausencia de identidad se reporta, no se rellena con un valor plausible.

## Consolidación en la spec general

Al cierre, una sola afirmación activa sobre de dónde sale cada versión que Axiom muestra o persiste.

## Estrategia E2E

Coherencia entre las cuatro superficies de versión; ejecución desde un checkout sin release publicada; migraciones existentes y estados persistidos con `0.1.0`; default de `--target-version` según lo decidido.

## Trazabilidad y fuentes

Acción `ACC-085` y sesión R-14 del 2026-09-11 en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md`; antecedente `ACC-080`.

## Estado de validación humana

Pendiente. Bloqueado hasta que `INC-20260911-r14-workflow-instance-selection` permita seleccionar instancia, `D-03` esté cerrada e `INC-20260911-r14-single-self-update-surface` haya fijado la superficie superviviente.
