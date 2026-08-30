# 02 Cambios de Modelo

## Estructuras afectadas

- `ProjectsFile` conserva `schemaVersion: 2` y un mapa de proyectos validado íntegramente.
- `ProjectEntryV2` mantiene identidad, nombre, timestamps y repos; los estados calculados no se persisten.
- `RepoAvailability = present-directory | missing | not-directory`.
- `ProjectAvailability = available | partial | unavailable` y `resolvable` separado cuando se solicite.
- `UserWorkspaceError` separa schema inválido/no soportado, ID/path collision, duplicate-id, not-found, lock-timeout y errores I/O sanitizados.
- `CliJsonEnvelopeV1<T>` fija versión, comando, éxito y payload/error mutuamente excluyentes.

## Persistencia

`projects.yml` es el único archivo. El read-modify-write usa un lock adyacente acotado y `atomicWriteFile` con nombre temporal único en el mismo directorio. El contenido se valida antes del rename. Un error conserva el archivo previo byte a byte y limpia solo temporales propios.

## Compatibilidad

No hay migración ni fallback v1. Se retiran exports y pruebas de compatibilidad. El schema 2 de `projects.yml` no se renombra ni degrada.

## Ownership del primitive

El primitive genérico se ubica en `@axiom/core` para evitar dependencias cíclicas. C debe reutilizarlo para topology/bindings; E puede componerlo, pero no redefinirlo.
