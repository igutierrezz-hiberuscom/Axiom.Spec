# 03 Criterios de Aceptación

## Criterios de aceptación

### Happy path

1. Un documento con frontmatter `tags: [memory, engram]` aparece en el índice con esas tags normalizadas; uno sin metadata usa su carpeta como fallback y el de raíz usa `repo`.
2. Con tags de tarea, el selector devuelve exactamente `mandatory.always`, `mandatory.whenTags` coincidente y `available` coincidente, en orden estable y sin duplicados.
3. `spec.recommendedContextList` y `spec.implementationContextRead` devuelven la misma selección para el mismo índice y `taskTags`; la lectura conserva sus niveles de presupuesto.
4. La indexación real de `Axiom.Spec/context/**` funciona con un vocabulario pequeño y reutilizable de tags propietarias.

### Validaciones y errores

5. Tags vacías, no-string, duplicadas o con forma inválida se normalizan/rechazan de forma determinista sin alterar rutas ni índice manualmente.
6. Sin `taskTags`, ninguna entrada disponible se recomienda; solo aparecen los obligatorios aplicables.
7. Una ruta repetida entre categorías aparece una sola vez; una ruta que salga del spec repo se rechaza.
8. El índice se actualiza mediante el comando/generador canónico y las escrituras derivadas son atómicas.

### Permisos y visibilidad

9. Las señales adicionales proceden únicamente de campos estructurados permitidos de plan/rol; no hay IA, scoring ni lectura libre del texto.
10. La salida diferencia obligatorio de recomendado y no expone contenido fuera del presupuesto configurado.

### Estados y efectos observables

11. Tests focalizados cubren generación, normalización, fallback `repo`, matching, ausencia de tags, deduplicación, presupuesto, aislamiento, handlers y dos miembros; build pasa.
12. La documentación operativa/canónica describe el contrato determinista resultante sin mantener como vigente el comportamiento de recomendar todo `available`.
