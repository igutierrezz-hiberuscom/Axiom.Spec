# Modelo de datos y configuración

Fuente: `Axiom/docs/configuration/**`, `Axiom/docs/generated-files.md`, `Axiom/docs/cli/*.md`.

## `axiom.yaml` — manifiesto raíz por proyecto adoptante

Generado por `axiom init`. Campos relevantes (`Axiom/docs/configuration/project-structure.md`): `project.name`, `project.status`, `project.product_implementation_status`, `project.mode`, `scopes`, `rules`, `artifact_id_policy`, `lifecycle_commands`, `initial_capabilities`.

Encapsula el **profile triple**:

- `functionalProfile`: `builder` (más cubierto en runtime) | `product-owner`.
- `operationalOverlay`: `local-only` (fuerza filesystem, sin gateway) | `standard` (filesystem por defecto, gateway opt-in) | `enterprise` (exige gateway).
- `adapterTarget`: uno de 8 targets declarados (ver `../architecture/04-adapters-y-model-routing.md`).

Triple recomendado para primer proyecto: `builder` + `local-only` + `opencode` (`Axiom/docs/first-project-readiness.md`).

Editable con criterio a mano; no editar `init.json`, `install-profile.json`, `last-start.json`, `last-sync.json` salvo diagnóstico (son estado derivado).

## `.sdd/` — estado project-scoped

- `.sdd/local/`: overlay NO versionada. Overrides locales, markers (`last-sync.json`). Regida por `local-overlay-policy.yaml`.
- `.sdd/<projectName>/`: `init.json`, `members.yaml`, `install-profile.json`, `last-start.json`.
- `.sdd/config/<projectName>/`: `managed-state.json`, `model-assignments.json`, `components-state.json`, `gateway-state.json`.
- `.sdd/<projectName>/checkpoints/<id>/`: snapshots pre-mutación, últimos 5 conservados.

## `axiom.spec/config/*.yaml` — catálogo declarativo esperado dentro del proyecto adoptante

Nombre en minúsculas, **no confundir con `Axiom.Spec/`** (este repo de workspace). Según `Axiom/docs/configuration/README.md`, el runtime espera ~20 YAML:

`axiom.workspace.yaml`, `branch-policy.yaml`, `capabilities.yaml`, `clarification-policy.yaml`, `command-protocol.yaml`, `external-work-items.yaml`, `id-policy.yaml`, `integrations.yaml`, `lifecycle-policy.yaml`, `local-overlay-policy.yaml`, `model-routing-policy.yaml`, `onboarding.yaml`, `orchestration-policy.yaml`, `policy-as-code.yaml`, `profiles.yaml`, `providers.yaml`, `repositories.yaml`, `scaffolding-contract.yaml`, `skills.yaml`, `telemetry-sinks.yaml`, `tool-routing-policy.yaml`.

No todos se consumen con el mismo nivel de profundidad hoy en runtime. Esta carpeta no existe en la raíz del propio repo `Axiom/` (ver `../references/03-riesgos-y-brechas-conocidas.md`).

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

| Comando | Escribe |
|---|---|
| `init` | `axiom.yaml`, `.gitignore`, `.sdd/local/`, `.sdd/<projectName>/`, `init.json` |
| `join` | `.sdd/<projectName>/members.yaml` |
| `configure` | `.sdd/<projectName>/install-profile.json` (+ surfaces del target) |
| `sync` | `.sdd/local/last-sync.json` (+ outputs del adapter) |
| `start` | `.sdd/<projectName>/last-start.json` |
| `upgrade` | `.sdd/config/<projectName>/managed-state.json`, checkpoints |
| `model set/unset/reset` | `.sdd/config/<projectName>/model-assignments.json` (+ `.opencode/model-routing.json`) |
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

Caso especial Copilot: `configure` puede escribir `.github/copilot-instructions.md` vía `@axiom/document-bootstrap`, usando template versionado en `axiom.spec/templates/`, resolviendo variables del proyecto y preservando el bloque `TEAM:CUSTOM` con escritura atómica.

## Instalación user-level (fuera del proyecto)

El binario `axiom` se instala una sola vez por operador (no por proyecto), vía `scripts/install-global.mjs`:
- macOS/Linux: `npm prefix -g` o `$HOME/.local/bin` (`npm link`).
- Windows: `%USERPROFILE%\.local\bin\axiom.cmd` (shim, con rollback si el smoke post-install falla).

Manifest de versión user-level: `~/.axiom/install.json` (`@axiom/user-workspace`, `UserWorkspacePaths.installPath`). `axiom self-update` gestiona esta versión, separada del `axiom upgrade` project-scoped; requiere `--apply` explícito para mutar (preview-only por defecto).
