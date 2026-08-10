# Completar sync de adapters activos

> **Codigo**: INC-20260809-sync-all-adapters
> **Estado**: closed
> **Fecha de creacion**: 2026-08-09
> **Tipo de cambio**: modificar

## Objetivo

Hacer que `axiom sync` reconcilie todos los targets activos con sus generators
reales y reporte el resultado efectivo, sin declarar exito vacio.

## Contexto y motivacion

`workspace setup` ya tiene dispatch para Antigravity, Visual Studio 2026 y
Codex, pero `packages/cli-commands/src/commands/sync.ts` no los invoca. Hoy un
target sin rama puede dejar `generatedFilesCount: 0` y escribir el marker como
si la reconciliacion hubiera terminado.

## Dependencias

- `INC-20260809-remove-litellm-target` define el conjunto sin LiteLLM.
- `INC-20260809-unify-copilot-target` y
	`INC-20260809-visual-studio-common-instructions` definen el target comun.
- `INC-20260809-mcp-project-target-filter` define el filtro MCP que debe usar
	cualquier proyeccion nativa.

## Alcance tipado

- `Axiom/packages/cli-commands/src/commands/sync.ts` y sus helpers.
- Generators de `packages/adapters/{opencode,claude-code,github-copilot,vscode,
	cursor,codex,antigravity,visual-studio-2026}` que sigan activos.
- `Axiom/apps/cli/src/commands/workspace-adapters.ts` solo para compartir
	dispatch/normalizacion sin duplicar ownership.
- Installer registry, marker `last-sync.json`, tests y docs de sync.

## No incluido

- No reintroducir LiteLLM.
- No crear un adapter nuevo ni un fallback que escriba archivos no declarados.
- No ocultar fallos de generator como `generatedFilesCount: 0`.

## Criterios de aceptacion

1. Cada target activo tiene una rama de sync que llama a su generator o una
	 razon explicita de por que no puede materializarse.
2. Antigravity y Codex usan sus generators dedicados; Visual Studio usa el
	 writer comun definido por sus incrementos previos.
3. El marker registra el numero real de archivos y warnings/errores de forma
	 determinista; un target no soportado no se presenta como exito vacio.
4. El gate de sync sigue ejecutandose antes de cualquier mutacion.
5. Tests dirigidos, build y regresiones de configure/workspace pasan.

## Correcciones de revision independiente

El registro de archivos debe representar todos los archivos que un generator
puede escribir. En particular, Cursor debe incluir la regla opcional
`.cursor/rules/axiom-common.mdc` que `generateCursorConfig` ya devuelve en un
cold start; si la regla ya existe y se preserva por no-clobber, el conteo real
puede omitirla en esa corrida.

## Validacion prevista

- Tests de sync, generators de adapters, workspace setup y last-sync marker.
- Smoke tests con cada target activo en un proyecto temporal.
- `npm run build` desde `Axiom/`.

## Integracion de spec y contexto

No se modificaron `Axiom.Spec/specs/00..08` ni `Axiom.Spec/context/**`.
La integracion de conocimiento estable se realiza en la pasada unica del
orquestador; ACC-029 se implemento en su propio incremento y no se duplica
su politica aqui.

## Resultado de implementacion

### Cambios realizados

- `packages/cli-commands/src/commands/sync.ts` ahora normaliza el target y
	dispatchea los generators de opencode, claude-code, github-copilot, vscode,
	cursor, antigravity y codex. `visual-studio-2026` usa la proyeccion comun de
	Copilot.
- `copilot-vscode` se normaliza a `github-copilot` antes del dispatch y no
	genera una segunda superficie.
- LiteLLM sigue rechazado por `RemovedAdapterTargetError`; no existe rama de
	generacion para ese target.
- Un target sin dispatch produce `adapterGenerationFailed` con
	`kind="unsupported-target"` en vez de escribir un marker vacio.
- `last-sync.json` persiste el conteo real, `writtenFiles` y `warnings`.
	Las referencias y dependencias locales de `@axiom/cli-commands` incluyen
	los generators de Antigravity y Codex.
- Se actualizaron los tests de sync para los ocho targets activos, el alias,
	el target retirado, el target no soportado y el contrato ampliado del marker.

### Criterios verificados

- [x] Cada target activo tiene dispatch a su generator o una proyeccion comun
			explicita.
- [x] Antigravity y Codex usan generators dedicados; Visual Studio usa el
			writer comun de Copilot.
- [x] El marker informa resultados efectivos y los fallos no se presentan como
			exito vacio.
- [x] El gate de telemetria permanece antes de la materializacion y de la
			escritura del marker.
- [x] Tests dirigidos y build pasan.

### Validacion y receipts

- `npx vitest run apps/cli/tests/sync.test.ts`: 1 archivo, 18 tests passed.
- Tests de los ocho generators activos, `apps/cli/tests/workspace-adapters.test.ts`,
	`apps/cli/tests/configure.test.ts` y
	`packages/versioning/tests/checkpoints.test.ts`: 11 archivos, 124 tests
	passed.
- `npm run --workspace @axiom/cli-commands typecheck`: passed.
- `npm run build`: passed (`tsc -b`).
- El test de sync verifico explicitamente el rechazo de un target desconocido y
	la ausencia de `last-sync.json` en ese camino; el escenario existente de
	LiteLLM verifico el rechazo explicito y la ausencia del marker.

### Fallos clasificados

- No quedan fallos introducidos por la implementacion.
- La primera ejecucion focalizada encontro cuatro fixtures antiguos que
	usaban Antigravity sin `install-profile.json`, bajo la suposicion anterior
	de que no tenia generator. Se actualizaron esos fixtures al contrato activo
	y la repeticion termino 18/18.
- La salida de Vitest mantiene solo el warning preexistente sobre la API CJS de
	Vite; no afecta el resultado.

### Decisiones para consolidacion

- No se reintrodujo LiteLLM ni se duplico la politica MCP de ACC-029.
- El writer comun de Visual Studio no crea `.vs/AXIOM.md`; la superficie sigue
	siendo `.github/copilot-instructions.md`.
- El incremento queda `closed` porque los criterios, validaciones, revision y
	limites de integracion estan documentados en este README.

### Reparacion de revision ACC-028

- Hash del candidato canonico congelado antes de la reparacion:
	`355bd2d33f9362d30560c80a4aa6b3d63843aac32ab6ffd9bd350480b808cb46`.
- `GENERATED_FILES_BY_TARGET['cursor']` ahora declara la superficie potencial
	completa: `.cursor/settings.json`, `.cursor/AGENTS.md` y
	`.cursor/rules/axiom-common.mdc`.
- `installProfile.generatedFiles` toma esa misma lista potencial. No se cambio
	la semantica runtime: `generateCursorConfig.writtenFiles` puede omitir la
	regla cuando ya existe y el no-clobber la preserva.
- Se reconciliaron el README y los tipos/documentacion de Cursor, el manual de
	archivos generados y los tests del installer/generator. El E2E ya consume el
	registry de forma dinamica y, en cold start, verifica cada ruta declarada;
	no se agrego una expectativa separada que confundiera superficie potencial
	con escritura garantizada.

### Validacion de la reparacion

- `npx vitest run packages/installer/tests/installer.test.ts packages/adapters/cursor/tests/generator.test.ts`: 2 archivos, 23 tests passed.
- `npx vitest run apps/cli/tests/e2e/adapters.e2e.test.ts`: 1 archivo, 3 tests passed.
- `npm run build`: passed (`tsc -b`).
- El chequeo de errores de VS Code no reporto errores en los archivos TypeScript
	afectados.

### Estado final

La reparacion fue verificada de forma independiente con tests de sync,
generators, workspace y `npm run build`. El incremento queda **closed**.

## General spec integration

- Specs `00..08` relevantes: sync cubre targets activos, marker real y Cursor
	declara su regla potencial no-clobber.
- `context/architecture/01`, `02`, `04`, `context/operations/02`,
	`context/references/01` y `context/TECHNICAL_CONTEXT.md`: registry, outputs
	y cobertura de packages reconciliados.

### Criterios ACC-028 verificados

- [x] El registry de Cursor incluye settings, AGENTS y
	  `.cursor/rules/axiom-common.mdc`.
- [x] Docs y tests distinguen la regla potencialmente no-clobbered de los
	  archivos efectivamente escritos en una corrida.
- [x] `installProfile.generatedFiles` y el E2E permanecen alineados con el
	  registry; el cold start materializa las tres rutas.
- [x] Tests dirigidos y `npm run build` pasan.

### Fallos clasificados de la reparacion

- No quedan fallos introducidos por ACC-028.
- Vitest mantiene el warning preexistente sobre la API CJS de Vite; no afecta
	los resultados.
- El cambio previo no relacionado en `apps/cli/tests/e2e/adapters.e2e.test.ts`
	se dejo intacto.
