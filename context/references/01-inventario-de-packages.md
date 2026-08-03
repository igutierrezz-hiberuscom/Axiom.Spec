# Inventario de packages

Fuente: `Axiom/README.md`, README de cada package cuando existe, estructura de `src/` cuando no. Fecha de relevamiento inicial: 2026-07-02; reconciliado 2026-07-29 (conteos y packages nuevos verificados por `ls`/grep directo, ver nota de conteo abajo).

> **Conteo verificado 2026-07-29**: `ls packages/*/package.json | wc -l` → **34** top-level (`packages/adapters/` es solo un directorio contenedor, sin `package.json` propio, no cuenta como package) + `ls packages/adapters/*/package.json | wc -l` → **9** adapters = **43 packages totales**. El baseline documentaba 28. Packages nuevos verificados por `ls packages/`: `launcher`, `mcp-server`, `mcp-tools`, `providers`, `technical-context`, `telemetry`, `tracker`, `tracker-ado` (8 nuevos — la nota original del incremento mencionaba 7, pero `telemetry` también es nuevo y no estaba en el inventario 2026-07-02).

## Capa dominio/core

| Package | Responsabilidad | Exports/tipos clave | Tests | Notas |
|---|---|---|---|---|
| `@axiom/core` | Branded IDs, `Result<T,E>` sin excepciones, constantes de path compartidas, **errores tipados** | `asProjectId`, `asSkillId`, `asCapabilityId`, `ok`, `err`, `Result`, `LOCAL_OVERLAY_DIRNAME`, `AxiomError`, `AXIOM_ERROR_CODES`, `AxiomErrorCode` | Sí (`packages/core/tests/`, incl. `error.test.ts`) | README disponible |
| `@axiom/capability-model` | Modelo de capabilities/providers/discovery/compliance | `CapabilityDefinition`, `ProviderDefinition`, `CapabilityDomain`, `ComplianceClass`, `CANONICAL_CAPABILITY_IDS` | No documentados en README | Depende de filesystem-truth, config-validation, isolation, project-resolution |
| `@axiom/config-validation` | Validación Zod de YAML (`axiom.yaml`, `integrations.yaml`, `policy.yaml`, `capability-model.yaml`, `provider-registry.yaml`, `install-profiles.yaml`) | `validateWithSchema`, `validateAxiomYamlContent`, `AxiomYamlSchema`, `validateInstallProfilesYamlContent` | Sí (`packages/config-validation/tests/`) | Sin I/O propio. **Consumido por `apps/cli` desde `INC-20260730-exact-scope`**: los 3 loaders de `profiles.yaml` (`roles.ts`, `init.ts`, `topology.ts`) usan `validateInstallProfilesYamlContent`; antes el `InstallProfilesYamlSchema` no tenía consumidores |

## Capa descubrimiento/aislamiento

| Package | Responsabilidad | Exports/tipos clave | Tests | Notas |
|---|---|---|---|---|
| `@axiom/filesystem-truth` | Descubrimiento read-only del árbol Axiom (`axiom.yaml`, `.axiom-state/`), path validation; exporta también `LOCAL_OVERLAY_DIRNAME`/`AXIOM_CONFIG_DIRNAME` | `discoverAxiomRoot`, `readFileContent`, `buildScopeInfo`, `getLocalOverlayPath` | No documentados | Sin escritura |
| `@axiom/project-resolution` | Resolución de proyecto único/ambiguo | `resolveProject`, `assertUnambiguous`, `ProjectResolution`, `ProjectStatus`, `ProjectMode` | No documentados | Package minimal |
| `@axiom/isolation` | Contexto de aislamiento por proyecto, path-guard, MCP permitidos (**3 por defecto**, verificado en `src/p0.ts`: `sdd`, `spec`, `serena` — corrige la cifra "28" del baseline, que era incorrecta) | `buildProjectScopedPaths`, `checkMcpAllowed`, `assertProjectIsolation`, `DEFAULT_ALLOWED_MCP_SERVERS` | No documentados | README disponible (no citado en detalle) |

## Capa persistencia

| Package | Responsabilidad | Exports/tipos clave | Tests | Notas |
|---|---|---|---|---|
| `@axiom/persistence` | `FilesystemStore` (read/write/delete/list), escritura atómica, aislamiento cross-project | `createFileSystemStore` | Sí (5 escenarios / 21 sub-tests) | README disponible |
| `@axiom/memory` | Curación de memoria por `MemoryKind`, recall con ranking, **gate de evidencia fail-closed** | `MemoryBackend`, `MemoryEntry`, `MemoryKind`, `recall`, `store`, `validateMemoryEvidence`, `MIN_EVIDENCE_LENGTH` | Sí (`packages/memory/tests/`, incl. `evidence.test.ts`) | GATE 0024: memoria ≠ fuente de verdad. Desde `INC-20260730-engram-evidence`, `save` exige `rationale`/`source` (trim > 3) en runtime desde los 3 puntos de entrada |
| `@axiom/versioning` | State machine de upgrade, checkpoints, migraciones, managed state | `loadManagedState`, `createCheckpoint`, `restoreCheckpoint`, `executeUpgrade` | No documentados | — |

## Capa instalación

| Package | Responsabilidad | Exports/tipos clave | Tests | Notas |
|---|---|---|---|---|
| `@axiom/install-profiles` | Compositor puro del profile triple → `ResolvedInstallProfile` | `resolveInstallProfile`, `loadProfilesData`, `FUNCTIONAL_PROFILES`, `OPERATIONAL_OVERLAYS` | No documentados | README disponible (spec 0018-A4) |
| `@axiom/installer` | Materializa el perfil resuelto, persiste `install-profile.json` | `installProfile`, `GENERATED_FILES_BY_TARGET`, `EXTERNAL_DEPS_BY_CAPABILITY` | Sí | README disponible (spec 0018-A4) |

## Capa adapters (9 sub-packages de `packages/adapters/`, verificado 2026-07-29)

> Baseline documentaba 6; `codex`, `antigravity`, `visual-studio-2026` se añadieron en `INC-20260726-adapter-mcp-parity`. Detalle completo y vigente en `../architecture/04-adapters-y-model-routing.md` (9 packages / 10 targets canónicos / 8 headline).

| Package | Target | Nivel de soporte | Notas |
|---|---|---|---|
| `@axiom/adapters-opencode` | `opencode` | `multi-mode` | Agents-md + skills-lock + routing-snapshot |
| `@axiom/adapters-claude-code` | `claude-code` | `single-mode` | Operativo 0031/A |
| `@axiom/adapters-github-copilot` | `github-copilot` | `fallback-only` | Operativo 0031/B |
| `@axiom/adapters-vscode` | `vscode` | `fallback-only` | Operativo 0031/C |
| `@axiom/adapters-cursor` | `cursor` | `fallback-only` | Operativo 0031/D.1; integración profunda P1 pendiente |
| `@axiom/adapters-litellm` | `litellm` | `fallback-only` | Operativo 0031/D.2; router P1 pendiente |
| `@axiom/adapters-codex` | `codex` | `fallback-only` | `.codex/AGENTS.md`; generador de primera clase desde `INC-20260726-adapter-mcp-parity` |
| `@axiom/adapters-antigravity` | `antigravity` | `fallback-only` | `.antigravity/AGENTS.md`; ídem |
| `@axiom/adapters-visual-studio-2026` | `visual-studio-2026` | `fallback-only` | `.vs/AXIOM.md`; ídem |

Contrato común: `generate<Target>Config(args) → Promise<Result<GeneratorResult, AdapterGeneratorError>>`.

## Capa tooling/manifests

| Package | Responsabilidad | Exports/tipos clave | Notas |
|---|---|---|---|
| `@axiom/toolchain` | Manifest `toolchain.yaml`, detección, repair | `ToolchainManifest`, `detectAllTools`, `repairToolchain` | Spec 0023/0027; catálogo real reconciliado en ADR-0031 (`cmm` reemplaza codegraph/graphify) |
| `@axiom/topology` | Manifest `axiom.config/topology.yaml` (multi-repo, roles, QA lanes); ahora opt-in/derivado-en-lectura | `TopologyManifest`, `RepoRef`, `RoleAssignment`, `loadTopology` | Spec 0021/0022; `INC-20260703-config-dedup` (cerrado) — `init` ya no lo escribe, `loadTopology` deriva fallback desde `axiom.yaml` |
| `@axiom/workflow` | State machine SDD, hooks, branch naming, **receipts de fase** | `WorkflowState`, `applyTransition`, `createHookEngine`, `writePhaseReceipt`, `PhaseReceipt` | Spec 0022 |
| `@axiom/model-routing` | Routing de modelo por slot, assignments, projection a opencode, checks de drift MRC-001..004 | `ModelRoutingPolicy`, `resolveSlot`, `SUPPORT_MATRIX` | Ver `../architecture/04-adapters-y-model-routing.md`. Sus checks corren vía `axiom model validate`, NO como parte de `@axiom/doctor` |
| `@axiom/tool-routing` | Dispatcher de `ToolCall`, fallback chain, telemetría | `routeTool`, `resolveToolDispatch` | Spec 0008 (ADR-0008/0013/0020) |

## Capa MCP (nueva desde el baseline 2026-07-02)

| Package | Responsabilidad | Exports/tipos clave | Notas |
|---|---|---|---|
| `@axiom/mcp-server` | Dispatcher JSON-RPC 2.0 hand-rolled (sin SDK externo) + transporte stdio | `createMcpServer`, `runStdioServer`, `toolDescriptorsForKind` | `INC-20260708-mcp-runnable-server`; kinds `sdd`/`spec`/`memory`/`axiom` |
| `@axiom/mcp-tools` | Handlers de capability agrupados por dominio (`sdd.*`, `spec.*`, `memory.*`, `axiom.*`) | `MCP_TOOL_HANDLERS`, tipos de registry | Consumido por `@axiom/mcp-server` y por `native-mcp-config.ts` |
| `@axiom/providers` | Registry/discovery de providers, cliente MCP stdio, code-intel | `createStdioMcpClient`, registry de providers | Usado por `axiom doctor --deep` (TC-019) |

## Capa catálogos (materialización idempotente)

| Package | Responsabilidad | Exports/tipos clave | Tests | Notas |
|---|---|---|---|---|
| `@axiom/skills` | `skills-catalog.yaml` → `.opencode/skills/<id>/SKILL.md`, drift, apply | `materializeSkillSet`, `refreshSkillRegistry`, `applySkillSet` | Sí | Spec 0032 (D3/A). Byte-exact idempotente |
| `@axiom/components` | Catálogo de components, install/uninstall con checkpoint | `installComponent`, `uninstallComponent`, `restoreComponentsState` | No documentados | Spec 0019 (D1/D2) |
| `@axiom/agents` | `agents-catalog.yaml` → `.opencode/agents/<id>/AGENT.md`, verificación SHA-256 | `materializeAgentSet`, `AGENT_MANIFEST_FILENAME` | Sí (4+5+extra escenarios) | README disponible (spec 0033). **No confundir** con la superficie portable adapter-agnóstica `.axiom/agents/<id>.md` (+ `.axiom/commands/<id>.md`, `.axiom/skills/<id>/SKILL.md`) que escribe SIEMPRE `materializeProcessSurfaces` (`apps/cli/src/commands/workspace-process-surfaces.ts`) — son dos materializaciones distintas y coexistentes: esta fila es la nativa de opencode, la portable es adapter-agnóstica y existe para todo adapter (incl. codex/antigravity/vs2026) |

## Capa operación

| Package | Responsabilidad | Exports/tipos clave | Tests | Notas |
|---|---|---|---|---|
| `@axiom/doctor` | Suite de health checks (~19 categorías con IDs propios; ver `../operations/02-doctor-troubleshooting-y-telemetria.md`), modo opt-in `--deep` | `runDoctorChecks`, `runDoctorChecksDeep`, `formatReport` | No documentados | README disponible; creció mucho desde el baseline (6 familias → ~19) |
| `@axiom/orchestrator` | State machine **8 lifecycle + 19 intent commands** (verificado en README propio, corrige "7+15" del baseline), gates, rollback; solo 7 de los 8 lifecycle pasan realmente por el gate desde `apps/cli` (`doctor-command` es la excepción documentada) | `gateFor`, `runCommand`, `CommandContext` | Sí (smoke E2E por comando) | README disponible |
| `@axiom/cli-commands` | Barrel de re-export de `runX` desde `apps/cli/src/commands/*` | `runConfigure`, `runSync`, `runModel`, etc. | No documentados | README disponible; re-export trivial |
| `@axiom/tui` | Router puro (menú), screens, previews, post-run summaries; incluye desde `INC-20260703-tui-init-wizard` un wizard genérico (`wizard-select`/`wizard-text`) para el flujo de `init` sin proyecto | `initialState`, `reduce`, `buildPostRunSummary` | No documentados | Spec 0019 (B1-B3) + INC-20260703 |
| `@axiom/launcher` | Front web operador (default de `axiom app`), routing de adapters a skill/mcp-tool, prompt-builder | `AXIOM_ADAPTER_ROUTING`, `apiGetLauncherData`, prompt-builder | No documentados | Nuevo desde el baseline; ver `../architecture/04-adapters-y-model-routing.md` |
| `apps/cli` | Entry point, **81 ficheros** de comando (verificado: `ls apps/cli/src/commands/*.ts \| wc -l`; 10 de ellos son helpers `_`-prefixed sin comando propio; **129 ficheros `*.test.ts`** verificados bajo `apps/cli/` — el conteo "31 test files, 201 tests" del baseline no se pudo re-verificar exactamente y se trata como stale/no confiable) | `axiom <comando>` | Sí | README disponible; el propio README interno del package aún cita "31 test files/201 tests", no actualizado en este pase (fuera de alcance: ese README vive en `Axiom/`, no en `Axiom.Spec/`) |

## Capa documentación/disciplina

| Package | Responsabilidad | Exports/tipos clave | Notas |
|---|---|---|---|
| `@axiom/document-bootstrap` | Writer de Copilot instructions, idempotente, preserva `TEAM:CUSTOM` | `writeCopilotInstructions`, `classifyAndPreserve` | Spec 0018-B4 |
| `@axiom/cavekit-discipline` | Invariantes puros, backprop de fallos, drift check, gate GGA opcional | `evaluateInvariant`, `backpropFromFailure`, `applyGgaGate` | Spec 0015 |
| `@axiom/user-workspace` | Registry de proyectos user-level, self-update | `loadRegistry`, `addProject`, `loadInstallManifest` | Spec 0020 (A1/B1) |

## Capa contexto/telemetría/tracking (nueva desde el baseline 2026-07-02)

| Package | Responsabilidad | Notas |
|---|---|---|
| `@axiom/technical-context` | Índice de contexto técnico (generación + lectura) servible por MCP (`spec.technicalContextIndexRead`) | `src/`: `generate-index.ts`, `technical-context-index.ts` |
| `@axiom/telemetry` | Bus de eventos, sinks (`null`/`log`/`remote`/`audit-trail`), verificación de audit trail | `src/`: `bus.ts`, `sink.ts`, `sinks/`, `audit-trail-verify.ts` |
| `@axiom/tracker` | Puertos/tipos genéricos de tracker externo (agnóstico de proveedor) + `null-tracker` | `src/`: `ports.ts`, `types.ts`, `null-tracker.ts` |
| `@axiom/tracker-ado` | Implementación concreta para Azure DevOps (config, git, sprints, builds, adjuntos, normalizadores) | `src/`: `ado-config.ts`, `ado-git.ts`, `ado-sprints.ts`, `ado-build.ts`, `ado-attachments.ts` |

## Patrón arquitectónico dominante (observado en todos los packages)

`Result<T,E>` sin excepciones; escritura atómica (tmp+rename); configuración YAML declarativa; interfaces duck-typed (memory backend, hook engine); GATEs explícitas numeradas; diseño spec-first (60+ incrementos referenciados en el código).

## Cobertura de tests (agregada, según README/estructura)

Cifra agregada del monorepo **verificada 2026-08-02** con `npx vitest run` completo: **336 ficheros de test, 3489 tests, 3483 en verde y 6 en rojo**. Los 6 fallos son preexistentes a la tanda `INC-20260730-*` y están caracterizados: 5 deterministas en `packages/install-profiles/tests/composer.test.ts` (matrices `builder`/`product-owner` con `opencode` y enablement de capabilities) y 1 por **timeout de 5000 ms bajo contención de la suite completa** en `packages/memory/tests/engram-backend.test.ts` (`query()` con filtro de kind), que pasa en verde al ejecutar `packages/memory` en aislamiento. `apps/cli/tests/launcher-panels.test.ts` presenta la misma clase de flake por contención de forma intermitente.

Advertencia metodológica confirmada en esta medición: una ejecución única de la suite **no** es evidencia suficiente — dos corridas del mismo árbol dieron 9 y 6 fallos respectivamente, y una tercera murió con `Segmentation fault` en `npx` devolviendo un exit code engañoso. Contrastar siempre contra varias corridas antes de atribuir un fallo a un cambio.

Tests por escenario en `orchestrator`, `persistence`, `agents`, `skills`; tests de validación en `config-validation`. La mayoría de los 43 packages **no documentan** su conteo de tests en README — no asumir cobertura sin verificarlo en el repo real antes de decisiones críticas.

Nota estructural relevante para cualquier garantía basada en tipos: los `tsconfig.json` de cada package usan `"include": ["src/**/*"]`, por lo que **`tsc -b` no typechequea los ficheros de test** y vitest transpila sin typecheck. Un build verde no prueba que los tests compilen ni que respeten los contratos de tipos.
