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

`D-02` del plan, que debe cerrarse antes de empezar: `TC-009` (`runAdapterRuntimeCoverageCheck` en `Axiom/packages/doctor/src/checks.ts`) exige `src/generator.ts` y `dist/index.js` presentes para los ocho adapters y falla si alguno no está. Si se dejan de versionar esos `dist/`, un clone limpio sin build previo haría fallar el check. Las tres salidas posibles son: garantizar build antes del doctor, relajar `TC-009` con una señal honesta que distinga «no construido» de «adapter ausente», o conservarlos versionados de forma deliberada y documentada.

## Decisiones funcionales cerradas

- Los 19 artefactos bajo `src/` se retiran en cualquiera de los tres escenarios de `D-02`.
- La higiene no queda a merced de una inspección manual: se añade gate.

## Consolidación en la spec general

Al cierre, dejar registrado qué se versiona y qué no, y el comportamiento de `TC-009` sobre un árbol sin build.

## Estrategia E2E

Gate anti-regresión ejercitado con un artefacto compilado introducido a propósito; comprobación explícita del comportamiento de `TC-009` en un árbol sin build según lo decidido en `D-02`; `npm run build`, doctor, readiness y `git diff --check`.

## Trazabilidad y fuentes

Acción `ACC-084` y sesión R-14 del 2026-09-11 en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md`.

## Estado de validación humana

Pendiente. Bloqueado hasta que `INC-20260911-r14-workflow-instance-selection` permita seleccionar instancia y hasta que `D-02` esté cerrada.
