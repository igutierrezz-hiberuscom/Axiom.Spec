# 02 Cambios de Modelo

## Objetivo del documento

Precisar metadata, datos derivados y entradas MCP para el selector por etiquetas.

## Entidades o estructuras afectadas

- Documento Markdown de `context/**`: frontmatter YAML opcional con `tags: string[]`.
- Entrada del índice técnico: `tags: string[]` normalizadas y ordenadas, derivadas de frontmatter o fallback.
- Parámetro opcional `taskTags` de lista recomendada y lectura de implementación.
- Selector compartido que recibe índice, tags y señales estructuradas ya normalizadas.

## Contratos o estados afectados

Las rutas de contexto siguen siendo relativas al spec repo y el índice sigue siendo derivado/regenerable. `available[]` deja de significar “recomendado siempre”: solo se selecciona con una coincidencia explícita de tags. La lectura compuesta expone claramente qué documentos son obligatorios y cuáles recomendados.

## Notas de compatibilidad

Los documentos sin frontmatter continúan siendo indexables mediante fallback de carpeta. Las entradas de índice previas sin tags se regeneran; los handlers sin `taskTags` conservan solo el contexto obligatorio, que es el comportamiento seguro de menor exposición.
