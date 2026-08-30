# 04 Interacciones UI

## Superficies

CLI `topology show|validate`, `roles`, launcher y MCP que proyectan el grafo.

## Flujo

Toda superficie localiza primero `axiomRepo`, carga el manifest autoral y usa el validator común. Las vistas pueden mostrar el origen autoral y, si existe, el estado de una proyección derivada.

## Estados visibles

Los errores distinguen carga/schema de findings semánticos. Nunca se muestra un manifest fallback sintético como si fuera real. Los detalles de envelopes se fijan en C.

## Reactividad

Una mutación autoral invalida/regenera proyecciones; no reancla y reescribe copias independientes. Axiom.SDD permanece read-only en el dogfood.
