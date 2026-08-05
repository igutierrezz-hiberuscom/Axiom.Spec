# Requisitos

## RQ-001. Contrato declarativo versionado

El plugin puede declarar `schemaVersion: 1`, capacidades, tabs, acciones y
campos. `command` se conserva como etiqueta informativa/compatibilidad; nunca
es una autoridad de ejecucion. Los plugins legacy sin `schemaVersion` pueden
descubrirse, pero una accion sin `handler` explicito queda declarativa y no
ejecutable.

## RQ-002. Discovery tolerante y determinista

La ausencia de la carpeta, JSON malformado, schema no soportado, archivos no
JSON y ids duplicados producen warnings y no detienen el resto del catalogo.
Los archivos se procesan en orden estable y el endpoint existente conserva la
forma `plugins` + `warnings`.

## RQ-003. Allowlist y resolucion separada

Toda accion ejecutable debe tener un `handler` del registro estatico del
runtime. La resolucion valida handler, clase de accion, command compatible y
campos allowlisted, pero nunca tokeniza, spawnea o interpreta el command como
proceso. Se rechazan handlers/commands desconocidos, metacaracteres, paths,
binarios y scripts antes de alcanzar tracker, filesystem o runner.

## RQ-004. Gates por clase de accion

Read-only puede ejecutarse sin confirmacion y no realiza mutaciones. Toda
accion `local-mutation` o `external-mutation` devuelve primero un preview sin
writes ni red, y solo ejecuta con `confirmed: true`. El endpoint existente de
launcher sigue siendo compatible y el endpoint plugin-scoped delega al mismo
dispatcher.

## RQ-005. Plugins opcionales y Azure DevOps

La ausencia o desactivacion de plugins no rompe core, CLI, doctor ni MCP. El
bridge ADO reutiliza `runExternalSync`, `IWorkItemTracker`, `NullTracker`,
`AdoWorkItemTracker` y los ports existentes. `kind: none` es local-only y no
hace red; `kind: ado` se prueba mediante seams/fakes, nunca contra el servicio
real.

## RQ-006. Resultados honestos

El catalogo expone origen, warnings, capacidades y estado configurado/no
configurado. La UI distingue acciones no ejecutables, preview y resultado
aplicado; no afirma que una declaracion fue aplicada si no hubo handler o
confirmacion.

## RQ-007. Secretos y respuestas

Las credenciales permanecen en los ports/configuracion segura existentes. No
se serializan en JSON de plugin, registry, localStorage, UI, logs ni
respuestas. El puente proyecta solo mensaje, exitCode, estado local y
`externalRefs` necesarios.
