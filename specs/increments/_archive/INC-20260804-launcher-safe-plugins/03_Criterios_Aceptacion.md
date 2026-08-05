# Criterios de aceptación

- [x] Discovery/schema tolerante y versionado; malformed, legacy, schema no
      soportado y duplicados producen warnings sin crash.
- [x] Dispatcher allowlisted sin shell ni ejecucion arbitraria; command
      desconocido, handler desconocido, metacaracteres, paths y scripts se
      rechazan antes de ejecutar.
- [x] Read-only ejecuta sin confirmacion y las mutaciones local/externa pasan
      por preview y exigen `confirmed: true`.
- [x] Preview de mutacion no invoca el bridge; la confirmacion delega en
      `runExternalSync` y devuelve resultado seguro y `externalRefs`.
- [x] Azure DevOps reutiliza tracker/configuracion/secrets existentes; se
      probo `NullTracker` local-only sin red y ADO con fake transport.
- [x] Secretos ausentes de UI, logs y respuestas; el resultado del plugin se
      proyecta a un shape seguro.
- [x] Core y endpoint historico funcionan sin plugins; el GET conserva
      `plugins` + `warnings` y la UI marca declaraciones no ejecutables.
- [x] Tests de plugin/app/launcher/external-sync/tracker/tracker-ADO, build y
      E2E disponible ejecutados.
- [x] Freeze actualizado, receipt `verify`, review e integracion canonica
      documentados.
