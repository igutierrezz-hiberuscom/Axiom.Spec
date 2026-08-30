# 03 Criterios de Aceptación

## Control plane

- **CA-F1 / ACC-070**: server real rechaza host no-loopback, sesión ausente/falsa, origin cross-site, OPTIONS CORS, método/content-type/body/schema inválidos y traversal/symlink escape.
- Respuestas incluyen headers; 500 no filtra path/stack; body timeout/limit corta conexión de forma controlada.
- SSE autenticado admite hasta límite, emite heartbeat, limpia close y desconecta backpressure.

## Transportes

- **CA-F2 / ACC-071**: endpoint request-controlled, protocolos/hosts/IP/redirects prohibidos, timeout y respuesta oversized fallan sin enviar prompt a destino no allowlisted.
- Fixtures HTTP locales cubren éxito allowlisted y redirects; no hay red externa.
- Clipboard progresa con ack; VSCode ficticio no existe; ninguna ruta marca delivered sin evidence.

## Confirmación

- **CA-F3 / ACC-072**: token correcto ejecuta una vez; payload/proyecto/acción/session distintos, expiry, replay y dos executes concurrentes fallan cerrado.
- Crafts fuera de orden no reemplazan preview; edición desarma confirmación; execute usa snapshot; required fields bloquean todas las superficies.

## Ausencia de mutación

Todo rechazo previo al execute deja artefactos/estado funcional idénticos; solo contadores de seguridad in-memory permitidos.

## Evidencia

Matriz ACC-076 focalizada en server/wrapper/frontend helpers, suites launcher existentes, build/typecheck y diff-check.
