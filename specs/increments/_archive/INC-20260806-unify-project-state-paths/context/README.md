# Context

## Propósito

Documentar hechos técnicos verificables de la resolución de paths y de la
migración de estado runtime.

## Qué puede vivir aquí

- La tabla de rutas canónicas y legacy.
- El orden de precedencia y las garantías de migración.
- Las fronteras entre project-scoped, local y execution-scoped.
- Los consumers que reciben `projectKey` y aliases explícitos.
- Las garantías específicas de restore de checkpoints, repair de toolchain y
	selección de providers en worktrees.

## Qué no debe vivir aquí

- Propuestas de base de datos, índices globales o lifecycle enterprise.
- Estado de una ejecución concreta o secretos del operador.
- Claims no verificadas que contradigan el código fuente.

## Estructura sugerida

Fuente primaria: `Axiom/packages/filesystem-truth/src/state-paths.ts`.
Fuentes de integración: `persistence`, `workflow`, `memory`, `versioning`,
`toolchain`, `providers`, `doctor` y `apps/cli` worktree provisioning.
