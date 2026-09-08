# Context

## Propósito

Conservar el contexto técnico estable y específico de R-13 que ayuda a revisar o mantener el control plane del launcher, sin sustituir la spec canónica ni duplicar el historial completo del apply.

## Hechos estables reconciliados

- `axiom app` es un servidor local: solo acepta bind literal `127.0.0.1` o `::1`, crea una sesión por proceso y exige cookie, `Host` y `Origin` local en la API.
- Los bodies mutantes son JSON con schema cerrado, límite de 256 KiB y timeout de lectura de 5 s; los errores 5xx se sanitizan.
- Browse y static serving canonicalizan rutas y rechazan traversal/symlink escape; SSE tiene sesión, límite de suscriptores, backpressure, heartbeat y cierre explícito durante shutdown.
- `HttpLaunch` obtiene el endpoint de configuración, valida allowlist/protocolo/IP, fija DNS, desactiva redirects y limita timeout/respuesta; no existe un transporte VS Code efectivo. Clipboard queda `client-instructed` hasta ack/evidence.
- Craft/execute usan grants server-side single-use con TTL de 120 s y binding por sesión, proyecto, acción, payload normalizado y adapter cuando aplica. Targets plugin/launcher ambiguos se rechazan antes de consumir o despachar.
- Doctor aporta diagnóstico server-side; nunca convierte sus checks en autorización de una mutación.

## Evidencia final

La verificación del 2026-09-08 registró 8 suites y 192 tests, typechecks de launcher/CLI, build, doctor PASS y readiness PASS. El lifecycle Core alcanzó `verifying` y emitió receipts de `increment-verify` y `verify`.

## Pendiente

El incremento no se archiva ni se marca cerrado: el worktree observado contiene cambios concurrentes sin commit focal y la provenance no permite atribución exclusiva a R-13. La review dejó además como observaciones no bloqueantes el drift de comentarios históricos de VSCode y el alcance no demostrado del deep-link SPA anidado.

## Qué no debe vivir aquí

No deben vivir metadata estructural, IDs, status, enlaces, receipts manuales ni índices generados; tampoco la especificación normativa completa ni secretos. Los cambios estructurales y lifecycle se gestionan mediante Axiom Core.
