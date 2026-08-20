# 02 Cambios de Modelo

## Objetivo del documento

Retirar del grafo runtime el paquete Cavekit que no tenía consumidores productivos.

## Entidades o estructuras afectadas

- Package `@axiom/cavekit-discipline`, sus fuentes, pruebas y project reference.
- Dependencias y catálogos activos del runtime que lo enumeraban.

## Contratos o estados afectados

Cavekit deja de ser capacidad runtime; no se añade reemplazo ni se modifica un comando. `zod` permanece disponible para sus consumidores ajenos a Cavekit.

## Notas de compatibilidad

El antecedente histórico `0015-cavekit-discipline-post-mvp.md` se preserva sin editar. La decisión Core `DEC-20260818-134600-3jfjak`, enlazada a este incremento, registra que la retirada supera ese antecedente.
