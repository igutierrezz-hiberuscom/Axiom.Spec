# Capabilities, providers y toolchain externo

Fuente: `Axiom/docs/configuration/providers-and-capabilities.md`, `Axiom/docs/configuration/files/{capabilities,providers}.md`, informe de incrementos 0015-0030, `Axiom/packages/toolchain/src/probe.ts`, `Axiom/axiom.config/toolchain-catalog.yaml`, `Axiom/docs/0031-adr-cmm-replaces-graphify-and-codegraph.md`, `Axiom/packages/mcp-server/src/`, `Axiom/packages/mcp-tools/src/`, `Axiom/packages/isolation/src/p0.ts`.

> Reconciliado 2026-07-29: la sección de toolchain y la de MCP de este documento describían un estado superado. `codegraph`/`graphify` fueron removidos (ADR-0031, `cmm` los reemplaza); Axiom ya tiene servidor MCP propio (`@axiom/mcp-server` + `@axiom/mcp-tools`), contrario a lo que decía el baseline 2026-07-02.

## Modelo de capabilities

`capabilities.yaml` define el catálogo y el marco de degradación: `capabilities.required` / `.optional` / `.postMvpOptional`, `supportLevels`, `degradationPolicy`. Cada capability declara `id`, `domain` (`sdd`, `spec`, `code`, `memory`), `name`, `version`, `compliance`, `requiredTools`, `optionalTools`, `fallbacks`, `deprecated`, `schemaRef`. Implementado en `@axiom/capability-model`.

## Providers y discovery

`providers.yaml` modela el registry de providers y sus perfiles declarativos de discovery. Discovery modes: `filesystem`, `workspace`, `gateway`. Discovery Provider Profiles (`filesystem-first`, `gateway-first`, `local-only`) definen `discoveryOrder`, `preferredProviders`, `optionalProviders`, `gatewayExpectation`, y modo degradado.

Regla del MVP: **Serena** es el baseline de inteligencia de código; el resto de providers son opcionales o evolutivos según capability y entorno. Reglas globales: Serena baseline, providers opcionales post-MVP, defaults MCP de producto, providers prohibidos por defecto.

Impacta directamente a: `configure` (compone install profile), `start` (resuelve provider efectivo y modo de discovery), `doctor` (evalúa consistencia del modelo y defaults).

## Toolchain externo (`@axiom/toolchain`)

Manifest `toolchain.yaml`: npm/pnpm/yarn/bun, Node, git, python — con detección dinámica (`detect.ts`), loader, validador y repair idempotente.

> **Catálogo reconciliado 2026-07-29 (ADR-0031, `docs/0031-adr-cmm-replaces-graphify-and-codegraph.md`)**: las tools P1 del baseline (`CodeGraph`, `Graphify`, añadidas en el incremento 0027) fueron **removidas y reemplazadas por `cmm`** (`codebase-memory-mcp`). El catálogo real vigente, verificado en `axiom.config/toolchain-catalog.yaml` (self-bootstrap del propio repo `Axiom/`) y en `packages/toolchain/src/probe.ts`, es: **`serena`** (code-intelligence), **`cmm`** (code-intelligence, reemplaza codegraph/graphify), **`engram`** (library-context), **`context7`** (library-context), **`rtk`** (input-optimizer), **`caveman`** (output-optimizer), **`autoskills`** (skill-surface) — 7 tools, todas `mvp: false` (catálogo permitido/opcional, no requerido por defecto; el operador las agrega vía `axiom toolchain add --id <id>`). `cmm` tiene evidencia de fallback específica: un fichero `.cmm/sync-state.json` en el proyecto se trata como prueba de instalación funcionando (`packages/toolchain/src/probe.ts`).

### Versionado reproducible del toolchain (`INC-20260730-toolchain-versioning`)

`axiom.config/toolchain-catalog.yaml` usa `schemaVersion: 2`. Cada entrada puede declarar `versionExtractor`, versiones de política por canal (`stable`, `candidate`, `edge`) y `compatibility.axiomMinVersion`. El catálogo es una allowlist/política: no implica que todas las tools estén habilitadas en `axiom.config/toolchain.yaml` ni que deban instalarse.

El estado de versiones fijadas vive en `.axiom-state/<project>/toolchain.lock`, un YAML `schemaVersion: 1` ignorado junto con `.axiom-state/`. El lock contiene `projectId`, `lockedAt` y `tools`; cada entrada registra `id`, `version`, `channel` y, opcionalmente, `probeCommand`, `probeOutput` y `probedAt`. `loadToolchainLock` devuelve un lock vacío para proyectos que aún no han fijado versiones y valida schema, IDs, versiones y canales; `saveToolchainLock` persiste mediante `tmp` + `rename` (`packages/toolchain/src/lockfile.ts`).

`probeAllToolVersions` solo ejecuta un contrato local conocido y extrae la versión con la regex `versionExtractor`; hoy conoce probes para `serena`, `cmm` y `engram`. `context7`, `rtk`, `caveman` y `autoskills` no reciben un comando supuesto. Las observaciones fallidas se omiten y no elevan por sí mismas el estado a `installed-working`.

`axiom toolchain plan` es read-only y compara el lockfile con el canal solicitado; `--id` restringe el conjunto. La función pura no da de alta implícitamente todas las entradas del catálogo cuando no recibe IDs explícitos. `axiom toolchain upgrade` solo actualiza el lockfile: requiere `--yes` para persistir, `--dry-run` mantiene el preview, y usa checkpoint/rollback. Ninguna de las dos operaciones descarga, instala o reemplaza binarios externos.

Bug corregido en el incremento 0027: `resolveDetectionPath` calculaba rutas relativas al `cwd` en vez de a `projectRoot`.

Comandos: `axiom toolchain repair [--id <id>]` (idempotente), `axiom toolchain add --id <id> --path <repoId>` (selección por repo), `axiom toolchain gitignore [--write <file>]` (output ordenado y deduplicado).

## MCP (Model Context Protocol) — Axiom YA expone servidor propio

> **Corrección 2026-07-29**: el baseline 2026-07-02 afirmaba "Axiom no expone hoy un servidor MCP propio y genérico" — esa afirmación es **stale**. Desde entonces se añadieron `@axiom/mcp-server` y `@axiom/mcp-tools`.

- **`@axiom/mcp-server`** (`packages/mcp-server/src/`): dispatcher JSON-RPC 2.0 **hand-rolled** (deliberadamente sin `@modelcontextprotocol/sdk`), transporte-agnóstico (`createMcpServer`), más un transporte stdio newline-delimited (`runStdioServer`), sobre los handlers de `@axiom/mcp-tools`.
- **`@axiom/mcp-tools`** (`packages/mcp-tools/src/`): handlers agrupados por dominio (`sdd.*`, `spec.*`, `memory.*`, `axiom.*`) — cada `McpToolRegistryEntry` declara su `domain`, que determina a qué "server kind" pertenece.
- **Server kinds** (`packages/mcp-server/src/tool-sets.ts`): `sdd` (server `sdd-mcp-server`, expone `sdd.*`), `spec` (server `spec-mcp-broker`, expone `spec.*`), `memory` (server `memory-mcp-server`, expone `memory.*`, único kind async — puede spawnear un subproceso `engram mcp` local), y `axiom` (broker unificado `axiom-mcp-broker`, `INC-20260724-unified-axiom-mcp`: UNIÓN de todo `spec.*` + el subset de escritura `sdd.transitionApply`/`sdd.gitCommitSync` + los 3 reads nuevos `axiom.topologyRead`/`axiom.migrationManifestRead`/`axiom.adoptionStateRead`).
- **Proyección a config nativa** (`apps/cli/src/commands/native-mcp-config.ts`, `writeNativeMcpConfig`): hoy solo proyecta los **dos** servers managed `sdd-mcp-server`/`spec-mcp-broker` (no `memory`/`axiom` todavía) al schema nativo VERIFICADO de cada tool (`.mcp.json` claude-code, `.cursor/mcp.json` cursor, `.vscode/mcp.json` vscode/copilot-vscode/github-copilot, `opencode.json` opencode, `.vs/mcp.json` visual-studio-2026 como asunción documentada); `codex`/`antigravity` reciben solo una nota informativa (config user-global, no project-scoped); `litellm` no tiene schema nativo verificado. Ver `../architecture/04-adapters-y-model-routing.md` para la tabla completa.
- **`.axiom/mcp.yml`**: fichero leído por el probe `TC-019` (MCP liveness) de `axiom doctor --deep` — contiene el `command`/`args` de lanzamiento real de los servers managed.
- Catálogo de MCP servers permitidos vía `@axiom/isolation` (`DEFAULT_ALLOWED_MCP_SERVERS`, verificado en `packages/isolation/src/p0.ts`): **3 por defecto** — `sdd` (solo lectura), `spec` (escritura brokerizada), `serena` (navegación semántica, MVP baseline). Corrige la cifra "28 servidores" del baseline 2026-07-02, que era errónea/no verificada.
- Comando `axiom mcp repair --id <mcpId>` (repair de binding, incremento 0029) y `axiom mcp inventory` (total, required, optional, readonly, por installMode); comando adicional nuevo `mcp-serve.ts` (arranque directo de un server MCP, verificar en código antes de tratarlo como contrato estable).
- `backend-mcp`/`frontend-mcp` bloqueados explícitamente en MVP (verificado por `doctor`).
- Ejemplo real de allowlist auto-adoptada (`Axiom/axiom.config/integrations.yaml`, self-bootstrap del propio repo): declara `sdd-mcp-server` (7 capabilities `sdd.*`) y `spec-mcp-broker` (10 capabilities `spec.*`, read-only) bajo `mcp_servers`, y `serena` en `allowed_tools`.

## Memoria y recall (`@axiom/memory`)

Curación auto/manual por `MemoryKind` (`decision`, `bug`, `learning`, `pattern`, `context`), con invariante scope + `projectId` por entry. GATE 0024: la memoria **no** es fuente de verdad; en conflicto, la spec prevalece. Ranking de recall (incremento 0029): text match (case-insensitive, start-boost, occurrence boost) + recencia (ventana 90 días) + kind boost (`decision=1.5`, `bug=1.4`, `pattern=1.2`, `learning=1.1`, `context=1.0`); cada hit devuelve `reason` explicativo. Recall es opt-in (helper disponible; wire-up al state machine diferido a la fecha del incremento 0029). Comando: `axiom memory inventory`.

## Extensiones opcionales: app plugins y bridge externo

`@axiom/app` (incremento 0030) implementa un sistema de plugins project-scoped con discovery en `.axiom-state/<project>/app-plugins/*.json` (renombrado desde el prefijo antiguo, `INC-20260703-config-folder-renames`; schema guard tolerante: malformados → warnings, no abort; IDs únicos por proyecto) y un bridge declarativo hacia Azure DevOps (mutación externa exige `confirmed: true`). Los plugins no introducen lógica de negocio paralela — el bridge solo declara contrato. Desde entonces se sumó un bridge real de tracker (`@axiom/tracker`/`@axiom/tracker-ado`, comandos `app-launcher-ado.ts`/`external-sync.ts`/`_tracker-status.ts`) — no confundir con este sistema de plugins del incremento 0030, son superficies distintas; verificar en código antes de asumir que comparten implementación. No hay integración con Jira/Confluence implementada.
