# Requisitos

1. La documentacion operativa debe describir solo la CLI y configuracion
   vigentes: builder/local-only implicitos, targets, roles y fases.
2. Las plantillas product-owned no deben generar instrucciones para gateway,
   overlays retirados, generated snapshots o perfiles funcionales antiguos.
3. Deben quedar documentados el audit trail P365D, MCP real independiente y
   la separacion de capabilities provider-routed/MCP-only.
4. Los hechos no implementados deben expresarse como warning, fallback o
   compatibilidad legacy segun corresponda, nunca como capacidad vigente.
