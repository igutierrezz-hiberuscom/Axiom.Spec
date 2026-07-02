# 03 Modelo Operativo y Datos

> Esta sección documenta el modelo de datos REAL implementado hoy en `Axiom/`. Para el modelo de topología futuro (registro `~/.axiom/projects.yml`, `axiom.yml` por repo con roles `sdd`/`spec`/`code`), ver la sección "Roadmap futuro" al final de este documento — ese modelo no está implementado.

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

## Roadmap futuro: topología de repos (no implementado)

`specs/increments/INC-20260702-axiom-redesign-roadmap/` describe un modelo posterior, todavía no implementado, que sí distinguiría tres repos por rol dentro de CADA proyecto gestionado por Axiom:

1. repo `sdd` (método/factory);
2. repo `spec` (conocimiento canónico);
3. repo `code` (runtime instalable).

Con datos operativos esperados: catálogo de repos del proyecto, aliases canónicos, ubicación por workspace local, resolución alternativa por adapter/MCP/surface externa, y un registro global fuera del proyecto (`~/.axiom/config.yml`, `~/.axiom/projects.yml`). Este modelo reemplazaría/generalizaría el actual `axiom.yaml` único, pero **la migración no ha comenzado** (INC-01 del roadmap sigue bloqueado por decisiones abiertas: stack de implementación y si `Axiom.SDD`/`Axiom.Spec` deben reestructurarse). No tratar esta sección como estado actual.