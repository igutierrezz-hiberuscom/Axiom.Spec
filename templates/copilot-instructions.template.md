# Plantilla de Copilot Instructions para Axiom

<!-- axiom:variables
  {{project.name}}       <- product.manifest.project.name
  {{project.id}}         <- product.manifest.project.id
  {{mcp.allowlist}}      <- union(product.manifest.providers, providers.yaml#providers, local-overlay.nonSharedProviderBindings)
  {{support.level}}      <- environment-support-matrix.supportLevel
  {{target.id}}          <- adapter target id (copilot-vscode | copilot-cli)
-->

Este template sirve como base para un futuro .github/copilot-instructions.md dentro de un proyecto Axiom.

## Alcance

- Trabajar solo dentro de este proyecto Axiom.
- Respetar boundaries: _builder, axiom.spec, runtime.
- No tratar tooling del builder como runtime del producto.

## Protocolo de commands

- Priorizar comandos explicitos para acciones mutantes.
- Permitir router /axiom para lenguaje natural.
- Pedir confirmacion ante ambiguedad o riesgo.

## Alcance de tools y MCP

MCPs y tools permitidos para este proyecto Axiom:
- serena-this-project
- azure-devops-this-project
- engram-this-project

No usar MCPs ni tools de otros proyectos aunque sean visibles.
Si una tool requerida no está listada aquí, debe tratarse como no disponible.

## Fallbacks

- Si falta una tool opcional, aplicar fallback documentado.
- Reportar cuando el resultado quede degradado por falta de capacidades.