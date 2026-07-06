# 06 Integraciones y Capacidades

## Integraciones esperadas

1. workspace local multi-root (`@axiom/topology`, layout `installed-multi-repo`);
2. adapters de ejecución (6 packages `@axiom/adapters-*` operativos);
3. surfaces basadas en MCP (catálogo vía `@axiom/toolchain`, comandos `axiom mcp`, `axiom toolchain`; sin server MCP genérico propio de Axiom todavía — eso es roadmap futuro, `INC-13`/`INC-14`);
4. herramientas de análisis y contexto cuando aporten valor real: Serena como baseline de inteligencia de código; CodeGraph, Graphify, Headroom/RTK, Caveman, Autoskills como tools P1 con `mvp: true`, detección dinámica y repair idempotente (incremento `0027`).

## Targets soportados hoy (verificado, `SUPPORT_MATRIX` de `@axiom/model-routing`)

| Target | Nivel de soporte | Routing per-slot |
|---|---|---|
| `opencode` | `multi-mode` | Sí — único target con cobertura completa hoy |
| `claude-code` | `single-mode` | No — cae a `medium` (`fallbackReason: per-slot-routing-unsupported`) |
| `github-copilot`, `vscode`, `cursor`, `litellm` | `fallback-only` | No |
| `copilot-vscode`, `antigravity`, `visual-studio-2026` | `fallback-only` (sin adapter package dedicado) | No |

Esto reemplaza la lista aspiracional previa de "targets iniciales" (Opencode CLI principal, Antigravity IDE secundario, VS Code+Copilot IDE principal, Claude Code CLI secundario): en el código actual, `opencode` es el único target con soporte profundo; `antigravity` está solo declarado, sin adapter propio.

## Capability model y providers

`@axiom/capability-model` gestiona capabilities por dominio (`sdd`, `spec`, `code`, `memory`), con `supportLevels` y `degradationPolicy`. Providers se resuelven por `discoveryOrder` con perfiles `filesystem-first` / `gateway-first` / `local-only`. Serena es el provider de código baseline en el MVP; otros quedan opcionales o post-MVP.

## Integraciones externas opcionales (post-MVP)

`@axiom/app` (portable operator app, incremento `0030`) expone un sistema de plugins project-scoped con discovery en `.axiom-state/<project>/app-plugins/*.json` y un bridge declarativo hacia Azure DevOps (mutación externa exige `confirmed: true`; los plugins no introducen lógica de negocio paralela). No hay integración con Jira/Confluence implementada — solo mencionada como posible extensión en el roadmap futuro (`INC-16`).

## Capacidad clave

El runtime debe saber dónde está cada repo del proyecto y cómo resolverlo en cada surface soportada. Implementado hoy dentro de un único proyecto vía `topology.yaml`; la resolución de "en qué repo Axiom vive cada pieza del propio producto" (Axiom / Axiom.SDD / Axiom.Spec) es manual, gobernada por `Axiom.SDD/AGENTS.md`, no por una topología ejecutable.

## Capa de herramientas MCP (roadmap de rediseño, cerrado)

Resuelve la pregunta de arquitectura Q4: el sistema existente `@axiom/capability-model` / `@axiom/tool-routing` / `@axiom/isolation` se mantiene **completamente ortogonal** a los documentos de decisión MCP más nuevos — las herramientas MCP se añaden como `capabilityId`s nuevos y aditivos, despachados a través del mecanismo `routeTool` existente y sin modificar, nunca como un sistema paralelo o de reemplazo.

- **`@axiom/mcp-tools`** se sitúa por encima tanto de `@axiom/tool-routing`/`@axiom/capability-model` (cero dependencias de paquetes de dominio) como de los paquetes de dominio (`@axiom/workflow`, `@axiom/skills`, `@axiom/technical-context`, `@axiom/user-workspace`, `@axiom/cli-commands`). Registra un mapa `capabilityId -> handler` (`MCP_TOOL_HANDLERS`, `invokeMcpTool(capabilityId, input)`) y nunca añade una dependencia hacia ninguno de los paquetes por debajo de él.
- **17 capability ids registrados** (dominios `sdd.*`/`spec.*`): `sdd.projectRegistryRead`, `sdd.projectReposRead`, `spec.planRead`, `spec.incrementRead`, `spec.bugRead`, `spec.adrIndexRead`, `spec.decisionIndexRead`, `sdd.skillIndexRead`, `spec.skillCatalogRead`, `spec.skillRead`, `spec.technicalContextIndexRead`, `spec.recommendedContextList`, `sdd.allowedWriteScopeRead`, `sdd.changesValidate`, `sdd.indexesRebuild`, `sdd.indexesValidate`, `spec.implementationContextRead`. Los primeros 16 son cada uno una traducción delgada de una función existente y sin modificar — no se escribió lógica de negocio nueva para ninguno de ellos.
- **`spec.implementationContextRead`** (`getImplementationContext`/`buildImplementationContext`) es la lectura compuesta "insignia": una composición sobre los 16 handlers hermanos, no una dependencia de paquete de dominio nueva. Forma de la respuesta: `project`, `repositories{spec,sdd,target}`, `plan`, `relatedSpec`, `relatedAdrs`, `mandatory{sddSkills,repoSkills,technicalContext,rules,commands}`, `indexes{...}`, `recommended{skills,technicalContext,adrs,commands}`, `allowedWriteScope`, `confidence` (`high`/`medium`/`low`), `missingMetadata`.
  - `relatedSpec` es un **placeholder**, no una herramienta `get_related_specs` completa: solo resuelve `plan.links.incrementId` -> `getIncrement`.
  - `contextBudget` (`small`|`medium`|`large`, `small` por defecto) controla solo el inlining de contenido, nunca la presencia de campos: `small` devuelve solo referencias; `medium` incluye el contenido de `plan`+`relatedSpec`+`mandatory.*`; `large` añade además el contenido de `relatedAdrs`. `recommended.*` nunca incluye contenido en ningún nivel.
  - Resolución de `repositories.target`: gana el `input.targetRepoId` explícito; si no, si queda exactamente una entrada de repo tras excluir las claves `spec`/`sdd`, esa es `target`; si no, `target` es `null` y se marca en `missingMetadata` — un no-adivinar deliberado para proyectos con múltiples repos target: el llamador debe pasar `targetRepoId` explícitamente en ese caso.
  - `mandatory.rules`/`mandatory.commands` (y sus contrapartes en `indexes`/`recommended`) están permanentemente vacíos/ausentes: no existe ningún tipo de artefacto "rules" o "commands" que los respalde. `indexes.commands`/`indexes.rules` llevan `unsupported: true` para distinguir este hueco de producto permanente de un caso de "índice no encontrado esta vez" por proyecto.
- **Mapeo de proveedor `sdd`/`spec` -> `axiom-gateway`**: la allowlist MCP de `@axiom/isolation` (`sdd`/`spec`/`serena`) y los `CANONICAL_PROVIDER_IDS` de `@axiom/capability-model` (`filesystem`, `axiom-gateway`, `serena`, `codegraph`, `graphify`, `engram`, `generated-snapshots`) son dos listas independientes que solo coinciden en `serena`. Toda capability futura de sabor `sdd`/`spec` debería mapear al `axiom-gateway` provider id existente en vez de acuñar uno canónico nuevo (lo que requeriría ampliar el conjunto cerrado de 7 entradas y una ADR).
- **`@axiom/cli-commands`** es el seam establecido para exponer las funciones `runX` de `apps/cli/src/commands/*` a cualquier consumidor que no deba importar `apps/cli/src/...` directamente (construido originalmente para `@axiom/tui`). Toda nueva herramienta MCP respaldada por CLI o flujo de TUI debería reusar este seam como re-export trivial, no inventar uno nuevo.
- **Checks `TR-001..004` de `@axiom/doctor`** hacen smoke-test de `routeTool` vía un fixture en memoria; no dependen de ningún capability id específico. Extender la cobertura de doctor al registro de capability/provider de `@axiom/mcp-tools` (`TR-005`+) se consideró y quedó diferido — ningún installer/scaffold escribe hoy un `capabilities.yaml`/`providers.yaml` real a ningún proyecto, así que no hay call site de producción vivo que comprobar.

### `mcp.yml` vs `mcp-manifest.yaml`

Ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md) para el schema completo y la tabla comparativa — son dos ficheros con responsabilidades distintas que no deben fusionarse.

### Generación de config MCP durante el setup de workspace (SDD MCP + Spec MCP) — INC-20260705-workspace-mcp-generation

El setup de workspace multi-repo (`runWorkspaceSetup`, ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)) genera, como **paso best-effort final tras un registro exitoso**, la config MCP por proyecto. Es el primer y único call site que escribe un `mcp.yml` real a un proyecto (`apps/cli/src/commands/workspace-mcp.ts`, funciones `buildWorkspaceMcpServers` + `writeWorkspaceMcpConfig`).

- **`.axiom/mcp.yml`** (`McpProjectConfig`, `schemaVersion: 1`) se escribe en el repo de control, con exactamente dos entradas `McpServerEntry`, ambas `enabled: true`, `scope: 'repo'`:
  - `sdd-mcp-server` (SDD MCP) con `targetRepo` = la `roleKey` de registro del repo de control;
  - `spec-mcp-broker` (Spec MCP) con `targetRepo` = la `roleKey` de registro del repo de spec.
  
  Los `targetRepo` referencian claves del mapa de repos del registro v2 (no paths), y la validación reusa `validateMcpProjectConfig` (`@axiom/user-workspace`, sin re-implementar) contra el mismo `homeDir` que usó el registro. En el caso degenerado de que control y spec colapsen a la misma `roleKey` (workspace single-repo), solo se emite `sdd-mcp-server` más un warning, evitando un par `duplicate-type-target-repo` inválido.
- **Proyección a cada adapter MCP-capaz seleccionado** (generalizada por `INC-20260705-workspace-adapters-multiselect`): originalmente la proyección solo cubría el `target` primario único (`opencode` → `.opencode/mcp.json`, `claude-code` → `.claude/mcp.json`); ahora `writeWorkspaceMcpConfig` acepta la lista `adapters` y recorre el subconjunto MCP-capaz (`opencode`/`claude-code`) presente en la selección, produciendo **un fichero de config de adapter por cada adapter MCP-capaz** (`.opencode/mcp.json` y/o `.claude/mcp.json`, vía `generateOpencodeMcpJson`/`generateClaudeCodeMcpJson`, servers `enabled` mirroreados verbatim). Los adapters seleccionados sin capacidad MCP no producen fichero de proyección (se registra un warning nombrando el target no soportado). `WriteWorkspaceMcpConfigResult` expone `adapterConfigPaths: string[]` (plural; `adapterConfigPath` se conserva como la primera entrada por compatibilidad). `.axiom/mcp.yml` se escribe exactamente una vez, con independencia de cuántos adapters se seleccionen.
- **Semántica best-effort**: la generación solo corre si el registro tuvo éxito (`register !== false` y `registryRegistered: true`); si se saltó o falló, la generación MCP se salta con un warning explicativo. El paso está envuelto en try/catch que nunca escala: los fallos (incluidos issues de validación) se acumulan en `warnings`, nunca abortan el setup; `.axiom/mcp.yml` se escribe aunque la validación reporte issues (validate-after-write).
- **Caveat de gitignore**: `.axiom/` está gitignoreado, así que `mcp.yml` es un artefacto machine-local (no versionado) — coherente con que declara qué procesos servidor MCP están habilitados localmente para que los adapters generen config runtime.

Esto no toca `axiom configure`/`sync`, `GENERATED_FILES_BY_TARGET` ni el deferral TR-005 para la ruta no-workspace; tampoco emite entradas MCP para repos de rol/código (solo SDD + Spec MCP).

### Generación multi-adapter en todos los repos del workspace — INC-20260705-workspace-adapters-multiselect

Tras el registro y la generación MCP, `runWorkspaceSetup` materializa, como paso best-effort, los ficheros de **cada adapter seleccionado** (`spec.adapters`, ver el wizard en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md)) en **cada repo del workspace** — control, spec y cada repo de rol. Antes de este incremento el motor nunca escribía la salida real de ningún adapter en los repos scaffoldeados (solo `.axiom/mcp.yml` + su proyección); esta subsección supersede esa limitación. La lógica vive en `apps/cli/src/commands/workspace-adapters.ts` (`generateWorkspaceAdapters`).

- **Un `installProfile` por repo**: para cada repo se resuelve un único `ResolvedInstallProfile` (vía `installProfile`, usando `adapters[0]` como adapter primario y `DEFAULT_PROFILES` de `@axiom/install-profiles` como `profilesData` de fallback), y ese perfil resuelto se reutiliza en cada generador de adapter de ese repo.
- **Tabla de despacho `target -> generador`** que cubre los 8 `ADAPTER_TARGETS`:

  | Target | Generador / escritura |
  |---|---|
  | `opencode` | `generateOpencodeConfig` (`.opencode/AGENTS.md`, `.opencode/mcp.json`) |
  | `claude-code` | `generateClaudeCodeConfig` (`.claude/AGENTS.md`, `.claude/mcp.json`) |
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

Los generadores de `opencode`/`claude-code`/`copilot-vscode` dependían de plantillas de instrucciones (`agents-md-template.md`, `copilot-instructions.template.md`) que no estaban bundleadas en el producto — solo existían en `Axiom.Spec/templates/` de este workspace. En repos recién scaffoldeados eso degradaba a warnings `template-missing` y los ficheros de instrucciones no se escribían. Este incremento cierra la brecha:

- `agents-md-template.md` y `copilot-instructions.template.md` se bundlean verbatim como constantes TS (`AGENTS_MD_TEMPLATE`/`COPILOT_INSTRUCTIONS_TEMPLATE`, `apps/cli/src/commands/workspace-adapter-templates.ts`) — TS string constants porque `tsc -b` no copia assets no-TS a `dist`.
- `generateOpencodeConfig`/`generateClaudeCodeConfig` aceptan un parámetro opcional `templateContent?: string`: cuando se provee, se usa en vez de leer `templatePath` de disco; omitido, el comportamiento es byte-por-byte idéntico al previo (back-compat total, `configure`/`sync` intactos).
- `generateWorkspaceAdapters` pasa `AGENTS_MD_TEMPLATE` como `templateContent` a opencode/claude-code y `COPILOT_INSTRUCTIONS_TEMPLATE` como plantilla a `writeCopilotInstructions`. Resultado: opencode/claude-code/copilot-vscode ahora producen ficheros de instrucciones reales y no vacíos en un repo recién creado, sin warnings `template-missing`.

### Baseline de Axiom SKILLS scaffoldeada en el repo SDD recién creado — INC-20260705-workspace-sdd-skills

Cuando `runWorkspaceSetup` **crea desde cero** el repo de control/SDD (`created === true`), scaffoldea una baseline de SKILLS de Axiom en él como capacidad de agente, usando la API real de `@axiom/skills` (`loadSkillRegistry`, `applySkillSet`, `loadSkillsRoleIndex`/`validateSkillsRoleIndex`, `computeSkillBundleHash`). La lógica vive en `apps/cli/src/commands/workspace-skills.ts` (`scaffoldSddSkills`). Es best-effort, gateada y exclusiva del repo de control — las skills son una preocupación del repo SDD, no del repo de spec ni de los repos de rol.

- **Catálogo semilla bundleado**: `axiom.config/skills-catalog.yaml` (`schemaVersion: 1`) sembrado con un conjunto canónico pequeño de 5 ids — 3 ids canónicos de Axiom (`axiom-sdd-orchestrator`, `axiom-context-persistence`, `axiom-capability-router`) más los dos `DEFAULT_DESIRED_SKILLS` (`serena-this-project`, `context7-this-project`), de modo que un `applyDefaultSkillSet` también resolvería contra este catálogo. El contenido fuente de cada skill se bundlea como constantes TS y se escribe bajo `axiom.spec/target-axiom-skills/<id>.md`; el `bundleHash` de cada entrada se computa con `computeSkillBundleHash` (hash byte-exacto de la fuente bundleada), así que el catálogo es internamente consistente.
- **Materialización**: `applySkillSet` materializa `.opencode/agents/<id>/SKILL.md` por skill sembrada y `.axiom-state/<projectId>/skills-pending.json`.
- **Índice de skills por rol**: se escribe un `axiom.config/skills-index/<role>.yaml` por cada rol funcional declarado en el workspace (ids derivados de `role.functionalRoleId ?? role.roleKey` de cada repo `kind: 'role'`), validado con `validateSkillsRoleIndex` antes/después de escribir (warnings, nunca bloqueante). Comparte el schema `SkillsRoleIndex` de RF-AXM-020 (ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md)); `axiom-sdd-orchestrator` + `axiom-context-persistence` se marcan `mandatory` por rol y el resto `available`.
- **Gating + no-clobber**: solo corre si el repo de control se creó recién en esta misma llamada; si ya existía, se salta por completo (nunca clobbera). El paso entero es idempotente y best-effort — cualquier throw de `@axiom/skills` se degrada a un único warning sin abortar el setup.

## Plugins externos (confirmado ya cierto: Azure DevOps es opcional y no bloqueante)

Confirma que "las integraciones externas son plugins opcionales, Axiom Core funciona sin ellos" ya es cierto hoy:

- `apps/cli/src/commands/app-plugins-azure-devops.ts` es una constante `AppPlugin` **puramente declarativa** (schema de formulario: tabs, acciones, campos) — sin cliente HTTP, sin SDK de Azure DevOps, sin manejo de credenciales. Sus campos `command` referencian `axiom-external-sync-command`, uno de los intent commands `notImplemented` de `@axiom/orchestrator`. **Ninguna acción que este plugin declara puede ejecutarse hoy realmente.**
- Estructuralmente aislado de Axiom Core: solo alcanzable a través de la `axiom app` (PWA/API local opcional). Ningún comando o paquete estructural bajo `Axiom/packages/*` importa ninguno de los dos ficheros del plugin; borrar ambos ficheros solo afectaría a `app-api.ts` y sus propios tests.
- El `field.externalRef?: boolean` del plugin es un flag de UI no relacionado — no lee ni escribe el mecanismo real de artefacto `externalRefs` (ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md)). Esto es correcto, no un hueco: cablearlo antes de que exista una ruta de ejecución real sería código especulativo sin llamador.
- Una vez exista una ruta de ejecución real de Azure DevOps contra una API en vivo, debería persistir resultados vía el mecanismo `externalRefs` existente en vez de inventar uno nuevo. No programado.