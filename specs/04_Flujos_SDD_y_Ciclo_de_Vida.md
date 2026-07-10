# 04 Flujos SDD y Ciclo de Vida

## Flujo base de este workspace (dogfooding, vigente)

1. especificar en `Axiom.Spec/` (specs numeradas, incrementos, bugs);
2. operar el workflow desde `Axiom.SDD/` (`AGENTS.md`: entender → localizar/crear spec → refinar con goal/scope/non-goals/acceptance criteria → implementar → validar → revisar → cerrar → integrar conocimiento estable en la spec);
3. implementar en `Axiom/`;
4. validar resultado (orden de descubrimiento: README → package scripts → task runners → test configs → build configs);
5. cerrar y reintegrar conocimiento en la spec. Un incremento/bug solo puede marcarse `closed` si: el objetivo es claro, hay acceptance criteria, se implementó (o hay justificación explícita de no-code), se corrió la validación disponible, se revisó contra el intent original y se integró conocimiento estable. Si falta algo, queda `status: pending` con motivo explícito.

### Tooling de orquestación del workspace: `/axiom-autopilot` (INC-20260708-axiom-autopilot-command)

Nota de tooling de desarrollo (no funcionalidad de runtime del producto Axiom): el flujo base de arriba se puede conducir de forma **desatendida** sobre una tanda de cambios mediante el comando/skill de Claude Code `/axiom-autopilot` (`.claude/commands/axiom-autopilot.md` + `.claude/skills/axiom-autopilot.md`). Codifica el playbook manual de orquestación multi-incremento —descomponer una tanda de cambios en incrementos foco, ejecutarlos secuencialmente vía subagentes `axiom-increment`, auto-resolver ambigüedad registrando cada decisión, verificar cada resultado de forma independiente, integrar el conocimiento estable en los ficheros canónicos `00–08` al final, archivar cada incremento y cerrar con un resumen de decisiones— sin detenerse a preguntar y sin ejecutar git mutante. Vive en `.claude/` (config de Claude Code para CONSTRUIR Axiom), no en `Axiom/` (runtime del producto); es la contraparte de tanda del comando `axiom-increment` por incremento único, no una capacidad del producto adoptable por proyectos de terceros.

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
6. **Archivado físico (`INC-20260710-lifecycle-correctness-fixes`)**: `axiom-increment archive` / `axiom-bug archive` no sólo escriben `status: archived` en `metadata.yml` — al completar esa transición, mueven físicamente (rename atómico) la carpeta de la instancia de `<specPath>/{increments,bugs}/<ID>/` a `<specPath>/{increments,bugs}/_archive/<ID>/` (`archiveArtifactDir`, `@axiom/workflow`'s `artifact-store.ts`). Nunca sobreescribe: si ya existe una carpeta archivada con el mismo ID, la operación falla con un mensaje claro en vez de clobberear. Best-effort desde la CLI (un fallo del move se reporta por stderr pero no revierte la transición de estado ya persistida). `listArtifacts`/`axiom-increment list` sólo escanea el nivel directo de `<kindFolder>/`, así que un artefacto archivado deja de aparecer en el listado por defecto tras esta relocación — comportamiento esperado, consistente con la convención `_archive/` ya usada por el propio repo de spec (`specs/increments/_archive/`).

## Flujos de bootstrap

Dos rutas mecánicas de alcance mínimo (no las cadenas literales de 7 subagentes del documento fuente):

1. **`axiom bootstrap from-code --level minimal|basic [--role <role>]`**: analiza el repo actual (`detectStacks`, `buildRepoMap`, `detectCommands`), redacta documentos de contexto técnico con banner `<!-- AXIOM:DRAFT -->`, y puebla `TechnicalContextIndex.available` (nunca `mandatory.*`, que queda para curación humana). Nivel 2 (Standard) en adelante queda diferido — requiere comprensión arquitectónica/de negocio genuina.
2. **`axiom bootstrap from-legacy-sdd <path> [--dry-run]`**: escanea un repo SDD/spec legado (carpetas `increments/`/`bugs/`/`plans/`/`adr/`/`decisions/`), migra cada entrada como artefacto nuevo vía las primitivas de `@axiom/workflow` (nunca sobrescribe, nunca aborta el lote ante colisión), con banner de procedencia `<!-- AXIOM:MIGRATED -->`. `--dry-run` no escribe nada. Ninguna de las dos rutas toca `workflow-state.json`.

Ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md) (RF-AXM-021) para el detalle completo de ambas rutas.

## Flujos operativos de `configure`/`upgrade`/`repair` sobre instalaciones existentes

- `axiom configure`: re-aplica el perfil persistido completo (single-shot, sin flags incrementales). No cubre añadir/quitar repo, rol, adapter o tool/MCP — ver el hueco de 7 operaciones (NFR-AXM-015) documentado en [02_Requisitos_No_Funcionales.md](02_Requisitos_No_Funcionales.md). Desde `INC-20260708-incremental-operations` las 4 operaciones ADD (`axiom repo/adapter/provider/role add`, idempotentes y no-clobber) cubren la mitad aditiva de ese hueco; los REMOVE quedan diferidos.
- `axiom upgrade`: calcula y aplica migraciones de `ManagedState` con checkpoint rollback-first; soporta `--dry-run`/`--from-checkpoint`/`--target-version`.
- `axiom repair`: ejecuta `axiom doctor`, agrupa hallazgos por categoría, y despacha las 4 categorías conocidas-como-corregibles (`install-profiles`, `artifact-index`, `toolchain`, `memory`) a las funciones de reparación ya existentes; el resto se reporta como no auto-corregible. Soporta `--dry-run`.
- Los tres flujos anteriores están cableados en la TUI (`packages/tui/src/flows/configure.ts`/`upgrade.ts`/`repair.ts`, wrappers sin lógica de negocio); `repair` y `upgrade` exigen preview dry-run + confirmación Y/n antes de mutar el filesystem.

Ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md) (RF-AXM-022) y [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md) para el detalle de datos persistidos por cada operación.

## Aprendizaje continuo (captura al cierre, recall al inicio del trabajo) — INC-20260708-continuous-learning

El ciclo de vida gana una capa de aprendizaje continuo MÍNIMA y CONCRETA, anclada en sistemas reales ya entregados (la memoria engram-backed de `@axiom/memory`, ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md), y los registros reales de audit-trail) — no un "motor de instintos" especulativo. Se apoya en tres funciones deterministas de `@axiom/memory` (`extractLessons`/`persistLessons`/`recallLessons`) y se surface en el flujo por dos puntos:

- **Recall al inicio del trabajo**: `axiom context status` (el comando existente que un agente/operador corre para ver el estado del proyecto) añade un bloque best-effort de "lecciones recientes" (las lecciones más relevantes vía `resolveMemoryBackend` + `recallLessons`), que nunca falla el comando si la memoria está vacía o no disponible.
- **Captura explícita**: `axiom learn capture [--from-audit] [--text "..."]` persiste una lección (como `MemoryEntry` `kind: 'pattern'` tag `'lesson'`, topic-keyed para upsert-in-place); `axiom learn list` la recupera. La captura es SIEMPRE explícita (nunca un efecto lateral silencioso de otro comando). Los hooks de sesión (SessionStart/SessionStop) quedan como un snippet `.claude/settings.json` documentado y OPT-IN — Axiom no ejecuta ningún motor de hooks de sesión ni auto-aplica nada. Ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md).

## Señal de delegación no-bloqueante en el ciclo — INC-20260708-delegation-triggers

`axiom context status` surface además un bloque best-effort de `delegationSuggestions` (mismo contrato de degradación que el bloque de lecciones): lee un `session-metrics.json` OPCIONAL en `.axiom-state/<project>/` y, si está presente y cruza umbral, sugiere delegar a un agent del roster curado. Es una señal de sugerencia PURA y estructuralmente no-bloqueante (nunca en el control flow del state machine) — el detalle del evaluador y del roster vive en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

## Revisión con ledger de hallazgos (loop-until-dry + re-review scoped) — INC-20260709-review-findings-ledger

El paso de REVISIÓN del ciclo de vida (el reviewer de producto `axiom-reviewer` y el agent bootstrap `axiom-review` de este workspace) adopta el contrato **review-findings-ledger** (adoptado de gentle-ai, adaptado a Axiom — no copiado literal):

- **Primera pasada exhaustiva (loop-until-dry)**: en vez de una única lectura, el reviewer barre el scope repetidamente hasta que N barridos consecutivos no arrojen hallazgos NUEVOS (default N=2; una lente readability-only puede usar N=1; techo duro de 4 barridos, el loop es finito). Una re-review usa N=1.
- **Ledger de hallazgos persistido**: cada hallazgo se registra con `id` (`{LENS}-{NNN}`, p. ej. `REVIEW-001`), `lens`, `location` (`file:line`), `severity` (BLOCKER|CRITICAL|WARNING|SUGGESTION), `status` (open|fixed|verified|wont-fix|info) y `evidence`. Si la primera pasada no encuentra nada, se persiste un registro de ledger VACÍO (no se salta la persistencia).
- **Persistencia que honra el artifact store**: si existe la carpeta de artefacto en `Axiom.Spec` para el cambio bajo revisión (`.../increments/<INC-id>/` o `.../bugs/<BUG-id>/`, modelo folder-per-artifact de `@axiom/workflow`, ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)), el ledger se escribe como `review-ledger.md` dentro de esa carpeta; si no, se upsertea un topic de Engram (`topicKey` `sdd/<change>/review-ledger`, reusando el backend de memoria engram-backed de `@axiom/memory`); si ninguno está disponible, queda in-context (sin escribir fichero ni topic, completando review → fix → re-review en la misma sesión).
- **Re-review scoped**: una re-review toma el ledger persistido + el diff del fix, verifica la resolución de cada hallazgo del ledger y revisa SOLO las líneas tocadas por el fix (nunca re-lee el diff original completo). Un hallazgo sobre una línea NO tocada se registra con `status: info` (señal de calidad de la primera pasada) y NO dispara por sí solo una ronda nueva.

El contrato se bundlea como UNA constante TS canónica (`@axiom/document-bootstrap`'s `review-ledger-contract.ts`: `REVIEW_LEDGER_CONTRACT`), embebida verbatim (bloque delimitado por marcadores `AXIOM:REVIEW-LEDGER-CONTRACT`) en el agent catalog `axiom.spec/target-axiom-agents/axiom-reviewer.md` y guardada por un test de drift (`packages/document-bootstrap/tests/review-ledger-contract.test.ts`) que falla si la constante y el asset divergen. El agent bootstrap `.claude/agents/axiom-review.md` adopta el mismo flujo. Ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md).