# 05 Interfaces Operativas

## Surfaces principales

1. CLI del runtime (`axiom`, entry point `apps/cli/dist/index.js`, instalado vía `scripts/install-global.mjs` como shim/`npm link` único en el PATH del usuario);
2. TUI operativa (`axiom tui`, `@axiom/tui`);
3. adapters que conectan Axiom con harnesses o IDEs (`@axiom/adapters-*`);
4. manifests/YAML que describen la configuración y topología del proyecto (`axiom.yaml`, `axiom.config/*.yaml`, `topology.yaml`).

## CLI: comandos documentados en profundidad

`init`, `join`, `configure`, `sync`, `start`, `audit`, `doctor`, `upgrade`, `tui`, `model` (`show`/`set`/`unset`/`reset`/`validate`), `components` (`list`/`show`/`install`/`uninstall`/`restore`), `skills` (`list`/`refresh`/`drift`). Fuente: `Axiom/docs/cli/*.md`.

## CLI: comandos presentes en código sin documentación operativa equivalente

`apps/cli/src/commands/` tiene 36 ficheros (~16.400 líneas) según el conteo original de esta entrada — desactualizado (ver nota de conteo en la sección de arriba sobre `intent`; un re-audit completo del conteo/LOC no es parte de este incremento). Además de los documentados arriba existen: `app`, `app-api`, `app-plugins`, `app-plugins-azure-devops`, `axiom-bug`, `axiom-increment`, `axiom-plan`, `axiom-qa-e2e`, `axiom-role`, `capability`, `context`, `gateway`, `mcp`, `memory`, `projects`, `qa-archive-gate`, `repo`, `roles`, `self-update`, `toolchain`, `topology`. (`intent` fue removido de esta lista: `INC-20260710-honesty-and-toolchain-states` eliminó `apps/cli/src/commands/intent.ts` por dead code confirmado — ver la sección de arriba.) Estos se mencionan de forma dispersa en los documentos de incremento (`0019`–`0030`) pero no tienen una página propia bajo `docs/cli/`. Tratar su comportamiento como verificable en código, no como contrato documentado establecido, hasta que se audite comando por comando.

## TUI (`axiom` / `axiom tui`)

Punto de entrada: `axiom` sin subcomando abre la TUI (acción por defecto de Commander en `apps/cli/src/index.ts`, reusa `runTuiCli`); `axiom tui` es equivalente. Los subcomandos, `--help` y `--version` matchean antes y quedan sin cambios.

Con proyecto Axiom resuelto: menú operativo de 6 items (configurar, sincronizar, diagnóstico, upgrade, model routing, salir; 7 con "reparar instalación" — ver sección del roadmap más abajo). Para mutaciones (`configure`, `sync`, `upgrade`) muestra preview read-only, pide confirmación y muestra un post-run summary con restore point y follow-ups. Exige TTY.

Sin proyecto Axiom (y con TTY): en vez de abortar con `projectNotFound`, la TUI abre el menú de bootstrap `setup` (ver "wizard guiado de `init`" más abajo). En no-TTY se preserva `projectNotFound` + exit 1 (CI-safe). Fuente: `Axiom/docs/cli/tui.md`, package `@axiom/tui`.

## Adapters (surfaces por IDE/CLI externo)

6 adapters con package dedicado y contrato común `generate<Target>Config(args) → Promise<Result<GeneratorResult, AdapterGeneratorError>>`: `opencode` (`multi-mode`), `claude-code` (`single-mode`), `github-copilot`, `vscode`, `cursor`, `litellm` (los 4 últimos `fallback-only`). 3 targets adicionales declarados sin adapter dedicado (`copilot-vscode`, `antigravity`, `visual-studio-2026`) caen al default del IDE. Fuente: `Axiom/packages/adapters/README.md`.

## Documentación operativa navegable

`Axiom/docs/README.md` es el índice hacia manuales de instalación, configuración, uso diario, CLI, ficheros generados y troubleshooting — ya existe y está mantenido, no es un artefacto a construir desde cero.

## Regla

Las interfaces operativas se implementan en `Axiom/`, pero su comportamiento esperado se define primero en `Axiom.Spec/`. Regla vigente pero con brecha real: parte de la superficie de comandos (sección anterior) ya existe en código sin haber pasado primero por una spec formal en `Axiom.Spec/`.

## Superficie de comandos ampliada por el roadmap de rediseño (cerrado)

La superficie realmente alcanzable y primaria coincide con la forma preferida `axiom increment ...`, `axiom bug ...`, `axiom plan ...`, `axiom role ...` (registrados en `apps/cli/src/index.ts`), más:

- `axiom-adr`/`axiom-decision` (`create`/`link-plan`/`link-increment`/`list`, más `axiom-adr supersede <old-id> <new-id>`);
- `axiom index rebuild|validate|list` (todos los tipos de artefacto, `--json`);
- `axiom validate changes --project <id> --plan <planId>` (validación de write-scope);
- `axiom bootstrap from-code --level minimal|basic [--role <role>]` y `axiom bootstrap from-legacy-sdd <path> [--dry-run]`;
- `axiom repair` (general, top-level, distinto de `axiom toolchain repair` y `axiom mcp repair`).
- `axiom mcp serve --kind <sdd|spec> --project-root <path> [--home-dir <path>]` (`INC-20260708-mcp-runnable-server`): lanza el server MCP ejecutable `@axiom/mcp-server` (JSON-RPC 2.0 sobre stdio delimitado por saltos de línea) exponiendo las herramientas `sdd.*` (7) o `spec.*` (10) de `@axiom/mcp-tools` según `--kind`. Es el proceso servidor real detrás de los identificadores `sdd-mcp-server`/`spec-mcp-broker` y el comando al que apunta el `command`/`args` generado en `.axiom/mcp.yml` (ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md)). Se añade al grupo `mcp` existente (junto a `list`/`validate`/`repair`/`inventory`), registrado desde un fichero app-owned (`apps/cli/src/commands/mcp-serve.ts`).

Ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md) para el contrato funcional completo de cada uno.

### Superficie CLI añadida por la tanda INC-20260708-* (providers, memoria/aprendizaje, autoskills, operaciones incrementales)

Comandos nuevos entregados por esta tanda, todos app-owned (registrados directo en `apps/cli/src/index.ts`, no vía `@axiom/cli-commands`, para evitar el gotcha de single-ownership de ese paquete):

- **`axiom configure --providers <csv>`** (`INC-20260708-wizard-configure-provider-selection`): persiste la selección de providers LOCALES habilitados para el proyecto en `.axiom-state/<projectId>/workspace.json#providers` (merge-write, creando el fichero si falta), sin alterar el resto del comportamiento de `configure`. Los ids seleccionables son exactamente `codegraph`/`serena`/`graphify`/`engram` (los 4 que mapean 1:1 a una tool local opcionalmente instalable; `filesystem` es always-on baseline). **No** toca `axiom.config/providers.yaml` (registry canónico cerrado de 7 ids, schema-locked por `CC-001`/`CC-003`) — ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md) y [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md).
- **`axiom learn capture [--from-audit] [--text "..."] [--limit N]`** y **`axiom learn list [--query ...] [--limit N]`** (`INC-20260708-continuous-learning`): captura/recall de lecciones sobre el backend real de `@axiom/memory` (`resolveMemoryBackend`), mismas convenciones que `axiom memory` (`resolveMemoryScope`, UI en español, split `run*`/`register*`). Ver [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md).
- **`axiom skills suggest [--repo <path>] [--apply]`** (`INC-20260708-autoskills-wizard-phase`): detección de stack + sugerencia de skills curadas para un repo (default cwd); sin `--apply` solo imprime (read-only), con `--apply` instala el set sugerido en `axiom.config/skills-index/<roleId>.yaml#available` de ese repo. Se une al grupo `skills` existente (`list`/`refresh`/`drift`); usable contra cualquier repo (no solo el proyecto activo).
- **`axiom repo add`**, **`axiom adapter add <target>`**, **`axiom provider add <id>`**, **`axiom role add <roleId> --path <path>`** (`INC-20260708-incremental-operations`): las 4 operaciones incrementales ADD-only, idempotentes y no-clobber, que resuelven el proyecto desde cwd y reusan el motor multi-repo (`runWorkspaceSetup` + helpers exportados). `repo add` escribe el `axiom.yaml` del repo nuevo con `paths` recíprocos y refresca los de los hermanos, actualiza `topology.yaml`, registra en `projects.yml` y genera adapters/MCP/skills/rules para los adapters habilitados del proyecto; `adapter add` persiste en `workspace.json#adapters` y regenera ese adapter en todos los repos; `provider add` persiste en `workspace.json#providers` (validación best-effort vía `buildProjectProviderRegistry`, sin spawnear nada); `role add` es un wrapper de `repo add --kind role` + asignación de rol en topología. Los REMOVE quedan diferidos (ver NFR-AXM-015 en [02_Requisitos_No_Funcionales.md](02_Requisitos_No_Funcionales.md)). Registrados desde `apps/cli/src/commands/workspace-incremental.ts`.

### `axiom workspace` — setup headless completo + reparación/reuso granular (`INC-20260710-workspace-command-parity`)

Hasta este incremento, `runWorkspaceSetup` (el motor multi-repo detrás del wizard de la TUI, ver la sección de abajo) no tenía ningún caller headless: no había forma de scriptear un install completo, correrlo en CI, ni reparar/reusar una PARTE de un install ya existente sin re-correr todo el setup. `axiom workspace` (`apps/cli/src/commands/workspace.ts`, registrado en `index.ts` justo después de `registerWorkspaceIncremental`) cierra ese hueco, 100% no-interactivo (sin TTY, sin prompts — todo por flags):

- **`axiom workspace setup`**: equivalente headless COMPLETO del wizard de la TUI. Mapea flags a un `WorkspaceSetupSpec` (`buildWorkspaceSetupSpecFromFlags`, el análogo headless de `tui.ts`'s `buildWorkspaceSetupSpecFromWizard`) y llama al MISMO `runWorkspaceSetup`, sin cambios. Flags: `--name`/`--spec-path` (requeridos), `--control-path` (default cwd), `--role <name>:<path>` (repetible — roles de equipo dinámicos, 1..N, sin enum fijo, Decision D5), `--adapters`/`--profile`/`--overlay`/`--providers <csv>`, `--no-register`, `--json`.
- **Sub-comandos granulares, re-ejecutables sobre un install YA EXISTENTE** (resuelven el proyecto desde cwd con el mismo `resolveExistingProject` que `repo add`/`adapter add` ya usan): `axiom workspace spec-base --spec-path <dir>` (→ `scaffoldSpecRepoBase`, no requiere proyecto registrado), `axiom workspace adapters [--path <repo>] [--adapters <csv>]` (→ `generateWorkspaceAdapters`; nunca toca `workspace.json` ni la config MCP nativa — para eso siguen existiendo `adapter add`/`mcp-config` respectivamente), `axiom workspace skills [--path <repo>]` (→ `scaffoldSddSkills`/`scaffoldCodeRepoSkills`, sin gatear en "recién creado" — a propósito, es el punto de un comando de reparación), `axiom workspace rules [--path <repo>]` (→ `scaffoldRules`), `axiom workspace mcp-config [--path <repo>]` (→ `buildWorkspaceMcpServers` + `writeWorkspaceMcpConfig` + `writeWorkspaceNativeMcpConfigs`), `axiom workspace config-scaffold` (→ `scaffoldArchitectDeclarations`, `INC-20260710-architect-member-handoff`; ver más abajo).

Reparto de responsabilidad, deliberadamente sin superposición: los comandos ADD-only (`repo add`/`adapter add`/`provider add`/`role add`, más `roles register`) agregan algo NUEVO a un install; `axiom workspace <granular>` re-aplica/repara una parte de lo que YA EXISTE (sin agregar nada nuevo al registro/`workspace.json`). Ninguno de los dos grupos duplica lógica de scaffolding: ambos son wrappers finos sobre los mismos step fns del motor (`workspace-setup.ts`/`workspace-adapters.ts`/`workspace-skills.ts`/`workspace-rules.ts`/`workspace-mcp.ts`/`workspace-spec-base.ts`).

**`INC-20260710-architect-member-handoff`** cerró un gap confirmado en vivo: `runWorkspaceSetup` escribía `.axiom/mcp.yml` (canónico, GENERADO) pero nunca `axiom.config/mcp-manifest.yaml` ni `axiom.config/toolchain-catalog.yaml` — las declaraciones COMMITTEADAS que `axiom member install` (abajo) necesita leer. Un miembro clonando un proyecto bootstradeado vía `workspace setup` recibía CERO config MCP nativo. Ahora `runWorkspaceSetup` scaffoldea ambos archivos como paso best-effort, no-clobber (`scaffoldArchitectDeclarations`, `workspace-config-scaffold.ts`; nunca sobreescribe uno ya committeado, incluyendo el que el repo `Axiom` producto ya shippea sembrado a mano); `axiom workspace config-scaffold` es el comando granular equivalente para repararlo en un install pre-existente que nunca los tuvo.

`axiom init --layout installed-multi-repo` (RF-AXM-006, single-repo, sin cambios de comportamiento) solo escribe el `axiom.yaml` del repo que lo invoca — nunca crea el repo de spec que su propio `paths.specification` referencia. Desde este incremento, el output de `registerInit` imprime un puntero explícito a `axiom workspace setup` (con el comando exacto) cuando `layout === 'installed-multi-repo'`, en vez de dejar ese hueco en silencio.

### `axiom member install` y `axiom bindings` — onboarding headless de miembros del equipo (`INC-20260710-per-member-install`)

Separación explícita de responsabilidades: el ARQUITECTO corre `axiom workspace setup`/`axiom init` UNA VEZ y commitea el resultado (ver "Separación de responsabilidades" en [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)); cada MIEMBRO del equipo, tras clonar esos repos, corre lo siguiente en SU máquina:

- **`axiom member install --member <id> [--bind <repoId>:<path>]... [--no-register] [--home-dir <path>] [--path <repo>] [--json]`** (`apps/cli/src/commands/member-install.ts`, registrado junto a `topology`/`roles`): onboarding IDEMPOTENTE en un solo comando.
  1. Registra al miembro (reusa `runJoin`; sintetiza `init.json` local primero si falta — necesario para que el gate de `join-command` no bloquee proyectos bootstradeados vía `workspace setup`, ver 03).
  2. Bindea cada repo lógico de `topology.yaml` a un path real: `--bind` explícito gana; si no, intenta auto-detectar el sibling por el `ref` declarado (existe en disco → resuelto, sin persistir nada nuevo); si no existe, lo reporta como `unbound` con la remediación exacta (`axiom bindings set ...`).
  3. Materializa el config MCP nativo (`.mcp.json`/`.cursor/mcp.json`/`.vscode/mcp.json`/`opencode.json`, vía los mismos writers de `native-mcp-config.ts`) a partir de los ids declarados en `axiom.config/mcp-manifest.yaml` (committeado — scaffoldeado por `axiom workspace setup` desde `INC-20260710-architect-member-handoff`, ver arriba), con un launch command resuelto para ESTA máquina (`resolveMcpLaunchCommand()`, ver 03).
  4. Activa el estado local (`.axiom-state/<projectName>/toolchain/<id>/`) de cada utilidad que el proyecto habilitó en `axiom.config/toolchain.yaml` (vía `repairTool`, `@axiom/toolchain`) e imprime la guía de instalación externa exacta cuando se conoce una (nunca inventa un comando).
  5. Imprime un resumen que distingue explícitamente SHARED (ya en git, sin tocar) de PERSONAL (recién materializado en esta máquina).
- **`axiom bindings show|set|remove`** (`apps/cli/src/commands/bindings.ts`): CRUD acotado sobre `.axiom-state/local/topology-bindings.yaml` para arreglar UN SOLO repo lógico (e.g. porque un miembro movió una carpeta) sin re-correr `member install` completo. `show` resuelve cada repo lógico → path efectivo + si existe en disco; `set --repo <id> --path <path>` persiste un override; `remove --repo <id>` lo quita. Ambos read-only/mutación simple, sin `runOrchestrated` (mismo patrón que `topology`/`roles`).

Ambos comandos son app-owned (registrados directo en `index.ts`, no vía `@axiom/cli-commands`) y `runMemberInstall`/`runBindings*` están separados de sus `register*` de commander (mismo patrón que el resto de la CLI) para poder testearse sin spawn del binario.

### Superficie CLI añadida por la tanda INC-20260711-* (review de write-scope + repo-affinity)

- **`axiom validate changes --plan <id> --all-repos`** (`INC-20260711-per-role-review`, extiende el `validate changes` existente): modo AGREGADO que resuelve cada `targetRepo` del plan a ruta absoluta vía `LocalBindings`, diffea y valida cada uno, y emite un reporte consolidado per-repo (✓/✗) + repos NO RESUELTOS explícitos + resultado global (exit 1 ante cualquier violación o repo no resuelto). Pensado para ejecutarse desde el repo de SPEC (`runValidateChangesAggregate` + `formatAggregateReport`, `validate-changes.ts`).
- **`axiom-role complete --no-review` / `--force`** (`INC-20260711-per-role-review`): `axiom-role complete` corre ahora un review de write-scope del `git diff` del repo de rol contra el `allowedWriteScope` del plan como gate explícito que bloquea la completion ante una violación; `--no-review` (o `--force`) omite ese gate. Ver el modelo de review en [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md).
- **Repo-affinity de `axiom-increment`/`axiom-bug`/`axiom-plan`/`axiom-role`** (`INC-20260711-repo-affinity-guard`): estos cuatro entrypoints se rechazan (exit 1) si se ejecutan desde el repo equivocado en un workspace multi-repo con roles definidos (increment/bug/plan ↔ repo de spec; role X ↔ repo del rol X) — NO-OP fuera de ese caso. Ver [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md).

### Superficie CLI y front de prompts añadidos por la tanda sdd-launcher-port (2026-07-11)

Port del sdd-launcher de KVP25 a Axiom core (P1/P2/P3/P4/PX; contexto en [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md) y [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md)):

- **`axiom scaffold increment|bug|plan`** (`INC-20260711-sdd-launcher-p1-cli-subcommands`): genera el esqueleto estructural completo delegando en el generador P0 (`axiom-increment create` es un alias, sin duplicación); `scaffold e2e` diferido (no hay clase de artefacto e2e).
- **`axiom normalize`**: canonicaliza el `status` de forma idempotente contra la tabla de vocabulario de ciclo de vida.
- **`axiom integrate`**: archiva + aplica la transición terminal (reusa `archiveArtifactDir`).
- **`axiom validate transition`**: rechaza transiciones ilegales con el error tipado `invalid-transition` y lista las legales.
- **`axiom state`**: inspector de estado actual / transiciones disponibles / recomendada.
- **`axiom external-sync azure-devops …`** (`INC-20260711-sdd-launcher-p2-tracker`): la ruta de sync ADO, ahora respaldada por un cliente real detrás de `IWorkItemTracker` (ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md)).
- **Front de prompts en `axiom app`** (`INC-20260711-sdd-launcher-p3-front-server`; core operativo, largo plazo diferido): el front re-alojado se sirve bajo `/launcher/` desde el server `axiom app` (endpoints preview/execute/`confirmed`) tras un transport shim sin APIs de VSCode — nav + formulario dinámico + preview de prompt + execute/confirm + registry funcionan. La abstracción `Launcher` tiene tres impls: `ClipboardLaunch` / `HttpLaunch` / `VSCodeLaunch`.
- **Routing de adapter** (`INC-20260711-sdd-launcher-p4-launcher`, `@axiom/launcher`): la misma acción resuelve a ids reales de comando/skill/MCP por adapter `claude-code` / `github-copilot` / `cli`, con fallback a `defaultAgentMention`.

### Git side-effects, gate de verify y canal push — tanda git-services / functional-verify / front-longtail (2026-07-11)

Superficie añadida sobre la tanda sdd-launcher-port; contexto de flujo en [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md):

- **`axiom-role start --create-branch [--commit] [--push] --confirm`** (`INC-20260711-git-services`): efecto git OPT-IN de la transición `start` — crea la rama de rol y (con `--commit`) commitea LOCALMENTE, empujando solo con `--push` explícito y `--confirm`. Sin flags el comportamiento por defecto no cambia (ningún git).
- **Gate de verificación funcional** (`INC-20260711-functional-verify`): `axiom-increment verify` y `axiom-role complete` descubren y ejecutan la validación del repo destino y BLOQUEAN ante fallo; `--no-verify`/`--force` saltan el gate y `--preview`/`--dry-run` reporta lo descubierto sin correr. `axiom-qa-e2e verify --run-validation` opta la lane QA a una corrida real (inline-noop sigue siendo el default).
- **Tools MCP de acción git** `sdd.gitRoleBranch` / `sdd.gitCommitSync` (`INC-20260711-git-services`): espejo de `sdd.transitionApply` — preview por defecto, mutan solo con `{ confirmed: true }`, nunca push salvo `{ push: true }`.
- **Canal push SSE en `axiom app`** (`INC-20260711-front-longtail`): el server emite `text/event-stream` sobre el `http.Server` existente y el shim del front lo consume con `EventSource`, degradando a fetch cuando no está disponible; añade el mapeo execute (confirm-gated) de `plan-new`/`plan-execute`, cerrando el largo plazo P3 mantenido.

### Colisión de nombrado: dos mecanismos "intent" no relacionados (uno ya eliminado)

Había una colisión de nombrado, no un router único compartido: (1)
`apps/cli/src/commands/intent.ts` (spec 0028) era un catálogo cerrado
de 3 entradas `IntentChain` (`increment-new`, `bug-new`,
`implement-role`), cada uno un wrapper literal alrededor de las
funciones de subcomando estructuradas existentes — **nunca estuvo
registrado en `index.ts`**, inalcanzable por ningún usuario pese a
tener sus propios tests unitarios (y esos tests sólo ejercitaban el
catálogo estático, nunca `runIntentCommand` en sí). (2) Los intent
commands de `@axiom/orchestrator` (`state-machine.ts`) son una unión
de 19 entradas `axiom-*-command`, cada una bloqueada por la
precondición siempre-fallida `notImplemented`; ninguna ha sido
cableada a lógica real. **`INC-20260710-honesty-and-toolchain-states`
eliminó (1)** — `intent.ts` y su test — al confirmar por audit
repo-wide que no tenía ningún consumidor fuera de su propio test
(dead code genuino, de bajo riesgo de remover). **(2) se mantiene**,
ahora anotado explícitamente en código como "no es la vía de
ejecución real" (ver `packages/orchestrator/README.md` y los
comentarios de `state-machine.ts`/`types.ts`): removerlo habría
significado reescribir tests legítimos del contrato interno de la
state machine (`gates.test.ts`/`runner.test.ts`/
`state-machine.test.ts`) sin cambiar ningún comportamiento real, ya
que ningún caller pasa esos 19 ids hoy. La vía de ejecución REAL del
workflow SDD (increment/bug/plan/role) es `@axiom/workflow`, invocada
directamente por `axiom-increment`/`axiom-bug`/`axiom-plan`/
`axiom-role` — nunca por el orchestrator. Un router opcional real (p.
ej. resucitar la idea de `intent.ts` cableada en `index.ts` como
`axiom run <action>`) seguiría siendo superficie de producto nueva
que requeriría su propio incremento — no un gap de cumplimiento.

## TUI — pantallas y flujos añadidos por el roadmap de rediseño

`router.ts`'s `MENU_ITEMS` tiene 7 entradas: configurar, sincronizar, diagnóstico, upgrade, **reparar instalación** (posición 4, entre upgrade y model routing — añadida por este roadmap), model routing, salir. `packages/tui/src/flows/configure.ts`/`upgrade.ts`/`repair.ts` son wrappers sin lógica de negocio (solo formateo de mensajes). Esto extiende, sin reemplazar, el menú de 6 items descrito en la sección "TUI" de arriba.

Las pantallas `mcp-inventory`/`memory-inventory` permanecen accesibles solo por flag de CLI, no como `MENU_ITEMS` de primera clase — promoverlas es un incremento explícitamente no iniciado (`INC-20260702-tui-menu-promote-inventory-screens`, diferido), no un hueco de esta spec.

## TUI — menú de bootstrap `setup` y wizard guiado de setup de workspace (`INC-20260705-tui-workspace-setup-wizard`)

Cuando `axiom`/`axiom tui` abre en una carpeta sin proyecto Axiom, la screen `setup` (items estáticos `SETUP_ITEMS`) ofrece: "Inicializar Axiom en esta carpeta" · "Actualizar Axiom" (self-update) · "Ver proyectos Axiom registrados" · "Salir". Elegir un proyecto de la lista de registrados hace `chdir` + abre la TUI operativa sobre él.

Elegir "Inicializar Axiom en esta carpeta" recorre un WIZARD GUIADO MULTI-REPO que prepara un entorno de trabajo completo (SDD + Spec + repos de rol, cruzados entre sí, registrados y con MCP configurado) en una sola operación. **Supersede al wizard single-repo de 6 pasos de `INC-20260703-tui-init-wizard`**: aquella acción `init` ya no llama a `runInit`, sino que ahora ensambla un `WorkspaceSetupSpec` y llama a `runWorkspaceSetup` (`INC-20260705-workspace-multirepo-setup-engine`, con generación MCP de `INC-20260705-workspace-mcp-generation`). El comando no-interactivo `axiom init`/`runInit` (RF-AXM-006) sigue siendo single-repo y sin cambios; solo cambió la acción `init` de la screen `setup`.

Orden de steps, cada uno con un default sensato pre-cargado que el usuario acepta (Enter) o cambia:

1. `name` (texto libre — del que se deriva el `projectId`);
2. `sddPath` (texto, default `.`);
3. `specPath` (texto, default `../<slug>.spec`);
4. `roles` (multi-select — cero o más roles funcionales/de implementación a integrar; opciones tomadas en vivo de `DEFAULT_PROFILES`, perfil `builder`, `activatesImplementationRoles` de `@axiom/install-profiles`, no una lista hardcodeada/duplicada);
5. `rolePath:<role>` (texto, uno por rol seleccionado, default `../<slug>-<role>` — inyectados dinámicamente, ver `expand` abajo);
6. `profile` (select);
7. `overlay` (select);
8. `adapters` (multi-select — cero o más adapters a instalar; opciones = los 8 `ADAPTER_TARGETS`; default pre-seleccionado `['opencode']`) — **supersede al step `target` `select` único** que describía la versión previa de este wizard (`INC-20260705-workspace-adapters-multiselect`);
9. un único resumen de confirmación.

El valor recogido del step `adapters` se serializa como los `value`s separados por coma en orden de opción; `buildWorkspaceSetupSpecFromWizard` los parsea en `spec.adapters` y deriva `spec.target = adapters[0]` (el primario, back-compat). La selección de adapters gobierna la generación de ficheros de adapter en TODOS los repos del workspace (control + spec + cada rol) y la proyección MCP a cada adapter MCP-capaz seleccionado — ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md) y [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md).

Cada step de path muestra en su subtítulo una nota `fs.existsSync` "(nuevo, se generará)"/"(existente, se parametrizará)" para su valor DEFAULT, y el step `adapters` avisa de que se generará config de adapter + MCP al confirmar. Solo al confirmar se llama a `runWorkspaceSetup`; en éxito se imprime un resumen de resultado (repos con su estado `created` real y autoritativo, path de topología, estado de registro, warnings) y se abre la TUI operativa sobre el repo de control. Cancelar (en cualquier step o en la confirmación) no llama a `runWorkspaceSetup` y vuelve al menú `setup` sin efectos. Si `runWorkspaceSetup` lanza, el error se captura, se imprime y se vuelve al menú `setup` sin crashear la TUI.

Implementación (capas): `@axiom/tui` (`packages/tui/src/driver.ts`) se mantiene GENÉRICO, sin conocimiento de roles/enums/negocio. Sobre las dos pantallas genéricas previas (`wizard-select`, `wizard-text`) este incremento añade:

- un nuevo tipo de step genérico `multi-select` (`WizardMultiSelectStep`, renderer puro `wizard-multi-select.ts` con `[x]`/`[ ]` por opción): elige cero o más opciones de una lista pasada por el caller; el valor recogido se serializa como string de `value`s separados por coma en orden de opción (vacío = ninguno);
- un mecanismo genérico `expand?: (answerValue: string) => WizardStep[]` en los steps: cuando un step con `expand` se responde, el driver splicea (una vez) los steps devueltos justo después del step actual — usado por el step `roles` para inyectar un step `text` por rol seleccionado.

El paquete no importa de `apps/cli` ni duplica valores de enum: `apps/cli/src/commands/tui.ts`'s `buildWorkspaceSetupWizardSteps` (reemplaza a `buildInitWizardSteps` como builder cableado en la acción `init`) arma la lista de steps con opciones/labels/defaults resueltos desde los enums de `apps/cli/src/commands/init.ts` y los roles de `@axiom/install-profiles`, y `runNoProjectBootstrap` ensambla el `WorkspaceSetupSpec` y llama a `runWorkspaceSetup`. El step `adapters` (`INC-20260705-workspace-adapters-multiselect`) reusa el tipo de step genérico `multi-select`; toma sus opciones de `ADAPTER_TARGETS` (`init.ts`, fuente única) con `defaultValues: ['opencode']`, sin nuevo tipo de step en `@axiom/tui`. El wizard queda fuera del alcance de `axiom init` (no-interactivo, sin cambios) y de `axiom repo attach`.

### Mejoras de UX y roles custom del wizard (`INC-20260705-workspace-wizard-ux-custom-roles`)

Esta tanda supersede varios puntos de la descripción de arriba (el flujo general, el orden de steps y las capas se mantienen):

- **Echo de captura genérico**: tras responder cualquier step (`text`/`select`/`multi-select`), la siguiente pantalla muestra una línea `✓ <título>: <valor>` con lo recogido en el step inmediatamente anterior, de modo que el input de texto (p. ej. el `name` del proyecto) queda visiblemente registrado. Es un mecanismo genérico de `@axiom/tui` (slot `lastCaptured: { title, value } | null` en el driver, renderizado por los tres renderers de step justo tras el header, antes del banner del step); sin conocimiento de negocio en el paquete. Para `select` se hace echo del `label` de la opción; para `multi-select`, de los labels unidos por coma (o `(ninguno)`); para `text`, del valor crudo. El primer step del wizard no muestra echo.
- **Step `roles` ahora es texto libre (roles arbitrarios), superando el multi-select fijo**: donde antes el step `roles` era un `multi-select` de una lista fija tomada de `DEFAULT_PROFILES` (`builder`.`activatesImplementationRoles`, típicamente `backend`/`frontend`/`qa-e2e`), ahora es un step `text` de nombres de rol separados por coma que acepta CUALQUIER nombre de rol y CUALQUIER cantidad (incluido cero). Los nombres se parsean, sanitizan (mismo algoritmo lowercase/colapso-a-guion que `slugifyProjectName`) y deduplican preservando el orden de primera aparición; `backend`/`frontend`/`qa-e2e` tecleados sanitizan a sí mismos (comportamiento previo intacto). El mecanismo genérico `expand` sigue inyectando un step de path por cada rol parseado — ahora dirigido por el texto libre en vez de por la serialización del multi-select. `buildWorkspaceSetupSpecFromWizard` reenvía cada rol sanitizado verbatim como `roleKey`/`functionalRoleId` de un `WorkspaceRepoSpec` (el motor ya los acepta como `string` abierto, sin enum). El tipo de step genérico `multi-select` de `@axiom/tui` **se conserva**, aún usado por el step `adapters`.
- **Defaults de path estilo hermano + rutas absolutas aceptadas**: los defaults de `sddPath`/`specPath` y de cada `rolePath:<role>` pasan de `.` / `../<slug>.spec` a rutas hermanas derivadas del nombre: `../<slug>-sdd`, `../<slug>-spec`, `../<slug>-<role>`. Sus subtítulos documentan explícitamente el escape hatch de path absoluto ("Relativa a esta carpeta o una ruta absoluta"); `path.resolve(cwd, <ruta absoluta>)` devuelve la ruta absoluta sin cambios (semántica nativa, verificada por test con una ruta tmp real de Windows). Limitación estructural preexistente conservada: la lista de steps es dato estático construido una vez antes de correr el wizard, así que los defaults de `sddPath`/`specPath` usan el slug del basename de la carpeta (no la respuesta viva del step `name`); los defaults de path por rol SÍ usan la respuesta viva de `name` porque `expand` es un closure invocado en tiempo de confirmación.
- **Subtítulos explicativos de `profile`/`overlay`**: ambos steps `select` ganan subtítulos que explican su efecto real (redactados desde los campos reales de `default-profiles.ts`, no de una feature inventada): `builder` habilita `code.*` + roles de implementación vs. `product-owner` limitado a spec/SDD con `code.*` bloqueado; `local-only` = todo local sin gateway (recomendado para dev), `standard` = gateway opcional, `enterprise` = gateway requerido para la ruta primaria + gobierno/compliance ampliado.

### Step `providers` (multi-select) — INC-20260708-wizard-configure-provider-selection

El wizard gana un step `providers` (`multi-select`, colocado tras `adapters`, reusando el mismo tipo genérico de step de `@axiom/tui`, sin preselección por defecto) que ofrece los 4 providers LOCALES seleccionables (`codegraph`/`serena`/`graphify`/`engram`). La selección fluye vía `buildWorkspaceSetupSpecFromWizard` a `WorkspaceSetupSpec.enabledProviders` y `runWorkspaceSetup` la persiste en `.axiom-state/<projectId>/workspace.json#providers` — la MISMA elección que `axiom configure --providers` escribe fuera del wizard. Los providers elegidos quedan cableados (init-on-setup best-effort para code-intel, probe best-effort para engram) y disponibles a `buildProjectProviderRegistry`/las MCP handlers. Mata la "false breadth": un proyecto fresco solo declara lo que el usuario eligió, sin implicar más amplitud. Ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md) y [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md).

### Fase final del wizard: autoskills por repo de código — INC-20260708-autoskills-wizard-phase

Tras un `runWorkspaceSetup` exitoso (antes de reabrir la TUI operativa), el wizard corre una fase final OPT-IN de autoskills, uno-por-uno por CADA repo de código (`kind: 'role'`) recién creado: detecta su stack (`detectStack`, reusa la inferencia por marcadores de `INC-20260708-rules-layer` más una lectura best-effort de `package.json` para `react`/`nextjs`/`express`) y, solo si hay al menos una skill sugerida (nunca un prompt vacío), lanza un `runTuiDriver` separado con un único `multi-select` de las skills sugeridas para ese repo. Confirmar instala las elegidas en `axiom.config/skills-index/<roleId>.yaml#available` de ese repo (`installSuggestedSkills`, la misma ruta que `axiom skills suggest --apply`); declinar (nada marcado) es un no-op para ese repo. Best-effort estricto (el fallo de un repo no afecta a los demás ni al setup ya completado). Se implementa como N `runTuiDriver` secuenciales porque `RunTuiArgs.wizardSteps` es estático (resuelto antes de arrancar el driver) mientras que el set de repos + stacks solo se conoce tras `runWorkspaceSetup` — sin inventar un concepto de "wizard anidado" en el driver genérico. Ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

## Adapters — MCP config por proyecto (cableada por el setup de workspace)

`generateOpencodeMcpJson`/`generateClaudeCodeMcpJson` (ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md) para el schema `mcp.yml`) están completamente implementados y testeados, y permanecen exportados en sus paquetes. Su call site vivo en la ruta de workspace fue **retirado por `INC-20260708-mcp-native-config-mapping`**: `runWorkspaceSetup` escribe `.axiom/mcp.yml` en el repo de control (fuente canónica) y, en vez de proyectar al custom-shape `.opencode/mcp.json`/`.claude/mcp.json`, emite ahora el **schema MCP NATIVO real** de cada herramienta — `opencode.json` (opencode), `.mcp.json` (claude-code), `.cursor/mcp.json` (cursor), `.vscode/mcp.json` (copilot-vscode/github-copilot) — por cada repo × cada adapter seleccionado, merge-preserving y atómico (`writeNativeMcpConfig`/`writeWorkspaceNativeMcpConfigs`). Los targets sin schema MCP nativo verificado (antigravity/visual-studio-2026/litellm) degradan a warning sin fichero. `.axiom/mcp.yml` se sigue escribiendo exactamente una vez por proyecto. Fuera de esa ruta, `runConfigure`/`sync` siguen sin llamar a los generadores (ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md)).

## Entrada MCP nativa de engram (stdio) en cada adapter — INC-20260709-engram-mcp-stdio-native-config

Cuando el provider `engram` está habilitado (`workspace.json#providers`), la MISMA emisión de config MCP nativa por herramienta descrita arriba incluye además una entrada `engram` LOCAL de stdio (`command: 'engram'`, `args: ['mcp','--project',<projectId>,'--tools','agent']`) en `.mcp.json`/`.cursor/mcp.json` (`mcpServers.engram`), `.vscode/mcp.json` (`servers.engram`, `type:'stdio'`) y `opencode.json` (`mcp.engram`, `type:'local'`). Un dev que abra cualquier repo del workspace con cualquiera de esas herramientas obtiene memoria persistente de engram por MCP stdio **sin** arrancar el daemon `engram serve`. Ningún fichero generado referencia engram por HTTP/puerto 7437/`ENGRAM_URL`. Ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md) y [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md).

## Reviewer con ledger de hallazgos — INC-20260709-review-findings-ledger

El agent de revisión (`axiom-reviewer` en el catálogo de agents materializable, y el agent bootstrap `axiom-review` de este workspace) emite ahora, además de su recomendación de cierre, un **ledger de hallazgos** estructurado (`id`/`lens`/`location`/`severity`/`status`/`evidence`) producido por una primera pasada exhaustiva loop-until-dry, y hace re-review scoped al ledger + diff del fix. El detalle del contrato y su persistencia vive en [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md).