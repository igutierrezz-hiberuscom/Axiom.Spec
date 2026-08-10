# 03 Criterios de Aceptación

## Criterios de aceptación

- [x] El registry y `tools/list` del único `axiom-mcp-broker` exponen la unión
      completa de `sdd.*`, `spec.*`, `axiom.*` y `memory.*`.
- [x] `McpServerKind` solo acepta `axiom`; `sdd`, `spec` y `memory` ya no son
      kinds de servidor gestionados.
- [x] `mcp-manifest.yaml`, `.axiom/mcp.yml`, `member-install` y las
      proyecciones nativas materializan una sola entrada `axiom-mcp-broker`.
- [x] `server/discover` devuelve la forma MCP `2026-07-28` y una petición sin
      versión compatible recibe `UnsupportedProtocolVersionError` (`-32022`).
- [x] `initialize`, `notifications/initialized` y `ping` no forman parte del
      protocolo implementado.
- [x] `tools/list` y `tools/call` incluyen `resultType: complete`; las respuestas
      cacheables incluyen `ttlMs` y `cacheScope`.
- [x] El cliente stdio de Axiom usa metadata moderna y no intenta un handshake
      legacy.
- [x] La misma instancia puede leer un incremento o bug y ejecutar una
      transición de estado en preview y confirmación.
- [x] Las mutaciones siguen exigiendo confirmación y respetan el aislamiento de
      proyecto, repo y scope.
- [x] `notifications/cancelled` y `progressToken` cumplen las reglas básicas de
      MCP sin introducir Tasks.
- [x] Engram continúa funcionando como backend/provider de `memory.*` sin crear
      un segundo server gestionado.
- [x] Tests unitarios, tests de configuración, E2E stdio real, `npm run build`,
      doctor relevante y revisión independiente pasan; las fallas preexistentes
      quedan clasificadas.
- [x] La documentación activa y `Axiom.Spec/context/**` dejan de presentar los
      brokers SDD, Spec y memory como servidores gestionados independientes.

### Happy path

1. Un proyecto materializa una sola configuración MCP.
2. Un cliente descubre `axiom-mcp-broker` y negocia `2026-07-28`.
3. El cliente lista la superficie completa.
4. El cliente lee un incremento o bug.
5. El cliente solicita una transición; recibe preview.
6. El cliente repite con confirmación; el estado cambia y queda trazabilidad.

### Validaciones y errores

- Binding ausente, ambiguo o de otro proyecto: fallo cerrado.
- Versión no soportada: error MCP `-32022`.
- Método legacy: método no encontrado.
- Tool no registrada o fuera de la superficie: error de tool sin excepción.
- Transición ilegal o mutación sin confirmación: resultado de error/preview
  conforme a los handlers existentes.
- Cancelación de petición activa: no se emite una respuesta posterior para esa
  petición.

### Permisos y visibilidad

- El broker solo ve el proyecto que se le pasa al arrancar.
- La lista de tools no revela brokers, rutas o proyectos ajenos.
- Las tools de escritura siguen bajo el control de los handlers existentes.

### Estados y efectos observables

- El estado del protocolo es stateless por petición.
- Los estados persistidos de workflow e incrementos se modifican únicamente
  mediante las mutaciones existentes y confirmadas.
- No se crea ni persiste estado de Tasks en esta entrega.
