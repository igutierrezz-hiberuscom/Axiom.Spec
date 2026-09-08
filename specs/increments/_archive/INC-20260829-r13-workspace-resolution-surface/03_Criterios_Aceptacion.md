# 03 Criterios de Aceptación

## Happy path

- **CA-D1 / ACC-063**: desde axiomRepo y cada code repo, con `--no-register`, se obtienen el mismo grafo y se ejecutan reparaciones/setup incremental.
- **CA-D2 / ACC-063**: registro ausente/stale/divergente no impide resolver; divergencia queda visible y no se sobrescribe fuera de una mutación solicitada.
- **CA-D3 / ACC-064**: `projects add|join` cataloga; `repo add` agrega code repo nuevo/existente y `role add` es shorthand equivalente.
- **CA-D4 / ACC-069**: workspace state válido es compartido por setup/adapters/providers/MCP/launcher; updates concurrentes conservan ambos cambios y no-op conserva bytes.

## Negativos

Puntero/identidad mismatch, topology inválida, repo foráneo, segundo axiomRepo, kind control/spec/legacy, JSON truncado/futuro/parcial/unknown fields y lock timeout fallan con exit no cero y cero overwrite.

## Superficie retirada

`axiom repo --help` no contiene attach; evento, docs, imports y pruebas dedicadas desaparecen. `discover` permanece independiente.

## Evidencia

Suites workspace setup/adopt/incremental, repo/role/projects, project-resolution, user-workspace, launcher/MCP/member install; tests multiproceso de workspace state; build y diff-check.
