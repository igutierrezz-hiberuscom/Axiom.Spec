# 01 Requisitos

- **F-070.1**: bind no-loopback se rechaza; API/SSE requieren sesión local válida y same-origin, sin CORS wildcard.
- **F-070.2**: métodos/content-type/body/timeouts/schemas/errores/headers son uniformes y browse queda dentro de raíces de topology.
- **F-070.3**: SSE tiene heartbeat, subscriber/backpressure limits y cleanup.
- **F-071.1**: HttpLaunch usa endpointId configurado y controles SSRF/redirect/timeout/response; body no controla destino.
- **F-071.2**: clipboard/VS Code distinguen estados y nunca declaran delivery sin ack; transporte VSCode ficticio se retira.
- **F-072.1**: todo execute mutante exige token preview single-use ligado a payload/proyecto/acción/session/TTL.
- **F-072.2**: frontend resuelve carreras, desarma confirmación al editar, usa snapshot y valida required fields en todas las superficies.
- **F-076.F**: tests ejercitan server/wrapper real y negativos sin mutación ni red externa.

## Regla de seguridad

Ningún booleano del cliente, incluido `confirmed`, concede autorización o sustituye token/preflight.
