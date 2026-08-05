# Increment: Retirada de la TUI publica

Status: closed
Date: 2026-08-04
Action: ACC-005 / R-14
Dependencies: INC-20260804-cli-commands-package-output, INC-20260804-launcher-onboarding-migration, INC-20260804-launcher-sdd-lifecycle, INC-20260804-launcher-safe-plugins

## Goal

Retirar `axiom tui` y el paquete `@axiom/tui` como superficie operativa
publica solo despues de demostrar que el launcher cubre sus capacidades y
que no queda ninguna funcion exclusiva accesible unicamente desde terminal.
Conservar los comandos CLI/Core y los `run*` que sigan siendo consumidores
validos.

## Context

`apps/cli/src/index.ts` registra `registerTui`; `commands/tui.ts` delega en
`@axiom/tui`, que mantiene driver, screens, flows, prompts, previews y tests.
El launcher ya es la superficie por defecto de `axiom app`, pero la auditoria
R-13/R-14 dejo abierta la paridad de onboarding/migracion, ciclo SDD y plugins.
Este incremento es deliberadamente final y depende de sus cierres.

## Scope

- Ejecutar una matriz repo-wide de capacidades de TUI frente a launcher, CLI
  headless y endpoints disponibles; registrar cualquier capacidad no cubierta.
- Migrar o exponer en launcher las capacidades que la matriz encuentre como
  exclusivas, si no quedaron cubiertas por los incrementos dependientes.
- Eliminar el registro de `axiom tui`, el wrapper de comando, el paquete
  `@axiom/tui`, driver, screens, flows, prompts, tests y documentacion dedicada
  cuando la matriz y las pruebas demuestren que ya no son necesarios.
- Limpiar dependencias, aliases, project references, package scripts, README,
  docs y referencias activas; mantener la logica de negocio compartida.
- Añadir una prueba de ausencia de `tui` en help/runtime y una regresion de
  launcher para las capacidades sustituidas.

## Non-goals

- No eliminar `axiom app`, comandos headless ni `@axiom/cli-commands`.
- No borrar run functions usados por CLI, launcher, MCP o tests activos.
- No preservar rutas de compatibilidad que sigan anunciando TUI publica, salvo
  una razon de migracion documentada y no operativa.
- No modificar comportamiento de lifecycle ajeno a la retirada.

## Acceptance criteria

- [x] La matriz de paridad enumera cada screen/flow/flag de TUI y su sustituto
      real, con evidencia de test o se deja el incremento pendiente.
- [x] No queda capacidad operativa exclusiva de TUI sin migrar o una decision
      explicita de no soportarla.
- [x] `axiom --help` no registra `tui`; el binario construido no puede cargar
      `@axiom/tui` por una ruta activa.
- [x] `@axiom/tui`, su driver/screens/flows/tests y docs dedicados se eliminan
      solo si no tienen consumidores activos; no se borran run functions
      compartidos.
- [x] Project references, aliases, dependencias y README/docs quedan limpios.
- [x] Build, pruebas de CLI/launcher/MCP y readiness/doctor disponible pasan.
- [x] Se confirma que `axiom app` y sus workflows principales siguen operativos.

## Risks

- La TUI contiene wizard multi-repo y modos directos que pueden no aparecer en
  la primera lectura del launcher; la matriz debe cubrir rutas y flags, no solo
  pantallas del menu.
- Eliminar el paquete antes de limpiar project references puede dejar el build
  roto aunque el CLI parezca correcto.
- La documentacion historica puede conservar la descripcion de TUI, pero debe
  marcarla explicitamente como historia y no como instruccion operativa vigente.

## Open questions

No hay preguntas bloqueantes. Si la matriz encuentra una capacidad exclusiva,
se resolvera dentro del launcher antes del borrado; si no puede cerrarse de
forma segura, el incremento queda `pending` y ACC-005 no se falsea como
completado.

## Assumptions

- El launcher y la CLI headless son las superficies de reemplazo autorizadas.
- Los tests de TUI no son razon suficiente para conservar la superficie si el
  comportamiento ya esta cubierto por tests de CLI/launcher equivalentes.
- Los artefactos `dist/` se regeneran; no se editan ni se usan como evidencia
  de consumidores fuente.

## Implementation notes

El worker debe hacer la busqueda de consumidores antes de borrar y anotar la
matriz en el README. La eliminacion debe ser un cambio final, con validacion
inmediata del build y de la ayuda compilada.

Archivos retirados: `apps/cli/src/commands/tui.ts`,
`apps/cli/tests/tui.test.ts`, `docs/cli/tui.md` y todo `packages/tui/`
(package, driver, router, renderers, screens, flows, prompts y tests).
Archivos de wiring actualizados: entrypoint CLI, `package.json`,
`package-lock.json`, `tsconfig.json`, `apps/cli/tsconfig.json`,
`vitest.config.ts`, README/docs y comentarios de ownership. Se conservaron
`@axiom/cli-commands`, runners de CLI, endpoints/launcher y handlers MCP.

## Matriz repo-wide pre-borrado (ACC-005 / R-14)

Esta matriz se construyo antes de eliminar cualquier archivo. El corte incluye
los cuatro repos del workspace (`Axiom`, `Axiom.Spec`, `Axiom.SDD` y
`Axiom.Pruebas`) y distingue consumidores fuente, documentacion operativa e
historia archivada.

### Metodo y evidencia

- Busqueda fuente y documental realizada desde cada repo con `rg`, excluyendo
  solo `node_modules`, `dist`, `.git` y `coverage`, para
  `@axiom/tui`, `axiom tui`, `registerTui`, `commands/tui`, `packages/tui`,
  `tui`, los cinco flags publicos y sus nombres de capacidades.
- La busqueda de `Axiom.SDD` no encontro consumidores operativos despues de
  actualizar su regla de autopilot para las superficies soportadas.
- La busqueda de `Axiom.Pruebas` no encontro consumidores.
- Antes de este README, las pruebas de sustitutos pasaron: 9 archivos y 120
  tests de CLI/launcher, y 5 archivos y 29 tests de MCP.
- Evidencia de launcher: `apps/cli/src/commands/app-api.ts` redirige `/` al
  launcher y expone onboarding, registry, workflows, plugins, doctor,
  prompts/ejecucion confirmada y SSE; `app-launcher.ts` y
  `app-onboarding.ts` delegan en los runners reales.
- Evidencia de CLI: `@axiom/cli-commands`, `apps/cli/src/commands/workspace.ts`,
  `projects.ts`, `topology.ts`, `skills.ts`, `memory.ts` y los comandos
  `init`/`self-update` conservan runners headless.
- Evidencia de MCP: `packages/mcp-tools/tests/cli-backed-handlers.test.ts`,
  `topology-handlers.test.ts`, `registry-handlers.test.ts`, `registry.test.ts`
  y `transition-handlers.test.ts` pasaron en conjunto.
- El freeze previo, conservado como evidencia histórica, sigue identificado por el hash
  `2db3c619f2db498add0d770e58a182a00fc7d0f9f63b466617020fa7b5248aec`.
  Este README cambió después de ese freeze; el hash previo no se usa como
  evidencia fresca de esta matriz y queda supersedido por el freeze final.

### Capacidades operativas y entradas publicas

| Capacidad TUI | Consumidores y fuente | Sustituto real en launcher | Sustituto real en CLI / MCP | Decision y evidencia |
|---|---|---|---|---|
| Registro `axiom tui`, wrapper `registerTui` y `runTuiCli` | `apps/cli/src/index.ts` y `apps/cli/src/commands/tui.ts`; `@axiom/tui` es carga transitiva | `axiom app` sirve `/launcher/` como superficie web | Comandos headless y `@axiom/cli-commands`; MCP consume runners/handlers, no TUI | Retirable despues de agregar tests de ausencia; no contiene negocio propio |
| Accion implicita `axiom` sin subcomando | `apps/cli/src/index.ts` llama `runTuiCli()` como accion por defecto | `axiom app` es la entrada web explicita | `axiom --help` y subcomandos CLI explicitos | Retirable; evita una segunda entrada oculta a TUI |
| `--projects` | `TuiArgs.projectsMode`, screen `projects`, `listProjectsV2` | `GET /api/projects`, selector de proyecto del launcher y onboarding | `axiom projects list|add|join|use`; handlers de registry MCP | Paridad operativa probada por `projects.test.ts`, launcher y tests MCP de registry |
| `--use` | `TuiCliOptions.use`, `process.chdir` y re-entry del driver | Seleccion de proyecto en el launcher mediante registry/contexto `projectId` | `axiom projects use <id>` actualiza `lastUsedAt` y devuelve `cd`; `projects` tests | El re-entry visual es UX; seleccionar/usar proyecto no es exclusivo |
| `--topology` | `initialScreen: topology`, loaders `loadTopology`/`validateTopology` | No hay panel launcher dedicado vigente; el antiguo endpoint UI fue retirado | `axiom topology show|validate`; `axiom.topologyRead` MCP | Paridad operativa probada por `apps/cli/tests/topology.test.ts` y `topology-handlers.test.ts` |
| `--model-validate` | `initialScreen: model-validate`, `runModelValidate` | Doctor del launcher cubre health/drift del proyecto; no se inventa una pantalla equivalente | `axiom model validate`; runner en `@axiom/cli-commands` y handlers CLI-backed | Paridad de operacion por CLI/shared runner; la screen es solo presentacion |
| `--components-show <id>` | `initialScreen: components-show`, `runComponentsShow` | No hay panel dedicado vigente; el launcher mantiene sus acciones declarativas | `axiom components show <id>` y runner `runComponentsShow`; tests `components.test.ts` | Paridad operativa probada; no borrar el runner compartido |
| Pantalla `menu` y menu de acciones | `router.ts` `MENU_ITEMS`; driver navega configure/sync/doctor/upgrade/repair/model-list | Launcher action catalog, forms y workflow panels | `axiom configure`, `sync`, `doctor`, `upgrade`, `repair`, `model`; MCP usa handlers compartidos donde existen | Menu ANSI retirable; cada accion tiene una superficie headless o launcher documentada |
| Pantalla `configure` y flow `runConfigureFlow` | `flows/configure.ts`, `FlowRunners.configure`, `@axiom/cli-commands#runConfigure` | Onboarding `install|join|workspace/setup` materializa configuracion; endpoint confirmado | `axiom configure`; `runConfigure` compartido | No exclusiva; tests de configure y onboarding pasan |
| Pantalla `sync` y flow `runSyncFlow` | `flows/sync.ts`, `runSync`, preview y resumen | Onboarding/launcher conserva materializacion de adapters y acciones confirmadas | `axiom sync`; `axiom workspace adapters`; runner `runSync` compartido | No exclusiva; tests de sync y launcher cubren el motor |
| Pantalla `doctor` y flow `runDoctorFlow` | `flows/doctor.ts`, `runDoctorChecks` | `GET /api/projects/:id/launcher/doctor` y doctor profundo | `axiom doctor`; `@axiom/doctor` es compartido | Paridad directa y probada por launcher doctor/CLI |
| Pantalla `upgrade` y flow `runUpgradeFlow` | `flows/upgrade.ts`, preview dry-run y `runUpgrade` | El launcher no finge un upgrade runtime; conserva lifecycle/workflow confirmado | `axiom upgrade [--dry-run]`, checkpoints, sync/doctor y runner compartido | No exclusiva; CLI es el sustituto autorizado |
| Pantalla `repair` y flow de repair | `flows/repair.ts`, `runRepair` y `@axiom/cli-commands` | No hay panel especifico; no se inventa uno | `axiom repair [--dry-run]`; runner compartido y tests `repair.test.ts` | No exclusiva; CLI mantiene la operacion segura |
| Pantalla `model-list` y flows `runModelSetFlow`/`runModelUnsetFlow` | `screens/model-list.ts`, `flows/model.ts`, `runModelSet/Unset` | El launcher puede mostrar contexto/modelo en sus formularios, pero no se afirma mutacion equivalente | `axiom model show|set|unset|reset`; runner `@axiom/model-routing` y handlers CLI-backed | Operacion cubierta por CLI/shared code; UI TUI retirable |
| Pantallas `memory-inventory` y `mcp-inventory` | Exports y renderers del package; no son entradas de `registerTui` ni flags activos del wrapper | Los endpoints exclusivos del UI viejo fueron retirados; no se reclama un panel launcher inexistente | `axiom memory` y `axiom mcp`; `@axiom/mcp-tools` registry/handlers | Renderers sin ruta publica activa; inventarios operativos conservados en CLI/MCP |
| Pantalla `setup` sin proyecto | `runNoProjectBootstrap`, `SETUP_ITEMS`: init, self-update, projects | `/api/launcher/install`, `/join`, `/workspace/setup`, `/workspace/adopt`, browse y options | `axiom init`, `axiom workspace setup`, `axiom projects`, `axiom self-update` | Paridad de bootstrap probada por `launcher-onboarding-migration.test.ts` y tests de workspace |
| Deteccion fallback cwd/ancestor/git-root/registry | `resolveProjectWithFallback` usado por `tui.ts` | Launcher recibe `projectId`/paths mediante registry y onboarding | `@axiom/project-resolution`, `axiom projects join/use`, comandos project-scoped | Logica de resolucion compartida; no es propiedad del driver |

### Interaccion, wizard y salida

| Capacidad TUI | Consumidores y fuente | Sustituto real en launcher | Sustituto real en CLI / MCP | Decision y evidencia |
|---|---|---|---|---|
| Driver readline, TTY gate, teclas `q/b`, aliases y re-entry | `packages/tui/src/driver.ts`; tests `driver.test.ts` y `tui.test.ts` | Formularios web, navegacion por proyecto y `confirmed: true`; no requiere TTY | Comandos no interactivos; errores/exit codes; MCP request/response | Presentacion retirada; el gate TTY no es una capacidad de negocio |
| Router, stack, `select` y pantallas `exit`/`placeholder` | `router.ts`, `renderers`, `exit.ts`, `placeholder.ts` | Router del frontend del launcher | Commander, JSON/text output y exit codes | Infraestructura de UI sin consumidor externo; retirable |
| Render ANSI/header/footer/menu | `render.ts` y screens de menu | HTML/CSS/JS de `/launcher/` | Texto humano/JSON de cada CLI; MCP payloads | Solo representacion; no borrar serializers ni runners |
| Prompts `promptYesNo` y pantalla `confirm` | `prompts.ts`, `screens/confirm.ts`, driver | Preview + `confirmed` en install/join/workspace/plugin/launcher execute | `--yes`, `--dry-run`, `--apply` y comandos explicitos | Paridad de control de mutaciones probada por launcher y tests CLI |
| Previews `computeConfigurePreview`, `computeSyncPreview`, `computeUpgradePreview`, gates | `flows/preview.ts`, `screens/preview.ts` | Endpoints de onboarding preview y `launcher/craft`; respuestas no mutantes | `upgrade --dry-run`, `repair --dry-run`, outputs de configure/sync y `--json` cuando aplica | Helpers TUI son presentacion; se conservan dry-runs reales |
| Summaries `buildPostRunSummary`, `summary`, `postrun` | `flows/postrun.ts`, `screens/summary.ts` | Respuestas `result`, `preview`, mensajes y estados del launcher | Output humano/JSON y exit codes de runners CLI/MCP | Solo formateo; no hay perdida de resultado operativo |
| Wizard `init`: name, role, layout, profile, overlay, target | `buildInitWizardSteps` en `tui.ts`; `wizard-select/text` | `GET /api/launcher/options` + forms install/join | `axiom init` con flags y runners compartidos | Cada valor tiene input explicito; no exclusivo |
| Wizard workspace multi-repo: paths, roles libres, profile, overlay, adapters, providers | `buildWorkspaceSetupWizardSteps`, `buildWorkspaceSetupSpecFromWizard`, `wizard-multi-select` | `/api/launcher/workspace/setup`, `/adopt`, options, browse y confirmacion | `axiom workspace setup --name ... --control-path ... --spec-path ... --role ... --adapters ... --providers ...`; tests workspace/onboarding | Paridad probada; el modo headless reemplaza prompts uno a uno |
| Mini-wizard de skills por stack y repo | `runAutoskillsWizardPhase`, `suggestSkillsForRepo`, `installSuggestedSkills` | Catalogo onboarding declara `autoskills` sin fingir instalacion automatica | `axiom skills suggest [--repo] [--apply]`; `axiom workspace skills` para reaplicar | No exclusiva: el mismo motor y tests `workspace-autoskills.test.ts` |
| Seleccion multiple, expansion dinamica y cancelacion de wizard | `wizard-select.ts`, `wizard-text.ts`, `wizard-multi-select.ts`, `driver-wizard.test.ts` | Campos select/multiselect/text y preview/confirmado en launcher | Flags repetibles/CSV (`--role`, `--adapters`, `--providers`) y comandos JSON | UX de captura sustituida; semantica de valores conservada |
| Inventario de flows/screens y barrel publico | `packages/tui/src/index.ts`, todos `src/screens/*`, `src/flows/*`, 11 suites de tests TUI | No es API del launcher | Runners en `@axiom/cli-commands`, CLI y MCP; tests dirigidos verdes | Package, screens, flows y tests eliminados tras limpiar aliases/references |

### Decision de paridad

La matriz no encontro una capacidad operativa exclusiva de TUI. Las partes que
solo existen en TUI son el driver ANSI/readline, navegacion, prompts y
formateo; las mutaciones, lecturas, bootstrap, registry, lifecycle, modelo,
componentes, topologia, memoria, MCP y skills tienen runner headless, endpoint
de launcher o handler MCP real. Por tanto ACC-005 queda **elegible para el
borrado condicionado**: se añadieron pruebas de ausencia (`axiom --help` y
runtime compilado), se limpio todo el wiring y se repitieron las pruebas de
regresion. La ausencia de capacidad exclusiva quedó confirmada por la bateria
final.

## Validation

Validacion final ejecutada:

- `npm run build`: PASS.
- `npx vitest run` dirigido de CLI/launcher/MCP: 23 archivos, 221 tests, PASS.
  Incluye `init`, `join`, `configure`, `sync`, `start`, `projects`,
  `topology`, `model`, `components`, `memory`, `mcp`, `repair`, `upgrade`,
  launcher, onboarding, ausencia TUI y los cinco archivos MCP de paridad.
- `npm run doctor`: PASS, 46/61 checks OK, 0 fallos, 3 warnings y 12 omitidos.
- `npm run readiness:first-project`: PASS.
- CLI compilada: `node apps/cli/dist/index.js --help` exit 0 sin comando `tui`;
  `node apps/cli/dist/index.js tui` exit 1 como comando desconocido.
- La prueba de retirement ejecuta directamente ambas regresiones: `axiom tui`
  devuelve `unknown command 'tui'` y la invocacion sin subcomando no inicia una
  TUI ni carga `@axiom/tui`.
- `get_errors` en todos los archivos tocados: sin errores.
- Búsquedas repo-wide: `Axiom` solo conserva literales de aserción en los
  tests negativos; `Axiom.SDD` y manuales activos no tienen referencias.
- `git diff --check`: sin errores de whitespace; solo warnings de conversión
  LF/CRLF del working tree de Windows.

## Result

Paridad operativa confirmada y documentada en la matriz. La superficie pública
de TUI fue retirada sin borrar runners compartidos. El incremento queda
`closed`: paridad, limpieza, validación, review e integración canónica
completadas. El cierre conserva el freeze final y el receipt `verify` emitidos
sobre esta versión; el archivado físico es la operación final del orquestador.

### Clasificacion de fallos

- Fallos introducidos por este incremento: ninguno.
- Warnings del doctor: preexistentes del fixture/repositorio actual (scope de
  spec ausente, scope runtime no declarado y `axiom.yaml` legacy v1); no están
  relacionados con el retiro de TUI.
- El primer intento del test de ausencia encontró un junction stale de npm;
  se eliminó el artefacto generado y la batería final quedó verde.

## General spec integration

La integración final del lote ya reconcilió los claims activos de TUI en
`specs/00..08` y `context/**`; la historia se conserva solo en secciones
marcadas como históricas. El re-freeze final y el archivado físico siguen
siendo gates del orquestador para esta preparación de archivo.
