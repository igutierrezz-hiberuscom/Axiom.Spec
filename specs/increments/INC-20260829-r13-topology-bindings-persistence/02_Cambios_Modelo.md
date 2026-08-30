# 02 Cambios de Modelo

## Bindings

`LocalBindingsV2 { schemaVersion: 2; localPaths: Record<RepoId, AbsoluteCanonicalPath> }` se valida de forma cerrada. El estado físico es calculado y no persistido.

## Errores

`TopologyError` conserva categorías de I/O, YAML, manifest, schema, profile/config, binding schema, unknown repo ID, invalid path, lock timeout y validation finding. El envelope JSON proyecta la categoría sin cambiarla.

## Writer

`readValidated -> mutate pure -> validate -> atomic replace` ocurre dentro del lock compartido. El temporal usa mismo directorio, PID/nonce y cleanup propio. Lock stale solo se recupera con metadata verificable y nunca se roba a un owner vivo.

## Ownership de YAML

Topology contiene bloque generado marcado y extensiones humanas fuera preservadas byte a byte. Si el parse/markers no permiten separar ownership, la mutación falla. Bindings declara archivo completamente generado; contenido no reconocido bloquea la reescritura.

## Compatibilidad

Schema 1 se rechaza sin migración.
