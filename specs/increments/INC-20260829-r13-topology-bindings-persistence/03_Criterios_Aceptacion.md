# 03 Criterios de Aceptación

## Happy path

- **CA-C1 / ACC-060**: set/show/remove funcionan para axiomRepo, cada codeRepo y legacy source con paths canónicos desde cwd distintos.
- **CA-C2 / ACC-061**: show/validate devuelven un documento JSON v1 real y la causa exacta en éxito/error.
- **CA-C3 / ACC-062**: roles, workspace y bindings convergen en el mismo writer; escrituras concurrentes preservan todos los cambios e idempotencia byte a byte.

## Negativos

ID desconocido, vacío, path relativo no normalizable, fichero, missing, URI remota, YAML corrupto, schema futuro, topology inválida, profiles inválido, lock timeout y fallo write/rename producen error tipado/exit no cero sin overwrite.

## Comentarios y concurrencia

Un comentario humano fuera del bloque gestionado sobrevive byte a byte. Un documento ambiguo no se reescribe. Dos procesos y un temporal stale están cubiertos; no quedan artefactos propios tras éxito/fallo.

## Evidencia

Suites topology/bindings/roles/member-install/workspace, wrappers Commander y binario compilado, build/typecheck y diff-check.
