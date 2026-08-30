# 04 Interacciones UI

## Catálogo y preview

Acciones muestran campos realmente requeridos, incluido ID, y el comando `axiom ...` exacto. Error de reconciliación bloquea craft en vez de rutear genéricamente.

## ADO local-first

La UI confirma primero el resultado local. Después ofrece crear o vincular work item si ADO está configurado. Estado/fallo remoto aparece separado y no reclasifica ni revierte el local.

## Enlaces

Solo URL HTTP(S) validada se renderiza como link con protección; cualquier otro valor queda texto. Copy elimina promesas ADO-first/validación previa no ejecutada.
