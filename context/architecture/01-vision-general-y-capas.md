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

## Capas por responsabilidad (28 packages + `apps/cli`)

| Capa | Packages | Patrón dominante |
|---|---|---|
| Dominio/Core | `core`, `capability-model`, `config-validation` | Tipos puros + `Result<T,E>` + validación Zod |
| Descubrimiento/aislamiento | `filesystem-truth`, `project-resolution`, `isolation` | Filesystem read-only, path-guard, resolución de proyecto único/ambiguo |
| Persistencia | `persistence`, `memory`, `versioning` | Store atómico, curación de memoria, checkpoints/migraciones |
| Instalación | `install-profiles`, `installer` | Composición pura del profile triple → materialización |
| Adapters | `adapters/{opencode,claude-code,github-copilot,vscode,cursor,litellm}` | Generator pattern, contrato común |
| Tooling/manifests | `toolchain`, `topology`, `workflow`, `model-routing`, `tool-routing` | Manifests YAML + state machines + dispatcher |
| Catálogos | `skills`, `components`, `agents` | Materialización idempotente (tmp+rename) |
| Operación | `doctor`, `orchestrator`, `cli-commands`, `tui`, `apps/cli` | State machines, hooks, telemetría, 36 comandos |
| Documentación/disciplina | `document-bootstrap`, `cavekit-discipline`, `user-workspace` | Bootstrap de surfaces, invariantes nativos, registry user-level |

Detalle package por package: [../references/01-inventario-de-packages.md](../references/01-inventario-de-packages.md).

## Dependencias críticas (verificadas por imports/estructura)

- `@axiom/core` es la base: branded IDs (`asProjectId`, `asSkillId`, `asCapabilityId`) y `Result<T,E>`.
- `@axiom/filesystem-truth` es la "fuente de verdad" read-only sobre el árbol Axiom (detecta `axiom.yaml`, `.sdd/`).
- `@axiom/isolation` aplica path-guard y scope por `projectId` sobre cualquier operación.
- `@axiom/capability-model` modela capabilities/providers/discovery, consumido por `configure`, `start`, `doctor`.
- `@axiom/install-profiles` es el compositor puro del profile triple.
- `@axiom/adapters-*` son la salida final hacia cada IDE/CLI externo.

## Qué NO es Axiom hoy (límites verificados)

- No hay separación física de repos "sdd/spec/code" por proyecto adoptante — eso es roadmap futuro (ver `specs/03_Modelo_Operativo_y_Datos.md`, sección "Roadmap futuro").
- No hay un servidor MCP genérico propio de Axiom; el MCP se gestiona hoy como entrada de catálogo de `@axiom/toolchain` y comandos `axiom mcp`.
- No hay runtime persistente de larga vida para `start` (documentado explícitamente como fuera del MVP en `overview.md`).
