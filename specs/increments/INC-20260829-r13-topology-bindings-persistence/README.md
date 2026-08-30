# Bindings, errores y persistencia topológica R-13

> **Código**: INC-20260829-r13-topology-bindings-persistence
> **Estado documental**: especificación refinada; lifecycle gestionado por Axiom Core
> **Fecha**: 2026-08-29
> **Acciones**: ACC-060..ACC-062
> **Dependencia**: B completado y validado

## Objetivo

Alinear bindings con topology schema 2, preservar errores de carga/validación hasta CLI/API y centralizar las escrituras topológicas en el mismo primitive seguro introducido por A.

## Revalidación

`LocalBindings` sigue en schema 1, acepta cualquier key/path y cae a mapa vacío en consumidores. `topology show|validate` recodifica causas o sintetiza fallback. `saveLocalBindings` y `roles.ts` usan temporales fijos y writers distintos sin lock/validator.

## Alcance

- ACC-060: bindings schema 2 para axiom/code/legacy IDs conocidos, paths absolutos canonicalizados, estados precisos y corrupción visible.
- ACC-061: `TopologyError` real y envelope JSON v1 estable en `show|validate`, launcher y MCP.
- ACC-062: writer común validado, atómico, lockeado, idempotente y resistente a temporales stale/fallo.
- Roles, workspace y member install reutilizan writer; ningún writer serializa fields legacy/derivados.

## No objetivos

Resolución workspace/catalog optional (D), coordinación multiarchivo (E), cambio de autoridad/modelo ya propiedad de B, seguridad launcher y R-13.5.

## Decisiones cerradas

1. `topology-bindings.yaml` usa `schemaVersion: 2` y `localPaths: Record<repoId, absoluteCanonicalPath>`.
2. Solo IDs del manifest autoral son bindables, incluidos axiom, code y legacy. URI remota no materializada no es un path local y no se persiste como binding.
3. Estado de binding resuelto: `present-directory | missing | not-directory | remote-unmaterialized`; el valor persistido siempre es path absoluto local.
4. Overrides relativos y `--project-path` se resuelven contra el cwd efectivo antes de localizar autoridad; luego se canonicalizan.
5. Loader corrupto/futuro devuelve `TopologyError`, nunca `{}`. Los consumidores sensibles no caen a refs sin informar.
6. Envelopes reutilizan el contrato A: `{schemaVersion:1, ok, command, data?|error?}`. Error incluye `kind`, código estable y mensaje sanitizado; stdout contiene un único JSON con `--json`.
7. Se reutilizan `withFileLock`/`atomicWriteFile` de `@axiom/core`; no se crea lock propio. El lock se toma sobre el recurso autoral y el read-modify-write ocurre dentro.
8. Política de comentarios: topology autoral se gestiona dentro de markers `AXIOM:TOPOLOGY:GENERATED`; bytes humanos fuera del bloque se preservan. Contenido YAML no reconciliable o documento sin ownership claro falla antes de escribir. Bindings es generado/user-local, lleva marker explícito y rechaza mutación de contenido humano no reconocido en vez de borrarlo.
9. Toda topología se valida semánticamente justo antes del rename; bindings se validan contra la misma topología bajo lock.

## Riesgos

Canonicalización Windows/symlink/8.3; remote refs; lock compartido entre roles/workspace; preservación de comments; cambios locales R-11/R-12 en member-install/MCP que deben mantenerse.

## Compatibilidad

No se acepta bindings/topology legacy. No hay migrador. Un binding schema 1 o ID desconocido falla con guía de recreación explícita.

## Validación prevista

Tests de loader/CLI/binario, paths Windows/espacios/symlink/remote URI, corrupción y schema futuro, writers concurrentes/fallos/temporales/comments, build y diff-check.

## Integración estable

Diferida a la pasada final; el worker no edita `specs/00..08` ni `context/**`.
