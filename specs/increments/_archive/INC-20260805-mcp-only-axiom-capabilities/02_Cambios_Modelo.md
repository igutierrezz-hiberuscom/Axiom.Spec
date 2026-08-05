# Cambios de modelo: superficie MCP-only para `axiom.*`

El modelo separa dos catálogos que hoy se solapan parcialmente:

- capabilities provider-routed: las que `@axiom/capability-model`,
  `providers.yaml` y `routeTool` usan para seleccionar un provider;
- capabilities MCP-only: ids registrados por `@axiom/mcp-tools` y expuestos
  por un kind MCP concreto.

Los tres ids `axiom.*` pertenecen al segundo grupo. No se crea un provider
nuevo para representarlos. La fuente operativa de sus handlers sigue siendo
`@axiom/mcp-tools`; el broker puede utilizar `axiom-gateway` como proceso o
transporte sin convertir esos ids en entradas del registry tradicional.
