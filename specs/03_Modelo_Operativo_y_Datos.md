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

## Estado project-scoped: `.sdd/`

- `.sdd/local/`: overlay no versionada (overrides locales, markers como `last-sync.json`). Regida por `local-overlay-policy.yaml`.
- `.sdd/<projectName>/`: estado persistido del proyecto resuelto: `init.json`, `members.yaml`, `install-profile.json`, `last-start.json`.
- `.sdd/config/<projectName>/`: estado de mutación de subsistemas: `managed-state.json` (versionado), `model-assignments.json`, `components-state.json`, `gateway-state.json`.
- `.sdd/<projectName>/checkpoints/<id>/`: snapshots pre-mutación (upgrade, uninstall de components); se conservan los últimos 5.

## Catálogo de configuración declarativa: `axiom.spec/config/*.yaml`

El runtime espera, dentro del proyecto adoptante, una carpeta `axiom.spec/config/` (minúsculas — no confundir con `Axiom.Spec/`, el repo de este workspace) con ~20 YAML de política y capacidad (`Axiom/docs/configuration/README.md`):

`axiom.workspace.yaml`, `branch-policy.yaml`, `capabilities.yaml`, `clarification-policy.yaml`, `command-protocol.yaml`, `external-work-items.yaml`, `id-policy.yaml`, `integrations.yaml`, `lifecycle-policy.yaml`, `local-overlay-policy.yaml`, `model-routing-policy.yaml`, `onboarding.yaml`, `orchestration-policy.yaml`, `policy-as-code.yaml`, `profiles.yaml`, `providers.yaml`, `repositories.yaml`, `scaffolding-contract.yaml`, `skills.yaml`, `telemetry-sinks.yaml`, `tool-routing-policy.yaml`.

No todos se consumen hoy con el mismo nivel de profundidad en runtime, pero forman el mapa documental y de control declarado. Ver discrepancia real: esta carpeta **no existe hoy** en la raíz del propio repo `Axiom/` (detalle en `context/references/03-riesgos-y-brechas-conocidas.md`).

## Modelo de capabilities y providers

`capabilities.yaml` declara `capabilities.required` / `.optional` / `.postMvpOptional`, `supportLevels` y `degradationPolicy`. Cada capability tiene `id`, `domain` (`sdd`, `spec`, `code`, `memory`), `name`, `version`, `compliance`, `requiredTools`, `optionalTools`, `fallbacks`. `providers.yaml` declara el registry de providers y perfiles de discovery (`filesystem-first`, `gateway-first`, `local-only`) con `discoveryOrder`, `preferredProviders`, `optionalProviders`, `gatewayExpectation`. Implementado en `@axiom/capability-model` y consumido por `configure`, `start` y `doctor`.

## Ficheros generados por comando (ciclo de vida)

| Comando | Escribe |
|---|---|
| `init` | `axiom.yaml`, `.gitignore`, `.sdd/local/`, `.sdd/<projectName>/`, `init.json` |
| `join` | `.sdd/<projectName>/members.yaml` |
| `configure` | `.sdd/<projectName>/install-profile.json` (+ surfaces del target) |
| `sync` | `.sdd/local/last-sync.json` (+ regeneración de outputs del adapter) |
| `start` | `.sdd/<projectName>/last-start.json` |
| `upgrade` | `.sdd/config/<projectName>/managed-state.json`, checkpoints |
| `model set/unset/reset` | `.sdd/config/<projectName>/model-assignments.json` (+ `.opencode/model-routing.json` si target es opencode) |
| `components install/uninstall` | `.sdd/config/<projectName>/components-state.json` |

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

Un YAML global del producto no debe mezclar visión documental, builder tooling y runtime. Cada YAML debe pertenecer a una capa concreta y a una responsabilidad concreta. Vigente: el catálogo de `axiom.spec/config/*.yaml` ya está desglosado por responsabilidad concreta (policy, capability, telemetry, routing) en vez de un único fichero monolítico.

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
- `multi-repo` está completamente soportado por el schema (cada repo tiene un `id` y un `ref`, path relativo a `projectRoot` o path/URI absoluto). La resolución real de `LocalBindings` para multi-repo más allá de la heurística "ref relativo a projectRoot" sigue siendo una preocupación P1 — no existe todavía ningún proyecto con contenido no trivial de `topology-bindings.yaml` contra el que validar.
- Los bindings locales por usuario (`.sdd/local/topology-bindings.yaml`, `LocalBindings { schemaVersion: 1; localPaths: Record<string, string> }`) explícitamente no se versionan.
- Todos los helpers conscientes de topología del código (write-scope, límite de dogfooding, etc.) resuelven repos vía `loadTopology`/`loadLocalBindings`/`resolveRepoPath` de `@axiom/topology`, todos libres de `homeDir` — patrón establecido y reusable para cualquier check futuro que necesite ser parametrizado por rol en vez de hardcodeado a nombres de repo concretos.

### Registro global v2 (`~/.axiom/projects.yml`)

`Axiom/packages/user-workspace/src/registry.ts`: `schemaVersion: 2`, YAML, en `~/.axiom/projects.yml`, **aditivo** junto al legado `~/.axiom/registry.json` (`schemaVersion: 1`, JSON) en vez de un rename/reemplazo destructivo — ambos formatos son leídos/escritos por el mismo paquete, seleccionados según qué fichero está presente. La forma de `projects.yml` mapea un proyecto a sus `repos` (mapa por rol), consistente con el modelo de topología de arriba. `getProjectV2`/`addProjectV2`, etc. (la API `*V2`) es la superficie actual y de cara al futuro; el lector/escritor legado de `registry.json` se mantiene por compatibilidad con proyectos ya instalados.

### `axiom.yaml` — `schemaVersion: 2` (cutover cerrado)

`AxiomYamlSchemaV2` (`projectId`/`name`/`repoId`/`role`/`mode`/`paths`), emitido por `buildAxiomYaml` en `apps/cli/src/commands/init.ts` para ambos layouts de scaffold (`self-hosted` e `installed-multi-repo`). `resolveProject` (`@axiom/project-resolution`) y los checks `MC-001`/`BC-001`/`BC-002` de `@axiom/doctor` son conscientes de versión: aceptan tanto `schemaVersion: 1` (legado — `project.mode` + `scopes: { name: { path, product_runtime } }`) como `schemaVersion: 2`, nunca resolviendo mal uno como el otro. Este cutover quedó cerrado tras verificación end-to-end dos veces por dos roles distintos (incluyendo una reproducción de test negativo y una re-ejecución independiente de `axiom doctor` contra un proyecto v2 separado). Durante el cierre se encontraron y corrigieron dos consumidores v1-only no detectados antes: `configure.ts`'s `readAxiomYamlProjectName` (habría fallado duro para proyectos v2 sin `product.manifest.yaml`) y `@axiom/topology/src/loader.ts`'s `tryLoadTopologyHint` (habría resuelto mal, en silencio, la topología de un proyecto v2 `installed-multi-repo` al default single-repo).

### `mcp.yml` (config MCP por proyecto) vs `mcp-manifest.yaml`

`mcp.yml` es una declaración **por proyecto** de qué procesos servidor MCP expone un proyecto — genuinamente distinta del `mcp-manifest.yaml` preexistente (spec 0024). Las dos responden preguntas distintas y no deben fusionarse (decisión Q-mcp-1, declinada explícitamente):

| | `mcp-manifest.yaml` (spec 0024) | `mcp.yml` |
|---|---|---|
| Responde | "¿qué capabilities MCP declara el catálogo de este proyecto, y están vinculadas las obligatorias?" | "¿qué procesos servidor MCP están habilitados, en qué scope, para que los adapters generen config runtime?" |
| Forma | `McpEntry {id, displayName, capabilities, installMode, projectBinding, readonly}` | `McpServerEntry {id, type, scope, targetRepo?, enabled}` |
| Resolución | `@axiom/memory#resolveMemoryScope` | `getProjectV2` (registro v2) |
| Consumidor | `axiom mcp list\|validate\|repair\|inventory` | Generadores `mcp.json` por adapter |
| Escritura | `axiom mcp repair` / manual | El propietario del proyecto edita directamente |

Ningún loader lee el fichero del otro — sin acoplamiento runtime compartido. Schema (`@axiom/user-workspace`'s `mcp-config.ts`), cargado desde `<specRoot>/.axiom/mcp.yml`:

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

`loadMcpProjectConfig(path)` lee + parsea YAML + valida forma únicamente (sin validación semántica). `validateMcpProjectConfig(config, homeDir)` corre seis reglas acumulativas: `schema-version`, `unknown-project`, `duplicate-server-id`, `missing-target-repo`, `unknown-target-repo`, `unexpected-target-repo`, `duplicate-type-target-repo`. La generación por adapter (`generateOpencodeMcpJson`/`generateClaudeCodeMcpJson`) escribe los servers `enabled: true` verbatim en `.opencode/mcp.json`/`.claude/mcp.json`, con el mismo patrón atómico tmp-write-then-rename usado en el resto del producto; no se inventan campos de transporte (`command`, `args`, `env`). El aislamiento es por scoping simple de filesystem (`mcp.yml` vive dentro del repo de spec de un único proyecto) — no requiere código de `@axiom/isolation`. **No está aún cableado a producción**: `runConfigure`/`sync` no llaman a los generadores todavía, y ningún installer/scaffold escribe `mcp.yml` a un proyecto real; `.opencode/mcp.json`/`.claude/mcp.json` están intencionalmente excluidos de `GENERATED_FILES_BY_TARGET` hasta que exista un call site generador real.

### Versionado, upgrades y migración de schema (`@axiom/versioning`)

`ManagedState` (`Axiom/packages/versioning/src/managed-state.ts`, `schemaVersion: 1`, JSON en `<root>/.sdd/config/<projectName>/managed-state.json`): `{schemaVersion: 1, runtime: {package, version}, adapterTargets: [{id, version, lastSyncedAt}], lastUpgrade: {fromVersion, toVersion, at, checkpointId} | null, lastCheckpointId: string | null}`. Esta forma es estructuralmente distinta del `version.yml` propuesto originalmente por el documento fuente (`axiom: {projectContractVersion, installedWithAxiomVersion, lastUpgradedWithAxiomVersion}`, `assets: {sddSkillsVersion, adaptersVersion, guidesVersion, templatesVersion}`, `appliedMigrations: [...]`) — no es una variante de nombrado. Cuatro conceptos del documento fuente no tienen equivalente (`projectContractVersion`, `installedWithAxiomVersion`, los cuatro campos `assets.*`, y `appliedMigrations` como lista acumulativa — solo se conserva el `lastUpgrade` más reciente). Este hueco queda abierto (ver "Pendientes conocidos" abajo) pero ya no está bloqueado por la falta de una ruta `schemaVersion: 2`.

Mecanismo de migración/rollback confirmado real por lectura directa y ejecución completa de tests: exactamente una migración registrada (`0.0.0 -> 0.1.0`, pura, idempotente), un servicio de checkpoints funcional (crear/listar/restaurar/podar, restauración atómica, retención por defecto de 5) y un `executeUpgrade` rollback-first (checkpoint antes de mutar, `failWithRollback` restaura y relanza ante cualquier fallo posterior). `--dry-run`/`--from-checkpoint <id>`/`--target-version <v>`/`--no-sync`/`--no-doctor` son flags reales de Commander, cada uno con test dedicado. La salida de `axiom upgrade` (`formatPlan`/`formatResult`) imprime solo un resumen de transición de versión (`fromVersion`/`toVersion`/conteo de migraciones/`checkpointId`/`syncRun`/`doctorRun`) — no se muestra información de cambio a nivel fichero o por repo hoy, aunque `CheckpointRecord.files` ya guarda los paths relevantes internamente.

## Pendientes conocidos de este modelo

- **D1 — multi-repo como modo primario/por defecto.** `defaultSingleRepoManifest` (`@axiom/topology/src/loader.ts`) sigue sin lógica de warning de deprecación en su fallback silencioso, y el check `TC-001` de `@axiom/doctor` sigue sin una rama `warn` correspondiente. Es la única mitad restante del par original D1/D3 (D3, la ruta opt-in `schemaVersion: 2`, está cerrada — ver arriba). D1 se secuenció deliberadamente después de D3 porque su warning solo tiene sentido una vez existe una ruta de opt-in genuina para los usuarios, lo cual ya es cierto. No programado a ningún incremento concreto todavía.
- **`@axiom/versioning`'s hueco de forma `projectContractVersion`/`assets.*`/`appliedMigrations`** — cuatro conceptos del documento fuente sin equivalente en `ManagedState` hoy (ver arriba). Ya no bloqueado por la falta de una ruta `schemaVersion: 2`, pero no programado todavía.
- **Reporte de cambios por repo en `axiom upgrade`** (addendum §8) — necesita tanto un `TopologyManifest` real (no-default) como un plan aprobado activo, más cableado de `axiom upgrade` consciente de topología. Genuinamente desbloqueado ahora que D3 está cerrado, pero ningún incremento ha construido el cableado todavía.
- **Registro de preguntas abiertas de arquitectura Q1-Q5** — ver el cierre completo en `specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`. Resumen: Q1 (single-repo deprecado vs. opt-in) no formalmente decidido pero superado en la práctica (single-repo sigue siendo el default); Q2 (formato de registro) resuelto pragmáticamente como se describe arriba; Q3 (supervivencia del prefijo `axiom.spec/` tras la separación de repos) formalmente sigue abierto, no bloqueante; Q4 (relación de capability/provider/telemetry/gateway con el modelo MCP nuevo) resuelto — ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md); Q5 (formato destino de `axiom.yml`) resuelto y entregado como se describe arriba.