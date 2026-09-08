# 01 Requisitos

- **E-065.1**: un preflight compartido valida todos los inputs/paths/owners/outputs antes de cualquier write.
- **E-065.2**: reserved IDs, duplicados, aliases físicos, flags incoherentes y solapes target/source fallan con error tipado y snapshot filesystem idéntico. Setup/repo/role rechazan identidad foránea; adopt puede preservar una identidad válida de otro proyecto como `skipped`/no-clobber, pero rechaza YAML inválido, ambiguo o conflictivo con el proyecto activo.
- **E-065.3**: solo `ENOENT` prueba ausencia. `ENOTDIR`, `EACCES`, `EIO` y cualquier observación desconocida fallan con envelope que conserva path, causa, operación, clasificación y código.
- **E-066.1**: desired state de identity/topology/bindings/registry/workspace state valida completo antes de apply.
- **E-066.2**: commit estructural es recuperable y concurrente; fallo de cualquier recurso requerido no termina en éxito.
- **E-066.3**: resultados distinguen created/updated/unchanged/skipped, warnings derivados y error; CLI/launcher comparten envelope/exit.
- **E-066.4**: `--no-register` conserva el guard read-only y puede coordinarse con un lock temporal de registry, pero no persiste ni deja home/registry/lock creados por el protocolo.
- **E-066.5**: toda adquisición parcial de locks de apply o recovery libera lo adquirido ante retorno o excepción; si cleanup no puede probar seguridad, preserva artifacts y devuelve diagnóstico tipado/recovery-required.
- **E-067.1**: reconciliación de axiom.yaml toca solo bloque Axiom y preserva bytes humanos.
- **E-067.2**: documento inválido/ambiguo/conflictivo falla en preflight; se retira mapa recíproco duplicado.

## Regla absoluta

Un rechazo de preflight produce cero mutaciones, incluidos mkdir, lock, tmp, journal, logs, registry y outputs derivados. Los locks temporales pertenecen únicamente a apply, después de un preflight aceptado, y deben limpiarse de forma verificable.

## Fuera de alcance

Compensaciones destructivas sobre legacy/human content, reparación granular ACC-068 y R-13.4/ACC-070..076.
