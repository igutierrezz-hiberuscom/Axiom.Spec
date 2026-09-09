# 03 Criterios de Aceptación

## Telemetría

- **CA-H1 / ACC-075**: archivo grande demuestra bytes/eventos acotados; últimos 20 newest-first son correctos, con <20 y corrupción incluidos.
- **CA-H2**: dos project roots devuelven solo sus eventos; process counters están etiquetados y no se presentan como project-scoped.
- **CA-H3**: búsqueda/import test demuestra que launcher no usa fs/auditLogPath; solo APIs sancionadas leen audit trail.
- **CA-H4**: lessons no aparece en response, routes, frontend ni tests activos.

## Matriz ACC-076

Debe cubrir con server/wrapper efectivo: sesión/origin/CORS, content-type/body/timeouts, headers/browse, SSE auth/limits/backpressure, SSRF/protocol/DNS/redirect/timeout/size, delivery ack, preview binding/race/edit/replay/expiry/required, catálogo/no-fallback/comandos, ID mismatch en todas las lanes, ADO local-first/URL schemes y telemetría order/scope.

Cada caso negativo verifica ausencia de mutación cuando aplica. Conteos PASS/FAIL exactos son requisito de cierre; timeout se divide y no se declara éxito.

## Evidencia

Suites telemetry + 20 suites launcher existentes + nuevas regresiones, build/typecheck y diff-check.
