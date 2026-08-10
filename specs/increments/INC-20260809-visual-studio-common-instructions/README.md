# Proyectar Visual Studio a la instruccion comun

> **Codigo**: INC-20260809-visual-studio-common-instructions
> **Estado**: pending
> **Fecha de creacion**: 2026-08-09
> **Tipo de cambio**: modificar

## Objetivo

Hacer que Visual Studio 2026 use la misma fuente `.github/copilot-instructions.md`
que GitHub Copilot, Copilot CLI y VS Code, retirando `.vs/AXIOM.md` como
superficie activa.

## Contexto y motivacion

La documentacion oficial de Visual Studio confirma `.github/copilot-instructions.md`
para Copilot Chat. No confirma `.vs/AXIOM.md`; ese path es una convencion
interna de Axiom. El adapter actual tambien trata `.vs/mcp.json` como una
suposicion documentada, no como un schema verificado en Visual Studio real.

## Dependencias

- `INC-20260809-unify-copilot-target` define el target y writer canonicos.
- `ACC-024` mantiene la semantica de preservacion y migracion del writer.

## Alcance tipado

- `Axiom/packages/adapters/visual-studio-2026/` y su registro de package.
- `Axiom/apps/cli/src/commands/{workspace-adapters,native-mcp-config}.ts`.
- `Axiom/packages/cli-commands/src/commands/{configure,sync}.ts` si contienen
	dispatch especifico.
- `Axiom/packages/installer/src/registry.ts`, perfiles, docs y tests de VS.

## Decisiones cerradas

- `visual-studio-2026` puede permanecer como alias de compatibilidad para
	migrar estados existentes, pero no conserva una fuente separada.
- No se escribe ni se recomienda `.vs/AXIOM.md` como output activo.
- `.vs/mcp.json` no se trata como contrato verificado; solo se conserva si una
	prueba real o documentacion oficial establece path y schema.
- Un archivo legacy existente se migra de forma no destructiva y se conserva
	si contiene contenido que no puede atribuirse con seguridad.

## No incluido

- No inventar una integracion especifica de Visual Studio para MCP.
- No cambiar el contenido comun del prompt mas alla de la proyeccion de target.
- No retirar Visual Studio como destino completo.

## Criterios de aceptacion

1. El target de Visual Studio genera o reutiliza `.github/copilot-instructions.md`.
2. `.vs/AXIOM.md` no aparece en outputs nuevos ni en el registry activo.
3. La migracion de un `.vs/AXIOM.md` existente es conservadora y no pierde
	 contenido humano.
4. Cualquier soporte MCP de `.vs` queda explicitamente verificado o fuera del
	 output, sin asumir schema.
5. Tests focalizados y `npm run build` pasan.

## Validacion prevista

- Tests del adapter VS, document-bootstrap, native MCP, installer y sync.
- Barrido de outputs activos y de referencias no historicas a `.vs/AXIOM.md`.
- `npm run build` desde `Axiom/`.

## Integracion de spec y contexto

Pendiente para la consolidacion del orquestador. Este worker no edita
`Axiom.Spec/specs/00..08` ni `Axiom.Spec/context/**` por restriccion de alcance.

## Implementation notes

- `visual-studio-2026` conserva su package e id como alias de compatibilidad,
	pero delega en `@axiom/adapters-github-copilot` y en el writer comun.
- El registry declara `.github/copilot-instructions.md` para ambos targets;
	`.vs/AXIOM.md` deja de ser output activo.
- El writer comun reconoce `.vs/AXIOM.md` como fuente legacy: escribe primero el
	destino comun, preserva contenido humano y elimina la fuente solo cuando la
	migracion es segura. Ante divergencia conserva ambas copias y devuelve warning.
- `configure`, `sync`, workspace adapters y process surfaces proyectan VS al
	target `github-copilot` y deduplican la salida cuando ambos se seleccionan.
- Visual Studio queda fuera de `NATIVE_MCP_TARGETS`. Su dispatcher conserva la
	interfaz para una futura verificacion, pero no escribe `.vs/mcp.json` y
	devuelve un warning explicito de schema/path no verificados.
- No se modifico el dispatch general de sync de Antigravity/Codex ni se
	implemento filtrado MCP por proyecto.

## Validation

- `npm run build` desde `Axiom`: OK.
- `npm exec vitest run packages/adapters/visual-studio-2026/tests/generator.test.ts packages/adapters/github-copilot/tests/generator.test.ts packages/document-bootstrap/tests/writer.test.ts apps/cli/tests/native-mcp-config.test.ts packages/installer/tests/installer.test.ts apps/cli/tests/workspace-adapters.test.ts apps/cli/tests/workspace-process-surfaces.test.ts apps/cli/tests/document-bootstrap.test.ts apps/cli/tests/sync.test.ts apps/cli/tests/e2e/adapters.e2e.test.ts apps/cli/tests/workspace-mcp.test.ts --config=vitest.config.ts`: 11 archivos, 132 tests OK.
- Validacion adicional de registry/perfiles/model-routing: 3 archivos, 42 tests OK.
- `get_errors` sobre los archivos de implementacion afectados: sin errores.

## Result

Implementacion completada para ACC-027 en `Axiom`, con proyeccion comun de
Visual Studio, migracion legacy conservadora, registry consistente y MCP no
verificado sin escritura. No se detectaron fallos introducidos por este
incremento en la validacion dirigida. El estado queda `pending` hasta que el
orquestador integre el conocimiento estable en los archivos canonicos
correspondientes.

## Receipts

- `build`: `tsc -b` finalizado con exit code 0.
- `focused-tests`: 132 tests finalizados con exit code 0.
- `registry-profile-model-routing`: 42 tests finalizados con exit code 0.
- `get_errors`: sin errores en los paths afectados.

## Acceptance review

1. Cumplido: VS genera/reutiliza `.github/copilot-instructions.md` mediante el
	 writer comun.
2. Cumplido: registry y outputs nuevos no declaran ni escriben `.vs/AXIOM.md`.
3. Cumplido: la migracion legacy preserva contenido y conserva la fuente ante
	 conflicto.
4. Cumplido: `.vs/mcp.json` no se escribe y el warning explicita que el schema
	 y path no estan verificados.
5. Cumplido: alias legacy y seleccion conjunta VS/Copilot no duplican el
	 archivo canonico.
6. Cumplido: tests dirigidos, build y `get_errors` pasan.

## Failures classified

- Fallos introducidos: ninguno en la validacion final.
- Fallos preexistentes: ninguno observado en el alcance validado.

## General spec integration

- Specs `00`, `01`, `03`, `05`, `06` y `08`: Visual Studio usa
	`.github/copilot-instructions.md`; `.vs/AXIOM.md` y `.vs/mcp.json` no son
	outputs activos.
- `context/architecture/04`, `context/integrations/01` y
	`context/TECHNICAL_CONTEXT.md`: alias y límite MCP no verificado actualizados.
