# Unificar target de Copilot

> **Codigo**: INC-20260809-unify-copilot-target
> **Estado**: closed
> **Fecha de creacion**: 2026-08-09
> **Tipo de cambio**: modificar

## Objetivo

Dejar un unico destino publico de instrucciones para GitHub Copilot, Copilot
CLI, VS Code y Visual Studio: `github-copilot` genera o consume
`.github/copilot-instructions.md`.

## Contexto y motivacion

`ACC-024` ya unifico el writer y la ruta de instrucciones. Los nombres
`github-copilot` y `copilot-vscode` siguen representando la misma instruccion,
mientras `vscode` mezcla la configuracion del editor con el prompt. La fuente
canonica debe ser una sola; cualquier salida nativa debe ser derivada.

## Dependencias

- `INC-20260809-remove-litellm-target` no es bloqueante para el writer, pero el
	conjunto final de targets debe quedar reconciliado.
- `ACC-024` / `INC-20260809-unify-copilot-instruction-surfaces` es la base ya
	archivada y validada.

## Alcance tipado

- `Axiom/packages/document-bootstrap/` y
	`Axiom/packages/adapters/github-copilot/` solo para converger APIs y aliases.
- `Axiom/apps/cli/src/commands/workspace-adapters.ts` y
	`Axiom/packages/cli-commands/src/commands/{configure,sync}.ts`.
- `Axiom/packages/installer/src/registry.ts`,
	`Axiom/packages/model-routing/src/support-matrix.ts`, tipos y listas de
	targets.
- `Axiom/axiom.config/profiles.yaml`, docs y tests afectados.

## Decisiones cerradas

- `github-copilot` es el target publico canonico.
- `copilot-vscode` se conserva solo como alias legacy durante la migracion.
- `vscode` no es una segunda fuente de instrucciones; solo representa
	`settings.json`, `extensions.json` y `mcp.json` opcionales.
- `.github/instructions/*.instructions.md` son process surfaces por ruta y no
	se deben duplicar como prompt general.

## No incluido

- No retirar en este incremento la configuracion nativa de VS Code.
- No cambiar el contenido funcional ya validado por `ACC-024`.
- No crear una nueva fuente `AGENTS.md` mantenida manualmente.

## Criterios de aceptacion

- [x] El contrato publico documenta un solo target de instrucciones: `github-copilot`.
- [x] `copilot-vscode` se normaliza de forma determinista al target canonico y los
	estados legacy no generan archivos duplicados.
- [x] `vscode` solo genera configuracion de VS Code y no otro prompt.
- [x] `configure`, `workspace setup` y `sync` usan el writer comun y preservan
	idempotencia, bloque `TEAM:CUSTOM` y migracion legacy.
- [x] Las pruebas focalizadas y `npm run build` pasan.

## Implementation notes

- `CANONICAL_ADAPTER_TARGETS`, `DEFAULT_PROFILES`, `profiles.yaml`, el installer
	y `SUPPORT_MATRIX` quedan reconciliados sobre los ocho targets activos;
	`github-copilot` es el unico target publico de instrucciones.
- `normalizeAdapterTarget` y `normalizeAdapterTargets` convierten
	`copilot-vscode` en `github-copilot`, preservan el orden determinista y
	eliminan duplicados antes de instalar, generar superficies o proyectar MCP.
- `configure`, `sync`, `generateWorkspaceAdapters`, superficies de proceso y
	`member install` usan la salida canonica. `vscode` conserva un dispatch
	separado para `settings.json`, `extensions.json` y `mcp.json`; no escribe
	instrucciones.
- El writer comun canonicaliza `target.id`, migra
	`.vscode/copilot-instructions.md` a `.github/copilot-instructions.md`,
	conserva `TEAM:CUSTOM` y evita doble receipt para configuraciones MCP
	compartidas.
- Se retiraron el alias de los selectores, plantillas y documentacion activas.
	Las referencias restantes son exclusivamente compatibilidad de estado legacy,
	tipos de entrada o pruebas de migracion. El contenido funcional de ACC-024 y
	la retirada de LiteLLM no se modificaron.

## Archivos afectados

- Targets y perfiles: `packages/capability-model`, `packages/install-profiles`,
	`packages/model-routing`, `packages/installer`, `axiom.config/profiles.yaml`.
- Emision: `packages/document-bootstrap`,
	`packages/adapters/github-copilot`, `packages/cli-commands`.
- CLI/workspace: `apps/cli/src/commands/{init,workspace-adapters,workspace-process-surfaces,member-install,native-mcp-config,workspace-mcp}.ts`.
- Tests y documentacion directamente afectados en esos paquetes, `apps/cli` y
	`docs/`; no se modificaron `Axiom.Spec/specs/00..08` ni
	`Axiom.Spec/context/**`.

## Validacion prevista

- Tests de document-bootstrap, adapters Copilot, installer, CLI y workspace.
- Barrido de targets activos y archivos generados.
- `npm run build` desde `Axiom/`.

## Validacion ejecutada

- `[BUILD-001]` `npm run build` desde `Axiom/`: **PASS** (`tsc -b`, exit code 0).
- `[TEST-001]` `npx vitest run packages/document-bootstrap/tests packages/adapters/github-copilot/tests packages/installer/tests/installer.test.ts packages/install-profiles/tests packages/model-routing/tests/support-matrix.test.ts apps/cli/tests/configure.test.ts apps/cli/tests/configure-template-resolver-dedup.test.ts apps/cli/tests/document-bootstrap.test.ts apps/cli/tests/member-install.test.ts apps/cli/tests/workspace-adapters.test.ts apps/cli/tests/workspace-process-surfaces.test.ts apps/cli/tests/workspace-mcp.test.ts apps/cli/tests/workspace-incremental.test.ts apps/cli/tests/workspace-setup.test.ts apps/cli/tests/sync.test.ts apps/cli/tests/e2e/adapters.e2e.test.ts apps/cli/tests/e2e/workspace-mcp.e2e.test.ts`: **PASS**, 27 test files / 292 tests.
- `[TEST-002]` `npx vitest run apps/cli/tests/launcher-onboarding.test.ts apps/cli/tests/schemaversion2-e2e.test.ts`: **PASS**, 2 test files / 23 tests.
- `[TEST-003]` `npx vitest run apps/cli/tests/sync.test.ts apps/cli/tests/workspace-mcp.test.ts apps/cli/tests/native-mcp-config.test.ts`: **PASS**, 3 test files / 73 tests.
- `[CHECK-001]` Barrido de referencias activas: el alias solo queda en las
	fronteras de migracion/compatibilidad y no como selector publico independiente.

## Fallos clasificados

- **Preexistentes en el candidato congelado, corregidos durante este apply**:
	faltaba la reexportacion runtime de `normalizeAdapterTarget` desde
	`@axiom/install-profiles`, y habia expectativas antiguas que contaban tres
	outputs para `copilot-vscode` o lo mostraban en el selector publico. Tambien
	habia un fixture E2E que permitia solo el alias, no su target normalizado.
- **Introducidos por este incremento**: ninguno pendiente. Las regresiones
	detectadas durante la edicion fueron expectativas/fixtures obsoletos y se
	repararon en el mismo slice; la bateria final queda verde.

## Result

Incremento implementado y cerrado. Se cumplen los cinco criterios de
aceptacion; no quedan archivos duplicados para Copilot legacy y `vscode` solo
materializa configuracion nativa del editor.

## General spec integration

- Specs `00`, `01`, `02`, `03`, `05`, `06` y `08`: una fuente común de
	instrucciones, alias Copilot y VS Code editor-only.
- `context/architecture/01-02`, `context/architecture/04` y
	`context/TECHNICAL_CONTEXT.md`: targets y outputs actuales reconciliados.
