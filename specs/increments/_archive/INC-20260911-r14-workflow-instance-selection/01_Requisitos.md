# 01 Requisitos

## Objetivo del documento

Definir los requisitos técnicos y funcionales para permitir la convivencia de múltiples instancias de artefactos en vuelo (`increment`, `bug`, `plan`) en `workflow-state.json` con selección explícita por ID y comando `select`, eliminando el bloqueo actual por singleton.

## Requisitos del incremento

- **REQ-087-01: Estado por instancia en workflow-state.json (schemaVersion: 2)**
  El almacén de estado en `.axiom-state/<projectKey>/workflow-state.json` debe indexar las instancias por `metadataId` dentro de cada workflow aplicable (`increment`, `bug`, `plan`), guardando un puntero opcional `activeInstanceId` por workflow.

- **REQ-087-02: Migración determinista e idempotente de schemaVersion 1 a 2**
  Si existe un `workflow-state.json` con `schemaVersion: 1`, debe migrarse automáticamente y sin pérdida de datos a `schemaVersion: 2`. Los records vigentes se reubican bajo `instances[metadataId]` y su ID se asigna a `activeInstanceId`.

- **REQ-087-03: Selección explícita por flag `--id <id>`**
  Cualquier subcomando de workflow (`refine`, `specify`, `plan`, `plan-approve`, `verify`, `archive`, etc.) que reciba `--id <id>` debe operar sobre la instancia identificada sin rechazo por `ID mismatch`, estableciéndola como la instancia activa del workflow.

- **REQ-087-04: Comando explícito de selección (`select`)**
  Añadir el subcomando `select --id <id>` a `axiom-increment`, `axiom-bug` y `axiom-plan` para seleccionar la instancia activa sin mutar su estado.

- **REQ-087-05: Error accionable ante ambigüedad o ausencia de selección**
  Si se invoca un subcomando sin `--id` y no hay `activeInstanceId` seleccionada, o si el ID solicitado no existe, el CLI debe listar las instancias disponibles en vuelo con su estado y sugerir el comando exacto para seleccionar.

- **REQ-087-06: Purga limpia al archivar**
  Al completar la transición de archivo (`archive`), la instancia terminada se purga del bloque `instances` activo en `workflow-state.json`. El registro permanente e inmutable reside en `metadata.yml`.

- **REQ-087-07: Inspección multi-instancia en `axiom state`**
  `axiom state --workflow <id>` debe mostrar todas las instancias en vuelo, señalando cuál es la activa (`*`), su estado actual y transiciones recomendadas.

## Reglas de negocio relevantes

- Los workflows sin artefactos folder-per-instance (`role`, `qa-e2e`) conservan su semántica actual sin alteraciones.
- Ninguna transición ilegal se vuelve legal por el cambio de esquema.
- El aislamiento entre instancias es estricto: ninguna instancia hereda `vars` ni contexto de otra.
- Comportamiento estrictamente fail-closed.

## Fuera de alcance funcional

- Ejecución concurrente con bloqueo atómico distribuido o paralelo (concurrencia de escritura).
- Alterar el formato de los receipts o el cálculo de hashes SHA-256.
