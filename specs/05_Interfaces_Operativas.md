# 05 Interfaces Operativas

## Surfaces principales

1. CLI del runtime (`axiom`, entry point `apps/cli/dist/index.js`, instalado vía `scripts/install-global.mjs` como shim/`npm link` único en el PATH del usuario);
2. TUI operativa (`axiom tui`, `@axiom/tui`);
3. adapters que conectan Axiom con harnesses o IDEs (`@axiom/adapters-*`);
4. manifests/YAML que describen la configuración y topología del proyecto (`axiom.yaml`, `axiom.config/*.yaml`, `topology.yaml`).

## CLI: comandos documentados en profundidad

`init`, `join`, `configure`, `sync`, `start`, `audit`, `doctor`, `upgrade`, `tui`, `model` (`show`/`set`/`unset`/`reset`/`validate`), `components` (`list`/`show`/`install`/`uninstall`/`restore`), `skills` (`list`/`refresh`/`drift`). Fuente: `Axiom/docs/cli/*.md`.

## CLI: comandos presentes en código sin documentación operativa equivalente

`apps/cli/src/commands/` tiene 36 ficheros (~16.400 líneas). Además de los documentados arriba existen: `app`, `app-api`, `app-plugins`, `app-plugins-azure-devops`, `axiom-bug`, `axiom-increment`, `axiom-plan`, `axiom-qa-e2e`, `axiom-role`, `capability`, `context`, `gateway`, `intent`, `mcp`, `memory`, `projects`, `qa-archive-gate`, `repo`, `roles`, `self-update`, `toolchain`, `topology`. Estos se mencionan de forma dispersa en los documentos de incremento (`0019`–`0030`) pero no tienen una página propia bajo `docs/cli/`. Tratar su comportamiento como verificable en código, no como contrato documentado establecido, hasta que se audite comando por comando.

## TUI (`axiom` / `axiom tui`)

Punto de entrada: `axiom` sin subcomando abre la TUI (acción por defecto de Commander en `apps/cli/src/index.ts`, reusa `runTuiCli`); `axiom tui` es equivalente. Los subcomandos, `--help` y `--version` matchean antes y quedan sin cambios.

Con proyecto Axiom resuelto: menú operativo de 6 items (configurar, sincronizar, diagnóstico, upgrade, model routing, salir; 7 con "reparar instalación" — ver sección del roadmap más abajo). Para mutaciones (`configure`, `sync`, `upgrade`) muestra preview read-only, pide confirmación y muestra un post-run summary con restore point y follow-ups. Exige TTY.

Sin proyecto Axiom (y con TTY): en vez de abortar con `projectNotFound`, la TUI abre el menú de bootstrap `setup` (ver "wizard guiado de `init`" más abajo). En no-TTY se preserva `projectNotFound` + exit 1 (CI-safe). Fuente: `Axiom/docs/cli/tui.md`, package `@axiom/tui`.

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

## TUI — menú de bootstrap `setup` y wizard guiado de setup de workspace (`INC-20260705-tui-workspace-setup-wizard`)

Cuando `axiom`/`axiom tui` abre en una carpeta sin proyecto Axiom, la screen `setup` (items estáticos `SETUP_ITEMS`) ofrece: "Inicializar Axiom en esta carpeta" · "Actualizar Axiom" (self-update) · "Ver proyectos Axiom registrados" · "Salir". Elegir un proyecto de la lista de registrados hace `chdir` + abre la TUI operativa sobre él.

Elegir "Inicializar Axiom en esta carpeta" recorre un WIZARD GUIADO MULTI-REPO que prepara un entorno de trabajo completo (SDD + Spec + repos de rol, cruzados entre sí, registrados y con MCP configurado) en una sola operación. **Supersede al wizard single-repo de 6 pasos de `INC-20260703-tui-init-wizard`**: aquella acción `init` ya no llama a `runInit`, sino que ahora ensambla un `WorkspaceSetupSpec` y llama a `runWorkspaceSetup` (`INC-20260705-workspace-multirepo-setup-engine`, con generación MCP de `INC-20260705-workspace-mcp-generation`). El comando no-interactivo `axiom init`/`runInit` (RF-AXM-006) sigue siendo single-repo y sin cambios; solo cambió la acción `init` de la screen `setup`.

Orden de steps, cada uno con un default sensato pre-cargado que el usuario acepta (Enter) o cambia:

1. `name` (texto libre — del que se deriva el `projectId`);
2. `sddPath` (texto, default `.`);
3. `specPath` (texto, default `../<slug>.spec`);
4. `roles` (multi-select — cero o más roles funcionales/de implementación a integrar; opciones tomadas en vivo de `DEFAULT_PROFILES`, perfil `builder`, `activatesImplementationRoles` de `@axiom/install-profiles`, no una lista hardcodeada/duplicada);
5. `rolePath:<role>` (texto, uno por rol seleccionado, default `../<slug>-<role>` — inyectados dinámicamente, ver `expand` abajo);
6. `profile` (select);
7. `overlay` (select);
8. `adapters` (multi-select — cero o más adapters a instalar; opciones = los 8 `ADAPTER_TARGETS`; default pre-seleccionado `['opencode']`) — **supersede al step `target` `select` único** que describía la versión previa de este wizard (`INC-20260705-workspace-adapters-multiselect`);
9. un único resumen de confirmación.

El valor recogido del step `adapters` se serializa como los `value`s separados por coma en orden de opción; `buildWorkspaceSetupSpecFromWizard` los parsea en `spec.adapters` y deriva `spec.target = adapters[0]` (el primario, back-compat). La selección de adapters gobierna la generación de ficheros de adapter en TODOS los repos del workspace (control + spec + cada rol) y la proyección MCP a cada adapter MCP-capaz seleccionado — ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md) y [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md).

Cada step de path muestra en su subtítulo una nota `fs.existsSync` "(nuevo, se generará)"/"(existente, se parametrizará)" para su valor DEFAULT, y el step `adapters` avisa de que se generará config de adapter + MCP al confirmar. Solo al confirmar se llama a `runWorkspaceSetup`; en éxito se imprime un resumen de resultado (repos con su estado `created` real y autoritativo, path de topología, estado de registro, warnings) y se abre la TUI operativa sobre el repo de control. Cancelar (en cualquier step o en la confirmación) no llama a `runWorkspaceSetup` y vuelve al menú `setup` sin efectos. Si `runWorkspaceSetup` lanza, el error se captura, se imprime y se vuelve al menú `setup` sin crashear la TUI.

Implementación (capas): `@axiom/tui` (`packages/tui/src/driver.ts`) se mantiene GENÉRICO, sin conocimiento de roles/enums/negocio. Sobre las dos pantallas genéricas previas (`wizard-select`, `wizard-text`) este incremento añade:

- un nuevo tipo de step genérico `multi-select` (`WizardMultiSelectStep`, renderer puro `wizard-multi-select.ts` con `[x]`/`[ ]` por opción): elige cero o más opciones de una lista pasada por el caller; el valor recogido se serializa como string de `value`s separados por coma en orden de opción (vacío = ninguno);
- un mecanismo genérico `expand?: (answerValue: string) => WizardStep[]` en los steps: cuando un step con `expand` se responde, el driver splicea (una vez) los steps devueltos justo después del step actual — usado por el step `roles` para inyectar un step `text` por rol seleccionado.

El paquete no importa de `apps/cli` ni duplica valores de enum: `apps/cli/src/commands/tui.ts`'s `buildWorkspaceSetupWizardSteps` (reemplaza a `buildInitWizardSteps` como builder cableado en la acción `init`) arma la lista de steps con opciones/labels/defaults resueltos desde los enums de `apps/cli/src/commands/init.ts` y los roles de `@axiom/install-profiles`, y `runNoProjectBootstrap` ensambla el `WorkspaceSetupSpec` y llama a `runWorkspaceSetup`. Tanto el step `roles` como el step `adapters` (`INC-20260705-workspace-adapters-multiselect`) reusan el mismo tipo de step genérico `multi-select`; el step `adapters` toma sus opciones de `ADAPTER_TARGETS` (`init.ts`, fuente única) con `defaultValues: ['opencode']`, sin nuevo tipo de step en `@axiom/tui`. El wizard queda fuera del alcance de `axiom init` (no-interactivo, sin cambios) y de `axiom repo attach`.

## Adapters — MCP config por proyecto (cableada por el setup de workspace)

`generateOpencodeMcpJson`/`generateClaudeCodeMcpJson` (ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md) para el schema `mcp.yml`) están completamente implementados y testeados. Su primer y único call site vivo hoy es el setup de workspace multi-repo (`runWorkspaceSetup`, INC-20260705-workspace-mcp-generation): tras registrar el proyecto, escribe `.axiom/mcp.yml` en el repo de control y proyecta a `.opencode/mcp.json`/`.claude/mcp.json` como paso best-effort. La proyección se generalizó (`INC-20260705-workspace-adapters-multiselect`) para cubrir **cada adapter MCP-capaz seleccionado** en `spec.adapters` (el subconjunto `opencode`/`claude-code`), no solo el `target` primario: se escribe `.opencode/mcp.json` y/o `.claude/mcp.json` según qué adapters MCP-capaces se hayan multi-seleccionado. `.axiom/mcp.yml` se sigue escribiendo exactamente una vez por proyecto; los adapters seleccionados sin capacidad MCP no lo afectan. Fuera de esa ruta, `runConfigure`/`sync` siguen sin llamar a los generadores (ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md)).