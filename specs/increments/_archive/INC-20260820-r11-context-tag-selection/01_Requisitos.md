# 01 Requisitos

## Objetivo del documento

Definir el contrato determinista de selección de contexto técnico recomendado.

## Requisitos del incremento

1. Cada documento indexable puede declarar YAML frontmatter opcional `tags`, como lista de strings; las tags se normalizan, deduplican y validan antes de entrar al índice.
2. Un documento sin tags explícitos recibe exactamente una tag fallback derivada de su carpeta principal bajo `context/`; un documento en la raíz usa `repo`.
3. `axiom context index` deriva tags en el índice existente; no se edita ningún índice/cache manualmente.
4. Con `taskTags`, el selector único devuelve por orden estable y sin rutas duplicadas: `mandatory.always`, coincidencias de `mandatory.whenTags` y documentos `available` con al menos una tag en común.
5. Sin `taskTags`, el selector devuelve solo documentos obligatorios; no devuelve `available` por defecto.
6. `spec.recommendedContextList` y `spec.implementationContextRead` delegan en el mismo selector; el segundo acepta `taskTags?` y combina únicamente señales estructuradas declaradas de plan/rol.
7. La salida mantiene categorías obligatorio/recomendado, límites de presupuesto y todas las rutas bajo el spec repo.
8. La generación, el selector y ambos handlers reportan entradas inválidas sin ampliar rutas ni producir recomendaciones no deterministas.

## Reglas de negocio relevantes

Las tags son una clasificación declarativa del documento, no un mecanismo de autoridad. El contexto obligatorio siempre prevalece; una ruta presente en más de una categoría aparece una sola vez en su primera posición contractual.

## Fuera de alcance funcional

No se cambia el contenido semántico de los documentos ni se implementa ranking, aprendizaje, embeddings o inferencia automática.
