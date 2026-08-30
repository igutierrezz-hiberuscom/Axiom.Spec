# 05 Interfaces Operativas

## Surfaces principales

1. CLI del runtime (`axiom`, entry point `apps/cli/dist/index.js`, instalado vía `scripts/install-global.mjs` como shim/`npm link` único en el PATH del usuario);
2. launcher web local (`axiom app`, servido bajo `/launcher/`);
3. adapters que conectan Axiom con harnesses o IDEs (`@axiom/adapters-*`);
4. manifests/YAML que describen la configuración y topología del proyecto (`axiom.yaml`, `axiom.config/*.yaml`, `topology.yaml`);
5. servidores MCP ejecutables y handlers project-scoped.

## CLI: comandos documentados en profundidad

`init`, `join`, `configure`, `sync`, `start`, `audit`, `doctor`, `upgrade`, `model` (`show`/`set`/`unset`/`reset`/`validate`), `components` (`list`/`show`/`install`/`uninstall`/`restore`), `skills` (`list`/`refresh`/`drift`). Fuente: `Axiom/docs/cli/*.md`. `tui` ya no es un comando registrado.

## CLI: comandos presentes en código sin documentación operativa equivalente

La superficie adicional de `apps/cli/src/commands/` incluye las rutas de aplicación/launcher, workflow, workspace multi-repo, repos y roles, contexto, MCP, memoria, toolchain, bootstrap, adopción, actualización y diagnóstico. `axiom learn` fue retirado; la memoria general se opera explícitamente mediante `axiom memory`. No se conserva aquí un conteo de ficheros o líneas como dato normativo: el registro efectivo está en `apps/cli/src/index.ts` y el contrato de cada familia se describe en las secciones siguientes. Un comando sin documentación propia debe tratarse como capacidad verificable en código y tests, no como contrato estable solo por existir un archivo.

### `axiom projects`: catálogo user-level

- `axiom projects list [--json]` lista el único catálogo `~/.axiom/projects.yml`, ordenado por `lastUsedAt` descendente e `id` ascendente. La vista distingue disponibilidad física por repo y resolubilidad Axiom; no elimina entradas ausentes.
- `axiom projects add --path <path> [--name <name>] [--id <id>] [--role <role>] [--json]` cataloga un directorio existente sin exigir que contenga un proyecto Axiom. `projects join` acepta `--path`, `--id` y `--role`, pero primero exige que `resolveProject` lo resuelva.
- `axiom projects use <id> [--role <role>] [--print-path] [--json]` actualiza únicamente `lastUsedAt`. No persiste un proyecto activo: el contexto sigue resolviéndose desde `cwd` o un `projectId` explícito. `--role` prevalece; sin él se prefiere la key `axiom` y después la key lexicográficamente menor. `--print-path` escribe exactamente el path y un salto de línea, nunca un comando de shell.
- Para cambiar de directorio, el caller consume el path según su shell: PowerShell `Set-Location -LiteralPath (axiom projects use <id> --print-path)`; POSIX `cd -- "$(axiom projects use <id> --print-path)"`; `cmd.exe` `for /f "delims=" %P in ('axiom projects use <id> --print-path') do @cd /d "%P"`.
- Con `--json`, `list|add|join|use` escriben en stdout exactamente un envelope v1 `{schemaVersion:1, ok, command, data?|error?}` en éxito o error. Los diagnósticos humanos van a stderr y los errores terminan con exit code 1.

### `axiom toolchain`: catálogo, lockfile y canales (`INC-20260730-toolchain-versioning`)

- `axiom toolchain show [--path <path>] [--json]` muestra las tools del manifest con `ID`, `KIND`, `SUPPORT`, `MVP`, versión observada, versión locked, canal y estado real. El probe de versión es best-effort; una tool sin contrato local no se presenta como instalada.
- `axiom toolchain validate [--path <path>] [--json]` conserva la validación del manifest y de la detección real. La ausencia de una tool opcional o un estado no verificado se reportan honestamente sin convertirlos en un error bloqueante salvo las reglas de tools requeridas existentes.
- `axiom toolchain plan [--path <path>] [--channel <stable|candidate|edge>] [--id <id>]... [--json]` calcula el diff entre el lockfile y el catálogo sin escribir. Por defecto considera las tools declaradas o lockeadas del proyecto; una tool que solo aparece en el catálogo no se añade implícitamente.
- `axiom toolchain upgrade [--path <path>] [--channel <stable|candidate|edge>] [--id <id>]... [--dry-run] [--yes] [--json]` actualiza únicamente `.axiom-state/<projectKey>/toolchain.lock`. Sin `--yes`, o con `--dry-run`, es preview; con `--yes` crea checkpoint y hace rollback si falla la persistencia o el probe posterior. No instala binarios externos.

El catálogo dedicado (`axiom.config/toolchain-catalog.yaml`) usa schema 2 y declara `versionExtractor`, canales y compatibilidad opcional. El lockfile usa schema 1, queda bajo `.axiom-state/<projectKey>/` y se escribe de forma atómica. `TC-020..TC-023` son las checks síncronas asociadas; el drift real de versión se comprueba mediante `axiom doctor --deep`.

Los comandos que leen estado legacy consultan primero el `projectKey` canónico
y luego aliases explícitos. `toolchain show/validate/repair` no hacen scans
globales, y el provisioning de worktree selecciona providers por
`Execution.projectId`.

### `axiom context index` y selección de contexto por tarea (`INC-20260820-r11-context-tag-selection`)

`axiom context index [--role <rol>] [--path <specRepo>]` regenera el índice derivado `technical-context/indexes/<rol>.index.yml`; no se edita manualmente. El generador usa `tags: [..]` de frontmatter YAML opcional y, en su ausencia, una tag de fallback de carpeta (`repo` para la raíz). Las tools MCP `spec.recommendedContextList` y `spec.implementationContextRead` comparten el mismo selector: en orden, devuelven `mandatory.always`, los grupos `mandatory.whenTags` que satisfacen todas sus tags y los documentos `available` que coinciden con alguna `taskTag`, sin duplicar por ruta. Omitir `taskTags` o pasar `[]` devuelve solo el bloque obligatorio. La lectura compuesta acepta `taskTags`; solo cuando hay alguna tag explícita suma las señales estructuradas `plan.taskType` y el rol explícito, sin inferir texto o usar IA. Conserva la separación `mandatory`/`recommended`, el path-guard dentro del spec repo y los presupuestos `small` (referencias), `medium` (contenido obligatorio) y `large` (añade ADRs relacionados).

### `axiom knowledge sync` y `axiom knowledge pull` (`INC-20260820-r11-knowledge-sync-hardening`)

`axiom knowledge sync --increment <id> --phase <phase>` previsualiza por defecto. Solo `--confirm` permite escribir chunks y ejecutar Git local; `--push` permite además enviar al remoto. Solo se comparte memoria `visibility: project-shared`, con evidencia completa y sin secretos en campos textuales serializados. `axiom knowledge pull` no acepta `--increment` y, con confirmación, importa todos los chunks pendientes; su marker personal project-scoped no forma parte del intercambio Git. Corrupciones o imports parciales se informan y permanecen reintentables.

## Launcher web (`axiom app`)

`axiom app` abre por defecto `/launcher/`. El launcher ofrece selector de
proyecto, install/join, workspace setup/adopt, doctor, registry, acciones del
ciclo SDD, plugins y paneles ADO/Git. Las operaciones mutantes usan
preview→confirmacion y delegan en los runners canonicos; el servidor no
introduce una segunda logica de negocio. Para transiciones de workflow, CLI,
launcher y MCP convergen en `runGovernedTransition`: el launcher reenvía
`confirmed: true` al subcomando de cada workflow y la ausencia de confirmación
permanece como preview; `--force`/`--no-verify` no son sustitutos de esa
confirmación. La ausencia de `axiom tui` y de la accion implicita sin
subcomando se comprueba en el binario compilado.

## Adapters (surfaces por IDE/CLI externo)

Hay 8 adapter targets canónicos activos. Todos tienen paquete dedicado y contrato común `generate<Target>Config(args) → Promise<Result<GeneratorResult, AdapterGeneratorError>>`: `opencode`, `claude-code`, `antigravity`, `visual-studio-2026`, `cursor`, `github-copilot`, `vscode` y `codex`. `copilot-vscode` no es una surface, tipo ni alias público; solo el lector interno de `configure` puede migrar ese valor cuando existe en `init.json` persistido. LiteLLM fue retirado. `SupportLevel` de model-routing (`multi-mode`, `single-mode`, `fallback-only`) es un eje independiente de la existencia de un generador. Fuente: `Axiom/packages/adapters/README.md` y `apps/cli/src/commands/init.ts`.

## Documentación operativa navegable

`Axiom/docs/README.md` es el índice hacia manuales de instalación, configuración, uso diario, CLI, ficheros generados y troubleshooting — ya existe y está mantenido, no es un artefacto a construir desde cero.

## Regla

Las interfaces operativas se implementan en `Axiom/`, pero su comportamiento esperado se define primero en `Axiom.Spec/`. Regla vigente pero con brecha real: parte de la superficie de comandos (sección anterior) ya existe en código sin haber pasado primero por una spec formal en `Axiom.Spec/`.

## Superficie de comandos ampliada por el roadmap de rediseño (cerrado)

La superficie realmente alcanzable y primaria coincide con los entrypoints con guion `axiom-increment ...`, `axiom-bug ...`, `axiom-plan ...` y `axiom-role ...` (registrados en `apps/cli/src/index.ts`), más:

- `axiom-adr`/`axiom-decision` (`create`/`link-plan`/`link-increment`/`list`, más `axiom-adr supersede <old-id> <new-id>`);
- `axiom index rebuild|validate|list` (todos los tipos de artefacto, `--json`);
- `axiom validate changes --project <id> --plan <planId>` (validación de write-scope);
- `axiom bootstrap from-code --level minimal|basic [--role <role>]` y `axiom bootstrap from-legacy-sdd <path> [--dry-run]`;
- `axiom repair` (general, top-level, distinto de `axiom toolchain repair` y `axiom mcp repair`).
- `axiom mcp serve --kind axiom --project-root <path> [--home-dir <path>]`: lanza el único server MCP ejecutable de `@axiom/mcp-server`, `axiom-mcp-broker`, sobre JSON-RPC 2.0 por stdio. El único `McpServerKind` aceptado es `axiom`; el broker expone el catálogo registrado de capabilities `sdd.*`, `spec.*`, `memory.*` y `axiom.*` bajo un único binding al proyecto. Las capabilities `axiom.*` son MCP-only y no exigen un provider tradicional. El comando se registra desde `apps/cli/src/commands/mcp-serve.ts`; los antiguos IDs `sdd-mcp-server` y `spec-mcp-broker` no son brokers emitidos ni kinds aceptados.

Ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md) para el contrato funcional completo de cada uno.

### Superficie CLI añadida por la tanda INC-20260708-* (providers, memoria, autoskills, operaciones incrementales)

Comandos nuevos entregados por esta tanda, todos app-owned (registrados directo en `apps/cli/src/index.ts`, no vía `@axiom/cli-commands`, para evitar el gotcha de single-ownership de ese paquete):

- **`axiom configure --providers <csv>`** (`INC-20260708-wizard-configure-provider-selection`): persiste la selección de providers LOCALES habilitados para el proyecto en `.axiom-state/<projectId>/workspace.json#providers` (merge-write, creando el fichero si falta), sin alterar el resto del comportamiento de `configure`. Los ids seleccionables son exactamente `cmm`/`serena`/`engram`; `filesystem` es el baseline siempre disponible. `cmm` y `serena` se registran como providers de code-intel y `engram` se resuelve mediante `MemoryBackend`. **No** toca `axiom.config/providers.yaml` (registry canónico local cerrado de 4 ids, schema-locked por `CC-001`/`CC-003`) — ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md) y [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md).
- **`axiom skills suggest [--repo <path>] [--apply]`** (`INC-20260708-autoskills-wizard-phase`): detección de stack + sugerencia de skills curadas para un repo (default cwd); sin `--apply` solo imprime (read-only), con `--apply` instala el set sugerido en `axiom.config/skills-index/<roleId>.yaml#available` de ese repo. Se une al grupo `skills` existente (`list`/`refresh`/`drift`); usable contra cualquier repo (no solo el proyecto activo).
- **`axiom repo add`**, **`axiom adapter add <target>`**, **`axiom provider add <id>`**, **`axiom role add <roleId> --path <path>`** (`INC-20260708-incremental-operations`): las 4 operaciones incrementales ADD-only, idempotentes y no-clobber, que resuelven el proyecto desde cwd y reusan el motor multi-repo (`runWorkspaceSetup` + helpers exportados). `repo add` escribe el `axiom.yaml` del repo nuevo con `paths` recíprocos y refresca los de los hermanos, actualiza `topology.yaml`, registra en `projects.yml` y genera adapters/MCP/skills/rules para los adapters habilitados del proyecto; `adapter add` persiste en `workspace.json#adapters` y regenera ese adapter en todos los repos; `provider add` persiste en `workspace.json#providers` (validación best-effort vía `buildProjectProviderRegistry`, sin spawnear nada); `role add` es un wrapper de `repo add --kind role` + asignación de rol en topología. Los REMOVE quedan diferidos (ver NFR-AXM-015 en [02_Requisitos_No_Funcionales.md](02_Requisitos_No_Funcionales.md)). Registrados desde `apps/cli/src/commands/workspace-incremental.ts`.

### `axiom workspace` — setup headless completo + reparación/reuso granular (`INC-20260710-workspace-command-parity`)

Antes de `axiom workspace`, `runWorkspaceSetup` era el motor multi-repo detrás del wizard interactivo histórico y no tenía caller headless. `axiom workspace` (`apps/cli/src/commands/workspace.ts`, registrado en `index.ts` justo después de `registerWorkspaceIncremental`) cerró ese hueco y sigue siendo 100% no-interactivo (sin TTY, sin prompts — todo por flags):

- **`axiom workspace setup`**: setup headless COMPLETO del workspace. Mapea flags a un `WorkspaceSetupSpec` (`buildWorkspaceSetupSpecFromFlags`) y llama al MISMO `runWorkspaceSetup`, sin cambios. Flags: `--name`/`--spec-path` (requeridos), `--control-path` (default cwd), `--role <name>:<path>` (repetible — roles de equipo dinámicos, 1..N, sin enum fijo, Decision D5), `--adapters`/`--providers <csv>`, `--no-register`, `--json`. La configuración funcional `builder` y la política `local-only` son implícitas.
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
  4. Activa el estado local project-scoped (`.axiom-state/<projectKey>/toolchain/<id>/`) de cada utilidad que el proyecto habilitó en `axiom.config/toolchain.yaml` (vía `repairTool`, `@axiom/toolchain`) e imprime la guía de instalación externa exacta cuando se conoce una (nunca inventa un comando).
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
- **Gate de verificación funcional** (`INC-20260711-functional-verify`): `axiom-increment verify` y `axiom-role complete` descubren y ejecutan la validación del repo destino y BLOQUEAN ante fallo; `--no-verify`/`--force` saltan el gate y `--preview`/`--dry-run` reporta lo descubierto sin correr. `axiom-qa-e2e` usa el mismo runner gobernado en ambos carriles: `start → verify [--run-validation] → pass` registra la evidencia de QA; en inline esa evidencia `passed` es la condición previa del archive.
- **Tools MCP de acción git** `sdd.gitRoleBranch` / `sdd.gitCommitSync` (`INC-20260711-git-services`): espejo de `sdd.transitionApply` — preview por defecto, mutan solo con `{ confirmed: true }`, nunca push salvo `{ push: true }`.
- **Canal push SSE en `axiom app`** (`INC-20260711-front-longtail`): el server emite `text/event-stream` sobre el `http.Server` existente y el shim del front lo consume con `EventSource`, degradando a fetch cuando no está disponible; añade el mapeo execute (confirm-gated) de `plan-new`/`plan-execute`, cerrando el largo plazo P3 mantenido.
- **Paneles operadores del front `/launcher/`** (`INC-20260711-epic-close-panels`): el front bajo `/launcher/` gana tres paneles, cada uno UI fina sobre un motor ya entregado — **sugerencias ADO** (lista read-only vía `IWorkItemTracker.listWorkItems`; enlace display-only, degrada a "ADO no configurado" con `NullTracker`, sin red) y **role-branch** + **commit-sync** (preview→confirm sobre las git-services de `@axiom/workflow`; git local-only, sin push por defecto). Un apply git confirmado emite `registryChanged` por el canal SSE de arriba.

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

## Histórico: TUI — pantallas y flujos añadidos por el roadmap de rediseño

Las subsecciones siguientes conservan la forma histórica de la interfaz
retirada para trazabilidad. No describen una superficie operativa vigente ni
deben usarse como instrucciones de runtime; los sustitutos actuales son el
launcher web, la CLI headless y MCP.

`router.ts`'s `MENU_ITEMS` tiene 7 entradas: configurar, sincronizar, diagnóstico, upgrade, **reparar instalación** (posición 4, entre upgrade y model routing — añadida por este roadmap), model routing, salir. `packages/tui/src/flows/configure.ts`/`upgrade.ts`/`repair.ts` son wrappers sin lógica de negocio (solo formateo de mensajes). Esto extiende, sin reemplazar, el menú de 6 items descrito en la sección "TUI" de arriba.

Las pantallas `mcp-inventory`/`memory-inventory` permanecen accesibles solo por flag de CLI, no como `MENU_ITEMS` de primera clase — promoverlas es un incremento explícitamente no iniciado (`INC-20260702-tui-menu-promote-inventory-screens`, diferido), no un hueco de esta spec.

## TUI — menú de bootstrap `setup` y wizard guiado de setup de workspace (`INC-20260705-tui-workspace-setup-wizard`)

Cuando `axiom`/`axiom tui` abre en una carpeta sin proyecto Axiom, la screen `setup` (items estáticos `SETUP_ITEMS`) ofrece: "Inicializar Axiom en esta carpeta" · "Actualizar Axiom" (self-update) · "Ver proyectos Axiom registrados" · "Salir". Elegir un proyecto de la lista de registrados hace `chdir` + abre la TUI operativa sobre él.

Elegir "Inicializar Axiom en esta carpeta" recorre un WIZARD GUIADO MULTI-REPO que prepara un entorno de trabajo completo (SDD + Spec + repos de rol, cruzados entre sí, registrados y con MCP configurado) en una sola operación. **Supersede al wizard single-repo de 6 pasos de `INC-20260703-tui-init-wizard`**: aquella acción `init` ya no llama a `runInit`, sino que ahora ensambla un `WorkspaceSetupSpec` y llama a `runWorkspaceSetup` (`INC-20260705-workspace-multirepo-setup-engine`, con generación MCP de `INC-20260705-workspace-mcp-generation`). El comando no-interactivo `axiom init`/`runInit` (RF-AXM-006) sigue siendo single-repo y sin cambios; solo cambió la acción `init` de la screen `setup`.

Orden de steps, cada uno con un default sensato pre-cargado que el usuario acepta (Enter) o cambia:

1. `name` (texto libre — del que se deriva el `projectId`);
2. `sddPath` (texto, default `../<slug>-sdd`);
3. `specPath` (texto, default `../<slug>-spec`);
4. `roles` (texto libre — nombres de rol separados por coma, cero o más, incluidos roles custom; se sanitizan y deduplican al confirmar);
5. `rolePath:<role>` (texto, uno por rol seleccionado, default `../<slug>-<role>` — inyectados dinámicamente, ver `expand` abajo);
6. `profile` (select);
7. `overlay` (select);
8. `adapters` (multi-select — cero o más adapters a instalar; opciones = los 8 `ADAPTER_TARGETS` activos; default pre-seleccionado `['opencode']`) — **supersede al step `target` `select` único** que describía la versión previa de este wizard (`INC-20260705-workspace-adapters-multiselect`);
9. `providers` (multi-select — providers LOCALES seleccionables `cmm`/`serena`/`engram`; sin preselección por defecto);
10. un único resumen de confirmación.

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
- **Configuración funcional y operativa**: el launcher no muestra steps de `profile` u `overlay`; usa `builder` + `local-only` implícitos y mantiene como elecciones independientes el target/adapters, providers locales, roles de equipo y `executionMode`.

### Step `providers` (multi-select) — INC-20260708-wizard-configure-provider-selection

El wizard gana un step `providers` (`multi-select`, colocado tras `adapters`, reusando el mismo tipo genérico de step de `@axiom/tui`, sin preselección por defecto) que ofrece los 3 providers LOCALES seleccionables (`cmm`/`serena`/`engram`). La selección fluye vía `buildWorkspaceSetupSpecFromWizard` a `WorkspaceSetupSpec.enabledProviders` y `runWorkspaceSetup` la persiste en `.axiom-state/<projectId>/workspace.json#providers` — la MISMA elección que `axiom configure --providers` escribe fuera del wizard. `cmm` y `serena` quedan disponibles para `buildProjectProviderRegistry`; `engram` se resuelve con `resolveMemoryBackend`. La inicialización y los probes son best-effort y una tool ausente produce degradación, no un setup fallido. Mata la "false breadth": un proyecto fresco solo declara lo que el usuario eligió, sin implicar más amplitud. Ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md) y [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md).

### Fase final del wizard: autoskills por repo de código — INC-20260708-autoskills-wizard-phase

Tras un `runWorkspaceSetup` exitoso (antes de reabrir la TUI operativa), el wizard corre una fase final OPT-IN de autoskills, uno-por-uno por CADA repo de código (`kind: 'role'`) recién creado: detecta su stack (`detectStack`, reusa la inferencia por marcadores de `INC-20260708-rules-layer` más una lectura best-effort de `package.json` para `react`/`nextjs`/`express`) y, solo si hay al menos una skill sugerida (nunca un prompt vacío), lanza un `runTuiDriver` separado con un único `multi-select` de las skills sugeridas para ese repo. Confirmar instala las elegidas en `axiom.config/skills-index/<roleId>.yaml#available` de ese repo (`installSuggestedSkills`, la misma ruta que `axiom skills suggest --apply`); declinar (nada marcado) es un no-op para ese repo. Best-effort estricto (el fallo de un repo no afecta a los demás ni al setup ya completado). Se implementa como N `runTuiDriver` secuenciales porque `RunTuiArgs.wizardSteps` es estático (resuelto antes de arrancar el driver) mientras que el set de repos + stacks solo se conoce tras `runWorkspaceSetup` — sin inventar un concepto de "wizard anidado" en el driver genérico. Ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

## Adapters — MCP config por proyecto (cableada por el setup de workspace)

`generateOpencodeMcpJson`/`generateClaudeCodeMcpJson` (ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md) para el schema `mcp.yml`) están completamente implementados y testeados, y permanecen exportados en sus paquetes. Su call site vivo en la ruta de workspace fue **retirado por `INC-20260708-mcp-native-config-mapping`**: `runWorkspaceSetup` escribe `.axiom/mcp.yml` en el repo de control (fuente canónica) y, en vez de proyectar al custom-shape `.opencode/mcp.json`/`.claude/mcp.json`, emite ahora el **schema MCP nativo por herramienta** — `opencode.json` (opencode), `.mcp.json` (claude-code), `.cursor/mcp.json` (cursor) y `.vscode/mcp.json` (github-copilot/vscode) — por cada repo × cada adapter seleccionado, merge-preserving y atómico (`writeNativeMcpConfig`/`writeWorkspaceNativeMcpConfigs`), tras pasar el filtro project-bound (`filterProjectBoundMcpServers`). `codex` y `antigravity` tienen solo una entrada informativa user-global, sin fichero de proyecto; `visual-studio-2026` no recibe fichero (schema MCP no verificado). `.axiom/mcp.yml` se sigue escribiendo exactamente una vez por proyecto. Fuera de esa ruta, ni `runConfigure` ni `sync` llaman a esos dos generators custom-shape MCP. `runConfigure` instala el perfil, mantiene el fallback acotado de OpenCode y escribe instrucciones para `github-copilot`; `sync` despacha generators reales para `opencode`, `claude-code`, `github-copilot`, `vscode` y `cursor`, mientras `antigravity`, `visual-studio-2026` y `codex` producen cero outputs.

## Entrada MCP nativa de engram (stdio) en cada adapter — INC-20260709-engram-mcp-stdio-native-config

Cuando el provider `engram` está habilitado (`workspace.json#providers`), la MISMA emisión de config MCP nativa por herramienta descrita arriba incluye además una entrada `engram` LOCAL de stdio (`command: 'engram'`, `args: ['mcp','--project',<projectId>,'--tools','agent']`) en `.mcp.json`/`.cursor/mcp.json` (`mcpServers.engram`), `.vscode/mcp.json` (`servers.engram`, `type:'stdio'`) y `opencode.json` (`mcp.engram`, `type:'local'`). Un dev que abra cualquier repo del workspace con cualquiera de esas herramientas obtiene memoria persistente de engram por MCP stdio **sin** arrancar el daemon `engram serve`. Ningún fichero generado referencia engram por HTTP/puerto 7437/`ENGRAM_URL`. Ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md) y [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md).

## Reviewer con ledger de hallazgos — INC-20260709-review-findings-ledger

El agent de revisión (`axiom-reviewer` en el catálogo de agents materializable, y el agent bootstrap `axiom-review` de este workspace) emite ahora, además de su recomendación de cierre, un **ledger de hallazgos** estructurado (`id`/`lens`/`location`/`severity`/`status`/`evidence`) producido por una primera pasada exhaustiva loop-until-dry, y hace re-review scoped al ledger + diff del fix. El detalle del contrato y su persistencia vive en [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md).

## Superficie de adopción/migración de repos foráneos (2026-07-11) — tanda migración

Comandos añadidos para adoptar un proyecto preexistente (detalle de capacidad en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md); flujo en [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md)):

- **`axiom bootstrap from-context <path>`** (`INC-20260711-mig-context-ingest`): ingiere un contexto técnico existente (`ARCHITECTURE.md`/`docs/**`/ADRs) a `technical-context/*` + un `TechnicalContextIndex` `draft` servido por `spec.technicalContextIndexRead` (MCP); cada doc lleva banner `AXIOM:MIGRATED` (distinto del `AXIOM:DRAFT` de `from-code`); re-corrida sin clobber.
- **`axiom bootstrap from-legacy-sdd`** ahora es **format-aware** (`INC-20260711-mig-spec-adopter`): además de carpetas Axiom-shaped, detecta repos spec foráneos (`openspec` / `docs-adr` / `generic-folders`) vía un registro de detectores enchufable y convierte a la plantilla canónica de incremento; el `--dry-run` lista el/los formato(s) detectado(s).
- **`axiom workspace setup --adopt-spec / --adopt-sdd / --ingest-context [--dry-run] [-y]`** (`INC-20260711-mig-adopt-ux`): orquesta en install-time la conformancia del repo de control (subject B: reporte present/added/must-reconcile) + la migración del repo de spec foráneo (subject A) + la ingesta de contexto; por defecto muestra un preview dry-run y exige confirmación antes de escribir (`-y` la salta).

## Superficies generadas por rol/adapter y desbloqueo de `upgrade` en adoptados (2026-07-13) — tanda fixes + north-star

- **Agents/commands/skills de proceso generados por rol y por adapter** en cada repo del workspace (`INC-20260713-ns2-adapter-generation`): claude-code + opencode reciben superficies nativas COMPLETAS; cursor / github-copilot un fichero de instrucción role-differentiated bajo `.github/instructions/<id>.instructions.md`; y cada repo además superficies portables adapter-agnósticas bajo `.axiom/{agents,commands,skills}/`. El rol determina cuáles: spec → `axiom-spec-author` + `axiom-role-planner`; cada repo de código → `axiom-role-implementer` (parametrizado con el slug/repo del rol); sdd/control → `axiom-sdd-orchestrator`. `AGENTS.md` se renderiza con placeholders (`{role}`/`{specRepo}`/`{repoPath}`/`{projectId}`) resueltos de la topología, así que el AGENTS.md del repo de spec difiere del de un repo de código; las skills materializadas quedan registradas en `skills-index/<role>.yaml` (`repoSkills` deja de ser `[]`). Cableado en `runWorkspaceSetup` (y por delegación en `runWorkspaceAdopt`, `repo add`, `role add`).

### Superficie común de instrucciones Copilot (ACC-024)

`github-copilot` escribe `.github/copilot-instructions.md` como instrucción general del repositorio. `.vscode/` contiene solo configuración propia de VS Code y MCP (`settings.json`, `extensions.json`, `mcp.json`); las instrucciones específicas por ruta usan `.github/instructions/*.instructions.md`.

`configure`, `sync`, `workspace setup` y el adapter dedicado convergen en
`@axiom/document-bootstrap`: el template on-disk gana cuando se puede leer y
el bundle actúa como fallback. El writer reemplaza únicamente
`AXIOM:GENERATED`, preserva el preámbulo, la cola humana y `TEAM:CUSTOM`, y
migra `.vscode/copilot-instructions.md` de manera atómica y conservadora.
`copilot-vscode` no participa como target: solo `configure` puede migrar ese
literal si ya está persistido en `init.json`, antes de instalar o despachar.
- **Adopción materializa `init.json`** (`INC-20260713-fix-adopt-upgrade`): `axiom workspace setup`/`adopt` escribe ahora `<control>/.axiom-state/<name>/init.json` (antes solo lo hacía `axiom init`), de modo que el gate `upgrade-command` (`hasInitJson`) pasa y **`axiom upgrade` funciona out-of-the-box sobre un proyecto adoptado** (ya no exige `--no-sync --no-doctor`).
- **Fallback de plantilla `AGENTS.md` bundleada en `sync`/`upgrade`** (`INC-20260713-fix-adopt-upgrade`): `materializeAdapterOutputs` cae a la plantilla `agents-md-template.md` bundleada cuando no está en disco (topología adoptada), evitando el hard-fail `template-missing` que forzaba el rollback del `upgrade` por defecto.

## Cierre de brechas post-KVP25: rollback, upgrade cross-repo y front launcher/ADO (2026-07-14) — tanda INC-20260714-*

### CLI

- **`axiom rollback [checkpointId] [--list] [--dry-run]`** (`INC-20260714-op-rollback-restore`): comando de operador que **restaura el `ManagedState`** (y los demás archivos del manifest: `init.json`, `install-profile.json`) desde un checkpoint conocido, exponiendo la superficie de restore ya pública de `@axiom/versioning` (`restoreCheckpoint`/`listCheckpoints`). `--list` enumera los checkpoints; `--dry-run <id>` previsualiza los archivos sin mutar; un `id` inexistente → error claro **sin mutar** (el check de existencia del manifest ocurre antes de cualquier escritura). Es una acción de recuperación: **NO** se gatea por el orchestrator `upgrade-command` (gatearla podría bloquear la recuperación). Sinergia: tras un `upgrade` fan-out, cada repo migrado se puede revertir individualmente con su `checkpointId`.
- **`axiom upgrade` fan-out cross-repo + `--repo-only`** (`INC-20260714-cross-repo-upgrade-fanout`): ejecutado desde el **repo de control** de un proyecto **multi-repo** (`loadTopology().mode === 'multi-repo'`), `axiom upgrade` ahora itera **todos los repos de la topología** (`sddRepo`, `specRepo`, `roleCodeRepositories`), migrando/checkpointeando cada uno (vía `executeUpgrade` directo, que no exige `init.json` en repos de rol/spec), y **`sync`+`doctor` corren una sola vez** a nivel workspace. Reporte por-repo (`repoId`/`role`/`path`/from→to/checkpoint/ok|failed). `--repo-only` (o correr desde un repo no-control) conserva el modo per-repo previo; un proyecto single-repo es byte-idéntico (el fan-out sólo se dispara en multi-repo → sin regresión).

### Front del launcher (`axiom app`, navegador, cero VSCode APIs)

- **Rediseño UX guiado** (`INC-20260714-launcher-ux-ado`, P0): `apps/cli/static/launcher/*` rediseñado como un launcher pulido y guiado — sistema visual con escalas de tipografía/espaciado/elevación sobre las `--ax-*` vars (light/dark), sin frameworks ni deps nuevas, invariante RISK-004 (cero `acquireVsCodeApi`/`--vscode-*`) intacto. **Flujo guiado de 3 pasos** ("¿qué querés hacer?" → familia increment/bug/plan/implementación(back/front/e2e) → modo → formulario dinámico con `visibleWhen`) que produce el **prompt pregenerado** con **copiar a 1 clic** + lanzar + ejecutar opcional (scaffold de la estructura vía los run-functions, preview→confirm). Vistas por tabs (Crear / Registro / ADO & Git) + registro en vivo (SSE). El miembro sólo introduce la información necesaria; la operativa la automatizan los scripts existentes.
- **Workflow ADO en el front** (`INC-20260714-launcher-ux-ado`, P1): tarjeta "Workflow ADO" con formularios para **crear work item, cambiar estado, estimar, marcar horas, enlazar rama/PR/commit y enlazar con el artefacto Axiom**, cada uno preview→confirm y config-gated sobre `tracker.kind` (si no es `ado` → "ADO no configurado", sin red). Delega en `@axiom/tracker-ado`; detalle de capacidad en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md). Coexiste con los 3 paneles previos (sugerencias ADO, git role-branch, commit-sync).

## Launcher como front visual completo: onboarding, gate de doctor, tuning de adapters y puente ADO (2026-07-15) — tanda INC-20260715-*

Amplía el front del launcher (`axiom app`, navegador, framework-free ES5) para que un equipo pueda operar sin terminal/TUI. Transporte inalterado (`AxiomBridge.postMessage` → HTTP → `window` `message`). Todo mutante sigue el contrato `confirmed:true` (preview→confirm). Gate verde por incremento (build limpio; suite de launcher/onboarding/doctor/ado-bridge + `init`/`projects`/`roles` + `tracker`/`tracker-ado` en verde, 204 tests consolidados).

- **Gate de doctor pre-lanzamiento** (`INC-20260715-launcher-doctor-gate`): al seleccionar proyecto, el front pide `GET /api/projects/:id/launcher/doctor` (thin wrapper sobre `runDoctorChecks`, best-effort/no-crash) y muestra un panel de salud (ok/warn/fallo) con las incidencias (`category`/`description`/`evidence`). Ejecutar/lanzar avisan si hay fallos y exigen un segundo clic (gate visible, no bloqueo).
- **Onboarding visual** (`INC-20260715-launcher-onboarding` / `INC-20260726-launcher-onboarding-config-front`): la pestaña "Instalar / Unirse" permite instalar un proyecto nuevo o unirse a uno existente, registrar/asignar roles y explorar carpetas. El formulario de install/join expone `name`/`path`/`profile`/`overlay`/`layout`/rol, adapter primario, adapters adicionales, tools y `execution-mode`, con preview y confirmación preservados. El adapter primario se resuelve mediante `runInit` y los adicionales mediante `generateWorkspaceAdapters`; las tools se muestran, pero solo se aplican cuando existe un catálogo de toolchain válido. Los endpoints siguen siendo server-level `POST /api/launcher/{install,join}` y project-scoped `POST /api/projects/:id/launcher/roles/{register,assign}`, junto con `GET /api/launcher/options` y `GET /api/launcher/browse`.
- **Selección de adapter → prompt pregenerado + tuning** (`INC-20260715-adapter-agent-tuning`): el selector de adapter ya existente ahora muestra el tuning (`low/pragmatic`) y el prompt pregenerado incluye el preámbulo "Ajustes del agente". La superficie de "seleccionar adapter para generar el prompt en consecuencia" ya existía (`craftPrompt` + `apiCraftLauncherPrompt`); esta tanda la enriquece con el tuning por adapter (capacidad en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md)).
- **Puente ADO en creación** (`INC-20260715-launcher-ado-bridge`): tras crear un incremento/bug (confirmado) en la pestaña Crear, si el plugin ADO está configurado, se ofrece un work item pre-rellenado de un clic (reusa la tarjeta/endpoint ADO existente); si no, nota informativa sin red. Detalle en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md) y [manuales/12_Plugin_Azure_DevOps.md](manuales/12_Plugin_Azure_DevOps.md).

Documentación de usuario de estas superficies: [manuales/11_Launcher_Visual.md](manuales/11_Launcher_Visual.md).

## Superficies SDD instaladas + revisión en el launcher (2026-07-15) — tanda INC-20260715-*

- **Nuevas superficies de proceso** materializadas por rol de repo (ver reparto en [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)): `axiom-phase-reviewer` y `axiom-qa-validator` en el repo `sdd`; `axiom-spec-integrator` y `axiom-tech-context` en el repo `spec`. Se suman a las 3 de flujo (`axiom-spec-author`/`axiom-role-planner`/`axiom-role-implementer`) y al orquestador.
- **Revisión como acción del launcher** (`INC-20260715-phase-reviewer`): la familia `review` del catálogo (`review-spec`/`review-plan`/`review-code`) genera un prompt de revisión (prompt-only, sin mutación) vía el `promptIntent: 'review'` que delega en `buildReviewPrompt`; se selecciona como cualquier otra acción y se cruza con el adapter elegido.

## Superficies del modelo `<project>.axiom`, eject y ejecución en worktree (2026-07-24) — tanda INC-20260724-*

Superficies operativas nuevas de la graduación a *full product lifecycle*. Flujos en [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md); datos en [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md).

- **`axiom topology show` — campos del modelo gestionado** (`INC-20260724-topology-single-axiom-repo`): superficie `schemaVersion`, `axiom-repo` y `legacy-repos` además de las líneas previas (preservadas verbatim), más una nota de forma legacy cuando el manifest es `schemaVersion: 1` (`deprecated-legacy-shape`, no bloqueante).
- **`axiom eject` — salida de Axiom** (`INC-20260724-export-eject-rollback`): comando nuevo (verbo único, sin colisión con el `axiom rollback` de checkpoints) que vuelca los artefactos rollback-eligible (`axiom-native` + `migrated-and-modified`) a `<project>.axiom/exports/<exportId>/` con `EXPORT_REPORT.md` + `export-manifest.yaml`. `--dry-run` es el **default** (cero escritura); `--write-export-folder` escribe; si se pasan ambos gana `--dry-run`. Nunca escribe en legacy. Registrado en `apps/cli/src/index.ts` tras `registerBootstrap` (su inverso semántico).
- **`axiom mcp serve --kind axiom` — broker MCP unificado** (`INC-20260724-unified-axiom-mcp`, forma vigente): `--kind` acepta exclusivamente `axiom` y lanza `axiom-mcp-broker`. `axiom.config/mcp-manifest.yaml` declara el único binding ejecutable (`id: axiom`, `server: axiom-mcp-broker`, `projectBinding: required`, `installMode: project-scoped`); las antiguas entradas `sdd`/`spec`/`memory` no son kinds ni brokers productivos. Detalle de tools en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).
- **Selección de modo de ejecución in-place / worktree** (`INC-20260724-worktree-mode-selection`):
  - `axiom configure --execution-mode <in-place|worktree>`: fija el default del arquitecto (persistido en `install-profile.json`, default `in-place`, preservado entre re-configuraciones).
  - `axiom-role start --worktree` / `--in-place`: override por run (mutuamente exclusivos). En modo worktree, `start` compone worktreeAdd + Execution + provisioning, y `complete` corre harvest+cleanup (nunca `force` por defecto — un worktree con trabajo real sin integrar es hard stop).
  - **Launcher** (`axiom app`): campo `executionMode` (select `in-place`/`worktree`) en las acciones `back-new`/`front-new`, **solo para el preview** del comando CLI equivalente; el ejecutor autoritativo es la CLI (`axiom-role start --worktree`), el path de ejecución real del launcher no cambia.
- **Catálogo runtime tras la tanda**: 18 skills / 14 agents (incluye las 4 disciplinas transversales reutilizables y el `axiom-security-reviewer` ya con cuerpo real). Detalle de capacidades en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md); guía en [manuales/13_Skills_Agentes_y_Roles.md](manuales/13_Skills_Agentes_y_Roles.md).

### Launcher como front por defecto de `axiom app` + retirada de la UI de operador antigua (2026-07-27) — INC-20260727-launcher-default-and-old-ui-removal

Consolida el comportamiento de `axiom app` con el front real que ya se servía bajo `/launcher/` (ver la subsección "sdd-launcher-port" arriba). Supersede el default previo, que abría la vieja UI de operador raíz (`static/index.html`+`app.js`, la shell PWA de `0025 Lote A`, con su modal "Confirmar acción" atascado al cargar y el bug de render `kvp25 (undefined)`):

- **Default**: `appStart` abre `${url}/launcher/` y el banner del CLI imprime esa misma URL. `--port`/`--no-open` sin cambios (`apps/cli/src/commands/app.ts`).
- **Redirección server-side**: `GET /` y `GET /index.html` devuelven `302 Location: /launcher/` (belt-and-suspenders — la vieja `index.html` raíz tampoco existe ya en disco). `apps/cli/src/commands/app-api.ts`.
- **Borrado del shell antiguo**: `static/index.html`, `app.js`, `style.css`, `sw.js`, `manifest.json` eliminados, y con ellos sus 6 endpoints backend EXCLUSIVOS (topology/roles/toolchain/memory/mcps/workflow-state como rutas, más la ruta `commands/preview` con su función `buildCommandPreview`, y la ruta `commands/execute`), cada uno confirmado muerto por una búsqueda de consumidores repo-wide antes de borrar. `executeSubcommand` (la función, usada in-process por el propio path de ejecución del launcher `/launcher/execute`) y tres endpoints NO ligados a la vieja UI (`GET /api/projects/:id`, `.../plugins`, `/api/help`) se conservan intactos.
- **Verificado en runtime** contra el CLI construido: `GET /` → 302 `/launcher/`; `GET /launcher/` → 200 con `<title>Axiom Launcher</title>`; `GET /app.js` → 404.
### Superficie CLI de gobierno verificable (2026-08-02) — tanda `INC-20260730-*`

- **`axiom freeze --increment <id>`** — congela el candidate escribiendo `candidate-freeze.json` en la carpeta del incremento (hash combinado de memoria Engram filtrada + `README.md`). No existe `--force-json` ni backend JSON alternativo. Si Engram no está disponible, el error queda visible y el freeze no simula un hash correcto. Expone además la API programática `checkCandidateFreeze(incrementId, cwd) → { ok, reason? }`, ya consumida por `axiom-increment` como gate previo al apply. `INC-20260730-candidate-freeze`, reconciliado por R-12.
- **`axiom phase receipt --increment <id> --phase <name> --status <success|failure> [--details <msg>]`** — emisión manual de un receipt. Se mantiene intacta tras `INC-20260730-phase-receipts`, pero **deja de ser la única vía**: `runGovernedTransition` emite el receipt desde la ruta común cuando el caller habilita `receipt`. El comando manual queda para fases sin transición CLI propia y para los incrementos hand-authored spec-first de este mismo repo.

Nota de vocabulario, relevante al leer receipts: el campo `phase` lleva el **nombre real de la transición** (`increment-create`, `increment-verify`, `bug-archive`…), no el vocabulario `design/tasks/apply/verify/knowledge/freeze/close` que aparece en el Scope original del incremento. Ese segundo vocabulario no existe como tal en la CLI, cuyos subcomandos reales son 8 para `increment` y 4 para `bug`; se prefirió registrar lo que efectivamente se ejecutó, verificable 1:1 contra el state machine, antes que inventar un mapeo.

## Workflow gobernado R-10

Los entrypoints públicos son `axiom-increment`, `axiom-bug`, `axiom-plan`, `axiom-role` y el inspector `axiom state`; no existen `axiom sdd advance`, `axiom plan` ni `axiom role start` como comandos públicos. Los 19 intents `not-implemented` de `@axiom/orchestrator` son contratos internos de prueba y no rutas de CLI.

`axiom-plan approve --id <id>` muestra preview sin escribir y, sólo con confirmación explícita, ejecuta `draft → plan-approved` mediante el runner común; `axiom-role start` se bloquea hasta que state y metadata del plan estén aprobados. `axiom integrate`, archive de increment/bug, launcher y MCP muestran o aplican la misma decisión QA: `parallel` permite estados no passed con aviso; inline o rol QA requerido bloquea todo estado distinto de `passed`. El resolvedor de workflow compartido toma el YAML de proyecto válido o, sólo si falta, el asset empaquetado; ante YAML presente inválido informa error en todas las superficies.
