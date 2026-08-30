# 02 Cambios de Modelo

## Objetivo del documento

Delimitar las superficies que deben reflejar el modelo MCP ya ejecutable.

## Entidades o estructuras afectadas

- Prosa activa en docs del runtime, spec canónica, contexto técnico y plan R-11.
- Comentarios cercanos al dispatcher de `@axiom/mcp-server` y sus handlers si describen el modelo anterior.
- Configuración declarativa `mcp-manifest.yaml` e `integrations.yaml` como fuentes de contraste, sin cambiar su estructura salvo que la evidencia revele un claim incorrecto.

## Contratos o estados afectados

No cambia el contrato binario ni el schema: `McpServerKind` continúa restringido al kind `axiom`; el broker sigue despachando directamente a `invokeMcpTool`. La corrección solo elimina declaraciones documentales activas incompatibles.

## Notas de compatibilidad

Los IDs retirados pueden permanecer en rutas de limpieza/migración y en historia archivada. Esa presencia no implica que sean procesos activos.
