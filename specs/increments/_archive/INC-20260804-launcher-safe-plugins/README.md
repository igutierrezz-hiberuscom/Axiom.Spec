# Increment: Plugins externos seguros en el launcher

Status: closed
Date: 2026-08-04
Action: ACC-008 / R-12
Dependencies: INC-20260804-launcher-onboarding-migration, INC-20260804-launcher-sdd-lifecycle
Freeze previo al apply, conservado como evidencia histórica:
`8073fda966c740b2f5392f5b13c9213cdb075b5e24b3ad32790529d87e215e04`.
El cierre gobernado usa un freeze final emitido sobre esta versión, después
del hardening de review.

## Goal

Conectar plugins declarativos externos al launcher con un contrato seguro de
descubrimiento, preview y ejecucion. Las acciones deben distinguir lectura,
mutacion local y mutacion externa; las mutaciones requieren confirmacion y
solo pueden delegar en handlers internos registrados. Azure DevOps debe
seguir siendo opcional y conservar secretos fuera de la UI.

## Context

`app-plugins.ts` ya descubre JSON declarativo con warnings tolerantes y
`app-plugins-azure-devops.ts` declara acciones. El runtime ya tiene
`external-sync azure-devops`, `IWorkItemTracker`, `NullTracker`,
`AdoWorkItemTracker` y endpoints ADO especializados con gates. El hueco de
ACC-008 es la ejecucion generica del contrato: un campo `command` no debe
convertirse en shell arbitrario ni en una ruta que pueda ejecutar cualquier
texto enviado por un plugin.

## Scope

- Refinar el schema de plugin con `schemaVersion: 1`, capacidades, clase de
  accion, `handler` declarativo y campos de entrada. `command` queda como
  texto informativo/compatibilidad y nunca concede permiso de ejecucion.
- Separar las etapas `schema -> discovery -> handler resolution -> execution`.
  Discovery conserva origen, warnings y estado de configuracion; resolution
  consulta un registro estatico allowlisted y no interpreta comandos como
  procesos, rutas o scripts.
- Mantener `GET /api/projects/:id/plugins` compatible en su forma existente
  (`plugins` + `warnings`) y exponer en el mismo catalogo el estado
  ejecutable/no ejecutable de cada accion sin afirmar que una declaracion se
  aplico.
- Exponer preview read-only, confirmacion obligatoria para
  `local-mutation`/`external-mutation`, y resultado uniforme con errores,
  enlaces externos y ausencia de secretos.
- Conectar Azure DevOps al handler real existente, incluyendo create/list/
  mapping y `externalRefs` cuando la ruta canonica ya lo produzca, sin
  duplicar el tracker ni meter credenciales en el front.
- Cubrir plugins ausentes, malformados, duplicados, schema no soportado,
  comandos desconocidos, handlers no permitidos, acciones read-only y
  mutaciones confirmadas/no confirmadas.

## Non-goals

- No permitir shell generico, `child_process.exec`, scripts arbitrarios,
  paths ejecutables ni comandos proporcionados por la UI.
- No convertir plugins en dependencias obligatorias del core.
- No añadir providers nuevos ni una API completa de Jira/Confluence.
- No mover secretos al registry, JSON del plugin, localStorage o respuestas
  HTTP.
- No retirar la TUI ni cambiar el endpoint existente de launcher para sus
  acciones internas.

## Contract and handler matrix

El input se valida como datos antes de descubrirlo. Un plugin con schema
ausente se trata como legado compatible solo para discovery; sus acciones no
son ejecutables salvo que aporten un `handler` conocido. Un schema no
soportado, un action malformado o un duplicado genera warning y no aborta el
resto del catalogo.

| Action | kind | handler allowlisted | command compatible | Gate |
| --- | --- | --- | --- | --- |
| create work item | external-mutation | `external-sync.azure-devops.create` | `axiom external-sync azure-devops create` | preview + `confirmed: true` |
| list work items | read | `external-sync.azure-devops.list` | `axiom external-sync azure-devops list` | read-only |
| show mapping | read | `external-sync.azure-devops.mapping` | `axiom external-sync azure-devops mapping` | read-only |
| register mapping | local-mutation | `external-sync.azure-devops.mapping-set` | `axiom external-sync azure-devops mapping-set` | preview + `confirmed: true` |

La resolucion exige que `handler` y la forma declarativa de `command` sean
conocidos por el registro estatico; no se tokeniza ni se ejecuta el string.
Un command con metacaracteres, path, binario, flags no registrados o un
handler desconocido se rechaza antes de tocar tracker, filesystem de mapeos,
red o cualquier runner.

La proyeccion HTTP usa DTOs explicitas: no serializa `sourcePath` ni
propiedades desconocidas del plugin/accion, las opciones se reducen a
`value`/`label`, y `command` solo aparece cuando el handler estatico resuelve
con clase y etiqueta exactas. Los mensajes del tracker se redactan y las
URLs de `externalRefs` se limitan a esquema, host y pathname, sin userinfo,
query ni fragmento.

## Acceptance criteria

- [x] Un plugin valido se descubre con schema versionado y acciones
      clasificadas; malformados, schema desconocido y duplicados producen
      warnings, no crash.
- [x] Toda accion ejecutable se resuelve mediante un handler allowlisted; una
      accion con `command` desconocido, shell metacharacters, path, script o
      handler no registrado se rechaza sin ejecucion.
- [x] Read-only genera resultado sin confirmacion y no muta; mutacion local o
      externa siempre devuelve preview y exige `confirmed: true`.
- [x] Preview no hace writes ni llamadas externas; la confirmacion delega en
      el handler canonico y devuelve resultado, error y referencia externa
      cuando existe.
- [x] Azure DevOps usa tracker/configuracion/secrets existentes, funciona en
      `kind: none` sin red y en `kind: ado` con seams/fakes de test.
- [x] Ninguna credencial aparece en UI, logs de respuesta o artefactos del
      plugin; el handler recibe secretos solo por sus ports existentes.
- [x] El launcher muestra acciones, estado configurado y resultado de forma
      honesta, sin afirmar que un plugin no ejecutable se aplico.
- [x] Build, pruebas focalizadas de plugin/app/launcher/external-sync/tracker-
      ADO y el E2E disponible pasan.

## Risks

- Un dispatcher demasiado general puede reintroducir ejecucion arbitraria
  aunque el schema parezca declarativo.
- El plugin filesystem puede intentar colisionar con un built-in o falsificar
  un id; el orden y la validacion deben ser deterministas.
- La distincion preview/execute debe mantenerse tambien en endpoints
  especializados para no crear una segunda ruta insegura.
- Cambiar el README después del freeze invalida su `specsHash`; por eso el
  freeze final se emite después de cerrar la documentación y el receipt
  `verify` se conserva junto al incremento.

## Open questions

No hay preguntas bloqueantes. Se elige un registro estatico de handlers
allowlisted dentro del runtime y se rechaza cualquier accion que no tenga uno,
en vez de inferir ejecucion desde el string `command`.

## Decisions

- `handler` es la unica autoridad de ejecucion; `command` se valida como
  etiqueta compatible y se conserva para preview/diagnostico, pero nunca se
  tokeniza ni se pasa a un runner.
- El dispatcher de plugins vive separado del catalogo de acciones internas y
  conserva dos entradas: `POST /api/projects/:id/plugins/execute` y la forma
  compatible `pluginId/actionId` dentro de `POST /launcher/execute`.
- `create` es `external-mutation`, `mapping-set` es `local-mutation` y
  `list/mapping` son `read`; todos reutilizan `runExternalSync` y sus ports.
- Azure DevOps sigue opt-in. El catalogo puede mostrar el bridge built-in como
  `no configurado` sin crear tracker ni hacer red; la ejecucion usa
  `NullTracker` o `AdoWorkItemTracker` segun la configuracion existente.

## Assumptions

- Azure DevOps sigue siendo opt-in mediante `tracker.json`; el default es
  `NullTracker` sin red.
- Los handlers reciben un identificador de proyecto y valores validados,
  pero nunca un comando de shell construido por el cliente.
- La UI puede conservar `command` como texto informativo, pero no como fuente
  de permisos; una accion declarativa no ejecutable se muestra como tal.
- La forma `plugins` + `warnings` de `GET /api/projects/:id/plugins` sigue
  siendo consumible por clientes existentes.

## Implementation notes

El worker separa schema y discovery en `app-plugins.ts`, resolution y
catalogo/puente en el launcher, y execution en handlers internos que
reutilizan `runExternalSync`, `apiAdo*`, `IWorkItemTracker`, `NullTracker` y
`AdoWorkItemTracker`. Los tests usan handlers/fakes y verifican que preview no
llama red ni escribe, que un command no registrado no alcanza tracker ni
runner, y que `kind: none` permanece local-only.

## Validation

Ejecutado por el worker:

- `npm run build` en `Axiom`: correcto.
- `npx vitest run apps/cli/tests/app-plugins.test.ts apps/cli/tests/app-launcher.test.ts`
  : cobertura de DTO, redaccion, tipos/opciones, envelope y gates correcta.
- Bateria focalizada de app/API, launcher, external-sync, launcher ADO,
  `packages/tracker` y `packages/tracker-ado`: 20 archivos, 178 tests
  correctos.
- `npx vitest run apps/cli/tests/e2e/launcher.e2e.test.ts`: 1 E2E correcto,
  artefacto local con `externalRefs` vacio.
- `node --check` para `static/launcher/launcher.js` y `transport.js`,
  `get_errors` y `git diff --check`: correctos.

## Result

Implementado el contrato seguro `schema -> discovery -> resolution ->
execution`: schema v1/capacidades, discovery tolerante y ordenada, registro
estatico de handlers ADO, normalizacion de campos, gates de preview/
confirmacion, endpoint plugin-scoped, alias compatible del launcher y tarjeta
visible de plugins. Las respuestas del catalogo y bridge se proyectan con DTO
estricta, validacion de tipos/opciones, redaccion de errores/URLs y sin
detalles de adaptador ni secretos. La re-review N=1 verifico REVIEW-001..005
como `verified`; los riesgos residuales son la naturaleza basada en patrones
del redactor y la falta de un E2E contra un plugin shadow malicioso, sin
hallazgo abierto en el alcance.

El freeze previo queda como evidencia histórica porque este README y el
runtime cambiaron después del hash congelado. La implementación, la review
técnica y la integración canónica están completas; el incremento queda
`closed` y el cierre conserva el freeze final y el receipt `verify` emitidos
sobre esta versión.

## General spec integration

Integrado en `specs/00..08`, `context/integrations/01-capabilities-providers-
y-toolchain.md` y `context/operations/01-instalacion-y-onboarding.md`: plugins
opcionales, allowlist de handlers, validacion estricta, redaccion y gates de
confirmacion.
