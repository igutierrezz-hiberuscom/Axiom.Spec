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
- **Registro de TODOS los repos en `~/.axiom/projects.yml`** (registro v2) bajo un único `projectId` (`= slugifyProjectId(projectName)`) en **una sola llamada idempotente** vía el helper nuevo `upsertProjectReposV2` (`@axiom/user-workspace`, `registry.ts`): carga → merge del mapa de repos (las claves de repo nuevas ganan en conflicto, porque un re-run significa "este workspace ahora conoce este path") → guarda, creando el proyecto si no existía. Re-ejecutar el mismo setup es seguro (no lanza `duplicate-id`). Reusable por futuros llamadores (`repo attach`, etc.). El registro es best-effort: un fallo de registro deja `registryRegistered: false` sin abortar el setup.
- Solo el repo de control recibe `topology.yaml` + `.axiom-state/<projectId>/`; los repos de rol/spec reciben `axiom.yaml` (+ `.gitignore`/`AGENTS.md` best-effort en directorios nuevos) y `.axiom-state/local/` + `.axiom-state/<projectId>/`.

Directorios nuevos (`create: true`) se scaffoldean desde cero; directorios existentes (`create: false`) se parametrizan in-place. **Guarda no-clobber**: si un `axiom.yaml` preexistente parsea con un `projectId` distinto (o es v1/no-parseable-pero-presente), su escritura se salta y se registra como warning, nunca como error (ver ángulo de ownership en [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md)).

Tras un registro exitoso, el motor invoca best-effort la generación de config MCP (`.axiom/mcp.yml` + proyección al adapter) — ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md) y la sección `mcp.yml` de abajo.

### Artefactos adicionales del setup de workspace (round 2, INC-20260705-*)

La segunda tanda de incrementos de workspace añade cuatro clases de artefacto persistido/scaffoldeado al motor. Todos son pasos **best-effort** (un fallo nunca aborta el setup) y comparten las semánticas de gating descritas al final de esta subsección. El comportamiento y las tablas de despacho viven en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md); aquí se documentan solo las formas de datos.

- **`<controlRepo>/.axiom-state/<projectId>/workspace.json`** — registro de la selección de adapters (`INC-20260705-workspace-adapters-multiselect`), escrito una sola vez en el repo de control (el mismo ancla project-scoped que `topology.yaml`/`topology-bindings.yaml`), incluido en `filesCreated`:

  ```ts
  interface WorkspaceSetupRecord {
    schemaVersion: 1;
    adapters: AdapterTarget[]; // los adapters multi-seleccionados
    profile: string;           // functional profile resuelto
    overlay: string;           // operational overlay resuelto
    createdAt: string;         // ISO timestamp
  }
  ```

- **`install-profile.json` por repo** — `generateWorkspaceAdapters` resuelve un `ResolvedInstallProfile` por repo del workspace (vía `installProfile`, un solo call por repo con `adapters[0]` como primario), persistido en el `.axiom-state/<projectId>/` de ese repo, con el mismo shape que ya escribe `axiom configure` (ver "Ficheros generados por comando"). Los ficheros de adapter derivados (`.opencode/AGENTS.md`, `.claude/AGENTS.md`, `.antigravity/AGENTS.md`, `.vs/AXIOM.md`, etc.) se escriben en **cada** repo por adapter seleccionado, según la tabla de despacho `target -> generador` de [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

- **Baseline de skills en el repo de control** (`INC-20260705-workspace-sdd-skills`), solo cuando el repo de control se crea recién:
  - `axiom.config/skills-catalog.yaml` (`schemaVersion: 1`) — catálogo semilla de 5 ids con `bundleHash` byte-exacto por entrada (`computeSkillBundleHash`); las fuentes de cada skill se escriben bajo `axiom.spec/target-axiom-skills/<id>.md`. Es el mismo `SkillsCatalog` que consume el check `TC-010` de doctor (ver [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md)).
  - `.opencode/agents/<id>/SKILL.md` (materializado por `applySkillSet`) + `.axiom-state/<projectId>/skills-pending.json`.
  - `axiom.config/skills-index/<role>.yaml` — un `SkillsRoleIndex` (schema de RF-AXM-020, ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md)) por cada rol funcional declarado en el workspace.

- **Base de spec en el repo de spec** (`INC-20260705-workspace-spec-base`), solo cuando el repo de spec se crea recién: la estructura canónica de spec + contexto técnico, scaffoldeada desde plantillas bundleadas como constantes TS (guardado per-file: nunca sobrescribe un fichero preexistente, skip + warning):
  - `specs/README.md` + `specs/00_Resumen_Ejecutivo.md` .. `specs/08_Glosario.md` (9 ficheros numerados);
  - `context/TECHNICAL_CONTEXT.md` + `context/README.md`;
  - directorios estructurales vacíos (vía `.gitkeep`): `specs/{increments,bugs,archive}/` y `context/{architecture,integrations,operations,references}/`.

**Semánticas de gating (`created`)**: la generación multi-adapter + `workspace.json` corre **siempre**, para todos los repos, tras un registro exitoso. La baseline de skills corre **solo si el repo de control se creó recién** en esta misma llamada (`repoResults.find(r => r.topologyId === control.topologyId)?.created === true`); la base de spec corre **solo si el repo de spec se creó recién** (`… === specRepo.topologyId …`). Un repo parametrizado in-place (`create: false`, ya existente) se salta silenciosamente en ambos casos — sin clobber, sin ruido. El flag `created` lo computa `writeOneRepo` antes en la misma llamada.

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
}
```

`loadMcpProjectConfig(path)` lee + parsea YAML + valida forma únicamente (sin validación semántica). `validateMcpProjectConfig(config, homeDir)` corre seis reglas acumulativas: `schema-version`, `unknown-project`, `duplicate-server-id`, `missing-target-repo`, `unknown-target-repo`, `unexpected-target-repo`, `duplicate-type-target-repo`. La generación por adapter (`generateOpencodeMcpJson`/`generateClaudeCodeMcpJson`) escribe los servers `enabled: true` verbatim en `.opencode/mcp.json`/`.claude/mcp.json`, con el mismo patrón atómico tmp-write-then-rename usado en el resto del producto; no se inventan campos de transporte (`command`, `args`, `env`). El aislamiento es por scoping simple de filesystem — no requiere código de `@axiom/isolation`.

El **primer y único call site generador vivo** hoy es el setup de workspace multi-repo (`runWorkspaceSetup`, INC-20260705-workspace-mcp-generation), que escribe `.axiom/mcp.yml` en el repo de control y proyecta a `.opencode/mcp.json`/`.claude/mcp.json` como paso best-effort final — ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md) para el detalle de la generación, los dos servers (`sdd-mcp-server`/`spec-mcp-broker`) y el caveat de que `.axiom/` está gitignoreado. Fuera de esa ruta el estado previo se mantiene: `runConfigure`/`sync` **no** llaman a los generadores, y `.opencode/mcp.json`/`.claude/mcp.json` siguen intencionalmente excluidos de `GENERATED_FILES_BY_TARGET` para la ruta `configure`/`sync` (deferral TR-005 intacto).

### Versionado, upgrades y migración de schema (`@axiom/versioning`)

`ManagedState` (`Axiom/packages/versioning/src/managed-state.ts`, `schemaVersion: 1`, JSON en `<root>/.axiom-state/config/<projectName>/managed-state.json`): `{schemaVersion: 1, runtime: {package, version}, adapterTargets: [{id, version, lastSyncedAt}], lastUpgrade: {fromVersion, toVersion, at, checkpointId} | null, lastCheckpointId: string | null}`. Esta forma es estructuralmente distinta del `version.yml` propuesto originalmente por el documento fuente (`axiom: {projectContractVersion, installedWithAxiomVersion, lastUpgradedWithAxiomVersion}`, `assets: {sddSkillsVersion, adaptersVersion, guidesVersion, templatesVersion}`, `appliedMigrations: [...]`) — no es una variante de nombrado. Cuatro conceptos del documento fuente no tienen equivalente (`projectContractVersion`, `installedWithAxiomVersion`, los cuatro campos `assets.*`, y `appliedMigrations` como lista acumulativa — solo se conserva el `lastUpgrade` más reciente). Este hueco queda abierto (ver "Pendientes conocidos" abajo) pero ya no está bloqueado por la falta de una ruta `schemaVersion: 2`.

Mecanismo de migración/rollback confirmado real por lectura directa y ejecución completa de tests: exactamente una migración registrada (`0.0.0 -> 0.1.0`, pura, idempotente), un servicio de checkpoints funcional (crear/listar/restaurar/podar, restauración atómica, retención por defecto de 5) y un `executeUpgrade` rollback-first (checkpoint antes de mutar, `failWithRollback` restaura y relanza ante cualquier fallo posterior). `--dry-run`/`--from-checkpoint <id>`/`--target-version <v>`/`--no-sync`/`--no-doctor` son flags reales de Commander, cada uno con test dedicado. La salida de `axiom upgrade` (`formatPlan`/`formatResult`) imprime solo un resumen de transición de versión (`fromVersion`/`toVersion`/conteo de migraciones/`checkpointId`/`syncRun`/`doctorRun`) — no se muestra información de cambio a nivel fichero o por repo hoy, aunque `CheckpointRecord.files` ya guarda los paths relevantes internamente.

## Pendientes conocidos de este modelo

- **D1 — multi-repo como modo primario/por defecto.** `defaultSingleRepoManifest` (`@axiom/topology/src/loader.ts`) sigue sin lógica de warning de deprecación en su fallback silencioso, y el check `TC-001` de `@axiom/doctor` sigue sin una rama `warn` correspondiente. Es la única mitad restante del par original D1/D3 (D3, la ruta opt-in `schemaVersion: 2`, está cerrada — ver arriba). D1 se secuenció deliberadamente después de D3 porque su warning solo tiene sentido una vez existe una ruta de opt-in genuina para los usuarios, lo cual ya es cierto. No programado a ningún incremento concreto todavía.
- **`@axiom/versioning`'s hueco de forma `projectContractVersion`/`assets.*`/`appliedMigrations`** — cuatro conceptos del documento fuente sin equivalente en `ManagedState` hoy (ver arriba). Ya no bloqueado por la falta de una ruta `schemaVersion: 2`, pero no programado todavía.
- **Reporte de cambios por repo en `axiom upgrade`** (addendum §8) — necesita tanto un `TopologyManifest` real (no-default) como un plan aprobado activo, más cableado de `axiom upgrade` consciente de topología. Genuinamente desbloqueado ahora que D3 está cerrado, pero ningún incremento ha construido el cableado todavía.
- **Registro de preguntas abiertas de arquitectura Q1-Q5** — ver el cierre completo en `specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`. Resumen: Q1 (single-repo deprecado vs. opt-in) no formalmente decidido pero superado en la práctica (single-repo sigue siendo el default); Q2 (formato de registro) resuelto pragmáticamente como se describe arriba; Q3 (supervivencia del prefijo `axiom.spec/` tras la separación de repos) formalmente sigue abierto, no bloqueante; Q4 (relación de capability/provider/telemetry/gateway con el modelo MCP nuevo) resuelto — ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md); Q5 (formato destino de `axiom.yml`) resuelto y entregado como se describe arriba.