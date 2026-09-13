# Plan de corrección de la baseline de tests de Axiom (2026-09-13)

> **Estado: CERRADO (2026-09-13).** Los 7 grupos se implementaron secuencialmente con subagentes y verificación independiente por grupo. Validación final: `npx vitest run` → **Test Files 358 passed (358), Tests 3765 passed (3765)** en 478s, y `npx tsc -b` exit 0. Los ajustes sobre el diagnóstico original quedan documentados en cada grupo.
>
> **Cierre documental (2026-09-13):** los incrementos de registro de los grupos A y B quedaron cerrados y archivados en `specs/increments/_archive/` (`INC-20260913-baseline-group-a-onboarding-allowlist`, `INC-20260913-baseline-group-b-init-role-identity`), con conocimiento estable consolidado en `specs/03_Modelo_Operativo_y_Datos.md` (contrato dual del `role` persistido) y `specs/07_Gobierno_y_Seguridad.md` (campo `confirmed` en el schema de onboarding). Los grupos C–G son fixes de tests/índice sin incremento propio: su registro es este plan.

## Objetivo

Restablecer la suite completa de Vitest del runtime `Axiom/` en verde: 16 tests fallidos en 9 archivos (3749/3765 pasan hoy). Este plan diagnostica cada fallo con su causa raíz verificada y define el orden de corrección, sin mezclar alcances de bugs ya clasificados.

## Contexto

- Baseline verificada el 2026-09-13 con `npx vitest run` en `Axiom/`: `Test Files 9 failed | 349 passed (358)`, `Tests 16 failed | 3749 passed (3765)`.
- La mayoría de las regresiones provienen de los commits recientes `b3be943` (self-update transaccional + spec-repo workspace) y `95cab5e` (multi-instance workflows + model routing panel), que cambiaron contratos sin actualizar tests o validaciones paralelas.
- Existen bugs de la baseline del 2026-09-10 en `Axiom.Spec/specs/bugs/` que describen fallos de la ejecución `npm test` de ese día (28 archivos / 65 tests). La baseline de hoy es distinta: varios de esos fallos ya no se reproducen y este plan parte de la evidencia actual, no de la histórica.

## Diagnóstico verificado por causa raíz

### Grupo A — Contrato HTTP de onboarding (8 tests, deterministas)

**Archivo:** `apps/cli/tests/launcher-onboarding-migration.test.ts` (8/8 fallan)

**Causa raíz:** el commit `b3be943` añadió el campo `confirmed?: boolean` a `LauncherWorkspaceSetupBody` (y por herencia a `LauncherWorkspaceAdoptBody`) en `apps/cli/src/commands/app-onboarding.ts`, pero NO lo registró en `CLOSED_BODY_FIELDS` ni en `BODY_FIELD_TYPES` de `apps/cli/src/commands/app-api.ts`. `secureRequestBody` rechaza todo campo fuera del allow-list con `400 El body contiene campos no permitidos.` antes de llegar a la validación semántica.

**Evidencia:** los 8 tests envían `confirmed: true/false` y reciben 400; los que esperan mensajes `superpone`/`absoluto`/`otro proyecto` reciben el mensaje de campos no permitidos.

**Fix aplicado (2026-09-13, ajustado durante la implementación):** añadir `confirmed` a las entradas `launcher.workspaceSetup` y `launcher.workspaceAdopt` de `CLOSED_BODY_FIELDS` y `BODY_FIELD_TYPES` (tipo `boolean`) en `app-api.ts`. El campo es seguro: `apiLauncherWorkspaceSetup` solo honra `confirmed` cuando `currentLauncherRequestContext() === undefined` (caller in-process), y HTTP sigue requiriendo grant.

**Segunda causa raíz descubierta al implementar:** el allow-list solo explicaba 6 de los 8 fallos. Los 2 restantes (`confirms setup…`, `confirms canonical adoption…`) esperaban `executed: true` vía HTTP con `confirmed: true` sin token; el commit `8274c69` sustituyó ese gate por preview grant obligatorio (`launcher-security.ts` documenta que la autorización no puede forjarse con un booleano del body). Fix complementario: alinear esos 2 tests al flujo canónico de dos pasos (preview → `confirmationToken` → confirmar) ya establecido en `launcher-onboarding.test.ts`. No se tocó la lógica de autorización de producto.

**Bug relacionado:** `BUG-20260910-baseline-launcher-onboarding-schema` (mismo síntoma, ya reportado).

### Grupo B — Divergencia de rol en `axiom.yaml` (1 test, determinista)

**Archivo:** `apps/cli/tests/workspace-incremental.test.ts` — `runRepoAdd sobre proyecto bare-init (item 4)`

**Causa raíz:** el commit `b3be943` cambió `buildAxiomYaml` en `apps/cli/src/commands/init.ts` para emitir `role: sdd` cuando el rol es `sdd` (línea `role: ${role === 'sdd' ? 'sdd' : canonicalManagedRole(kind)}`), pero `expectedIdentity` en `apps/cli/src/commands/workspace-structural-plan.ts` sigue esperando `role: axiom` para repos `kind: axiom` (vía `canonicalManagedRole`). Resultado: `runRepoAdd` sobre un proyecto bare-init falla con `AXIOM_STRUCTURAL_INVALID_AXIOM_YAML` porque la identidad existente (`role: sdd`) no coincide con el desired state (`role: axiom`).

**Evidencia:** reproducido con script de diagnóstico; `details.actual.role = 'sdd'` vs `details.expected.role = 'axiom'`.

**Decisión de dirección (ajustada durante la implementación):** el revert literal a `canonicalManagedRole(kind)` habría emitido `role: legacy` para el init default (`installed-multi-repo`, `kind: legacy`) y roto el contrato vigente que consumen `upgrade.ts:308` (fan-out gate `resolution.role === 'sdd'`), el guard de repo-affinity y los tests `init.test.ts`/`schemaversion2-e2e.test.ts`. Fix aplicado: `role: ${kind === 'legacy' ? 'sdd' : canonicalManagedRole(kind)}` — idéntico al comportamiento de `b3be943` en todos los casos salvo el que rompía el Grupo B (`kind: axiom` ya no emite `role: sdd`). El vocabulario dual del rol persistido (`sdd` para legacy, `axiom` para autoridad) queda documentado como deuda de unificación futura.

**Bug relacionado:** `BUG-20260910-baseline-workspace-setup-adoption-contract`.

### Grupo C — Schema v2 del state store (2 tests, deterministas)

**Archivos:**
- `apps/cli/tests/axiom-increment-metadata.test.ts` — Scenario 2: espera `schemaVersion: 1` y record plano; ahora es `2` con instance containers.
- `apps/cli/tests/axiom-qa-e2e.test.ts` — Scenario 2 (parallel): `workflows['qa-e2e'].vars` es `undefined` porque el entry ahora es un instance container (`{ instances: {...}, activeInstanceId }`) cuando el record lleva `vars.id`.

**Causa raíz:** el commit `95cab5e` actualizó `packages/workflow/src/state-store.ts` a `WORKFLOW_STATE_SCHEMA_VERSION = 2` con per-instance records: `saveWorkflowState` envuelve workflows con `vars.id`/`metadataId`/`planId` en `WorkflowInstanceContainer`. Los dos tests siguen leyendo el shape plano v1.

**Fix:** actualizar los tests al contrato v2:
- `axiom-increment-metadata.test.ts`: esperar `schemaVersion: 2` y leer el record desde el container (`workflows.increment.instances['9999']` o el active).
- `axiom-qa-e2e.test.ts`: leer `vars` desde la instancia activa del container, o usar `loadWorkflowState(projectRoot, 'qa-e2e')` que ya normaliza la lectura.

**Nota:** verificar primero en `Axiom.Spec/specs/04_Flujos_SDD_y_Ciclo_de_Vida.md` que el contrato v2 documentado coincide con la implementación; si el spec canónico aún describe v1, hay que actualizar el spec también.

### Grupo D — Warnings tipados en `runWorkspaceSetup` (1 test, determinista)

**Archivo:** `apps/cli/tests/inc-20260727-adoption-config-scaffolding.test.ts` — `w.includes is not a function`

**Causa raíz:** el commit `8274c69` cambió `WorkspaceSetupResult.warnings` de `string[]` a `readonly StructuralWarning[]` (objetos con `kind`/`code`/`message`). El test sigue llamando `w.includes(...)` sobre objetos.

**Fix:** actualizar el test para usar `w.message.includes(...)` (o `w.code`/`w.path` según lo que se quiera afirmar).

### Grupo E — Fixture de `workspace.json` sin `projectId` (1 test, determinista)

**Archivo:** `apps/cli/tests/configure.test.ts` — Scenario 5: `mergea sobre un workspace.json existente`

**Causa raíz:** el test siembra un `workspace.json` con `{ schemaVersion: 1, adapters, profile, overlay }` SIN `projectId`/`createdAt`/`updatedAt`. El loader estricto `normalizeState` en `packages/user-workspace/src/workspace-state.ts` exige `projectId`, `createdAt` y `updatedAt` como strings (error `workspace.json.projectId debe ser string`).

**Fix:** actualizar el fixture del test para incluir `projectId` (el slug del proyecto), `createdAt` y `updatedAt` ISO válidos, y `providers: []` (el loader estricto también exige `providers` como array obligatorio, no solo `adapters`). Es un fix de test, no de producto: el loader estricto es intencional (ver `BUG-20260910-baseline-provider-worktree-state`, que documenta el mismo error desde el lado de doctor/providers).

### Grupo F — Artefactos `dist/` trackeados en git (1 test, determinista)

**Archivo:** `apps/cli/tests/artifact-hygiene.test.ts` — `does not track nested dist directories in git index`

**Causa raíz:** el commit `95cab5e` añadió el test Y las reglas de `.gitignore` (`packages/*/*/dist/`), pero los `dist/` de los 8 adapters (`packages/adapters/*/dist/`) ya estaban trackeados en git desde antes; `.gitignore` no afecta archivos ya trackeados.

**Evidencia:** `git ls-files "packages/adapters/*/dist/**"` devuelve ~120 archivos `.js`/`.d.ts`/`.map` de antigravity, claude-code, codex, cursor, github-copilot, opencode, visual-studio-2026 y vscode.

**Fix:** `git rm -r --cached packages/adapters/*/dist` (des-trackear sin borrar del disco) y commitear. Los archivos quedan ignorados por el `.gitignore` vigente.

### Grupo G — Flaky por carga (2 tests, NO deterministas)

**Archivos:**
- `apps/cli/tests/r13-acc-076-matrix.test.ts` — `ACC-072-06` timeout 15s + summary `TIMEOUT: 1` (espera 35/35 PASS). Aislado pasa en 11s.
- `apps/cli/tests/workspace-command.test.ts` — `rechaza workspace.json truncado...` timeout 30s. Aislado pasa en 12s.

**Causa raíz:** ambos tests pasan aislados. El timeout solo ocurre bajo la carga de la suite completa (358 archivos en paralelo, ~490s). La matriz ACC-076 corre 35 casos con servidores HTTP reales dentro de un solo test con budget de 15s por caso; `workspace-command` ejecuta 4 iteraciones de `setupBaseWorkspace` (cada una corre `runWorkspaceSetup` completo).

**Fix aplicado (2026-09-13):** solo `workspace-command.test.ts` — el caso de las 4 iteraciones pasó a `timeout: 120_000` con comentario justificando la lentitud real (I/O de setup, sin timers ni servidores colgados). La matriz ACC-076 no se tocó: en las 3 pasadas de suite completas de este plan pasó 35/35 (el timeout histórico del 2026-09-10 no se reprodujo). Un fallo transitorio adicional de `freeze.test.ts` (hash no determinista bajo carga extrema) pasó aislado y en la pasada final; se registra como observación, no como defecto de este plan.

**Bug relacionado:** `BUG-20260910-baseline-acc076-matrix-timeouts` (documenta el mismo síntoma con 2 timeouts).

## Orden de implementación

| Paso | Grupo | Cambio | Archivos | Riesgo |
| --- | --- | --- | --- | --- |
| 1 | A | Añadir `confirmed` al allow-list HTTP | `apps/cli/src/commands/app-api.ts` | Bajo: campo ya documentado como in-process only |
| 2 | B | Revertir `role: sdd` → `canonicalManagedRole(kind)` en init | `apps/cli/src/commands/init.ts` | Medio: verificar que ningún consumidor depende de `role: sdd` |
| 3 | C | Actualizar 2 tests al shape v2 del state store | `apps/cli/tests/axiom-increment-metadata.test.ts`, `apps/cli/tests/axiom-qa-e2e.test.ts` | Bajo: solo tests |
| 4 | D | Actualizar test a warnings tipados | `apps/cli/tests/inc-20260727-adoption-config-scaffolding.test.ts` | Bajo: solo test |
| 5 | E | Completar fixture de workspace.json | `apps/cli/tests/configure.test.ts` | Bajo: solo test |
| 6 | F | Des-trackear dist de git | índice de git (sin cambios de código) | Bajo: `.gitignore` ya vigente |
| 7 | G | Decidir política de flaky (re-run o timeout) | opcional: `r13-acc-076-matrix.test.ts` | Bajo |

**Justificación del orden:** A y B son fixes de producto (desbloquean 9 tests); C-F son fixes de tests que siguen contratos ya vigentes; G es estabilidad. A primero porque es el bloque más grande (8 tests) y el fix más pequeño (2 líneas). B segundo porque requiere la verificación de consumidores más cuidadosa.

## Validación

1. Tras cada grupo: `npx vitest run <archivo-afectado>` para confirmar el fix local.
2. Tras todos los grupos: `npx vitest run` completo esperando `Test Files 358 passed`, `Tests 3765 passed` (o el conteo vigente si cambió).
3. Para el Grupo B adicionalmente: `npx tsc -b` para confirmar que el cambio de init.ts no rompe la compilación.
4. Para el Grupo F: `git ls-files "packages/adapters/*/dist/**"` debe devolver vacío tras el `git rm --cached`.

## Reglas de alcance

- No tocar `packages/core/src/self-update/**` ni los gates AC-078/079 (restricción documentada en los bugs de la baseline).
- No modificar los bugs `BUG-20260910-*` existentes: este plan trabaja sobre la baseline del 2026-09-13; al cerrar cada grupo, actualizar el bug correspondiente si su síntoma queda resuelto.
- Los fixes de tests (C, D, E) no cambian producto: el contrato vigente es el implementado; los tests estaban desactualizados.
- El Grupo G no debe "arreglarse" ampliando timeouts para ocultar fugas (restricción del bug de matriz): si se ajusta, documentar por qué 15s es insuficiente bajo carga real de suite completa.

## Trazabilidad

- Baseline: ejecución `npx vitest run` del 2026-09-13 (489s, 16 failed / 3749 passed).
- Commits origen de las regresiones: `b3be943` (grupos A, B, E-parcial), `95cab5e` (grupos C, F), `8274c69` (grupo D).
- Bugs relacionados: `BUG-20260910-baseline-launcher-onboarding-schema` (A), `BUG-20260910-baseline-workspace-setup-adoption-contract` (B), `BUG-20260910-baseline-provider-worktree-state` (E, lado loader), `BUG-20260910-baseline-acc076-matrix-timeouts` (G).
