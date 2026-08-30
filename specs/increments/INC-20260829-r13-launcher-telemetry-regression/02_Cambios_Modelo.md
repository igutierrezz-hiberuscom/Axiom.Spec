# 02 Cambios de Modelo

## API tail

`AuditTail` incluye eventos válidos, parseErrors, bytesRead, linesExamined, truncated y orden. Límites se validan y clamp/fail según contrato; nunca hay lectura ilimitada.

## Respuesta launcher

`projectMetrics` contiene auditTrail/aggregates/recentEvents ligados al root. `processMetrics` incluye `scope: process-wide` y counters de bus. El shape está versionado por el envelope HTTP.

## Corrupción

Líneas inválidas incrementan parseErrors sin exponer contenido. Error de archivo/confinamiento es `AuditReadError` tipado. Archivo ausente produce tail vacío válido.

## Compatibilidad

Se retira `busCounters` ambiguo y cualquier acceso directo; lessons sigue retirado.
