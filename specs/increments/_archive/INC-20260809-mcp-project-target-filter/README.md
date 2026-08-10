# Filtrar MCP por proyecto y target

> **Codigo**: INC-20260809-mcp-project-target-filter
> **Estado**: closed
> **Fecha de creacion**: 2026-08-09
> **Tipo de cambio**: modificar

## Objetivo

Evitar que un agente vea o use un MCP perteneciente a otro proyecto, incluso
cuando el destino solo admite configuracion global de usuario.

## Contexto y motivacion

El servidor MCP ejecutable ya queda ligado a un proyecto y rechaza argumentos
cross-project. Eso protege el proceso una vez arrancado, pero Codex y
Antigravity usan configuraciones globales y actualmente solo reciben una nota
manual. Ademas, `@axiom/isolation` usa nombres logicos antiguos (`sdd`, `spec`,
`serena`) que no coinciden siempre con los procesos ejecutables actuales.

## Alcance tipado

- `Axiom/apps/cli/src/commands/{workspace-mcp,native-mcp-config,mcp-serve}.ts`.
- `Axiom/packages/{isolation,mcp-server,user-workspace,mcp-tools}/src/` solo en
	los puntos necesarios para resolver y aplicar el filtro.
- `Axiom/axiom.config/{mcp-manifest,integrations}.yaml` y loaders/validadores
	relacionados si hace falta reconciliar IDs.
- Configuracion nativa project-scoped y notas user-global de Codex/Antigravity.
- Tests de dos proyectos, incluyendo KVP25 y EMT, y doctor si se necesita un
	check observable.

## Decisiones cerradas

- La disponibilidad efectiva se deriva del proyecto actual, no de una lista
	global fija.
- Solo se proyectan MCP declarados, enabled, project-bound y compatibles con
	las capacidades/providers del proyecto.
- Para targets user-global, si Axiom no puede demostrar la identidad del
	proyecto al lanzar el server, el default es no ofrecerlo.
- Un mismatch de proyecto debe fallar cerrado con un mensaje accionable.
- Se conservan servidores reales; no se crea un broker enterprise nuevo.

## No incluido

- No sustituir el aislamiento ya enforced por el servidor.
- No escribir automaticamente en ficheros globales de usuario sin un guard
	verificable.
- No permitir shell arbitrario ni resolver un proyecto por scan ambiguo.

## Criterios de aceptacion


- [x] La proyeccion nativa filtra por `mcp.yml`, `mcp-manifest.yaml`, capacidades
	 habilitadas y target, con IDs logicos y ejecutables reconciliados.
- [x] KVP25 no puede ofrecer su MCP al proyecto EMT, y EMT no puede ofrecer el de
	 KVP25, aun compartiendo la maquina y el fichero global.
- [x] Codex y Antigravity quedan fuera por defecto cuando no existe una forma
	 segura de pinnear el proyecto; muestran una advertencia clara.
- [x] Los servidores MCP project-scoped existentes siguen funcionando y el
	 servidor rechaza referencias cross-project en `tools/call`.
- [x] Tests focalizados, doctor relevante y `npm run build` pasan.
- [x] El provisioning de worktree pasa por `filterProjectBoundMcpServers`
	 antes de llamar a cualquier writer MCP nativo.
- [x] `mcp.yml` ausente, inválido o perteneciente a otro `projectId` produce
	 cero brokers en el worktree, warning y limpieza selectiva de stale.
- [x] La reconciliación nativa cubre `.mcp.json`, `.cursor/mcp.json`,
	 `opencode.json` y `.vscode/mcp.json`, preservando servers y keys custom.

## Validacion prevista

- Tests unitarios de resolucion/filtrado y de native MCP.
- E2E con dos proyectos y configuracion global simulada.
- Tests de `mcp-server`/`mcp-tools` y `npm run build` desde `Axiom/`.

## Integracion de spec y contexto

No se actualizan `Axiom.Spec/specs/00..08` ni `Axiom.Spec/context/**`, conforme al
alcance solicitado. La implementacion y sus decisiones estables quedan
documentadas en este incremento.

## Implementation notes

- `filterProjectBoundMcpServers` es el unico gate previo a
	`writeNativeMcpConfig`. Confirma el registry del `projectId`, carga y valida
	el `mcp.yml` project-scoped y reconcilia el manifest separado mediante la
	tabla cerrada `sdd` -> `sdd-mcp-server`, `spec` -> `spec-mcp-broker` y
	`axiom` -> `axiom-mcp-broker`. El provisioning de worktree resuelve de forma
	determinista `homeDir`, `mcp.yml` y `mcp-manifest.yaml` desde el repo Axiom
	explícito o `sourceRootPath`, y queda fail-closed si el binding completo no
	se confirma.
- Entries ausentes, invalidas, disabled, con `targetRepo` ajeno o con aliases
	ambiguos producen warning y cero brokers nativos. Los callers de setup,
	member install, operaciones incrementales y `workspace mcp-config` pasan
	`projectId`, home, `mcp.yml` y manifest explicitamente.
- Codex y Antigravity no escriben archivos globales ni recomiendan copiar
	servidores sin binding. `engram` y code-intel sólo se proyectan cuando el
	caller los entrega ligados al proyecto confirmado.
- Los writers nativos reconcilian únicamente el allowlist de IDs gestionados
	por Axiom (`sdd-mcp-server`, `spec-mcp-broker`, `axiom-mcp-broker`, `cmm`,
	`serena` y `engram`). Una lista permitida vacía elimina esos IDs stale de
	los cuatro schemas nativos, conserva servidores ajenos y no crea un archivo
	nuevo si no existía. JSON inválido o mapas con schema ilegible se conservan
	con warning y la escritura sigue siendo atómica.
- No se proyecta automáticamente engram ni code-intel si el proyecto no puede
	confirmarse; targets user-global no reciben configuración global.
- El contexto del servidor MCP conserva el binding de startup: un proyecto
	multi-repo sin `mcp.yml` validado no expone `tools/list` ni acepta
	`tools/call`; el fallback single-repo sigue siendo local y no acepta un
	`projectId` inventado por el caller.

## Validation

- Validación base previa a la reparación:
- `npx vitest run apps/cli/tests/workspace-mcp.test.ts apps/cli/tests/native-mcp-config.test.ts apps/cli/tests/mcp-real-manifest.test.ts apps/cli/tests/mcp-serve.test.ts apps/cli/tests/member-install.test.ts apps/cli/tests/workspace-incremental.test.ts apps/cli/tests/workspace-command.test.ts packages/user-workspace/tests/mcp-config.test.ts packages/mcp-server/tests/server.test.ts packages/mcp-server/tests/stdio.test.ts packages/mcp-server/tests/input-builders.test.ts --reporter=dot`
	-> 11 archivos, 185 tests, todos pasan.
- `npm run build` -> `tsc -b` pasa.
- `npm run doctor` -> `PASS`, 45/60 checks OK, 0 fallos, 4 advertencias y 11
	checks omitidos. Las advertencias son diagnosticos preexistentes del repo
	(scope documental/runtime legacy, cobertura opcional y formato legacy de
	`axiom.yaml`); ninguna corresponde a este incremento.
- `get_errors` sobre todos los fuentes tocados -> sin errores.
- Revision manual: el filtro ocurre antes de construir `NativeMcpServerInput`
	y los caminos de config ausente/invalida/mismatch retornan cero brokers.
- Validación de reparación:
- `npx vitest run apps/cli/tests/workspace-worktree-provision.test.ts apps/cli/tests/workspace-mcp.test.ts apps/cli/tests/native-mcp-config.test.ts --reporter=dot`
	-> 3 archivos, 86 tests, todos pasan; cubre worktree gated, stale cross-
	project y los cuatro schemas nativos.
- `npx vitest run apps/cli/tests/axiom-role-worktree.test.ts --reporter=dot`
	-> 1 archivo, 7 tests, todos pasan.
- `npx vitest run apps/cli/tests/e2e/worktree-provision.e2e.test.ts --reporter=dot`
	-> 1 archivo, 1 test, pasa con registry y binding MCP temporales explícitos.
- `npm run build` -> `tsc -b` pasa después de la reparación.
- `get_errors` sobre fuentes y tests tocados -> sin errores.

## Correcciones de revision independiente

La proyeccion de worktrees debe pasar por `filterProjectBoundMcpServers` antes
de llamar a `writeNativeMcpConfig`; no puede construir el broker unificado y
escribirlo directamente. Cuando el filtro falla cerrado, la reconciliacion
debe retirar solo las entradas gestionadas por Axiom de los archivos nativos
existentes y preservar servidores ajenos del usuario. Se cubren ambos casos:
`mcp.yml` ausente/cruzado y archivo nativo previo con entradas Axiom stale más
una entrada propia del usuario.

## Result

La reparación cumple el filtro project-bound en worktree y la reconciliación
selectiva de configs nativos. El provisioning no escribe ni recomienda MCP
global para Codex/Antigravity; ante identidad o configuración no confirmable
entrega cero servers, avisa y elimina únicamente IDs Axiom stale. El
incremento queda **closed**.

## Receipts

- `Axiom.Spec/specs/increments/INC-20260809-mcp-project-target-filter/candidate-freeze.json`
	-> el hash vigente se revalida por el orquestador antes de cada reparación.
- Freeze vigente posterior a la reparacion: se revalida por el orquestador
	antes del cierre.
- Apply receipt de la reparacion: `native-mcp-config.ts`, `workspace-mcp.ts`,
	`workspace-worktree-provision.ts` y sus tests dirigidos, incluida la fixture
	E2E de provisioning afectada por el nuevo binding obligatorio.
- Verify receipt: `94` tests MCP/worktree/callers + `npm run build` y
	`get_errors`, todos OK.
- La integración general y el contexto técnico fueron aplicados por el
	orquestador después de la reparación.

## General spec integration

- Specs `00`, `03`, `05`, `06`, `07` y `08`: filtro project-bound, limpieza
	selectiva, worktree sin bypass y targets user-global cerrados por defecto.
- `context/architecture/04`, `context/integrations/01` y
	`context/TECHNICAL_CONTEXT.md`: binding, IDs ejecutables y límites MCP
	reconciliados.
