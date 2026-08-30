# 04 Interacciones UI

## Objetivo del documento

Definir las respuestas observables de las herramientas MCP de contexto.

## Superficie UI afectada

`spec.recommendedContextList`, `spec.implementationContextRead` y el comando CLI `axiom context index`.

## Flujo de interacción

El operador regenera el índice desde el spec repo. Un caller aporta opcionalmente `taskTags`; recibe documentos obligatorios primero y documentos recomendados solo si sus tags coinciden. La lectura compuesta recibe la misma selección y aplica el presupuesto existente a referencias/contenido.

## Estados visibles

La respuesta identifica la categoría de cada documento y distingue referencias obligatorias de recomendadas. Sin tags de tarea, no enumera el catálogo disponible entero.

## Cascadas y comportamiento reactivo

No hay UI gráfica nueva. Cambiar las tags de un documento requiere regenerar el índice canónico; no hay cache manual ni recomendación implícita fuera de la regla declarada.
