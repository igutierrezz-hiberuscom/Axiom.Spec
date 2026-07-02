# Inventario de packages

Fuente: `Axiom/README.md`, README de cada package cuando existe, estructura de `src/` cuando no. Fecha de relevamiento: 2026-07-02.

## Capa dominio/core

| Package | Responsabilidad | Exports/tipos clave | Tests | Notas |
|---|---|---|---|---|
| `@axiom/core` | Branded IDs, `Result<T,E>` sin excepciones, constantes de path compartidas | `asProjectId`, `asSkillId`, `asCapabilityId`, `ok`, `err`, `Result`, `LOCAL_OVERLAY_DIRNAME` | Sí (`packages/core/tests/`) | README disponible |
| `@axiom/capability-model` | Modelo de capabilities/providers/discovery/compliance | `CapabilityDefinition`, `ProviderDefinition`, `CapabilityDomain`, `ComplianceClass`, `CANONICAL_CAPABILITY_IDS` | No documentados en README | Depende de filesystem-truth, config-validation, isolation, project-resolution |
| `@axiom/config-validation` | Validación Zod de YAML (`axiom.yaml`, `integrations.yaml`, `policy.yaml`, `capability-model.yaml`, `provider-registry.yaml`, `install-profiles.yaml`) | `validateWithSchema`, `validateAxiomYamlContent`, `AxiomYamlSchema` | No documentados | Sin I/O propio |

## Capa descubrimiento/aislamiento

| Package | Responsabilidad | Exports/tipos clave | Tests | Notas |
|---|---|---|---|---|
| `@axiom/filesystem-truth` | Descubrimiento read-only del árbol Axiom (`axiom.yaml`, `.sdd/`), path validation | `discoverAxiomRoot`, `readFileContent`, `buildScopeInfo`, `getLocalOverlayPath` | No documentados | Sin escritura |
| `@axiom/project-resolution` | Resolución de proyecto único/ambiguo | `resolveProject`, `assertUnambiguous`, `ProjectResolution`, `ProjectStatus`, `ProjectMode` | No documentados | Package minimal |
| `@axiom/isolation` | Contexto de aislamiento por proyecto, path-guard, MCP permitidos (28 por defecto) | `buildProjectScopedPaths`, `checkMcpAllowed`, `assertProjectIsolation`, `DEFAULT_ALLOWED_MCP_SERVERS` | No documentados | README disponible (no citado en detalle) |

## Capa persistencia

| Package | Responsabilidad | Exports/tipos clave | Tests | Notas |
|---|---|---|---|---|
| `@axiom/persistence` | `FilesystemStore` (read/write/delete/list), escritura atómica, aislamiento cross-project | `createFileSystemStore` | Sí (5 escenarios / 21 sub-tests) | README disponible |
| `@axiom/memory` | Curación de memoria por `MemoryKind`, recall con ranking | `MemoryBackend`, `MemoryEntry`, `MemoryKind`, `recall`, `store` | No documentados | GATE 0024: memoria ≠ fuente de verdad |
| `@axiom/versioning` | State machine de upgrade, checkpoints, migraciones, managed state | `loadManagedState`, `createCheckpoint`, `restoreCheckpoint`, `executeUpgrade` | No documentados | — |

## Capa instalación

| Package | Responsabilidad | Exports/tipos clave | Tests | Notas |
|---|---|---|---|---|
| `@axiom/install-profiles` | Compositor puro del profile triple → `ResolvedInstallProfile` | `resolveInstallProfile`, `loadProfilesData`, `FUNCTIONAL_PROFILES`, `OPERATIONAL_OVERLAYS` | No documentados | README disponible (spec 0018-A4) |
| `@axiom/installer` | Materializa el perfil resuelto, persiste `install-profile.json` | `installProfile`, `GENERATED_FILES_BY_TARGET`, `EXTERNAL_DEPS_BY_CAPABILITY` | Sí | README disponible (spec 0018-A4) |

## Capa adapters (6 sub-packages de `packages/adapters/`)

| Package | Target | Nivel de soporte | Notas |
|---|---|---|---|
| `@axiom/adapters-opencode` | `opencode` | `multi-mode` | Agents-md + skills-lock + routing-snapshot |
| `@axiom/adapters-claude-code` | `claude-code` | `single-mode` | Operativo 0031/A |
| `@axiom/adapters-github-copilot` | `github-copilot` | `fallback-only` | Operativo 0031/B |
| `@axiom/adapters-vscode` | `vscode` | `fallback-only` | Operativo 0031/C |
| `@axiom/adapters-cursor` | `cursor` | `fallback-only` | Operativo 0031/D.1; integración profunda P1 pendiente |
| `@axiom/adapters-litellm` | `litellm` | `fallback-only` | Operativo 0031/D.2; router P1 pendiente |

Contrato común: `generate<Target>Config(args) → Promise<Result<GeneratorResult, AdapterGeneratorError>>`.

## Capa tooling/manifests

| Package | Responsabilidad | Exports/tipos clave | Notas |
|---|---|---|---|
| `@axiom/toolchain` | Manifest `toolchain.yaml`, detección, repair | `ToolchainManifest`, `detectAllTools`, `repairToolchain` | Spec 0023/0027 |
| `@axiom/topology` | Manifest `topology.yaml` (multi-repo, roles, QA lanes) | `TopologyManifest`, `RepoRef`, `RoleAssignment` | Spec 0021/0022 |
| `@axiom/workflow` | State machine SDD, hooks, branch naming | `WorkflowState`, `applyTransition`, `createHookEngine` | Spec 0022 |
| `@axiom/model-routing` | Routing de modelo por slot, assignments, projection a opencode | `ModelRoutingPolicy`, `resolveSlot`, `SUPPORT_MATRIX` | Ver `../architecture/04-adapters-y-model-routing.md` |
| `@axiom/tool-routing` | Dispatcher de `ToolCall`, fallback chain, telemetría | `routeTool`, `resolveToolDispatch` | Spec 0008 (ADR-0008/0013/0020) |

## Capa catálogos (materialización idempotente)

| Package | Responsabilidad | Exports/tipos clave | Tests | Notas |
|---|---|---|---|---|
| `@axiom/skills` | `skills-catalog.yaml` → `.opencode/skills/<id>/SKILL.md`, drift, apply | `materializeSkillSet`, `refreshSkillRegistry`, `applySkillSet` | Sí | Spec 0032 (D3/A). Byte-exact idempotente |
| `@axiom/components` | Catálogo de components, install/uninstall con checkpoint | `installComponent`, `uninstallComponent`, `restoreComponentsState` | No documentados | Spec 0019 (D1/D2) |
| `@axiom/agents` | `agents-catalog.yaml` → `.opencode/agents/<id>/AGENT.md`, verificación SHA-256 | `materializeAgentSet`, `AGENT_MANIFEST_FILENAME` | Sí (4+5+extra escenarios) | README disponible (spec 0033) |

## Capa operación

| Package | Responsabilidad | Exports/tipos clave | Tests | Notas |
|---|---|---|---|---|
| `@axiom/doctor` | Suite de health checks, GATEs 0010/0031/0011 | `runAllChecks`, `generateReport` | No documentados | README disponible |
| `@axiom/orchestrator` | State machine 7 lifecycle + 15 intent commands, gates, rollback | `gateFor`, `runCommand`, `CommandContext` | Sí (smoke E2E por comando) | README disponible |
| `@axiom/cli-commands` | Barrel de re-export de `runX` desde `apps/cli/src/commands/*` | `runConfigure`, `runSync`, `runModel`, etc. | No documentados | README disponible; re-export trivial |
| `@axiom/tui` | Router puro (menú), screens, previews, post-run summaries | `initialState`, `reduce`, `buildPostRunSummary` | No documentados | Spec 0019 (B1-B3) |
| `apps/cli` | Entry point, 36 ficheros de comando (~16.400 líneas) | `axiom <comando>` | Sí (31 test files, 201 tests) | README disponible |

## Capa documentación/disciplina

| Package | Responsabilidad | Exports/tipos clave | Notas |
|---|---|---|---|
| `@axiom/document-bootstrap` | Writer de Copilot instructions, idempotente, preserva `TEAM:CUSTOM` | `writeCopilotInstructions`, `classifyAndPreserve` | Spec 0018-B4 |
| `@axiom/cavekit-discipline` | Invariantes puros, backprop de fallos, drift check, gate GGA opcional | `evaluateInvariant`, `backpropFromFailure`, `applyGgaGate` | Spec 0015 |
| `@axiom/user-workspace` | Registry de proyectos user-level, self-update | `loadRegistry`, `addProject`, `loadInstallManifest` | Spec 0020 (A1/B1) |

## Patrón arquitectónico dominante (observado en todos los packages)

`Result<T,E>` sin excepciones; escritura atómica (tmp+rename); configuración YAML declarativa; interfaces duck-typed (memory backend, hook engine); GATEs explícitas numeradas; diseño spec-first (60+ incrementos referenciados en el código).

## Cobertura de tests (agregada, según README/estructura)

E2E smoke en `apps/cli` (31 files, 201 tests); tests por escenario en `orchestrator`, `persistence`, `agents`, `skills`; tests de validación en `config-validation`. La mayoría de los 28 packages **no documentan** su conteo de tests en README — no asumir cobertura sin verificarlo en el repo real antes de decisiones críticas.
