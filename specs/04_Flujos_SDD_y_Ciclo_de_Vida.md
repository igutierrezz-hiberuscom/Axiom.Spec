# 04 Flujos SDD y Ciclo de Vida

## Flujo base de este workspace (dogfooding, vigente)

1. especificar en `Axiom.Spec/` (specs numeradas, incrementos, bugs);
2. operar el workflow desde `Axiom.SDD/` (`AGENTS.md`: entender → localizar/crear spec → refinar con goal/scope/non-goals/acceptance criteria → implementar → validar → revisar → cerrar → integrar conocimiento estable en la spec);
3. implementar en `Axiom/`;
4. validar resultado (orden de descubrimiento: README → package scripts → task runners → test configs → build configs);
5. cerrar y reintegrar conocimiento en la spec. Un incremento/bug solo puede marcarse `closed` si: el objetivo es claro, hay acceptance criteria, se implementó (o hay justificación explícita de no-code), se corrió la validación disponible, se revisó contra el intent original y se integró conocimiento estable. Si falta algo, queda `status: pending` con motivo explícito.

La integración estable de este workspace se enruta a las ocho specs numeradas
`Axiom.Spec/specs/00..08`; no existe un `general-spec.md` alternativo que pueda
actuar como fuente canónica.

### Tooling de orquestación del workspace: `/axiom-autopilot` (INC-20260708-axiom-autopilot-command, INC-20260729-copilot-autopilot-reconciliation)

Nota de tooling de desarrollo (no funcionalidad de runtime del producto Axiom): el flujo base de arriba se puede conducir de forma **desatendida** sobre una tanda de cambios mediante `/axiom-autopilot`, disponible como comando/skill de Claude Code (`.claude/commands/axiom-autopilot.md` + `.claude/skills/axiom-autopilot.md`) y como custom agent, skill y prompt de GitHub Copilot (`Axiom.SDD/.github/agents/axiom-autopilot.agent.md`, `Axiom.SDD/.github/skills/axiom-autopilot/SKILL.md` y `Axiom.SDD/.github/prompts/axiom-autopilot.prompt.md`). Codifica el playbook manual de orquestación multi-incremento —descomponer una tanda de cambios en incrementos foco, ejecutarlos secuencialmente vía subagentes `axiom-increment`, auto-resolver ambigüedad registrando cada decisión, verificar cada resultado de forma independiente, integrar el conocimiento estable en los ficheros canónicos `00–08`, reconciliar el contexto técnico en `Axiom.Spec/context/**` y actualizar su fecha de validación, archivar cada incremento y cerrar con un resumen de decisiones— sin detenerse a preguntar y sin ejecutar git mutante. La consolidación es una reconciliación activa: modifica o elimina afirmaciones actuales que hayan quedado obsoletas; `SUPERSEDE` por sí solo no convierte texto antiguo en histórico. Los claims que deban conservarse por trazabilidad se reencuadran bajo una marca histórica explícita. Vive en `.claude/` y `Axiom.SDD/.github/` (configuración para CONSTRUIR Axiom), no en `Axiom/` (runtime del producto); es la contraparte de tanda del comando `axiom-increment` por incremento único, no una capacidad del producto adoptable por proyectos de terceros. La ampliación de contexto técnico quedó consolidada por `INC-20260729-autopilot-technical-context-step` y se portó con reconciliación normativa a Copilot en `INC-20260729-copilot-autopilot-reconciliation`.

## Ciclo de vida real del producto (lo que el runtime ejecuta para un proyecto adoptante)

```
init → join → configure → sync → start → audit → doctor
                                              ↓
                                          upgrade (cuando cambia la versión target)
```

Verificado por el script de smoke `Axiom/scripts/verify-first-project-readiness.mjs` (`npm run readiness:first-project`), que ejecuta exactamente esta secuencia contra un proyecto temporal: `init → configure → toolchain repair → sync → start → audit → doctor`, y falla si algún paso devuelve exit code distinto de 0 o falta un artefacto esperado.

Contrato por comando (lee/escribe) detallado en [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md) y en `context/architecture/`.

### Onboarding del operador mediante launcher y CLI headless

El punto de entrada guiado vigente es `axiom app`, que sirve el launcher local
bajo `/launcher/`. Sus endpoints de install/join y de workspace setup/adopt
ofrecen preview read-only y requieren confirmacion antes de escribir. La CLI
mantiene las rutas headless `axiom init`, `axiom workspace setup` y
`axiom workspace setup --adopt-*`; MCP expone handlers de registry y workflow
sin depender de una interfaz terminal. `axiom` sin subcomando no abre una
superficie implicita y `axiom tui` ya no existe como comando.

Modelo de datos tras el init (fuente única de verdad): `axiom.yaml` es la fuente autoral de la identidad del proyecto (`projectId`/`name`/`repoId`/`role`) y del mapa de repos. `init` escribe `axiom.yaml`, `AGENTS.md` canónico (aditivo, best-effort), `.gitignore`, `.axiom-state/local/` y `.axiom-state/<projectKey>/init.json` (con `profileTriple`+`createdAt`+`version`, sin `projectName` propio); `projectKey` es `projectId` v2 o el slug estable del nombre v1. `init` ya NO escribe `topology.yaml`, que se deriva de `axiom.yaml` al leer y se materializa de forma perezosa solo al asignar roles (`INC-20260703-config-dedup`; ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)). Además intenta registrar el proyecto en el registry user-level de forma best-effort y admite opt-out con `--no-register`. Un `axiom.yaml` v1/v2 con `mode: gateway` o `mode: hybrid` se lee por compatibilidad y se normaliza a `local-only`; esos literales no abren una rama operativa.

## Baseline operativa actual

1. un solo rol funcional cubierto en profundidad (`functionalProfile: builder`);
2. soporte multi-repo dentro de un proyecto vía `@axiom/topology` (`installed-multi-repo` layout), no todavía como separación de repos de Axiom-el-producto;
3. ejecución local (`local-only`) implícita y adapters hacia 8 targets activos, todos con paquete dedicado; `copilot-vscode` sólo se traduce al leer estado legacy persistido y no es target ni superficie materializable.

## Registro histórico: fuera de la baseline inicial del MVP

Según la documentación inicial de readiness, los overlays `standard`/`enterprise` y los providers que entonces se consideraban post-MVP (`engram`, `codegraph`, `graphify`) no eran requisitos de entrada. Este bloque conserva ese contexto histórico y no limita la baseline operativa actual: `engram` está implementado, `cmm` sustituyó a `codegraph`/`graphify`, LiteLLM fue retirado y el registro vigente cubre 8 targets activos. Siguen fuera del alcance inicial los bridges externos/plugins/lanes paralelos avanzados y la instalación user-level del binario como paso obligatorio.

## Punto de partida de un repo de spec recién creado (setup de workspace)

Cuando el setup de workspace multi-repo (`runWorkspaceSetup`) crea desde cero el repo de Spec, lo scaffoldea desde la base canónica de plantillas (`specs/README.md` + `specs/00..08` + `context/TECHNICAL_CONTEXT.md`/`README.md` + los directorios estructurales `specs/{increments,bugs,archive}` y `context/*`) — `INC-20260705-workspace-spec-base`. Así un workspace nuevo arranca ya con la estructura numerada de spec y las carpetas de artefacto donde vive el ciclo de vida de abajo, en vez de una carpeta vacía. Es best-effort y guardado per-file (nunca sobrescribe contenido existente); ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md) y [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

## Ciclo de vida de artefactos (increment/bug/plan/ADR/decision) — roadmap de rediseño, cerrado

Capacidad añadida de forma aditiva por el roadmap de rediseño (23 incrementos, cerrado 2026-07-03); convive con el flujo base de dogfooding de la sección anterior sin reemplazarlo.

1. **Creación**: `axiom-increment/bug/plan/adr/decision create` escribe una carpeta nueva `<specPath>/{increments,bugs,plans,adr,decisions}/<ID>/` con `metadata.yml`, vía las primitivas de `@axiom/workflow`'s `artifact-store.ts`. El ID se genera por sistema (no texto libre).
2. **Refinado/especificación**: `refine`/`specify` actualizan el `metadata.yml` existente; `link-plan`/`link-increment`/`link-bug` establecen relaciones entre artefactos.
3. **Transición de estado**: para `increment`/`bug`/`plan`, el estado (`status: WorkflowState`, 9 valores) es dirigido por la máquina de estados de `workflow-state.json` — pero esa máquina es UN registro singleton por `WorkflowId` (tipo de workflow), no por instancia de artefacto; `metadata.yml` (identidad de instancia) y `workflow-state.json` (máquina de estados por tipo) son almacenes independientes. Para `adr`/`decision`, el estado sigue su propio vocabulario no dirigido por máquina de estados (`AdrStatus`/`DecisionStatus`, ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md)) y se escribe directamente en `metadata.yml`, sin pasar por `workflow-state.json`.
4. **Supersesión de ADR**: `axiom-adr supersede <old-id> <new-id>` es la única transición especial — actualiza ambos ADR atómicamente; Decision no tiene equivalente (sin cadena de supersesión en su schema).
5. **Cierre**: sigue las mismas reglas de cierre que el flujo base de dogfooding — `closed` solo si objetivo claro, acceptance criteria, implementación o justificación no-code, validación ejecutada, revisión contra intent, y conocimiento estable integrado.
6. **Archivado físico y coordinado (`INC-20260710-lifecycle-correctness-fixes`, ACC-041)**: `axiom-increment archive` / `axiom-bug archive` no sólo escriben `status: archived` en `metadata.yml` — `runGovernedTransition` resuelve legalidad, preview, confirmación y gate QA antes de mutar, coordina metadata y efectos locales declarados compatibles, mueve físicamente (rename atómico) la carpeta de la instancia de `<specPath>/{increments,bugs}/<ID>/` a `<specPath>/{increments,bugs}/_archive/<ID>/` (`archiveArtifactDir`, `@axiom/workflow`'s `artifact-store.ts`), y persiste `workflow-state.json` al final. Nunca sobreescribe: si ya existe una carpeta archivada con el mismo ID, la operación falla con un mensaje claro en vez de clobberear. Ante un error del efecto, move o persistencia de state, el runner restaura los snapshots y el move cuando puede; si la recuperación no es completa, devuelve una inconsistencia explícita. No existe éxito parcial silencioso. `listArtifacts`/`axiom-increment list` sólo escanea el nivel directo de `<kindFolder>/`, así que un artefacto archivado deja de aparecer en el listado por defecto tras esta relocación — comportamiento esperado, consistente con la convención `_archive/` ya usada por el propio repo de spec (`specs/increments/_archive/`).

## Flujos de bootstrap

Dos rutas mecánicas de alcance mínimo (no las cadenas literales de 7 subagentes del documento fuente):

1. **`axiom bootstrap from-code --level minimal|basic [--role <role>]`**: analiza el repo actual (`detectStacks`, `buildRepoMap`, `detectCommands`), redacta documentos de contexto técnico con banner `<!-- AXIOM:DRAFT -->`, y puebla `TechnicalContextIndex.available` (nunca `mandatory.*`, que queda para curación humana). Nivel 2 (Standard) en adelante queda diferido — requiere comprensión arquitectónica/de negocio genuina.
2. **`axiom bootstrap from-legacy-sdd <path> [--dry-run]`**: escanea un repo SDD/spec legado (carpetas `increments/`/`bugs/`/`plans/`/`adr/`/`decisions/`), migra cada entrada como artefacto nuevo vía las primitivas de `@axiom/workflow` (nunca sobrescribe, nunca aborta el lote ante colisión), con banner de procedencia `<!-- AXIOM:MIGRATED -->`. `--dry-run` no escribe nada. Ninguna de las dos rutas toca `workflow-state.json`.

Ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md) (RF-AXM-021) para el detalle completo de ambas rutas.

### Versionado de herramientas externas: `plan` y `upgrade`

El versionado de toolchain es un flujo project-scoped distinto de `axiom upgrade`, que continúa gestionando migraciones de `ManagedState` del runtime:

1. El operador declara el subset de tools en `axiom.config/toolchain.yaml`; el catálogo global `axiom.config/toolchain-catalog.yaml` aporta las versiones de política, los canales y los extractores.
2. `axiom toolchain plan --channel <stable|candidate|edge>` compara el lockfile local con el canal solicitado y muestra el diff sin mutar. `--id <id>` repetido limita la operación; el catálogo por sí solo no ordena instalar todas sus entradas.
3. `axiom toolchain upgrade` funciona como preview por defecto. Solo `--yes` persiste la actualización en `.axiom-state/<projectKey>/toolchain.lock`; `--dry-run` fuerza explícitamente la vista previa.
4. La persistencia se protege con checkpoint y rollback. Si la escritura o la comprobación posterior por probe falla, el estado previo se conserva; no se ejecuta instalación, descarga ni sustitución de binarios externos.
5. `axiom doctor` ejecuta TC-020, TC-021 y TC-023 en su recorrido síncrono y deja TC-022 como indicación de que hace falta probe real. `axiom doctor --deep` sustituye esa indicación por la comparación instalada-versus-locked, manteniendo la regla never-fail de los probes.

## Flujos operativos de `configure`/`upgrade`/`repair` sobre instalaciones existentes

- `axiom configure`: re-aplica el perfil persistido completo (single-shot, sin flags incrementales). Al leer un `init.json` histórico, y solo en ese borde interno, migra `profileTriple.adapterTarget: "copilot-vscode"` a `github-copilot` y persiste la normalización antes de installer o dispatcher; el alias no es aceptado por la CLI ni por otras APIs públicas. `configure` no cubre añadir/quitar repo, rol, adapter o tool/MCP — ver el hueco de 7 operaciones (NFR-AXM-015) documentado en [02_Requisitos_No_Funcionales.md](02_Requisitos_No_Funcionales.md). Desde `INC-20260708-incremental-operations` las 4 operaciones ADD (`axiom repo/adapter/provider/role add`, idempotentes y no-clobber) cubren la mitad aditiva de ese hueco; los REMOVE quedan diferidos.
- `axiom upgrade`: calcula y aplica migraciones de `ManagedState` con checkpoint rollback-first; soporta `--dry-run`/`--from-checkpoint`/`--target-version`.
- `axiom repair`: ejecuta `axiom doctor`, agrupa hallazgos por categoría, y despacha las 4 categorías conocidas-como-corregibles (`install-profiles`, `artifact-index`, `toolchain`, `memory`) a las funciones de reparación ya existentes; el resto se reporta como no auto-corregible. Soporta `--dry-run`.
- Los tres flujos anteriores tienen superficies CLI/launcher confirm-gated;
  `repair` y `upgrade` exigen preview dry-run y confirmacion antes de mutar el
  filesystem. La antigua interfaz TUI que los presentaba se conserva solo en
  el historial de incrementos archivados.

Ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md) (RF-AXM-022) y [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md) para el detalle de datos persistidos por cada operación.

## Aprendizaje continuo (captura al cierre, recall al inicio del trabajo) — INC-20260708-continuous-learning

El ciclo de vida gana una capa de aprendizaje continuo MÍNIMA y CONCRETA, anclada en sistemas reales ya entregados (la memoria engram-backed de `@axiom/memory`, ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md), y los registros reales de audit-trail) — no un "motor de instintos" especulativo. Se apoya en tres funciones deterministas de `@axiom/memory` (`extractLessons`/`persistLessons`/`recallLessons`) y se surface en el flujo por dos puntos:

- **Recall al inicio del trabajo**: `axiom context status` (el comando existente que un agente/operador corre para ver el estado del proyecto) añade un bloque best-effort de "lecciones recientes" (las lecciones más relevantes vía `resolveMemoryBackend` + `recallLessons`), que nunca falla el comando si la memoria está vacía o no disponible.
- **Captura explícita**: `axiom learn capture [--from-audit] [--text "..."]` persiste una lección (como `MemoryEntry` `kind: 'pattern'` tag `'lesson'`, topic-keyed para upsert-in-place); `axiom learn list` la recupera. La captura es SIEMPRE explícita (nunca un efecto lateral silencioso de otro comando). Los hooks de sesión (SessionStart/SessionStop) quedan como un snippet `.claude/settings.json` documentado y OPT-IN — Axiom no ejecuta ningún motor de hooks de sesión ni auto-aplica nada. Ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md).

## Señal de delegación no-bloqueante en el ciclo — INC-20260708-delegation-triggers

`axiom context status` surface además un bloque best-effort de `delegationSuggestions` (mismo contrato de degradación que el bloque de lecciones): lee un `session-metrics.json` OPCIONAL en `.axiom-state/<projectKey>/` y, si está presente y cruza umbral, sugiere delegar a un agent del roster curado. Es una señal de sugerencia PURA y estructuralmente no-bloqueante (nunca en el control flow del state machine) — el detalle del evaluador y del roster vive en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

## Revisión con ledger de hallazgos (loop-until-dry + re-review scoped) — INC-20260709-review-findings-ledger

El paso de REVISIÓN del ciclo de vida (el reviewer de producto `axiom-reviewer` y el agent bootstrap `axiom-review` de este workspace) adopta el contrato **review-findings-ledger** (adoptado de gentle-ai, adaptado a Axiom — no copiado literal):

- **Primera pasada exhaustiva (loop-until-dry)**: en vez de una única lectura, el reviewer barre el scope repetidamente hasta que N barridos consecutivos no arrojen hallazgos NUEVOS (default N=2; una lente readability-only puede usar N=1; techo duro de 4 barridos, el loop es finito). Una re-review usa N=1.
- **Ledger de hallazgos persistido**: cada hallazgo se registra con `id` (`{LENS}-{NNN}`, p. ej. `REVIEW-001`), `lens`, `location` (`file:line`), `severity` (BLOCKER|CRITICAL|WARNING|SUGGESTION), `status` (open|fixed|verified|wont-fix|info) y `evidence`. Si la primera pasada no encuentra nada, se persiste un registro de ledger VACÍO (no se salta la persistencia).
- **Persistencia que honra el artifact store**: si existe la carpeta de artefacto en `Axiom.Spec` para el cambio bajo revisión (`.../increments/<INC-id>/` o `.../bugs/<BUG-id>/`, modelo folder-per-artifact de `@axiom/workflow`, ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)), el ledger se escribe como `review-ledger.md` dentro de esa carpeta; si no, se upsertea un topic de Engram (`topicKey` `sdd/<change>/review-ledger`, reusando el backend de memoria engram-backed de `@axiom/memory`); si ninguno está disponible, queda in-context (sin escribir fichero ni topic, completando review → fix → re-review en la misma sesión).
- **Re-review scoped**: una re-review toma el ledger persistido + el diff del fix, verifica la resolución de cada hallazgo del ledger y revisa SOLO las líneas tocadas por el fix (nunca re-lee el diff original completo). Un hallazgo sobre una línea NO tocada se registra con `status: info` (señal de calidad de la primera pasada) y NO dispara por sí solo una ronda nueva.

El contrato se bundlea como UNA constante TS canónica (`@axiom/document-bootstrap`'s `review-ledger-contract.ts`: `REVIEW_LEDGER_CONTRACT`), embebida verbatim (bloque delimitado por marcadores `AXIOM:REVIEW-LEDGER-CONTRACT`) en el agent catalog `axiom.spec/target-axiom-agents/axiom-reviewer.md` y guardada por un test de drift (`packages/document-bootstrap/tests/review-ledger-contract.test.ts`) que falla si la constante y el asset divergen. El agent bootstrap `.claude/agents/axiom-review.md` adopta el mismo flujo. Ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md).

## Repo-affinity y review por rol (2026-07-11) — INC-20260711-repo-affinity-guard / INC-20260711-per-role-review

Dos endurecimientos del ciclo de vida para workspaces multi-repo con roles↔repos definidos; ambos son un **NO-OP estricto** fuera de ese caso (single-repo, `axiom.yaml` schemaVersion 1, o sin asignaciones — el dogfood de Axiom incluido), así que no afectan al flujo por defecto.

### Repo-affinity de los comandos de ciclo de vida (INC-20260711-repo-affinity-guard)

Un guard compartido `checkRepoAffinity` (mismo patrón que `checkPlanIsApproved`) cableado en los cuatro entrypoints de ciclo de vida enforce DESDE QUÉ repo se ejecuta cada comando:

- `axiom-increment` / `axiom-bug` / `axiom-plan` deben ejecutarse desde el **repo de SPEC**; desde el repo de control (`sdd`) o un repo de rol/código se rechazan (exit 1, mensaje accionable que nombra el repo correcto).
- `axiom-role` para el rol **X** (`start`/`apply`/`complete`/`sync-graph`) debe ejecutarse SOLO desde el repo asignado al rol **X**; abrir el repo de otro rol se rechaza nombrando el repo correcto.
- **Condiciones de gating (las tres deben cumplirse; si no, NO-OP):** `loadTopology(repoActual)` OK y `mode === 'multi-repo'`; `resolveProject` resuelto con `role` no vacío (i.e. `axiom.yaml` schemaVersion 2); `assignments.length > 0`. El rol destino de `axiom-role` se deriva del `--slug`/`--id` operado contra `topology.yaml#roles`/`#assignments`.

Depende de que `topology.yaml` esté materializado en CADA repo (control + spec + cada rol), anclado per-repo, para que `loadTopology(repoActual)` responda "cuál es mi rol / qué repo dueña el rol X" desde cualquier repo — ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md). La identidad de repo (`role`/`repoId`) surface como campos aditivos de `ProjectResolution`.

### Review de write-scope por rol y agregado (INC-20260711-per-role-review)

La revisión de write-scope pasa de estar conceptualmente solo en el archive-time desde el spec a dos superficies complementarias que comparten la MISMA primitiva `validateWriteScope` (`@axiom/workflow`) y los helpers de change-set de `@axiom/doctor` (sin lógica duplicada):

- **Review por rol en `axiom-role complete`** (gate explícito dentro de `runRoleSubcommand`, no un hook): valida el `git diff` real del repo de rol contra el `allowedWriteScope` del plan para ESE repo y **bloquea la completion** (exit 1, el estado sigue `in-progress`) ante cualquier violación, salvo `--no-review`/`--force`. Degrada a un SKIP explícito (la completion procede) cuando el repo no es identificable, no hay plan aprobado local, o el plan no tiene scope para este repo — nunca crashea. Resuelve el plan solo del contexto local del repo de rol; el estado cross-repo del plan queda como follow-up (Q-001).
- **Validación agregada desde el spec**: `axiom validate changes --plan <id> --all-repos` resuelve CADA `targetRepo` a ruta absoluta vía `LocalBindings`, diffea y valida cada uno, y emite un reporte consolidado per-repo (✓/✗) + repos NO RESUELTOS explícitos + resultado global (exit 1 ante cualquier violación o repo no resuelto). `buildRepoChangeSets` (`@axiom/doctor`) ahora resuelve rutas bindings-preferred (cierra el gap P1, back-compat).

El paso de archive / `WS-001` de doctor preexistente se mantiene como red de seguridad; estas dos superficies añaden puntos de validación más tempranos (local) y más amplios (multi-repo). Superficie de comandos en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md).

## Generador canónico, subcomandos de ciclo y plano de control MCP (2026-07-11) — tanda sdd-launcher-port

El ciclo de vida gana una **ruta de estructura scriptada** (la IA solo rellena prosa, nunca inventa estructura) y un **plano de control cross-repo**, portados del sdd-launcher de KVP25 sobre `@axiom/workflow` sin reescribir la máquina de estados existente:

- **Generador canónico (P0) + subcomandos de ciclo (P1, `INC-20260711-sdd-launcher-p1-cli-subcommands`)**: `axiom scaffold increment|bug|plan` emite el esqueleto completo desde `Axiom.Spec/templates/*` (delega en el generador P0, sin duplicar); el generador prioriza el directorio `templates/` del scope del proyecto y usa el contenido bundleado como fallback, escribe el árbol por archivo con no-clobber y deja `metadata.yml` bajo la responsabilidad del artifact store; `axiom normalize` canonicaliza el `status` de forma idempotente contra la tabla de vocabulario de ciclo de vida; `axiom integrate` archiva + aplica la transición terminal (reusa `archiveArtifactDir`); `axiom validate transition` rechaza transiciones ilegales con el error tipado `invalid-transition`; `axiom state` inspecciona estado actual / disponibles / recomendado. Superficie en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md).
- **Efectos por transición declarados (P0)**: cada transición del grafo declara su mutación de YAML local y su llamada opcional al tracker (`{ localYaml, tracker }` + runner `transition-effects.ts`), sacando la lógica de lockstep de los wrappers de CLI al grafo; la variante `script/action` se entrega en `INC-20260711-git-services` (ver la sección de abajo y [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md)).
- **Plano de control cross-repo (PX, `INC-20260711-cross-repo-mcp-wiring`)**: el gate de `axiom-role start` y el review de write-scope por rol leen el estado/`allowedWriteScope` aprobado del plan DESDE EL REPO DE SPEC (lectura directa de bindings como vía primaria de la CLI; las tools MCP `spec.planRead`/`sdd.allowedWriteScopeRead` cubren clientes externos; se preserva el fallback local-only). La nueva tool MCP de ACCIÓN `sdd.transitionApply` aplica una transición detrás de un gate `confirmed` (sin confirmar → preview; ilegal → error tipado; project-pinned): MCP pasa a ser un plano de control bidireccional, no solo lectura (ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md)).

## Verificación funcional en `verify` y efectos git por transición (2026-07-11) — INC-20260711-functional-verify / INC-20260711-git-services

Dos endurecimientos aditivos del ciclo de vida (no-op cuando no aplican; la suite existente queda verde):

- **`verify` ahora corre validación funcional (INC-20260711-functional-verify)**: `axiom-increment verify`, `axiom-role complete` y `axiom-qa-e2e verify --run-validation` dejan de ser una transición de estado pura — DESCUBREN y EJECUTAN la validación propia del repo destino (test/build/lint/typecheck) vía un `ValidationRunner` inyectable y BLOQUEAN la transición ante fallo (exit 1, nombra el comando que falló). El orden de descubrimiento replica AGENTS.md (pistas de README → scripts de package → Makefile/task runner); si no se descubre ningún comando se emite la sentencia best-effort exacta de AGENTS.md y se trata como paso no bloqueante. `--no-verify`/`--force` saltan el gate; `--preview`/`--dry-run` reporta lo descubierto sin ejecutar. En multi-repo agrega por repo (los repos no resueltos se reportan, nunca se saltan en silencio). Superficie en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md).
- **Efectos git por transición (INC-20260711-git-services)**: la variante `script/action` de los efectos declarados por transición (§ arriba) ya está entregada — un runner puro de acciones despacha `NamedAction`s a un registro de handlers, con los servicios git (`role-branch`/`commit-sync`, en `@axiom/workflow` detrás de un `GitRunner` inyectable) como primeros handlers. Son LOCAL-ONLY (nunca push por defecto), con guard anti path-escape y confirm-gated. Se cablean OPT-IN en `axiom-role start` (`--create-branch`/`--commit`/`--push`/`--confirm`); el comportamiento por defecto no cambia. Superficie CLI/MCP en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md).

## Adopción de un proyecto preexistente en install-time (2026-07-11) — tanda migración

Un proyecto que precede a Axiom puede ADOPTARSE al instalar, sin recrear su historia: un **repo de spec foráneo** (OpenSpec, `docs/adr`, carpetas alternativas) se convierte en artefactos canónicos `specs/{increments,bugs,…}/<id>/` y su **contexto existente** (`ARCHITECTURE.md`/`docs/**`/ADRs) se ingiere a `technical-context/*` MCP-queryable, todo idempotente (una segunda corrida no duplica). Es aditivo sobre el ciclo de vida: los artefactos adoptados aterrizan en estado seguro (draft/proposed) y requieren revisión humana como cualquier artefacto nuevo. Superficie de comandos en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md); detalle de capacidad en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

## Modelo operativo por-adapter/rol (north-star, 2026-07-13) — tanda fixes + north-star

El ciclo de vida se materializa como **tres flujos operativos por rol**, cada uno respaldado por un skill/agent de proceso (el HOW) del catálogo, **generado por rol y por adapter** en cada repo del workspace por `axiom workspace setup` / `adopt` (ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md), `INC-20260713-ns1-process-skills` / `INC-20260713-ns2-adapter-generation`):

- **ANALISTA** (`axiom-spec-author`, en el **repo de spec**): descubre antes de escribir, cierra ambigüedades y autora contenido de spec en estado seguro (draft); toma el HOW del broker MCP único (`axiom-mcp-broker`, `sdd.skillIndexRead`) y expande el increment/bug vía las lecturas `spec.*Read` + `sdd.transitionApply` confirm-gated.
- **ARQUITECTO** (`axiom-role-planner`, en el **repo de spec**): lee spec + contexto técnico y emite **un plan por rol registrado** con repos destino y `allowedWriteScope`, usando el broker MCP único (`axiom-mcp-broker`) + code-intel (`cmm`/`serena`) con **fallback físico** (Read/Grep) cuando el code-intel MCP no está disponible.
- **IMPLEMENTADOR** (`axiom-role-implementer`, en el **repo de código del rol**): consume el bundle WHAT vía `spec.implementationContextRead`, carga las **skills repo-local**, usa code-intel con fallback físico, se mantiene dentro del write-scope, valida y conduce git confirm-gated.

La **ruta de spec-scope converge** en los tres flujos (`INC-20260713-fix-spec-scope-path` / `INC-20260713-fix-impl-context-path`): en un **repo de spec dedicado** (`axiom.yaml role === 'spec'`) los artefactos viven en la RAÍZ del scope (`<scope>/increments|plans|bugs`, sin segmento `axiom.spec`) y son MCP-queryable; el resto (single-repo / self-hosted / co-located) mantiene `axiom.spec` sin cambios. El helper puro `resolveSpecRelPathForScope(role)` (`@axiom/workflow`) es la única fuente de verdad de esa decisión, y `spec.implementationContextRead` ahora puebla `plan`/`relatedSpec`/`allowedWriteScope` sobre un repo de spec dedicado. Detalle de capacidad en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

## Runtime de versionado: rollback invocable y upgrade cross-repo (2026-07-14) — tanda post-KVP25

El flujo de actualización del runtime (`axiom upgrade`) es **rollback-first** (checkpoint pre-upgrade → migraciones → persistencia; restore automático ante fallo). Dos capacidades de operador cierran las brechas del test KVP25:

- **Rollback invocable por operador** (`INC-20260714-op-rollback-restore`): `axiom rollback <checkpointId>` restaura el `ManagedState` desde un checkpoint conocido (antes el restore sólo ocurría automático ante fallo; `--from-checkpoint` sólo migra hacia adelante). Acción de recuperación no gateada; `--dry-run`/`--list` para inspeccionar. Superficie en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md).
- **Fan-out cross-repo** (`INC-20260714-cross-repo-upgrade-fanout`): en un proyecto **multi-repo**, `axiom upgrade` desde el repo de control migra/checkpointea **todos** los repos de la topología (cada uno con su propio `ManagedState`/checkpoint bajo su `.axiom-state`), con `sync`+`doctor` una sola vez a nivel workspace y reporte por-repo. `--repo-only` conserva el modo per-repo; single-repo byte-idéntico.

## Onboarding guiado desde el launcher (2026-07-15) — tanda INC-20260715-*

El ciclo de vida ahora tiene una **puerta de entrada visual** que no requiere
terminal (`INC-20260715-launcher-onboarding`): desde el launcher web, un
miembro instala Axiom en un proyecto nuevo, se une a uno existente, configura
workspace, adopta spec/contexto y registra roles. Todas las acciones son
confirm-gated (preview→confirmar), best-effort y reutilizan los mismos
run-functions del CLI. Antes de lanzar cualquier acción, el launcher corre el
doctor del proyecto y muestra lo que falta; también presenta el prompt
pregenerado para el adapter seleccionado. Superficies en
[05_Interfaces_Operativas.md](05_Interfaces_Operativas.md); guía de usuario en
[manuales/11_Launcher_Visual.md](manuales/11_Launcher_Visual.md).

## Ciclo con gates de calidad instaladas (2026-07-15) — tanda INC-20260715-*

El ciclo instalado (no sólo el dogfooding) incorpora ahora las puertas de calidad que antes faltaban, todas de solo lectura y opcionales/no bloqueantes salvo donde el proyecto las exija:

1. **Analista** (`axiom-spec-author`) redacta la spec + checklist `CF-xx` → **revisión de spec** (`axiom-phase-reviewer`, lente spec, OK/KO).
2. **Arquitecto** (`axiom-role-planner`) con **análisis de alcance opcional** (por dimensión backend/frontend/qa/transversal según señales de complejidad; `INC-20260715-planner-analysis-fanout`) → **revisión de plan** (phase-reviewer, lente plan).
3. **Implementador** (`axiom-role-implementer`) dentro del `allowedWriteScope` → **revisión de código** (phase-reviewer, lente code) + **QA-validation** (`axiom-qa-validator`: plan de pruebas desde criterios, 1:N, `SIN COBERTURA`) + **revisión de seguridad** opcional (`axiom-security-reviewer`).
4. **Cierre**: `axiom-tech-context` actualiza contexto técnico y detecta spec-drift; `axiom-spec-integrator` consolida el conocimiento estable en la spec canónica y **archiva** el incremento (atómico, confirm-gated).

## Ciclo de vida del modelo `<project>.axiom` y ejecución en worktree (2026-07-24) — tanda INC-20260724-*

Flujos nuevos de la graduación a *full product lifecycle* (modelo de repo único, worktrees, provenance/rollback). Solo aplican a proyectos **instalados**; los repos propios de Axiom no se migran. Formas de datos en [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md); superficies en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md).

### Adopción → crea `<project>.axiom`, legacy intacto (INC-20260724-adopt-creates-axiom-repo)

`axiom workspace setup --adopt-spec/--adopt-sdd` crea un repo hermano nuevo `../${projectName}.axiom` y migra la spec/context legacy DENTRO de él (formato Axiom); los repos legacy quedan **byte-for-byte intactos** (fuentes read-only registradas en `legacyRepos[]`). El writer de migración y los readers de CLI convergen en la raíz del repo nuevo (sin anidamiento `axiom.spec/`). `--dry-run` = cero escritura; re-run idempotente.

### Transición de lifecycle al editar un migrado (INC-20260724-provenance-lifecycle-manifest)

Cualquier escritura de Axiom sobre un artefacto cuyo `lifecycle.state` es `migrated` lo transiciona automáticamente a `migrated-and-modified` en el choke point `saveArtifactMetadata` (sin comando aparte). Cada run de migración que crea artefactos nuevos actualiza el manifest global `<project>.axiom/migration/migration-manifest.yaml`. Esto es lo que después habilita la salida limpia de Axiom.

### Salida de Axiom: `axiom eject` (INC-20260724-export-eject-rollback)

Como el legacy está intacto, "rollback" desde Axiom NO significa restaurar legacy: significa dejar de usar Axiom llevándote lo que Axiom creó. `axiom eject` selecciona los artefactos rollback-eligible (`axiom-native` + `migrated-and-modified`, los 5 kinds), excluye los `migrated` sin modificar y toda superficie interna (config/state/skills/…), y los vuelca a `<project>.axiom/exports/<exportId>/` con `EXPORT_REPORT.md` + `export-manifest.yaml`. Default `--dry-run` (cero escritura); `--write-export-folder` escribe. **Nunca** escribe en legacy. Distinto del `axiom rollback` preexistente (restore de checkpoint de upgrade) — sin colisión ni código compartido.

### Ejecución en worktree como modo seleccionable (INC-20260724-worktree-mode-selection + W1/W2/W3/W5)

Un incremento/bug/plan puede ejecutarse **in-place** (comportamiento actual, default) o en un **git worktree** dedicado. El default lo elige el arquitecto en la instalación (`executionMode` en `install-profile.json`) y es overridable por run. `axiom-role start` resuelve el modo (install default → override `--worktree`/`--in-place`) vía `runRoleSubcommandAsync` (el `runRoleSubcommand` síncrono queda byte-idéntico para todos los callers existentes: MCP `sdd.transitionApply`, `app-api`, etc.). Flujo worktree:

1. **start**: nombre de rama parametrizado (`role/<id>-<slug>` por defecto, igual que in-place) + path de worktree único por ejecución (bajo `<repoBasename>.worktrees/`) → `worktreeAdd` (`@axiom/workflow`, INC-20260724-git-worktree-services: dry-run/preview, guards de path fuera-del-repo, nunca el repo principal, rechaza worktree sucio salvo `force`, local-only) + `ExecutionStore.create` (compose helper `createExecutionForWorktree`) → `provisionWorktreeExecution` (INC-20260724-worktree-provisioning).
2. **provisioning**: materializa en el worktree la superficie `.axiom` portable + config MCP del broker unificado `axiom` + config code-intel por-worktree (cmm/serena apuntados al worktree) + el layout `.axiom-state` execution-scoped. Best-effort, no-clobber, created-gated. Portable-only: solo se copian 3 ficheros no-secretos (`init.json`/`install-profile.json`/`workspace.json`); **secretos y `.axiom-state/local/` nunca se copian**.
3. **complete**: corre las gates y luego `harvestAndCleanupExecution` (INC-20260724-worktree-harvest-cleanup) en orden estricto **kill → harvest → teardown → remove**: matar procesos rastreados (best-effort) → copiar logs/evidence/outputs a la ubicación central que sobrevive al borrado → `teardownWorktreeCodeIntel` (borra `.cmm`/`.serena` derivados) → `worktreeRemove`. Un worktree con trabajo real sin commitear es **hard stop** (nunca `force` por defecto); harvest siempre precede a cualquier borrado; `dryRun` no muta nada.

### Correctitud del cierre en worktree (INC-20260724-worktree-close-correctness)

Fix fast-follow de 2 defectos HIGH que el barrido transversal encontró y reprodujo:

- **FIX 1**: el provisioning reescribe legítimamente ficheros *tracked* del worktree (config MCP nativa, superficie de adapters), lo que hacía que el guard de "worktree sucio" bloqueara todo cierre normal. Ahora el provisioning registra en `Execution.provisionedPaths` exactamente lo que escribió, y el cierre lo **neutraliza** (`resetWorktreeGeneratedFiles`: revierte tracked vía `git checkout --`, borra untracked) justo antes del dirty check — así un worktree normal cierra limpio SIN force, mientras que trabajo genuino (cualquier path fuera de ese set) **sigue bloqueando**. Si el cierre aún hace hard-stop, un **rollback compensatorio** revierte la transición ya persistida `archived → in-progress` (evita el estado archived+huérfano); la `Execution` queda en `harvested` para reintentar.
- **FIX 2**: en modo worktree las gates de verify/review ahora apuntan a `execution.worktreePath` (no a `projectRoot`) — un worktree cuyos tests fallan ya no pasa la gate por validar el repo fuente.

### Freshness de artefactos SDD (INC-20260724-sdd-artifact-freshness)

Como los worktrees comparten el repo `<project>.axiom` central, al leer/editar un increment/bug/plan se hace un **auto-fetch de solo lectura, best-effort y time-bounded** (acotado a la rama upstream del artefacto, nunca `--all`) y se compara → `fresh | stale | unknown`. Si está `stale` (el remoto tiene commits más nuevos tocando esa carpeta) se emite un **warning** `stale-artifact` — nunca bloquea, nunca da error (todo fallo degrada a `unknown` sin warning). Se superficie en las lecturas MCP (`spec.incrementRead`/`bugRead`/`planRead`) y en la escritura (`sdd.gitCommitSync`). El push va **acotado** a la carpeta del artefacto (`git add -- <paths>`, nunca `-A`). Sin hook git obligatorio (auto-fetch on-demand en read/edit). GIT-level, distinto de la freshness de índices de code-intel (INC-20260724-worktree-provider-isolation).

### Barrido e2e transversal + revisión adversarial (INC-20260724-cross-cutting-e2e-review)

Cierre del batch: una suite e2e encadena la mayor parte del flujo (adopción → `<project>.axiom` → MCP unificado → worktree start/close → eject dry-run → skills RTK/concisión → AutoSkills) y una revisión adversarial busca defectos de integración cruzada. Encontró y reprodujo los 2 defectos HIGH de cierre en worktree, corregidos en INC-20260724-worktree-close-correctness. No cambió código de producto.

Las disciplinas transversales (`axiom-structured-doubts`, `axiom-functional-checklist-coverage`, `axiom-plan-drift-alignment`, `axiom-role-close-doc`) atraviesan todas las fases. Detalle de superficies por rol en [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md) y [manuales/13_Skills_Agentes_y_Roles.md](manuales/13_Skills_Agentes_y_Roles.md).

## Memoria viva entre fases y Knowledge Harvest (2026-07-29) — tanda INC-20260729-knowledge-*

El ciclo de vida gana una capa de **memoria viva compartida entre fases** usando Engram como backend (`@axiom/memory` + `engram-backend.ts`, ya existente). Cada fase guarda conocimiento útil con metadata de fase trazable; las fases posteriores lo consultan antes de actuar. Al archivar el incremento, `axiom knowledge harvest` clasifica las memorias y propone promociones a contexto técnico y skills.

### Flujo de memoria entre fases

```text
Fase A (analysis)
  → resolveMemoryBackend (Engram/JSON)
  → save memoria con metadata: increment, phase=analysis, actorRole=analyst, stability, knowledgeKind
  → la memoria queda persistida localmente (SQLite vía Engram, o JSON fallback)

Fase B (architecture)
  → resolveMemoryBackend
  → query memorias filtradas por increment=<id> y phase=analysis
  → usa lo aprendido por el analista
  → save sus propias memorias con phase=architecture

Fase C (frontend/backend/QA/validator)
  → repiten el patrón: query fases anteriores → actuar → save
```

La memoria compartida se logra a través de la superficie MCP de Engram (mismo proceso local, mismo `--project`), sin requerir sincronización Git de `.engram/` ni base de datos centralizada. El backend JSON de fallback también funciona para proyectos sin Engram instalado.

### Metadata de fase en MemoryEntry

Cada memoria guardada por una fase lleva 7 campos opcionales (`INC-20260729-knowledge-phase-metadata`):

- `increment`: ID del incremento/bug/plan
- `phase`: `analysis` | `architecture` | `frontend` | `backend` | `qa` | `validator` | `archive`
- `actorRole`: `analyst` | `architect` | `frontend` | `backend` | `qa` | `validator` | `orchestrator`
- `knowledgeKind`: `decision` | `constraint` | `discovery` | `bugfix` | `gotcha` | `pattern` | `risk` | `open-question` | `workaround` | `convention`
- `stability`: `temporary` | `candidate-project-context` | `candidate-skill` | `historical-only`
- `visibility`: `project-shared` | `private`
- `sourceArtifact`: path al artefacto fuente

En el backend engram, la metadata se codifica como frontmatter YAML-like al inicio del `content` y se decodifica en lectura vía `mem_get_observation`. El backend in-memory la preserva vía serialización JSON.

### Contrato de memoria por fase

Ver [manuales/13_Skills_Agentes_y_Roles.md](manuales/13_Skills_Agentes_y_Roles.md) §"Contrato de memoria Engram por fase" para las reglas detalladas de qué guardar y qué NO guardar en cada fase.

### Knowledge Harvest al archivar

Al finalizar un incremento, `axiom knowledge harvest --increment <id>` (`INC-20260729-knowledge-harvest-command`):

1. Lee todas las memorias del proyecto activo (`resolveMemoryBackend`)
2. Filtra client-side por `entry.increment === <id>`
3. Clasifica por `stability`:
   - `candidate-project-context` → propuesta de promoción a `context/technical/`
   - `candidate-skill` → propuesta de creación/actualización de skill
   - `temporary` / `historical-only` → solo memoria histórica del incremento
   - Sin metadata de fase → descartada (ruido)
4. Genera `knowledge-harvest.md` en `<specRepo>/specs/increments/<id>/`

El harvest es **read-only**: nunca muta contexto técnico ni skills. La promoción real requiere revisión humana o `axiom knowledge promote` (follow-up diferido). `--dry-run` imprime el reporte en stdout sin escribir archivo.

### Relación con el harvest de worktree

El harvest de worktree (`harvestAndCleanupExecution`, INC-20260724-worktree-harvest-cleanup) copia logs/evidence/outputs del worktree a una ubicación central que sobrevive al borrado. El Knowledge Harvest (`axiom knowledge harvest`) es un concepto distinto: clasifica la memoria semántica del incremento para proponer promociones a contexto técnico y skills. Ambos coexisten sin solaparse: el harvest de worktree es operacional (archivos), el Knowledge Harvest es de conocimiento (memoria).

### Sync de memoria entre miembros del equipo

`axiom knowledge sync` y `axiom knowledge pull` (`INC-20260729-knowledge-sync-command`) permiten compartir la memoria de Engram entre miembros del equipo vía Git en `<project>.axiom`:

**Al cerrar una fase** (`axiom knowledge sync --increment <id> --phase <phase>`):
1. Lee todas las memorias del incremento vía `resolveMemoryBackend` + `queryMemory`
2. Filtra: solo `visibility: 'project-shared'` (o sin visibility = default), sin secretos
3. Serializa como chunk JSON en `.engram/chunks/<hash>.json`
4. Actualiza `.engram/manifest.json` (append-only)
5. `git add .engram/ && git commit -m "knowledge: sync <id> <phase>" && git push`
6. Si no hay memorias nuevas, termina OK sin crear commits vacíos

**Al iniciar una fase** (`axiom knowledge pull --increment <id>`):
1. `git pull --rebase` en `<project>.axiom`
2. Lee `.engram/manifest.json`, filtra chunks no importados
3. Para cada chunk nuevo, lee el JSON e importa cada memoria vía `saveMemory`
4. Actualiza `.engram/.imported` (tracking de chunks ya importados)
5. Si hay conflicto de Git, reporta sin ocultar

El diseño de chunks append-only (nunca se modifican chunks viejos) + `manifest.json` pequeño y mergeable evita conflictos de Git en el caso común. `.engram/engram.db` está gitignored (solo se versionan chunks + manifest).
## Gobierno verificable en el ciclo (2026-08-02) — tanda `INC-20260730-*`

### Receipt desde el boundary de transición gobernada

Cuando un caller habilita `receipt`, `runGovernedTransition` es la única ruta que decide y emite el receipt JSON del artefacto; CLI, launcher y `sdd.transitionApply` MCP llegan al mismo boundary. No hay una segunda capa de wrappers `…Core` encargada de receipts. El `phase` es el nombre real de la transición declarado en `workflows.yaml` (`increment-verify`, `bug-archive`…), sin mapeos inventados a otro vocabulario.

Un preview (`--preview`/`--dry-run`, launcher o MCP) no emite receipt porque no aplica una transición. La escritura es best-effort/no bloqueante: un fallo se reporta como aviso y no altera el resultado ya decidido. En archive, el runner usa la carpeta ya movida a `_archive/<id>/`, de modo que el receipt se co-localiza en el destino archivado y nunca recrea una carpeta fantasma pre-archive.

### Gate de freeze antes del apply

Antes de delegar un apply a un subagente, el orquestador congela el candidate (`axiom freeze --increment <id>`) y verifica con `checkCandidateFreeze` que los inputs no mutaron. Un hash distinto del congelado significa que la memoria del incremento o su `README.md` cambiaron desde la revisión, y el apply no debe delegarse sobre esa base sin re-congelar. Un `candidate-freeze.json` corrupto o truncado devuelve `{ ok: false, reason }` como cualquier otro fallo — no lanza una excepción fuera de la función.

### Evidencia al escribir memoria durante el ciclo

Cualquier fase que persista conocimiento debe aportar `rationale` y `source` (RF-AXM-058). Esto se compone con el contrato de memoria por fase de la tanda `INC-20260729-knowledge-*`: aquélla define **qué** guardar en cada fase; ésta impone que lo guardado venga **justificado y con origen**, y lo hace cumplir en runtime.

## Contratos R-10 de workflow y cierre (2026-08-18)

`axiom.config/workflows.yaml` es la única definición editable del grafo; el build empaqueta ese asset y `resolveWorkflowConfig` lo usa en CLI, launcher, MCP e integrate. Sólo la ausencia del YAML de proyecto elige el default empaquetado; un YAML presente inválido o con schema no soportado falla cerrado. Las acciones alcanzables se derivan del grafo efectivo, incluido el override del launcher.

`runGovernedTransition` es el único límite mutante: calcula legalidad, preview, confirmación, QA, efectos soportados, metadata, archive recuperable, state y receipts habilitados. Un preview no escribe y `--force`/`--no-verify` no sustituyen `confirmed: true` cuando la transición exige aprobación. La aprobación de plan sólo permite `draft → plan-approved` tras validar su metadata; `axiom-role start` requiere state y metadata de ese plan aprobado.

Antes de archivar incrementos o bugs, `QaArchiveDecision` aplica un contrato único. Con `qaLane: parallel`, `pending`, `failed` o `cancelled` permiten continuar con aviso; con carril inline o rol QA requerido, sólo `passed` permite archive y policy/evidencia no evaluable bloquea. El comando `axiom-qa-e2e` registra también el carril inline por el runner común: `start → verify [--run-validation] → pass` produce la evidencia `passed`; sólo entonces el gate inline permite archive.
