# 03 Criterios de Aceptación

## Criterios de aceptación

### AC-A-01: Cuerpos con `confirmed` aceptados en el schema HTTP
Un POST a `/api/launcher/workspace/setup` o `/api/launcher/workspace/adopt` con
`confirmed: true|false` en el body no devuelve `400 El body contiene campos no
permitidos.`; llega a la validación semántica del handler.

### AC-A-02: Tipado correcto de `confirmed`
Un body con `confirmed` de tipo no booleano devuelve `400 El campo 'confirmed'
tiene un tipo inválido.`; un body sin `confirmed` no produce error de tipo.

### AC-A-03: Suite de migración en verde
`npx vitest run apps/cli/tests/launcher-onboarding-migration.test.ts` pasa 8/8,
incluidos los 2 tests "confirms" ejecutados vía flujo preview → token → confirmar.

### AC-A-04: Tests vecinos sin regresión
`npx vitest run apps/cli/tests/launcher-onboarding.test.ts` se mantiene en verde
(35/35), incluidos los casos de tipos inválidos de las rutas workspace.

### AC-A-05: Sin cambios de producto en la lógica de autorización
`secureRequestBody`, `apiLauncherWorkspaceSetup`, `apiLauncherWorkspaceAdopt` y
`launcher-security.ts` no modifican su lógica; el cambio se limita a los
allow-lists y a los tests.
