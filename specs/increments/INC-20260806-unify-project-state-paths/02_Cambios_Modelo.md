# 02 Cambios de Modelo

## Objetivo del documento

Registrar la topología de paths y el contrato de compatibilidad de los
stores, lectores y writers afectados.

## Entidades o estructuras afectadas

- `projectStateDir` y `projectStateFilePath` de `@axiom/filesystem-truth`.
- `resolveProjectStateFile` y `migrateProjectStateDirectory`, con candidatos
	canonical, legacy-project, legacy-config, legacy-scope y legacy-local.
- `FilesystemStore`/`resolveScopeDir`: `config` apunta a la raíz canónica y
	otros scopes son subdirectorios project-scoped.
- Checkpoints, toolchain markers, workflow, memoria, MCP bindings, installer,
	components, model-routing, workspace/provider registry y doctor reciben la
	identidad canónica y aliases legacy cuando leen.

## Contratos o estados afectados

- `config` sigue siendo un label de API, no una carpeta obligatoria.
- La restauración de checkpoints remapea destinos legacy al projectKey y
	elimina la copia runtime anterior solo después de escribir el destino.
- Toolchain detect/probe reconoce aliases y repair migra markers sin duplicar
	canonical y legacy.
- Worktree provisioning selecciona providers por `Execution.projectId`, no por
	un scan global de `.axiom-state`.

## Notas de compatibilidad

La lectura legacy es lazy y best-effort. La precedencia es canonical, path
directo antiguo, path `config`, scope antiguo y solo después paths locales
conocidos. Si hay conflicto se conserva el canonical o se elige el primer
legacy determinista y se informa mediante `StatePathWarning`.
