# 01 Requisitos

- **H-075.1**: telemetry ofrece única API tail bounded/validated; launcher no importa fs/auditLogPath ni lee audit.log.
- **H-075.2**: recentEvents son los últimos 20 newest-first y agregados usan ventana acotada del proyecto correcto.
- **H-075.3**: corrupción/I/O/archivo ausente se distinguen; dos roots no mezclan eventos.
- **H-075.4**: métricas audit project-scoped y counters process-wide se presentan en bloques/labels separados; lessons ausente.
- **H-076.1**: matriz agregada cubre todos los escenarios ACC-070..075 en server/wrapper real, incluidos negativos/ausencia de mutación.
- **H-076.2**: fixtures HTTP/ADO son herméticos y no hay red externa; se registran PASS/FAIL exactos.

## Fuera de alcance

Persistir counters por proyecto o recuperar lessons.
