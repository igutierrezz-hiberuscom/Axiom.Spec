# 06 Integraciones y Capacidades

## Integraciones esperadas

1. workspace local multi-root (`@axiom/topology`, layout `installed-multi-repo`);
2. adapters de ejecución (**8** packages `@axiom/adapters-*` dedicados desde `INC-20260726-adapter-generators`; `litellm` fue retirado y `copilot-vscode` es alias legacy sin package; ver "Paridad de alcance de adapters + launcher onboarding" al final);
3. surfaces basadas en MCP (catálogo vía `@axiom/toolchain`, comandos `axiom mcp`, `axiom toolchain` y servers ejecutables `@axiom/mcp-server` para los kinds `sdd`, `spec`, `memory` y `axiom`); los brokers se lanzan con `axiom mcp serve --kind <kind>` y fijan el proyecto al que sirven;
4. herramientas de análisis y contexto cuando aporten valor real: `cmm` como proveedor estructural de code-intel, Serena como proveedor semántico y Engram como backend de memoria. RTK, Caveman y Autoskills son capacidades opcionales/curadas del toolchain, no dependencias vendorizadas ni tools marcadas como instaladas por defecto. `cmm` sustituyó a `codegraph` y `graphify` como providers seleccionables; las menciones a esos ids que siguen en el documento forman parte del historial de migración, no del contrato actual.

## Targets soportados hoy (verificado, `SUPPORT_MATRIX` de `@axiom/model-routing`)

| Target | Nivel de soporte | Routing per-slot |
|---|---|---|
| `opencode` | `multi-mode` | Sí — único target con cobertura completa hoy |
| `claude-code` | `single-mode` | No — cae a `medium` (`fallbackReason: per-slot-routing-unsupported`) |
| `github-copilot`, `vscode`, `cursor`, `codex` | `fallback-only` | No |
| `copilot-vscode`, `antigravity`, `visual-studio-2026` | `fallback-only` | No |

Esto reemplaza la lista aspiracional previa de "targets iniciales" (Opencode CLI principal, Antigravity IDE secundario, VS Code+Copilot IDE principal, Claude Code CLI secundario): en el código actual, `opencode` es el único target con soporte de routing per-slot profundo.

**Ojo — dos ejes SEPARADOS (actualizado por `ACC-025..ACC-029`)**: el `SupportLevel` de esta tabla mide SOLO la proyección de model-routing per-slot. NO debe confundirse con "tiene generador dedicado / MCP nativo": `SUPPORT_MATRIX` tiene 8 entradas activas, los **8 targets** tienen paquete `@axiom/adapters-<target>` dedicado (todos salvo `copilot-vscode`, que es alias legacy de `github-copilot`; `litellm` fue retirado) y **5** reciben config MCP nativa de proyecto. Un target puede ser `fallback-only` en routing y a la vez tener generador dedicado + MCP nativo real.

### Contrato vigente de instrucciones Copilot (ACC-024)

`github-copilot` y `copilot-vscode` escriben la instrucción general en
`.github/copilot-instructions.md`, la ubicación oficial de instrucciones de
repositorio para GitHub Copilot y VS Code. `copilot-vscode` escribe además
`.vscode/settings.json`, `.vscode/extensions.json` y `.vscode/mcp.json`; no
usa `.vscode/copilot-instructions.md`. Las process surfaces específicas de
ambos targets se escriben una sola vez en
`.github/instructions/<id>.instructions.md`.

Los entrypoints `configure`, `sync`, `workspace setup` y
`@axiom/adapters-github-copilot` convergen en el writer de
`@axiom/document-bootstrap`. El resolver usa la plantilla versionada del
proyecto cuando es legible y, si no, la plantilla bundleada. La regeneración
solo sustituye `AXIOM:GENERATED`; el contenido humano fuera de ese bloque y
`TEAM:CUSTOM` se conserva byte a byte. Una copia legacy en
`.vscode/copilot-instructions.md` se migra escribiendo primero el destino; si
el destino y la fuente contienen zonas humanas divergentes, la fuente se
conserva y se emite un warning.

### Registro histórico: adapter depth y snapshot diagnóstico — INC-20260708-adapters-depth

> Este bloque conserva el diagnóstico de `claude-code` y las decisiones intermedias sobre `antigravity`/`visual-studio-2026`. Para el contrato vigente de generadores y MCP nativo, prevalecen las secciones de paridad de adapters de 2026-07-26 al final de este capítulo.

El `single-mode` de `claude-code` (tabla de arriba) es una decisión de diseño **intencional y testeada**, no un hueco pendiente de completar: `SUPPORT_MATRIX`, `applyFallback` (`fallback.ts`) y `resolveSlot` (`resolver.ts`) hacen que CUALQUIER slot resuelva siempre a `medium` con `fallbackReason:'per-slot-routing-unsupported'` para este target — el resolver ni siquiera llega a consultar la policy. No existe hoy un schema nativo verificado de routing per-slot para Claude Code (a diferencia de `opencode.json`, que sí soporta `model` por agente). Por eso este incremento **no fabrica** capacidad per-slot para claude-code:

- **`projectToClaudeCode`** (`@axiom/model-routing/claude-code-projection.ts`) es un espejo estructural de `projectToOpencode`, invocado por `axiom model validate` cuando el target es `claude-code`. Escribe un snapshot **diagnóstico** en `<root>/.claude/model-routing.json`, con la MISMA forma que `.opencode/model-routing.json` (`project`, `target`, `supportLevel`, `slots`, `projectedAt`) — pero como `single-mode` siempre cae a fallback, los 7 slots quedan uniformemente en `{ modelClass:'medium', fellBack:true, fallbackReason:'per-slot-routing-unsupported' }`. Esa uniformidad ES la señal honesta: un consumidor que lea el archivo ve inmediatamente que el routing per-slot no está activo para este target, en vez de que el archivo esté silenciosamente ausente.
- **`@axiom/adapters-claude-code`** expone un loader `loadRoutingSnapshot`/`getSnapshotModelClass` (mismo shape que el de `@axiom/adapters-opencode`) para leer ese snapshot — informativo únicamente; no se conecta hoy al render del `## Routing note` estático del `AGENTS.md` generado (queda como trabajo futuro no bloqueante).
- **`antigravity`**: su config MCP real es de usuario GLOBAL (`~/.gemini/config/mcp_config.json`, clave `mcpServers`), no por proyecto. Axiom **nunca** escribe ahí automáticamente (evitar mezclar servers de distintos proyectos sin decisión explícita del operador) — el `.antigravity/AGENTS.md` generado ahora incluye una nota operator-facing documentando el path y que agregar los servers de Axiom ahí es un paso manual.
- **`visual-studio-2026`**: no existe hoy un schema nativo de MCP verificado para este target (`NATIVE_MCP_TARGETS` no lo incluye). El `.vs/AXIOM.md` generado ahora incluye una nota informativa al respecto (y menciona que `.vscode/mcp.json`, si `copilot-vscode`/`github-copilot` también está seleccionado, podría ser legible por VS2026 según sus rutas de búsqueda documentadas) — sin implementar ninguna escritura automática no verificada. *(Actualizado por `ACC-027`, 2026-08-10: `visual-studio-2026` ya no genera `.vs/AXIOM.md`; se proyecta sobre la instrucción común `.github/copilot-instructions.md`. Este claim describe el estado previo a esa retirada.)*
- **Dos correcciones de registro encontradas al construir el e2e de adapters**: `GENERATED_FILES_BY_TARGET['copilot-vscode']` declaraba `.vscode/copilot-instructions.md` + `.vscode/settings.json`, pero la ruta de workspace-setup nunca escribía ahí (caía al path default de `writeCopilotInstructions`, `.github/copilot-instructions.md`, compartido por coincidencia con el target `github-copilot`); y `GENERATED_FILES_BY_TARGET['cursor']` declaraba un path (`.cursor/rules/axiom.mdc`) que ningún generator escribió jamás, mientras el generator real (`.cursor/settings.json`+`.cursor/AGENTS.md`) no estaba declarado. Ambos corregidos.

## Capability model y providers

`@axiom/capability-model` separa 16 capabilities provider-routed por los dominios `sdd`, `spec`, `code` y `memory`, con `supportLevels` y `degradationPolicy`, de 3 capabilities MCP-only (`axiom.topologyRead`, `axiom.migrationManifestRead`, `axiom.adoptionStateRead`) que se registran en la superficie MCP pero no exigen un provider tradicional. Providers se resuelven con el único perfil `local-only` y `discoveryOrder: [filesystem]`. En el estado actual, filesystem cubre workflow/spec, `cmm` cubre las capacidades estructurales, `serena` cubre la navegación semántica y `engram` implementa ambos recalls de memoria; tres capabilities opcionales (`code.symbolSearch`, `code.referenceSearch`, `code.impactAnalysis`) permanecen sin provider y Doctor las reporta como warning.

`@axiom/doctor` comprueba la cobertura del catálogo provider-routed con `CC-004`: usa un universo canónico independiente de las capabilities que ya aparecen en `providers.yaml`. Una capability requerida activa sin provider produce `fail`; una opcional o post-MVP sin provider produce `warn`; una capability `disabled` o `unavailable` queda visible sin fallo activo; las capabilities MCP-only quedan fuera de esta obligación. El repo `Axiom` produce `CC-004 warning` porque 13 de las 16 capabilities provider-routed están servidas y las tres opcionales citadas no tienen provider.

**Providers cableados y seleccionables** (`INC-20260708-*`, actualizado por `INC-20260724-cmm-replaces-graphify-codegraph`): existe una capa de ejecución real (`@axiom/providers`) con clientes locales de code-intel y backend local de memoria, degradación tipada `not-installed` y fallback explícito. El registry canónico contiene 4 ids locales (`filesystem`, `serena`, `cmm`, `engram`); la selección project-scoped se guarda aparte en `workspace.json#providers`. Los ids `codegraph`, `graphify`, `axiom-gateway` y `generated-snapshots` quedan fuera del registry, del routing y de la UX de selección.

### Seam de ejecución de providers (`@axiom/providers`) — INC-20260708-provider-runtime-execution-seam

Paquete nuevo `@axiom/providers`: la capa de EJECUCIÓN que faltaba entre las declaraciones de provider de `@axiom/capability-model` y la resolución pura de `@axiom/tool-routing`. Construido SOBRE el dispatcher `routeTool` existente y sin modificar (cero reimplementación de routing), LOCAL-only.

- **`ProviderClient`** (interfaz) + **`ProviderInvokeResult`** (unión discriminada `ok`/`degraded`, **nunca lanza**) + **`ProviderRegistry`** (register/lookup por provider id).
- **`invokeCapabilityLive(capabilityId, input, ctx, { registry, routing })`**: resuelve el provider efectivo + cadena de fallback vía `routeTool`, ejecuta a través del `ProviderClient` registrado, camina la cadena de fallback ante degradación, y nunca lanza — un provider ausente/no instalado devuelve un resultado `degraded` tipado.
- **`createStdioMcpClient({ command, args, cwd, env })`**: cliente MCP stdio LOCAL reusable — spawnea un proceso hijo local, habla JSON-RPC 2.0 delimitado por saltos de línea (`initialize`/`tools/list`/`tools/call`), con timeouts y shutdown limpio. Es el mismo framing hecho a mano de `@axiom/mcp-server` (sin `@modelcontextprotocol/sdk`, sin dependencias npm nuevas), pero es un CLIENTE (escribe requests al stdin del hijo, lee del stdout) — módulo separado del server transport.
- **`LOCAL_ONLY`/`isLocalTarget`**: guard puro que rechaza/degrada cualquier config de provider que apunte a un endpoint no-local (chequeo de host por string, sin resolución DNS).
- Un cliente `filesystem` de referencia in-process prueba el seam end-to-end.

Es el cimiento que los clientes concretos (code-intel, memoria) reutilizan en vez de inventar su propio transporte.

### Registro histórico: providers de code-intel cableados (codegraph/serena/graphify) — INC-20260708-code-intel-providers-wired

> Este bloque conserva el diseño implementado antes de ADR-0031. No es el contrato actual: `cmm` sustituyó a `codegraph` y `graphify`; la matriz vigente de providers está en la apertura de este capítulo y en la sección de stack externo de 2026-07-24.

Sobre el seam anterior, los tres providers de code-intel que `axiom.config/providers.yaml` mapea son ahora `ProviderClient`s LOCALES reales, cada uno construido desde un factory compartido (`createCodeIntelProviderClient`) que spawnea el server MCP local de la tool por llamada, invoca la única tool MCP mapeada, normaliza el resultado y clasifica fallos de spawn/handshake en `not-installed` vs. `error` ordinario:

| Provider | Comando de lanzamiento LOCAL | Tool MCP | Capability |
|---|---|---|---|
| `codegraph` | `codegraph serve --mcp` (stdio) | `codegraph_explore` | `code.knowledgeGraph` (fallback: `serena`) |
| `serena` | `serena start-mcp-server --context ide --project <root>` | `find_symbol` | `code.semanticNavigation` (fallback: `filesystem`) |
| `graphify` | `python -m graphify.serve <root>/graphify-out/graph.json` | `query_graph` | `code.structureAnalysis` (fallback: `codegraph`) |

- `registerCodeIntelProviders(registry, opts?)` registra los tres en orden de prioridad (codegraph → serena → graphify, la misma cadena de fallback de `providers.yaml`); `opts` permite override de comando/paths para tests y instalaciones locales.
- **Sin bundling/vendoring, sin dependencias npm nuevas**: instalar cada tool local sigue siendo responsabilidad del usuario; Axiom solo la invoca si está instalada o degrada limpiamente (`not-installed`, con mensaje accionable que nombra el comando exacto) si falta.
- `runWorkspaceSetup` ganó un hook `initializeCodeIntelIndexes` OPT-IN y best-effort (no-op salvo que el proyecto habilite ≥1 provider; nunca falla el setup), y el `AGENTS.md` canónico gana una sección condicional "Code intelligence tools" que describe solo lo realmente cableado.
- **Hueco honesto conservado**: `code.impactAnalysis`/`code.symbolSearch`/`code.referenceSearch` siguen SIN provider mapeado en `providers.yaml` (fuera de scope aquí — ampliar el capability model es otra preocupación).

### Backend de memoria real (engram) — INC-20260708-memory-real-local-backend

`@axiom/memory` pasa de un MVP (mapa in-memory + persistencia JSON) a tener un backend REAL, LOCAL, cross-session, cableando ENGRAM (binario Go + SQLite+FTS5 local, MCP stdio) como una implementación aditiva de `MemoryBackend`, con fallback limpio al backend JSON cuando engram no está instalado:

- **`createEngramBackend(projectRoot, scope, opts?)`**: `MemoryBackend` que habla con un proceso `engram mcp --project=<projectId>` LOCAL vía `createStdioMcpClient` (AB3). Mapea las ops de memoria de Axiom a las tools reales de engram: `save → mem_save` (con `topic_key`/`session_id`), `load`/`query → mem_search`, `saveSessionSummary → mem_session_summary`.
- **`resolveMemoryBackend(projectRoot, scope, opts?)`**: probe (un `initialize()` real, no solo un chequeo de PATH) y selecciona engram-si-disponible o el backend JSON de fallback. Nunca lanza; siempre devuelve un `note` de una línea.
- **UPSERT topic-keyed** y **session summaries** añadidos de forma aditiva a AMBOS backends (`MemoryEntry.topicKey`/`.sessionId`; engram nativo vía `topic_key`, JSON vía find-and-replace por `(projectId, topicKey)`).
- **Superficie MCP real**: `memory.decisionRecall`/`memory.contextRecall` — antes capabilities declaradas con arrays de handler vacíos — tienen ahora handlers reales en `@axiom/mcp-tools`, expuestos por un nuevo `McpServerKind` `memory` (`axiom mcp serve --kind memory`). Requirió promover `McpServer.handle()` a `async` project-wide (mecánico, sin cambio de comportamiento para `sdd.*`/`spec.*`).
- **GATE 0024 preservado y extendido**: la garantía de aislamiento cross-project se conserva y gana un segundo pin a nivel de proceso (`--project=`); el input-builder `memory.*` fija SIEMPRE `projectRoot` al `context` del server (nunca lee un `projectRoot`/`projectId` del llamador), un pin más estricto aún que el de `sdd.*`/`spec.*` (ver [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md)). Sin dependencia SQLite npm nativa — la SQLite local de engram se usa exclusivamente vía su superficie MCP stdio.

### Registro histórico: selección de providers (project-scoped, no false-breadth) — INC-20260708-wizard-configure-provider-selection

> Este bloque conserva la forma de selección anterior a ADR-0031. El estado actual permite seleccionar `cmm`, `serena` y `engram`; `codegraph` y `graphify` ya no forman parte de `SELECTABLE_PROVIDER_IDS`.

La selección de qué providers LOCALES habilita un proyecto es ahora EXPLÍCITA y persistida, sin implicar más amplitud de la que el usuario pidió:

- **En esta implementación histórica, `providers.yaml` no se recortaba** (desviación deliberada y documentada del brief literal): su schema (`ProviderRegistrySchema.length(7)`) y los checks `CC-001`/`CC-003` de doctor lo mantenían como el registry cerrado de 7 ids canónicos. La selección de un proyecto se persistía como estado project-scoped en `.axiom-state/<projectId>/workspace.json#providers`. El registry actual tiene 4 ids locales y se documenta en la apertura de este capítulo.
- **`SELECTABLE_PROVIDER_IDS` histórico = `['codegraph','serena','graphify','engram']`** (`@axiom/providers/selectable.ts`): eran los 4 ids ofrecidos por aquella UX. El conjunto vigente es `['cmm','serena','engram']`; `filesystem` sigue siendo always-on. `axiom-gateway` y `generated-snapshots` son referencias históricas retiradas y no forman parte del registry, routing ni UX actuales.
- **`buildProjectProviderRegistry(projectRoot, opts?)`**: lee la selección (de `workspace.json`, o de `opts.enabledProviders`) y devuelve un `ProviderRegistry` con exactamente los clientes code-intel habilitados registrados (filtrado sobre `registerCodeIntelProviders`) más el `filesystem` always-on. `engram` NO se registra como `ProviderClient` (tiene su propia interfaz `MemoryBackend`, resuelta vía `resolveMemoryBackend`) — la función reporta `engramEnabled: boolean` en su lugar, sin inventar un shim.
- **Superficies de selección**: el step `providers` del wizard (multi-select, sin preselección) y `axiom configure --providers <csv>` (ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md)) — ambos persisten en `workspace.json#providers`.
- **Check de doctor `PS-001`**: reporta por provider habilitado si está registrado/probablemente-instalado (`warn` — nunca `fail` — cuando está habilitado-pero-no-instalado; una tool local ausente no es un defecto del proyecto).
- **Nota de cableado pendiente**: `invokeCapabilityLive`/`ProviderRegistry` aún no tiene consumidor in-repo fuera de tests y del check `PS-001`; cablearlo en cada ruta de comando/skill/agent (para que un `axiom` real enrute un `code.*` en runtime) queda como follow-up.

### Capa de reglas de código language-scoped (`axiom.config/rules/`) — INC-20260708-rules-layer

Axiom gana una capa curada, LOCAL, bundleada y SCOPED-POR-LENGUAJE de reglas de estándar de código (inspirada en, no portada de, `rules/` de ECC — se rechazó explícitamente ingerir su librería de ~20 lenguajes). Es un artefacto análogo pero independiente de la capa de skills: mismo patrón "bundle como constantes TS, scaffold best-effort, no-clobber por fichero".

- **Catálogo curado bundleado** (constantes TS en `apps/cli/src/commands/workspace-rules.ts`): `common` (guías de ingeniería Axiom/SDD: `Result<T,E>` en vez de throw, escritura atómica, no arquitectura especulativa, disciplina de test, workflow spec-first) + `typescript`/`python`/`csharp`/`angular` (un doc conciso cada uno).
- **`scaffoldRules({ repoPath, languages })`**: escribe `axiom.config/rules/common.md` (siempre) + `axiom.config/rules/<language>.md` por lenguaje pedido, best-effort, no-clobber por fichero. `inferRepoLanguages(repoPath)` infiere por marcadores (`package.json`/`tsconfig.json`→typescript, `*.csproj`→csharp, `angular.json`→angular, `pyproject.toml`/`requirements.txt`→python).
- **Cableado en `runWorkspaceSetup`**: `common` en el repo de control (gated on `created`) y `common` + lenguajes inferidos en cada repo de código (NO gated on `created` — las reglas son baratas, aditivas y no-clobber, útiles incluso en un repo preexistente que se cablea en un workspace).
- **`AGENTS.md`**: sección condicional "Coding rules" listando los `axiom.config/rules/<scope>.md` realmente presentes en disco.
- **Proyección nativa a adapter**: el generador de cursor proyecta best-effort el doc `common` a `.cursor/rules/axiom-common.mdc` (la única ubicación nativa de reglas confirmada y sin uso previo), no-clobber contra ediciones del usuario. Ningún otro adapter se toca.

### Skill de comunicación terse inter-agente (`axiom-terse-comms`) — INC-20260708-caveman-terse-comms-skill

Skill semilla bundleada nueva (`axiom-terse-comms`, la 6ª en `CANONICAL_SEED_SKILLS`, marcada `available` no `mandatory`) que codifica la FILOSOFÍA de compresión de tokens de caveman, pero acotada ESTRICTAMENTE a comunicación agente-Axiom↔agente-Axiom (handoffs, briefs de subagente, status estructurado). Empareja con un bloque opcional "Inter-agent communication style" en el `AGENTS.md` canónico (`interAgentTerseComms?: boolean`, renderizado solo si `true`).

- **Split humano/agente duro y explícito**: la convención terse aplica SOLO al tráfico inter-agente; todo lo que un humano lee o valida (specs 00-08, READMEs de incremento/bug, docs de decisión, mensajes de PR/commit, resúmenes user-facing, documentación generada) queda LIGERO y LEGIBLE, siempre. Nada auto-comprime.
- **No es un compresor runtime**: es GUÍA (opt-in en espíritu), no una tool que reescribe output. No se vendoriza `C:\repos\caveman`. Incluye una leyenda de símbolos curada y pequeña (`→ ∀ ! ⊥ ? ~`, `ok`/`fail`) para que los mensajes terse queden decodables.

El legend + reglas de esta guía se de-duplicaron a una fuente única en `INC-20260709-terse-comms-single-source`: viven ahora en UNA constante TS bundleada (`@axiom/document-bootstrap`'s `inter-agent-comms.ts`: `INTER_AGENT_SYMBOL_LEGEND` / `INTER_AGENT_TERSE_RULES`), consumida TANTO por `renderInterAgentCommsBlock` (el bloque del `AGENTS.md` canónico) COMO por el seed skill `axiom-terse-comms` (`workspace-skills.ts`) — antes estaban hand-duplicadas (inglés en el renderer, español en la skill) y podían divergir en silencio. Un test de drift (`apps/cli/tests/terse-comms-drift.test.ts`) falla si cualquiera de las dos superficies re-inlinea una copia divergente.

### Detección de stack + sugerencia de skills (autoskills) — INC-20260708-autoskills-wizard-phase

Trae la IDEA de `C:\repos\autoskills` (auto-detectar el stack de un repo y sugerir skills curadas) a Axiom, sin ingerir su registro de 100+ skills. Coexiste como catálogo curado, pequeño, bundleado-como-TS, independiente de (y aditivo a) la baseline de skills y la capa de reglas.

- **`workspace-autoskills.ts`**: `detectStack(repoPath)` (reusa `inferRepoLanguages` de la capa de reglas + lectura best-effort de `package.json` para `react`/`nextjs`/`express`), `suggestSkillsForRepo` (compone detección contra `STACK_SKILL_MAP` — 5 stack skills nuevas curadas: `react`, `nextjs`, `angular`, `dotnet-api`, `python`), `installSuggestedSkills` (append no-clobber al `skills-catalog.yaml` existente del repo + materialización vía `@axiom/skills` + append a `skills-index/<roleId>.yaml#available`).
- **Dos superficies convergentes**: la fase final del wizard (una-por-una por repo de código recién creado, ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md)) y el comando standalone `axiom skills suggest [--repo] [--apply]` (usable contra cualquier repo). Ambas convergen en `installSuggestedSkills`, así que las skills instaladas siempre aterrizan en `skills-index/<roleId>.yaml#available` — el artefacto al que las implementation agents de ese repo ya apuntan.

### engram vía MCP stdio en la config nativa por herramienta (sin daemon `engram serve`) — INC-20260709-engram-mcp-stdio-native-config

Cuando el provider `engram` está seleccionado para el workspace (`workspace.json#providers`, ver "Selección de providers" arriba), `runWorkspaceSetup` inyecta además una entrada de servidor MCP **LOCAL de engram** en la config MCP nativa de cada herramienta (`writeWorkspaceNativeMcpConfigs`, `apps/cli/src/commands/workspace-mcp.ts`), por cada repo del workspace × cada adapter seleccionado. Es una entrada **stdio-only y pinneada al proyecto**:

- **Comando/args**: `command: 'engram'`, `args: ['mcp', '--project', <projectId>, '--tools', 'agent']`. `engram mcp` levanta el server MCP de engram sobre **stdio** (no HTTP); `--project <projectId>` lo aísla al proyecto de este workspace (el mismo `effectiveProjectId` del registro); `--tools agent` selecciona el perfil de tools de agente de engram (flags verificados contra `cmd/engram/main.go`). Reutiliza los writers nativos existentes (`writeNativeMcpConfig`), así que aterriza en la forma correcta de cada tool: `mcpServers.engram` (claude-code `.mcp.json`, cursor `.cursor/mcp.json`), `servers.engram` con `type:'stdio'` (vscode `.vscode/mcp.json`), y `mcp.engram` con `type:'local'` + `command:['engram',...]` + `enabled:true` (opencode `opencode.json`).
- **Sin daemon `engram serve`**: con esta entrada, OpenCode y el resto de adapters hablan con engram por MCP stdio directamente — **NO** hace falta levantar el daemon HTTP `engram serve` (puerto 7437). NINGUNA config generada por Axiom referencia engram vía HTTP / puerto 7437 / `ENGRAM_URL`: la integración es stdio pura por construcción (guardado por test).
- **Merge-preserving y fuera de `.axiom/mcp.yml`**: reusa el merge por-id de los writers nativos (preserva servers/keys del usuario, escritura atómica). La entrada `engram` NUNCA se escribe en `.axiom/mcp.yml` — ese fichero es el config canónico de brokers `type:'axiom'` de Axiom (`sdd-mcp-server`/`spec-mcp-broker`); engram es un server MCP local de terceros, no un broker Axiom. La escritura nativa se dispara si hay ≥1 broker axiom **o** si engram está habilitado, de modo que un workspace engram-only también recibe la entrada.

Coherente con el backend de memoria real (`createEngramBackend`, arriba), que ya hablaba con `engram mcp` LOCAL vía stdio para el runtime de `@axiom/memory`; ahora esa MISMA vía stdio queda declarada también en la config nativa que los adapters externos leen. Ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md) y [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md).

### Tuning de recall del backend de memoria engram — INC-20260709-engram-backend-recall-tuning

`createEngramBackend` (`packages/memory/src/engram-backend.ts`) afina su uso de `mem_search` de engram para mejorar recall, sin cambiar cómo se lanza engram:

- **`match_mode: 'any'`** en `query()`: engram soporta `match_mode` `"all"` (AND sobre tokens, default FTS5) o `"any"` (OR); `query()` pasa `'any'` para ampliar el recall de consultas multi-palabra (matchea entradas con CUALQUIERA de los términos, no todas).
- **Límite honesto (`ENGRAM_MAX_SEARCH_RESULTS = 20`)**: engram clampa `limit` a `MaxSearchResults = 20`; `load()` pedía `50` (intención muerta, nunca devolvía más de 20) y ahora pide el tope real `20` vía la constante nombrada.
- **Recall con filtro de kind client-side**: `query.kinds` es un post-filtro client-side (engram no tiene filtro de kind nativo); cuando está activo, `query()` pide el tope `20` a engram (pool máximo de candidatos) y re-aplica el `limit` del llamador con un `slice` DESPUÉS de filtrar, para no perder recall tras el post-filtrado.

### Metadata de fase en memoria y Knowledge Harvest — INC-20260729-knowledge-*

La capa de memoria gana trazabilidad de fase SDD y un comando de harvest para clasificar conocimiento entre fases:

- **Metadata de fase en `MemoryEntry`** (`INC-20260729-knowledge-phase-metadata`): 7 campos opcionales (`increment`, `phase`, `actorRole`, `knowledgeKind`, `stability`, `visibility`, `sourceArtifact`) + 5 tipos cerrados (`SddPhase`, `ActorRole`, `KnowledgeKind`, `Stability`, `Visibility`). En el backend engram, la metadata se codifica como frontmatter YAML-like al inicio del `content` y se decodifica en lectura vía `mem_get_observation`. El backend in-memory la preserva vía JSON. Módulo `phase-metadata.ts` con helpers `encodePhaseMetadata`/`decodePhaseMetadata`.
- **`axiom knowledge harvest --increment <id>`** (`INC-20260729-knowledge-harvest-command`): comando read-only que lee memorias del proyecto, filtra por `increment`, clasifica por `stability` y genera `knowledge-harvest.md`. No requiere engram (funciona con backend JSON de fallback). `--dry-run` imprime en stdout.
- **Contrato de memoria en skills** (`INC-20260729-knowledge-skill-contract`): documentado en [manuales/13_Skills_Agentes_y_Roles.md](manuales/13_Skills_Agentes_y_Roles.md) §"Contrato de memoria Engram por fase".
- **`axiom knowledge sync` y `axiom knowledge pull`** (`INC-20260729-knowledge-sync-command`): comandos que sincronizan la memoria de Engram entre miembros del equipo vía Git en `<project>.axiom`. `sync` exporta chunks JSON versionables a `.engram/chunks/` + `manifest.json` (append-only, sin merge conflicts) y hace git add/commit/push. `pull` hace git pull e importa chunks nuevos a la BD local vía `saveMemory`. `.engram/engram.db` está gitignored. Incluye validación anti-secretos y filtro de `visibility: 'private'`.

La memoria compartida entre fases se logra vía MCP stdio al mismo proceso local de Engram. Entre miembros del equipo, se logra vía `axiom knowledge sync/pull` + Git en `<project>.axiom`. El harvest propone promociones a contexto técnico y skills; la promoción real (`axiom knowledge promote`) queda como follow-up diferido.

La selección de providers es siempre project-scoped: `readEnabledProviders`
recibe `projectKey` y aliases legacy cuando el caller los conoce; el fallback
de scan global solo aplica a callers sin identidad. En worktrees, la identidad
obligatoria es `Execution.projectId`, para impedir que dos proyectos vecinos
compartan accidentalmente un `workspace.json`.

## Integraciones externas opcionales (post-MVP)

`@axiom/app` (portable operator app, incremento `0030`) expone un sistema de plugins project-scoped con discovery en `.axiom-state/<projectKey>/app-plugins/*.json` y un bridge declarativo hacia Azure DevOps (mutación externa exige `confirmed: true`; los plugins no introducen lógica de negocio paralela). No hay integración con Jira/Confluence implementada — solo mencionada como posible extensión en el roadmap futuro (`INC-16`).

## Capacidad clave

El runtime debe saber dónde está cada repo del proyecto y cómo resolverlo en cada surface soportada. Implementado hoy dentro de un único proyecto vía `topology.yaml`. **SUPERSEDE** la afirmación previa de que la resolución de "en qué repo Axiom vive cada pieza del propio producto" (Axiom / Axiom.SDD / Axiom.Spec) era puramente manual (gobernada solo por `Axiom.SDD/AGENTS.md`, no por una topología ejecutable): desde `INC-20260710-dogfooding-workflow-configs`, el propio repo `Axiom` ship un `axiom.config/topology.yaml` real (`mode: multi-repo`, `sddRepo` → `../Axiom.SDD`, `specRepo` → `../Axiom.Spec`, con `Axiom` mismo declarado como `roleCodeRepository` asignado al profile `builder`), por lo que `axiom topology show/validate` ya reflejan esa topología de forma ejecutable — ver la subsección de workflows/topología dogfooded más abajo.

### Workflows (`workflows.yaml`) y topología (`topology.yaml`) dogfooded en `Axiom` — INC-20260710-dogfooding-workflow-configs

Ambos ficheros son canónicos pero **OPCIONALES a nivel de archivo**, siguiendo el mismo patrón que `profiles.yaml`/`DEFAULT_PROFILES` (`BUG-20260703-configure-needs-bundled-profiles`): si el proyecto no tiene su propio `axiom.config/workflows.yaml`, `@axiom/workflow` cae a un `DEFAULT_WORKFLOWS` embebido (`packages/workflow/src/default-workflows.ts`) con los 5 workflows canónicos (`increment`, `bug`, `plan`, `role`, `qa-e2e`) derivados 1:1 de los 5 comandos CLI que los consumen; `axiom.config/topology.yaml` ya tenía este mismo comportamiento vía `@axiom/topology`'s `defaultSingleRepoManifest`. Un fichero PRESENTE siempre gana sobre el fallback, y si está presente pero mal formado sigue siendo un error tipado — el fallback nunca tapa un archivo roto, solo uno ausente.

- **Antes de este incremento**, ninguno de los dos ficheros existía en `Axiom/axiom.config/`, por lo que `axiom-increment`/`axiom-bug`/`axiom-plan approve`/`axiom-role`/`axiom-qa-e2e` fallaban en vivo con "No se encontró workflows.yaml..." y `axiom topology show` reportaba silenciosamente `mode: single-repo` (el default pre-archivo).
- **Ahora**, `Axiom/axiom.config/workflows.yaml` y `Axiom/axiom.config/topology.yaml` existen y son idénticos en contenido al fallback embebido correspondiente (verificado por un test de sync exacto en `packages/workflow/tests/default-workflows.test.ts`).
- **`@axiom/doctor` TC-014/TC-015** (categoría `workflow-config`) verifican que, SI cualquiera de los dos ficheros existe, sea válido — ausente sigue siendo `pass` (fallback embebido); presente y mal formado es `fail`. Complementan (no reemplazan) a TC-001/TC-002, que cubren cobertura de roles y antes absorbían un manifest inválido en un `skip` silencioso en vez de un `fail`.
- **`topology.yaml`'s `assignments[].roleId`** usa el id de functional profile `builder` (de `profiles.yaml`), no un id de rol de implementación (`backend`/`frontend`/`qa-e2e`) — ese mapeo de profile-a-rol-de-implementación queda diferido a un incremento posterior.

## Registro histórico: primera capa de herramientas MCP (roadmap de rediseño, cerrado)

> Este bloque documenta la primera entrega de 17 capability ids y no representa el conjunto completo actual. El registro vigente contiene 25 ids e incorpora el broker `axiom`; los brokers `sdd`, `spec` y `memory` siguen existiendo por compatibilidad.

Resuelve la pregunta de arquitectura Q4: el sistema existente `@axiom/capability-model` / `@axiom/tool-routing` / `@axiom/isolation` se mantiene **completamente ortogonal** a los documentos de decisión MCP más nuevos — las herramientas MCP se añaden como `capabilityId`s nuevos y aditivos, despachados a través del mecanismo `routeTool` existente y sin modificar, nunca como un sistema paralelo o de reemplazo.

- **`@axiom/mcp-tools`** se sitúa por encima tanto de `@axiom/tool-routing`/`@axiom/capability-model` (cero dependencias de paquetes de dominio) como de los paquetes de dominio (`@axiom/workflow`, `@axiom/skills`, `@axiom/technical-context`, `@axiom/user-workspace`, `@axiom/cli-commands`). Registra un mapa `capabilityId -> handler` (`MCP_TOOL_HANDLERS`, `invokeMcpTool(capabilityId, input)`) y nunca añade una dependencia hacia ninguno de los paquetes por debajo de él.
- **17 capability ids registrados** (dominios `sdd.*`/`spec.*`): `sdd.projectRegistryRead`, `sdd.projectReposRead`, `spec.planRead`, `spec.incrementRead`, `spec.bugRead`, `spec.adrIndexRead`, `spec.decisionIndexRead`, `sdd.skillIndexRead`, `spec.skillCatalogRead`, `spec.skillRead`, `spec.technicalContextIndexRead`, `spec.recommendedContextList`, `sdd.allowedWriteScopeRead`, `sdd.changesValidate`, `sdd.indexesRebuild`, `sdd.indexesValidate`, `spec.implementationContextRead`. Los primeros 16 son cada uno una traducción delgada de una función existente y sin modificar — no se escribió lógica de negocio nueva para ninguno de ellos.
- **`spec.implementationContextRead`** (`getImplementationContext`/`buildImplementationContext`) es la lectura compuesta "insignia": una composición sobre los 16 handlers hermanos, no una dependencia de paquete de dominio nueva. Forma de la respuesta: `project`, `repositories{spec,sdd,target}`, `plan`, `relatedSpec`, `relatedAdrs`, `mandatory{sddSkills,repoSkills,technicalContext,rules,commands}`, `indexes{...}`, `recommended{skills,technicalContext,adrs,commands}`, `allowedWriteScope`, `confidence` (`high`/`medium`/`low`), `missingMetadata`.
  - `relatedSpec` es un **placeholder**, no una herramienta `get_related_specs` completa: solo resuelve `plan.links.incrementId` -> `getIncrement`.
  - `contextBudget` (`small`|`medium`|`large`, `small` por defecto) controla solo el inlining de contenido, nunca la presencia de campos: `small` devuelve solo referencias; `medium` incluye el contenido de `plan`+`relatedSpec`+`mandatory.*`; `large` añade además el contenido de `relatedAdrs`. `recommended.*` nunca incluye contenido en ningún nivel.
  - Resolución de `repositories.target`: gana el `input.targetRepoId` explícito; si no, si queda exactamente una entrada de repo tras excluir las claves `spec`/`sdd`, esa es `target`; si no, `target` es `null` y se marca en `missingMetadata` — un no-adivinar deliberado para proyectos con múltiples repos target: el llamador debe pasar `targetRepoId` explícitamente en ese caso.
  - `mandatory.rules`/`mandatory.commands` (y sus contrapartes en `indexes`/`recommended`) están permanentemente vacíos/ausentes: no existe ningún tipo de artefacto "rules" o "commands" que los respalde. `indexes.commands`/`indexes.rules` llevan `unsupported: true` para distinguir este hueco de producto permanente de un caso de "índice no encontrado esta vez" por proyecto.
- **Mapeo histórico de proveedor `sdd`/`spec` -> `axiom-gateway`**: la allowlist MCP de `@axiom/isolation` (`sdd`/`spec`/`serena`) y los `CANONICAL_PROVIDER_IDS` de aquella versión (`filesystem`, `axiom-gateway`, `serena`, `codegraph`, `graphify`, `engram`, `generated-snapshots`) eran listas independientes. El registry vigente de 4 ids locales y la sustitución por `cmm` prevalecen sobre esta fotografía.
- **`@axiom/cli-commands`** es el seam establecido para exponer las funciones `runX` de los comandos compartidos a cualquier consumidor que no deba importar fuentes app-side directamente. Publica `dist/index.js` y `dist/index.d.ts`, mantiene ownership único y es consumido por CLI, launcher y MCP; no es una dependencia de una interfaz TUI.
- **Checks `TR-001..004` de `@axiom/doctor`** hacen smoke-test de `routeTool` vía un fixture en memoria; no dependen de ningún capability id específico. Extender la cobertura de doctor al registro de capability/provider de `@axiom/mcp-tools` (`TR-005`+) se consideró y quedó diferido — ningún installer/scaffold escribe hoy un `capabilities.yaml`/`providers.yaml` real a ningún proyecto, así que no hay call site de producción vivo que comprobar.

### Technical-context index GENERADO (deja de servir `null`) — INC-20260710-technical-context-served

**Supersede** la afirmación implícita anterior de que `spec.technicalContextIndexRead`/`spec.recommendedContextList` "ya funcionan" simplemente porque el loader (`@axiom/technical-context`) existe: antes de este incremento NINGÚN `technical-context/indexes/<roleOrKind>.index.yml` existía nunca en la práctica (curado a mano o de otro modo), así que ambas tools devolvían `null` para cualquier proyecto real, incluido `Axiom.Spec` mismo — el contexto técnico curado a mano en `context/**/*.md` existía, pero nunca llegaba a un agente vía MCP.

- **Generador nuevo (`@axiom/technical-context`'s `generateTechnicalContextIndex`/`writeGeneratedTechnicalContextIndex`, `packages/technical-context/src/generate-index.ts`)**: escanea `<specRepo>/context/**/*.md` y produce un `TechnicalContextIndex` válido — `TECHNICAL_CONTEXT.md` promovido a `mandatory.always` (hecho estructural, el propio documento se autodescribe como puerta de entrada, no un juicio de curación), el resto listado en `available[]` con `tags` derivados de la subcarpeta y `summary` desde el primer heading, `status: 'draft'` siempre. Expuesto vía `axiom context index [--role <r>] [--path <specRepo>]`, un SUB-COMANDO NUEVO del `axiom context` ya existente (no un rename, no un comando top-level nuevo) — `axiom context --help` ahora distingue explícitamente sus dos concerns (`refresh`/`status` = install profile; `index` = technical-context index MECÁNICO). NO confundir con `axiom index rebuild/validate` (otro índice, de increments/bugs/plans) ni con `axiom bootstrap from-code`'s propio ensamblado de `TechnicalContextIndex` (ese redacta documentos NUEVOS desde código; este generador solo cataloga documentos que YA existen bajo `context/`).
- **`Axiom.Spec` ahora dogfoodea un índice real**: `Axiom.Spec/technical-context/indexes/repo.index.yml` (role/repoKinds `repo`, generado por el comando anterior) referencia los 11 documentos reales de `context/` (1 mandatory + 10 available).
- **Fix de resolución de path en `spec.implementationContextRead`**: `inlineContent` (`packages/mcp-tools/src/implementation-context-handler.ts`) resolvía cada `path` de `mandatory.technicalContext` contra `specRepoRoot` directamente, pero el contrato documentado del índice (`TECHNICAL_CONTEXT_INDEX_RELATIVE_DIR`) dice que `path` es relativo al directorio DEL PROPIO ARCHIVO DE ÍNDICE (`<specRepo>/technical-context/indexes/`) — un bug latente independiente del `null` anterior, que hubiera impedido el inlining correcto incluso con un índice presente. Corregido: `baseDir` ahora es el directorio del índice; se agregó además un GUARD de scope (`guardRoot`, default `specRepoRoot`) que descarta (con diagnóstico, nunca lee) cualquier `path` cuyo `..` resuelva fuera del spec repo.
- **Confirmado en vivo**: `tools/call spec.technicalContextIndexRead {roleOrKind:"repo"}` contra `Axiom.Spec` pasó de `null` a devolver el índice real; `spec.recommendedContextList` idem; `spec.implementationContextRead` (contextBudget `medium`) inlinea el contenido real de `TECHNICAL_CONTEXT.md` en `mandatory.technicalContext[0].content`.

### Endurecimiento de correctitud 2026-07-11 (technical-context + toolchain) — INC-20260711-audit-bug-fixes

- **Fallback de role-key en `spec.implementationContextRead`**: cuando el lookup del índice de technical-context para el `role` del plan devuelve `null`, ahora reintenta UNA vez con la clave por defecto del generador (`DEFAULT_TECHNICAL_CONTEXT_ROLE = 'repo'`) antes de declarar el contexto ausente — así un proyecto que solo generó `repo.index.yml` (como `Axiom.Spec` mismo) sí sirve technical-context a un plan de rol `backend`, en vez de devolver TC vacío por falta de índice por-rol. (`packages/mcp-tools/src/implementation-context-handler.ts`.)
- **`toolchain validate` avisa por tool opcional declarada-pero-ausente**: el loop de tools declaradas emitía solo drift; ahora emite un WARNING (nunca error — la semántica de exit-code se preserva, los errores siguen reservados a tools required/mvp) para cualquier tool no-required declarada cuyo estado sea `absent` (`optional-tool-absent`) o `marker`/`declared` cuando hay detecciones reales (`optional-tool-unverified`). Complementa los estados diferenciados del toolchain descritos en [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md).

### Server MCP ejecutable (`@axiom/mcp-server` + `axiom mcp serve`) — INC-20260708-mcp-runnable-server / INC-20260708-mcp-launch-config-wiring

**Supersede** la afirmación previa de que `sdd-mcp-server`/`spec-mcp-broker` eran solo identificadores string sin proceso servidor ejecutable detrás. El paquete `@axiom/mcp-server` implementa un server MCP real que habla JSON-RPC 2.0 sobre stdio delimitado por saltos de línea y expone los handlers de `@axiom/mcp-tools`. Se lanza con el comando CLI `axiom mcp serve --kind <sdd|spec|memory|axiom> --project-root <path> [--home-dir <path>]` (ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md)).

- **Protocolo**: `createMcpServer` es un dispatcher JSON-RPC 2.0 puro y transport-agnostic que atiende `initialize` (protocolVersion + `capabilities.tools` + `serverInfo`), `notifications/initialized` (sin respuesta), `tools/list`, `tools/call` (despacha a `invokeMcpTool` vía un input-builder por capability y envuelve el `ok`/`err` en `content`/`isError`), `ping`, y devuelve el error JSON-RPC `-32601` para métodos desconocidos. `runStdioServer` es el bucle de transporte stdio (líneas JSON delimitadas por `\n`, tolerante a líneas en blanco, `-32700` en errores de parseo con id recuperable).
- **Conjunto de herramientas por `--kind`**: `sdd` expone las capabilities `sdd.*`, `spec` las `spec.*`, `memory` las capabilities reales de memoria y `axiom` el broker unificado con la unión acotada de lecturas de spec, las dos escrituras SDD confirmadas y las lecturas de plano de proyecto. El registro global contiene 25 capability ids; el kind no concede acceso a capabilities de otro dominio por accidente.
- **Resolución de contexto**: `resolveMcpServerContext({ projectRoot, homeDir? })` deriva `{ homeDir, projectRoot, projectId, specScopeAbsolutePath, role }` a partir de las primitivas existentes (`@axiom/project-resolution`, `@axiom/user-workspace`, `@axiom/topology`), siempre best-effort (nunca lanza ante datos de registro/topología ausentes). Los campos que no pueda inferir (`projectId`, `role`) quedan `undefined`, y las capabilities que los requieran devuelven un `tools/call` con `isError: true` (nunca un throw).
- **Config de lanzamiento en `mcp.yml` (INC-20260708-mcp-launch-config-wiring)**: en aquella primera entrega los dos servers que `runWorkspaceSetup` generaba llevaban un comando de lanzamiento real: `command: 'axiom'` + `args: ['mcp', 'serve', '--kind', <sdd|spec>, '--project-root', <path absoluto del repo>]`. La forma vigente añade los kinds `memory` y `axiom`; los campos `command`/`args` siguen siendo opcionales y aditivos en `McpServerEntry`.
- **Config MCP nativa por herramienta (SUPERSEDE el caveat previo — INC-20260708-mcp-native-config-mapping)**: el caveat honesto que decía que mapear los documentos MCP generados por Axiom al schema NATIVO de config MCP de cada herramienta destino seguía pendiente **está ahora RESUELTO**. La ruta de setup de workspace ya emite, por cada repo × por cada adapter seleccionado, el fichero de config MCP en el **schema nativo real** que cada herramienta lee para auto-lanzar sus servers MCP — ver la sección "Config MCP nativa por herramienta destino" más abajo. Ya no se escribe el antiguo formato custom-shape (`{schemaVersion, projectId, servers:[...]}`) de `.opencode/mcp.json`/`.claude/mcp.json` desde la ruta de workspace; `.axiom/mcp.yml` permanece como la fuente canónica de Axiom, sin cambios.
- **Sin dependencias npm nuevas**: el framing JSON-RPC/stdio es hecho a mano (no `@modelcontextprotocol/sdk`), minimalista pero suficientemente correcto para que un cliente MCP estándar haga `initialize` + `tools/list` + `tools/call`.
- **Aislamiento project-scoped enforced (INC-20260708-mcp-project-isolation-hardening)**: cada server MCP está atado a UN proyecto al arrancar y ahora **hace cumplir** ese binding — solo lee/opera sobre SU propio proyecto. Todo campo que identifica proyecto en los argumentos de `tools/call` (`projectId`/`homeDir`/`projectRoot`/`specScopeAbsolutePath`/`sddScopeAbsolutePath`) se fija al `context` del server; un valor del llamador que apunte a OTRO proyecto se rechaza con `isError: true` en vez de aceptarse, y `sdd.projectRegistryRead` devuelve únicamente la entrada del proyecto propio (nunca el registro machine-wide). Los selectores dentro-de-proyecto (`id`, `planId`, `skillId`, `specRelPath`, `role`, `taskTags`, `targetRepoId`, …) siguen siendo caller-supplied. Detalle completo (framing como garantía de ownership/aislamiento, paridad con GATE 0024 de `@axiom/memory`) en [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md).

### Config MCP nativa por herramienta destino — INC-20260708-mcp-native-config-mapping

> **Actualizado por `ACC-027`/`ACC-029` (2026-08-10):** la tabla de abajo describe el despacho de la entrega INC-20260708. El contrato vigente (ver `Axiom.Spec/context/architecture/04-adapters-y-model-routing.md`) difiere en dos filas: `visual-studio-2026` **no recibe fichero MCP** (schema/path no verificado; no se escribe `.vs/mcp.json`) y `litellm` fue **retirado** del contrato activo. El resto de filas se mantiene.

**Supersede el "caveat honesto (pendiente)"** de la sección anterior: mapear las dos entradas MCP de Axiom (`sdd-mcp-server`, `spec-mcp-broker`) al schema NATIVO de config MCP de cada herramienta destino ya **NO** es un follow-up abierto — está entregado. `runWorkspaceSetup` emite ahora, para cada entrada MCP de Axiom `{ id, command:'axiom', args:[...], env? }`, el fichero de config nativa REAL que un install real de cada herramienta lee para auto-lanzar sus servers, sustituyendo al antiguo `.opencode/mcp.json`/`.claude/mcp.json` de forma custom-shape que la ruta de workspace ya no escribe. La lógica vive en el módulo nuevo `apps/cli/src/commands/native-mcp-config.ts` (`writeNativeMcpConfig({ target, repoRoot, servers })`), app-layer junto a `workspace-mcp.ts`/`workspace-adapters.ts`.

- **Tabla fichero + schema nativo por target** (despacho INC-20260708; ver nota de actualización arriba):

  | Target | Fichero nativo (relativo al repo) | Schema |
  |---|---|---|
  | `claude-code` | `.mcp.json` | `{ "mcpServers": { "<id>": { command, args, env? } } }` |
  | `cursor` | `.cursor/mcp.json` | igual que claude-code (`mcpServers` + `{ command, args, env? }`) |
  | `copilot-vscode`, `github-copilot`, `vscode` | `.vscode/mcp.json` | `{ "servers": { "<id>": { type:'stdio', command, args, env? } } }` (clave `servers`, `type:'stdio'` obligatorio) |
  | `opencode` | `opencode.json` | `{ "$schema": "https://opencode.ai/config.json", "mcp": { "<id>": { type:'local', command:[cmd,...args], enabled:true, environment? } } }` (`command` es un solo array `[command, ...args]`; `env` renombrado `environment`) |
  | `visual-studio-2026` | `.vs/mcp.json` | `{ "servers": { "<id>": { type:'stdio', command, args, env? } } }` (asunción documentada y override-able) — **retirado por ACC-027** |
  | `codex` | — (ninguno) | nota user-global `~/.codex/config.toml`, sección `[mcp_servers]` |
  | `antigravity` | — (ninguno) | nota user-global `~/.gemini/config/mcp_config.json`, clave `mcpServers` |
  | `litellm` | — (ninguno) | warning genérico: no se inventa un schema MCP — **retirado por ACC-025** |

  `env` se omite cuando está vacío; en opencode se emite como `environment`.

- **Merge-preserving**: cada writer parsea el fichero nativo preexistente si lo hay, reemplaza **solo** las entradas Axiom-managed (keyed por id) dentro del mapa de servers de esa herramienta, y **preserva verbatim** cualquier otra key/server ya presente (entradas de servers del propio usuario + keys top-level ajenas sobreviven). Crea un fichero mínimo válido cuando el nativo está ausente. Escritura **atómica** (tmp+rename, reusando `atomicWriteFile` de `init.ts`). Un fichero nativo preexistente con JSON **malformado** se trata como best-effort-failed para ese repo/adapter: warning + fichero dejado intacto (nunca se clobbera).
- **Por repo × por adapter, en TODO el workspace**: `writeWorkspaceNativeMcpConfigs` (nueva función en `workspace-mcp.ts`) recorre CADA repo del workspace (control + spec + cada repo de rol) × CADA adapter seleccionado y llama a `writeNativeMcpConfig`, acumulando `filesCreated`/`warnings`. Así un dev que abra CUALQUIER repo con CUALQUIERA de las herramientas seleccionadas ve los servers. Best-effort estricto por repo/adapter: un fallo nunca aborta el setup.
- **`.axiom/mcp.yml` sigue canónico y sin cambios**: `writeWorkspaceMcpConfig` ahora **solo** escribe `.axiom/mcp.yml` (fuente canónica de Axiom, `McpProjectConfig` con `command`/`args` de lanzamiento reales, ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)); ya **no** llama a `generateOpencodeMcpJson`/`generateClaudeCodeMcpJson`. Los generadores custom-shape antiguos **permanecen** exportados en sus paquetes y cubiertos por sus propios tests — solo dejó de llamarlos la ruta de workspace-setup. El e2e (`workspace-mcp.e2e.test.ts`) ahora **spawnea los servers leyendo los args del `.mcp.json` NATIVO**, probando que la config nativa es realmente ejecutable (initialize + tools/list + tools/call + stdio JSON-RPC).
- **Seam de build**: el runtime del server vive en `packages/mcp-server/`; el comando `axiom mcp serve` vive en un fichero app-owned (`apps/cli/src/commands/mcp-serve.ts`, no en el `mcp.ts` compilado por `@axiom/cli-commands`) para evitar un ciclo real de project-references (`cli-commands -> mcp-server -> mcp-tools -> cli-commands`).

### `mcp.yml` vs `mcp-manifest.yaml`

Ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md) para el schema completo y la tabla comparativa — son dos ficheros con responsabilidades distintas que no deben fusionarse. Ver también, en esa misma sección, la nota sobre la forma REAL committed (minimal) de `mcp-manifest.yaml` vs la forma rica que el reader deriva (`INC-20260710-schema-reconciliation`).

### Catálogo global del toolchain (`axiom.config/toolchain-catalog.yaml`) — INC-20260710-schema-reconciliation + INC-20260730-toolchain-versioning

Fuente canónica dedicada para el catálogo global de IDs de tools permitidas por `axiom toolchain add/show/validate` (`apps/cli/src/commands/toolchain.ts`), consumida también por `@axiom/components` (D1, `packages/components/src/loader.ts`) y por las checks de doctor TC-004/TC-005 (`packages/doctor/src/checks.ts`). Antes de este incremento, los tres lectores derivaban este catálogo de `integrations.yaml#axiomManagedDefaults.install` + `installationDecisions.optionalPostMvp` — pero el `integrations.yaml` real committed (Spec 0013 PC-001) solo declara la allowlist MCP (`mcp_servers`/`allowed_tools`), nunca esas claves, así que el catálogo quedaba vacío en el propio repo `Axiom/` (`axiom toolchain add --id serena` fallaba con "IDs disponibles: []").

- **Schema vigente**: `{schemaVersion: 2, tools: [{id, kind, mvp, displayName, supportLevel, versionExtractor?, channels?, compatibility?}, ...]}`. `kind` sigue las convenciones de `deriveKindForId`; `mvp: true` marca una tool como parte del subset P0 requerido (fuente de `catalogue.managed`), `mvp: false` como declarada/opcional (fuente de `catalogue.optional`). `versionExtractor` define la regex con la que `@axiom/toolchain` extrae la versión de la salida del probe; `channels` fija las versiones de política `stable`/`candidate`/`edge`; `compatibility.axiomMinVersion` declara la versión mínima de Axiom compatible.
- **Versionado reproducible**: las tools con contrato local de versión pueden fijar una entrada en `.axiom-state/<projectKey>/toolchain.lock`, con `id`, `version`, `channel` y metadatos opcionales del probe (`probeCommand`, `probeOutput`, `probedAt`). `axiom toolchain plan` calcula el diff entre el lockfile y el canal solicitado; `axiom toolchain upgrade --yes` actualiza únicamente ese lockfile con checkpoint y rollback ante fallos de persistencia o probe posterior. El catálogo autoriza IDs y versiones, pero no da de alta implícitamente todas sus entradas. No instala, descarga ni sustituye binarios externos.
- **Set inicial** (todas `mvp: false`, ninguna requerida por defecto): `serena`, `cmm` (`code-intelligence`); `engram`, `context7` (`library-context`); `rtk` (`input-optimizer`); `caveman` (`output-optimizer`); `autoskills` (`skill-surface`). `codegraph` y `graphify` no forman parte del catálogo vigente.
- **Fallback legacy preservado**: si `toolchain-catalog.yaml` no existe, los tres lectores caen a la lectura previa de `integrations.yaml#axiomManagedDefaults`/`installationDecisions.optionalPostMvp` byte-a-byte idéntica (back-compat: ningún test ni instalación existente que ya declare esas claves se rompe).
- **`integrations.yaml` (Spec 0013 PC-001) NO se toca ni se repropone**: su responsabilidad sigue siendo exclusivamente la allowlist MCP (`mcp_servers`/`allowed_tools`); es una fuente NO relacionada que, antes de este incremento, tres lectores distintos consultaban por una convención de nombres de clave heredada, no porque fuera su responsabilidad declarada.
- **`deriveKindForId` extendido**: además del sufijo project-scoped tradicional (`"serena-this-project"`), ahora también reconoce el id "bare" del catálogo global (`"serena"`). Sin esta extensión, ids como `engram`/`context7`/`rtk`/`caveman`/`autoskills` habrían caído silenciosamente al `kind` default `'code-intelligence'` en vez de su kind real.

### Registro histórico: generación de config MCP durante el setup de workspace (SDD MCP + Spec MCP) — INC-20260705-workspace-mcp-generation

> Este bloque conserva la evolución del primer escritor de MCP. La forma vigente de la configuración nativa, la paridad de targets y el auto-cableado de code-intel se describen en las secciones actualizadas de este capítulo y en la tanda de paridad de 2026-07-26.

El setup de workspace multi-repo (`runWorkspaceSetup`, ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)) genera, como **paso best-effort final tras un registro exitoso**, la config MCP por proyecto. Es el primer y único call site que escribe un `mcp.yml` real a un proyecto (`apps/cli/src/commands/workspace-mcp.ts`, funciones `buildWorkspaceMcpServers` + `writeWorkspaceMcpConfig`).

- **`.axiom/mcp.yml`** (`McpProjectConfig`, `schemaVersion: 1`) se escribe en el repo de control, con exactamente dos entradas `McpServerEntry`, ambas `enabled: true`, `scope: 'repo'`:
  - `sdd-mcp-server` (SDD MCP) con `targetRepo` = la `roleKey` de registro del repo de control;
  - `spec-mcp-broker` (Spec MCP) con `targetRepo` = la `roleKey` de registro del repo de spec.
  
  En aquella versión, ambas entradas llevaban un comando de lanzamiento real (`command: 'axiom'` + `args: ['mcp','serve','--kind',<sdd|spec>,'--project-root',<path absoluto>]`). Los `targetRepo` referencian claves del mapa de repos del registro v2 (no paths); la forma vigente y sus cuatro kinds se describen arriba. En el caso degenerado de que control y spec colapsen a la misma `roleKey` (workspace single-repo), solo se emite `sdd-mcp-server` más un warning, evitando un par `duplicate-type-target-repo` inválido.
- **Proyección a cada adapter MCP-capaz seleccionado**: la forma vigente usa el schema MCP nativo por adapter y la matriz completa de targets está en la sección "Config MCP nativa por herramienta destino".
- **Semántica best-effort**: la generación solo corre si el registro tuvo éxito (`register !== false` y `registryRegistered: true`); si se saltó o falló, la generación MCP se salta con un warning explicativo. El paso está envuelto en try/catch que nunca escala: los fallos (incluidos issues de validación) se acumulan en `warnings`, nunca abortan el setup; `.axiom/mcp.yml` se escribe aunque la validación reporte issues (validate-after-write).
- **Caveat de gitignore**: `.axiom/` está gitignoreado, así que `mcp.yml` es un artefacto machine-local (no versionado) — coherente con que declara qué procesos servidor MCP están habilitados localmente para que los adapters generen config runtime.

Esto no toca `axiom configure`/`sync`, `GENERATED_FILES_BY_TARGET` ni el deferral TR-005 para la ruta no-workspace; tampoco emite entradas MCP para repos de rol/código (solo SDD + Spec MCP).

### Registro histórico: generación multi-adapter en todos los repos del workspace — INC-20260705-workspace-adapters-multiselect

> La tabla siguiente es la fotografía de la primera entrega y conserva el dato histórico de ocho targets. El despacho vigente (actualizado por `ACC-025..ACC-029`) cubre **8 targets activos** y **8 paquetes dedicados** (`copilot-vscode` es alias legacy de `github-copilot` y `litellm` fue retirado); prevalece la sección "Paridad de alcance de adapters + launcher onboarding" y el detalle canónico en `Axiom.Spec/context/architecture/04-adapters-y-model-routing.md`.

Tras el registro y la generación MCP, `runWorkspaceSetup` materializa, como paso best-effort, los ficheros de **cada adapter seleccionado** (`spec.adapters`, ver el wizard en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md)) en **cada repo del workspace** — control, spec y cada repo de rol. Antes de este incremento el motor nunca escribía la salida real de ningún adapter en los repos scaffoldeados (solo `.axiom/mcp.yml` + su proyección); esta subsección supersede esa limitación. La lógica vive en `apps/cli/src/commands/workspace-adapters.ts` (`generateWorkspaceAdapters`).

- **Un `installProfile` por repo**: para cada repo se resuelve un único `ResolvedInstallProfile` (vía `installProfile`, usando `adapters[0]` como adapter primario y `DEFAULT_PROFILES` de `@axiom/install-profiles` como `profilesData` de fallback), y ese perfil resuelto se reutiliza en cada generador de adapter de ese repo.
- **Tabla de despacho `target -> generador` de aquella entrega** (snapshot histórico; no es el inventario vigente):

  | Target | Generador / escritura |
  |---|---|
  | `opencode` | `generateOpencodeConfig` (`.opencode/AGENTS.md`; el custom-shape `.opencode/mcp.json` que producía queda superseded por la config MCP nativa `opencode.json` — ver abajo) |
  | `claude-code` | `generateClaudeCodeConfig` (`.claude/AGENTS.md`; el custom-shape `.claude/mcp.json` queda superseded por la config MCP nativa `.mcp.json` — ver abajo) |
  | `vscode` | `generateVscodeConfig` |
  | `cursor` | `generateCursorConfig` |
  | `github-copilot` | `generateGithubCopilotConfig` |
  | `litellm` | `generateLitellmConfig` |
  | `copilot-vscode` | `writeCopilotInstructions` (`.github/copilot-instructions.md` + settings/extensions VS Code) |
  | `antigravity` | escritura canónica `AGENTS.md` (sin generador dedicado) → `.antigravity/AGENTS.md` |
  | `visual-studio-2026` | escritura canónica `AGENTS.md` (sin generador dedicado) → `.vs/AXIOM.md` |

- **`antigravity`/`visual-studio-2026` vía el escritor canónico AGENTS.md**: al no tener adapter dedicado, su fichero declarado en `GENERATED_FILES_BY_TARGET` se escribe con `renderCanonicalAgentsMd` + `writeGuardedFile`/`writeCanonicalAgentsMd` (un `AGENTS.md` canónico delgado, no un formato inventado más rico).
- **Best-effort estricto**: cada llamada a un generador va envuelta; un fallo (input faltante, `Result` en error) se acumula como warning por repo/adapter y nunca aborta el setup global.

### Plantillas de adapter bundleadas (contenido real de instrucciones) — INC-20260705-workspace-adapter-templates

La fotografía de este incremento histórico describía el hueco que existía
cuando los repos recién scaffoldeados dependían de templates solo presentes en
`Axiom.Spec/templates/` y podían degradar a `template-missing`. El contrato
vigente conserva el bundle para que `tsc -b` no dependa de assets no-TS, pero la
propiedad está separada: `AGENTS_MD_TEMPLATE` sigue siendo responsabilidad de
`workspace-adapter-templates.ts` para OpenCode/Claude Code, mientras
`COPILOT_INSTRUCTIONS_TEMPLATE` y `resolveCopilotInstructionsTemplate` viven
en `@axiom/document-bootstrap` y son compartidos por `configure`, `sync`,
`workspace setup` y ambos targets de Copilot (ACC-024).

Los templates on-disk tienen precedencia cuando son legibles; el bundle es el
fallback para proyectos adoptados o recién creados. La salida de Copilot usa
`.github/copilot-instructions.md`, preserva las zonas humanas y no trata la
ausencia del template on-disk como `template-missing` por sí sola. La historia
de este incremento no debe leerse como una afirmación vigente de que el bundle
de Copilot pertenece a `workspace-adapter-templates.ts`.

### Baseline de Axiom SKILLS scaffoldeada en el repo SDD recién creado — INC-20260705-workspace-sdd-skills

Cuando `runWorkspaceSetup` **crea desde cero** el repo de control/SDD (`created === true`), scaffoldea una baseline de SKILLS de Axiom en él como capacidad de agente, usando la API real de `@axiom/skills` (`loadSkillRegistry`, `applySkillSet`, `loadSkillsRoleIndex`/`validateSkillsRoleIndex`, `computeSkillBundleHash`). La lógica vive en `apps/cli/src/commands/workspace-skills.ts` (`scaffoldSddSkills`). Es best-effort, gateada y exclusiva del repo de control — las skills son una preocupación del repo SDD, no del repo de spec ni de los repos de rol.

- **Catálogo semilla bundleado**: `axiom.config/skills-catalog.yaml` (`schemaVersion: 1`) sembrado con un conjunto canónico pequeño de 5 ids — 3 ids canónicos de Axiom (`axiom-sdd-orchestrator`, `axiom-context-persistence`, `axiom-capability-router`) más los dos `DEFAULT_DESIRED_SKILLS` (`serena-this-project`, `context7-this-project`), de modo que un `applyDefaultSkillSet` también resolvería contra este catálogo. El contenido fuente de cada skill se bundlea como constantes TS y se escribe bajo `axiom.spec/target-axiom-skills/<id>.md`; el `bundleHash` de cada entrada se computa con `computeSkillBundleHash` (hash byte-exacto de la fuente bundleada), así que el catálogo es internamente consistente.
- **Materialización**: `applySkillSet` materializa `.opencode/agents/<id>/SKILL.md` por skill sembrada y `.axiom-state/<projectId>/skills-pending.json`.
- **Índice de skills por rol**: se escribe un `axiom.config/skills-index/<role>.yaml` por cada rol funcional declarado en el workspace (ids derivados de `role.functionalRoleId ?? role.roleKey` de cada repo `kind: 'role'`), validado con `validateSkillsRoleIndex` antes/después de escribir (warnings, nunca bloqueante). Comparte el schema `SkillsRoleIndex` de RF-AXM-020 (ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md)); `axiom-sdd-orchestrator` + `axiom-context-persistence` se marcan `mandatory` por rol y el resto `available`.
- **Gating + no-clobber**: solo corre si el repo de control se creó recién en esta misma llamada; si ya existía, se salta por completo (nunca clobbera). El paso entero es idempotente y best-effort — cualquier throw de `@axiom/skills` se degrada a un único warning sin abortar el setup.

### Skills también scaffoldeadas en cada repo de código recién creado — INC-20260705-workspace-code-repo-skills

Esto supersede la restricción "sólo el repo de control recibe skills": las skills se scaffoldean ahora en **AMBOS** el repo de control/SDD (vía `scaffoldSddSkills`, arriba, sin cambios) **Y** cada repo de CÓDIGO/rol recién creado (vía la nueva `scaffoldCodeRepoSkills`, `apps/cli/src/commands/workspace-skills.ts`). Tras el paso de skills del repo de control, `runWorkspaceSetup` itera cada repo `kind: 'role'` y, **solo si ese repo se creó recién** (`created === true`), instala en él su propia baseline scoped a su rol.

- **Misma semilla y maquinaria, sin duplicación**: `scaffoldCodeRepoSkills` reusa el mismo pool de catálogo semilla bundleado (5 ids) y las mismas primitivas de `@axiom/skills` (`loadSkillRegistry` + `applySkillSet`) que el repo de control, factorizadas en un helper interno compartido — el contenido de la semilla y la lógica de materialización viven en un solo sitio.
- **Índice de skills scoped al propio rol del repo**: la única diferencia con el repo de control es el `skills-index` — el repo de código escribe UN solo `axiom.config/skills-index/<roleId>.yaml` scoped a su propio rol (posiblemente custom: `role: roleId`, `repoKinds: ['role']`, `roleId = repo.functionalRoleId ?? repo.roleKey`), mientras que el repo de control escribe uno por CADA rol funcional del workspace. Usa el mismo split mandatory/available (`axiom-sdd-orchestrator` + `axiom-context-persistence` mandatory) y la misma validación pre/post-escritura (`validateSkillsRoleIndex`, warnings nunca bloqueantes) que el repo de control.
- **Gating + best-effort por repo**: gateado estrictamente por el `created` propio de cada repo de rol (mismo patrón que los gates de control/spec); un repo de rol preexistente (`create: false`) se salta entero, nunca se clobbera. Cualquier throw se captura por iteración — el fallo de un repo no salta los demás ni aborta el setup. El repo de spec no recibe skills. Las formas de datos por repo de código están en [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md).

### Aprendizaje continuo (módulo `learning.ts` de `@axiom/memory`) — INC-20260708-continuous-learning

`@axiom/memory` gana un módulo `learning.ts` — una capa fina, determinista, sobre el backend de memoria real (AB5) y los registros reales de audit-trail; explícitamente NO un "motor de instintos" especulativo (los conceptos de `homunculus` de ECC — scoring de confianza, herencia, pipelines de promoción — fueron leídos y rechazados por out-of-bootstrap-scope).

- **Tres funciones**: `extractLessons` (PURA, deriva `LessonCandidate[]` de un texto explícito del operador y/o una muestra de registros reales de audit-trail vía heurística determinista de repeat-count — sin ML, sin scoring), `persistLessons` (persiste cada lección como `MemoryEntry` `kind: 'pattern'` tag `'lesson'`, con `topicKey` derivado para upsert-in-place reusando el mecanismo de AB5), `recallLessons` (reusa el `rankEntries`/`buildRecallResult` existente, filtrado a `tags: ['lesson']` — sin algoritmo de ranking nuevo). Sin nuevo `MemoryKind`, sin nuevo backend, sin dependencias npm nuevas.
- **Superficie CLI y de flujo**: `axiom learn capture|list` (app-owned) y el bloque best-effort de "lecciones recientes" en `axiom context status` — ver [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md) y [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md). GATE 0024 se preserva gratis (todo persist/recall pasa por un backend scope-bound). Los hooks de sesión son un snippet `.claude/settings.json` documentado y opt-in, NUNCA auto-aplicado — Axiom no ejecuta ningún motor de hooks de sesión.

### Delegation triggers (señal no-bloqueante) y roster curado de agents de delegación — INC-20260708-delegation-triggers

`@axiom/orchestrator` incorpora `evaluateDelegation` (`packages/orchestrator/src/delegation.ts`): una función PURA que evalúa un `DelegationMetrics` explícito (`filesRead`, `filesTouched`, `sequentialCommands`, `taskScopeSize`) contra umbrales bundled (`DEFAULT_DELEGATION_THRESHOLDS`, configurables) y retorna 0+ `DelegationSuggestion`s, cada una nombrando el agent del roster recomendado (`recommendAgentForTrigger`) y el umbral cruzado. Es una señal de sugerencia inspirada en la disciplina "thin orchestrator, delegate by default" de GentleAI (heurísticas documentadas de Hermes: "broad exploration (4+ files)", "touching 2+ non-trivial files → fresh review").

- **Honestamente no-bloqueante, no solo por convención**: el módulo NUNCA es importado por `gates.ts` ni `runner.ts` — está estructuralmente fuera del control flow del state machine, no puede vetar ni alterar el outcome de ningún command.
- **Sin instrumentación fabricada**: este codebase no tiene una capa de instrumentación de tool-calls en vivo (no hay wrapping de Read/Grep/Bash, no hay contador de sesión). Por eso `DelegationMetrics` es un input EXPLÍCITO que el caller provee — la función es real y testeada, pero no se inventó un contador sintético de "archivos leídos" que no corresponde a nada real hoy. Poblarlo desde un harness/hook real (e.g. Claude Code) queda como integración futura documentada, no implementada acá.
- **Roster curado (5 agents, catalog-driven)**: `packages/agents/src/roster.ts` (`DELEGATION_ROSTER`) referencia 5 ids del catálogo real (`axiom.config/agents-catalog.yaml`) — 4 nuevos (`axiom-explorer`, `axiom-reviewer`, `axiom-security-reviewer`, `axiom-tester`) más el pre-existente `axiom-spec-planner` reutilizado (no duplicado). Bundled con el mismo patrón de "catálogo semilla" que la sección anterior de skills — catálogo YAML + fuentes `.md` reales bajo `axiom.spec/target-axiom-agents/`, materializados por el `materializeAgentSet` sin cambios. El `AgentRole` (cerrado: `sdd-orchestration | planning | product-review`) no requirió expansión.
- **Superficie en `axiom context status`**: un bloque `delegationSuggestions` best-effort, wireado igual que el bloque `recentLessons` de AB10 — lee un `session-metrics.json` OPCIONAL en `.axiom-state/<projectKey>/`; si está ausente, malformado, o todas las métricas están debajo de umbral, el bloque se omite silenciosamente (nunca falla `axiom context status`).
- **Fuera de scope explícito**: agregar más IDE/harness adapter targets (Gemini CLI, Windsurf, Codex, Kilo, etc.) es un follow-up separado, acotado y on-demand — no una brecha de este incremento; cero archivos de `packages/adapters/*` fueron tocados.

## Registro histórico: plugins externos declarativos (Azure DevOps) — INC-20260708-app-plugins-azure-devops

> Este bloque describe el estado inicial, cuando la superficie ADO del app era solo declarativa. El contrato vigente es el del tracker desacoplado y del plugin ADO del launcher documentado más abajo: ADO es opcional, real cuando está configurado y siempre confirm-gated/no bloqueante.

En el estado histórico de esta entrega, "las integraciones externas son plugins opcionales, Axiom Core funciona sin ellos" se cumplía porque la superficie del app aún no ejecutaba mutaciones:

- `apps/cli/src/commands/app-plugins-azure-devops.ts` es una constante `AppPlugin` **puramente declarativa** (schema de formulario: tabs, acciones, campos) — sin cliente HTTP, sin SDK de Azure DevOps, sin manejo de credenciales. Sus campos `command` referencian `axiom-external-sync-command`, uno de los intent commands `notImplemented` de `@axiom/orchestrator`. **Ninguna acción que este plugin declara puede ejecutarse hoy realmente.**
- Estructuralmente aislado de Axiom Core: solo alcanzable a través de la `axiom app` (PWA/API local opcional). Ningún comando o paquete estructural bajo `Axiom/packages/*` importa ninguno de los dos ficheros del plugin; borrar ambos ficheros solo afectaría a `app-api.ts` y sus propios tests.
- El `field.externalRef?: boolean` del plugin es un flag de UI no relacionado — no lee ni escribe el mecanismo real de artefacto `externalRefs` (ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md)). Esto es correcto, no un hueco: cablearlo antes de que exista una ruta de ejecución real sería código especulativo sin llamador.
- La ejecución real, el gate `confirmed` y la persistencia en `externalRefs` se añadieron después; ver la sección normativa "Plugin ADO en el launcher" y [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md).

## sdd-launcher-port: tracker desacoplado, launcher y plano de control MCP bidireccional (2026-07-11)

La tanda sdd-launcher-port cierra los dos huecos genuinamente ausentes (motor de prompts, catálogo de acciones) y respalda el stub declarativo de ADO de arriba con un cliente real, sin acoplar Axiom Core a ADO:

- **`@axiom/tracker` + `@axiom/tracker-ado`** (`INC-20260711-sdd-launcher-p2-tracker`): nuevo puerto `IWorkItemTracker` con `NullTracker` (no-op, **default local-only**) + puertos `SecretStore`/`promptForSecret`; `@axiom/tracker-ado` porta el cliente ADO real (raw `https` a `dev.azure.com` `_apis`, sin import de `vscode`) detrás del puerto. Superficie WIT lifecycle-critical + 4 mappings (increment→US, bug→Bug, role→Task, e2e→US+Task) COMPLETA; la superficie ADO periférica build/git-traceability/sprints/attachments ya está cubierta por 4 sub-interfaces OPCIONALES de `IWorkItemTracker` con impl ADO real detrás del transporte (`NullTracker` las no-opea; guard sin `vscode` verde; tests con fakes, sin red) — `INC-20260711-ado-peripheral`. Config `tracker.kind: 'ado'|'none'`; el swap es solo-config (ningún code path de core cambia); los tests usan fakes en memoria (sin red). Respalda el `AZURE_DEVOPS_PLUGIN` declarativo y `axiom external-sync azure-devops …` de la sección anterior (que dejan de ser un stub sin ejecución).
- **`@axiom/launcher`** (`INC-20260711-sdd-launcher-p4-launcher`): motor de prompts puro + catálogo de acciones familias×modos + tabla de routing adapter-agnóstica para `claude-code` / `github-copilot` / `cli`, que resuelve a ids reales de comando/skill/MCP (fallback a `defaultAgentMention`). Consumido por el front `axiom app` (ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md)).
- **MCP como plano de control bidireccional** (`INC-20260711-cross-repo-mcp-wiring`): además de las tools de lectura/validación (`spec.*Read`, `sdd.allowedWriteScopeRead`, `spec.planRead`), la nueva tool de ACCIÓN **`sdd.transitionApply`** aplica una transición de estado detrás de un gate `confirmed` (sin confirmar → preview; ilegal → error tipado; project-pinned). Es la vía por la que un repo de rol lee el estado del plan desde el repo de spec y empuja su transición cross-repo (bindings directos como fallback sin server). Ver [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md).

## Adopción/migración de repos foráneos (2026-07-11) — tanda migración (Fases 0-5, entregada)

Axiom puede ahora ADOPTAR un proyecto que precede a Axiom, convirtiéndolo a forma Axiom-apt y MCP-queryable sin recrear su historia. Es una capacidad de integración entregada en cinco fases (Fase 0 en `INC-20260711-audit-bug-fixes`; 1+2 `INC-20260711-mig-spec-adopter`; 3 `INC-20260711-mig-context-ingest`; 4 `INC-20260711-mig-idempotency`; 5 `INC-20260711-mig-adopt-ux`):

- **Adopter de repo de spec foráneo** (`from-legacy-sdd` format-aware): un registro de detectores enchufable (`axiom-native` / `openspec` / `docs-adr` / `generic-folders`) reemplaza el mapa folder-name-only, con lectura de `status` por front-matter, conversión ESTRUCTURAL a la plantilla canónica de incremento (con fallback verbatim graceful, nunca pérdida de datos) y un reporte de migración por corrida. Adopta repos spec foráneos reales, no solo carpetas Axiom-shaped.
- **Ingesta `from-context`** (`axiom bootstrap from-context <path>`): mapea un contexto técnico existente (`ARCHITECTURE.md`/`docs/**`/ADRs) a `technical-context/*` + un `TechnicalContextIndex` `draft` servido por `spec.technicalContextIndexRead` (MCP) — MCP-queryabilidad gratis, sin superficie MCP nueva; banner `AXIOM:MIGRATED` distinto del `AXIOM:DRAFT` de `from-code`; re-corrida sin clobber.
- **Idempotencia por provenance**: un mapa source→artifact en `<specScope>/technical-context/.migration-provenance.yml` (keyed por path normalizado + sha256) cubre AMBOS subjects; una segunda corrida salta las fuentes ya migradas y nunca duplica. Por defecto acuña ids frescos (Q-004), preservando un id Axiom-style foráneo solo si es válido y no colisiona; un mapa ausente/corrupto degrada a fallback vacío seguro.
- **UX de adopción en install-time** (`axiom workspace setup --adopt-spec/--adopt-sdd/--ingest-context [--dry-run] [-y]`): orquesta la conformancia del repo de control (subject B: reporte present/added/must-reconcile-by-hand) + la migración del repo de spec foráneo (subject A) + la ingesta de contexto, tras un gate dry-run→confirmación. Superficie de comandos en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md) y flujo en [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md).

Las cuatro **garantías preservadas** en toda ruta de adopción: no-clobber (nunca sobrescribe un artefacto / `axiom.yaml` / doc de contexto existente), provenance (`AXIOM:MIGRATED` nombrando la fuente), dry-run (cero escrituras, reporta qué crearía y a qué estado) y collision-skip (una entrada mala se salta con motivo, nunca aborta el lote).

**Refinamientos 2026-07-27 (validados end-to-end contra KVP25 real — ver `manuales/E2E-20260727-kvp25-adoption.md`):**
- **Migración de contexto técnico POR DEFECTO** (INC-20260727-adopt-context-default): la adopción ya no exige `--ingest-context`. Si el operador no lo pasa, `runWorkspaceAdopt` auto-detecta una carpeta de contexto convencional dentro del repo legacy adoptado — `autoDetectContextSource` prefiere `technical-context/` y luego `context/`, spec-repo antes que sdd-repo, y NUNCA la raíz del repo (ingeriría por error las carpetas de artefactos de spec). `--ingest-context` explícito siempre gana; sin carpeta de contexto, la adopción procede sin ingest (comportamiento previo). Verificado: adoptar KVP25 (`Kvp.Spec/context`, 164 `.md`) migra los 164 a `technical-context/{architecture,operations,references,testing}` sin flag.
- **Ruido de telemetría eliminado** (BUG-20260727-adopt-telemetry-sinks-warn): un `axiom.config/telemetry-sinks.yaml` AUSENTE es el default local-only (nada lo scaffoldea), no un defecto — los callers (`index.ts` bus global por comando + `sync.ts`) ya no emiten WARN cuando `loadEnabledSinks` devuelve `missing-file`; sólo ante error de config REAL (`invalid-yaml`/`malformed-shape`).

## Epic sdd-launcher-port: entregado y archivado (2026-07-11)

Ya no queda ningún incremento `INC-20260711-*` ACTIVO en `specs/increments/`: todos están archivados. El epic umbrella del sdd-launcher-port quedó **totalmente entregado y archivado** (su `integrationMap` apuntaba a esta sección y a `04`/`05`), igual que el plan de migración foránea `INC-20260711-spec-sdd-migration-plan` (ver "Adopción/migración de repos foráneos" arriba):

- **`INC-20260711-sdd-launcher-core-port`** (epic umbrella) — re-home del sdd-launcher de KVP25 a Axiom core, **TOTALMENTE ENTREGADO Y ARCHIVADO**: P0 (generador canónico + estado explícito), P1 (subcomandos `scaffold`/`normalize`/`integrate`/`validate transition`/`state`), P2 (`@axiom/tracker` + `@axiom/tracker-ado`), P4 (`@axiom/launcher`), PX (wiring cross-repo + `sdd.transitionApply`) y P3 (front + server; push channel SSE + execute `plan-new`/`plan-execute` en `INC-20260711-front-longtail`) shipped y archivados; también entregadas las git-services + la variante de side-effect `script/action` (`INC-20260711-git-services`) y la superficie ADO periférica build/git-traceability/sprints/attachments (`INC-20260711-ado-peripheral`). El último ítem — los tres paneles del front (sugerencias ADO + role-branch/commit-sync git, UI sobre motor ya entregado) — se entregó en `INC-20260711-epic-close-panels` (ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md)). No queda nada pendiente salvo lo **DESCARTADO fuera de Axiom** (decisión de owner 2026-07-11 — no diferido, no pendiente): cost-dashboard (copilot-usage) y deployment/trace.

## Convergencia de spec-scope, code-intel MCP y skills de proceso (2026-07-13) — tanda fixes + north-star

- **Convergencia de la ruta de spec-scope (role-derived)** (`INC-20260713-fix-spec-scope-path`): CLI de ciclo de vida, broker spec-MCP y migración/adopción resuelven los artefactos (`increments/`/`bugs/`/`plans/`/`adr/`/`decisions/`) en el MISMO árbol. En un repo de spec DEDICADO (`role === 'spec'`) el rel-path es `.` (artefactos en la raíz del scope, sin `axiom.spec`); el resto mantiene `axiom.spec`. Fuente única: `resolveSpecRelPathForScope(role)` (`@axiom/workflow`).
- **`spec.implementationContextRead` puebla sobre repo de spec dedicado** (`INC-20260713-fix-impl-context-path`): el handler composite `buildImplementationContext` threadea el `specRelPath` role-derived a las lecturas de plan/relatedSpec/adr/allowedWriteScope, de modo que el bundle WHAT devuelve `plan`/`relatedSpec`/`allowedWriteScope` poblados (antes `null`/`[]`) en un repo de spec dedicado.
- **Breadth y polish de adopción** (`INC-20260713-adoption-breadth` / `INC-20260713-adoption-polish`): los detectores descienden a `specs/`; `e2e/`→increment (banner `AXIOM:E2E-ORIGIN`); se ignoran ficheros de registry; `from-context` reconoce la taxonomía de contexto KVP; preview de re-corrida vía provenance; warning de baseline-commit (sin mutación git); comando MCP resoluble.
- **Code-intel MCP auto-cableado en repos de código cuando está habilitado** (`INC-20260713-ns3-codeintel-mcp`, actualizado por `INC-20260724-*`): las configs MCP nativas de los repos de CÓDIGO (`.mcp.json`/`.cursor/mcp.json`/`.vscode/mcp.json`/`opencode.json`) reciben una entrada por provider code-intel HABILITADO, `cmm` o `serena`, apuntada a ese repo, junto a `sdd-mcp-server`/`spec-mcp-broker` cuando corresponda. Fuente única `buildCodeIntelNativeServers` (`@axiom/providers`). Default vacío → nada emitido; el repo de SPEC no recibe code-intel nativo.
- **Los 3 skills/agents de proceso en el catálogo** (`INC-20260713-ns1-process-skills`): `axiom-spec-author` (analista), `axiom-role-planner` (arquitecto) y `axiom-role-implementer` (implementador), registrados en `axiom.config/skills-catalog.yaml` (bundleHash sha256), MCP-tool-referencing y parametrizados por placeholder; NS-2 los genera en los repos.
- **Catálogo de corner-cases e2e** (`INC-20260713-e2e-corner-case-catalog`): survey read-only de las suites existentes (274 ficheros de test) mapeando cada corner-case a COVERED (`fichero::test`) o NONE, + tests e2e nuevos SOLO para los huecos cross-cutting P0/P1 genuinamente descubiertos (encadenan `runWorkspaceSetup`/`runWorkspaceAdopt`/`runBootstrapFromLegacySdd`/`runUpgrade`/`buildImplementationContext`, que ningún test existente ejercía end-to-end).

### Brechas conocidas (2026-07-13) — TODAS RESUELTAS en la tanda post-KVP25 (2026-07-14, ver sección siguiente)

- **Restore de rollback invocable por operador** — ✅ RESUELTA (`INC-20260714-op-rollback-restore`): nuevo comando `axiom rollback <id>`.
- **`axiom upgrade` es per-repo** — ✅ RESUELTA (`INC-20260714-cross-repo-upgrade-fanout`): fan-out sobre la topología desde el repo de control; `--repo-only` conserva el modo per-repo.
- **`member install` / `adapter add` no emiten superficies code-intel/proceso** — ✅ RESUELTA (`INC-20260714-member-adapter-surface-parity`): ambos materializan process-surfaces + code-intel a paridad con `repo add`.
- **"Install" de toolchain es declare+marker** — ✅ RESUELTA (`INC-20260714-toolchain-install-honesty`): `validate` refleja marker vs installed-working (sin verde-falso); UX `add`/`repair` clarificada como declare/scaffold.
- **`configure.ts` con llamada al generador casi-duplicado** — ✅ RESUELTA (`INC-20260714-configure-generation-dedup`): `configure` y `sync` comparten `resolveAgentsMdTemplateContent` (on-disk → bundled).

## Cierre de brechas post-KVP25 + workflow ADO en el front (2026-07-14) — tanda INC-20260714-*

Tanda que cierra las 5 brechas cazadas en el test de integración KVP25 (ver sección anterior) y amplía el front con un workflow ADO. Gate verde por incremento (build/test/typecheck/doctor; suite 2791→2832, +41 tests, cero regresiones; los timeouts 5000ms observados son el flake conocido de I/O bajo carga, confirmados verdes en aislamiento).

- **Paridad de superficies en `member install` / `adapter add`** (`INC-20260714-member-adapter-surface-parity`): ambos comandos ahora materializan las **process-surfaces por-rol** (NS-2, `materializeProcessSurfaces`) y las **entradas MCP de code-intel** cuando hay provider habilitado (NS-3, `writeWorkspaceNativeMcpConfigs` con `codeIntelProviders` + `kind` por-repo), replicando el patrón de `runRepoAdd`. `member install` cubre cada repo que el miembro tiene localmente (bindeado + en disco); `adapter add` cubre todos los repos del proyecto. Señal de habilitación idéntica al resto del motor (`workspace.json#providers` filtrado por `isCodeIntelProviderId`) → default vacío = salida byte-idéntica (sin regresión).
- **Honestidad de toolchain "install"** (`INC-20260714-toolchain-install-honesty`): decisión de owner = **opción (b)** (honestidad, NO instaladores reales especulativos — alineado con los límites de bootstrap). `toolchain validate` es probe-backed y **nunca** presenta como "✓ válido/OK" un tool declarado cuyo estado real no es `installed-working` (marker/declared/absent), aunque sea `mvp:false` — cierra el "verde falso"; añade un bloque "Estado real de las tools declaradas (X/Y instaladas)". Se preserva la semántica de exit-code (un opcional ausente no es fallo bloqueante → `doctor` sigue PASS). UX de `add`/`repair` clarificada (declara/scaffold, no instala). Instaladores reales por proveedor quedan como consideración futura documentada.
- **Workflow ADO en el front del launcher** (`INC-20260714-launcher-ux-ado`, P1): nuevos endpoints del launcher (`apps/cli/src/commands/app-launcher-ado.ts`) que exponen la superficie de escritura ya existente de `@axiom/tracker-ado` — **crear work item, cambio de estado, estimación, marcar horas, enlazar rama/PR/commit (ArtifactLink), y enlazar el work item con el artefacto Axiom** (vía `runExternalRefAdd` → `metadata.yml#externalRefs`). Contrato de seguridad idéntico a `/launcher/execute`: sin mutación sin `confirmed:true` (preview primero); **config-gated** sobre `tracker.kind` (si no es `ado` → estado claro "no configurado", sin red). Tests con `NullTracker` + `FakeHttpTransport` (nunca `dev.azure.com`). Azure DevOps sigue siendo un plugin opcional y no bloqueante (invariante P2).

## Tuning de agente por adapter + puente ADO en creación (2026-07-15) — tanda INC-20260715-*

- **Tuning de agente por adapter** (`INC-20260715-adapter-agent-tuning`): la entrada de routing de adapter (`@axiom/launcher`) admite `agentTuning` (`verbosity` / `personality` / `model?`). `craftPrompt`/`buildPrompt` inyectan un preámbulo determinista ("Ajustes del agente" + directiva de trabajo directo/económico en tokens). Los tres adapters de serie traen `{ verbosity:'low', personality:'pragmatic' }` (idéntico → el cuerpo del prompt sigue siendo byte-idéntico entre adapters con el mismo tuning, invariante del snapshot preservada); un proyecto puede diferenciarlos. Capacidad de prompt-shaping pura, desacoplada de model-routing/providers. Superficie en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md).
- **Puente Azure DevOps en creación** (`INC-20260715-launcher-ado-bridge`): tras crear incremento/bug desde el launcher, si el tracker ADO está realmente configurado (`isRealTrackerRequested`: `kind:'ado'` + `enabled` + org + project), la respuesta de `execute` incluye una `trackerSuggestion` (mapeo incremento→`User Story`, bug→`Bug`) que el front ofrece como creación de work item de un clic, reusando `apiAdoCreateWorkItem` (confirm-gated). La detección es network-free (`resolveLauncherTrackerStatus`, módulo neutral `_tracker-status.ts`, sin construir tracker ni tocar red, sin import circular). NO acopla el ciclo de vida del incremento/bug ni modifica `@axiom/tracker`/`@axiom/tracker-ado`; Azure DevOps sigue siendo opcional y no bloqueante. Config del plugin y ubicación del PAT documentadas en [manuales/12_Plugin_Azure_DevOps.md](manuales/12_Plugin_Azure_DevOps.md) y [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md).

## Ampliación del catálogo de skills/agents SDD (2026-07-15) — tanda INC-20260715-*

El catálogo runtime (`axiom.config/skills-catalog.yaml` / `agents-catalog.yaml`, materializado por `@axiom/skills` / `@axiom/agents` con `bundleHash` verificado por doctor TC-010/TC-011) crece de 10→**18 skills** y 10→**14 agents** para alinear el set instalado con un sistema role-specialized, sin perder genericidad:

- **Disciplinas transversales reutilizables** (`INC-20260715-reusable-discipline-skills`): `axiom-structured-doubts`, `axiom-functional-checklist-coverage`, `axiom-plan-drift-alignment`, `axiom-role-close-doc`.
- **Gate de revisión** (`INC-20260715-phase-reviewer`): skill+agent `axiom-phase-reviewer` (lentes spec/plan/código, OK/KO, loop-until-dry).
- **Consolidación y contexto** (`INC-20260715-consolidation-surfaces`): `axiom-spec-integrator` + `axiom-tech-context` (con detección de spec-drift).
- **QA + seguridad** (`INC-20260715-quality-gates`): `axiom-qa-validator` (test plan desde criterios, 1:N, `SIN COBERTURA`) y `axiom-security-reviewer` promovido de stub a cuerpo real (10 familias de riesgo + severidad, solo-lectura/defensivo, no bloqueante).
- **Planner** (`INC-20260715-planner-analysis-fanout`): `axiom-role-planner` gana un análisis de alcance opcional por dimensión, portable (sin exigir subagentes).

## Stack externo (cmm, RTK, concisión, AutoSkills), MCP unificado y code-intel por worktree (2026-07-24) — tanda INC-20260724-*

Tanda de graduación a *full product lifecycle*. Reordena el stack externo (proveedor estructural único `cmm`, RTK/concisión como skills, higiene de AutoSkills), unifica el MCP del repo `<project>.axiom` y aísla el code-intel por worktree.

### `cmm` sustituye a `graphify` y `codegraph` como único proveedor estructural — ADR-0031 (INC-20260724-cmm-replaces-graphify-codegraph)

**SUPERSEDE** la sección "Providers de code-intel cableados (codegraph/serena/graphify) — INC-20260708-code-intel-providers-wired" y el `SELECTABLE_PROVIDER_IDS` de "Selección de providers" (arriba). `codebase-memory-mcp` (`cmm`) pasa a ser el **único** proveedor estructural; `graphify` y `codegraph` dejan de ser seleccionables/registrables/enrutables en cualquier parte. El set cerrado `CANONICAL_PROVIDER_IDS` baja de 7 a **6** ids (fuera `codegraph`/`graphify`, dentro `cmm`); el cambio del set cerrado se autoriza vía **ADR-0031** (`Axiom.Spec/decisions/0031-adr-cmm-replaces-graphify-and-codegraph.md`), honrando la regla ADR-0021.

- `cmm` sirve AMBAS capabilities estructurales (`code.knowledgeGraph` + `code.structureAnalysis`) con un `ProviderClient` real (`cmm-client.ts`, patrón `serena-client.ts`); nuevo `ProviderKind` `'structural-code-intel'`.
- `serena` **sin cambios** = simbólico (`code.semanticNavigation`). Separación fuerte cmm (estructural/grafo/blast-radius/dependencias/trazas) ↔ serena (def/refs/rename).
- **Fallback siempre**: `cmm → filesystem`, `serena → filesystem`. El routing sigue gobernado por la maquinaria existente `@axiom/tool-routing` (cero cambios de código; solo cambió el DATO en `providers.yaml`).
- **Freshness/auto-sync mínimo para `cmm`** (no existía antes): marcador `.cmm/sync-state.json`, chequeo `fresh/stale/unknown` vs. ventana de edad, auto-sync best-effort antes de un análisis estructural — **on-demand, nunca un hook git** (ver [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md)). En aquella tanda el doctor `CC-001` y el probe del toolchain se actualizaron de 7 a 6 providers; R-04 retiró después gateway y snapshots, dejando 4 ids locales actuales.
- El binario/tools de `cmm` (`codebase-memory-mcp`, subcomandos `mcp`/`sync`, tools `explore`/`query_structure`) son una ASUNCIÓN documentada, **totalmente overridable** (comando de lanzamiento y nombres de tool por-capability), al no haber README verificable en el entorno.

### MCP unificado `axiom` del repo `<project>.axiom` (INC-20260724-unified-axiom-mcp)

**SUPERSEDE** el modelo de 2 brokers (`sdd` control + `spec` conocimiento) como forma OBJETIVO para un repo `<project>.axiom`, de forma aditiva (los brokers `sdd`/`spec`/`memory` siguen existiendo, retrocompat). Un code repo enlazado a `<project>.axiom` usa **un solo** broker `axiom` (`axiom-mcp-broker`, un `McpServerKind` nuevo) que expone la UNIÓN de: todos los reads `spec.*` + el subconjunto de escritura `sdd.transitionApply` + `sdd.gitCommitSync` + 3 reads nuevos de plano de proyecto — `axiom.topologyRead`, `axiom.migrationManifestRead`, `axiom.adoptionStateRead` (este último derivado de topology + manifest, sin nueva máquina de estados). El registro pasa de 22 a **25** capability ids (dominio nuevo `axiom.*`). `sdd.gitRoleBranch` **no** se expone en el broker `axiom` (el subconjunto de escritura es exactamente 2 tools). Engram/memory sigue en su propio proceso/kind aparte. El registro vigente de 25 ids y su agrupación por `--kind` quedan descritos en la sección "Server MCP ejecutable"; la cifra histórica de 17 ids se conserva únicamente en el bloque de primera capa de herramientas MCP.

### RTK invocado solo por skill + `axiom-terminal-output-efficient` (INC-20260724-rtk-skill-invoked)

RTK (reductor de output de terminal, `kind: input-optimizer`) pasa a ser usable **solo vía skill** — nunca un hook git, nunca un wrapper global/transparente sobre comandos. Skill nueva `axiom-terminal-output-efficient` (`axiom.config/skills-catalog.yaml`, catálogo 18→**19**, `bundleHash` verificado por doctor `TC-010`) que codifica: una tabla de decisión cuándo-sí/cuándo-no; un flujo de fallback (ejecutar optimizado → si no explica el fallo, re-ejecutar SIN RTK → guardar output completo como artifact → degradar a full si RTK no está instalado); y una lista de exclusiones **never-compress** (memoria Engram, spec e increments/bugs, ADRs/decisions, evidencia de compliance/seguridad — ver [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md)). RTK sigue con detección passive-only (sin probe activo fabricado).

### Disciplina de concisión (filosofía de Caveman) — `axiom-concision-discipline` (INC-20260724-concision-skills-policy)

Skill de disciplina nueva `axiom-concision-discipline` (catálogo 19→**20**, `bundleHash` verificado) que adopta la **FILOSOFÍA** de concisión de Caveman **sin instalar ni vendorizar** Caveman (sin runtime; la entrada `caveman` del toolchain-catalog queda intacta). Principios: no repetir la petición, sin ceremonia (intros/outros), no narrar operaciones triviales, conclusión primero; regla de oro absoluta: **la concisión nunca puede ocultar información necesaria para validar el trabajo** (evidencia, errores, restricciones, riesgo, incertidumbre y necesidad de revisión humana siempre se preservan). Distinta y con cross-link a `axiom-terse-comms` (inter-agente, otro mecanismo de catálogo) y a `axiom-terminal-output-efficient` (eje de output de terminal).

### Higiene del lock de AutoSkills (INC-20260724-autoskills-lock-hygiene)

Las skills instaladas por AutoSkills en un code repo son ahora identificables y gobernadas. `CatalogEntry` (`@axiom/skills`) gana 2 campos opcionales aditivos: `provenance` (`SkillProvenance`: `'autoskills' | 'axiom-native' | 'project-native' | 'user-local'`; default `'axiom-native'` — retrocompat) e `installedAt` (ISO-8601, reloj inyectable, nunca inventado). Toda entrada instalada por AutoSkills lleva `provenance: autoskills` + `installedAt`. Gate de policy allow/deny/licencia vía un fichero **opcional** `axiom.config/autoskills-policy.yaml` en el code repo (ausente/malformado ⇒ allow-all; orden denylist → licencia → allowlist); una skill denegada se salta con razón clara (`policySkipped` + `warnings`), nunca se instala. AutoSkills sigue corriendo **solo** por-code-repo en install-time (sin hook de pull/sesión/worktree). Gobierno en [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md).

### Provisioning y aislamiento de code-intel por worktree (INC-20260724-worktree-provisioning / -worktree-provider-isolation)

- **Provisioning** (`provisionWorktreeExecution`): materializa en un worktree la superficie `.axiom` portable + config MCP del broker unificado `axiom` + config code-intel nativa (cmm/serena) apuntada al path del worktree + el layout `.axiom-state` execution-scoped, **reutilizando** primitivas existentes (`materializeAdapterOutputs`/`buildAxiomMcpBrokerEntry`/`buildCodeIntelNativeServers`/`buildExecutionScopedPaths`), no reimplementando. Best-effort, no-clobber, created-gated; portable-only (secretos nunca copiados — ver [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md)).
- **Aislamiento de providers por worktree**: `ProviderInvokeContext` gana `worktreeRoot?`; `resolveCodeIntelRoot(ctx) = ctx.worktreeRoot ?? ctx.projectRoot` es la única función de resolución que usan `cmm-client`/`serena-client` para `cwd`, argumento de proyecto y freshness. Cada worktree obtiene su propio índice/caché (`.cmm`/`.serena`), **nunca** un grafo mutable compartido; freshness por worktree. `teardownWorktreeCodeIntel` (síncrono, best-effort, single-target) borra el estado derivado de UN worktree, **nunca** toca el índice del repo principal.

### Warning MCP `stale-artifact` (INC-20260724-sdd-artifact-freshness)

`McpToolResult` gana un campo opcional aditivo `warnings?: McpToolWarning[]` (omitido, no `[]`, cuando no hay nada que avisar — preserva los tests de igualdad exacta). Las lecturas `spec.incrementRead`/`bugRead`/`planRead` y la escritura `sdd.gitCommitSync` adjuntan un warning `stale-artifact` cuando `checkArtifactFreshness` (`@axiom/workflow`) reporta que el artefacto va por detrás de su remoto; el warning llega al `content[]` de la respuesta `tools/call` real (no solo un campo en memoria). Nunca bloquea. Flujo en [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md).

Todo el contenido es adapter/stack-agnóstico; la profundidad específica de cada proyecto se inyecta por `skills-index/<role>.yaml` + contexto técnico + skills de rol (ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)). Requisitos en [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md) (RF-AXM-034..039).

## Paridad de alcance de adapters + launcher onboarding (2026-07-26) — tanda INC-20260726-*

Tanda de 7 incrementos que lleva el conjunto de adapters a paridad de primera clase (registro, generadores, MCP nativo, routing/prompt del launcher), añade probes de runtime opt-in a `doctor` y convierte el onboarding del launcher en un front config-rich. Requisitos en [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md) (RF-AXM-049..055); no funcionales en [02_Requisitos_No_Funcionales.md](02_Requisitos_No_Funcionales.md) (NFR-AXM-020/021).

### Hardening del launcher y plugins externos (2026-08-04) — ACC-006/007/008

El launcher delega setup/adopcion y lifecycle en los runners canonicos. Valida
ownership de destinos antes de adoptar, rechaza paths de repos solapados y
devuelve resultados parciales con provenance y warnings. El lifecycle propaga
`executionMode` al handler async de roles y `runIntegrate` hace preflight y
rollback antes de informar un archive exitoso.

Los plugins declarativos usan `schemaVersion`, `handler` y una allowlist
estatica. `command` solo se compara como etiqueta; no se ejecuta. La
proyeccion de catalogo/resultado es explicita, valida tipos/options/required,
redacta mensajes y URLs, y omite propiedades desconocidas. Azure DevOps sigue
opcional: `NullTracker` no hace red y ADO real solo se construye mediante sus
ports/configuracion existentes.

### Registro canónico de adapter targets (fuente única) — INC-20260726-adapter-registry-canonical, actualizado por ACC-025..ACC-029

> **Actualización (ACC-025..ACC-029, 2026-08-10):** el vocabulario canónico vigente es de **8 targets** (`opencode, claude-code, github-copilot, vscode, cursor, antigravity, visual-studio-2026, codex`). `litellm` fue **retirado** del contrato activo (`REMOVED_ADAPTER_TARGETS`) y `copilot-vscode` quedó como **alias legacy** de `github-copilot` (`LEGACY_ADAPTER_TARGETS`). El detalle canónico completo vive en `Axiom.Spec/context/architecture/04-adapters-y-model-routing.md`; esta subsección conserva la fotografía de la entrega INC-20260726 como historial.

La entrega INC-20260726-adapter-registry-canonical reconcilió el vocabulario a fuente única a través del CLI (`init.ts` + `_adapter-labels.ts`), el composer de install-profiles (`default-profiles.ts`, `allowedTargets` de ambos profiles), `axiom.yaml#capabilities.adapters` y `SUPPORT_MATRIX`/`MVP_TARGETS` de model-routing. En aquella entrega el vocabulario era de 10 ids e incluía `litellm`; `vscode` dejó de ser invisible a `init`/`workspace-adapters` y `codex` se añadió como id nuevo. Detalle del invariante de reconciliación en RF-AXM-049 y NFR-AXM-021.

### Generadores de adapter dedicados (codex/antigravity/visual-studio-2026) — INC-20260726-adapter-generators, actualizado por ACC-025..ACC-029

**SUPERSEDE** la subsección "Adapter depth" de arriba en lo tocante a `antigravity`/`visual-studio-2026` "solo declarados / vía el escritor canónico AGENTS.md": ambos (y `codex`) tienen ahora paquete `@axiom/adapters-<target>` de primera clase, single-file y merge-preserving, byte-por-byte igual que `@axiom/adapters-claude-code` salvo la nota de adapter final. Escriben `.codex/AGENTS.md` y `.antigravity/AGENTS.md`. El despacho de `workspace-adapters.ts` para estos 3 targets llama a los generadores reales (`generate{Codex,Antigravity,VisualStudio2026}Config`), no al fallback thin-canonical. El check `TC-009` (GATE 0031) cubre los **8** packages de adapter activos (LiteLLM retirado). `writeThinCanonicalAgentsMd` y sus constantes quedan como código muerto intencional (fallback disponible para un futuro target sin generador). El nivel `'fallback-only'` de model-routing para estos 3 targets es un eje separado y no cambió.

> **Actualización (ACC-027, 2026-08-10):** `visual-studio-2026` **no genera `.vs/AXIOM.md`** como salida activa. Se proyecta sobre la instrucción común `.github/copilot-instructions.md` (mismo writer que `github-copilot`) y conserva su package solo como alias de compatibilidad para migrar estados legacy. El detalle canónico vive en `Axiom.Spec/context/architecture/04-adapters-y-model-routing.md`.

### Paridad de MCP nativo + superficie portable incondicional — INC-20260726-adapter-mcp-parity, actualizado por ACC-025..ACC-029

**SUPERSEDE** la nota de "Adapter depth" que decía que `visual-studio-2026` no tiene schema MCP nativo verificado, y la fila "`antigravity`, `visual-studio-2026`, `litellm` → sin schema, no fichero" de la tabla de "Config MCP nativa por herramienta destino". `writeNativeMcpConfig` (`native-mcp-config.ts`) da a cada target un fichero MCP nativo real (si hay schema verificado o de asunción documentada) o una nota informativa honesta — nunca un schema inventado. Tabla de despacho vigente (ACC-029):

| Target | Fichero | Shape | Verificación |
|---|---|---|---|
| `claude-code` | `.mcp.json` | `{ mcpServers }` | VERIFIED |
| `cursor` | `.cursor/mcp.json` | `{ mcpServers }` | VERIFIED |
| `copilot-vscode` / `github-copilot` / `vscode` | `.vscode/mcp.json` | `{ servers, type:'stdio' }` | VERIFIED |
| `opencode` | `opencode.json` | `{ mcp, command:[...], type:'local', enabled }` | VERIFIED |
| `visual-studio-2026` | — (ninguno) | warning explícito | schema/path MCP **no verificado**; no se escribe `.vs/mcp.json` |
| `codex` | ninguno | nota: `~/.codex/config.toml`, `[mcp_servers]` | user-global |
| `antigravity` | ninguno | nota: `~/.gemini/config/mcp_config.json`, `mcpServers` | user-global |

- `NATIVE_MCP_TARGETS` (gate de fichero de proyecto) = las primeras 4 filas (**7 ids**, `vscode` comparte fila con `copilot-vscode`/`github-copilot`). `NATIVE_MCP_INFORMATIVE_TARGETS` = `['codex','antigravity']` (segundo gate distinto, "tiene `case` explícito, sin fichero"). `litellm` fue retirado y no forma parte del contrato.
- La nota de `codex`/`antigravity` (helper `buildUserGlobalMcpNote`) nombra la ubicación user-global real, la clave esperada y los servers Axiom lanzables (`sdd-mcp-server`/`spec-mcp-broker`, con `command`/`args` reales).
- Ambos callers reales de `writeNativeMcpConfig` (`writeWorkspaceNativeMcpConfigs` en `workspace-mcp.ts` y `provisionWorktreeExecution`'s `isNativeMcpTarget` en `workspace-worktree-provision.ts`) despachan ahora `vscode` a escritura real y `codex`/`antigravity` a la nota informativa — el segundo caller tenía el mismo gap de gating y se corrigió (o la nota sería código muerto para él).
- La superficie portable `.axiom/{agents,commands,skills}/` (`materializeProcessSurfaces`) se confirma adapter-agnóstica e incondicional (se escribe incluso con `adapters: []`, antes de cualquier rama adapter-específica), así que todo adapter es descubrible por ella sin un formato de skill nativo por-IDE (diferido).
- El texto de la nota bundleada en `packages/adapters/{visual-studio-2026,antigravity}`'s `agents-md.ts` queda cosméticamente stale (aún dice "no hay schema MCP nativo verificado") — hueco de documentación conocido, no funcional; pertenece al scope de INC-20260726-adapter-generators, no a éste.

### Paridad de routing y enriquecimiento de prompt del launcher — INC-20260726-launcher-adapter-routing-parity / -launcher-prompt-context-enrichment

**Amplía** la sección "sdd-launcher-port" (que dejó `AXIOM_ADAPTER_ROUTING` con solo 3 ids: `claude-code`/`github-copilot`/`cli`). Ahora la tabla lista 9 entradas — los 8 adapters headline + `cli` — todas reusando `skillRoutingMap()`/`cliRoutingMap()`, de modo que cada acción de cada adapter resuelve al id de skill real (`axiom-sdd-orchestrator`/`axiom-phase-reviewer`) + `sdd.transitionApply`, nunca el fallback `@axiom` de portapapeles. El endpoint `apiGetLauncherData` expone los 9 con `label` legible (`ADAPTER_LABELS`). Sobre eso, cada prompt crafteado (`craftPrompt`/`buildPrompt`) lleva ahora:

- **DÓNDE leer**: `specFolderPath`/`specReadmePath`/`metadataPath` repo-relativos reales resueltos con `resolveArtifactDir` + `resolveSpecArtifactRelPath` de `@axiom/workflow` (la MISMA resolución del endpoint de registro; sin esquema a mano) — o, si el id aún no existe, una instrucción "id no asignado" en vez de una ruta falsa.
- **Bloque "Herramientas y ubicacion"**: nombra los MCP gestionados (`AXIOM_MANAGED_MCP_SERVERS = ['sdd-mcp-server','spec-mcp-broker']`, override-able por `CraftPromptOptions.mcpServers`), el `mcpTool` de mutación confirmada (`sdd.transitionApply`) y la skill a aplicar, derivados del `RoutingTarget` ya resuelto. Byte-idéntico entre adapters skill-routed; el adapter `cli` omite solo la línea de skill (target `command`-kind).

### Probes de runtime opt-in en doctor (`--deep`) — INC-20260726-doctor-runtime-probes

`axiom doctor` gana un superset asíncrono OPT-IN `--deep` (`runDoctorChecksDeep`, `packages/doctor/src/deep-checks.ts`) que añade, sobre el árbol síncrono de solo-configuración sin tocarlo, dos checks de categoría nueva `runtime-probes`:

- **`TC-018-<toolId>`** (`runToolchainFunctionalProbeCheck`): probe funcional best-effort por tool declarada en `toolchain.yaml`, reusando el contrato `resolveProbeCommand`/`probeToolInstalled` de `@axiom/toolchain` (`--version` para serena/cmm/engram). Hace `skip` honesto de toda tool sin contrato de binario (`rtk`/`caveman`/`context7`/`autoskills`) — nunca fabrica un probe.
- **`TC-019-<serverId>`** (`runMcpServerLivenessCheck`): handshake `initialize` JSON-RPC MCP **real** por server gestionado (`sdd-mcp-server`/`spec-mcp-broker`), contra el `command`/`args` leído de `.axiom/mcp.yml` (`loadMcpProjectConfig` de `@axiom/user-workspace`), reusando `createStdioMcpClient` de `@axiom/providers` y el handler `initialize` de `@axiom/mcp-server`. Un proyecto sin `.axiom/mcp.yml` (self-hosted) da `warn` honesto, no una adivinanza.

El endpoint del gate de doctor del launcher admite el mismo opt-in vía `?deep=1`. Aditivo/never-fail (nunca cambia `summary.failed` del árbol síncrono) — ver NFR-AXM-020. `@axiom/doctor` gana una dependencia hacia el paquete-hoja `@axiom/user-workspace` (leer `.axiom/mcp.yml`), sin violar el boundary package-no-depende-de-app.

### Front de onboarding config-rich del launcher — INC-20260726-launcher-onboarding-config-front

**Amplía** el onboarding del launcher (antes wrappers thin sobre `runInit`/`runProjectsJoin`, ver RF-AXM-031): install/join cubren ahora `name`/`path`/`profile`/`overlay`/`layout`/rol/adapter-primario/adapters-adicionales/tools/execution-mode con preview→confirm preservado. Cableado real por parámetro:

| Campo (install/join) | Cableado vía | ¿Ruta real? |
|---|---|---|
| name/path/profile/overlay/layout/rol/adapter primario | `runInit` | Sí (pre-existente) |
| adapters adicionales | `generateWorkspaceAdapters` (primario + adicionales) | Sí (NUEVO — cierra el hueco de que install nunca materializaba output por-adapter) |
| execution-mode | `runConfigure` | Sí, best-effort (NUEVO; fallo → `executionMode.warning`, sin rollback) |
| tools | — | NO — superficializado + nota honesta "pendiente" (`tools.applied:false`) |
| asignación de rol equipo/código (join) | `apiLauncherRolesAssign` vía auto-select del front | Sí (endpoint pre-existente, reachability nueva) |

Las tools se superficializan pero no se cablean porque `runToolchainAdd` exige un `axiom.config/toolchain-catalog.yaml` que `runInit` no scaffoldea — no hay ruta limpia de instalación desde un init recién hecho (diferido, nunca fingido como aplicado). `ADAPTER_LABELS` pasa a un fichero compartido `apps/cli/src/commands/_adapter-labels.ts` (unión de `ADAPTER_TARGETS` + `AXIOM_ADAPTER_ROUTING`); reconfirma la convención single-ownership (`configure.ts` importado desde `@axiom/cli-commands`, nunca por path relativo, o falla en runtime aunque `tsc -b` pase).
### Plugin ADO en el launcher — cierre de bucle spec↔work-item + doctor unificado (INC-20260727-*)

Consolida las capacidades del plugin Azure DevOps del launcher (sobre `@axiom/tracker` +
`@axiom/tracker-ado`), verificadas end-to-end contra ADO real (org KVPHiberus / proyecto KVP25):

- **Peripherals de formulario** (`GET /launcher/ado-peripherals?kinds=sprints,features,tags`):
  pobla los desplegables/datalist de crear-incremento/bug desde ADO — sprints
  (`sprints.listIterations()`), feature-padre (`listWorkItems({type:'Feature'})`) y tags (nueva
  capacidad `ITrackerTags.listTags()` → `GET /_apis/wit/tags`). Config-gated: si el tracker no es
  `ado`, devuelve listas vacías y el front degrada a texto libre. Verificado real: 18 sprints /
  80 features / 31 tags de KVP25.
- **Mapeo de campos al work item**: la creación de work item desde el launcher envía el work item
  completo (tags → System.Tags, sprint → System.IterationPath con el centinela `__BACKLOG__`
  omitido, feature-padre/parent → parentId, priority/severity/assignedTo/description/repro), no
  sólo `{type,title}`.
- **Cierre de bucle spec↔ADO** (INC-20260727-launcher-ado-external-ref-link): al crear el work
  item desde la tracker-suggestion, su id + url se **persisten** en el `metadata.yml` local del
  artefacto (`externalRefs: [{provider,type,id,url}]`), reusando `runExternalRefAdd`. Antes el id/url
  sólo se devolvían al front; el `create` de increment/bug sigue emitiendo `externalRefs: []` y el
  enlace ocurre en este paso post-creación (o vía el endpoint explícito `link-axiom` / el subcomando
  `external-ref add`). Best-effort: un fallo de enlace no aborta la creación.
- **Check de conectividad ADO en doctor** (INC-20260727-cli-doctor-ado-check): el probe real
  (`tags.listTags()` autenticado) vive en un helper compartido `runAdoConnectivityDoctorCheck` que
  usan TANTO el panel Doctor del launcher COMO el CLI `axiom doctor`. Devuelve `null` (sin check,
  sin red) para proyectos local-only; `pass`/`fail` con evidencia cuando ADO es el tracker activo.
  `@axiom/doctor` NO gana dependencia de tracker — la inyección es en la capa CLI/app. Verificado
  real: `axiom doctor --deep` → `ado-connectivity: pass` contra KVP25.

### Scaffolding de config de adopción en `runWorkspaceSetup` (PC-001/PC-002/TC-011/GC-001/GC-002) — INC-20260727-adoption-config-scaffolding

`runWorkspaceSetup` (el motor puro tras `axiom workspace setup` Y `axiom workspace adopt`) scaffoldea ahora, best-effort y no-clobber, los 4 artefactos de config que un proyecto recién adoptado/seteado no tenía y que hacían fallar 5 checks de doctor out-of-the-box. Reusa los dos precedentes ya establecidos (`scaffoldArchitectDeclarations` para ficheros de sola-existencia; `scaffoldSddSkills`/`scaffoldCodeRepoSkills` para artefactos con `bundleHash`), sin inventar un patrón nuevo. Es el complemento automático, para cualquier proyecto adoptado, de lo que `INC-20260708-product-repo-self-bootstrap` hizo a mano solo para el propio repo `Axiom/` (ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)):

- **`scaffoldPolicyDeclarations(control.path)`** (`apps/cli/src/commands/workspace-config-scaffold.ts`, step 2c-bis, junto a `scaffoldArchitectDeclarations`): siembra `axiom.config/integrations.yaml` (**PC-001**) y `axiom.config/policy-as-code.yaml` (**PC-002**). Ambas checks solo validan EXISTENCIA (sin hash), así que el contenido semilla no lleva datos por-proyecto. Función SEPARADA de `scaffoldArchitectDeclarations` para no tocar el `filesCreated` de una función ya testeada.
- **`scaffoldProcessSurfaceCatalogs(control.path)`** (`apps/cli/src/commands/workspace-catalog-scaffold.ts`, step 7b-ter, INMEDIATAMENTE tras el loop `materializeProcessSurfaces` — el orden importa: los escáneres leen los `.axiom/agents/*`/`.axiom/skills/*` de disco):
  - `axiom.config/agents-catalog.yaml` (**TC-011**): una entrada por cada `.axiom/agents/<id>.md` realmente materializado, con `source` = ruta repo-relativa de ese fichero y `bundleHash = computeAgentBundleHash(<contenido de ese mismo fichero>)` (`@axiom/agents`). NO se escribe (queda ausente) si no hay ningún `.axiom/agents/*.md` — TC-011 falla ante un `agents: []` vacío, así que un catálogo vacío sería peor que ninguno. En un `runWorkspaceSetup` real el repo de control siempre recibe sus 3 process-surfaces (`surfaceIdsForRole('sdd')`), así que ese path ausente es teórico.
  - `axiom.skills.lock` raíz (**GC-001/GC-002/GC-007**): una entrada por cada `.axiom/skills/<id>/SKILL.md` materializado, `bundleHash = computeSkillBundleHash(...)` (`@axiom/skills`); `source: 'axiom-process-surfaces'` (tag de provenance, deliberadamente ≠ `'product-registry'` para que GC-013 no lo considere). SIEMPRE escribe un lockfile válido (`schemaVersion: 1`, `installed: []` si aún no hay nada) porque GC-001/GC-002 solo exigen que exista/sea válido una vez `.axiom/*` tiene CUALQUIER contenido gestionado (agents/commands/skills), no específicamente skills.
- **Garantía de hash**: cada scaffolder hashea el MISMO fichero que referencia como `source`/`installedTo`, con el mismo algoritmo (`sha256:<hex>`) que el recompute del doctor (`runAgentsCatalogCoverageCheck` para TC-011, `checkGC007` para GC-007) — mismo fichero + mismo algoritmo, leídos una sola vez en memoria, sin re-render intermedio → match garantizado.
- **Alcance**: cableado solo en `runWorkspaceSetup` (compartido por adopt + setup no-adopt). `member install`/`workspace-incremental` NO se tocaron (un miembro recién unido ya hereda estos ficheros del `runWorkspaceSetup` inicial del arquitecto); extenderlos queda como follow-up si se detecta un hueco. Probado end-to-end (`apps/cli/tests/inc-20260727-adoption-config-scaffolding.test.ts`): un proyecto scaffoldeado de cero pasa PC-001/PC-002/TC-011/GC-001/GC-002 (y GC-007) cuando corren las funciones reales del doctor contra él, y un segundo run es idempotente.

## `@axiom/config-validation` como validador canónico consumido por la CLI (2026-08-02) — `INC-20260730-exact-scope`

`@axiom/config-validation` existía desde la tanda de schema v2 con validadores para `axiom.yaml` (v1/v2), `integrations.yaml`, `policy-as-code.yaml`, `capabilities.yaml` y `providers.yaml`, además del `InstallProfilesYamlSchema`. Ese último schema **no tenía ningún consumidor**: `apps/cli` no importaba el package en absoluto (solo lo mencionaba en un comentario), y mantenía en paralelo tres loaders de `profiles.yaml` escritos a mano.

A partir de este incremento `apps/cli` depende realmente de `@axiom/config-validation` y los tres loaders comparten `validateInstallProfilesYamlContent()`. La función se diseñó para devolver `data` tipada (a diferencia de las otras cuatro `validate*YamlContent`, que devuelven solo validez + errores) porque estos call sites necesitan el objeto parseado de vuelta.

Regla derivada, aplicable a futuras integraciones: **si ya existe un schema canónico para un fichero, consumirlo es obligatorio antes que escribir un loader nuevo**. Un schema sin consumidores es deuda, no cobertura.
