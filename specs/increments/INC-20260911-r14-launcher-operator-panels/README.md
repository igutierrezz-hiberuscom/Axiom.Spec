# r14 launcher operator panels

> **Código**: INC-20260911-r14-launcher-operator-panels
> **Estado**: En Borrador
> **Fecha de creación**: 2026-09-11
> **Tipo de cambio**: nueva superficie de operador en el launcher
> **Acción de origen**: `ACC-082`
> **Plan**: `PLAN-INC-20260911-r14-operator-control-surfaces` (posición 8 de 8)

## Resumen

Añadir al launcher las dos pantallas de operador que hoy no existen: model routing y actualización de Axiom. Ambas delegan en los comandos y endpoints canónicos, sin duplicar lógica.

## Contexto y motivación

Verificado el 2026-09-11:

- `apps/cli/src/commands/app-self-update.ts` implementa el adaptador headless y `app-api.ts` enruta las ocho operaciones (`status`, `check`, `plan`, `apply`, `recoverPlan`, `recover`, `cancel`, `getOperation`), con 11 pruebas en `apps/cli/tests/app-self-update.test.ts`.
- El frontend estático (`apps/cli/static/launcher/{index.html,launcher.js,panels.js,transport.js}`) no contiene ninguna superficie de self-update ni de model routing. Los paneles reales son ADO suggestions, rama de rol Git, workflow ADO y telemetría.
- `axiom model` no tiene endpoint ni pantalla: solo existe como CLI.
- Tras la retirada de la TUI, el launcher es la única interfaz guiada, así que estos dos mandos quedan sin ventanilla.

## Alcance

### Incluido

- Pantalla de model routing: política base, slots, override efectivo por slot, nivel de soporte real del destino, y acciones `set`/`unset`/`reset` y `validate` con preview y confirmación.
- Pantalla de actualización: versión publicada, descargada e instalada con su relación tipada, frescura y procedencia del cache, y acciones `check`, `plan`, `apply`, `recover` y `cancel` con progreso acotado y cierre/reinicio explícito.
- Reutilización de los gates vigentes del control plane: sesión, same-origin y grant single-use con TTL y binding de proyecto/acción/payload.
- Presentación honesta del soporte por destino, según el contrato de `ACC-086`.

### Excluido

- Duplicar lógica de negocio, SemVer, Git o mutaciones de fichero en el frontend.
- Redefinir los gates de `ACC-070`..`ACC-076`.
- Crear una matriz de regresión paralela: se extiende la hermética existente.
- Endpoints nuevos para self-update: ya existen y se consumen.

## Dudas abiertas

Ninguna propia. Depende de que estén cerradas `D-01` (superficie superviviente de self-update), `D-03` (identidad de versión que la pantalla muestra) y `D-04` (contrato de recomendación y aviso). Riesgo residual conocido y heredado de R-13.4: la cobertura DOM del launcher.

## Decisiones funcionales cerradas

- El launcher es la ventanilla única; estas dos pantallas no vuelven a la terminal.
- Donde el enrutado no se obedece, la pantalla lo dice en lugar de sugerir control efectivo.

## Consolidación en la spec general

Al cierre, la spec de interfaces operativas debe reflejar las dos pantallas nuevas y qué acciones exponen con confirmación.

## Estrategia E2E

Extensión de la matriz hermética del launcher con las dos pantallas, incluidos negativos, grants caducados, operación en curso, cancelación y ausencia de mutación en las rutas read-only.

## Trazabilidad y fuentes

Acción `ACC-082` y sesión R-14 del 2026-09-11 en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md`; antecedentes `ACC-070`..`ACC-080`.

## Estado de validación humana

Pendiente. Es el último del lote: bloqueado hasta que estén cerrados los incrementos de superficie de self-update, identidad de versión y contrato de recomendación.
