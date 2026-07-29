# Modelo de datos y configuración

Fuente: `Axiom/docs/configuration/**`, `Axiom/docs/generated-files.md`, `Axiom/docs/cli/*.md`, `Axiom/packages/filesystem-truth/src/discovery.ts`, `Axiom/apps/cli/src/commands/init.ts`.

> Reconciliado 2026-07-29: este documento describía dos nombres de carpeta que `INC-20260703-config-folder-renames` (cerrado) ya renombró en el código real — el overlay oculto project-scoped (antiguo prefijo, hoy `.axiom-state`) y la carpeta de catálogo declarativo (antigua `axiom.spec` + subcarpeta `config`, hoy `axiom.config`). Verificado en `packages/filesystem-truth/src/discovery.ts#LOCAL_OVERLAY_DIRNAME`/`AXIOM_CONFIG_DIRNAME` (re-exportados vía `@axiom/core`); no queda ningún literal de los nombres antiguos en el código fuente (solo en `dist/` sin recompilar).

## `axiom.yaml` — manifiesto raíz por proyecto adoptante

Generado por `axiom init`. Campos relevantes (`Axiom/docs/configuration/project-structure.md`): `project.name`, `project.status`, `project.product_implementation_status`, `project.mode`, `scopes`, `rules`, `artifact_id_policy`, `lifecycle_commands`, `initial_capabilities`.

Encapsula el **profile triple**:

- `functionalProfile`: `builder` (más cubierto en runtime) | `product-owner`.
- `operationalOverlay`: `local-only` (fuerza filesystem, sin gateway) | `standard` (filesystem por defecto, gateway opt-in) | `enterprise` (exige gateway).
- `adapterTarget`: uno de **10 targets canónicos** declarados (8 "headline"; ver `../architecture/04-adapters-y-model-routing.md` — documento ya vigente, tratarlo como referencia de verdad para adapters/model-routing).

Triple recomendado para primer proyecto: `builder` + `local-only` + `opencode` (`Axiom/docs/first-project-readiness.md`).

Editable con criterio a mano; no editar `init.json`, `install-profile.json`, `last-start.json`, `last-sync.json` salvo diagnóstico (son estado derivado). **`init.json` ya no incluye `projectName`** (`INC-20260703-config-dedup`, cerrado): solo persiste `profileTriple`, `createdAt`, `version` — `projectName` es derivable del nombre del directorio `.axiom-state/<projectName>/` que lo contiene (verificado: `apps/cli/src/commands/init.ts`, comentario junto a la escritura de `init.json`).

## `.axiom-state/` — estado project-scoped

> Renombrado desde el antiguo prefijo oculto (`sdd`, con punto delante, sin este sufijo `-state`) por `INC-20260703-config-folder-renames` (cerrado). Verificado: `packages/filesystem-truth/src/discovery.ts#LOCAL_OVERLAY_DIRNAME = '.axiom-state'`.

- `.axiom-state/local/`: overlay NO versionada. Overrides locales, markers (`last-sync.json`). Regida por `local-overlay-policy.yaml`.
- `.axiom-state/<projectName>/`: `init.json`, `members.yaml`, `install-profile.json`, `last-start.json`.
- `.axiom-state/config/<projectName>/`: `managed-state.json`, `model-assignments.json`, `components-state.json`, `gateway-state.json`.
- `.axiom-state/<projectName>/checkpoints/<id>/`: snapshots pre-mutación, últimos 5 conservados.

## `axiom.config/*.yaml` — catálogo declarativo esperado dentro del proyecto adoptante

> Renombrado desde la antigua carpeta `axiom.spec` con su subcarpeta `config` por `INC-20260703-config-folder-renames` (cerrado). Verificado: `packages/filesystem-truth/src/discovery.ts#AXIOM_CONFIG_DIRNAME = 'axiom.config'`. **No confundir con `Axiom.Spec/`** (este repo de workspace, mayúsculas, estructura totalmente distinta).

Según `Axiom/docs/configuration/README.md`, el runtime espera ~20 YAML (mismo catálogo, solo cambió el nombre de la carpeta contenedora):

`axiom.workspace.yaml`, `branch-policy.yaml`, `capabilities.yaml`, `clarification-policy.yaml`, `command-protocol.yaml`, `external-work-items.yaml`, `id-policy.yaml`, `integrations.yaml`, `lifecycle-policy.yaml`, `local-overlay-policy.yaml`, `model-routing-policy.yaml`, `onboarding.yaml`, `orchestration-policy.yaml`, `policy-as-code.yaml`, `profiles.yaml`, `providers.yaml`, `repositories.yaml`, `scaffolding-contract.yaml`, `skills.yaml`, `telemetry-sinks.yaml`, `tool-routing-policy.yaml`.

No todos se consumen con el mismo nivel de profundidad hoy en runtime. **Esta carpeta ya existe en la raíz del propio repo `Axiom/`** (`Axiom/axiom.config/`, verificado por listado directo: contiene al menos `integrations.yaml`, `policy-as-code.yaml`, `toolchain-catalog.yaml`) — la brecha 1 de `../references/03-riesgos-y-brechas-conocidas.md` (self-bootstrap ausente) está RESUELTA desde `INC-20260708-product-repo-self-bootstrap`.

Además, desde `INC-20260727-adoption-config-scaffolding` (cerrado), `axiom workspace setup`/`axiom workspace adopt` (vía el motor compartido `runWorkspaceSetup`) siembran, best-effort y no-clobber, `axiom.config/integrations.yaml` (PC-001), `axiom.config/policy-as-code.yaml` (PC-002) y `axiom.config/agents-catalog.yaml` (TC-011) más el `axiom.skills.lock` raíz (GC-001/GC-002/GC-007) en un proyecto recién adoptado/configurado — antes ningún flujo de scaffolding los producía. Fuente: `apps/cli/src/commands/workspace-config-scaffold.ts`, `workspace-catalog-scaffold.ts`.

### Bloques de cada YAML relevante (documentados)

- **`capabilities.yaml`**: `capabilities.required/.optional/.postMvpOptional`, `supportLevels`, `degradationPolicy`. Cada capability: `id`, `domain` (`sdd`|`spec`|`code`|`memory`), `name`, `version`, `compliance`, `requiredTools`, `optionalTools`, `fallbacks`, `deprecated`, `schemaRef`.
- **`providers.yaml`**: registry de providers + perfiles de discovery (`filesystem-first`, `gateway-first`, `local-only`) con `discoveryOrder`, `preferredProviders`, `optionalProviders`, `gatewayExpectation`.
- **`profiles.yaml`**: `profileBindings` (profile funcional → overlay por defecto, discovery provider profile, `allowedTargets`).
- **`command-protocol.yaml`**: `explicitCommands`, `runtimeCommands` (nombre, dónde corre, qué lee/escribe, si exige binding explícito de proyecto), `intentCommands`, `safety` (confirmaciones y bloqueos).
- **`policy-as-code.yaml`**: `sensitivityTags`, `artifactLifecycle`, `tools`/`compliance`, `projectIsolation`, `doctorValidation`.
- **`telemetry-sinks.yaml`**: `dataSensitivityBoundaries` por overlay, `sinks` (`null-sink`, `log-sink`, `remote-sink`, `audit-trail-sink`).
- **`onboarding.yaml`**: preguntas/defaults/docs generados de `init`, qué lee/escribe `join`, requisitos de `doctor`/`configure`/`start`, `generatedDocs`, `repairPlaybooks`.
- **`scaffolding-contract.yaml`**: contrato de qué se siembra en `init` vs `configure`.

## Ficheros generados por comando

> Tabla actualizada 2026-07-29: el prefijo oculto antiguo se sustituyó por `.axiom-state` en todas las filas (renombre verificado). Además, `init` **ya no** escribe `topology.yaml` para el layout `installed-multi-repo` (`INC-20260703-config-dedup`, cerrado): `topology.yaml` pasó a ser opt-in / derivado-en-lectura — `@axiom/topology#loadTopology` deriva un manifest de fallback a partir de `axiom.yaml` cuando el fichero está ausente (`tryLoadTopologyHint` + `defaultInstalledMultiRepoManifest`), y `topology.yaml` solo se materializa de forma perezosa cuando el proyecto corre `axiom roles assign`. Fuente: comentario junto a la escritura de `init.json` en `apps/cli/src/commands/init.ts`.

| Comando | Escribe |
|---|---|
| `init` | `axiom.yaml`, `.gitignore`, `.axiom-state/local/`, `.axiom-state/<projectName>/`, `init.json` (sin `topology.yaml`) |
| `join` | `.axiom-state/<projectName>/members.yaml` |
| `configure` | `.axiom-state/<projectName>/install-profile.json` (+ surfaces del target) |
| `sync` | `.axiom-state/local/last-sync.json` (+ outputs del adapter) |
| `start` | `.axiom-state/<projectName>/last-start.json` |
| `upgrade` | `.axiom-state/config/<projectName>/managed-state.json`, checkpoints |
| `model set/unset/reset` | `.axiom-state/config/<projectName>/model-assignments.json` (+ `.opencode/model-routing.json`) |
| `components install/uninstall` | `.axiom-state/config/<projectName>/components-state.json` |
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
