# Modelo de datos y configuración

Fuente: `Axiom/docs/configuration/**`, `Axiom/docs/generated-files.md`, `Axiom/docs/cli/*.md`, `Axiom/packages/filesystem-truth/src/{discovery,state-paths}.ts`, `Axiom/packages/persistence/src/{filesystem-store,isolation}.ts`, `Axiom/packages/versioning/src/checkpoints.ts`, `Axiom/apps/cli/src/commands/init.ts`.

> Reconciliado 2026-07-29: este documento describía dos nombres de carpeta que `INC-20260703-config-folder-renames` (cerrado) ya renombró en el código real — el overlay oculto project-scoped (antiguo prefijo, hoy `.axiom-state`) y la carpeta de catálogo declarativo (antigua `axiom.spec` + subcarpeta `config`, hoy `axiom.config`). Verificado en `packages/filesystem-truth/src/discovery.ts#LOCAL_OVERLAY_DIRNAME`/`AXIOM_CONFIG_DIRNAME` (re-exportados vía `@axiom/core`); no queda ningún literal de los nombres antiguos en el código fuente (solo en `dist/` sin recompilar).

## `axiom.yaml` — manifiesto raíz por proyecto adoptante

Generado por `axiom init`. Campos relevantes (`Axiom/docs/configuration/project-structure.md`): `project.name`, `project.status`, `project.product_implementation_status`, `project.mode`, `scopes`, `rules`, `artifact_id_policy`, `lifecycle_commands`, `initial_capabilities`.

Materializa la configuración efectiva `builder` + `local-only` + `adapterTarget`. `builder` y
`local-only` son implícitos y no seleccionables; los estados legacy se normalizan en
los bordes de lectura. `adapterTarget` pertenece al conjunto de 10 targets canónicos
declarados (ver `../architecture/04-adapters-y-model-routing.md`).

Editable con criterio a mano; no editar `init.json`, `install-profile.json`, `last-start.json`, `last-sync.json` salvo diagnóstico (son estado derivado). **`init.json` ya no incluye `projectName`** (`INC-20260703-config-dedup`, cerrado): solo persiste el campo de compatibilidad `profileTriple` normalizado a `builder` + `local-only` + target, además de `createdAt` y `version`. El segmento físico se deriva de `projectKey`: `projectId` v2 o slug estable de `project.name` v1.

## `.axiom-state/` — estado project-scoped

> Renombrado desde el antiguo prefijo oculto (`sdd`, con punto delante, sin este sufijo `-state`) por `INC-20260703-config-folder-renames` (cerrado). Verificado: `packages/filesystem-truth/src/discovery.ts#LOCAL_OVERLAY_DIRNAME = '.axiom-state'`.

- `.axiom-state/local/`: overlay NO versionada exclusivamente local al repo/operador: overrides, bindings de topología y audit trail.
- `.axiom-state/<projectKey>/`: único namespace físico del estado ligado al proyecto: `init.json`, `members.yaml`, `install-profile.json`, `workspace.json`, `last-start.json`, `last-sync.json`, `toolchain.lock`, `managed-state.json`, `model-assignments.json`, `components-state.json`, workflow, memoria, MCP bindings, plugins, skills pendientes y checkpoints.
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

> Tabla actualizada 2026-07-29: el prefijo oculto antiguo se sustituyó por `.axiom-state` en todas las filas (renombre verificado). Además, `init` **ya no** escribe `topology.yaml` para el layout `installed-multi-repo` (`INC-20260703-config-dedup`, cerrado): `topology.yaml` pasó a ser opt-in / derivado-en-lectura — `@axiom/topology#loadTopology` deriva un manifest de fallback a partir de `axiom.yaml` cuando el fichero está ausente (`tryLoadTopologyHint` + `defaultInstalledMultiRepoManifest`), y `topology.yaml` solo se materializa de forma perezosa cuando el proyecto corre `axiom roles assign`. Fuente: comentario junto a la escritura de `init.json` en `apps/cli/src/commands/init.ts`.

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
| `roles assign` | Materializa `axiom.config/topology.yaml` de forma perezosa si estaba ausente (nuevo comportamiento, ver nota arriba) |

## Ficheros generados por adapter target

> 9 packages de adapter / 10 targets canónicos / 8 "headline" — ver `../architecture/04-adapters-y-model-routing.md` para el detalle completo y actualizado (incluye `codex` y las superficies portables `.axiom/agents|commands|skills/`, no repetidas aquí).

| Target | Archivos |
|---|---|
| `opencode` | `.opencode/AGENTS.md`, `.opencode/skills-lock.yaml` |
| `claude-code` | `.claude/AGENTS.md` |
| `github-copilot` / `copilot-vscode` | `.github/copilot-instructions.md` (+ `.vscode/settings.json`, `.vscode/extensions.json` en `copilot-vscode`) |
| `vscode` | `.vscode/settings.json` |
| `cursor` | `.cursor/settings.json`, `.cursor/AGENTS.md` |
| `litellm` | `litellm.config.json` |
| `codex` | `.codex/AGENTS.md` (generador de primera clase desde `INC-20260726-adapter-mcp-parity`) |
| `antigravity` | `.antigravity/AGENTS.md` |
| `visual-studio-2026` | `.vs/AXIOM.md` |

Caso especial Copilot: `configure` puede escribir `.github/copilot-instructions.md` vía `@axiom/document-bootstrap`, usando template versionado en `axiom.spec/templates/`, resolviendo variables del proyecto y preservando el bloque `TEAM:CUSTOM` con escritura atómica.

## Instalación user-level (fuera del proyecto)

El binario `axiom` se instala una sola vez por operador (no por proyecto), vía `scripts/install-global.mjs`:
- macOS/Linux: `npm prefix -g` o `$HOME/.local/bin` (`npm link`).
- Windows: `%USERPROFILE%\.local\bin\axiom.cmd` (shim, con rollback si el smoke post-install falla).

Manifest de versión user-level: `~/.axiom/install.json` (`@axiom/user-workspace`, `UserWorkspacePaths.installPath`). `axiom self-update` gestiona esta versión, separada del `axiom upgrade` project-scoped; requiere `--apply` explícito para mutar (preview-only por defecto).
