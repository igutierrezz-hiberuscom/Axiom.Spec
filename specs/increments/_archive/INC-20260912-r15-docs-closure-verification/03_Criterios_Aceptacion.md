# 03 Criterios de Aceptación

## Criterios de aceptación

### AC-088-01: Alcance completo cubierto
[x] La verificación contempla el alcance causal del artefacto y su plan asociado; los paths documentales externos se declaran explícitamente con `affectedPaths`, cubriendo la especificación, manual, contexto, decisiones, planes y READMEs cuando aplican.

### AC-088-02: Detección de documentación sin revisar
[x] Un cierre con documentación en alcance sin revisar produce un resultado accionable que nombra el documento y el motivo; está cubierto por los tests de warning/block del runner.

### AC-088-03: Cierre limpio cuando todo está revisado
[x] Un cierre con la documentación revisada pasa y deja la decisión estructurada con estados `updated`/`unchanged` y motivos en resultado y receipt.

### AC-088-04: Cambio sin impacto documental
[x] Un cambio sin impacto documental se cierra sin fricción y se declara como `no-documentation-impact`.

### AC-088-05: Un solo camino
[x] La verificación se ejecuta desde `runGovernedTransition` y aplica igual desde CLI, launcher y MCP; no existe una ruta alternativa de archive.

### AC-088-06: Progresión aviso a bloqueo
[x] `warning` y `block` están implementados; la transición de aviso a bloqueo se prueba después de confirmar y no altera artefactos cuando bloquea.

### AC-088-07: Sin regresión de gobierno
[x] Ninguna transición ilegal pasa a ser legal, la confirmación sigue siendo obligatoria y no aparece edición manual de estado; los casos negativos pasan.

### AC-088-08: Sin redacción automática
[x] La verificación solo calcula y registra la decisión; no crea ni modifica contenido documental.

### AC-GEN-01: Integridad general
[x] `npm run build`, las suites focales de workflow/CLI/MCP/launcher, `npm run doctor`, `npm run readiness:first-project` y `git diff --check` pasan; doctor mantiene 0 fallos con advertencias/omisiones preexistentes.
