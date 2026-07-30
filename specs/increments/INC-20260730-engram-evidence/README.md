# INC-20260730-engram-evidence: Evidencia de Engram

## Goal
Forzar que el guardado en la memoria persistente (`mem_save`) requiera siempre un `rationale` (por qué se toma la decisión) y un `source` (de dónde proviene la información o directiva).

## Scope
- Actualizar el esquema de entrada o el handler de `mem_save` en `@axiom/memory` (o donde se declare el schema) para que `rationale` y `source` sean requeridos.
- Si un agente o subagente intenta inventar contexto o llama a `mem_save` sin esta evidencia, la llamada será rechazada (fail-closed).
- Modificar llamadas internas si existen, para asegurar que cumplen la nueva firma.

## Non-goals
- No se verificará en tiempo de ejecución si el rationale es "suficientemente bueno" semánticamente; la mera presencia y tipado estricto (string length > x) es el requisito para el MVP.

## Acceptance Criteria
- `mem_save` rechaza inputs sin `rationale` y `source`.
- Subagentes están obligados a justificar su memoria en los logs.
