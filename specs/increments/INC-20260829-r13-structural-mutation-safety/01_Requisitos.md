# 01 Requisitos

- **E-065.1**: un preflight compartido valida todos los inputs/paths/owners/outputs antes de cualquier write.
- **E-065.2**: reserved IDs, duplicados, aliases físicos, foreign YAML, flags incoherentes y solapes target/source fallan con error tipado y snapshot filesystem idéntico.
- **E-066.1**: desired state de identity/topology/bindings/registry/workspace state valida completo antes de apply.
- **E-066.2**: commit estructural es recuperable y concurrente; fallo de cualquier recurso requerido no termina en éxito.
- **E-066.3**: resultados distinguen created/updated/unchanged/skipped, warnings derivados y error; CLI/launcher comparten envelope/exit.
- **E-067.1**: reconciliación de axiom.yaml toca solo bloque Axiom y preserva bytes humanos.
- **E-067.2**: documento inválido/ambiguo/conflictivo falla en preflight; se retira mapa recíproco duplicado.

## Regla absoluta

Un rechazo de preflight produce cero mutaciones, incluidos mkdir, lock, tmp, journal, logs, registry y outputs derivados.

## Fuera de alcance

Compensaciones destructivas sobre legacy/human content y reparación granular ACC-068.
