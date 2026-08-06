# Requisitos del incremento

## R1. Politica unica

El runtime debe comportarse como `builder` + `local-only` sin que el usuario
seleccione esos valores. El target de adaptador y los roles de equipo siguen
siendo entradas independientes.

## R2. Compatibilidad de estado

`init.json`, `install-profile.json`, `workspace.json`, registry y cualquier
registro equivalente deben poder leerse cuando provienen de una version
anterior. La lectura debe convertir `product-owner`, `analista`,
`arquitecto`, `standard` y `enterprise` a la forma efectiva nueva. Las nuevas
escrituras no deben emitir esos identificadores.

## R3. Retirada de gateway y snapshots

La configuracion activa, schemas, tipos, routing, doctor y CLI no deben
ofrecer `axiom-gateway`, modos gateway, `gateway-state.json` ni
`generated-snapshots`. El backend JSON de memoria y los brokers MCP quedan
fuera de la retirada.

## R4. Trazabilidad

Toda mutacion continua emitiendo al `AuditTrailSink` local. El log es
append-only, conserva su sidecar SHA-256 y aplica una sola ventana de
retencion `P365D`. `axiom audit` mantiene su lectura read-only y sus estados de
verificacion.

## R5. No regresion de targets y MCP

Los diez targets canonicos siguen siendo validos y los brokers/handlers MCP
reales siguen registrados y probados.
