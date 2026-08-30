# 04 Interacciones UI

## Bootstrap

Abrir `/launcher/` establece sesión local; si expira/reinicia se solicita recarga. No se muestra token ni se coloca en URL/storage.

## Craft/confirm

Cada craft tiene requestId; solo la última respuesta vigente arma preview. Cualquier cambio de acción/campo invalida token y preview. Required fields se marcan y validan con el mismo schema antes de craft y execute.

## Delivery

La UI muestra generated, instrucción cliente, ack y delivery real por separado. Clipboard solicita copy y reporta ack; VS Code usa clipboard hasta existir bridge real.

## Errores

Mensajes públicos son genéricos/accionables; detalles técnicos no se renderizan. SSE reconecta solo tras sesión válida y aplica backoff.
