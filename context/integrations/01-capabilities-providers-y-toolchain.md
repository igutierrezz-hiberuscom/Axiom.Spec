# Capabilities, providers y toolchain externo

Fuente: `Axiom/docs/configuration/providers-and-capabilities.md`, `Axiom/docs/configuration/files/{capabilities,providers}.md`, informe de incrementos 0015-0030, `Axiom/packages/toolchain/src/probe.ts`, `Axiom/axiom.config/toolchain-catalog.yaml`, `Axiom.Spec/decisions/0031-adr-cmm-replaces-graphify-and-codegraph.md`, `Axiom/packages/mcp-server/src/`, `Axiom/packages/mcp-tools/src/`, `Axiom/packages/isolation/src/p0.ts`.

> Reconciliado 2026-08-06: `codegraph`/`graphify` fueron removidos (ADR-0031, `cmm` los reemplaza); Axiom ya tiene servidor MCP propio (`@axiom/mcp-server` + `@axiom/mcp-tools`). La tanda ACC-023 añadió propagación explícita de identidad a toolchain/providers/worktrees y remapeo de checkpoints legacy.

## Modelo de capabilities

`capabilities.yaml` define el catálogo y el marco de degradación: `capabilities.required` / `.optional` / `.postMvpOptional`, `supportLevels`, `degradationPolicy` y el mapa opcional `mcpOnlyCapabilities`. `@axiom/capability-model` mantiene 16 capabilities provider-routed en los dominios `sdd`, `spec`, `code` y `memory`, y 3 capabilities MCP-only en el dominio `axiom`. Las segundas se cargan en un mapa separado, no generan fallback chains y no requieren una entrada tradicional en `providers.yaml`. Implementado en `@axiom/capability-model`.

## Providers y discovery

`providers.yaml` modela el registry de cuatro providers locales (`filesystem`, `serena`, `cmm`, `engram`) y el único perfil declarativo de discovery `local-only`, con `discoveryOrder: [filesystem]` y fallbacks explícitos.

Regla vigente: `filesystem` es el provider local siempre disponible; `cmm` aporta code-intel estructural y `serena` navegación semántica, ambos seleccionables de forma opcional y degradables si falta su tool. Engram es distinto: el runtime de memoria lo resuelve siempre como backend local obligatorio; su indisponibilidad devuelve `MemoryError` y TC-024 de Doctor falla con remediación. `readEnabledProviders` recibe `projectKey` y aliases legacy cuando el caller los conoce; solo hace scan global cuando no hay identidad.

Impacta directamente a: `configure` (compone install profile), `start` (resuelve provider efectivo y modo de discovery), `doctor` (evalúa consistencia del modelo y defaults).

### Histórico: overlays y gateway (retirados en R-04)

El antiguo camino `enterprise` usaba política y estado local para exigir
gateway, y `axiom gateway start/status` mantenía un snapshot project-scoped.
No había daemon ni conexión remota. Ese eje fue retirado junto con
`axiom-gateway`; el contrato actual es siempre local-only.

## Toolchain externo (`@axiom/toolchain`)

Manifest `toolchain.yaml`: npm/pnpm/yarn/bun, Node, git, python — con detección dinámica (`detect.ts`), loader, validador y repair idempotente.

> **Catálogo reconciliado 2026-07-29 (ADR-0031, `Axiom.Spec/decisions/0031-adr-cmm-replaces-graphify-and-codegraph.md`)**: las tools P1 del baseline (`CodeGraph`, `Graphify`, añadidas en el incremento 0027) fueron **removidas y reemplazadas por `cmm`** (`codebase-memory-mcp`). El catálogo real vigente, verificado en `axiom.config/toolchain-catalog.yaml` (self-bootstrap del propio repo `Axiom/`) y en `packages/toolchain/src/probe.ts`, es: **`serena`** (code-intelligence), **`cmm`** (code-intelligence, reemplaza codegraph/graphify), **`engram`** (library-context), **`context7`** (library-context), **`rtk`** (input-optimizer), **`caveman`** (output-optimizer), **`autoskills`** (skill-surface) — 7 tools, todas `mvp: false` (catálogo permitido/opcional, no requerido por defecto; el operador las agrega vía `axiom toolchain add --id <id>`). `cmm` tiene evidencia de fallback específica: un fichero `.cmm/sync-state.json` en el proyecto se trata como prueba de instalación funcionando (`packages/toolchain/src/probe.ts`).

### Versionado reproducible del toolchain (`INC-20260730-toolchain-versioning`)

`axiom.config/toolchain-catalog.yaml` usa `schemaVersion: 2`. Cada entrada puede declarar `versionExtractor`, versiones de política por canal (`stable`, `candidate`, `edge`) y `compatibility.axiomMinVersion`. El catálogo es una allowlist/política: no implica que todas las tools estén habilitadas en `axiom.config/toolchain.yaml` ni que deban instalarse.

El estado de versiones fijadas vive en `.axiom-state/<projectKey>/toolchain.lock`, un YAML `schemaVersion: 1` ignorado junto con `.axiom-state/`. El lock contiene `projectId`, `lockedAt` y `tools`; cada entrada registra `id`, `version`, `channel` y, opcionalmente, `probeCommand`, `probeOutput` y `probedAt`. `loadToolchainLock` devuelve un lock vacío para proyectos que aún no han fijado versiones y valida schema, IDs, versiones y canales; `saveToolchainLock` persiste mediante `tmp` + `rename` (`packages/toolchain/src/lockfile.ts`).

`detectToolState`, `detectToolStateWithProbe` y `repairTool` aceptan aliases
legacy. Repair mueve un marker antiguo al destino canónico sin dejar copias
duplicadas. El provisioning de worktree pasa `Execution.projectId` al lector de
providers, de modo que dos proyectos en el mismo repo fuente no se mezclan.

`probeAllToolVersions` solo ejecuta un contrato local conocido y extrae la versión con la regex `versionExtractor`; hoy conoce probes para `serena`, `cmm` y `engram`. `context7`, `rtk`, `caveman` y `autoskills` no reciben un comando supuesto. Las observaciones fallidas se omiten y no elevan por sí mismas el estado a `installed-working`.

`axiom toolchain plan` es read-only y compara el lockfile con el canal solicitado; `--id` restringe el conjunto. La función pura no da de alta implícitamente todas las entradas del catálogo cuando no recibe IDs explícitos. `axiom toolchain upgrade` solo actualiza el lockfile: requiere `--yes` para persistir, `--dry-run` mantiene el preview, y usa checkpoint/rollback. Ninguna de las dos operaciones descarga, instala o reemplaza binarios externos.

Bug corregido en el incremento 0027: `resolveDetectionPath` calculaba rutas relativas al `cwd` en vez de a `projectRoot`.

Comandos: `axiom toolchain repair [--id <id>]` (idempotente), `axiom toolchain add --id <id> --path <repoId>` (selección por repo), `axiom toolchain gitignore [--write <file>]` (output ordenado y deduplicado).

## Contexto técnico derivado y memoria compartida (R-11)

`axiom context index [--role <rol>] [--path <specRepo>]` recorre `context/**/*.md` y regenera de forma atómica el índice derivado `technical-context/indexes/<rol>.index.yml`; ese archivo nunca se edita a mano. Un documento puede declarar `tags: [..]` en frontmatter YAML. Solo ese campo participa: se normaliza y valida; sin tags explícitas se usa una tag de fallback derivada de su carpeta de primer nivel o `repo` para la raíz. Tras actualizar documentos de contexto o sus tags, se debe volver a ejecutar el comando y revisar el índice generado, no modificarlo directamente.

Las tools `spec.recommendedContextList` y `spec.implementationContextRead` consumen el mismo selector: primero `mandatory.always`, luego los grupos `mandatory.whenTags` cuyo conjunto completo coincide y finalmente los documentos `available` que coinciden con alguna tag solicitada. El selector deduplica por path y clasifica cada entrada como `mandatory` o `recommended`. Sin `taskTags`, o con `taskTags: []`, no recomienda entradas disponibles. En la lectura compuesta, las tags estructuradas de `plan.taskType` y del rol se agregan únicamente cuando el caller ya aportó al menos una tag explícita; no hay inferencia de texto, scoring ni IA. Los paths siguen confinados al spec repo; `contextBudget` conserva referencias en `small`, contenido obligatorio en `medium` y añade ADRs relacionados en `large`.

`axiom knowledge sync --increment <id> --phase <phase>` intercambia memoria de equipo mediante chunks JSON append-only y un manifest versionable. Es preview por defecto; `--confirm` autoriza persistencia y Git local, y `--push` autoriza el remoto. Solo exporta `visibility: project-shared`, preserva evidencia y metadata estable y omite las entradas private, sin visibilidad o con secretos en cualquier campo textual serializado. `axiom knowledge pull` no se filtra por incremento: al confirmar procesa todos los chunks pendientes. El marker personal vive en `.axiom-state/<projectKey>/knowledge/imported-chunks.json`; la antigua `.engram/.imported` se migra o ignora. Un chunk solo queda completado tras importar todas sus entradas válidas; corrupción y fallos parciales permanecen visibles y reintentables. `.engram/engram.db` y los markers personales no son parte del intercambio versionable.

## MCP (Model Context Protocol) — Axiom YA expone servidor propio

> **Corrección 2026-07-29**: el baseline 2026-07-02 afirmaba "Axiom no expone hoy un servidor MCP propio y genérico" — esa afirmación es **stale**. Desde entonces se añadieron `@axiom/mcp-server` y `@axiom/mcp-tools`.

- **`@axiom/mcp-server`** (`packages/mcp-server/src/`): dispatcher JSON-RPC 2.0 **hand-rolled** (deliberadamente sin `@modelcontextprotocol/sdk`), transporte-agnóstico (`createMcpServer`), más un transporte stdio newline-delimited (`runStdioServer`), sobre los handlers de `@axiom/mcp-tools`.
- **`@axiom/mcp-tools`** (`packages/mcp-tools/src/`): handlers agrupados por dominio (`sdd.*`, `spec.*`, `memory.*`, `axiom.*`) — cada `McpToolRegistryEntry` declara su `domain` para routing y documentación interna. Todos los dominios se sirven a través del único `server kind` `axiom`; los ids `axiom.*` son MCP-only en el capability model y no se convierten en providers tradicionales por el hecho de vivir en el broker.
- **Server kind único** (`packages/mcp-server/src/tool-sets.ts`): `axiom` (`axiom-mcp-broker`) expone la unión completa de los dominios internos `sdd.*`, `spec.*`, `memory.*` y `axiom.*`, incluyendo las mutaciones `sdd.transitionApply`, `sdd.gitRoleBranch` y `sdd.gitCommitSync`. Los dominios no representan procesos separados; Engram puede seguir siendo el backend local de las tools `memory.*`.
- **Proyección a config nativa** (`apps/cli/src/commands/native-mcp-config.ts`, `writeNativeMcpConfig`): el setup valida primero el proyecto, `mcp.yml`, `mcp-manifest.yaml`, `enabled` y `targetRepo`; solo entonces proyecta servers managed al schema nativo de `.mcp.json` (claude-code), `.cursor/mcp.json` (cursor), `.vscode/mcp.json` (github-copilot/vscode) u `opencode.json` (opencode). Visual Studio no tiene schema verificado; `codex`/`antigravity` no reciben escritura ni recomendación global automática sin binding seguro. Los writers eliminan solo IDs Axiom stale y preservan servidores custom; JSON inválido se conserva con warning. Ver `../architecture/04-adapters-y-model-routing.md` para la tabla completa.
- **`.axiom/mcp.yml`**: fichero leído por el probe `TC-019` (MCP liveness) de `axiom doctor --deep` — contiene el `command`/`args` de lanzamiento real de los servers managed.
- Catálogo de MCP servers permitidos vía `@axiom/isolation` (`DEFAULT_ALLOWED_MCP_SERVERS`, verificado en `packages/isolation/src/p0.ts`): **uno por defecto** — `axiom-mcp-broker`, el único broker gestionado de Axiom. `serena`, `cmm` y `engram` son providers externos opcionales con sus propios clientes/backend y no son brokers de gobierno del proyecto.
- Comando `axiom mcp repair --id <mcpId>` (repair de binding, incremento 0029) y `axiom mcp inventory` (total, required, optional, readonly, por installMode); comando adicional nuevo `mcp-serve.ts` (arranque directo de un server MCP, verificar en código antes de tratarlo como contrato estable).
- `backend-mcp`/`frontend-mcp` bloqueados explícitamente en MVP (verificado por `doctor`).
- Ejemplo real de allowlist auto-adoptada (`Axiom/axiom.config/integrations.yaml`, self-bootstrap del propio repo): declara únicamente `axiom-mcp-broker` bajo `mcp_servers` y `serena` en `allowed_tools`.

## Memoria y recall (`@axiom/memory`)

La memoria usa `MemoryKind` (`decision`, `bug`, `learning`, `pattern`, `context`) con invariante de scope + `projectId` por entrada y se persiste explícitamente en el Engram local project-pinned. GATE 0024 fija que la memoria no es fuente de verdad: en conflicto, la spec prevalece. El recall conserva ranking por match de texto, recencia y kind, y cada hit devuelve `reason` explicativo. No existe curación automática desde el audit trail: `axiom learn` fue retirado; los agentes registran decisiones y bugs mediante `axiom memory add` y comunican su resultado. `axiom memory inventory` sigue disponible para inspección.

## Extensiones opcionales: app plugins y bridge externo

`@axiom/app` (incremento 0030) implementa un sistema de plugins project-scoped con discovery en `.axiom-state/<projectKey>/app-plugins/*.json` (renombrado desde el prefijo antiguo, `INC-20260703-config-folder-renames`; schema guard tolerante: malformados → warnings, no abort; IDs únicos por proyecto) y un bridge declarativo hacia Azure DevOps. El contrato vigente separa schema, discovery, resolución y ejecución: solo handlers estáticos allowlisted pueden ejecutarse; `command` es una etiqueta informativa. Read es read-only; `local-mutation`/`external-mutation` requieren preview y `confirmed: true`; valores, opciones y envelopes se validan antes del handler. El catálogo y resultados se proyectan sin propiedades desconocidas, secretos, userinfo ni query de URLs. `kind: none` usa `NullTracker` sin red; `kind: ado` usa los ports/fakes del tracker. No hay integración con Jira/Confluence implementada.
