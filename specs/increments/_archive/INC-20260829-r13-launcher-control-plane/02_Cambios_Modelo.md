# 02 Cambios de Modelo

## Sesión

`LauncherSession` vive solo en memoria del proceso, tiene token hash/createdAt y se transporta por cookie HttpOnly. Middleware común autentica y valida origin antes del router.

## Preview

`PreviewGrant` almacena token hash, sessionId, projectId, actionId, payload canonical JSON/digest, diagnostics, expiresAt y estado `issued|consumed`. El store consume mediante operación atómica.

## Delivery

`DeliveryState = generated | client-instructed | acknowledged | delivered`; `delivered` requiere evidence. `endpointId` reemplaza URL request-controlled. VSCode transport se retira; clipboard puede avanzar tras ack explícito del cliente.

## HTTP/SSE

Schemas cerrados y errores públicos genéricos. Limits/config centralizados. SSE subscriber registry aplica cuotas/queue y no incluye secretos.

## Compatibilidad

Se rechazan contratos anteriores sin sesión/token y `confirmed:true`.
