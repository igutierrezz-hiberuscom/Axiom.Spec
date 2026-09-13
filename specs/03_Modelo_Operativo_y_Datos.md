# 03 Modelo Operativo y Datos

> **R13 validado:** el modelo de self-update transaccional, manifiesto v2 como cache/receipt y estado de instalación global están validados y cerrados en los incrementos R13-1 a R13-5.

> Esta sección documenta el modelo de datos REAL implementado hoy en `Axiom/`. `~/.axiom/projects.yml` es el único catálogo user-level. El modelo de topología de repos por rol y `axiom.yaml schemaVersion: 2` están implementados; `single-repo` sigue siendo el modo por defecto práctico y la ruta multi-repo se activa explícitamente.

## Modelo de datos real: `axiom.yaml` (manifiesto raíz por proyecto)

Cada proyecto que adopta Axiom tiene un único `axiom.yaml` en su raíz. Campos relevantes del schema actual (`Axiom/docs/configuration/project-structure.md`):

- `project.name`, `project.status`, `project.product_implementation_status`, `project.mode`;
- `scopes`;
- `rules`;
- `artifact_id_policy`;
- `lifecycle_commands`;
- `initial_capabilities`.

Se genera con `axiom init` y persiste la configuración efectiva `builder` + `local-only` + `adapterTarget` (uno de 8 targets activos). `copilot-vscode` no es un valor de entrada público: `axiom configure` solo puede migrarlo si figura en `init.json#profileTriple.adapterTarget` y lo persiste como `github-copilot` antes de instalar o despachar. LiteLLM fue retirado. `builder` y `local-only` son implícitos; estados antiguos se normalizan al leerlos y los selectores de perfil/overlay ya no forman parte de la interfaz.

## Estado project-scoped: `.axiom-state/`

> Renombrado desde `.sdd/` en `INC-20260703-config-folder-renames` (sin
> migración: no había proyectos reales todavía). El nombre anterior
> `.sdd/` no comunicaba con claridad que se trata del estado runtime de
> Axiom.

- `.axiom-state/local/`: overlay no versionada exclusivamente local al repo/operador (overrides locales, bindings de topología y audit trail). No contiene estado que deba separarse por proyecto.
- `.axiom-state/<projectKey>/`: único namespace físico del estado ligado al proyecto. `projectKey` es `axiom.yaml#projectId` en schema v2 y el slug estable de `project.name` en v1. Contiene `init.json`, `members.yaml`, `install-profile.json`, `workspace.json`, `last-start.json`, `last-sync.json`, `toolchain.lock`, checkpoints, `managed-state.json`, `model-assignments.json`, `components-state.json`, workflow state, memoria, bindings MCP, plugins y skills pendientes.
  - `init.json` **no** lleva un campo `projectName` propio (`INC-20260703-config-dedup`, dedup #1): la identidad física se deriva del `projectKey` resuelto; el nombre humano no se usa como segmento nuevo en v2. `init.json` persiste el campo de compatibilidad `profileTriple` normalizado a `builder` + `local-only` + target, además de `createdAt` y `version`.
  - El scope API `config` de `@axiom/persistence` conserva su nombre por compatibilidad, pero ya no crea un segmento físico `config`.
- `.axiom-state/<projectKey>/checkpoints/<id>/`: snapshots pre-mutación (upgrade, uninstall de components); se conservan los últimos 5. Lectores de recuperación aceptan y migran checkpoints legacy bajo nombres anteriores.
- `.axiom-state/executions/<executionId>/`: estado aislado de una ejecución paralela; es una frontera independiente dentro de `.axiom-state/`.

La precedencia de lectura es canonical, directorio legacy directo, antiguo
`config`, scope legacy y solo después archivos conocidos bajo `local/`. Una
resolución conflictiva conserva el canonical o el primer candidato legacy
determinista y emite warning. `restoreCheckpoint` aplica el mismo principio a
los destinos del manifest: no basta con encontrar el snapshot, debe restaurar
el contenido bajo `projectKey`.

## Estado user-level del self-update (R13)

La instalación global del CLI conserva `~/.axiom/install.json` como receipt/cache derivado de la instalación real. La identidad del entrypoint y su procedencia son la autoridad; un receipt ausente, legacy, corrupto o no reconciliable no fabrica una versión por defecto.

La superficie R13 expone `axiom self-update status`, `check`, `plan`, `apply` y `recover`. `status`, `check` y `plan` son read-only; `--dry-run` solo se admite en `plan`. `apply` y `recover` consumen el motor transaccional de R13-3 y solo operan sobre una instalación gestionada: sus outcomes son `installed`, `unchanged`, `failed` o `recovery-required`, con códigos efectivos `0`, `70` y `74` según el resultado. Las escrituras usan el lock user-level, revisión/fingerprint esperados, journal y reemplazo atómico del Core; la lectura de un receipt v1 informa `migration_required` y no migra automáticamente.

## Catálogo de configuración declarativa: `axiom.config/*.yaml`

> Renombrado desde `axiom.spec/config/` en `INC-20260703-config-folder-renames`
> (sin migración): la carpeta se COLAPSA para vivir directamente en la
> raíz del proyecto como hermana de `axiom.spec/`, en vez de anidada
> dentro de él — son archivos de CONFIG generados, no contenido de
> spec, y el nombre anterior invitaba a confundirlos.

El runtime espera, dentro del proyecto adoptante, una carpeta `axiom.config/` en la raíz del proyecto (no confundir con `Axiom.Spec/`, el repo de este workspace, ni con el `axiom.spec/` del propio proyecto adoptante, que sigue existiendo para contenido de spec) con ~20 YAML de política y capacidad (`Axiom/docs/configuration/README.md`):

`axiom.workspace.yaml`, `branch-policy.yaml`, `capabilities.yaml`, `clarification-policy.yaml`, `command-protocol.yaml`, `external-work-items.yaml`, `id-policy.yaml`, `integrations.yaml`, `lifecycle-policy.yaml`, `local-overlay-policy.yaml`, `model-routing-policy.yaml`, `onboarding.yaml`, `orchestration-policy.yaml`, `policy-as-code.yaml`, `profiles.yaml`, `providers.yaml`, `repositories.yaml`, `scaffolding-contract.yaml`, `skills.yaml`, `telemetry-sinks.yaml`, `tool-routing-policy.yaml`.

No todos se consumen hoy con el mismo nivel de profundidad en runtime, pero forman el mapa documental y de control declarado. Verificado el 2026-07-30: `axiom.config/` y `axiom.spec/` existen en la raíz del propio repo `Axiom/`; `_builder/` sigue siendo un hueco menor que el script de readiness crea vacío en el proyecto temporal. Los riesgos históricos se conservan en `context/references/03-riesgos-y-brechas-conocidas.md`.

### Frontera entre `Axiom.Spec/` y `Axiom/axiom.spec/`

`Axiom.Spec/` es el repositorio canónico de especificación del workspace: contiene las specs 00–08, el contexto técnico, los incrementos y bugs canónicos bajo `specs/increments/` y `specs/bugs/`, los planes, los prompts, las decisiones gestionadas bajo `specs/decisions/` y los ADR bajo `specs/adr/`. Cuando forma parte de una topología schema 2, su identidad se declara en el único manifest autoral `axiomRepo/axiom.config/topology.yaml`; no existe un campo `specRepo` ni una copia autoral por repositorio. El `axiom.yaml` de un repo `code` o `legacy` sólo conserva identidad y un puntero local `axiomRepo` hacia esa autoridad.

`Axiom/axiom.spec/` es una baseline product-owned dentro del repositorio runtime: contiene incrementos, planes, agentes objetivo, skills objetivo y plantillas que consumen catálogos, adapters, readiness y el artifact store. Es legítima en su ubicación actual y no se mueve, elimina ni renombra por su similitud nominal con `Axiom.Spec/` (ADR-0032).

### `profiles.yaml`: dato producto canónico con default bundleado (no scaffoldeado)

> `BUG-20260703-configure-needs-bundled-profiles`. A diferencia del resto del catálogo (que documenta política del proyecto adoptante), `profiles.yaml` es dato **producto** — idéntico entre proyectos, no específico de cada uno. `axiom init` **no** lo scaffoldea. `@axiom/install-profiles` exporta `DEFAULT_PROFILES: ProfilesYaml` con el perfil funcional único `builder`, la política única `local-only` y los 8 adapter targets activos soportados por el CLI. `copilot-vscode` no pertenece a `allowedTargets`; si subsiste en `init.json`, `axiom configure` lo migra y persiste como `github-copilot` antes de instalar o despachar. LiteLLM fue retirado.
>
> - `installProfile` (`@axiom/installer`) usa `axiom.config/profiles.yaml` del proyecto cuando existe y es válido (override). Solo cae a `DEFAULT_PROFILES` cuando el archivo está ausente; si el archivo presente no se puede leer, no es YAML válido o no cumple el schema, devuelve `invalid-profiles-yaml`.
> - `axiom init` no expone selector de perfil u overlay; siempre persiste `builder` + `local-only` y conserva el target elegido.
> - Un proyecto puede crear su propio `axiom.config/profiles.yaml`; si está ausente se usa el catálogo bundleado, y si está presente pero inválido se devuelve `invalid-profiles-yaml`.
> - Los estados antiguos con `product-owner`, aliases, `standard` o `enterprise` se normalizan en los bordes de lectura. No se escriben esos valores en nuevos estados.

## Modelo de capabilities y providers

`capabilities.yaml` declara `capabilities.required` / `.optional` / `.postMvpOptional`, `supportLevels` y `degradationPolicy`. El modelo separa 16 capabilities provider-routed en los dominios `sdd`, `spec`, `code` y `memory` de 3 capabilities MCP-only bajo `mcpOnlyCapabilities` y el dominio `axiom` (`axiom.topologyRead`, `axiom.migrationManifestRead`, `axiom.adoptionStateRead`). `providers.yaml` declara cuatro providers locales y un único discovery profile `local-only` con `discoveryOrder: [filesystem]`; no declara gateway ni generated snapshots. `CC-004` sirve 13/16 y deja tres capabilities opcionales como warning.

## Ficheros generados por comando (ciclo de vida)

| Comando | Escribe |
|---|---|
| `init` | `axiom.yaml`, `.gitignore`, `.axiom-state/local/`, `.axiom-state/<projectKey>/`, `init.json` — **ya no** escribe `axiom.config/topology.yaml` (`INC-20260703-config-dedup`, dedup #2; ver "Topología de repos..." más abajo) |
| `join` | `.axiom-state/<projectKey>/members.yaml` |
| `configure` | `.axiom-state/<projectKey>/install-profile.json` y `workspace.json` si recibe providers (+ surfaces del target) |
| `sync` | `.axiom-state/<projectKey>/last-sync.json` (+ regeneración de outputs del adapter) |
| `start` | `.axiom-state/<projectKey>/last-start.json` |
| `upgrade` | `.axiom-state/<projectKey>/managed-state.json`, checkpoints y refresh de `docs/axiom/**` |
| `workspace setup` / `workspace adopt` | `docs/axiom/**` en el repositorio autoral, incluido `docs/axiom/manifest.json` |
| `toolchain upgrade` | `.axiom-state/<projectKey>/toolchain.lock` (schema 1), con checkpoint/rollback |
| `model set/unset/reset` | `.axiom-state/<projectKey>/model-assignments.json` (+ `.opencode/model-routing.json` si target es opencode) |
| `components install/uninstall` | `.axiom-state/<projectKey>/components-state.json` |

Los readers de workspace/providers, toolchain y worktree reciben el
`projectKey` de la resolución o de `Execution.projectId`; nunca deben elegir
el primer `workspace.json` o marker de otro namespace.

### Manual runtime distribuido

`Axiom/docs/**` es la fuente única del manual operativo del runtime. Durante el
build, `scripts/generate-manual-bundle.mjs` recorre ese árbol y genera el bundle
TypeScript de `@axiom/document-bootstrap`; no copia archivos de
`Axiom.Spec/specs/manuales/**`, `context/**` ni incrementos.

`distributeManual(rootPath)` materializa el bundle en `docs/axiom/` del repo
autoral y mantiene `docs/axiom/manifest.json` con `schemaVersion: 1`, el
`sourceHash` del bundle y hashes SHA-256 por archivo. Setup, adopción y upgrade
usan este writer único. Un archivo intacto se crea o refresca por contenido;
una edición local se conserva, se reporta como `stale` y recibe la versión nueva
en `docs/axiom/.stale/`. Preview no escribe y una segunda ejecución sin cambios
es un no-op.

## Ficheros generados por adapter target

| Target | Archivos |
|---|---|
| `opencode` | `.opencode/AGENTS.md`, `.opencode/skills-lock.yaml` |
| `claude-code` | `.claude/AGENTS.md` |
| `github-copilot` | `.github/copilot-instructions.md` |
| `vscode` | `.vscode/settings.json` |
| `cursor` | `.cursor/settings.json`, `.cursor/AGENTS.md`, `.cursor/rules/axiom-common.mdc` (potencial no-clobber) |
| `antigravity` | `.antigravity/AGENTS.md` |
| `visual-studio-2026` | `.github/copilot-instructions.md` (común; no genera `.vs/AXIOM.md`) |
| `codex` | `.codex/AGENTS.md` |

La instrucción general de `github-copilot` es
`.github/copilot-instructions.md`; `.vscode/` queda para configuración de VS
Code y MCP. Las instrucciones específicas por superficie se escriben bajo
`.github/instructions/<id>.instructions.md`. `configure`, `sync`, `workspace
setup` y el adapter de GitHub Copilot usan el writer común de
`@axiom/document-bootstrap`, con fallback bundleado y preservación de las
zonas humanas. Si `init.json#profileTriple.adapterTarget` conserva el literal
histórico `copilot-vscode`, únicamente `configure` lo migra y persiste como
`github-copilot` antes de instalar o despachar.

## Regla sobre YAML globales

Un YAML global del producto no debe mezclar visión documental, builder tooling y runtime. Cada YAML debe pertenecer a una capa concreta y a una responsabilidad concreta. Vigente: el catálogo de `axiom.config/*.yaml` ya está desglosado por responsabilidad concreta (policy, capability, telemetry, routing) en vez de un único fichero monolítico.

## Topología de repos y registro global (contrato vigente, R13)

El modelo de topología vigente usa exclusivamente `TopologyManifest.schemaVersion: 2`. El manifest es autoral, versionado y único por proyecto; no es una proyección por repositorio ni un dato que se derive silenciosamente desde `axiom.yaml`. El modo puede ser `single-repo` o `multi-repo`, pero ambos usan la misma forma schema 2.

### `TopologyManifest` (`Axiom/packages/topology/src/types.ts`, `schemaVersion: 2`)

```ts
interface TopologyManifest {
  schemaVersion: 2;
  mode: 'single-repo' | 'multi-repo';
  axiomRepo: AxiomRepoRef;
  codeRepos: readonly CodeRepoRef[];
  legacyRepos: readonly LegacyRepoRef[];
  roles: readonly RoleDef[];
  assignments: readonly RoleAssignment[];
  qaLane: 'inline' | 'parallel';
}
```

`axiomRepo` es la autoridad del grafo y contiene el único fichero autoral
`<axiomRepo>/axiom.config/topology.yaml`. `codeRepos` declara repositorios de
producto/implementación; `legacyRepos` declara fuentes de solo lectura y exige
`mode: 'read-only-source'` más `legacyFunction`. Las referencias son una unión
discriminada: los repos `axiom` y `code` no pueden llevar campos de legacy, y un
repo `legacy` no puede omitirlos. Los buckets antiguos `sddRepo`, `specRepo` y
`roleCodeRepositories` no forman parte del contrato activo y no se auto-mapean.

La autoridad se resuelve desde el `axiomRepo` pointer del `axiom.yaml` local de
un repo `code` o `legacy`; ese archivo conserva identidad y puntero, no el grafo
completo. El loader exige pointer válido, autoridad existente, autoridad de tipo
`axiom`, YAML parseable, `schemaVersion: 2` y forma/semántica válidas. La ausencia,
malformación, schema desconocido, autoridad ausente o autoridad no-`axiom`
terminan en error fail-closed: no se reabre fallback desde `axiom.yaml` ni se
acepta una copia local como autoridad. `defaultSingleRepoManifest()` solo es el
default local del propio repo `axiom` y su `authority.source === 'default'` no
habilita proyección de spec.

`LocalBindings` es un dominio separado y usa `schemaVersion: 2` con
`localPaths: Record<string, string>` bajo `.axiom-state/local/`. Solo admite IDs
presentes en el manifest autoral y paths locales absolutos canonicalizados; una
ref remota sin materializar no se persiste como path. La inspección distingue
`present-directory | missing | not-directory | remote-unmaterialized` y los
loaders devuelven `TopologyError` ante schema futuro, corrupción, ID desconocido,
path inválido o I/O: ningún consumidor sensible cae a `{}` ni oculta la causa.

`workspace.json` tiene un único contrato `WorkspaceStateV1` (`schemaVersion: 1`):
`projectId`, arrays ordenados/deduplicados `adapters` y `providers`, extensiones
opcionales `profile`/`overlay`, `createdAt` inmutable y `updatedAt`. La ausencia
`ENOENT` produce un estado inicial solo para una operación autorizada; JSON
inválido, schema futuro, campos desconocidos, tipos/valores inválidos o identidad
divergente abortan sin overwrite. `updateWorkspaceState` ejecuta read-modify-write
bajo el lock común, valida el temporal, renombra atómicamente y conserva bytes y
`updatedAt` en un no-op.

Las mutaciones de identidad, topología, bindings, workspace state, init state y
registro solicitado se describen mediante `StructuralMutationPlan`. Su journal
vive en `.axiom-state/<projectKey>/structural-transactions/<operationId>/` y
registra intención, hashes, staging y estados `prepared | committing | committed |
rolling-back | rolled-back | recovery-required`. Staging y backup son adyacentes
al destino; una recuperación solo publica/restaura bytes cuyos hashes e identidad
siguen siendo demostrables.

La resolución del scope de spec converge con esta autoridad: si el manifest
válido apunta a un directorio físico `<authority>/specs`, ese child tiene
prioridad; una autoridad dedicada con `role: spec` puede usar su raíz cuando no
existe el child; el resto conserva `undefined` y el almacén default
`axiom.spec`. Topología ausente, malformada, default-local o sin mapping no
reactiva un fallback local de proyección.

`runWorkspaceSetup` y sus mutaciones escriben el manifest únicamente en la
autoridad. Los repos `code` y `legacy` reciben identidad/puntero local cuando
corresponde, pero no copias de `topology.yaml` ni del grafo. Para una autoridad
`spec`, `buildRoleAwareAxiomYaml` usa el `topologyId` de `manifest.axiomRepo` y
no un id derivado alternativo; el `axiom.yaml` de los repos de código conserva
solo el puntero hacia esa autoridad.

### Dos ejes de "rol", desacoplados (Decision D5, `INC-20260710-dynamic-team-roles`)

Axiom tiene DOS conceptos distintos de "rol" que no deben conflarse:

1. **Configuración funcional** (`axiom.config/profiles.yaml#functionalProfiles`): `builder` implícito. Es el eje de **CAPACIDAD** y no se selecciona mediante `roles`; los estados legacy se normalizan al leerlos.
2. **Team/code roles** (`topology.yaml#roles`, `RoleDef { id; description? }`, campo OPCIONAL y aditivo del `TopologyManifest`): backend, frontend, mobile, qa, devops, o cualquier otro nombre — los roles que DUEÑAN repos de código. Eje de **EQUIPO/CÓDIGO**, deliberadamente sin un catálogo fijo: el arquitecto registra 1..N roles, uno por uno, vía `axiom roles register <id> [--description] [--repo]` (idempotente); `axiom roles unregister <id>` los quita (bloquea si el rol todavía tiene asignaciones activas — no cascadea el borrado). `axiom roles list` muestra ambos ejes por separado (`--json`: `{ installProfiles, teamRoles }`).

`validateTopology` (`@axiom/topology`) sigue tomando un `ReadonlySet<string>` de IDs válidos — no lee `manifest.roles` por sí mismo. Cada caller construye ese set como la UNIÓN de: `manifest.roles[].id` (eje 2) ∪ `profiles.yaml#functionalProfiles[].id` (eje 1) ∪ `functionalProfiles[].activatesImplementationRoles`. Callers actuales que ya construyen esta unión: `axiom topology validate` (`apps/cli/src/commands/topology.ts`; además cae a `DEFAULT_PROFILES` de `@axiom/install-profiles` cuando `profiles.yaml` está ausente, para no producir un set vacío sólo por eso) y el check `TC-001` de `@axiom/doctor`. `axiom roles assign`/`unassign` resuelven `--role` contra CUALQUIERA de los dos ejes.

Los dos productores conocidos de `topology.yaml` (`runWorkspaceSetup` vía `buildTopologyManifest`, y `runRepoAdd` vía su reconstrucción inline en `workspace-incremental.ts`) registran automáticamente cada rol funcional/de equipo elegido por el arquitecto en `roles` (helper compartido `buildRoleDefs`, exportado de `workspace-setup.ts`) y scaffoldean `axiom.config/profiles.yaml` (seed `DEFAULT_PROFILES`) si falta, best-effort, vía `scaffoldProfilesYamlIfMissing` — sin esto, todo `topology.yaml` generado por el wizard fallaba `axiom topology validate` con `unknown-role`, porque `assignments[].roleId` siempre referenciaba el eje 2 mientras el validador sólo conocía el eje 1.

### `PlanMetadata.roles` — role-split real de planes (P1-5, `INC-20260710-plan-role-split`)

Antes de este increment, `PlanMetadata` (`Axiom/packages/workflow/src/artifact-store.ts`) no tenía ningún campo de role-split, y `axiom-plan create` siempre dejaba `targetRepos`/`allowedWriteScope` vacíos — un plan nunca separaba trabajo por rol en la práctica, aunque `plan-metadata-template.yaml#roles` ya definía la forma. Cerrado: `PlanMetadata.roles?: { required: string[]; roleFiles: PlanRoleFileEntry[] }` es OPCIONAL (back-compat total — un `metadata.yml` sin este campo sigue parseando; el campo nunca se escribe ni siquiera como `undefined`).

`axiom-plan create` deriva el role-split de la registry DINÁMICA de team roles (eje 2 arriba, `topology.yaml#roles`/`#assignments`) — nunca de una lista fija:

- Sin `--roles`: usa TODOS los `manifest.roles[].id` registrados.
- Con `--roles <csv>` explícito: usa esa lista en vez de la registry.
- Para cada rol: agrega un entry a `roles.roleFiles` (`role`, `slug`, `file: role-<slug>.md`, `status: 'queued'`) y escribe un stub `role-<slug>.md` (un único archivo plano por rol, junto a `metadata.yml` — sin jerarquías autogeneradas profundas, límite de bootstrap de `AGENTS.md`).
- `targetRepos`/`allowedWriteScope` se pueblan con la UNIÓN de los repos que `topology.yaml#assignments` asigna a esos roles, con un glob `'**'` (repo completo) por repo — el modelo de topología asigna roles a REPOS enteros, no a sub-paths dentro de un repo, así que `'**'` es el default correcto, no un placeholder. Un `--target-repo` explícito sigue ganando (comportamiento sin cambios).
- Sin `topology.yaml` o sin roles registrados: degrada a un role-split vacío (`roles: {required: [], roleFiles: []}`) + una nota clara en el mensaje de resultado — nunca crashea.

Esto es lo que hace que `axiom validate changes`/`WS-001` (ver [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md)) tengan por fin algo real contra qué comparar: antes de este fix, `allowedWriteScope` de un plan recién creado estaba siempre vacío, así que la primitiva `validateWriteScope` (sin cambios de código — ya era genérica) no tenía nada que hacer cumplir.

### Catálogo user-level único (`~/.axiom/projects.yml`)

`Axiom/packages/user-workspace/src/registry.ts` mantiene un único `ProjectsFile` YAML (`schemaVersion: 2`). La ausencia del fichero equivale a un catálogo vacío; `~/.axiom/registry.json` no tiene lector, writer, tipos, error de compatibilidad ni migrador activos y nunca se modifica. Cada `ProjectEntryV2` contiene `id`, `name`, `addedAt`, `lastUsedAt` y un mapa no vacío `repos`, cuyas entradas llevan `role` y path absoluto.

`loadRegistryV2` valida estrictamente el documento antes de devolverlo: schema cerrado, clave/id coherentes, strings no vacíos, timestamps ISO, role igual a su key, paths válidos y ownership global no ambiguo. Los IDs explícitos son slugs ASCII; `slugifyProjectId` deriva desde NFKC, elimina diacríticos y colapsa separadores. Los paths se comparan por identidad canónica, con realpath cuando existe el destino y case-folding en Windows. `addProjectV2`/`upsertProjectReposV2` rechazan colisiones de ID, identidad o path sin escribir; un upsert solo es idempotente cuando preserva identidad y ownership.

Toda mutación ejecuta load→validate→modify→save dentro de `withLocalFileLockSync`. `saveRegistryV2` usa `atomicWriteFileSync`: temporal PID+UUID, flush/fsync, validación y rename atómico. El lock publica un owner durable bajo un lease de inicialización y usa reclaim claims ligados a generación/epoch, por lo que un reclaim retrasado no puede retirar el lock sucesor. La recuperación está limitada a artefactos propios y huérfanos.

`listProjectsV2` ordena por `lastUsedAt` descendente y después `id` ascendente. Cada repo expone `present-directory | missing | not-directory`; el proyecto agrega `available | partial | unavailable`. La resolubilidad Axiom es una proyección separada, calculada solo cuando el caller solicita y aporta el probe. `useProjectV2` modifica exclusivamente `lastUsedAt`; no existe un proyecto activo global.

### `axiom.yaml` — `schemaVersion: 2` (cutover cerrado)

`AxiomYamlSchemaV2` (`projectId`/`name`/`repoId`/`role`/`mode`/`paths`), emitido por `buildAxiomYaml` en `apps/cli/src/commands/init.ts` para ambos layouts de scaffold (`self-hosted` e `installed-multi-repo`). `resolveProject` (`@axiom/project-resolution`) y los checks `MC-001`/`BC-001`/`BC-002` de `@axiom/doctor` son conscientes de versión: aceptan tanto `schemaVersion: 1` (legado — `project.mode` + `scopes: { name: { path, product_runtime } }`) como `schemaVersion: 2`, nunca resolviendo mal uno como el otro. Este cutover quedó cerrado tras verificación end-to-end dos veces por dos roles distintos (incluyendo una reproducción de test negativo y una re-ejecución independiente de `axiom doctor` contra un proyecto v2 separado). Durante el cierre se encontraron y corrigieron dos consumidores v1-only no detectados antes: `configure.ts`'s `readAxiomYamlProjectName` (habría fallado duro para proyectos v2 sin `product.manifest.yaml`) y `@axiom/topology/src/loader.ts`'s `tryLoadTopologyHint` (habría resuelto mal, en silencio, la topología de un proyecto v2 `installed-multi-repo` al default single-repo).

`role` (`REPO_ROLES`: `sdd|spec|code`), `layout` (`PROJECT_LAYOUTS`) y `ADAPTER_TARGETS` son const arrays exportados por `apps/cli/src/commands/init.ts`, fuente única para la validación de `runInit` y para las opciones del launcher de onboarding. `builder` y `local-only` no son selectores.

El `role` **persistido** en el `axiom.yaml` emitido por `buildAxiomYaml` sigue un contrato dual por `kind` (INC-20260913-baseline-group-b-init-role-identity): `axiom` para el repo de autoridad (`kind: axiom`; el repoRole interno `sdd` nunca se filtra a la identidad persistida) y `sdd` para el control-repo legacy del layout default `installed-multi-repo` (`kind: legacy`, con `legacyFunction: sdd`), contrato que consumen el fan-out de `axiom upgrade` y el guard de repo-affinity. `expectedIdentity` (`workspace-structural-plan.ts`) valida la identidad con match estricto (`optionalRoleMatches` sin alias), por lo que cualquier emisor de `axiom.yaml` debe respetar ese contrato dual; la unificación del vocabulario de roles queda como deuda futura.

### Setup/adopción multi-repo y unidad estructural

`runWorkspaceSetup` y `runWorkspaceAdopt` comparten el modelo de una autoridad
`axiomRepo`, 0..N repos `code` y fuentes `legacy` read-only. CLI y launcher usan
los mismos runners: preview y preflight no escriben; apply requiere
confirmación en la superficie que corresponda.

El plan estructural cubre:

- el bloque gestionado de `axiom.yaml` por repo, con identidad y puntero a la autoridad;
- el único `<axiomRepo>/axiom.config/topology.yaml` schema 2;
- `.axiom-state/local/topology-bindings.yaml` schema 2;
- `.axiom-state/<projectKey>/workspace.json` e `init.json`;
- `~/.axiom/projects.yml` solo cuando se solicita registro;
- los directorios creados por la propia operación.

Antes del primer write se validan todos los repos, IDs, flags `create`,
ownership, solapamientos y destinos derivados. Setup, `repo add` y `role add`
rechazan un `axiom.yaml` foráneo, inválido o ambiguo; adopt puede preservar como
`skipped` una identidad foránea válida, pero no reclamarla. Apply adquiere locks
en orden, recupera journals incompletos, replantea bajo lock y comprueba bytes,
metadata, tipo, `realpath` e identidad física. Los recursos terminan en
`committed`, rollback demostrable o `recovery-required`.

### Registro opcional dentro de la transacción

El catálogo no es autoridad del grafo y no se necesita para resolver un
workspace local. Con registro solicitado, `projects.yml` es un recurso
estructural: validación, ownership, I/O o timeout fallidos abortan/revierten la
unidad. `--no-register` omite su persistencia y no crea home; no desactiva las
comprobaciones read-only de ownership necesarias cuando el catálogo ya existe.
`registry.json` residual se ignora y nunca se migra. Solo después del commit se
ejecutan config MCP, adapters, skills, reglas, catálogos y base de spec; sus
fallos quedan en warnings derivadas.

### Artefactos adicionales del setup de workspace (contrato vigente tras R-13)

Las tandas de workspace incorporaron estado persistido y materializaciones que hoy se gobiernan mediante `WORKSPACE_STEP_CATALOG`. El catálogo separa recursos estructurales requeridos de outputs derivados: `workspace.json` pertenece al commit estructural y **no** es best-effort; adapters, skills, reglas, MCP, catálogos y base de spec se materializan solo después de que esa unidad haya quedado `committed`. El comportamiento y las tablas de despacho viven en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md); aquí se documentan las formas de datos.

- **`<axiomRepo>/.axiom-state/<projectKey>/workspace.json`** — único estado persistido de selección de adapters/providers. Su contrato cerrado es:

  ```ts
  interface WorkspaceStateV1 {
    readonly schemaVersion: 1;
    readonly projectId: string;
    readonly adapters: readonly string[];
    readonly providers: readonly string[];
    readonly profile?: string;  // extensión de compatibilidad
    readonly overlay?: string;  // extensión de compatibilidad
    readonly createdAt: string; // ISO-8601, inmutable
    readonly updatedAt: string; // ISO-8601
  }
  ```

  `projectId` debe coincidir con el proyecto resuelto; adapters y providers se normalizan como arrays ordenados sin duplicados. Solo `ENOENT` representa ausencia y permite inicializar el documento dentro de una mutación autorizada. JSON inválido, schema futuro, campos desconocidos, tipos/valores inválidos o identidad divergente abortan sin overwrite. `updateWorkspaceState` ejecuta read-modify-write bajo el lock local común, valida el temporal, conserva `createdAt` y no reescribe bytes ni `updatedAt` en un no-op.

  El campo `providers` persiste la selección local habilitada del proyecto, distinta del catálogo completo `axiom.config/providers.yaml`. Lo actualizan las superficies add/enable —wizard, `axiom configure --providers` y `provider add`— mediante el contrato anterior; `buildProjectProviderRegistry` lo lee para registrar exactamente los clientes code-intel habilitados. Las operaciones repair/regenerate no incorporan capacidades nuevas.

- **`install-profile.json` por repo** — `generateWorkspaceAdapters` resuelve un `ResolvedInstallProfile` por repo del workspace y lo persiste bajo `.axiom-state/<projectKey>/`. Los ficheros de adapter (`.opencode/AGENTS.md`, `.claude/AGENTS.md`, `.antigravity/AGENTS.md`, `.github/copilot-instructions.md`, etc.) son outputs derivados post-commit y se escriben por adapter seleccionado según la tabla de despacho de [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

- **Baseline de skills en la autoridad `axiomRepo`** (`INC-20260705-workspace-sdd-skills`):
  - `axiom.config/skills-catalog.yaml` (`schemaVersion: 1`) — catálogo semilla de 5 ids con `bundleHash` byte-exacto por entrada (`computeSkillBundleHash`); las fuentes de cada skill se escriben bajo `axiom.spec/target-axiom-skills/<id>.md`. Es el mismo `SkillsCatalog` que consume el check `TC-010` de doctor (ver [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md)).
  - `.opencode/agents/<id>/SKILL.md` (materializado por `applySkillSet`) + `.axiom-state/<projectKey>/skills-pending.json`.
  - `axiom.config/skills-index/<role>.yaml` — un `SkillsRoleIndex` (schema de RF-AXM-020, ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md)) por cada rol funcional declarado en el workspace.

- **Base de spec en la autoridad** (`INC-20260705-workspace-spec-base`): estructura canónica de spec + contexto técnico, scaffoldeada desde plantillas bundleadas con guardado por fichero; nunca sobrescribe un fichero preexistente y reporta `skipped`/warning:
  - `specs/README.md` + `specs/00_Resumen_Ejecutivo.md` .. `specs/08_Glosario.md` (9 ficheros numerados);
  - `context/TECHNICAL_CONTEXT.md` + `context/README.md`;
  - directorios estructurales vacíos (vía `.gitkeep`): `specs/{increments,bugs,archive}/` y `context/{architecture,integrations,operations,references}/`.

**Semántica vigente de ejecución**: `WORKSPACE_STEP_CATALOG` es la única fuente declarativa de steps, owners, preflight, prerequisites y clases de output para setup y reparaciones granulares. `identity`, `topology`, `workspace-state` y `registry` alimentan la frontera estructural común: todos sus targets y también los destinos derivados se prevalidan antes del primer write. Tras el commit, los steps derivados se aíslan entre sí; un fallo se conserva como warning tipada con estado estructural `committed` y `exitCode: 0`, y no impide ejecutar los steps posteriores. En setup, los predicates de creación se calculan desde outcomes estructurales confiables; los comandos granulares pueden reparar targets ya declarados. Add/enable puede cambiar `WorkspaceStateV1`; repair/regenerate solo materializa lo ya habilitado.

### Autoskills por repo de código (round 3, `INC-20260705-workspace-code-repo-skills`)

Además de la baseline de skills del repo de control (round 2, arriba), cada repo de CÓDIGO/rol **recién creado** (`kind === 'role'` con su propio `created === true`) recibe ahora su propia baseline de skills scoped a su rol, escrita dentro de ESE repo de código (no solo en el de control). Reusa la misma semilla bundleada y la misma maquinaria de `@axiom/skills` que el repo de control (sin duplicación); las formas de datos por repo de código creado son:

- `axiom.config/skills-catalog.yaml` — el mismo `SkillsCatalog` (`schemaVersion: 1`, semilla de 5 ids) que el repo de control, más las fuentes bundleadas bajo `axiom.spec/target-axiom-skills/<id>.md`.
- `.opencode/agents/<id>/SKILL.md` (materializados por `applySkillSet`) + `.axiom-state/<projectId>/skills-pending.json` (con `projectId = effectiveProjectId`, consistente entre todos los repos del workspace).
- **UN único `axiom.config/skills-index/<roleId>.yaml`** scoped al propio rol (posiblemente custom) de ese repo (`role: roleId`, `repoKinds: ['role']`), donde `roleId = repo.functionalRoleId ?? repo.roleKey` — a diferencia del repo de control, que escribe un `skills-index` por CADA rol funcional del workspace.

En setup es un output derivado post-commit cuyo predicate se basa en el outcome estructural `created` del repo de código; un repo preexistente (`create: false`) no se reclama ni se clobbera. `axiom workspace skills` puede reparar la baseline de targets ya declarados y prevalidados. Cualquier fallo de materialización queda como warning tipada sin alterar el commit estructural. La autoridad sigue usando el executor compartido `scaffoldSddSkills`; el repo dedicado a spec no recibe skills. El comportamiento vive en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

### Artefactos y formas de datos de la tanda INC-20260708-* (providers, memoria, reglas, operaciones incrementales)

Formas de datos añadidas por esta tanda (el comportamiento vive en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md); aquí solo las estructuras persistidas y su ancla):

- **Selección de providers en `workspace.json#providers`** (`INC-20260708-wizard-configure-provider-selection`): usa `WorkspaceStateV1` y su writer lockeado descritos arriba. Es estado project-scoped bajo `<axiomRepo>/.axiom-state/<projectKey>/workspace.json`; `buildProjectProviderRegistry` (`@axiom/providers/project-registry.ts`) lo lee mediante el `projectKey` resuelto y aliases legacy explícitos, sin escanear otro proyecto. Devuelve un `ProviderRegistry` con los clientes code-intel habilitados + `filesystem` always-on, más `engramEnabled` (Engram no es un `ProviderClient`: se resuelve vía `resolveMemoryBackend`).

- **Modelo de memoria Engram-only (R-12)**: `MemoryEntry` mantiene `topicKey?`/`sessionId?` (aditivos), `MemorySessionSummary` y `MemoryBackend.saveSessionSummary?`. El UPSERT topic-keyed se implementa mediante `topic_key` de Engram: una misma combinación `(projectId, topicKey)` reemplaza la entrada correspondiente. `resolveMemoryBackend` inicia exclusivamente `engram mcp --project=<projectId> --tools=agent`, hace el handshake estándar `initialize` y devuelve un `MemoryError` ante cualquier indisponibilidad; no existen backend JSON, `createInMemoryBackend`, `memoryFilePath`, `forceJson` ni fallback de persistencia activos. Los JSON históricos quedan intactos y sin consumo runtime. TC-024 verifica de forma segura `engram --version`; su ausencia es un fallo de Doctor con guía de instalación.

- **Metadata de fase en `MemoryEntry`** (`INC-20260729-knowledge-phase-metadata`): 7 campos opcionales aditivos para trazabilidad SDD — `increment?` (ID del incremento/bug/plan), `phase?` (`SddPhase`: `analysis`|`architecture`|`frontend`|`backend`|`qa`|`validator`|`archive`), `actorRole?` (`ActorRole`: `analyst`|`architect`|`frontend`|`backend`|`qa`|`validator`|`orchestrator`), `knowledgeKind?` (`KnowledgeKind`: `decision`|`constraint`|`discovery`|`bugfix`|`gotcha`|`pattern`|`risk`|`open-question`|`workaround`|`convention`), `stability?` (`Stability`: `temporary`|`candidate-project-context`|`candidate-skill`|`historical-only`), `visibility?` (`Visibility`: `project-shared`|`private`), `sourceArtifact?` (path al artefacto fuente). Una entrada sin estos campos sigue siendo válida. Engram codifica la metadata como frontmatter YAML-like (`---\n...\n---\n`) al inicio de `content`; al leer, `mem_get_observation` decodifica el contenido completo, incluido el envelope `result` real de Engram 1.17. `phase-metadata.ts` aporta `encodePhaseMetadata`/`decodePhaseMetadata`.

- **Comando `axiom knowledge harvest`** (`INC-20260729-knowledge-harvest-command`): `axiom knowledge harvest --increment <id> [--spec-repo <path>] [--dry-run]` lee las memorias Engram del proyecto activo, filtra por `entry.increment`, clasifica por `stability` (`candidate-project-context` → propuesta de contexto técnico, `candidate-skill` → propuesta de skill, resto → histórico) y genera `knowledge-harvest.md` en `<specRepo>/specs/increments/<id>/`. Es read-only respecto de contexto y skills; `--dry-run` imprime y no escribe; no-clobber rechaza un archivo existente. Engram no disponible se informa como error, nunca como harvest vacío equivalente a éxito.

- **Comandos `axiom knowledge sync` y `axiom knowledge pull`** (`INC-20260820-r11-knowledge-sync-hardening`): `sync --increment <id> --phase <phase>` es preview por defecto; solo `--confirm` escribe chunks y ejecuta Git, y `--push` habilita además el envío remoto. Exporta únicamente entradas con `visibility: project-shared`, preservando `rationale`, `source` y toda metadata estable; omite y contabiliza las privadas, sin visibilidad o con secretos detectados en cualquier campo textual serializado. `pull` no acepta `--increment`: con `--confirm` importa todos los chunks pendientes. Valida manifest, chunk y memoria; las escrituras son atómicas e idempotentes y un chunk solo se marca importado tras persistir todas sus entradas válidas. El marker personal se guarda en `.axiom-state/<projectKey>/knowledge/imported-chunks.json`; el legacy `.engram/.imported` se migra o ignora sin volverlo versionable. `.engram/engram.db` permanece gitignored.

### Índice derivado de contexto técnico y selector por etiquetas (`INC-20260820-r11-context-tag-selection`)

`axiom context index` genera, mediante escritura atómica, `technical-context/indexes/<rol>.index.yml` a partir de `context/**/*.md`; el índice es derivado y no se edita a mano. Cada documento puede declarar exclusivamente `tags: [..]` en frontmatter YAML. Las tags se normalizan, ordenan y validan; si no hay frontmatter, se conserva una única tag de fallback derivada de la primera carpeta (`repo` para documentos bajo la raíz de `context/`). El selector compartido devuelve, sin duplicar paths, `mandatory.always`, después los grupos `mandatory.whenTags` cuyo conjunto completo de tags coincide y, solo cuando hay tags de tarea, los `available` que coinciden con al menos una. La clasificación `mandatory` o `recommended` forma parte del resultado; ausencia o lista vacía de tags devuelve exclusivamente el contexto obligatorio.

- **Artefactos de la capa de reglas** (`INC-20260708-rules-layer`): `axiom.config/rules/<scope>.md` — `common.md` (siempre) + `<language>.md` por lenguaje inferido (`typescript`/`python`/`csharp`/`angular`). Ubicación canónica análoga a `axiom.config/skills-*`, escrita por `scaffoldRules` best-effort no-clobber por fichero. El `AGENTS.md` canónico gana un campo `CanonicalAgentsMdIdentity.ruleScopes?` (poblado por el caller leyendo disco en tiempo de render) que lista los scopes presentes. Proyección nativa opcional: `.cursor/rules/axiom-common.mdc` (solo `common`, no-clobber).

- **Scaffold canónico del propio repo `Axiom/` (fotografía histórica de `INC-20260708-product-repo-self-bootstrap`)**: el repo de producto ganó en su raíz el set canónico que su runtime/tests esperaban — `axiom.config/` con contenido schema-válido real (`skills-catalog.yaml`, `agents-catalog.yaml`, `model-routing-policy.yaml`, `profiles.yaml`, `providers.yaml`, `capabilities.yaml`, `integrations.yaml`, `policy-as-code.yaml`, `mcp-manifest.yaml`, `telemetry-sinks.yaml`), `axiom.spec/target-axiom-skills/*.md` (20), `axiom.spec/target-axiom-agents/*.md` (14), `axiom.spec/templates/` (copiadas entonces desde la copia del repositorio canónico), `AGENTS.md` y `axiom.skills.lock`. El cierre de aquella tanda registró `readiness:first-project` y `doctor` verdes; esa fotografía histórica fue superada por la verificación del 2026-08-02, que devuelve ambos comandos en `PASS` (ver [00_Resumen_Ejecutivo.md](00_Resumen_Ejecutivo.md) y [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md)). La fuente vigente de plantillas es `Axiom/axiom.spec/templates/`; `profiles.yaml#allowedTargets` declara los 8 targets activos validados por `IP-003`; `copilot-vscode` no pertenece al conjunto público y únicamente se migra, si ya está persistido en `init.json`, durante `configure` antes de instalar o despachar. LiteLLM fue retirado.

- **Idempotencia de las operaciones incrementales** (`INC-20260708-incremental-operations`, reconciliada por R-13): `repo add` y `role add` resuelven el proyecto contra la autoridad topology schema 2 y producen un `StructuralMutationPlan`. La unidad incluye el bloque gestionado de identidad, el único manifest autoral, bindings locales, `WorkspaceStateV1`/init state y `projects.yml` cuando se solicita registro; no vuelve a emitir paths recíprocos ni copias del grafo en cada `axiom.yaml`. Apply preflighta sin escribir, adquiere locks en orden, replantea bajo lock y termina en commit completo, rollback demostrable o `recovery-required`. `adapter add` y `provider add` actualizan arrays deduplicados mediante `updateWorkspaceState` y después materializan los outputs seleccionados de `WORKSPACE_STEP_CATALOG`; repair/regenerate no habilita capacidades. Repetir los mismos argumentos converge a `unchanged`/no-op sin duplicados ni clobber. Si falta `workspace.json`, solo una mutación autorizada crea la forma completa `WorkspaceStateV1`; corrupción, schema futuro o identidad divergente fallan sin overwrite.

- **Scaffolding automático de la config de adopción** (`INC-20260727-adoption-config-scaffolding`): lo que `INC-20260708-product-repo-self-bootstrap` (arriba) hizo a mano para el propio repo `Axiom/`, `runWorkspaceSetup` (motor de `axiom workspace setup`/`adopt`) lo hace ahora automáticamente para CUALQUIER proyecto adoptado/seteado — siembra en el repo de control `axiom.config/integrations.yaml`, `axiom.config/policy-as-code.yaml`, `axiom.config/agents-catalog.yaml` y el `axiom.skills.lock` raíz (best-effort, no-clobber; `agents-catalog.yaml`/`axiom.skills.lock` con `bundleHash` recomputable byte-a-byte por el doctor). Las formas de `agents-catalog.yaml`/`skills.lock` son las ya descritas (materialización `@axiom/agents`/`@axiom/skills`, TC-010/TC-011); el comportamiento del motor de scaffolding vive en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

### `mcp.yml` (config MCP por proyecto) vs `mcp-manifest.yaml`

`mcp.yml` es una declaración **por proyecto** de qué procesos servidor MCP expone un proyecto — genuinamente distinta del `mcp-manifest.yaml` preexistente (spec 0024). Las dos responden preguntas distintas y no deben fusionarse (decisión Q-mcp-1, declinada explícitamente):

| | `mcp-manifest.yaml` (spec 0024) | `mcp.yml` |
|---|---|---|
| Responde | "¿qué capabilities MCP declara el catálogo de este proyecto, y están vinculadas las obligatorias?" | "¿qué procesos servidor MCP están habilitados, en qué scope, para que los adapters generen config runtime?" |
| Forma | `McpEntry {id, displayName, capabilities, installMode, projectBinding, readonly}` | `McpServerEntry {id, type, scope, targetRepo?, enabled}` |
| Resolución | `@axiom/memory#resolveMemoryScope` | `getProjectV2` (registro v2) |
| Consumidor | `axiom mcp list\|validate\|repair\|inventory` | Generadores `mcp.json` por adapter |
| Escritura | `axiom mcp repair` / manual | El propietario del proyecto edita directamente |

Ningún loader lee el fichero del otro — sin acoplamiento runtime compartido. Schema (`@axiom/user-workspace`'s `mcp-config.ts`); el fichero vive en `<controlRepo>/.axiom/mcp.yml` — el repo de control es el ancla project-scoped que `runWorkspaceSetup` ya usa para el resto de artefactos generados no-por-repo (`axiom.config/topology.yaml`, `.axiom-state/local/topology-bindings.yaml`), y es la resolución autoritativa para futuros llamadores (INC-20260705-workspace-mcp-generation):

```ts
interface McpProjectConfig {
  schemaVersion: 1;
  projectId: string; // debe resolver vía getProjectV2
  servers: readonly McpServerEntry[];
}
interface McpServerEntry {
  id: string;
  type: string;            // abierto: 'axiom' | 'serena' | 'integration' | ...
  scope: 'project' | 'repo';
  targetRepo?: string;     // obligatorio si scope === 'repo'
  enabled: boolean;
  command?: string;        // config de lanzamiento (opcional, aditivo) — INC-20260708-mcp-launch-config-wiring
  args?: readonly string[];
  env?: Readonly<Record<string, string>>;
}
```

Los tres campos `command`/`args`/`env` son **opcionales y puramente aditivos** (`INC-20260708-mcp-launch-config-wiring`): `isMcpServerEntryLike` solo los valida por forma cuando están presentes (string / string[] / Record<string,string>), sin añadir ninguna regla semántica nueva a las reglas de `validateMcpProjectConfig`. Backward-compatible: una entrada sin ellos valida y se proyecta byte a byte igual que antes.

**Forma REAL committed de `mcp-manifest.yaml` vs forma rica interna** (`INC-20260710-schema-reconciliation`): el `axiom.config/mcp-manifest.yaml` que el propio repo `Axiom/` tiene committed declara solo la forma **minimal** `{id, server?, projectBinding}` — NO los campos ricos (`displayName`, `capabilities`, `installMode`, `readonly`) de la fila `Forma` de la tabla arriba, que sigue siendo el shape público (`McpEntry`) que el resto del comando `axiom mcp` consume. El reader (`apps/cli/src/commands/mcp.ts`) acepta ambas: valida la forma minimal (`id` + `projectBinding` obligatorios; el resto opcional-si-presente-debe-tipar) y luego DERIVA los campos ricos ausentes (`displayName ← id`, `capabilities ← []`, `installMode ← 'project-scoped'`, `readonly ← false`). Una entrada que sí declare los campos ricos explícitamente se respeta tal cual (no se pisan). `@axiom/doctor`'s TC-007 tiene su propio parser inline independiente, ya tolerante a la forma minimal desde antes de este incremento — no necesitó cambios.

`loadMcpProjectConfig(path)` lee + parsea YAML + valida forma únicamente (sin validación semántica). `validateMcpProjectConfig(config, homeDir)` corre seis reglas acumulativas: `schema-version`, `unknown-project`, `duplicate-server-id`, `missing-target-repo`, `unknown-target-repo`, `unexpected-target-repo`, `duplicate-type-target-repo`. Los generadores custom-shape por adapter (`generateOpencodeMcpJson`/`generateClaudeCodeMcpJson`) escriben los servers `enabled: true` verbatim en el antiguo formato custom-shape `.opencode/mcp.json`/`.claude/mcp.json`, con el mismo patrón atómico tmp-write-then-rename usado en el resto del producto; siguen exportados y testeados en sus paquetes, pero **`INC-20260708-mcp-native-config-mapping` retiró su call site de la ruta de workspace** (ver abajo y [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md)). Desde `INC-20260708-mcp-launch-config-wiring` **sí** se emiten los campos de lanzamiento `command`/`args`/`env` cuando están presentes (spread condicional por entrada, preservando el orden de clave `id, type, scope, targetRepo?, command?, args?, env?`) — esto supersede la nota previa de "no se inventan campos de transporte": los que emite `runWorkspaceSetup` apuntan al comando real `axiom mcp serve` (ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md)). El aislamiento es por scoping simple de filesystem — no requiere código de `@axiom/isolation`.

El **call site generador vivo** hoy es el setup de workspace multi-repo (`runWorkspaceSetup`, INC-20260705-workspace-mcp-generation), que escribe `.axiom/mcp.yml` en el repo de control como fuente canónica con una única entrada `axiom-mcp-broker` y launch args `axiom mcp serve --kind axiom --project-root <control>` — ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md). **`INC-20260708-mcp-native-config-mapping` cambió qué config de herramienta produce esta ruta**: en vez del custom-shape `.opencode/mcp.json`/`.claude/mcp.json`, `runWorkspaceSetup` emite ahora el **schema MCP NATIVO real** de cada herramienta seleccionada — `opencode.json` (`{ $schema, mcp:{ <id>:{ type:'local', command:[cmd,...args], enabled, environment? } } }`), `.mcp.json` y `.cursor/mcp.json` (`{ mcpServers:{ <id>:{ command, args, env? } } }`), `.vscode/mcp.json` (`{ servers:{ <id>:{ type:'stdio', command, args, env? } } }`) — por cada repo del workspace × cada adapter seleccionado, merge-preserving y atómico (`native-mcp-config.ts`), tras pasar el filtro project-bound (`filterProjectBoundMcpServers`); `codex`/`antigravity` reciben nota informativa user-global sin fichero y `visual-studio-2026` no recibe fichero (schema MCP no verificado). El custom-shape queda superseded para la ruta de workspace; `.axiom/mcp.yml` permanece canónico y sin cambios de forma. Fuera de esa ruta, ni `runConfigure` ni `sync` llaman a los generators custom-shape MCP: `runConfigure` instala el perfil, conserva un fallback acotado de generación OpenCode y escribe instrucciones Copilot para `github-copilot`; `sync` materializa los generators de `opencode`, `claude-code`, `github-copilot`, `vscode` y `cursor`, y devuelve cero outputs para los otros tres targets. `.opencode/mcp.json`/`.claude/mcp.json` siguen excluidos de `GENERATED_FILES_BY_TARGET`.

### Versionado, upgrades y migración de schema (`@axiom/versioning`)

`ManagedState` (`Axiom/packages/versioning/src/managed-state.ts`, `schemaVersion: 1`, JSON en `<root>/.axiom-state/<projectKey>/managed-state.json`): `{schemaVersion: 1, runtime: {package, version}, adapterTargets: [{id, version, lastSyncedAt}], lastUpgrade: {fromVersion, toVersion, at, checkpointId} | null, lastCheckpointId: string | null}`. Esta forma es estructuralmente distinta del `version.yml` propuesto originalmente por el documento fuente (`axiom: {projectContractVersion, installedWithAxiomVersion, lastUpgradedWithAxiomVersion}`, `assets: {sddSkillsVersion, adaptersVersion, guidesVersion, templatesVersion}`, `appliedMigrations: [...]`) — no es una variante de nombrado. Cuatro conceptos del documento fuente no tienen equivalente (`projectContractVersion`, `installedWithAxiomVersion`, los cuatro campos `assets.*`, y `appliedMigrations` como lista acumulativa — solo se conserva el `lastUpgrade` más reciente). Este hueco queda abierto (ver "Pendientes conocidos" abajo) pero ya no está bloqueado por la falta de una ruta `schemaVersion: 2`.

Mecanismo de migración/rollback confirmado real por lectura directa y ejecución completa de tests: exactamente una migración registrada (`0.0.0 -> 0.1.0`, pura, idempotente), un servicio de checkpoints funcional (crear/listar/restaurar/podar, restauración atómica, retención por defecto de 5) y un `executeUpgrade` rollback-first (checkpoint antes de mutar, `failWithRollback` restaura y relanza ante cualquier fallo posterior). `--dry-run`/`--from-checkpoint <id>`/`--target-version <v>`/`--no-sync`/`--no-doctor` son flags reales de Commander, cada uno con test dedicado. La salida de `axiom upgrade` (`formatPlan`/`formatResult`) imprime solo un resumen de transición de versión (`fromVersion`/`toVersion`/conteo de migraciones/`checkpointId`/`syncRun`/`doctorRun`) — no se muestra información de cambio a nivel fichero o por repo hoy, aunque `CheckpointRecord.files` ya guarda los paths relevantes internamente.

## Artefacto de ledger de revisión y entrada MCP nativa de engram — tanda INC-20260709-*

- **`review-ledger.md` (artefacto de revisión, INC-20260709-review-findings-ledger)**: cuando el flujo de revisión (ver [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md)) persiste su ledger de hallazgos vía el artifact store, lo hace como `review-ledger.md` DENTRO de la carpeta folder-per-artifact del cambio (`<specPath>/{increments,bugs}/<ID>/review-ledger.md`) — un fichero adicional junto a `README.md`/`metadata.yml`, no un nuevo `ArtifactKind` ni un cambio de `metadata.yml`. Si no hay carpeta de artefacto, el ledger cae a un topic de Engram (`topicKey` `sdd/<change>/review-ledger`) o a in-context. La forma del contrato (campos del ledger) se bundlea como constante TS única en `@axiom/document-bootstrap` (`review-ledger-contract.ts`).
- **Entrada MCP nativa de engram (INC-20260709-engram-mcp-stdio-native-config)**: la config MCP nativa por herramienta (ver "Config MCP nativa por herramienta" en [08_Glosario.md](08_Glosario.md) y [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md)) gana, cuando `workspace.json#providers` incluye `engram`, una entrada `engram` de stdio local (`command:'engram'`, `args:['mcp','--project',<projectId>,'--tools','agent']`) en la forma nativa de cada tool. Es machine-local y project-pinned; NO se añade a `.axiom/mcp.yml` (reservado a los brokers `type:'axiom'`). Nunca se emite forma HTTP (puerto 7437/`ENGRAM_URL`).

## Separación de responsabilidades: instalación del ARQUITECTO vs. instalación del MIEMBRO (`INC-20260710-per-member-install`)

Axiom ya separaba, de forma implícita, dato SHARED/committeado (`axiom.yaml`, `axiom.config/*.yaml`, la spec) de dato PERSONAL/no-versionado (`.axiom-state/`) — ver "Estado project-scoped" y "Topología de repos..." arriba. Este incremento hace esa separación EXPLÍCITA como modelo de dos actores y cierra los huecos que el audit encontró:

| Actor | Qué hace | Artefactos que produce | ¿Se commitea? |
|---|---|---|---|
| **ARQUITECTO** (una vez, primer install) | `axiom workspace setup` (o `axiom init`) | `axiom.yaml` (por repo), `axiom.config/topology.yaml`, `axiom.config/mcp-manifest.yaml`, `axiom.config/toolchain-catalog.yaml`, `axiom.config/toolchain.yaml` (subset habilitado del catálogo), skills seed, spec base | **Sí** — SHARED, define QUÉ repos/MCPs/utilidades existen para el proyecto |
| **MIEMBRO** (cada clone, cada máquina) | `axiom member install --member <id>` | `.axiom-state/local/topology-bindings.yaml` (paths reales en SU máquina), config MCP nativo (`.mcp.json`/`.cursor/mcp.json`/`.vscode/mcp.json`/`opencode.json`) con un launch command RESOLVABLE en SU máquina, estado local de activación de toolchain (`.axiom-state/<projectKey>/toolchain/<id>/`), su propia entrada en `.axiom-state/<projectKey>/members.yaml` y `.axiom-state/<projectKey>/init.json` | **No** — PERSONAL, resuelve DÓNDE viven los repos y CÓMO se lanza cada cosa en SU máquina |

### Hallazgos del audit (Step 0) y su fix

1. **`buildGitignore()` (`apps/cli/src/commands/init.ts`) sólo ignoraba `.axiom-state/local/`, no `.axiom-state/` completo.** El `.gitignore` real, a mano, del propio repo `Axiom/` siempre ignoró `.axiom-state/` entero ("Axiom local overlay (per-repo runtime state; never versioned)") — el generador estaba desalineado con esa convención: `.axiom-state/<projectId>/` (`members.yaml`, `init.json`, `workspace.json`, checkpoints, telemetry, state-machine) habría quedado versionable en cualquier proyecto scaffoldeado por `init`/`workspace setup`. Fix: `buildGitignore()` ahora ignora `.axiom-state/` completo (además de `.axiom/`, `node_modules/`, `dist/`).
2. **`axiom join`/`axiom member install` no podían completarse nunca sobre un proyecto bootstradeado vía `axiom workspace setup`.** El gate del orchestrator (`hasInitJson`, `@axiom/orchestrator`) exige `.axiom-state/<projectKey>/init.json` para `join-command` — sólo `axiom init` lo escribe; `axiom workspace setup` no, y aunque lo escribiera sería inútil (vive en `.axiom-state/`, personal/gitignored, nunca llega al clone de un miembro). `axiom member install` ahora sintetiza ese `init.json` local ANTES de invocar `join` (best-effort, hereda `profile`/`overlay`/adapter real de `workspace.json` si existe; nunca clobberea uno preexistente) — mismo criterio que el resto del incremento: estado local, materializado por quien lo necesita.
3. **`MCP_LAUNCH_COMMAND = 'axiom'` asumía incondicionalmente que `axiom` está en el PATH de CADA máquina.** En una máquina de un miembro recién clonada eso no está garantizado (IDE no puede lanzar el server). Fix: `resolveMcpLaunchCommand()` (`workspace-mcp.ts`) resuelve, en runtime, `axiom` si resuelve en PATH; si no, `node <cliEntryPath>` (el propio `process.argv[1]`, el entrypoint REALMENTE en ejecución — sin adivinar dónde npm/npx lo instaló); si tampoco eso es viable, preserva el comportamiento previo como último recurso. `runWorkspaceSetup` (vía `WorkspaceSetupSpec.mcpLaunchCommandOverride`, default = resolver real) y `axiom member install` lo usan ambos.
4. **Hallazgo documentado, NO corregido en este incremento (fuera de alcance deliberado):** los configs MCP nativos por adapter (`.mcp.json`, `.cursor/mcp.json`, `.vscode/mcp.json`, `opencode.json`) embeben paths absolutos `--project-root <path>` específicos de CADA máquina, pero viven en la raíz del repo (no bajo `.axiom-state/`/`.axiom/`) y hoy **no** están cubiertos por `buildGitignore()`. Si un equipo commitea el primero que se genera, otros miembros heredarían un path que no es el suyo — mitigado en la práctica porque `writeNativeMcpConfig` (native-mcp-config.ts) es merge-preserving y `axiom member install` lo regenera con el path/launch command correctos de CADA máquina en cada corrida, pero la higiene de "no debería estar en git" queda como consideración futura, no como bug corregido acá (ver AGENTS.md: documentar, no implementar especulativamente).
5. **`axiom.config/toolchain.yaml`** (el subset de utilidades que el proyecto habilitó, vía `axiom toolchain add`) es la fuente de "qué utilidades activar" para `member install` — distinto de `axiom.config/toolchain-catalog.yaml` (el catálogo global de IDs permitidos). Axiom NO instala binarios de terceros reales — `member install` sólo activa el marcador de estado LOCAL de Axiom (`repairTool`, `@axiom/toolchain`) e imprime el comando de instalación externo exacto cuando se conoce uno documentado en el propio codebase (hoy: `serena` → `uv tool install -p 3.13 serena-agent`); para el resto, dice explícitamente que no conoce un comando automatizado en vez de inventar uno.

### Estados diferenciados del toolchain (`INC-20260710-honesty-and-toolchain-states`)

El punto 5 anterior habla del "marcador de estado LOCAL" — hasta este
incremento, ese marcador (un directorio vacío creado por `repairTool`)
se reportaba con el mismo estado (`'present'`) que una instalación
real y funcional, un falso positivo confirmado por audit:
`axiom toolchain validate` podía dar por satisfecha una tool
requerida sin ningún binario real instalado. El modelo se corrigió a
4 estados diferenciados (`@axiom/toolchain`'s `ToolState`):

- `declared` — la tool sólo está en el manifest/catálogo; sin
  ninguna evidencia de filesystem (tools `supportLevel:
  'instruction-only'`, que no tienen contrato de detección).
- `absent` — ni marcador en disco ni probe real positivo.
- `marker` — el `detectionPath` existe en disco (scaffoldeado por
  `repairTool`/`member install`), pero SIN confirmación real de
  instalación funcional. Es lo que el modelo viejo llamaba
  (incorrectamente) `present`.
- `installed-working` — un probe real, best-effort y nunca-lanzante
  (`@axiom/toolchain`'s `probe.ts`: `<tool> --version` por spawn, o
  para `cmm` la evidencia alternativa de un
  `.cmm/sync-state.json` real no vacío) confirmó positivamente que
  la tool está instalada y responde.

`axiom toolchain show`/`validate` (y sus `--json`) corren ese probe
por defecto y muestran el estado diferenciado. `validate` ya NO
cuenta un `marker` desnudo como "satisfecho" para una tool requerida
sin decirlo: ahora emite el warning `required-tool-not-verified`
(nunca un error — `ok` sigue dependiendo sólo de `absent` real, para
no romper callers pre-existentes como `@axiom/doctor`'s TC-004, que
no corre el probe). El punto 5's "no-op si ya present" de la
descripción de `axiom member install` (arriba) debe leerse hoy como
"no-op si ya tiene `marker`" — `repairTool` nunca corre el probe real,
así que su output se queda siempre en `declared`/`absent`/`marker`.

### Versionado reproducible del toolchain (`INC-20260730-toolchain-versioning`)

El modelo de toolchain separa el catálogo permitido, el subset habilitado y el estado fijado:

- `axiom.config/toolchain-catalog.yaml` es el catálogo global. En schema 2, cada entrada puede declarar `versionExtractor`, versiones por canal (`stable`, `candidate`, `edge`) y `compatibility.axiomMinVersion`. Las tools sin extractor o sin comando local conocido permanecen declaradas/instruction-only y no reciben un probe inventado.
- `axiom.config/toolchain.yaml` sigue siendo el manifest del subset que el proyecto habilitó. El catálogo no implica que todas sus entradas deban aparecer en este fichero ni en el lockfile.
- `.axiom-state/<projectKey>/toolchain.lock` es un YAML schema 1 local, generado e ignorado por Git junto con el resto de `.axiom-state/`. Su forma es `{ schemaVersion, projectId, lockedAt, tools }`; cada entrada de `tools` se indexa por el ID y contiene `id`, `version`, `channel` y, opcionalmente, `probeCommand`, `probeOutput` y `probedAt`. Si no existe, el loader devuelve un lock vacío para permitir una primera planificación.
- `axiom toolchain show` combina observación instalada, versión locked y canal en la misma tabla. `ToolEntry` admite además `version`, `channel`, `probeOutput` y `probedAt` como campos opcionales derivados del estado fijado/probado.
- `axiom toolchain plan` compara el lockfile contra la versión del canal solicitado y produce acciones `add`, `remove`, `upgrade`, `downgrade` o `none` sin escribir. En la CLI, el conjunto por defecto reúne tools declaradas y lockeadas; `--id` permite limitarlo. En la función pura, omitir el conjunto explícito solo revisa las tools ya lockeadas: una entrada adicional del catálogo no se convierte por sí sola en una instalación pendiente.
- `axiom toolchain upgrade --yes` escribe únicamente el lockfile. Antes guarda un checkpoint; si falla la escritura o el probe de verificación, restaura los bytes anteriores o elimina el lockfile recién creado. Sin `--yes`, o con `--dry-run`, solo imprime la vista previa. No descarga ni reemplaza binarios externos.

### `axiom member install` y `axiom bindings` (nuevos comandos)

Ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md) para la superficie CLI completa. Resumen de datos:

- `axiom member install --member <id> [--bind <repoId>:<path>]... [--no-register] [--home-dir <path>] [--json]` (`apps/cli/src/commands/member-install.ts`): reusa `runJoin` para el registro del miembro; reusa `@axiom/topology` (`loadTopology`/`loadLocalBindings`/`saveLocalBindings`/`resolveRepoPath`) para bindings — persiste sólo los `--bind` explícitos (los auto-detectados por `ref` no se persisten, ya cubiertos por el fallback existente de `resolveRepoPath`); repos sin resolver quedan listados con una nota apuntando a `axiom bindings set`. Materializa el config MCP nativo desde `axiom.config/mcp-manifest.yaml` (committeado) con `resolveMcpLaunchCommand()`. Idempotente end-to-end: cada paso individual ya lo es (join dedupe, `saveLocalBindings` sobreescribe con el mismo valor, `writeNativeMcpConfig` es merge-preserving, `repairTool` es no-op si ya `present`).
- `axiom bindings show|set --repo <id> --path <path>|remove --repo <id>` (`apps/cli/src/commands/bindings.ts`): CRUD acotado sobre `topology-bindings.yaml`, para arreglar UN SOLO repo sin re-correr todo el install.

## Pendientes conocidos de este modelo

- **D1 — multi-repo como modo primario/por defecto.** `defaultSingleRepoManifest` (`@axiom/topology/src/loader.ts`) sigue sin lógica de warning de deprecación en su fallback silencioso, y el check `TC-001` de `@axiom/doctor` sigue sin una rama `warn` correspondiente. Es la única mitad restante del par original D1/D3 (D3, la ruta opt-in `schemaVersion: 2`, está cerrada — ver arriba). D1 se secuenció deliberadamente después de D3 porque su warning solo tiene sentido una vez existe una ruta de opt-in genuina para los usuarios, lo cual ya es cierto. No programado a ningún incremento concreto todavía.
- **`@axiom/versioning`'s hueco de forma `projectContractVersion`/`assets.*`/`appliedMigrations`** — cuatro conceptos del documento fuente sin equivalente en `ManagedState` hoy (ver arriba). Ya no bloqueado por la falta de una ruta `schemaVersion: 2`, pero no programado todavía.
- **Reporte de cambios por repo en `axiom upgrade`** (addendum §8) — necesita tanto un `TopologyManifest` real (no-default) como un plan aprobado activo, más cableado de `axiom upgrade` consciente de topología. Genuinamente desbloqueado ahora que D3 está cerrado, pero ningún incremento ha construido el cableado todavía.
- **Registro histórico de preguntas de arquitectura Q1-Q5** — ver el cierre completo en `specs/increments/_archive/INC-20260702-axiom-redesign-roadmap/README.md`. Resumen: Q1 (single-repo deprecado vs. opt-in) no formalmente decidido pero superado en la práctica (single-repo sigue siendo el default); Q2 (formato de registro) resuelto pragmáticamente como se describe arriba; Q3 (supervivencia del prefijo `axiom.spec/` tras la separación de repos) quedó resuelto por ADR-0032: se conserva como baseline product-owned y no sustituye a `Axiom.Spec/`; Q4 (relación de capability/provider/telemetry/gateway con el modelo MCP nuevo) resuelto — ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md); Q5 (formato destino de `axiom.yml`) resuelto y entregado como se describe arriba.

## Superficies SDD por rol de repo + canal de inyección por proyecto (2026-07-15) — tanda INC-20260715-*

El generador de superficies de proceso (`workspace-process-surfaces.ts`, `surfaceIdsForRole`) materializa por rol de repo, tras esta tanda:

- **`sdd`** (repo de control/orquestación): `axiom-sdd-orchestrator`, `axiom-phase-reviewer`, `axiom-qa-validator` — orquestación + gates de revisión/QA.
- **`spec`** (repo de especificación): `axiom-spec-author`, `axiom-role-planner`, `axiom-spec-integrator`, `axiom-tech-context` — autoría, planificación, consolidación y contexto técnico.
- **`code`** (repos de código): `axiom-role-implementer` — parametrizado por `{role}`/`{repoPath}`.

## Modelo de repo único gestionado `<project>.axiom`, provenance y ejecuciones (2026-07-24) — tanda INC-20260724-*

Graduación consciente a *full product lifecycle* (excepción explícita a los "Explicit Bootstrap Limits", ver [00_Resumen_Ejecutivo.md](00_Resumen_Ejecutivo.md)): el modelo objetivo de los proyectos **instalados** deja de ser el par `sddRepo`+`specRepo` y pasa a un único repo gestionado `<project>.axiom` que concentra todo el "cerebro" (spec/increments/bugs/adr/technical-context/logs + skills/adapters/commands/rules). **No** se migran los repos propios de Axiom (`Axiom.SDD`/`Axiom.Spec`) a este modelo (dogfooding diferido al usuario).

### Topología `schemaVersion: 2` — `axiomRepo`/`codeRepos`/`legacyRepos` (INC-20260724-topology-single-axiom-repo)

**Supersede como MODELO OBJETIVO** el `TopologyManifest schemaVersion: 1` (`sddRepo`/`specRepo`/`roleCodeRepositories`) documentado arriba, de forma **aditiva y retrocompatible** (el schema v1 sigue cargando y validando). `@axiom/topology` (`packages/topology/src/types.ts`) añade:

- Discriminadores en `RepoRef`: `kind` (`'axiom' | 'code' | 'legacy'`) y `mode` (`'read-only-source'`).
- Campos nuevos en `TopologyManifest` (opcionales, aditivos): `axiomRepo?` (kind `axiom` — control + conocimiento en un solo repo), `codeRepos?` (repos de código; el rol se asocia por el mecanismo `assignments[]` existente, sin campo `role` inline), `legacyRepos?` (fuentes preexistentes del proyecto, `mode: 'read-only-source'`, Axiom nunca escribe en ellas).
- `schemaVersion: 1 | 2`. Un documento v1 (`sddRepo`/`specRepo`) **auto-mapea** a los campos gestionados (normalizador `normalizeTopologyManifest`, ambas formas siempre pobladas) y emite un warning **no bloqueante** `deprecated-legacy-shape`. El `axiomRepo` solo se deriva de un par legacy cuando es lossless (`sddRepo.ref === specRepo.ref`); un par genuinamente separado se deja `undefined` (no se fabrica un path fusionado).
- Nuevos `TopologyFinding`: `invalid-repo-kind` (error — el `kind` no cuadra con su bucket) y `deprecated-legacy-shape` (warning). `axiom topology show` superficie `schemaVersion`/`axiom-repo`/`legacy-repos` (ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md)).

### Adopción crea un `<project>.axiom` nuevo (INC-20260724-adopt-creates-axiom-repo)

La adopción (`axiom workspace setup --adopt-spec/--adopt-sdd`) deja de migrar **in-place** al repo legacy y pasa a **crear un repo hermano nuevo** `../${projectName}.axiom`, migrando el contenido de spec/context legacy DENTRO de él en formato Axiom. Implicaciones de datos/topología:

- Los repos legacy quedan **byte-for-byte intactos** (nunca se les escribe) y se registran como `legacyRepos[]` (`kind: 'legacy'`, `mode: 'read-only-source'`, `schemaVersion: 2`) en el `topology.yaml` del repo nuevo (ids convencionales `legacy-spec-source`/`legacy-sdd-source`, upsert por id, idempotente).
- El contenido migrado (increments/bugs/adr/technical-context) aterriza directamente en la RAÍZ del repo nuevo (`role: 'spec'`, **sin** anidamiento `axiom.spec/`), de modo que el writer de migración y los readers de CLI (`axiom-increment`/`axiom-bug list`) convergen en la misma raíz. `--control-path` sobreescribe el destino; `--dry-run` no escribe nada.

### Provenance/lifecycle en `metadata.yml` + manifest de migración (INC-20260724-provenance-lifecycle-manifest)

`metadata.yml` (`@axiom/workflow`, `BaseArtifactMetadataFields`, los 5 `ArtifactKind`: increment/bug/plan/adr/decision) gana 4 campos OPCIONALES y aditivos (retrocompatibles — un `metadata.yml` sin ellos carga, y los readers `resolveArtifactOrigin`/`resolveArtifactLifecycleState` defaultean a axiom-native):

- `origin.source`: `'migrated' | 'axiom-native'` (+ `migrationId`/`repository`/`originalPath`/`migratedAt` solo cuando `migrated`).
- `managedBy`: `{ tool, since, lastAxiomModificationAt? }`.
- `lifecycle.state`: `'migrated' | 'migrated-and-modified' | 'axiom-native'`.
- `exportPolicy`: `{ rollbackEligible, targetLegacyRepo?, targetLegacyPath? }`. `rollbackEligible` derivado: axiom-native → `true`; migrated (intacto) → `false`; migrated-and-modified → `true`.

Editar un artefacto migrado lo **auto-transiciona** `migrated → migrated-and-modified` (y sella `managedBy.lastAxiomModificationAt`) en el único choke point `saveArtifactMetadata`, sin comando aparte; un artefacto axiom-native o pre-existente sin origin nunca se marca migrado. Manifest global **no oculto** en `<project>.axiom/migration/migration-manifest.yaml` (runs de migración con `migrationId` + resumen fresco por artefacto), aditivo al `.migration-provenance.yml` oculto de idempotencia previo.

### Entidad `Execution` + `ExecutionStore` + paths execution-scoped (INC-20260724-worktree-isolation-execution)

Primera entidad de ejecución de primera clase para rastrear runs paralelos (típicamente en worktree). Deliberadamente **NO canónica** (no es un `ArtifactKind`, no se commitea):

- `@axiom/isolation`: entidad `Execution` (`id`, `projectId`, `artifactRef {kind: increment|bug|plan, id}`, `repoId`, `branch`, `worktreePath`, `state`, `agentTarget?`, `capabilities?`, `logsPath`, `evidencePath`, `createdAt`, `updatedAt`) con `ExecutionState` enum cerrado de 7 valores; id `EXE-<YYYYMMDD>-<HHMMSS>-<sufijo>` (clock inyectable). Paths execution-scoped `buildExecutionScopedPaths(executionId, rootPath)` → `.axiom-state/executions/<id>/{config,mcp,outputs,logs,evidence,local}`, disjuntos de los project-scoped y de otras ejecuciones (cubiertos por el `.gitignore` blanket de `.axiom-state/`).
- `@axiom/persistence`: `ExecutionStore` (create/get/list/update/close, escritura atómica tmp+rename, `list()` tolerante a carpetas huérfanas). `close()` es soft (state → `removed`, nunca borra ficheros). El store se ancla al repo FUENTE, así `logsPath`/`evidencePath` sobreviven al borrado del worktree (necesario para harvest). `UpdateExecutionPatch` gana `provisionedPaths?` (INC-20260724-worktree-close-correctness): el set exacto de ficheros que el provisioning escribió/reescribió, persistido para que el cierre los pueda neutralizar sin re-derivarlos.

### `executionMode` en `install-profile.json` (INC-20260724-worktree-mode-selection)

`ResolvedInstallProfile` (`@axiom/install-profiles`) gana `executionMode: 'in-place' | 'worktree'` (`DEFAULT_EXECUTION_MODE = 'in-place'`), persistido en `.axiom-state/<projectId>/install-profile.json` por `axiom configure --execution-mode`. Es el default elegido por el arquitecto en la instalación; se preserva a través de re-configuraciones no relacionadas (se relee el valor previo cuando el flag se omite) y es overridable por run (ver [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md) y [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md)).

**Canal de inyección por proyecto** (dónde va lo específico de cada stack, manteniendo el producto genérico): (i) `axiom.config/skills-index/<role>.yaml` — índice de skills por rol que leen las superficies (`sdd.skillIndexRead`); (ii) el contexto técnico del proyecto, propiedad de `axiom-tech-context`; (iii) las skills de rol del proyecto. Las superficies del producto se mantienen adapter/stack-agnósticas y parametrizables; a diferencia de un sistema role-specialized que hornea las reglas de stack en agentes por rol, Axiom las deja como DATO del proyecto para funcionar en cualquier adapter/stack sin perder profundidad. El manual runtime `Axiom/docs/usage/README.md` describe este canal.
## Artefactos y tipos de gobierno verificable (2026-08-02) — tanda `INC-20260730-*`

### `candidate-freeze.json` (por incremento)

Vive en `specs/increments/<id>/candidate-freeze.json`. Escrito por `axiom freeze --increment <id>`, leído por `checkCandidateFreeze`.

| Campo | Tipo | Significado |
|---|---|---|
| `incrementId` | `string` | Incremento congelado |
| `hash` | `string` (sha256) | Hash combinado sobre `memoryHash` + `specsHash`; es el valor que se compara para detectar mutación |
| `memoryHash` | `string` (sha256) | Sobre las entries de memoria con `entry.increment === incrementId`, ordenadas por `id` |
| `specsHash` | `string` (sha256) | Sobre el `README.md` del incremento |
| `timestamp` | ISO-8601 | Momento del congelado |

Nombre real del fichero: `candidate-freeze.json` (el Scope original del incremento decía `.frozen.json`; se conservó el nombre ya materializado en disco). Cobertura real del hash: memoria filtrada + `README.md`, **no** lockfiles ni `metadata.yml` — ver la limitación en RF-AXM-060.

Consecuencia operativa a tener presente: **reescribir el `README.md` de un incremento invalida su freeze por diseño** (cambia `specsHash`), y `checkCandidateFreeze` pasará a reportar `ok: false` hasta que se vuelva a congelar. Es el mecanismo funcionando, no un fallo.

### `receipts/<timestamp>-<phase>-<status>.json` (por incremento/bug)

Vive en `specs/<kind>s/<id>/receipts/`. Escrito por `writePhaseReceipt` (`@axiom/workflow`), automáticamente en cada transición y también por el comando manual `axiom phase receipt`.

| Campo | Tipo | Significado |
|---|---|---|
| `incrementId` | `string` | Artefacto al que pertenece la fase |
| `phase` | `PhaseName` | Nombre **real** de la transición (`increment-verify`, `bug-archive`…) |
| `status` | `'success' \| 'failure'` | Desenlace de la fase |
| `timestamp` | ISO-8601 | Fin de la fase |
| `details` | `string?` | Mensaje del resultado |
| `hash` | `string` (sha256) | Sobre `{incrementId, phase, status, timestamp, details}` |

El nombre de fichero sanea `:` (inválido en rutas Windows) y cualquier carácter no `[a-zA-Z0-9_-]` del nombre de fase.

### Evidencia requerida en `MemoryEntry`

`rationale` y `source` dejan de ser requeridos solo en tipos y pasan a ser **requeridos en runtime**: `string` con longitud `> 3` tras `trim()`. Nueva variante de error `MemoryError { kind: 'missing-evidence', field: 'rationale' | 'source', message }`. Constante exportada `MIN_EVIDENCE_LENGTH = 3` y guard puro `validateMemoryEvidence()`.

Nota de fondo relevante para cualquier futura garantía "de tipos": los `tsconfig.json` de cada package usan `"include": ["src/**/*"]`, de modo que **`tsc -b` no typechequea los ficheros de test** y vitest transpila sin typecheck. Un campo requerido en una interfaz no es, por sí solo, una garantía sobre lo que los tests o los callers JS realmente pasan — de ahí que el gate de evidencia sea un chequeo de runtime y no una declaración de tipo.

### `AXIOM_ERROR_CODES` (catálogo de `@axiom/core`)

Catálogo cerrado, documentado y **anclado a throw sites reales**; añadir un código va de la mano de migrar el/los sitio(s) correspondiente(s) en el mismo cambio. Entradas vigentes: `AXIOM_NO_PROJECT`, `AXIOM_GATE_FAILURE`, `AXIOM_MEMORY_SCOPE`, `AXIOM_MEMORY_QUERY`, `AXIOM_INIT_NOT_FOUND`, `AXIOM_INIT_INVALID_JSON`, `AXIOM_INSTALL_PROFILE_FAILED`, `AXIOM_INVALID_OPTION`, `AXIOM_ARTIFACT_ID_EXHAUSTED`, `AXIOM_BRANCH_TEMPLATE_VAR_MISSING`, `AXIOM_CHECKPOINT_NOT_FOUND`, `AXIOM_INVALID_CONFIG`.

`AXIOM_INVALID_OPTION` cubre toda validación de **input de CLI** (enum cerrado, flag requerido ausente, formato de `--role`, canal de toolchain); `AXIOM_INVALID_CONFIG` cubre **config inválida en disco**. Se mantienen distintos a propósito: permiten a un subagente distinguir "el operador escribió mal el comando" de "el fichero de configuración del proyecto está roto", que tienen recuperaciones diferentes.

## Modelo R-13 del launcher y telemetría (2026-09-08)

El envelope público del panel de telemetría es versionado (`schemaVersion: 2`) y contiene `projectMetrics` derivados del `projectRoot` resuelto y `processMetrics` etiquetados `process-wide`; `recentEvents` se proyecta newest-first desde la ventana tail. `readAuditTrailTail({ projectRoot, maxEvents, maxBytes })` es la API sancionada para leer el audit trail de forma acotada y reportar corrupción/I/O sin mezclar roots. Los grants y deliveries permanecen en estado runtime local, no en la spec.