# 03 Criterios de Aceptación

## Criterios de aceptación

### AC-087-01: Coexistencia de múltiples instancias en vuelo
Dado un repositorio con varios incrementos/bugs/planes creados, `workflow-state.json` mantiene sus estados individuales bajo `instances` sin sobreescrituras destructivas entre ellos.

### AC-087-02: Selección explícita por subcomando
Al invocar `axiom <axiom-increment|axiom-bug|axiom-plan> <subcomando> --id <id>`, el comando opera exitosamente sobre el artefacto indicado, sin lanzar error de `ID mismatch`, y lo establece como activo.

### AC-087-03: Comando `select` explícito
Al ejecutar `axiom <axiom-increment|axiom-bug|axiom-plan> select --id <id>`, la instancia activa queda fijada al ID indicado y devuelve confirmación exitosa con su estado actual.

### AC-087-04: Migración transparente desde schemaVersion 1
Un archivo `workflow-state.json` preexistente con `schemaVersion: 1` es leído y transformado a `schemaVersion: 2` preservando su estado, variables y timestamp sin errores.

### AC-087-05: Aislamiento estricto de variables
El avance de estado o mutación de `vars` en una instancia A no altera las `vars` ni el estado de la instancia B.

### AC-087-06: Purga limpia al archivar
Al ejecutar `archive --confirm` para un artefacto, la instancia terminada se elimina de las instancias activas de `workflow-state.json`.

### AC-087-07: Error accionable ante selección inválida o ausente
Si se invoca una transición sin `--id` y sin instancia activa configurada, o con un `--id` inexistente, el CLI lista las instancias disponibles y devuelve exit code 1 con mensaje de ayuda accionable.

### AC-087-08: Inspección enriquecida en `axiom state`
`axiom state --workflow <id>` muestra la lista de todas las instancias en vuelo, indicando cuál es la activa (`*`), su estado actual y transiciones recomendadas.
