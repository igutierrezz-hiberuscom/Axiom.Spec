# Plantilla de Copilot Instructions para Axiom

<!-- axiom:variables
  {{project.name}}       <- product.manifest.project.name
  {{project.id}}         <- product.manifest.project.id
  {{mcp.allowlist}}      <- union(product.manifest.providers, providers.yaml#providers, local-overlay.nonSharedProviderBindings)
  {{support.level}}      <- environment-support-matrix.supportLevel
  {{target.id}}          <- adapter target id (github-copilot)
-->

<!-- AXIOM:GENERATED:START -->
Este template sirve como base para un futuro .github/copilot-instructions.md dentro de un proyecto Axiom.

## Alcance

- Trabajar solo dentro de este proyecto Axiom.
- Respetar boundaries: _builder, axiom.spec, runtime.
- No tratar tooling del builder como runtime del producto.

## Protocolo de commands

- Priorizar comandos explícitos para acciones mutantes.
- Permitir router /axiom para lenguaje natural.
- Pedir confirmación ante ambigüedad o riesgo.

## Alcance de tools y MCP

MCPs y tools permitidos para este proyecto Axiom:
{{mcp.allowlist}}

No usar MCPs ni tools de otros proyectos aunque sean visibles.
Si una tool requerida no está listada aquí, debe tratarse como no disponible.

## Fallbacks

- Si falta una tool opcional, aplicar fallback documentado.
- Reportar cuando el resultado quede degradado por falta de capacidades.
<!-- AXIOM:GENERATED:END -->

<!-- TEAM:CUSTOM:START -->
## Team custom notes

<!-- TEAM:CUSTOM:END -->
