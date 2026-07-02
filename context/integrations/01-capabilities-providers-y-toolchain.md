# Capabilities, providers y toolchain externo

Fuente: `Axiom/docs/configuration/providers-and-capabilities.md`, `Axiom/docs/configuration/files/{capabilities,providers}.md`, informe de incrementos 0015-0030.

## Modelo de capabilities

`capabilities.yaml` define el catálogo y el marco de degradación: `capabilities.required` / `.optional` / `.postMvpOptional`, `supportLevels`, `degradationPolicy`. Cada capability declara `id`, `domain` (`sdd`, `spec`, `code`, `memory`), `name`, `version`, `compliance`, `requiredTools`, `optionalTools`, `fallbacks`, `deprecated`, `schemaRef`. Implementado en `@axiom/capability-model`.

## Providers y discovery

`providers.yaml` modela el registry de providers y sus perfiles declarativos de discovery. Discovery modes: `filesystem`, `workspace`, `gateway`. Discovery Provider Profiles (`filesystem-first`, `gateway-first`, `local-only`) definen `discoveryOrder`, `preferredProviders`, `optionalProviders`, `gatewayExpectation`, y modo degradado.

Regla del MVP: **Serena** es el baseline de inteligencia de código; el resto de providers son opcionales o evolutivos según capability y entorno. Reglas globales: Serena baseline, providers opcionales post-MVP, defaults MCP de producto, providers prohibidos por defecto.

Impacta directamente a: `configure` (compone install profile), `start` (resuelve provider efectivo y modo de discovery), `doctor` (evalúa consistencia del modelo y defaults).

## Toolchain externo (`@axiom/toolchain`)

Manifest `toolchain.yaml`: npm/pnpm/yarn/bun, Node, git, python — con detección dinámica (`detect.ts`), loader, validador y repair idempotente.

Tools P1 añadidas en el incremento 0027, todas con `mvp: true`, `detectionPaths`, `mcpServer`, `gitignoreEntries`: **CodeGraph**, **Graphify**, **Headroom/RTK**, **Caveman**, **Autoskills**. Bug corregido en ese incremento: `resolveDetectionPath` calculaba rutas relativas al `cwd` en vez de a `projectRoot`.

Comandos: `axiom toolchain repair [--id <id>]` (idempotente), `axiom toolchain add --id <id> --path <repoId>` (selección por repo), `axiom toolchain gitignore [--write <file>]` (output ordenado y deduplicado).

## MCP (Model Context Protocol) — estado actual, no un server genérico propio

Axiom no expone hoy un servidor MCP propio y genérico de proyecto (eso pertenece al roadmap futuro, `INC-13`/`INC-14` en `specs/increments/INC-20260702-axiom-redesign-roadmap/`). Lo que existe hoy:
- catálogo de MCP permitidos vía `@axiom/isolation` (`DEFAULT_ALLOWED_MCP_SERVERS`, 28 servidores MVP por defecto);
- comando `axiom mcp repair --id <mcpId>` (repair de binding, incremento 0029) y `axiom mcp inventory` (total, required, optional, readonly, por installMode);
- `backend-mcp`/`frontend-mcp` bloqueados explícitamente en MVP (verificado por `doctor`).

## Memoria y recall (`@axiom/memory`)

Curación auto/manual por `MemoryKind` (`decision`, `bug`, `learning`, `pattern`, `context`), con invariante scope + `projectId` por entry. GATE 0024: la memoria **no** es fuente de verdad; en conflicto, la spec prevalece. Ranking de recall (incremento 0029): text match (case-insensitive, start-boost, occurrence boost) + recencia (ventana 90 días) + kind boost (`decision=1.5`, `bug=1.4`, `pattern=1.2`, `learning=1.1`, `context=1.0`); cada hit devuelve `reason` explicativo. Recall es opt-in (helper disponible; wire-up al state machine diferido a la fecha del incremento 0029). Comando: `axiom memory inventory`.

## Extensiones opcionales: app plugins y bridge externo

`@axiom/app` (incremento 0030) implementa un sistema de plugins project-scoped con discovery en `.sdd/<project>/app-plugins/*.json` (schema guard tolerante: malformados → warnings, no abort; IDs únicos por proyecto) y un bridge declarativo hacia Azure DevOps (mutación externa exige `confirmed: true`). Los plugins no introducen lógica de negocio paralela — el bridge solo declara contrato. No hay integración con Jira/Confluence implementada; se menciona únicamente como extensión posible del roadmap futuro.
