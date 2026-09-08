# Modelo de datos y configuración

Fuente: `Axiom/docs/configuration/**`, `Axiom/docs/generated-files.md`, `Axiom/docs/cli/*.md`, `Axiom/packages/filesystem-truth/src/{discovery,state-paths}.ts`, `Axiom/packages/persistence/src/{filesystem-store,isolation}.ts`, `Axiom/packages/versioning/src/checkpoints.ts`, `Axiom/apps/cli/src/commands/init.ts`.

> Reconciliado 2026-07-29: este documento describía dos nombres de carpeta que `INC-20260703-config-folder-renames` (cerrado) ya renombró en el código real — el overlay oculto project-scoped (antiguo prefijo, hoy `.axiom-state`) y la carpeta de catálogo declarativo (antigua `axiom.spec` + subcarpeta `config`, hoy `axiom.config`). Verificado en `packages/filesystem-truth/src/discovery.ts#LOCAL_OVERLAY_DIRNAME`/`AXIOM_CONFIG_DIRNAME` (re-exportados vía `@axiom/core`); no queda ningún literal de los nombres antiguos en el código fuente (solo en `dist/` sin recompilar).

## `axiom.yaml` — identidad local y región gestionada

`axiom init` y las operaciones de workspace materializan la identidad del repo.
En un repo `code`, el documento contiene `projectId`, `repoId`, `kind` y el
pointer `axiomRepo` hacia la autoridad topológica; no replica el grafo. El modo
efectivo sigue siendo `local-only` y el adapter pertenece a los 8 targets
canónicos activos. `configure` puede normalizar el literal histórico
`copilot-vscode` desde `init.json`, pero ese valor no es un target público.

Los writers estructurales gestionan únicamente el bloque delimitado
`# AXIOM:MANAGED:START` / `# AXIOM:MANAGED:END`. Al actualizarlo preservan byte
a byte el prefijo, sufijo, comentarios y extensiones humanas. Un documento
markerless solo se convierte cuando el parser demuestra identidad compatible y
ningún campo conflictivo; YAML inválido, ambiguo o foráneo falla en preflight.
La lectura de compatibilidad de `axiom.yaml` v1/v2 en consumidores concretos no
reintroduce una topología schema 1 ni autoriza reserializar contenido humano.

`init.json`, `install-profile.json`, `last-start.json` y `last-sync.json` son
estado derivado y no deben editarse salvo diagnóstico. `init.json` no incluye
`projectName`: persiste `profileTriple`, `createdAt` y `version`; el segmento
físico se deriva de `projectKey` (`projectId` v2 o slug estable de v1).

## `.axiom-state/` — estado project-scoped

> Renombrado desde el antiguo prefijo oculto (`sdd`, con punto delante, sin este sufijo `-state`) por `INC-20260703-config-folder-renames` (cerrado). Verificado: `packages/filesystem-truth/src/discovery.ts#LOCAL_OVERLAY_DIRNAME = '.axiom-state'`.

- `.axiom-state/local/`: overlay NO versionada exclusivamente local al repo/operador: overrides, bindings de topología y audit trail.
- `.axiom-state/<projectKey>/`: único namespace físico del estado ligado al proyecto: `init.json`, `members.yaml`, `install-profile.json`, `workspace.json`, `last-start.json`, `last-sync.json`, `toolchain.lock`, `managed-state.json`, `model-assignments.json`, `components-state.json`, workflow, memoria, MCP bindings, plugins, skills pendientes, checkpoints y `structural-transactions/<operationId>/`.

`WorkspaceStateV1` es el único contrato de `workspace.json`: exige `projectId`,
`adapters`, `providers`, `createdAt` y `updatedAt`, admite solo las extensiones
`profile`/`overlay` y rechaza schema futuro/campos/tipos/identidad inválidos. La
ausencia `ENOENT` inicializa estado únicamente dentro de un update autorizado;
el writer usa lock + temporal validado + rename y conserva bytes en no-op.

`LocalBindingsV2` (`schemaVersion: 2`) vive en `local/`, separado de topology.
Solo admite IDs del manifest y paths absolutos canonicalizados. Los journals
estructurales contienen `intent.json`, hashes, staging y estado durable; recovery
los inspecciona antes de una mutación nueva y deja `recovery-required` cuando no
puede demostrar con seguridad rollback o roll-forward.
- `.axiom-state/executions/<executionId>/`: estado aislado por ejecución, separado del namespace project-scoped.

La lectura de estado legacy sigue la precedencia canonical, proyecto directo,
`config`, scope antiguo y archivos conocidos bajo `local/`. `state-paths.ts`
migra archivos con temp+rename, elimina la fuente solo después de confirmar el
destino y emite `StatePathWarning` ante conflictos o fallos. El scope API
`config` se conserva para compatibilidad, pero `resolveScopeDir` lo dirige a la
raíz del projectKey y no crea `.axiom-state/config/`.

Los checkpoints aplican la misma regla semántica: `list`/`restore` aceptan
aliases y raíces legacy, y `restoreCheckpoint` remapea los paths del manifest
al projectKey canónico antes de eliminar el destino antiguo. Esto cubre también
los buckets legacy conocidos de workflow, memoria, MCP, outputs y local.

## `axiom.config/*.yaml` — catálogo declarativo esperado dentro del proyecto adoptante

> Renombrado desde la antigua carpeta `axiom.spec` con su subcarpeta `config` por `INC-20260703-config-folder-renames` (cerrado). Verificado: `packages/filesystem-truth/src/discovery.ts#AXIOM_CONFIG_DIRNAME = 'axiom.config'`. **No confundir con `Axiom.Spec/`** (este repo de workspace, mayúsculas, estructura totalmente distinta).

Según `Axiom/docs/configuration/README.md`, el runtime espera ~20 YAML (mismo catálogo, solo cambió el nombre de la carpeta contenedora):

`axiom.workspace.yaml`, `branch-policy.yaml`, `capabilities.yaml`, `clarification-policy.yaml`, `command-protocol.yaml`, `external-work-items.yaml`, `id-policy.yaml`, `integrations.yaml`, `lifecycle-policy.yaml`, `local-overlay-policy.yaml`, `model-routing-policy.yaml`, `onboarding.yaml`, `orchestration-policy.yaml`, `policy-as-code.yaml`, `profiles.yaml`, `providers.yaml`, `repositories.yaml`, `scaffolding-contract.yaml`, `skills.yaml`, `telemetry-sinks.yaml`, `tool-routing-policy.yaml`.

No todos se consumen con el mismo nivel de profundidad hoy en runtime. **Esta carpeta ya existe en la raíz del propio repo `Axiom/`** (`Axiom/axiom.config/`, verificado por listado directo: contiene al menos `integrations.yaml`, `policy-as-code.yaml`, `toolchain-catalog.yaml`) — la brecha 1 de `../references/03-riesgos-y-brechas-conocidas.md` (self-bootstrap ausente) está RESUELTA desde `INC-20260708-product-repo-self-bootstrap`.

Además, desde `INC-20260727-adoption-config-scaffolding` (cerrado), `axiom workspace setup`/`axiom workspace adopt` (vía el motor compartido `runWorkspaceSetup`) siembran, best-effort y no-clobber, `axiom.config/integrations.yaml` (PC-001), `axiom.config/policy-as-code.yaml` (PC-002) y `axiom.config/agents-catalog.yaml` (TC-011) más el `axiom.skills.lock` raíz (GC-001/GC-002/GC-007) en un proyecto recién adoptado/configurado — antes ningún flujo de scaffolding los producía. Fuente: `apps/cli/src/commands/workspace-config-scaffold.ts`, `workspace-catalog-scaffold.ts`.

### Bloques de cada YAML relevante (documentados)

- **`capabilities.yaml`**: `capabilities.required/.optional/.postMvpOptional`, `supportLevels`, `degradationPolicy` y, cuando aplica, `mcpOnlyCapabilities`. El modelo provider-routed usa `id`, `domain` (`sdd`|`spec`|`code`|`memory`), `name`, `version`, `compliance`, `requiredTools`, `optionalTools`, `fallbacks`, `deprecated` y `schemaRef`; las tres capabilities MCP-only `axiom.*` se mantienen en su mapa separado.
- **`providers.yaml`**: registry de cuatro providers locales (`filesystem`, `serena`, `cmm`, `engram`) y el único perfil de discovery `local-only`, con `discoveryOrder: [filesystem]` y fallbacks declarados.
- **`profiles.yaml`**: dato bundleado con el perfil funcional único `builder`, la política operativa única `local-only` y los `adapterTarget` permitidos.
- **`command-protocol.yaml`**: `explicitCommands`, `runtimeCommands` (nombre, dónde corre, qué lee/escribe, si exige binding explícito de proyecto), `intentCommands`, `safety` (confirmaciones y bloqueos).
- **`policy-as-code.yaml`**: `sensitivityTags`, `artifactLifecycle`, `tools`/`compliance`, `projectIsolation`, `doctorValidation`.
- **`telemetry-sinks.yaml`**: política `local-only` y sinks locales (`local-audit-trail` y `local-log`), con audit trail append-only, sidecar SHA-256 y retención `P365D` por defecto.
- **`onboarding.yaml`**: preguntas/defaults/docs generados de `init`, qué lee/escribe `join`, requisitos de `doctor`/`configure`/`start`, `generatedDocs`, `repairPlaybooks`.
- **`scaffolding-contract.yaml`**: contrato de qué se siembra en `init` vs `configure`.

## Ficheros generados por comando

> Tabla actualizada para R13: `.axiom-state` es el estado local y `TopologyManifest.schemaVersion: 2` tiene una única autoridad física. `init` no crea una topología derivada. Un repo `code`/`legacy` solo conserva identidad y el pointer `axiomRepo`; el manifest se lee desde `<axiomRepo>/axiom.config/topology.yaml`. Ausencia de pointer, autoridad, YAML o schema válido es fail-closed: no existe fallback desde `axiom.yaml` ni materialización por repo.

| Comando | Escribe |
|---|---|
| `init` | `axiom.yaml`, `.gitignore`, `.axiom-state/local/`, `.axiom-state/<projectKey>/`, `init.json` (sin `topology.yaml`) |
| `join` | `.axiom-state/<projectKey>/members.yaml` |
| `configure` | `.axiom-state/<projectKey>/install-profile.json` (+ surfaces del target) |
| `sync` | `.axiom-state/<projectKey>/last-sync.json` (+ outputs del adapter) |
| `start` | `.axiom-state/<projectKey>/last-start.json` |
| `upgrade` | `.axiom-state/<projectKey>/managed-state.json`, checkpoints |
| `toolchain upgrade` | `.axiom-state/<projectKey>/toolchain.lock` (schema 1), con checkpoint/rollback |
| `model set/unset/reset` | `.axiom-state/<projectKey>/model-assignments.json` (+ `.opencode/model-routing.json`) |
| `components install/uninstall` | `.axiom-state/<projectKey>/components-state.json` |
| `workspace setup` / `repo add` | Actualizan únicamente `<axiomRepo>/axiom.config/topology.yaml`; escriben identidad/pointer en repos code/legacy sin copiar el manifest |
| `roles assign` | Actualiza el manifest autoral después de resolver y validar su autoridad; no crea una topología local alternativa |

## Ficheros generados por adapter target

> 8 packages de adapter / 8 targets canónicos activos — ver `../architecture/04-adapters-y-model-routing.md` para el detalle completo y actualizado (incluye `codex` y las superficies portables `.axiom/agents|commands|skills/`, no repetidas aquí). LiteLLM fue retirado y `visual-studio-2026` delega en la instrucción común de Copilot.

| Target | Archivos |
|---|---|
| `opencode` | `.opencode/AGENTS.md`, `.opencode/skills-lock.yaml` |
| `claude-code` | `.claude/AGENTS.md` |
| `github-copilot` | `.github/copilot-instructions.md` |
| `vscode` | `.vscode/settings.json` |
| `cursor` | `.cursor/settings.json`, `.cursor/AGENTS.md`, `.cursor/rules/axiom-common.mdc` (potencial no-clobber) |
| `codex` | `.codex/AGENTS.md` (generador de primera clase desde `INC-20260726-adapter-mcp-parity`) |
| `antigravity` | `.antigravity/AGENTS.md` |
| `visual-studio-2026` | `.github/copilot-instructions.md` (común; no genera `.vs/AXIOM.md`) |

Caso especial Copilot: `configure`, `sync`, `workspace setup` y el adapter
`github-copilot` escriben la instrucción general compartida en
`.github/copilot-instructions.md` vía `@axiom/document-bootstrap`. El template
versionado de `axiom.spec/templates/` gana cuando es legible y el bundle actúa
como fallback. El writer conserva el contenido humano fuera de
`AXIOM:GENERATED` y `TEAM:CUSTOM`, usa escritura atómica y migra de forma
conservadora la ruta legacy `.vscode/copilot-instructions.md`. `.vscode/` queda
para `settings.json`, `extensions.json` y `mcp.json`; las instrucciones por
ruta se escriben en `.github/instructions/*.instructions.md`.

## Instalación user-level (fuera del proyecto)

El binario `axiom` se instala una sola vez por operador (no por proyecto), vía `scripts/install-global.mjs`:
- macOS/Linux: `npm prefix -g` o `$HOME/.local/bin` (`npm link`).
- Windows: `%USERPROFILE%\.local\bin\axiom.cmd` (shim, con rollback si el smoke post-install falla).

Manifest de versión user-level: `~/.axiom/install.json` (`@axiom/user-workspace`, `UserWorkspacePaths.installPath`). `axiom self-update` gestiona esta versión, separada del `axiom upgrade` project-scoped; requiere `--apply` explícito para mutar (preview-only por defecto).

## Catálogo user-level de proyectos

Fuente: `Axiom/packages/user-workspace/src/{registry,registry-id,registry-types,errors}.ts`, `Axiom/packages/core/src/local-file.ts`, `Axiom/apps/cli/src/commands/projects.ts`.

`~/.axiom/projects.yml` (`ProjectsFile`, `schemaVersion: 2`) es el único catálogo de proyectos de la máquina. La ausencia del archivo equivale a `{schemaVersion: 2, projects: {}}`; no hay lector, tipos, error ni migrador de `registry.json`. Cada entrada contiene `id`, `name`, timestamps ISO y un mapa no vacío de repos `{ role, path }`. El parser valida de forma estricta el documento completo, incluidas clave/id, campos desconocidos y ownership global de paths, y devuelve `UserWorkspaceError` discriminado en lugar de lanzar.

La identidad explícita usa slug ASCII; la derivación de nombre normaliza NFKC, elimina diacríticos y colapsa separadores. Los paths se resuelven y se comparan con canonicalización física cuando existe el destino, respetando la sensibilidad de la plataforma. Una colisión de ID, identidad o path falla antes de escribir; `upsertProjectReposV2` solo acepta una reejecución que conserve la misma identidad y ownership.

Las mutaciones envuelven todo el ciclo load→validate→modify→save con `withLocalFileLockSync`. El primitivo de `@axiom/core` publica `owner.json` y lo fuerza a disco antes de retirar un lease de inicialización adyacente; los reclaim claims quedan ligados a la generación/epoch observada, de modo que ni un reclamador retrasado ni la ventana `mkdir`→owner pueden retirar un lock sucesor. `atomicWriteFileSync` usa un temporal propietario PID+UUID, flush/fsync, validación previa y rename atómico; la recuperación está limitada a temporales, leases y claims huérfanos que el primitivo puede atribuir con seguridad.

La proyección de estado distingue `present-directory`, `missing` y `not-directory`; el agregado es `available`, `partial` o `unavailable`. La resolubilidad Axiom se calcula aparte y solo cuando el caller aporta el probe de `@axiom/project-resolution`.
