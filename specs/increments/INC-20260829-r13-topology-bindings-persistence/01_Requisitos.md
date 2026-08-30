# 01 Requisitos

- **C-060.1**: bindings schema 2 enumeran únicamente IDs del manifest autoral y guardan paths absolutos canónicos.
- **C-060.2**: set/show/remove y consumidores distinguen path ausente, no-directorio y URI no materializada; corrupción nunca se oculta.
- **C-060.3**: `--project-path` y overrides relativos producen el mismo resultado desde cualquier cwd.
- **C-061.1**: show/validate preservan `invalid-yaml`, `invalid-manifest`, unsupported schema, profiles/config invalid e I/O.
- **C-061.2**: CLI Commander/binario, launcher y MCP comparten envelope/error; no hay manifest sintético.
- **C-062.1**: topology y bindings usan writer/lock común, validan antes de persistir y son idempotentes.
- **C-062.2**: concurrencia no pierde cambios, cada temporal es único y los fallos conservan bytes previos.
- **C-062.3**: roles/workspace/member-install no serializan mirrors ni borran contenido humano.

## Reglas

Los refs del manifest son defaults declarativos; un binding inválido no se ignora. Ninguna operación degrada error estructural a warning de éxito.

## Fuera de alcance

Transacción de varios archivos, catálogo opcional, setup preflight completo y frontend launcher.
