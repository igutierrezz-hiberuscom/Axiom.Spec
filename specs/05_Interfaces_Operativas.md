# 05 Interfaces Operativas

## Surfaces principales

1. CLI del runtime (`axiom`, entry point `apps/cli/dist/index.js`, instalado vía `scripts/install-global.mjs` como shim/`npm link` único en el PATH del usuario);
2. TUI operativa (`axiom tui`, `@axiom/tui`);
3. adapters que conectan Axiom con harnesses o IDEs (`@axiom/adapters-*`);
4. manifests/YAML que describen la configuración y topología del proyecto (`axiom.yaml`, `axiom.spec/config/*.yaml`, `topology.yaml`).

## CLI: comandos documentados en profundidad

`init`, `join`, `configure`, `sync`, `start`, `audit`, `doctor`, `upgrade`, `tui`, `model` (`show`/`set`/`unset`/`reset`/`validate`), `components` (`list`/`show`/`install`/`uninstall`/`restore`), `skills` (`list`/`refresh`/`drift`). Fuente: `Axiom/docs/cli/*.md`.

## CLI: comandos presentes en código sin documentación operativa equivalente

`apps/cli/src/commands/` tiene 36 ficheros (~16.400 líneas). Además de los documentados arriba existen: `app`, `app-api`, `app-plugins`, `app-plugins-azure-devops`, `axiom-bug`, `axiom-increment`, `axiom-plan`, `axiom-qa-e2e`, `axiom-role`, `capability`, `context`, `gateway`, `intent`, `mcp`, `memory`, `projects`, `qa-archive-gate`, `repo`, `roles`, `self-update`, `toolchain`, `topology`. Estos se mencionan de forma dispersa en los documentos de incremento (`0019`–`0030`) pero no tienen una página propia bajo `docs/cli/`. Tratar su comportamiento como verificable en código, no como contrato documentado establecido, hasta que se audite comando por comando.

## TUI (`axiom tui`)

Menú de 6 items (configurar, sincronizar, diagnóstico, upgrade, model routing, salir). Para mutaciones (`configure`, `sync`, `upgrade`) muestra preview read-only, pide confirmación y muestra un post-run summary con restore point y follow-ups. Exige TTY. Fuente: `Axiom/docs/cli/tui.md`, package `@axiom/tui`.

## Adapters (surfaces por IDE/CLI externo)

6 adapters con package dedicado y contrato común `generate<Target>Config(args) → Promise<Result<GeneratorResult, AdapterGeneratorError>>`: `opencode` (`multi-mode`), `claude-code` (`single-mode`), `github-copilot`, `vscode`, `cursor`, `litellm` (los 4 últimos `fallback-only`). 3 targets adicionales declarados sin adapter dedicado (`copilot-vscode`, `antigravity`, `visual-studio-2026`) caen al default del IDE. Fuente: `Axiom/packages/adapters/README.md`.

## Documentación operativa navegable

`Axiom/docs/README.md` es el índice hacia manuales de instalación, configuración, uso diario, CLI, ficheros generados y troubleshooting — ya existe y está mantenido, no es un artefacto a construir desde cero.

## Regla

Las interfaces operativas se implementan en `Axiom/`, pero su comportamiento esperado se define primero en `Axiom.Spec/`. Regla vigente pero con brecha real: parte de la superficie de comandos (sección anterior) ya existe en código sin haber pasado primero por una spec formal en `Axiom.Spec/`.

## Superficie de comandos ampliada por el roadmap de rediseño (cerrado)

La superficie realmente alcanzable y primaria coincide con la forma preferida `axiom increment ...`, `axiom bug ...`, `axiom plan ...`, `axiom role ...` (registrados en `apps/cli/src/index.ts`), más:

- `axiom-adr`/`axiom-decision` (`create`/`link-plan`/`link-increment`/`list`, más `axiom-adr supersede <old-id> <new-id>`);
- `axiom index rebuild|validate|list` (todos los tipos de artefacto, `--json`);
- `axiom validate changes --project <id> --plan <planId>` (validación de write-scope);
- `axiom bootstrap from-code --level minimal|basic [--role <role>]` y `axiom bootstrap from-legacy-sdd <path> [--dry-run]`;
- `axiom repair` (general, top-level, distinto de `axiom toolchain repair` y `axiom mcp repair`).

Ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md) para el contrato funcional completo de cada uno.

### Colisión de nombrado: dos mecanismos "intent" no relacionados

No es un router único compartido, sino una colisión de nombrado: (1) `apps/cli/src/commands/intent.ts` (spec 0028) es un catálogo cerrado de 3 entradas `IntentChain` (`increment-new`, `bug-new`, `implement-role`), cada uno un wrapper literal alrededor de las funciones de subcomando estructuradas existentes — **no está registrado en `index.ts`**, inalcanzable por ningún usuario hoy pese a tener sus propios tests unitarios. (2) Los intent commands de `@axiom/orchestrator` (`state-machine.ts`) son una unión de 19 entradas `axiom-*-command`, cada una bloqueada por la precondición siempre-fallida `notImplemented`; ninguna ha sido cableada a lógica real por este roadmap. Como ningún candidato a router es alcanzable, no existe hoy ninguna ruta de código donde un usuario sea dirigido hacia uno en vez de los comandos estructurados. Un router opcional real (p. ej. cablear `intent.ts` en `index.ts` como `axiom run <action>`) sería superficie de producto nueva que requeriría su propio incremento — no un gap de cumplimiento.

## TUI — pantallas y flujos añadidos por el roadmap de rediseño

`router.ts`'s `MENU_ITEMS` tiene 7 entradas: configurar, sincronizar, diagnóstico, upgrade, **reparar instalación** (posición 4, entre upgrade y model routing — añadida por este roadmap), model routing, salir. `packages/tui/src/flows/configure.ts`/`upgrade.ts`/`repair.ts` son wrappers sin lógica de negocio (solo formateo de mensajes). Esto extiende, sin reemplazar, el menú de 6 items descrito en la sección "TUI" de arriba.

Las pantallas `mcp-inventory`/`memory-inventory` permanecen accesibles solo por flag de CLI, no como `MENU_ITEMS` de primera clase — promoverlas es un incremento explícitamente no iniciado (`INC-20260702-tui-menu-promote-inventory-screens`, diferido), no un hueco de esta spec.

## Adapters — MCP config por proyecto (no cableado a producción)

`generateOpencodeMcpJson`/`generateClaudeCodeMcpJson` (ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md) para el schema `mcp.yml`) están completamente implementados y testeados, pero `runConfigure`/`sync` todavía no los llaman, y ningún installer/scaffold escribe `mcp.yml` a un proyecto real.