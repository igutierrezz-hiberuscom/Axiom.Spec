# baseline group A — onboarding HTTP allow-list

> **Código**: INC-20260913-baseline-group-a-onboarding-allowlist
> **Estado**: Implementado (pendiente de revisión humana)
> **Fecha de creación**: 2026-09-13
> **Tipo de cambio**: corrección de contrato HTTP de onboarding (Grupo A del plan de baseline)
> **Plan**: `PLAN-BASELINE-TESTS-20260913` (paso 1 de 7)
> **Bug relacionado**: `BUG-20260910-baseline-launcher-onboarding-schema`

## Resumen

Registrar el campo `confirmed` en el allow-list HTTP (`CLOSED_BODY_FIELDS`) y en el
mapa de tipos (`BODY_FIELD_TYPES`) de `app-api.ts` para las rutas
`launcher.workspaceSetup` y `launcher.workspaceAdopt`, de modo que los cuerpos que
lo envían no sean rechazados con `400 El body contiene campos no permitidos.`
antes de llegar a la validación semántica.

## Contexto y motivación

- Baseline del 2026-09-13 (`npx vitest run` en `Axiom/`): 16 tests fallidos en 9
  archivos; el Grupo A concentra 8 fallos en
  `apps/cli/tests/launcher-onboarding-migration.test.ts`.
- El commit `b3be943` añadió `confirmed?: boolean` a `LauncherWorkspaceSetupBody`
  (y por herencia a `LauncherWorkspaceAdoptBody`) en `app-onboarding.ts`, pero no
  lo registró en el allow-list de `app-api.ts`. `secureRequestBody` rechaza todo
  campo fuera del allow-list con `400` antes de la validación semántica.
- El campo es seguro: `apiLauncherWorkspaceSetup` solo honra `confirmed === true`
  cuando `currentLauncherRequestContext() === undefined` (caller in-process). Para
  requests HTTP reales el contexto siempre existe, así que `confirmed` es ignorado
  y la autorización sigue viniendo del preview grant (`confirmationToken`).

## Alcance

### Incluido

- `confirmed` en `CLOSED_BODY_FIELDS` para `launcher.workspaceSetup` y
  `launcher.workspaceAdopt`.
- `confirmed: 'boolean'` en `BODY_FIELD_TYPES` para ambas rutas.
- Alineación de los 2 tests "confirms" de `launcher-onboarding-migration.test.ts`
  al contrato de preview grant vigente desde `8274c69` (flujo de dos pasos:
  preview → `confirmationToken` → confirmar), que ya fallaban por diseño del
  contrato y no por el allow-list.

### Excluido

- Cambios en la lógica de `secureRequestBody`, `apiLauncherWorkspaceSetup` o
  `apiLauncherWorkspaceAdopt`.
- Cambios en `packages/core/src/self-update/**` ni en los gates AC-078/079.
- Resto de grupos (B–G) del plan de baseline.

## Dudas abiertas

Ninguna. El diagnóstico del plan para el Grupo A era incompleto: 6 de los 8 fallos
eran del allow-list; los 2 restantes ("confirms setup…", "confirms canonical
adoption…") ya fallaban desde `8274c69`, que sustituyó el gate `confirmed:true`
por preview grant obligatorio (la autorización no puede forjarse con un booleano
del body — ver comentario en `launcher-security.ts`). La corrección de esos 2
tests es de tests, no de producto.

## Decisiones funcionales cerradas

- `confirmed` es un campo opcional del body, permitido pero ignorado sobre HTTP;
  la ejecución real exige preview grant de un solo uso.
- Los tests "confirms" usan el flujo canónico de dos pasos ya establecido por
  `onboardingGrant` en `launcher-onboarding.test.ts`.

## Consolidación en la spec general

Al cierre, dejar registrado en la spec canónica de onboarding que `confirmed` es
parte del schema HTTP de setup/adopt (permitido, opcional, sin efecto
autorizativo sobre HTTP) y que la confirmación HTTP exige preview grant.

## Estrategia E2E

- `npx vitest run apps/cli/tests/launcher-onboarding-migration.test.ts` → 8/8.
- `npx vitest run apps/cli/tests/launcher-onboarding.test.ts` → 35/35 (vecinos).
- `npm run build` → exit 0.

## Trazabilidad y fuentes

Plan `Axiom.Spec/plans/PLAN-BASELINE-TESTS-20260913.md` (Grupo A), bug
`BUG-20260910-baseline-launcher-onboarding-schema`, commits `b3be943` (introduce
`confirmed` en el body type) y `8274c69` (introduce el gate de preview grant).

## Estado de validación humana

Pendiente de revisión humana. Validación automatizada ejecutada:

- `npx vitest run apps/cli/tests/launcher-onboarding-migration.test.ts` →
  **8/8 passed** (13.55s).
- `npx vitest run apps/cli/tests/launcher-onboarding.test.ts` → **35/35 passed**
  (7.70s).
- `npm run build` → exit 0.

## Resultado de implementación (2026-09-13)

Cambios aplicados:

1. `apps/cli/src/commands/app-api.ts`:
   - `CLOSED_BODY_FIELDS['launcher.workspaceSetup']`: añadido `'confirmed'`
     antes de `'confirmationToken'`.
   - `CLOSED_BODY_FIELDS['launcher.workspaceAdopt']`: añadido `'confirmed'`
     antes de `'adoptSpec'`.
   - `BODY_FIELD_TYPES['launcher.workspaceSetup']`: añadido
     `confirmed: 'boolean'`.
   - `BODY_FIELD_TYPES['launcher.workspaceAdopt']`: añadido
     `confirmed: 'boolean'`.
2. `apps/cli/tests/launcher-onboarding-migration.test.ts` (alineación al contrato
   de preview grant, no cambio de producto):
   - Test `confirms setup through runWorkspaceSetup…`: primera llamada de preview
     sin token (afirma `executed: false` + `confirmationToken` + ausencia de
     mutación), segunda llamada con el token (afirma `executed: true`).
   - Test `confirms canonical adoption…`: mismo flujo de dos pasos para la
     primera ejecución y grant adicional para la segunda llamada de idempotencia
     (los tokens son de un solo uso).

Hallazgo relevante: el diagnóstico del plan para el Grupo A era incompleto. 6 de
los 8 fallos eran del allow-list (400); los 2 restantes ya fallaban desde
`8274c69`, que sustituyó el gate `confirmed:true` por preview grant obligatorio.
El fix del allow-list solo desbloqueó 6/8; los otros 2 requerían alineación de
tests al contrato vigente.

Estado de criterios: AC-A-01 a AC-A-05 verificados. El incremento queda
`pending` de cierre formal hasta revisión humana y consolidación en la spec
canónica de onboarding.
