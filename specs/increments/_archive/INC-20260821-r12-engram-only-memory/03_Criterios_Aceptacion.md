# 03 Criterios de Aceptación

## Criterios de aceptación

1. No existen fuentes, exports ni rutas runtime de `createInMemoryBackend`, `memoryFilePath`, `forceJson` o selección `kind: 'json'`.
2. Las operaciones de memoria con un stub Engram correcto conservan save/load/query/session summary y el pin de proyecto.
3. Un Engram no disponible devuelve un `MemoryError` explícito y la CLI de memoria falla sin crear ni leer un JSON local.
4. `axiom doctor` incluye un check de memoria para Engram y devuelve `fail` con una remediación clara si el ejecutable no está disponible.
5. Las pruebas no usan el backend JSON retirado; usan stubs de `MemoryBackend` o proceso Engram hermético.
6. Pasan build, pruebas dirigidas de memoria/CLI/Doctor y el smoke de Doctor en el entorno con Engram instalado.

### Happy path

Con Engram disponible, `axiom memory add`, `show` y `query` interactúan exclusivamente con el proceso `engram mcp` project-pinned y preservan la metadata de la entry.

### Validaciones y errores

Sin Engram o ante un fallo MCP, el resultado contiene un error de memoria accionable. No existe lista vacía, guardado alternativo ni fallback silencioso.

### Permisos y visibilidad

Se preserva el aislamiento por proyecto y la semántica de `project-shared`/`private`; el cambio no agrega permisos ni acceso remoto.

### Estados y efectos observables

Doctor señala la indisponibilidad. Los JSON locales existentes no cambian y el runtime no los consume.
