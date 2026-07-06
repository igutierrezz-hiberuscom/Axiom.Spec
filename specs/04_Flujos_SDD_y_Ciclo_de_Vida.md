# 04 Flujos SDD y Ciclo de Vida

## Flujo base de este workspace (dogfooding, vigente)

1. especificar en `Axiom.Spec/` (specs numeradas, incrementos, bugs);
2. operar el workflow desde `Axiom.SDD/` (`AGENTS.md`: entender → localizar/crear spec → refinar con goal/scope/non-goals/acceptance criteria → implementar → validar → revisar → cerrar → integrar conocimiento estable en la spec);
3. implementar en `Axiom/`;
4. validar resultado (orden de descubrimiento: README → package scripts → task runners → test configs → build configs);
5. cerrar y reintegrar conocimiento en la spec. Un incremento/bug solo puede marcarse `closed` si: el objetivo es claro, hay acceptance criteria, se implementó (o hay justificación explícita de no-code), se corrió la validación disponible, se revisó contra el intent original y se integró conocimiento estable. Si falta algo, queda `status: pending` con motivo explícito.

## Ciclo de vida real del producto (lo que el runtime ejecuta para un proyecto adoptante)

```
init → join → configure → sync → start → audit → doctor
                                              ↓
                                          upgrade (cuando cambia la versión target)
```

Verificado por el script de smoke `Axiom/scripts/verify-first-project-readiness.mjs` (`npm run readiness:first-project`), que ejecuta exactamente esta secuencia extendida contra un proyecto temporal: `init → configure → toolchain repair → sync → start → gateway start → gateway status → audit → doctor`, y falla si algún paso devuelve exit code distinto de 0, falta un artefacto esperado, o `gateway status` no reporta `state: active`.

Contrato por comando (lee/escribe) detallado en [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md) y en `context/architecture/`.

### Onboarding del operador (entrada TUI y wizard de `init`)

El punto de entrada interactivo es `axiom` sin subcomando, que abre la TUI. En una carpeta sin proyecto Axiom (con TTY), la TUI ofrece un menú de bootstrap desde el que se inicializa el proyecto vía un **wizard guiado**: pregunta `name → role → layout → profile → overlay → target` con defaults inferidos, muestra un resumen de confirmación, y solo al confirmar ejecuta `runInit` (el primer paso del ciclo de vida de arriba) abriendo después la TUI operativa. `axiom init` sigue siendo la vía no-interactiva/scriptable equivalente. Detalle de UI y capas en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md) (RF-AXM-023 en [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md)).

Modelo de datos tras el init (fuente única de verdad): `axiom.yaml` es la fuente autoral de la identidad del proyecto (`projectId`/`name`/`repoId`/`role`) y del mapa de repos. `init` escribe `axiom.yaml`, `.gitignore`, `.axiom-state/local/` y `.axiom-state/<projectName>/init.json` (con `profileTriple`+`createdAt`+`version`, sin `projectName` propio) — y ya NO escribe `topology.yaml`, que se deriva de `axiom.yaml` al leer y se materializa de forma perezosa solo al asignar roles (`INC-20260703-config-dedup`; ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)).

## Baseline operativa actual

1. un solo rol funcional cubierto en profundidad (`functionalProfile: builder`);
2. soporte multi-repo dentro de un proyecto vía `@axiom/topology` (`installed-multi-repo` layout), no todavía como separación de repos de Axiom-el-producto;
3. ejecución local (`local-only`) o mediada por gateway (`standard`/`enterprise`), y adapters hacia 6 targets con package dedicado + 3 targets `fallback-only` sin adapter propio.

## Fuera de la baseline inicial (documentado explícitamente como no-goal del MVP)

Según `Axiom/docs/first-project-readiness.md`: overlays `standard`/`enterprise` como camino inicial obligatorio, `visual-studio-2026` como baseline de primer arranque, providers post-MVP (`engram`, `codegraph`, `graphify`) como requisito de entrada, bridges externos/plugins/lanes paralelos avanzados, e instalación user-level del binario como paso obligatorio.

## Punto de partida de un repo de spec recién creado (setup de workspace)

Cuando el setup de workspace multi-repo (`runWorkspaceSetup`) crea desde cero el repo de Spec, lo scaffoldea desde la base canónica de plantillas (`specs/README.md` + `specs/00..08` + `context/TECHNICAL_CONTEXT.md`/`README.md` + los directorios estructurales `specs/{increments,bugs,archive}` y `context/*`) — `INC-20260705-workspace-spec-base`. Así un workspace nuevo arranca ya con la estructura numerada de spec y las carpetas de artefacto donde vive el ciclo de vida de abajo, en vez de una carpeta vacía. Es best-effort y guardado per-file (nunca sobrescribe contenido existente); ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md) y [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

## Ciclo de vida de artefactos (increment/bug/plan/ADR/decision) — roadmap de rediseño, cerrado

Capacidad añadida de forma aditiva por el roadmap de rediseño (23 incrementos, cerrado 2026-07-03); convive con el flujo base de dogfooding de la sección anterior sin reemplazarlo.

1. **Creación**: `axiom-increment/bug/plan/adr/decision create` escribe una carpeta nueva `<specPath>/{increments,bugs,plans,adr,decisions}/<ID>/` con `metadata.yml`, vía las primitivas de `@axiom/workflow`'s `artifact-store.ts`. El ID se genera por sistema (no texto libre).
2. **Refinado/especificación**: `refine`/`specify` actualizan el `metadata.yml` existente; `link-plan`/`link-increment`/`link-bug` establecen relaciones entre artefactos.
3. **Transición de estado**: para `increment`/`bug`/`plan`, el estado (`status: WorkflowState`, 9 valores) es dirigido por la máquina de estados de `workflow-state.json` — pero esa máquina es UN registro singleton por `WorkflowId` (tipo de workflow), no por instancia de artefacto; `metadata.yml` (identidad de instancia) y `workflow-state.json` (máquina de estados por tipo) son almacenes independientes. Para `adr`/`decision`, el estado sigue su propio vocabulario no dirigido por máquina de estados (`AdrStatus`/`DecisionStatus`, ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md)) y se escribe directamente en `metadata.yml`, sin pasar por `workflow-state.json`.
4. **Supersesión de ADR**: `axiom-adr supersede <old-id> <new-id>` es la única transición especial — actualiza ambos ADR atómicamente; Decision no tiene equivalente (sin cadena de supersesión en su schema).
5. **Cierre**: sigue las mismas reglas de cierre que el flujo base de dogfooding — `closed` solo si objetivo claro, acceptance criteria, implementación o justificación no-code, validación ejecutada, revisión contra intent, y conocimiento estable integrado.

## Flujos de bootstrap

Dos rutas mecánicas de alcance mínimo (no las cadenas literales de 7 subagentes del documento fuente):

1. **`axiom bootstrap from-code --level minimal|basic [--role <role>]`**: analiza el repo actual (`detectStacks`, `buildRepoMap`, `detectCommands`), redacta documentos de contexto técnico con banner `<!-- AXIOM:DRAFT -->`, y puebla `TechnicalContextIndex.available` (nunca `mandatory.*`, que queda para curación humana). Nivel 2 (Standard) en adelante queda diferido — requiere comprensión arquitectónica/de negocio genuina.
2. **`axiom bootstrap from-legacy-sdd <path> [--dry-run]`**: escanea un repo SDD/spec legado (carpetas `increments/`/`bugs/`/`plans/`/`adr/`/`decisions/`), migra cada entrada como artefacto nuevo vía las primitivas de `@axiom/workflow` (nunca sobrescribe, nunca aborta el lote ante colisión), con banner de procedencia `<!-- AXIOM:MIGRATED -->`. `--dry-run` no escribe nada. Ninguna de las dos rutas toca `workflow-state.json`.

Ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md) (RF-AXM-021) para el detalle completo de ambas rutas.

## Flujos operativos de `configure`/`upgrade`/`repair` sobre instalaciones existentes

- `axiom configure`: re-aplica el perfil persistido completo (single-shot, sin flags incrementales). No cubre añadir/quitar repo, rol, adapter o tool/MCP — ver el hueco de 7 operaciones documentado en [02_Requisitos_No_Funcionales.md](02_Requisitos_No_Funcionales.md).
- `axiom upgrade`: calcula y aplica migraciones de `ManagedState` con checkpoint rollback-first; soporta `--dry-run`/`--from-checkpoint`/`--target-version`.
- `axiom repair`: ejecuta `axiom doctor`, agrupa hallazgos por categoría, y despacha las 4 categorías conocidas-como-corregibles (`install-profiles`, `artifact-index`, `toolchain`, `memory`) a las funciones de reparación ya existentes; el resto se reporta como no auto-corregible. Soporta `--dry-run`.
- Los tres flujos anteriores están cableados en la TUI (`packages/tui/src/flows/configure.ts`/`upgrade.ts`/`repair.ts`, wrappers sin lógica de negocio); `repair` y `upgrade` exigen preview dry-run + confirmación Y/n antes de mutar el filesystem.

Ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md) (RF-AXM-022) y [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md) para el detalle de datos persistidos por cada operación.