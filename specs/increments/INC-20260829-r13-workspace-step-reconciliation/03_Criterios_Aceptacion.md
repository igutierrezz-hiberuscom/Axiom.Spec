# 03 Criterios de Aceptación

- **CA-I1**: una prueba de matriz demuestra que cada output reparable de setup tiene owner y comando granular o exclusión explícita.
- **CA-I2**: instalación parcialmente dañada se repara por adapters/rules/config-scaffold/mcp-config y una segunda ejecución es byte-estable.
- **CA-I3**: adapters incluye process surfaces pero no MCP; rules conserva contenido humano de AGENTS; config-scaffold cubre el set de setup.
- **CA-I4**: path ajeno y target desconocido en cada comando producen error tipado y snapshot before/after idéntico.
- **CA-I5**: desde axiomRepo y code repo se selecciona la misma authority/outputs.

Evidencia: suites workspace granulares/setup, generators afectados, build/typecheck y diff-check.
