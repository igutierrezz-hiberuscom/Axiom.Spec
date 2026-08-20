# 02 Cambios de Modelo

## Objetivo del documento

Reparar residuos de contratos retirados detectados por TypeScript sin reabrir decisiones R-07 ni cambiar la política de adapters.

## Entidades o estructuras afectadas

- Registry de outputs del installer y los ocho targets vigentes.
- Callers MCP alineados al contexto/protocolo público actual.
- Integración Visual Studio y pruebas de surfaces modificadas.

## Contratos o estados afectados

LiteLLM y `copilot-vscode` no son targets ni dispatches activos. El registry expone sólo `opencode`, `claude-code`, `antigravity`, `visual-studio-2026`, `cursor`, `github-copilot`, `vscode` y `codex`. Se conserva la validación MCP project-scoped y el generator público de Visual Studio.

## Notas de compatibilidad

No se restaura ningún alias ni provider retirado. Los cambios corrigen coherencia de tipos y registro, no añaden comportamiento lifecycle.
