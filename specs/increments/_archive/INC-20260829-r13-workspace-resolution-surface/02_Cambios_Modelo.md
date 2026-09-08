# 02 Cambios de Modelo

## Resolución

`ResolvedWorkspace` contiene projectId, axiomRepo, codeRepos, roles, assignments, legacy refs y origen autoral. Se deriva de topology schema 2; la entrada user-level es metadata opcional separada.

`WorkspaceRepoSpec` usa unión discriminada `axiom|code`; no existen `control|spec`. Code repo exige repoId y roleId.

## Catálogo

Cada repo proyectado conserva `repoId`, `kind`, path y role/ownership opcional; la key de mapa no codifica semántica. La reconciliación compara identidad/path canónicos.

## Workspace state

`WorkspaceStateV1` es shape cerrada, validada y compartida. Ausente crea default solo dentro de una operación autorizada; inválido devuelve `WorkspaceStateError`. El writer usa el primitive común y preserva campos/timestamps.

## Compatibilidad

Se retiran attach e inferencias. No hay fallback/reset para JSON corrupto.
