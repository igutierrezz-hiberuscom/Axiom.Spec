# 02 Cambios de Modelo

## Estado actual

es un camino adicional y `memory-mcp-server` se reserva para memoria.
Antes del incremento, el runtime distinguía los kinds `sdd`, `spec`, `memory` y
`axiom`, y el setup podía materializar brokers SDD y Spec separados. El
protocolo wire era MCP `2024-11-05` con handshake `initialize`. Este párrafo es
la fotografía previa al cambio, no el contrato vigente.

## Estado objetivo

| Elemento | Estado objetivo |
|---|---|
| `McpServerKind` | Solo `axiom` |
| Server gestionado | Solo `axiom-mcp-broker` |
| Domains internos | `sdd`, `spec`, `axiom`, `memory`; no son procesos |
| Configuración | Una entrada en `mcp-manifest.yaml`, `.axiom/mcp.yml` y cada schema nativo aplicable |
| Protocolo | MCP `2026-07-28`, sin handshake legacy |
| Transporte | stdio newline-delimited |
| Mutaciones | Preview/confirmación existentes, con aislamiento project-scoped |
| Tasks | No implementado en este incremento |

El estado vigente es `McpServerKind = 'axiom'` y una única entrada gestionada
`axiom-mcp-broker`. Los dominios de capability se conservan como clasificación
interna del registry; no representan procesos MCP independientes.

## Contrato de respuesta

- `server/discover` devuelve `resultType: complete`, versiones soportadas,
  capabilities, identidad del servidor, `ttlMs` y `cacheScope`.
- `tools/list` devuelve `resultType: complete`, tools deterministas, `ttlMs` y
  `cacheScope`.
- `tools/call` devuelve `resultType: complete` y conserva el envelope de tool
  (`content` e `isError`) existente.
- Una versión no soportada devuelve el error MCP `-32022`.

## Migración de superficie

Las entradas y helpers que hoy distinguen SDD y Spec deben converger en una
entrada `axiom-mcp-broker`. Los nombres de capability no se renombran, porque
son parte del registry y de los contratos de handlers; cambia la frontera del
proceso, no la identidad funcional de cada tool.
