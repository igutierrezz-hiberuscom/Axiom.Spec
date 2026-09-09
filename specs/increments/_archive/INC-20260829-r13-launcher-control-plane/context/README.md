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

La matriz `Axiom/apps/cli/tests/r13-acc-076-matrix.test.ts` ejecuta 35 casos de ACC-070..ACC-075 más un test de resumen: `PASS=35 FAIL=0 TIMEOUT=0`, con 14 snapshots explícitos de ausencia de mutación. Incluye required-fields de catálogo, onboarding, plugins, roles/Git y ADO, además de server/wrapper real, fixtures loopback y fake ADO.

La validación complementaria observada es `npm run typecheck`, `npm run build`, `npm run doctor`, `npm run readiness:first-project` y `git diff --check`, todos PASS. Core conserva los receipts previos y emitirá los receipts finales de freeze/verify/knowledge/archive.

## Provenance

La provenance es reproducible a nivel de candidate freeze + matriz + receipts y declara rutas compartidas entre las lanes F/G/H; no se ejecutan commits ni operaciones Git. El drift de comentarios históricos de VSCode y el alcance no demostrado del deep-link SPA anidado son observaciones no bloqueantes.

## Qué no debe vivir aquí

No deben vivir metadata estructural, IDs, status, enlaces, receipts manuales ni índices generados; tampoco la especificación normativa completa ni secretos. Los cambios estructurales y lifecycle se gestionan mediante Axiom Core.
