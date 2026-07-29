# Visión general y capas del runtime Axiom

Fuente principal: `Axiom/README.md`, `Axiom/docs/overview.md`, estructura real de `Axiom/packages/`.

## Qué es Axiom en runtime

Axiom es un CLI Node/TypeScript (`axiom`) que coordina, para un proyecto adoptante concreto: estructura inicial, configuración declarativa, resolución de profiles, materialización de archivos para IDEs/CLIs externos, validaciones y checks de salud. Modelo mental (`Axiom/docs/overview.md`):

- la spec define qué reglas y contratos existen;
- la configuración por proyecto (YAML) activa esos contratos;
- el CLI materializa, valida y opera sobre ese contrato.

## Tres espacios dentro del repo `Axiom/` (según overview.md)

1. **Builder tooling** (`_builder/`): herramientas de construcción, no parte del runtime — no presente en este checkout (ver riesgos conocidos).
2. **Especificación del producto** (`axiom.spec/`, minúsculas): fuente de verdad documental interna al producto — no presente en este checkout.
3. **Runtime del producto** (`apps/` + `packages/`): implementación ejecutable real, sí presente y operativa.

## Capas por responsabilidad (43 packages + `apps/cli`)

> Conteo verificado 2026-07-29: `ls packages/*/package.json | wc -l` → **34** top-level (`packages/adapters/` en sí no cuenta, no tiene `package.json` propio) + `ls packages/adapters/*/package.json | wc -l` → **9** adapters = **43** total. El baseline 2026-07-02 documentaba 28; nuevos desde entonces: `launcher`, `mcp-server`, `mcp-tools`, `providers`, `technical-context`, `telemetry`, `tracker`, `tracker-ado`.

| Capa | Packages | Patrón dominante |
|---|---|---|
| Dominio/Core | `core`, `capability-model`, `config-validation` | Tipos puros + `Result<T,E>` + validación Zod |
| Descubrimiento/aislamiento | `filesystem-truth`, `project-resolution`, `isolation` | Filesystem read-only, path-guard, resolución de proyecto único/ambiguo |
| Persistencia | `persistence`, `memory`, `versioning` | Store atómico, curación de memoria, checkpoints/migraciones |
| Instalación | `install-profiles`, `installer` | Composición pura del profile triple → materialización |
| Adapters | `adapters/{opencode,claude-code,github-copilot,vscode,cursor,litellm,codex,antigravity,visual-studio-2026}` | Generator pattern, contrato común (9 packages, 10 targets canónicos — ver `04-adapters-y-model-routing.md`) |
| MCP | `mcp-server`, `mcp-tools`, `providers` | Dispatcher JSON-RPC hand-rolled + handlers de capability + registry/discovery de providers (nuevo desde el baseline) |
| Tooling/manifests | `toolchain`, `topology`, `workflow`, `model-routing`, `tool-routing` | Manifests YAML + state machines + dispatcher |
| Catálogos | `skills`, `components`, `agents` | Materialización idempotente (tmp+rename) |
| Operación | `doctor`, `orchestrator`, `cli-commands`, `tui`, `launcher`, `apps/cli` | State machines, hooks, telemetría, ~81 ficheros de comando; `launcher` es el front web operador por defecto (nuevo) |
| Contexto/telemetría/tracking | `technical-context`, `telemetry`, `tracker`, `tracker-ado` | Índice de contexto técnico servible por MCP, bus/sinks de telemetría, puertos de tracker + adapter Azure DevOps (todos nuevos desde el baseline) |
| Documentación/disciplina | `document-bootstrap`, `cavekit-discipline`, `user-workspace` | Bootstrap de surfaces, invariantes nativos, registry user-level |

Detalle package por package: [../references/01-inventario-de-packages.md](../references/01-inventario-de-packages.md).

## Dependencias críticas (verificadas por imports/estructura)

- `@axiom/core` es la base: branded IDs (`asProjectId`, `asSkillId`, `asCapabilityId`) y `Result<T,E>`. También re-exporta las constantes canónicas de nombre de carpeta: `LOCAL_OVERLAY_DIRNAME = '.axiom-state'`, `AXIOM_CONFIG_DIRNAME = 'axiom.config'` (fuente: `packages/filesystem-truth/src/discovery.ts`).
- `@axiom/filesystem-truth` es la "fuente de verdad" read-only sobre el árbol Axiom (detecta `axiom.yaml`, `.axiom-state/`).
- `@axiom/isolation` aplica path-guard y scope por `projectId` sobre cualquier operación.
- `@axiom/capability-model` modela capabilities/providers/discovery, consumido por `configure`, `start`, `doctor`.
- `@axiom/install-profiles` es el compositor puro del profile triple.
- `@axiom/adapters-*` (9 packages) son la salida final hacia cada IDE/CLI externo.
- `@axiom/mcp-server` + `@axiom/mcp-tools` son el servidor MCP propio (nuevo desde el baseline): dispatcher JSON-RPC 2.0 hand-rolled sobre los handlers de `@axiom/mcp-tools`, proyectado a config nativa por `apps/cli/src/commands/native-mcp-config.ts`.

## Qué NO es Axiom hoy (límites verificados)

- No hay separación física de repos "sdd/spec/code" por proyecto adoptante como default obligatorio — el roadmap de rediseño que exploraba esto (`INC-20260702-axiom-redesign-roadmap`) ya cerró y está archivado; su resultado estable vive en `specs/00_Resumen_Ejecutivo.md`-`specs/08_Glosario.md`, no en este documento.
- Axiom **ya no carece** de servidor MCP propio (afirmación stale del baseline 2026-07-02): existen `@axiom/mcp-server` (dispatcher) + `@axiom/mcp-tools` (handlers), con dos servers gestionados (`sdd-mcp-server`, `spec-mcp-broker`) proyectados a la config nativa de cada tool. Ver `../integrations/01-capabilities-providers-y-toolchain.md`.
- No hay runtime persistente de larga vida para `start` (documentado explícitamente como fuera del MVP en `overview.md`).
