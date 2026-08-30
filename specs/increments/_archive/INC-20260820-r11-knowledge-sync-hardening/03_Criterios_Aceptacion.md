# 03 Criterios de Aceptación

## Criterios de aceptación

### Happy path

1. `knowledge sync` previsualiza por defecto; con `--confirm` genera chunk/manifest y commit; solo `--confirm --push` intenta push.
2. `knowledge pull` sin incremento lista/importa todos los pendientes; con confirmación guarda todas las entradas correctas y marca el chunk una vez.
3. Los chunks exportados preservan `rationale`, `source`, metadata estable y solo entradas `project-shared` sin secretos.

### Validaciones y errores

4. Private, visibilidad ausente y secretos se omiten con contadores/motivos observables.
5. Manifest/chunk/memory malformados, archivos ausentes y fallos de save son diagnósticos visibles y no consumen el marker.
6. Dos ejecuciones confirmadas son idempotentes; la migración legacy es atómica y no deja el marker en `.engram` versionable.

### Permisos y visibilidad

7. Ninguna ruta sin confirmación muta disco, memoria o Git; ningún push ocurre sin `--push` y confirmación.
8. No se comparte estado local o datos de otro proyecto/miembro.

### Estados y efectos observables

9. Tests focalizados cubren preview, confirm/push, privacidad, secretos, evidencia, schema, fallos, reintentos y aislamiento de dos miembros; build pasa.
10. La documentación operativa/canónica explica el contrato resultante sin conservar comportamiento inseguro como vigente.
