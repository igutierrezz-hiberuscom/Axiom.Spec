# 01 Requisitos

## Objetivo del documento

Definir los requisitos técnicos para registrar el campo `confirmed` en el schema
HTTP de onboarding (`launcher.workspaceSetup` y `launcher.workspaceAdopt`) y
alinear los tests de migración al contrato de preview grant vigente.

## Requisitos del incremento

- **REQ-A-01: Allow-list de campos del body**
  Añadir `confirmed` a las entradas `launcher.workspaceSetup` y
  `launcher.workspaceAdopt` de `CLOSED_BODY_FIELDS` en
  `apps/cli/src/commands/app-api.ts`, en posición previa a `confirmationToken`.

- **REQ-A-02: Tipado de campos del body**
  Añadir `confirmed: 'boolean'` a las entradas `launcher.workspaceSetup` y
  `launcher.workspaceAdopt` de `BODY_FIELD_TYPES` en el mismo archivo. El campo
  es opcional (`confirmed?: boolean` en `LauncherWorkspaceSetupBody`), por lo que
  `bodyFieldTypeError` lo salta cuando no está presente.

- **REQ-A-03: Alineación de tests "confirms" al contrato de preview grant**
  Actualizar los tests `confirms setup through runWorkspaceSetup…` y
  `confirms canonical adoption…` de
  `apps/cli/tests/launcher-onboarding-migration.test.ts` al flujo de dos pasos
  (preview → `confirmationToken` → confirmar), incluida la segunda llamada de
  idempotencia de adopt con su propio grant (los tokens son de un solo uso).

## Reglas de negocio relevantes

- `secureRequestBody` rechaza con `400 El body contiene campos no permitidos.`
  todo campo fuera del allow-list antes de la validación semántica.
- Sobre HTTP, `confirmed` es ignorado: `apiLauncherWorkspaceSetup` solo lo honra
  cuando `currentLauncherRequestContext() === undefined` (caller in-process), y
  `apiLauncherWorkspaceAdopt` usa `hasLauncherRequestAuthorization()`, que solo
  es verdadero tras consumir un preview grant (`markLauncherRequestAuthorized`).
- La autorización no puede forjarse con un booleano del body (diseño explícito de
  `launcher-security.ts` desde `8274c69`).
