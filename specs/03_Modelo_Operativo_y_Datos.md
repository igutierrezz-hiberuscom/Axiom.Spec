# 03 Modelo Operativo y Datos

> Esta sección documenta el modelo de datos REAL implementado hoy en `Axiom/`. El modelo de `axiom.yaml` único (`schemaVersion: 1`) sigue siendo el modo por defecto. El modelo de topología de repos por rol, el registro global `~/.axiom/projects.yml` y `axiom.yaml schemaVersion: 2` (ver "Topología de repos y registro global" más abajo) ya están implementados como ruta aditiva/opt-in, no como reemplazo del default.

## Modelo de datos real: `axiom.yaml` (manifiesto raíz por proyecto)

Cada proyecto que adopta Axiom tiene un único `axiom.yaml` en su raíz. Campos relevantes del schema actual (`Axiom/docs/configuration/project-structure.md`):

- `project.name`, `project.status`, `project.product_implementation_status`, `project.mode`;
- `scopes`;
- `rules`;
- `artifact_id_policy`;
- `lifecycle_commands`;
- `initial_capabilities`.

Se genera con `axiom init` y encapsula el **profile triple**: `functionalProfile` (`builder` | `product-owner`) + `operationalOverlay` (`local-only` | `standard` | `enterprise`) + `adapterTarget` (uno de 8 targets declarados). El triple recomendado para primer proyecto es `builder` + `local-only` + `opencode` (`Axiom/docs/first-project-readiness.md`).

## Estado project-scoped: `.axiom-state/`

> Renombrado desde `.sdd/` en `INC-20260703-config-folder-renames` (sin
> migración: no había proyectos reales todavía). El nombre anterior
> `.sdd/` no comunicaba con claridad que se trata del estado runtime de
> Axiom.

- `.axiom-state/local/`: overlay no versionada (overrides locales, markers como `last-sync.json`). Regida por `local-overlay-policy.yaml`.
- `.axiom-state/<projectName>/`: estado persistido del proyecto resuelto: `init.json`, `members.yaml`, `install-profile.json`, `last-start.json`.
  - `init.json` **no** lleva un campo `projectName` propio (`INC-20260703-config-dedup`, dedup #1): el nombre ya está codificado en el nombre del directorio que lo contiene, que a su vez viene de `resolveProject(axiom.yaml).name` — `axiom.yaml` es la única fuente de verdad para el nombre del proyecto. `init.json` sólo persiste `profileTriple` (la elección inicial del usuario en `axiom init`), `createdAt` y `version`.
- `.axiom-state/config/<projectName>/`: estado de mutación de subsistemas (namespace interno del persistence-store, distinto de `axiom.config/` — ver más abajo): `managed-state.json` (versionado), `model-assignments.json`, `components-state.json`, `gateway-state.json`.
- `.axiom-state/<projectName>/checkpoints/<id>/`: snapshots pre-mutación (upgrade, uninstall de components); se conservan los últimos 5.

## Catálogo de configuración declarativa: `axiom.config/*.yaml`

> Renombrado desde `axiom.spec/config/` en `INC-20260703-config-folder-renames`
> (sin migración): la carpeta se COLAPSA para vivir directamente en la
> raíz del proyecto como hermana de `axiom.spec/`, en vez de anidada
> dentro de él — son archivos de CONFIG generados, no contenido de
> spec, y el nombre anterior invitaba a confundirlos.

El runtime espera, dentro del proyecto adoptante, una carpeta `axiom.config/` en la raíz del proyecto (no confundir con `Axiom.Spec/`, el repo de este workspace, ni con el `axiom.spec/` del propio proyecto adoptante, que sigue existiendo para contenido de spec) con ~20 YAML de política y capacidad (`Axiom/docs/configuration/README.md`):

`axiom.workspace.yaml`, `branch-policy.yaml`, `capabilities.yaml`, `clarification-policy.yaml`, `command-protocol.yaml`, `external-work-items.yaml`, `id-policy.yaml`, `integrations.yaml`, `lifecycle-policy.yaml`, `local-overlay-policy.yaml`, `model-routing-policy.yaml`, `onboarding.yaml`, `orchestration-policy.yaml`, `policy-as-code.yaml`, `profiles.yaml`, `providers.yaml`, `repositories.yaml`, `scaffolding-contract.yaml`, `skills.yaml`, `telemetry-sinks.yaml`, `tool-routing-policy.yaml`.

No todos se consumen hoy con el mismo nivel de profundidad en runtime, pero forman el mapa documental y de control declarado. Ver discrepancia real: esta carpeta **no existe hoy** en la raíz del propio repo `Axiom/` (detalle en `context/references/03-riesgos-y-brechas-conocidas.md`).

### `profiles.yaml`: dato producto canónico con default bundleado (no scaffoldeado)

> `BUG-20260703-configure-needs-bundled-profiles`. A diferencia del resto del catálogo (que documenta política del proyecto adoptante), `profiles.yaml` (functional profiles, operational overlays, adapter targets, profile bindings, `roleAliases`) es dato **producto** — idéntico entre proyectos, no específico de cada uno. `axiom init` **no** lo scaffoldea (mantiene bajo el conteo de archivos generados). En su lugar, `@axiom/install-profiles` exporta `DEFAULT_PROFILES: ProfilesYaml`, un catálogo canónico bundleado que cubre ambos perfiles funcionales, los 3 overlays, los 8 adapter targets soportados por el CLI (en `allowedTargets` de ambos perfiles) y los aliases `analista → product-owner` / `arquitecto → builder`.
>
> - `installProfile` (`@axiom/installer`) usa `axiom.config/profiles.yaml` del proyecto cuando existe y es legible (override); si no, cae a `DEFAULT_PROFILES`. Nunca falla sólo porque el proyecto no tiene el archivo.
> - `axiom init --profile analista|arquitecto` resuelve el alias contra `DEFAULT_PROFILES` cuando no hay ningún `profiles.yaml` de proyecto o de producto disponible.
> - Un proyecto que necesite desviarse del catálogo por defecto puede crear su propio `axiom.config/profiles.yaml`; ese archivo, si existe y es legible, siempre gana sobre el default.
> - `@axiom/doctor` (GW-001 `collectGatewayDigests`, IP-001) trata la ausencia de `profiles.yaml` como caso normal (no error): el hash de gateway state incluye un marcador `"...|0|missing"` y los checks de install-profile se saltan (`skip`) en vez de fallar.

## Modelo de capabilities y providers

`capabilities.yaml` declara `capabilities.required` / `.optional` / `.postMvpOptional`, `supportLevels` y `degradationPolicy`. Cada capability tiene `id`, `domain` (`sdd`, `spec`, `code`, `memory`), `name`, `version`, `compliance`, `requiredTools`, `optionalTools`, `fallbacks`. `providers.yaml` declara el registry de providers y perfiles de discovery (`filesystem-first`, `gateway-first`, `local-only`) con `discoveryOrder`, `preferredProviders`, `optionalProviders`, `gatewayExpectation`. Implementado en `@axiom/capability-model` y consumido por `configure`, `start` y `doctor`.

## Ficheros generados por comando (ciclo de vida)

| Comando | Escribe |
|---|---|
| `init` | `axiom.yaml`, `.gitignore`, `.axiom-state/local/`, `.axiom-state/<projectName>/`, `init.json` — **ya no** escribe `axiom.config/topology.yaml` (`INC-20260703-config-dedup`, dedup #2; ver "Topología de repos..." más abajo) |
| `join` | `.axiom-state/<projectName>/members.yaml` |
| `configure` | `.axiom-state/<projectName>/install-profile.json` (+ surfaces del target) |
| `sync` | `.axiom-state/local/last-sync.json` (+ regeneración de outputs del adapter) |
| `start` | `.axiom-state/<projectName>/last-start.json` |
| `upgrade` | `.axiom-state/config/<projectName>/managed-state.json`, checkpoints |
| `model set/unset/reset` | `.axiom-state/config/<projectName>/model-assignments.json` (+ `.opencode/model-routing.json` si target es opencode) |
| `components install/uninstall` | `.axiom-state/config/<projectName>/components-state.json` |

## Ficheros generados por adapter target

| Target | Archivos |
|---|---|
| `opencode` | `.opencode/AGENTS.md`, `.opencode/skills-lock.yaml` |
| `claude-code` | `.claude/AGENTS.md` |
| `github-copilot` / `copilot-vscode` | `.github/copilot-instructions.md` (+ `.vscode/settings.json`, `.vscode/extensions.json` en `copilot-vscode`) |
| `vscode` | `.vscode/settings.json` |
| `cursor` | `.cursor/settings.json`, `.cursor/AGENTS.md` |
| `litellm` | `litellm.config.json` |
| `antigravity` | `.antigravity/AGENTS.md` |
| `visual-studio-2026` | `.vs/AXIOM.md` |

## Regla sobre YAML globales

Un YAML global del producto no debe mezclar visión documental, builder tooling y runtime. Cada YAML debe pertenecer a una capa concreta y a una responsabilidad concreta. Vigente: el catálogo de `axiom.config/*.yaml` ya está desglosado por responsabilidad concreta (policy, capability, telemetry, routing) en vez de un único fichero monolítico.

## Topología de repos y registro global (roadmap de rediseño, cerrado, aditivo sobre el modelo por defecto)

`specs/increments/INC-20260702-axiom-redesign-roadmap/` (23 incrementos, cerrado 2026-07-03) implementó un modelo de topología de repos por rol dentro de cada proyecto gestionado por Axiom, y un registro global fuera del proyecto. Este modelo convive de forma **aditiva** con el `axiom.yaml` único descrito arriba — `single-repo` sigue siendo el modo por defecto en la práctica hoy; el modelo multi-repo es un opt-in real y ya materializado, no solo declarado.

### `TopologyManifest` (`Axiom/packages/topology/src/types.ts`, `schemaVersion: 1`)

```ts
interface TopologyManifest {
  schemaVersion: 1;
  mode: 'single-repo' | 'multi-repo';
  sddRepo: RepoRef;
  specRepo: RepoRef;
  roleCodeRepositories: readonly RepoRef[];
  assignments: readonly RoleAssignment[];
  roles?: readonly RoleDef[]; // Decision D5 — team/code role registry, ver abajo
  qaLane?: 'inline' | 'parallel';
}
```

Tres repos por rol dentro de un proyecto gestionado:

1. repo `sdd` (método/factory);
2. repo `spec` (conocimiento canónico);
3. repo(s) `code` (runtime instalable, `roleCodeRepositories`).

- `single-repo` sigue siendo el modo por defecto en la práctica: `sddRepo` y `specRepo` resuelven ambos a la raíz del proyecto, `roleCodeRepositories` queda vacío. El paso de "multi-repo como modo primario/por defecto" (D1, ver "Pendientes conocidos" abajo) no se ha dado, aunque la ruta opt-in a multi-repo (`schemaVersion: 2`, ver más abajo) ya es real y está entregada.
- **`axiom.yaml` es la fuente de verdad autoral del mapa de repos; `topology.yaml` es un artefacto opt-in/derivado** (`INC-20260703-config-dedup`, dedup #2, cerrado 2026-07-03). `axiom init` YA NO escribe `axiom.config/topology.yaml` para el layout `installed-multi-repo` (única ruta que antes lo hacía desde cero). `loadTopology` (`@axiom/topology`) deriva un `TopologyManifest` de fallback directamente desde `axiom.yaml` cuando `topology.yaml` está ausente (`tryLoadTopologyHint`, version-aware v1/v2, → `defaultInstalledMultiRepoManifest`/`defaultSingleRepoManifest`); todo consumidor de topología (doctor `TC-001`/`TC-003`/checks de límite de dogfooding, `axiom topology show`, `qa-archive-gate`) pasa por `loadTopology`, nunca lee el YAML crudo, así que se beneficia del fallback de forma transparente. `topology.yaml` se sigue materializando de forma perezosa recién cuando un proyecto corre una mutación real (`axiom roles assign`/`remove`, que llama a `loadTopology` y persiste con `writeTopologyYaml` en la primera asignación). Unificación completa de schema (que `defaultInstalledMultiRepoManifest` derive `specRepo.ref` leyendo literalmente `axiom.yaml#paths.specification.path` en vez de re-derivar la convención `../${projectName}.spec` de forma paralela) queda deliberadamente diferida: tocaría la firma pública de los builders de manifest default de `@axiom/topology` y sus tests de forma exacta, sin que haya todavía un proyecto real que fuerce la necesidad. `roleCodeRepositories`/`assignments`/`qaLane` no tienen equivalente en `axiom.yaml#paths` y seguirán siendo exclusivos de `TopologyManifest`.
- `multi-repo` está completamente soportado por el schema (cada repo tiene un `id` y un `ref`, path relativo a `projectRoot` o path/URI absoluto). La resolución real de `LocalBindings` para multi-repo más allá de la heurística "ref relativo a projectRoot" sigue siendo una preocupación P1 — no existe todavía ningún proyecto con contenido no trivial de `topology-bindings.yaml` contra el que validar.
- Los bindings locales por usuario (`.axiom-state/local/topology-bindings.yaml`, `LocalBindings { schemaVersion: 1; localPaths: Record<string, string> }`) explícitamente no se versionan.
- Todos los helpers conscientes de topología del código (write-scope, límite de dogfooding, etc.) resuelven repos vía `loadTopology`/`loadLocalBindings`/`resolveRepoPath` de `@axiom/topology`, todos libres de `homeDir` — patrón establecido y reusable para cualquier check futuro que necesite ser parametrizado por rol en vez de hardcodeado a nombres de repo concretos.

### Materialización de `topology.yaml` en cada repo, colapso de `repoId` y canonicalización de paths Windows (2026-07-11)

- **`topology.yaml` materializado en CADA repo** (control + spec + cada repo de rol), anclado per-repo (`INC-20260711-repo-affinity-guard`): supersede la nota de arriba de que solo el repo de control recibía `topology.yaml`. El mapa role→repo se escribe en todos los repos (por `runWorkspaceSetup`, `runRepoAdd` y best-effort por `member install`) para que `loadTopology(repoActual)` resuelva la identidad de rol desde cualquier repo — es la base del guard de repo-affinity (ver [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md)). `ProjectResolution` gana `role`/`repoId` aditivos (poblados desde `axiom.yaml` schemaVersion 2).
- **Colapso del doble segmento de `repoId`** (`INC-20260711-audit-bug-fixes`): `buildRoleAwareAxiomYaml` colapsa `repoId = ${projectId}-${repoRole}-${roleKey}` a un solo segmento cuando `repoRole === roleKey` (ya no `<project>-sdd-sdd`/`-spec-spec`). Cosmético: ningún lookup usa `repoId` (usan `roleKey`/`topologyId`).
- **Canonicalización de paths Windows 8.3 (compare-time)** (`INC-20260711-audit-bug-fixes`): nuevo `canonicalizePath` (`@axiom/filesystem-truth`) resuelve la forma corta 8.3 (`IGUTIE~1`) vs. larga (`igutierrezz`) a una forma consistente, aplicado en los sitios de COMPARACIÓN del registro/resolver (`findByRootPath`/`findByRepoPathV2`/`findByAncestorRepoPathV2`, `normalizeForAncestorCompare`, `relativeRef`). NO reescribe los paths ALMACENADOS (Decision D-001) — solo canonicaliza ambos operandos al comparar, para no romper asserts de path exacto.

### Dos ejes de "rol", desacoplados (Decision D5, `INC-20260710-dynamic-team-roles`)

Axiom tiene DOS conceptos distintos de "rol" que no deben conflarse:

1. **Install profiles** (`axiom.config/profiles.yaml#functionalProfiles`): `product-owner`/`builder` + aliases (`analista`/`arquitecto`, `roleAliases`). Eje de **CAPACIDAD** — parte de la tripleta de instalación (functionalProfile + overlay + adapterTarget). Gestionado por `axiom roles add` (catálogo fijo, pensado para crecer poco).
2. **Team/code roles** (`topology.yaml#roles`, `RoleDef { id; description? }`, campo OPCIONAL y aditivo del `TopologyManifest`): backend, frontend, mobile, qa, devops, o cualquier otro nombre — los roles que DUEÑAN repos de código. Eje de **EQUIPO/CÓDIGO**, deliberadamente sin un catálogo fijo: el arquitecto registra 1..N roles, uno por uno, vía `axiom roles register <id> [--description] [--repo]` (idempotente); `axiom roles unregister <id>` los quita (bloquea si el rol todavía tiene asignaciones activas — no cascadea el borrado). `axiom roles list` muestra ambos ejes por separado (`--json`: `{ installProfiles, teamRoles }`).

`validateTopology` (`@axiom/topology`) sigue tomando un `ReadonlySet<string>` de IDs válidos — no lee `manifest.roles` por sí mismo. Cada caller construye ese set como la UNIÓN de: `manifest.roles[].id` (eje 2) ∪ `profiles.yaml#functionalProfiles[].id` (eje 1) ∪ `functionalProfiles[].activatesImplementationRoles`. Callers actuales que ya construyen esta unión: `axiom topology validate` (`apps/cli/src/commands/topology.ts`; además cae a `DEFAULT_PROFILES` de `@axiom/install-profiles` cuando `profiles.yaml` está ausente, para no producir un set vacío sólo por eso), el check `TC-001` de `@axiom/doctor`, y la screen `topology` de `@axiom/tui`. `axiom roles assign`/`unassign` resuelven `--role` contra CUALQUIERA de los dos ejes (team role registrado primero; si no matchea, cae a `resolveRoleId` sobre `profiles.yaml`, que sí resuelve aliases — los team roles NO tienen aliasing).

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

### Registro global v2 (`~/.axiom/projects.yml`)

`Axiom/packages/user-workspace/src/registry.ts`: `schemaVersion: 2`, YAML, en `~/.axiom/projects.yml`, **aditivo** junto al legado `~/.axiom/registry.json` (`schemaVersion: 1`, JSON) en vez de un rename/reemplazo destructivo — ambos formatos son leídos/escritos por el mismo paquete, seleccionados según qué fichero está presente. La forma de `projects.yml` mapea un proyecto a sus `repos` (mapa por rol), consistente con el modelo de topología de arriba. `getProjectV2`/`addProjectV2`, etc. (la API `*V2`) es la superficie actual y de cara al futuro; el lector/escritor legado de `registry.json` se mantiene por compatibilidad con proyectos ya instalados.

### `axiom.yaml` — `schemaVersion: 2` (cutover cerrado)

`AxiomYamlSchemaV2` (`projectId`/`name`/`repoId`/`role`/`mode`/`paths`), emitido por `buildAxiomYaml` en `apps/cli/src/commands/init.ts` para ambos layouts de scaffold (`self-hosted` e `installed-multi-repo`). `resolveProject` (`@axiom/project-resolution`) y los checks `MC-001`/`BC-001`/`BC-002` de `@axiom/doctor` son conscientes de versión: aceptan tanto `schemaVersion: 1` (legado — `project.mode` + `scopes: { name: { path, product_runtime } }`) como `schemaVersion: 2`, nunca resolviendo mal uno como el otro. Este cutover quedó cerrado tras verificación end-to-end dos veces por dos roles distintos (incluyendo una reproducción de test negativo y una re-ejecución independiente de `axiom doctor` contra un proyecto v2 separado). Durante el cierre se encontraron y corrigieron dos consumidores v1-only no detectados antes: `configure.ts`'s `readAxiomYamlProjectName` (habría fallado duro para proyectos v2 sin `product.manifest.yaml`) y `@axiom/topology/src/loader.ts`'s `tryLoadTopologyHint` (habría resuelto mal, en silencio, la topología de un proyecto v2 `installed-multi-repo` al default single-repo).

`role` (`REPO_ROLES`: `sdd|spec|code`), `layout` (`PROJECT_LAYOUTS`), y el profile triple (`FUNCTIONAL_PROFILES`/`OPERATIONAL_OVERLAYS`/`ADAPTER_TARGETS`) son const arrays exportados por `apps/cli/src/commands/init.ts`, fuente única para la validación de `runInit` y para el wizard guiado de la TUI que pregunta cada campo con un default pre-seleccionado antes de escribir — ver "TUI — wizard guiado de `init`" en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md).

### Setup de workspace multi-repo en una operación (`runWorkspaceSetup`) — INC-20260705-workspace-multirepo-setup-engine

`runWorkspaceSetup(spec: WorkspaceSetupSpec): Promise<WorkspaceSetupResult>` (`apps/cli/src/commands/workspace-setup.ts`) es el motor que scaffoldea y cablea un workspace Axiom multi-repo **en una sola llamada**: un repo de control (rol `sdd`), un repo de spec (rol `spec`) y N repos de código por rol funcional (`backend`/`frontend`/`qa-e2e`/custom, strings abiertos). Es una ruta aditiva y distinta de `runInit`/`axiom init` (single-repo), que quedan sin cambios. Consumido hoy por el wizard guiado de la TUI (ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md)); no expone (todavía) flag `--workspace` ni comando `axiom` propio.

Datos que escribe el motor, con el modelo de datos ya descrito arriba como base:

- **`axiom.yaml` por repo, con `paths` recíproco y consciente de rol** (`schemaVersion: 2`, builder `buildRoleAwareAxiomYaml`, separado del `buildAxiomYaml` single-repo de `init.ts`). Cada repo recibe su propio `axiom.yaml` cuyo `paths` referencia a **todos** los repos hermanos con paths relativos calculados vía `path.relative` desde ese repo (así "se conocen entre sí"). El caso degenerado de dos repos que resuelven al mismo path absoluto emite `.` en vez de un relativo autorreferencial.
- **Un único `axiom.config/topology.yaml`**, escrito solo en el repo de control, con `sddRepo`/`specRepo`/`roleCodeRepositories`/`assignments` derivados del `spec`, `qaLane: 'inline'`, y `mode: multi-repo` cuando hay repos de rol o un repo de spec distinto (si no, `single-repo`). Round-trippea por `loadTopology` (`@axiom/topology`) — es el mismo `TopologyManifest` opt-in descrito arriba, ahora escrito por el motor en vez de perezosamente por `axiom roles assign`.
- **`.axiom-state/local/topology-bindings.yaml`** en el repo de control (`saveLocalBindings`), mapeando cada `topologyId` a su path absoluto local (no versionado, ver arriba).
- **Registro de TODOS los repos en `~/.axiom/projects.yml`** (registro v2) bajo un único `projectId` (`= slugifyProjectId(projectName)`) en **una sola llamada idempotente** vía el helper nuevo `upsertProjectReposV2` (`@axiom/user-workspace`, `registry.ts`): carga → merge del mapa de repos (las claves de repo nuevas ganan en conflicto, porque un re-run significa "este workspace ahora conoce este path") → guarda, creando el proyecto si no existía. Re-ejecutar el mismo setup es seguro (no lanza `duplicate-id`). Reusable por futuros llamadores (`repo attach`, etc.). El registro es **best-effort y no bloqueante** — ver "Registro no bloqueante y auto-migración de registry v1→v2" más abajo, que supersede su alcance.
- Solo el repo de control recibe `topology.yaml` + `.axiom-state/<projectId>/`; los repos de rol/spec reciben `axiom.yaml` (+ `.gitignore`/`AGENTS.md` best-effort en directorios nuevos) y `.axiom-state/local/` + `.axiom-state/<projectId>/`.

Directorios nuevos (`create: true`) se scaffoldean desde cero; directorios existentes (`create: false`) se parametrizan in-place. **Guarda no-clobber**: si un `axiom.yaml` preexistente parsea con un `projectId` distinto (o es v1/no-parseable-pero-presente), su escritura se salta y se registra como warning, nunca como error (ver ángulo de ownership en [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md)).

Tras el registro, el motor invoca best-effort la generación de config MCP (`.axiom/mcp.yml` + proyección al adapter) — ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md) y la sección `mcp.yml` de abajo.

### Registro no bloqueante y auto-migración de registry v1→v2 (`INC-20260705-workspace-setup-registry-robustness`)

El registro en el registro v2 es ahora **no bloqueante para TODO el scaffolding local**, no solo para MCP. Cualquier fallo de registro (registry legado v1 sin migrar, fallo de `ensureHomeDir`, o cualquier error de upsert) o un `spec.register === false` registra `registryRegistered: false` + un warning claro, pero **ya no hace early-return**: todo el scaffolding local (config MCP `.axiom/mcp.yml`, adapters, `workspace.json`, baseline de skills SDD, base de spec, skills por repo de código) corre siempre a continuación, best-effort — ninguno de esos pasos necesita realmente el registro. La generación MCP degrada a un warning `unknown-project` cuando el id no resuelve en el registro, pero `mcp.yml` se escribe igualmente (validate-after-write). Esto supersede la lectura previa de "tras un registro exitoso": el scaffolding local no está condicionado al éxito del registro; `registryRegistered` refleja la realidad y solo determina el valor de retorno final.

Auto-migración v1→v2 (`migrateLegacyRegistryV1ToV2(homeDir)`, `@axiom/user-workspace`, nuevo, barrel-exportado): cuando el upsert falla con causa `legacy-registry-not-migrated` (existe `~/.axiom/registry.json` v1 y falta `~/.axiom/projects.yml` v2), el motor invoca la migración **una sola vez** y reintenta el upsert **una sola vez**. La migración lee `registry.json` v1, mapea cada `ProjectEntry` v1 (`{id, name, rootPath, ...}`) a un `ProjectEntryV2` con un mapa `repos` de una entrada (`{ sdd: { role: 'sdd', path: rootPath } }` — v1 no tenía concepto de rol, así que su único repo se registra bajo la clave `'sdd'`), preserva `id`/`name`/`addedAt`/`lastUsedAt`, escribe `projects.yml` vía `saveRegistryV2`, y **preserva el fichero legado renombrándolo a `registry.json.migrated`** (nunca borra datos de usuario). Es idempotente (no-op si `projects.yml` ya existe). Si la migración o el reintento aún fallan, la ejecución cae a la misma ruta warn-and-continue no bloqueante (sin throw). Esto resuelve un caso real de instalación medio-completada donde un registry v1 estancado bloqueaba silenciosamente todo lo que venía después.

Alcance (`INC-20260710-lifecycle-correctness-fixes`, cierra la inconsistencia original): `axiom init` (`runInit`) y `axiom repo attach` (`runRepoAttach`) reutilizan el MISMO `migrateLegacyRegistryV1ToV2` con el mismo patrón migrar-una-vez-y-reintentar-una-vez, en vez de rendirse con un mensaje que le pedía al operador ejecutar `axiom upgrade` (comando que no existe/no soporta esta migración). Las tres rutas (`runWorkspaceSetup`, `runInit`, `runRepoAttach`) ahora auto-migran de forma consistente ante un registry v1 sin migrar; sólo fallan de forma visible (WARN o exit 1, según la ruta) si la migración misma falla.

### Artefactos adicionales del setup de workspace (round 2, INC-20260705-*)

La segunda tanda de incrementos de workspace añade cuatro clases de artefacto persistido/scaffoldeado al motor. Todos son pasos **best-effort** (un fallo nunca aborta el setup) y comparten las semánticas de gating descritas al final de esta subsección. El comportamiento y las tablas de despacho viven en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md); aquí se documentan solo las formas de datos.

- **`<controlRepo>/.axiom-state/<projectId>/workspace.json`** — registro de la selección de adapters (`INC-20260705-workspace-adapters-multiselect`), escrito una sola vez en el repo de control (el mismo ancla project-scoped que `topology.yaml`/`topology-bindings.yaml`), incluido en `filesCreated`:

  ```ts
  interface WorkspaceSetupRecord {
    schemaVersion: 1;
    adapters: AdapterTarget[]; // los adapters multi-seleccionados
    providers: string[];       // providers LOCALES habilitados (INC-20260708-wizard-configure-provider-selection)
    profile: string;           // functional profile resuelto
    overlay: string;           // operational overlay resuelto
    createdAt: string;         // ISO timestamp
  }
  ```

  El campo `providers` (`INC-20260708-wizard-configure-provider-selection`) persiste la SELECCIÓN de providers LOCALES habilitados del proyecto (subconjunto de `codegraph`/`serena`/`graphify`/`engram`; `[]` = ninguno, solo `filesystem` always-on). Es la ÚNICA fuente de verdad de "qué providers habilitó este proyecto" — distinta del MODELO cerrado de 7 ids de `axiom.config/providers.yaml` (schema-locked, nunca recortado). Lo escriben tanto el step `providers` del wizard como `axiom configure --providers <csv>` (merge-write) y las operaciones incrementales `provider add`. Lo lee `buildProjectProviderRegistry` (`@axiom/providers`) para registrar exactamente los clientes code-intel habilitados. Ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

- **`install-profile.json` por repo** — `generateWorkspaceAdapters` resuelve un `ResolvedInstallProfile` por repo del workspace (vía `installProfile`, un solo call por repo con `adapters[0]` como primario), persistido en el `.axiom-state/<projectId>/` de ese repo, con el mismo shape que ya escribe `axiom configure` (ver "Ficheros generados por comando"). Los ficheros de adapter derivados (`.opencode/AGENTS.md`, `.claude/AGENTS.md`, `.antigravity/AGENTS.md`, `.vs/AXIOM.md`, etc.) se escriben en **cada** repo por adapter seleccionado, según la tabla de despacho `target -> generador` de [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

- **Baseline de skills en el repo de control** (`INC-20260705-workspace-sdd-skills`), solo cuando el repo de control se crea recién:
  - `axiom.config/skills-catalog.yaml` (`schemaVersion: 1`) — catálogo semilla de 5 ids con `bundleHash` byte-exacto por entrada (`computeSkillBundleHash`); las fuentes de cada skill se escriben bajo `axiom.spec/target-axiom-skills/<id>.md`. Es el mismo `SkillsCatalog` que consume el check `TC-010` de doctor (ver [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md)).
  - `.opencode/agents/<id>/SKILL.md` (materializado por `applySkillSet`) + `.axiom-state/<projectId>/skills-pending.json`.
  - `axiom.config/skills-index/<role>.yaml` — un `SkillsRoleIndex` (schema de RF-AXM-020, ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md)) por cada rol funcional declarado en el workspace.

- **Base de spec en el repo de spec** (`INC-20260705-workspace-spec-base`), solo cuando el repo de spec se crea recién: la estructura canónica de spec + contexto técnico, scaffoldeada desde plantillas bundleadas como constantes TS (guardado per-file: nunca sobrescribe un fichero preexistente, skip + warning):
  - `specs/README.md` + `specs/00_Resumen_Ejecutivo.md` .. `specs/08_Glosario.md` (9 ficheros numerados);
  - `context/TECHNICAL_CONTEXT.md` + `context/README.md`;
  - directorios estructurales vacíos (vía `.gitkeep`): `specs/{increments,bugs,archive}/` y `context/{architecture,integrations,operations,references}/`.

**Semánticas de gating (`created`)**: la generación multi-adapter + `workspace.json` corre **siempre**, para todos los repos (con independencia del resultado de registro — ver "Registro no bloqueante" arriba). La baseline de skills del repo de control corre **solo si el repo de control se creó recién** en esta misma llamada (`repoResults.find(r => r.topologyId === control.topologyId)?.created === true`); la base de spec corre **solo si el repo de spec se creó recién** (`… === specRepo.topologyId …`). Un repo parametrizado in-place (`create: false`, ya existente) se salta silenciosamente en ambos casos — sin clobber, sin ruido. El flag `created` lo computa `writeOneRepo` antes en la misma llamada.

### Autoskills por repo de código (round 3, `INC-20260705-workspace-code-repo-skills`)

Además de la baseline de skills del repo de control (round 2, arriba), cada repo de CÓDIGO/rol **recién creado** (`kind === 'role'` con su propio `created === true`) recibe ahora su propia baseline de skills scoped a su rol, escrita dentro de ESE repo de código (no solo en el de control). Reusa la misma semilla bundleada y la misma maquinaria de `@axiom/skills` que el repo de control (sin duplicación); las formas de datos por repo de código creado son:

- `axiom.config/skills-catalog.yaml` — el mismo `SkillsCatalog` (`schemaVersion: 1`, semilla de 5 ids) que el repo de control, más las fuentes bundleadas bajo `axiom.spec/target-axiom-skills/<id>.md`.
- `.opencode/agents/<id>/SKILL.md` (materializados por `applySkillSet`) + `.axiom-state/<projectId>/skills-pending.json` (con `projectId = effectiveProjectId`, consistente entre todos los repos del workspace).
- **UN único `axiom.config/skills-index/<roleId>.yaml`** scoped al propio rol (posiblemente custom) de ese repo (`role: roleId`, `repoKinds: ['role']`), donde `roleId = repo.functionalRoleId ?? repo.roleKey` — a diferencia del repo de control, que escribe un `skills-index` por CADA rol funcional del workspace.

Es best-effort y gateado estrictamente por el `created` propio de cada repo de rol: un repo de rol preexistente (`create: false`) se salta por completo (nunca clobbera su catálogo). El repo de control sigue vía `scaffoldSddSkills` (sin cambios); el repo de spec no recibe skills. El comportamiento vive en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

### Artefactos y formas de datos de la tanda INC-20260708-* (providers, memoria, reglas, operaciones incrementales)

Formas de datos añadidas por esta tanda (el comportamiento vive en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md); aquí solo las estructuras persistidas y su ancla):

- **Selección de providers en `workspace.json#providers`** (`INC-20260708-wizard-configure-provider-selection`): ver el campo `providers` del `WorkspaceSetupRecord` arriba. Estado project-scoped en `<controlRepo>/.axiom-state/<projectId>/workspace.json`, leído por `buildProjectProviderRegistry` (`@axiom/providers/project-registry.ts`) que escanea `.axiom-state/*/workspace.json` bajo el `projectRoot` (mismo patrón que `readEnabledProviders`) y devuelve un `ProviderRegistry` con los clientes code-intel habilitados + `filesystem` always-on, más un `engramEnabled: boolean` (engram no es un `ProviderClient` — se resuelve vía `resolveMemoryBackend`).

- **Adiciones al modelo de memoria** (`INC-20260708-memory-real-local-backend`): `MemoryEntry` gana `topicKey?`/`sessionId?` (aditivos); nuevo tipo `MemorySessionSummary`; `MemoryBackend` gana `saveSessionSummary?` opcional. El UPSERT topic-keyed (misma `(projectId, topicKey)` reemplaza en sitio) aplica a AMBOS backends: `createInMemoryBackend` (JSON, `.axiom-state/local/memory/<projectId>.json`, find-and-replace) y el nuevo `createEngramBackend` (nativo vía `topic_key` de engram). `resolveMemoryBackend` selecciona entre ambos (probe de `engram mcp`, fallback JSON, nunca lanza). Las **lecciones** (`INC-20260708-continuous-learning`) NO añaden un `MemoryKind` nuevo: se almacenan como `MemoryEntry` `kind: 'pattern'` tag `'lesson'`, con `topicKey` derivado (`learning/<capabilityId>` para lecciones de audit, `learning/manual/<slug>` para texto explícito). Métricas de delegación opcionales en `.axiom-state/<project>/session-metrics.json` (shape `DelegationMetrics`, consumidor-only — Axiom no lo produce; ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md)).

- **Artefactos de la capa de reglas** (`INC-20260708-rules-layer`): `axiom.config/rules/<scope>.md` — `common.md` (siempre) + `<language>.md` por lenguaje inferido (`typescript`/`python`/`csharp`/`angular`). Ubicación canónica análoga a `axiom.config/skills-*`, escrita por `scaffoldRules` best-effort no-clobber por fichero. El `AGENTS.md` canónico gana un campo `CanonicalAgentsMdIdentity.ruleScopes?` (poblado por el caller leyendo disco en tiempo de render) que lista los scopes presentes. Proyección nativa opcional: `.cursor/rules/axiom-common.mdc` (solo `common`, no-clobber).

- **Scaffold canónico del propio repo `Axiom/`** (`INC-20260708-product-repo-self-bootstrap`): el repo de producto ganó en su raíz el set canónico que su runtime/tests esperaban — `axiom.config/` con contenido schema-válido real (`skills-catalog.yaml`, `agents-catalog.yaml`, `model-routing-policy.yaml`, `profiles.yaml`, `providers.yaml`, `capabilities.yaml`, `integrations.yaml`, `policy-as-code.yaml`, `mcp-manifest.yaml`, `telemetry-sinks.yaml`), `axiom.spec/target-axiom-skills/*.md` (7), `axiom.spec/target-axiom-agents/*.md` (3, ampliados a 7 por el roster de delegación de `INC-20260708-delegation-triggers`), `axiom.spec/templates/` (copiadas de `Axiom.Spec/templates/`), `AGENTS.md` y `axiom.skills.lock`. Con esto `readiness:first-project` y `doctor` pasan contra la raíz del propio repo (ver [00_Resumen_Ejecutivo.md](00_Resumen_Ejecutivo.md) y [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md)). `profiles.yaml#allowedTargets` se recortó a los 5 `MVP_TARGETS` para satisfacer el check estricto `IP-003`, mientras `adapterTargets` sigue declarando los 8 (5 `mvp:true` + 3 `mvp:false`).

- **Idempotencia de las operaciones incrementales** (`INC-20260708-incremental-operations`): `axiom repo/adapter/provider/role add` mutan los MISMOS artefactos del modelo multi-repo reusando los helpers exportados de `workspace-setup.ts` (`buildRoleAwareAxiomYaml`, `writeOneRepo`, `buildTopologyManifest`/`writeTopologyManifest`, `relativeRef`, `axiomYamlPathFor`, `tryReadExistingProjectId` — ampliados de module-private a exportados, sin cambio de comportamiento en `runWorkspaceSetup`). `repo add` re-deriva el bloque `paths` de CADA repo del proyecto (recíproco), actualiza `topology.yaml`, hace `upsertProjectReposV2` y genera adapters/MCP/skills/rules solo para el repo nuevo; `adapter add` hace append-dedup en `workspace.json#adapters` y regenera ese adapter en todos los repos; `provider add` hace append-dedup en `workspace.json#providers`. Re-ejecutar con los mismos args es no-op/merge (sin entradas duplicadas, sin clobber). Si no existe `workspace.json`, `adapter add`/`provider add` crean uno mínimo (schemaVersion 1, arrays vacíos) en vez de fallar.

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

El **call site generador vivo** hoy es el setup de workspace multi-repo (`runWorkspaceSetup`, INC-20260705-workspace-mcp-generation), que escribe `.axiom/mcp.yml` en el repo de control como fuente canónica — ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md) para el detalle, los dos servers (`sdd-mcp-server`/`spec-mcp-broker`) y el caveat de que `.axiom/` está gitignoreado. **`INC-20260708-mcp-native-config-mapping` cambió qué config de herramienta produce esta ruta**: en vez del custom-shape `.opencode/mcp.json`/`.claude/mcp.json`, `runWorkspaceSetup` emite ahora el **schema MCP NATIVO real** de cada herramienta seleccionada — `opencode.json` (`{ $schema, mcp:{ <id>:{ type:'local', command:[cmd,...args], enabled, environment? } } }`), `.mcp.json` y `.cursor/mcp.json` (`{ mcpServers:{ <id>:{ command, args, env? } } }`), `.vscode/mcp.json` (`{ servers:{ <id>:{ type:'stdio', command, args, env? } } }`) — por cada repo del workspace × cada adapter seleccionado, merge-preserving y atómico (`native-mcp-config.ts`); antigravity/visual-studio-2026/litellm degradan a warning sin fichero. El custom-shape queda superseded para la ruta de workspace; `.axiom/mcp.yml` permanece canónico y sin cambios de forma. Fuera de esa ruta el estado previo se mantiene: `runConfigure`/`sync` **no** llaman a los generadores, y `.opencode/mcp.json`/`.claude/mcp.json` siguen intencionalmente excluidos de `GENERATED_FILES_BY_TARGET` para la ruta `configure`/`sync` (deferral TR-005 intacto).

### Versionado, upgrades y migración de schema (`@axiom/versioning`)

`ManagedState` (`Axiom/packages/versioning/src/managed-state.ts`, `schemaVersion: 1`, JSON en `<root>/.axiom-state/config/<projectName>/managed-state.json`): `{schemaVersion: 1, runtime: {package, version}, adapterTargets: [{id, version, lastSyncedAt}], lastUpgrade: {fromVersion, toVersion, at, checkpointId} | null, lastCheckpointId: string | null}`. Esta forma es estructuralmente distinta del `version.yml` propuesto originalmente por el documento fuente (`axiom: {projectContractVersion, installedWithAxiomVersion, lastUpgradedWithAxiomVersion}`, `assets: {sddSkillsVersion, adaptersVersion, guidesVersion, templatesVersion}`, `appliedMigrations: [...]`) — no es una variante de nombrado. Cuatro conceptos del documento fuente no tienen equivalente (`projectContractVersion`, `installedWithAxiomVersion`, los cuatro campos `assets.*`, y `appliedMigrations` como lista acumulativa — solo se conserva el `lastUpgrade` más reciente). Este hueco queda abierto (ver "Pendientes conocidos" abajo) pero ya no está bloqueado por la falta de una ruta `schemaVersion: 2`.

Mecanismo de migración/rollback confirmado real por lectura directa y ejecución completa de tests: exactamente una migración registrada (`0.0.0 -> 0.1.0`, pura, idempotente), un servicio de checkpoints funcional (crear/listar/restaurar/podar, restauración atómica, retención por defecto de 5) y un `executeUpgrade` rollback-first (checkpoint antes de mutar, `failWithRollback` restaura y relanza ante cualquier fallo posterior). `--dry-run`/`--from-checkpoint <id>`/`--target-version <v>`/`--no-sync`/`--no-doctor` son flags reales de Commander, cada uno con test dedicado. La salida de `axiom upgrade` (`formatPlan`/`formatResult`) imprime solo un resumen de transición de versión (`fromVersion`/`toVersion`/conteo de migraciones/`checkpointId`/`syncRun`/`doctorRun`) — no se muestra información de cambio a nivel fichero o por repo hoy, aunque `CheckpointRecord.files` ya guarda los paths relevantes internamente.

## Artefacto de ledger de revisión y entrada MCP nativa de engram — tanda INC-20260709-*

- **`review-ledger.md` (artefacto de revisión, INC-20260709-review-findings-ledger)**: cuando el flujo de revisión (ver [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md)) persiste su ledger de hallazgos vía el artifact store, lo hace como `review-ledger.md` DENTRO de la carpeta folder-per-artifact del cambio (`<specPath>/{increments,bugs}/<ID>/review-ledger.md`) — un fichero adicional junto a `README.md`/`metadata.yml`, no un nuevo `ArtifactKind` ni un cambio de `metadata.yml`. Si no hay carpeta de artefacto, el ledger cae a un topic de Engram (`topicKey` `sdd/<change>/review-ledger`) o a in-context. La forma del contrato (campos del ledger) se bundlea como constante TS única en `@axiom/document-bootstrap` (`review-ledger-contract.ts`).
- **Entrada MCP nativa de engram (INC-20260709-engram-mcp-stdio-native-config)**: la config MCP nativa por herramienta (ver "Config MCP nativa por herramienta" en [08_Glosario.md](08_Glosario.md) y [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md)) gana, cuando `workspace.json#providers` incluye `engram`, una entrada `engram` de stdio local (`command:'engram'`, `args:['mcp','--project',<projectId>,'--tools','agent']`) en la forma nativa de cada tool. Es machine-local y project-pinned; NO se añade a `.axiom/mcp.yml` (reservado a los brokers `type:'axiom'`). Nunca se emite forma HTTP (puerto 7437/`ENGRAM_URL`).

## Separación de responsabilidades: instalación del ARQUITECTO vs. instalación del MIEMBRO (`INC-20260710-per-member-install`)

Axiom ya separaba, de forma implícita, dato SHARED/committeado (`axiom.yaml`, `axiom.config/*.yaml`, la spec) de dato PERSONAL/no-versionado (`.axiom-state/`) — ver "Estado project-scoped" y "Topología de repos..." arriba. Este incremento hace esa separación EXPLÍCITA como modelo de dos actores y cierra los huecos que el audit encontró:

| Actor | Qué hace | Artefactos que produce | ¿Se commitea? |
|---|---|---|---|
| **ARQUITECTO** (una vez, primer install) | `axiom workspace setup` (o `axiom init`) | `axiom.yaml` (por repo), `axiom.config/topology.yaml`, `axiom.config/mcp-manifest.yaml`, `axiom.config/toolchain-catalog.yaml`, `axiom.config/toolchain.yaml` (subset habilitado del catálogo), skills seed, spec base | **Sí** — SHARED, define QUÉ repos/MCPs/utilidades existen para el proyecto |
| **MIEMBRO** (cada clone, cada máquina) | `axiom member install --member <id>` | `.axiom-state/local/topology-bindings.yaml` (paths reales en SU máquina), config MCP nativo (`.mcp.json`/`.cursor/mcp.json`/`.vscode/mcp.json`/`opencode.json`) con un launch command RESOLVABLE en SU máquina, estado local de activación de toolchain (`.axiom-state/<projectName>/toolchain/<id>/`), su propia entrada en `.axiom-state/<projectName>/members.yaml` y `.axiom-state/<projectName>/init.json` | **No** — PERSONAL, resuelve DÓNDE viven los repos y CÓMO se lanza cada cosa en SU máquina |

### Hallazgos del audit (Step 0) y su fix

1. **`buildGitignore()` (`apps/cli/src/commands/init.ts`) sólo ignoraba `.axiom-state/local/`, no `.axiom-state/` completo.** El `.gitignore` real, a mano, del propio repo `Axiom/` siempre ignoró `.axiom-state/` entero ("Axiom local overlay (per-repo runtime state; never versioned)") — el generador estaba desalineado con esa convención: `.axiom-state/<projectId>/` (`members.yaml`, `init.json`, `workspace.json`, checkpoints, telemetry, state-machine) habría quedado versionable en cualquier proyecto scaffoldeado por `init`/`workspace setup`. Fix: `buildGitignore()` ahora ignora `.axiom-state/` completo (además de `.axiom/`, `node_modules/`, `dist/`).
2. **`axiom join`/`axiom member install` no podían completarse nunca sobre un proyecto bootstradeado vía `axiom workspace setup`.** El gate del orchestrator (`hasInitJson`, `@axiom/orchestrator`) exige `.axiom-state/<projectName>/init.json` para `join-command` — sólo `axiom init` lo escribe; `axiom workspace setup` no, y aunque lo escribiera sería inútil (vive en `.axiom-state/`, personal/gitignored, nunca llega al clone de un miembro). `axiom member install` ahora sintetiza ese `init.json` local ANTES de invocar `join` (best-effort, hereda `profile`/`overlay`/adapter real de `workspace.json` si existe; nunca clobberea uno preexistente) — mismo criterio que el resto del incremento: estado local, materializado por quien lo necesita.
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
  para `codegraph` la evidencia alternativa de una
  `.codegraph/codegraph.db` real no vacía) confirmó positivamente que
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

### `axiom member install` y `axiom bindings` (nuevos comandos)

Ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md) para la superficie CLI completa. Resumen de datos:

- `axiom member install --member <id> [--bind <repoId>:<path>]... [--no-register] [--home-dir <path>] [--json]` (`apps/cli/src/commands/member-install.ts`): reusa `runJoin` para el registro del miembro; reusa `@axiom/topology` (`loadTopology`/`loadLocalBindings`/`saveLocalBindings`/`resolveRepoPath`) para bindings — persiste sólo los `--bind` explícitos (los auto-detectados por `ref` no se persisten, ya cubiertos por el fallback existente de `resolveRepoPath`); repos sin resolver quedan listados con una nota apuntando a `axiom bindings set`. Materializa el config MCP nativo desde `axiom.config/mcp-manifest.yaml` (committeado) con `resolveMcpLaunchCommand()`. Idempotente end-to-end: cada paso individual ya lo es (join dedupe, `saveLocalBindings` sobreescribe con el mismo valor, `writeNativeMcpConfig` es merge-preserving, `repairTool` es no-op si ya `present`).
- `axiom bindings show|set --repo <id> --path <path>|remove --repo <id>` (`apps/cli/src/commands/bindings.ts`): CRUD acotado sobre `topology-bindings.yaml`, para arreglar UN SOLO repo sin re-correr todo el install.

## Pendientes conocidos de este modelo

- **D1 — multi-repo como modo primario/por defecto.** `defaultSingleRepoManifest` (`@axiom/topology/src/loader.ts`) sigue sin lógica de warning de deprecación en su fallback silencioso, y el check `TC-001` de `@axiom/doctor` sigue sin una rama `warn` correspondiente. Es la única mitad restante del par original D1/D3 (D3, la ruta opt-in `schemaVersion: 2`, está cerrada — ver arriba). D1 se secuenció deliberadamente después de D3 porque su warning solo tiene sentido una vez existe una ruta de opt-in genuina para los usuarios, lo cual ya es cierto. No programado a ningún incremento concreto todavía.
- **`@axiom/versioning`'s hueco de forma `projectContractVersion`/`assets.*`/`appliedMigrations`** — cuatro conceptos del documento fuente sin equivalente en `ManagedState` hoy (ver arriba). Ya no bloqueado por la falta de una ruta `schemaVersion: 2`, pero no programado todavía.
- **Reporte de cambios por repo en `axiom upgrade`** (addendum §8) — necesita tanto un `TopologyManifest` real (no-default) como un plan aprobado activo, más cableado de `axiom upgrade` consciente de topología. Genuinamente desbloqueado ahora que D3 está cerrado, pero ningún incremento ha construido el cableado todavía.
- **Registro de preguntas abiertas de arquitectura Q1-Q5** — ver el cierre completo en `specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`. Resumen: Q1 (single-repo deprecado vs. opt-in) no formalmente decidido pero superado en la práctica (single-repo sigue siendo el default); Q2 (formato de registro) resuelto pragmáticamente como se describe arriba; Q3 (supervivencia del prefijo `axiom.spec/` tras la separación de repos) formalmente sigue abierto, no bloqueante; Q4 (relación de capability/provider/telemetry/gateway con el modelo MCP nuevo) resuelto — ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md); Q5 (formato destino de `axiom.yml`) resuelto y entregado como se describe arriba.

## Superficies SDD por rol de repo + canal de inyección por proyecto (2026-07-15) — tanda INC-20260715-*

El generador de superficies de proceso (`workspace-process-surfaces.ts`, `surfaceIdsForRole`) materializa por rol de repo, tras esta tanda:

- **`sdd`** (repo de control/orquestación): `axiom-sdd-orchestrator`, `axiom-phase-reviewer`, `axiom-qa-validator` — orquestación + gates de revisión/QA.
- **`spec`** (repo de especificación): `axiom-spec-author`, `axiom-role-planner`, `axiom-spec-integrator`, `axiom-tech-context` — autoría, planificación, consolidación y contexto técnico.
- **`code`** (repos de código): `axiom-role-implementer` — parametrizado por `{role}`/`{repoPath}`.

**Canal de inyección por proyecto** (dónde va lo específico de cada stack, manteniendo el producto genérico): (i) `axiom.config/skills-index/<role>.yaml` — índice de skills por rol que leen las superficies (`sdd.skillIndexRead`); (ii) el contexto técnico del proyecto, propiedad de `axiom-tech-context`; (iii) las skills de rol del proyecto. Las superficies del producto se mantienen adapter/stack-agnósticas y parametrizables; a diferencia de un sistema role-specialized que hornea las reglas de stack en agentes por rol, Axiom las deja como DATO del proyecto para funcionar en cualquier adapter/stack sin perder profundidad. Guía operable en [manuales/13_Skills_Agentes_y_Roles.md](manuales/13_Skills_Agentes_y_Roles.md).