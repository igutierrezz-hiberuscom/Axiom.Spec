# 03 Criterios de Aceptación

## Criterios de aceptación

1. No existen funciones, exports ni documentación activa que afirmen o configuren auto-persistencia por `MemoryKind`.
2. `memory add` documenta y acepta `--visibility`, `--rationale` y `--source`; guarda los valores suministrados en la entry.
3. `memory add --visibility invalid` falla sin escribir memoria.
4. Las entries `private` y `project-shared` pueden leerse localmente; Knowledge Sync conserva su filtro de exportar solo `project-shared` explícito.
5. Las skills y agentes Kiro `axiom-increment` y `axiom-bug` contienen una instrucción obligatoria, una plantilla de comando y la guía de clasificación project-shared/private.
6. La guía prohíbe secretos en cualquier memoria y deja visible un error de guardado sin fallback automático.
7. Pasan el build y las pruebas dirigidas de memoria/Knowledge Sync afectadas.

### Happy path

Un agente confirma una decisión de arquitectura reutilizable y ejecuta `axiom memory add --kind decision --visibility project-shared --rationale ... --source ...`; la entry queda disponible localmente y elegible para Knowledge Sync. Para una hipótesis local, usa `private`; la entry sigue legible localmente y no se exporta.

### Validaciones y errores

Un valor de visibilidad inválido se rechaza. Si el backend devuelve error, la skill informa el fallo y no declara la memoria como capturada.

### Permisos y visibilidad

`private` es una política de exportación Git, no un control de acceso. La memoria no es un lugar para secretos o datos sensibles.

### Estados y efectos observables

El guardado es siempre una acción explícita del agente; no hay nueva escritura causada por lifecycle, audit, launcher o telemetría.
