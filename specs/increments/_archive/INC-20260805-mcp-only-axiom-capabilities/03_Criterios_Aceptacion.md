# Criterios de aceptación: superficie MCP-only para `axiom.*`

- **AC-01**: existe una clasificación explícita y testeada para los tres ids
  `axiom.*`.
- **AC-02**: el contrato TypeScript y los schemas ya no se contradicen sobre
  los dominios que acepta el modelo provider-routed.
- **AC-02a**: el mapa provider-routed rechaza `domain: axiom` y el mapa
  MCP-only rechaza ids `axiom.*` que no estén en el conjunto canónico.
- **AC-02b**: `config-validation` conserva `mcpOnlyCapabilities` al parsear
  un YAML válido.
- **AC-03**: `loadCapabilityModel` no exige providers para los ids MCP-only.
- **AC-04**: `DEFAULT_PROFILES` y el `profiles.yaml` dogfooded no incluyen los
  ids MCP-only como capabilities provider-routed habilitadas.
- **AC-05**: `MCP_TOOL_HANDLERS` y `AXIOM_TOOL_CAPABILITY_IDS` conservan los tres
  handlers y el broker no pierde ninguna herramienta.
- **AC-05a**: `CapabilityModel` y `RouteToolContextSchema` exponen el mapa
  MCP-only para los consumidores tipados.
- **AC-06**: hay una prueba que comprueba que un routing genérico no intenta
  resolver un id MCP-only como si fuera un provider capability.
- **AC-07**: build y pruebas dirigidas pasan.
