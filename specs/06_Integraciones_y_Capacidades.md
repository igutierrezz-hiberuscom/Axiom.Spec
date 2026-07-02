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