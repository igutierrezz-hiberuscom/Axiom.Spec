# 01 Requisitos

## Objetivo del documento

Definir el contrato documental de ACC-046 sin cambiar la semántica MCP del runtime.

## Requisitos del incremento

1. La documentación activa describirá un único broker gestionado: `axiom-mcp-broker`.
2. Indicará que se inicia con `axiom mcp serve --kind axiom`, es project-scoped y sirve los handlers internos publicados por `@axiom/mcp-tools` mediante `@axiom/mcp-server`.
3. Los nombres `sdd-mcp-server`, `spec-mcp-broker` y el kind `memory` se conservarán solo cuando expresen compatibilidad/historia o limpieza de configuración anterior.
4. Engram se describirá como integración MCP local de tercero y backend opcional de memoria, no como un segundo broker gestionado.
5. El plan, la spec canónica y el contexto técnico no tendrán claims activos contradictorios.

## Reglas de negocio relevantes

La evidencia ejecutable tiene prioridad: `McpServerKind`, manifest, configuración de integraciones, registro de tools y pruebas determinan el estado vigente. Los receipts e incrementos archivados no se modifican.

## Fuera de alcance funcional

No se añade ni retira ningún handler, provider, servidor ni comando MCP; es una reconciliación documental y de comentarios activos.
