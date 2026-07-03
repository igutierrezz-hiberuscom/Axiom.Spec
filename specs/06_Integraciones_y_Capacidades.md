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

`@axiom/app` (portable operator app, incremento `0030`) expone un sistema de plugins project-scoped con discovery en `.sdd/<project>/app-plugins/*.json` y un bridge declarativo hacia Azure DevOps (mutación externa exige `confirmed: true`; los plugins no introducen lógica de negocio paralela). No hay integración con Jira/Confluence implementada — solo mencionada como posible extensión en el roadmap futuro (`INC-16`).

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

## Plugins externos (confirmado ya cierto: Azure DevOps es opcional y no bloqueante)

Confirma que "las integraciones externas son plugins opcionales, Axiom Core funciona sin ellos" ya es cierto hoy:

- `apps/cli/src/commands/app-plugins-azure-devops.ts` es una constante `AppPlugin` **puramente declarativa** (schema de formulario: tabs, acciones, campos) — sin cliente HTTP, sin SDK de Azure DevOps, sin manejo de credenciales. Sus campos `command` referencian `axiom-external-sync-command`, uno de los intent commands `notImplemented` de `@axiom/orchestrator`. **Ninguna acción que este plugin declara puede ejecutarse hoy realmente.**
- Estructuralmente aislado de Axiom Core: solo alcanzable a través de la `axiom app` (PWA/API local opcional). Ningún comando o paquete estructural bajo `Axiom/packages/*` importa ninguno de los dos ficheros del plugin; borrar ambos ficheros solo afectaría a `app-api.ts` y sus propios tests.
- El `field.externalRef?: boolean` del plugin es un flag de UI no relacionado — no lee ni escribe el mecanismo real de artefacto `externalRefs` (ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md)). Esto es correcto, no un hueco: cablearlo antes de que exista una ruta de ejecución real sería código especulativo sin llamador.
- Una vez exista una ruta de ejecución real de Azure DevOps contra una API en vivo, debería persistir resultados vía el mecanismo `externalRefs` existente en vez de inventar uno nuevo. No programado.