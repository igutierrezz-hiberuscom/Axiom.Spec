# Criterios de aceptación: política de cobertura de capabilities en Doctor

- **AC-01**: una capability canónica requerida sin provider produce `fail`.
- **AC-02**: una capability canónica opcional o post-MVP sin provider produce
  `warn` no bloqueante.
- **AC-03**: una capability MCP-only no produce un falso fallo de provider.
- **AC-04**: una capability canónica ausente del YAML se diagnostica aparte de
  una capability declarada sin provider.
- **AC-05**: el proyecto real no informa un PASS de cobertura cuando faltan
  capabilities activas provider-routed.
- **AC-06**: las fixtures cubren requerida, opcional, MCP-only, deshabilitada,
  declaración ausente y provider presente.
- **AC-07**: build y pruebas dirigidas pasan.
