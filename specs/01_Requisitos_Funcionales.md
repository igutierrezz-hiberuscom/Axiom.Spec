# 01 Requisitos Funcionales

## Meta-requisitos del workspace de desarrollo (dogfooding)

### RF-AXM-001 Separación por ownership

Axiom debe distinguir claramente el repo de runtime (`Axiom/`), el repo de spec (`Axiom.Spec/`) y el repo SDD (`Axiom.SDD/`). Vigente hoy: `Axiom.SDD/AGENTS.md` fija esta separación como regla operativa activa para construir el propio producto.

### RF-AXM-002 Workflow SDD sobre Axiom

Axiom debe permitir que la especificación, planificación, implementación y validación del propio producto se ejecuten con sus propias reglas operativas (bootstrap lifecycle de `Axiom.SDD/AGENTS.md`: entender → localizar/crear spec → refinar → implementar → validar → revisar → cerrar → integrar conocimiento estable).

### RF-AXM-003 Modelo de topología de repos (multi-repo, dentro del producto)

El runtime debe conocer qué repos forman parte de un proyecto gestionado, cómo se llaman, qué alias usan y dónde se resuelven. Implementado hoy vía `@axiom/topology` (`topology.yaml`: `repoRefs` + `roleAssignments` + QA lanes) y expuesto por `axiom topology`/`axiom repo`.

### RF-AXM-004 Resolución multi-surface

La topología de repos y el proyecto activo deben poder resolverse tanto desde workspace local (`@axiom/filesystem-truth`, `@axiom/project-resolution`) como desde surfaces externas soportadas por adapters. No existe hoy una capa MCP genérica de resolución de proyecto (el roadmap futuro la introduce en `INC-13`/`INC-14`); los MCP actuales se gestionan como catálogo de herramientas vía `@axiom/toolchain` y comandos `axiom mcp`.

### RF-AXM-005 Operación con un único rol

La baseline actual debe ser usable por un único rol (`functionalProfile: builder`) sin impedir que el modelo evolucione a roles múltiples más adelante. Confirmado: `Axiom/docs/first-project-readiness.md` recomienda `builder` como perfil más cubierto en runtime hoy; `product-owner` existe como segundo profile declarado pero con menor cobertura runtime.

## Requisitos funcionales del producto (ciclo de vida real, verificado en código y docs)

### RF-AXM-006 Inicialización de proyecto (`axiom init`)

El CLI debe poder inicializar un proyecto validando el nombre (`^[a-z0-9][a-z0-9-]{0,62}$`), determinando el layout (`self-hosted` vs `installed-multi-repo`), aceptando el rol de este repo vía `--role sdd|spec|code` (default `sdd`, validado contra `REPO_ROLES`), generando `axiom.yaml` con `role`/`repoId: <slug>-<role>` y el profile triple (`functionalProfile` + `operationalOverlay` + `adapterTarget`), creando `.axiom-state/local/` y `.axiom-state/<projectName>/`, y persistiendo `init.json` (solo `profileTriple` + `createdAt` + `version`). Los enums canónicos (`REPO_ROLES`, `PROJECT_LAYOUTS`, `FUNCTIONAL_PROFILES`, `OPERATIONAL_OVERLAYS`, `ADAPTER_TARGETS`) están centralizados en `apps/cli/src/commands/init.ts` como const arrays, fuente única para la validación de `init` y para el wizard de la TUI (RF-AXM-023). `axiom init` YA NO escribe `axiom.config/topology.yaml` (`INC-20260703-config-dedup`; ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)). Fuente: `Axiom/docs/cli/init.md`.

### RF-AXM-007 Registro de miembros (`axiom join`)

El CLI debe registrar miembros del proyecto por id (`user:alice`, `agent:sdd`, etc.) en `.axiom-state/<projectName>/members.yaml`, deduplicando por igualdad exacta. Fuente: `Axiom/docs/cli/join.md`.

### RF-AXM-008 Composición del install profile (`axiom configure`)

El CLI debe leer el profile triple, componer el `ResolvedInstallProfile` real (`@axiom/install-profiles` + `@axiom/installer`) y persistir `install-profile.json`, materializando además surfaces derivadas según el target activo (p. ej. `.github/copilot-instructions.md` vía `@axiom/document-bootstrap`). Fuente: `Axiom/docs/cli/configure.md`.

### RF-AXM-009 Sincronización de adapters (`axiom sync`)

El CLI debe reconciliar los outputs del adapter activo contra el filesystem, validando primero el gate de telemetría del overlay activo; si el gate falla, debe abortar antes de escribir cualquier marker. Fuente: `Axiom/docs/cli/sync.md`.

### RF-AXM-010 Arranque de runtime (`axiom start`)

El CLI debe resolver el modo de discovery (`filesystem` | `gateway`) según el overlay y las flags (`--gateway`/`--no-gateway`), ejecutar un primer ruteo sintético de capability/provider y persistir `last-start.json`. Fuente: `Axiom/docs/cli/start.md`.

### RF-AXM-011 Auditoría de integridad (`axiom audit`)

El CLI debe verificar en modo solo lectura el audit trail del proyecto (hash SHA-256, conteo de líneas, retención, detección de reescritura externa) y reportar uno de tres estados: `compliant`, `absent` o `violation`. Fuente: `Axiom/docs/cli/audit.md`.

### RF-AXM-012 Diagnóstico de salud (`axiom doctor`)

El CLI debe ejecutar checks de boundaries, policies, manifests, aislamiento, modelo de capacidades y gateway, devolviendo un resultado agregable (PASS/FAIL) y soportando salida `--json`. Fuente: `Axiom/docs/cli/doctor.md`, package `@axiom/doctor`.

### RF-AXM-013 Versionado y upgrade (`axiom upgrade`)

El CLI debe calcular migraciones aplicables entre versiones, crear un checkpoint pre-upgrade, aplicar migraciones en orden y hacer rollback automático si alguna falla, con soporte `--dry-run` y `--from-checkpoint`. Fuente: `Axiom/docs/cli/upgrade.md`, package `@axiom/versioning`.

### RF-AXM-014 Model routing por slot (`axiom model`)

El CLI debe permitir ver, fijar, quitar y resetear el modelo asignado a cada slot operativo (`increment`, `bug`, `plan`, `implementation`, `qa-e2e`, `review`, `archive`), con fallback declarado por `SupportLevel` del target (`multi-mode`, `single-mode`, `fallback-only`) y proyección opcional a `.opencode/model-routing.json`. Fuente: `Axiom/docs/cli/model.md`, package `@axiom/model-routing`.

### RF-AXM-015 Gestión de components y skills

El CLI debe exponer catálogo, instalación/desinstalación con checkpoint (`axiom components`) y registro/drift de skills materializadas contra `.opencode/skills-lock.yaml` (`axiom skills`). Fuente: `Axiom/docs/cli/components.md`, `skills.md`.

### RF-AXM-016 Generación de surfaces por adapter target

El runtime debe poder generar configuración específica para al menos 6 targets con adapter dedicado (`opencode`, `claude-code`, `github-copilot`, `vscode`, `cursor`, `litellm`) mediante una función `generate<Target>Config` de firma común (`Promise<Result<GeneratorResult, AdapterGeneratorError>>`), y declarar 3 targets adicionales sin adapter dedicado que caen a `fallback-only` (`copilot-vscode`, `antigravity`, `visual-studio-2026`).

### RF-AXM-017 Superficie de comandos operativos más amplia que la documentada

El código de `apps/cli/src/commands/` contiene 36 ficheros de comando (incluyendo `app`, `app-plugins`, `app-plugins-azure-devops`, `capability`, `context`, `gateway`, `intent`, `mcp`, `memory`, `projects`, `qa-archive-gate`, `repo`, `roles`, `self-update`, `toolchain`, `topology`, `axiom-bug`, `axiom-increment`, `axiom-plan`, `axiom-qa-e2e`, `axiom-role`), de los cuales `Axiom/docs/cli/` solo documenta 9 en profundidad (`init`, `join`, `configure`, `sync`, `start`, `audit`, `upgrade`, `doctor`, `tui`) más `model`, `components` y `skills`. Esta spec no debe asumir que los comandos no documentados están estabilizados solo por existir en código; requieren verificación puntual antes de citarlos como contrato estable.

### RF-AXM-023 Punto de entrada TUI, menú de bootstrap sin proyecto y wizard de `init` guiado

- `axiom` sin subcomando debe abrir la TUI (acción por defecto de Commander en `apps/cli/src/index.ts`, reusando `runTuiCli`). Los subcomandos, `--help` y `--version` matchean antes que la acción por defecto y quedan sin cambios.
- `axiom`/`axiom tui` en una carpeta SIN proyecto Axiom (ni por resolución directa ni por fallback) y con input interactivo (TTY) debe abrir un menú de bootstrap estilizado (screen `setup` de `@axiom/tui`): inicializar Axiom en esta carpeta · actualizar Axiom (self-update) · ver proyectos registrados · salir. Elegir un proyecto registrado hace `chdir` + abre la TUI operativa sobre él. En no-TTY (CI, pipes) debe preservarse el error `projectNotFound` con exit 1.
- Elegir "inicializar" debe lanzar un WIZARD guiado (ver RF-AXM-024 para el comportamiento vigente hoy, que reemplazó el wizard single-repo original de este requisito). El wizard es UI aditiva: no reemplaza a `axiom init` no-interactivo (RF-AXM-006). El detalle de UI, el split de capas y las etiquetas viven en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md) — no se duplican aquí.

### RF-AXM-024 Setup de workspace multi-repo desde el wizard de la TUI (`INC-20260705-*`)

La acción "Inicializar Axiom en esta carpeta" de la screen `setup` debe preparar, en UNA operación, un workspace Axiom multi-repo completo — no solo inicializar el repo actual. Reemplaza el wizard single-repo de 6 pasos de RF-AXM-023 (`INC-20260703-tui-init-wizard`); el comando `axiom init`/`runInit` (RF-AXM-006) sigue siendo single-repo y sin cambios.

- El wizard debe recoger, en orden y con defaults sensatos pre-cargados: `name` (texto libre, del que se deriva el `projectId`) → `sddPath` (default `.`) → `specPath` (default `../<slug>.spec`) → `roles` (multi-select de roles funcionales/de implementación a integrar, opciones tomadas en vivo de `DEFAULT_PROFILES` de `@axiom/install-profiles`, no una lista duplicada) → un path por rol seleccionado (default `../<slug>-<role>`, inyectados dinámicamente) → `profile` → `overlay` → `target` → un único resumen de confirmación.
- Solo al confirmar, el CLI debe ensamblar un `WorkspaceSetupSpec` y llamar a `runWorkspaceSetup` (`INC-20260705-workspace-multirepo-setup-engine`), que scaffoldea/parametriza el repo de control (SDD), el repo de Spec y los N repos de rol; escribe un `axiom.yaml` por repo con un mapa `paths` recíproco y consciente de rol (los repos "se conocen entre sí"); escribe un `axiom.config/topology.yaml` y `.axiom-state/local/topology-bindings.yaml` en el repo de control; y registra TODOS los repos bajo un único proyecto en el registro v2 (`~/.axiom/projects.yml`) en una llamada idempotente (`upsertProjectReposV2`), con una guarda no-clobber que nunca sobrescribe un `axiom.yaml` de otro proyecto. Tras el registro, genera best-effort la config MCP (SDD MCP + Spec MCP, `INC-20260705-workspace-mcp-generation`). El detalle de datos vive en [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md); el de UI y capas en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md); el de MCP en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).
- Cancelar (en cualquier step o en la confirmación) no debe llamar a `runWorkspaceSetup` y debe volver al menú `setup` sin efectos; un fallo de `runWorkspaceSetup` debe capturarse, reportarse y volver al menú `setup` sin crashear la TUI.
- El step `target` (select único) se sustituye por un step `adapters` (multi-select de los 8 `ADAPTER_TARGETS`, default `['opencode']`); los adapters multi-seleccionados se instalan en TODOS los repos del workspace (control + spec + cada rol), su selección se persiste en `<controlRepo>/.axiom-state/<projectId>/workspace.json`, y la proyección MCP cubre cada adapter MCP-capaz seleccionado (`INC-20260705-workspace-adapters-multiselect`, ampliada por `INC-20260705-workspace-adapter-templates` para que opencode/claude-code/copilot-vscode produzcan ficheros de instrucciones reales).
- Cuando el repo SDD/control se crea recién, debe scaffoldearse en él una baseline de skills de Axiom (`axiom.config/skills-catalog.yaml` + `.opencode/agents/<id>/SKILL.md` materializados + un `axiom.config/skills-index/<role>.yaml` por rol funcional declarado), best-effort y solo en el repo de control (`INC-20260705-workspace-sdd-skills`).
- Cuando el repo de Spec se crea recién, debe scaffoldearse en él la base canónica de spec + contexto técnico (`specs/README.md` + `specs/00..08` + `context/TECHNICAL_CONTEXT.md`/`README.md` + los directorios estructurales), desde plantillas bundleadas, guardado per-file sin clobber, best-effort y solo en el repo de spec (`INC-20260705-workspace-spec-base`).
- **Roles custom (round 3, `INC-20260705-workspace-wizard-ux-custom-roles`)**: el step `roles` deja de ser un multi-select fijo (backend/frontend/qa-e2e desde `DEFAULT_PROFILES`) y pasa a texto libre separado por coma, aceptando nombres de rol ARBITRARIOS y cualquier cantidad (incluido cero); se parsean, sanitizan y deduplican, y se reenvían verbatim como `roleKey`/`functionalRoleId` (el motor ya los acepta como `string` abierto). El detalle de UI (echo de captura, defaults de path estilo hermano + rutas absolutas, subtítulos de `profile`/`overlay`) vive en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md).
- **Skills por repo de código (round 3, `INC-20260705-workspace-code-repo-skills`)**: además de la baseline en el repo de control, cada repo de código/rol recién creado debe recibir su propia baseline de skills scoped a su rol (mismo `skills-catalog.yaml` semilla + skills materializadas + UN `skills-index/<roleId>.yaml` scoped a su propio rol, posiblemente custom), best-effort y gateado por el `created` propio de cada repo de rol; el repo de spec no recibe skills.
- **Robustez del install (round 3, `INC-20260705-workspace-setup-registry-robustness`)**: el setup debe completar todo el scaffolding local (config MCP, adapters, `workspace.json`, skills, base de spec) **aunque** el registro en `~/.axiom/projects.yml` falle, el usuario tenga un registry legado v1 sin migrar, o se pase `register: false` — el registro es no bloqueante (registra `registryRegistered: false` + warning, sin early-return). Ante un registry legado v1, el setup lo auto-migra a v2 (`migrateLegacyRegistryV1ToV2`) preservando el fichero legado, y reintenta el registro una vez. Ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md).
- El detalle de las formas de datos de estos artefactos vive en [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md); el de comportamiento/despacho de adapters y skills en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md); el de UI del wizard en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md).

## Requisitos funcionales del roadmap de rediseño (cerrado, ver `00_Resumen_Ejecutivo.md`)

Las siguientes capacidades fueron implementadas en `Axiom/packages/*` por el roadmap de rediseño de 23 incrementos (INC-01 a INC-23, cerrado 2026-07-03). A diferencia de las RF-AXM-001 a 017 (modelo hoy vigente por defecto), estas capacidades conviven de forma aditiva con el modelo anterior — no lo reemplazan salvo donde se indique explícitamente.

### RF-AXM-018 Modelo de artefactos por carpeta (increments, bugs, plans, ADR, decisions)

Cada instancia de incremento/bug/plan/ADR/decisión es una carpeta en `<specPath>/{increments,bugs,plans,adr,decisions}/<ID>/` con al menos un `metadata.yml`, leído/escrito por `Axiom/packages/workflow/src/artifact-store.ts` (`loadArtifactMetadata`/`saveArtifactMetadata`). `<specPath>` resuelve por convención a `<projectRoot>/axiom.spec` (la convención in-repo del producto, distinta de este repo `Axiom.Spec/`).

- `PlanMetadata` declara `targetRepos: string[]`, `taskType: string` y `allowedWriteScope: { repo: string; paths: string[] }[]` (ver RF-AXM-019).
- `listArtifacts(projectRoot, kind)` es la única primitiva de escaneo directo para enumerar instancias de un tipo: tolera una carpeta de tipo ausente (resultado vacío, sin excepción) y un `metadata.yml` individual malformado (se reporta como entrada `failure`, no aborta el resto del escaneo). Todo consumidor (`axiom {increment,bug,plan,adr,decision} list`, `axiom index rebuild`/`validate`, el check `IX-001` de `@axiom/doctor`) llama a esta única función.
- No existe una caché persistente `.axiom/cache/*.index.json`; `axiom index rebuild`/`validate` son wrappers sin caché, de escaneo directo sobre `listArtifacts`, cubriendo los cinco tipos (`increment`, `bug`, `plan`, `adr`, `decision`) en un solo comando con soporte `--json`. Una caché persistente queda diferida hasta que una necesidad de rendimiento medida o un consumidor concreto (p. ej. una futura herramienta MCP) la justifique.
- `BaseArtifactMetadataFields` (`id`, `kind`, `title`, `createdAt`, `updatedAt`, `externalRefs`) es la base común. `IncrementMetadata`/`BugMetadata`/`PlanMetadata` la extienden con `status: WorkflowState` (vocabulario de 9 valores dirigido por máquina de estados). `AdrMetadata`/`DecisionMetadata` la extienden con vocabularios de estado propios y separados (`AdrStatus`: `proposed | accepted | superseded | rejected`; `DecisionStatus`: `proposed | accepted | rejected`, sin `superseded`) — `parseStatusForKind` en `artifact-store.ts` los valida como conjuntos cerrados y no solapados.
- `externalRefs` es un mecanismo agnóstico de proveedor (`artifact-store.ts` + `apps/cli/src/commands/artifact-metadata-cli.ts`'s `externalRefs add|list`), disponible en todo tipo de artefacto. No debe confundirse con el flag de UI `field.externalRef?: boolean` del plugin de Azure DevOps (ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md)).
- Carpetas ADR/Decision son intencionalmente asimétricas: `adr/<ADR-...>/` (singular) y `decisions/<DEC-...>/` (plural), tomado verbatim del documento fuente — no es una inconsistencia de nombrado a normalizar. `axiom-adr supersede <old-id> <new-id>` es la única operación específica de ADR: actualiza ambos ADR en una llamada, de forma idempotente; si `old-id` ya estaba superseded por un ADR distinto, `supersede` igual lo reasigna (corrección deliberada, no bloqueada) pero devuelve un warning explícito. Decision no tiene `supersede` (no hay cadena de supersesión en su schema) — asimetría documentada y correcta.
- El índice de ADR/Decision es **derivado, no curado**: `axiom index rebuild`/`validate` y el check `IX-001` cubren `adr`/`decision` reusando `listArtifacts` — sin fichero de índice nuevo ni modelo de obligatoriedad/prioridad. Un índice ADR curado/versionado queda diferido hasta que exista un consumidor concreto.
- `axiom-increment.ts`/`axiom-bug.ts`/`axiom-plan.ts`/`axiom-role.ts` implementan `create`/`refine`/`specify`/`link-plan`/`link-increment`/`link-bug`/`list` por tipo, más `plan create` (añadido por este roadmap) y generación de ID estable por sistema (no texto libre). `workflow-state.json` (`@axiom/workflow`'s `state-store.ts`) persiste UN registro singleton por `WorkflowId` (`'increment' | 'bug' | 'plan' | 'role' | 'qa-e2e'`) — una máquina de estados por tipo de workflow, no un registro por instancia de artefacto. Es un diseño deliberadamente paralelo: `metadata.yml` (identidad por instancia) y `workflow-state.json` (máquina de estados por tipo) son almacenes independientes que no se conocen entre sí. Por eso `axiom-adr create`/`axiom-decision create`, y las rutas de migración de bootstrap (ver RF-AXM-021), nunca tocan `workflow-state.json`.
- No existe ninguna vista agregada legible generada (tipo `REGISTRO_INCREMENTOS.md`); `listArtifacts` + `axiom-{kind} list` (por tipo, con `--json`) + `axiom index rebuild` (todos los tipos, conteos, un comando) ya cubren todo caso de uso concreto conocido. El único hueco es una tabla de detalle completo, todos-los-tipos, en un solo fichero — conveniencia marginal sin consumidor nombrado hoy; no se debe reproponer sin nombrar antes un consumidor concreto (paso de CI, pipeline de documentación, o herramienta MCP que necesite específicamente un fichero en vez de una llamada a `listArtifacts`).

### RF-AXM-019 Validación de write-scope

Todo plan declara `targetRepos`/`allowedWriteScope`; Axiom valida los cambios reales contra ese scope.

- `validateWriteScope` (`Axiom/packages/workflow/src/write-scope.ts`) es la primitiva de comparación pura (sin acceso a filesystem/git/topología). Toma el `allowedWriteScope`/`targetRepos` de un plan, una lista de paths cambiados por repo, una lista de `sddRoles`, una lista de globs de paths generados/cache conocidos, y los estados de plan considerados válidos — y devuelve violaciones estructuradas de 5 tipos: `outside-scope`, `undeclared-repo`, `unexpected-sdd-change` (variante etiquetada de `undeclared-repo` para repos con rol `sdd`), `generated-cache-tampering` e `inconsistent-metadata`.
- El matching de globs usa `minimatch` (la única dependencia de glob-matching del monorepo).
- Los helpers conscientes de topología (`Axiom/packages/doctor/src/write-scope.ts`) incluyen `runGitDiff` (git diff + git status, deduplicado), `aggregateKnownGeneratedPathGlobs` (agrega `GENERATED_FILES_BY_TARGET` de `@axiom/installer` más `.axiom/cache/**`) y `buildRepoChangeSets` (resuelve las etiquetas de rol y los paths cambiados de cada repo de la topología).
- Dos superficies consumidoras comparten esta única primitiva sin lógica duplicada: `axiom validate changes --project <id> --plan <planId>` (CLI) y el check `WS-001` de `@axiom/doctor` (categoría `write-scope`), que valida el plan activo y se salta limpiamente cuando no hay ninguno activo.
- El ref base para "cambios reales" es el árbol de trabajo vs. `HEAD` (no se rastrea ningún ref de punto de aprobación de plan) — elección deliberada y documentada.

### RF-AXM-020 Índices de skills y de contexto técnico por rol

Dos artefactos estructuralmente independientes y aditivos — nunca comparten tipo, constante ni ruta de código ejecutable, solo el patrón general de loader/validador (nunca lanza excepción, unión discriminada, narrowing manual, sin Zod; una función `validateX` que valida un valor ya parseado sin I/O de ficheros, reusada directamente por el check de doctor correspondiente).

- **Índice de skills por rol** (`SkillsRoleIndex`, `Axiom/packages/skills/src/role-index.ts`): `{schemaVersion: 1, role, repoKinds?, mandatory: [{id, path?, reason?}], available: [{id, path?, tags, summary?}]}`, versionado independientemente del `SkillsCatalog` preexistente (`Axiom/packages/skills/src/catalog.ts`), sin reemplazarlo ni remodelarlo. Un fichero por rol, en `<rootPath>/axiom.config/skills-index/<role>.yaml`, junto a `axiom.config/skills-catalog.yaml` (renombrados desde `axiom.spec/config/` en `INC-20260703-config-folder-renames`). El catálogo markdown por rol vive en `<rootPath>/axiom.spec/target-axiom-skills/catalogs/<role>.md` (contenido de spec, no renombrado). El check `TC-012` de `@axiom/doctor` se salta limpiamente cuando `skills-index/` no existe (opcional por proyecto, a diferencia del `skills-catalog.yaml` único obligatorio, `TC-010`, que falla si está ausente).
- **Índice de contexto técnico** (`TechnicalContextIndex`, `Axiom/packages/technical-context/src/technical-context-index.ts`): `{schemaVersion: 1, projectId?, role, repoKinds, mandatory: {always: [{id, path, reason?}], whenTags: [{tags: [...], documents: [{id, path, reason?}]}]}, available: [{id, path, summary?, tags}]}`. A diferencia del índice de skills, `repoKinds` es obligatorio y todo `path` de documento es obligatorio (sin catálogo externo de respaldo). Un fichero por rol/tipo-de-repo, en `<specScopeAbsolutePath>/technical-context/indexes/<role-o-kind>.index.yml`. `resolveMandatoryDocuments(index, taskTags)` devuelve `mandatory.always` más todo grupo `whenTags` cuyas `tags` sean subconjunto de `taskTags` (todas presentes, no cualquiera), deduplicado por `id`; un grupo con `tags` vacío nunca matchea. El check `TC-013` se salta limpiamente cuando `technical-context/indexes/` no existe. `TechnicalContextIndex.status?: 'draft' | 'reviewed'` es aditivo y opcional: presente y en `'draft'` en índices generados por `axiom bootstrap from-code`; no existe mecanismo para pasarlo a `'reviewed'` (paso de curación humana, deliberadamente no automatizado).
- Ni `TC-012` ni `TC-013` verifican que los `path` de documento resuelvan a ficheros existentes en disco — diferido deliberadamente.
- **Aclaración de nombrado "repo SDD"**: "el repo SDD" significa dos cosas distintas según el contexto: (1) el sentido de gobierno de este workspace — `Axiom.SDD`, que no contiene código de producto y es irrelevante para dónde van físicamente los artefactos de `@axiom/skills`; (2) el sentido de topología de un proyecto producto — `TopologyManifest.sddRepo`, que en el modo `single-repo` por defecto coincide con `specRepo` y `rootPath`. Toda spec futura debe indicar cuál sentido usa en vez de la frase desnuda.

### RF-AXM-021 Bootstrap (`axiom bootstrap from-code` y `from-legacy-sdd`)

Ambas rutas de bootstrap son deliberadamente de alcance mínimo: pasadas mecánicas de introspección/migración, no las cadenas literales de 7 subagentes que describe el documento fuente para cada vía.

**`axiom bootstrap from-code --level minimal|basic [--role <role>]`** (Nivel 0/1): `--level` es un enum cerrado — el Nivel 2 (Standard) en adelante requiere comprensión arquitectónica/de negocio genuina y queda explícitamente diferido. `--role` nombra el conjunto de documentos/índice (`'repo'` por defecto) — no existe auto-detección de rol basada en topología. Análisis (`analyzer.ts`): `detectStacks` (presencia de manifest/lockfile), `buildRepoMap` (directorios de primer nivel + tabla literal pequeña nombre-de-carpeta-a-propósito), `detectCommands` (`package.json#scripts`, copiado verbatim). Cada documento generado lleva un banner literal `<!-- AXIOM:DRAFT -->`. La población de índice solo llena `TechnicalContextIndex.available`; `mandatory.*` siempre queda vacío (paso de curación humana). El primitivo de escritura `writeGuardedFile`/`resolveGuardedPath` de `@axiom/document-bootstrap` (path-guard + escritura atómica tmp+rename) es reusable por cualquier generador futuro que necesite escribir un fichero nuevo de forma segura dentro de un `projectRoot`.

**`axiom bootstrap from-legacy-sdd <path> [--dry-run]`**: pasada única, mecánica, de escaneo-extracción-creación. Asume un repo legado con cero o más carpetas de primer nivel `increments/`/`bugs/`/`plans/`/`adr/`/`decisions/`. El escaneo (`scanner.ts`) es de solo lectura, nunca lanza excepción (fallos por entrada se acumulan en una lista `failures`, el escaneo continúa); el tipo se deriva exclusivamente del nombre de carpeta legado fijo (cero sniffing de contenido). La migración (`migrator.ts`) crea cada artefacto nuevo vía las primitivas existentes de `@axiom/workflow` — sin escritor nuevo; el contenido del README se migra verbatim con un banner de procedencia `<!-- AXIOM:MIGRATED -->`. Incremento/bug/plan por defecto quedan en `'draft'` salvo que el estado escaneado mapee a uno de los 9 valores reales de `WorkflowState`. Colisiones: skip-and-report, nunca sobrescribe, nunca aborta el lote; correr la misma migración dos veces es seguro. `--dry-run` no escribe nada en el filesystem. `workflow-state.json` deliberadamente no se toca. Quedan diferidos: la reestructuración del cuerpo de texto legado, la inferencia de ADR/Decision desde prosa no estructurada, un flag `--kind`, y la migración del propio contenido de `Axiom.Spec`.

### RF-AXM-022 Operaciones `configure`/`upgrade`/`repair`

- **`axiom configure`** es una operación single-shot, sin flags, de "re-aplicar todo el perfil persistido": relee `profileTriple` de `init.json`, re-ejecuta `installProfile()`, y (para targets `copilot-vscode`/`github-copilot`/`opencode`) re-materializa los ficheros generados del adapter. **No** soporta añadir/quitar incrementalmente un repo, rol, adapter o tool/MCP a una instalación existente — ver el hueco de 7 operaciones (NFR-AXM-015) en [02_Requisitos_No_Funcionales.md](02_Requisitos_No_Funcionales.md), parcialmente cerrado por las 4 operaciones ADD de `axiom repo/adapter/provider/role add` (`INC-20260708-incremental-operations`, ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md)).
- **`axiom upgrade`** avanza `ManagedState` vía la cadena de migraciones registrada, con checkpoint rollback-first (ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)).
- **`axiom repair`** (general, top-level, `apps/cli/src/commands/repair.ts`) es solo composición: ejecuta `runDoctorChecks`, agrupa hallazgos `fail`/`warn` por categoría, y despacha exactamente 4 categorías conocidas-como-corregibles a funciones ya existentes — `install-profiles` -> `runConfigure`, `artifact-index` -> `runIndexRebuild`, `toolchain` -> `runToolchainRepair`, `memory` (coherencia de bindings MCP, `TC-007`) -> `runMcpRepair`. Toda otra categoría se reporta como "no auto-corregible; requiere revisión manual" — no se escribió lógica de corrección nueva para ninguna de ellas. Soporta `--dry-run`. Es distinto de dos subcomandos previos, narrows y específicos de dominio: `axiom toolchain repair` (re-deriva el estado de detección de una sola tool) y `axiom mcp repair` (verifica una entrada de manifest MCP, no instala físicamente el MCP).
- `axiom repair` se une a `upgrade` en los conjuntos `REQUIRES_CONFIRMATION`/`REQUIRES_PREVIEW` del driver de TUI (preview dry-run, confirmación Y/n, luego despacho), ya que ambos mutan el filesystem.
- **`axiom index rebuild`/`validate`/`list`** — ver RF-AXM-018.

## Requisitos funcionales de la tanda INC-20260708-* (cerrado)

### RF-AXM-025 Ejecución y selección de providers LOCALES (`INC-20260708-provider-runtime-execution-seam` / `-code-intel-providers-wired` / `-wizard-configure-provider-selection`)

El runtime debe poder EJECUTAR (no solo declarar) los providers de code-intel LOCALES resueltos por `@axiom/tool-routing`, y el proyecto debe poder ELEGIR cuáles habilita. Verificado: `@axiom/providers` aporta `invokeCapabilityLive` (resuelve vía `routeTool`, ejecuta el `ProviderClient` registrado, camina el fallback, nunca lanza) + `createStdioMcpClient` (cliente MCP stdio LOCAL) + el guard `LOCAL_ONLY`; `codegraph`/`serena`/`graphify` tienen `ProviderClient`s LOCALES reales (spawn del server MCP de cada tool, tool mapeada, degradación `not-installed`). La selección se hace en el wizard (step `providers`) y en `axiom configure --providers <csv>`, se persiste en `workspace.json#providers` (nunca en el `providers.yaml` schema-locked) y se materializa vía `buildProjectProviderRegistry`; el doctor la reporta (`PS-001`, `warn` no `fail` para habilitado-pero-no-instalado). Ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md) y [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md).

### RF-AXM-026 Memoria persistente real LOCAL cross-session (`INC-20260708-memory-real-local-backend`)

`@axiom/memory` debe ofrecer un backend real, LOCAL, cross-session con UPSERT topic-keyed, session summaries y superficie MCP. Verificado: `createEngramBackend` (proceso `engram mcp --project=<projectId>` LOCAL, SQLite+FTS5 vía MCP stdio) implementa la misma interfaz `MemoryBackend` que el backend JSON; `resolveMemoryBackend` auto-selecciona (probe engram, fallback JSON, nunca lanza); `memory.decisionRecall`/`memory.contextRecall` tienen handlers reales expuestos por el `McpServerKind` `memory` (`axiom mcp serve --kind memory`). GATE 0024 preservado + pin de proceso (ver [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md)).

### RF-AXM-027 Aprendizaje continuo (captura/recall de lecciones) (`INC-20260708-continuous-learning`)

Axiom debe poder capturar lecciones al cierre y recuperarlas al inicio del trabajo, de forma determinista y sin motor especulativo. Verificado: `extractLessons`/`persistLessons`/`recallLessons` (módulo `learning.ts` de `@axiom/memory`) sobre el backend real (AB5) y registros de audit-trail; `axiom learn capture [--from-audit] [--text]`/`axiom learn list` como CLI; bloque best-effort de lecciones recientes en `axiom context status`. Sin nuevo `MemoryKind` (lecciones = `kind: 'pattern'` tag `'lesson'`), sin ML. Ver [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md) y [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

### RF-AXM-028 Operaciones incrementales ADD sobre instalaciones existentes (`INC-20260708-incremental-operations`)

El runtime debe permitir AÑADIR incrementalmente un repo / adapter / provider / rol a un proyecto ya inicializado, sin re-correr el setup completo. Verificado: `axiom repo add`, `axiom adapter add <target>`, `axiom provider add <id>`, `axiom role add <roleId> --path <path>` — idempotentes, no-clobber, resuelven el proyecto desde cwd y reusan el motor multi-repo (`runWorkspaceSetup` + helpers exportados). Cierra la mitad aditiva del hueco de 7 operaciones (NFR-AXM-015 en [02_Requisitos_No_Funcionales.md](02_Requisitos_No_Funcionales.md)); los REMOVE quedan diferidos. Ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md) y [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md).

## Requisitos funcionales de la tanda INC-20260715-* (launcher visual + tuning de adapters, cerrado)

### RF-AXM-029 Tuning de agente por adapter (`INC-20260715-adapter-agent-tuning`)

Cada adapter puede declarar `agentTuning` (`verbosity` / `personality` / `model` opcional). El prompt pregenerado del launcher inyecta un preámbulo conciso ("Ajustes del agente" + directiva de trabajo directo/pragmático) derivado de esos ajustes, para que el agente responda de forma terse y económica en tokens y centrada en la tarea. Los tres adapters de serie (`claude-code`, `github-copilot`, `cli`) traen `{ verbosity: 'low', personality: 'pragmatic' }`; sin `agentTuning` no se emite preámbulo (retrocompatible). Es prompt-shaping puro (no toca model-routing ni selección de providers). Ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md) y [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

### RF-AXM-030 Gate de doctor pre-lanzamiento en el launcher (`INC-20260715-launcher-doctor-gate`)

El launcher web ejecuta `runDoctorChecks` del proyecto seleccionado y muestra el estado (pass/warn/fallo) y lo que falta ANTES de lanzar/ejecutar. Las acciones mutantes (ejecutar/lanzar) avisan si hay checks en fallo y requieren un segundo clic para continuar (gate visible, no bloqueo duro). Endpoint `GET /api/projects/:id/launcher/doctor`, best-effort y no-crash. Ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md) y [09 revisiones en manuales](manuales/09_Revisiones.md).

### RF-AXM-031 Onboarding visual desde el launcher (`INC-20260715-launcher-onboarding`)

Desde el launcher, sin terminal ni TUI, un miembro puede: instalar Axiom en un proyecto nuevo (`runInit`), unirse a un proyecto existente (`runProjectsJoin`), y registrar roles + asociarlos a repos (`runRolesRegister` / `runRolesAssign`), con un explorador de carpetas (endpoint `GET /api/launcher/browse`). Endpoints server-level `POST /api/launcher/{install,join}` (pre-proyecto) y project-scoped `POST /api/projects/:id/launcher/roles/{register,assign}`, todos confirm-gated. Tras un install/join exitoso el proyecto aparece en el selector. Ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md) y [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md).

### RF-AXM-032 Puente Azure DevOps en creación desde el launcher (`INC-20260715-launcher-ado-bridge`)

Cuando el tracker ADO está configurado (`kind:'ado'` + `enabled` + org/project), tras crear un incremento/bug desde el launcher se ofrece un work item pre-rellenado (incremento→`User Story`, bug→`Bug`), editable y confirm-gated de un clic, reusando el endpoint ADO existente (`apiAdoCreateWorkItem`). NO acopla el ciclo de vida (la creación en Axiom es idéntica con o sin plugin); si no está configurado se muestra sólo una nota informativa sin red. Ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md) y [manuales/12_Plugin_Azure_DevOps.md](manuales/12_Plugin_Azure_DevOps.md).

### RF-AXM-033 Manuales de operación en la spec (`INC-20260715-spec-manuales`)

La spec incluye `specs/manuales/`: guías de usuario cruzadas (qué es cada cosa, configuración, actualización de versiones, generación de spec/contexto técnico, incrementos, bugs, planes, implementación, revisiones, archivado, launcher visual y plugin de Azure DevOps), pensadas para un equipo recién instalado. Ver [manuales/README.md](manuales/README.md) y [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md).

## Requisitos funcionales de la tanda INC-20260715-* (alineación con sistemas role-specialized, cerrado)

Tras una revisión comparativa contra un sistema SDD role-specialized (KVP25 `.github`), esta tanda cierra los huecos reales del set que Axiom INSTALA en un proyecto — manteniendo la genericidad adapter-agnóstica (lo específico de stack se inyecta por proyecto, RF-AXM-039). Catálogo tras la tanda: 18 skills / 14 agents.

### RF-AXM-034 Disciplinas transversales como skills reutilizables (`INC-20260715-reusable-discipline-skills`)

Axiom expone como skills de catálogo independientes las 4 disciplinas transversales antes sólo embebidas: `axiom-structured-doubts` (parada y consulta con opciones cerradas), `axiom-functional-checklist-coverage` (cobertura `CF-xx` como contrato entre fases), `axiom-plan-drift-alignment` (reconciliación spec↔plan por versión, impacto por rol) y `axiom-role-close-doc` (cierre documental técnico por rol). Son citables/evolucionables y las referencian los flujos. Ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

### RF-AXM-035 Gate de revisión por fase instalado (`INC-20260715-phase-reviewer`)

`axiom-phase-reviewer` (skill+agent+superficie en el repo `sdd`) revisa spec/plan/código con lentes dedicadas, devuelve **VEREDICTO OK|KO** y aplica un barrido exhaustivo *loop-until-dry* con ledger de hallazgos; de solo lectura. La revisión se expone además como acciones del launcher (`review-spec`/`review-plan`/`review-code`, prompt-only) reusando `buildReviewPrompt`. Ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md) y [manuales/09_Revisiones.md](manuales/09_Revisiones.md).

### RF-AXM-036 Superficies de consolidación y contexto técnico (`INC-20260715-consolidation-surfaces`)

`axiom-spec-integrator` consolida el conocimiento estable de un incremento/bug implementado en la spec canónica del proyecto y lo archiva (atómico, confirm-gated); `axiom-tech-context` autorea/mantiene el contexto técnico (verificable contra código) y detecta **spec-drift** criterio-a-criterio. Ambos en el repo `spec`. Ver [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md).

### RF-AXM-037 Gates de QA y seguridad (`INC-20260715-quality-gates`)

`axiom-qa-validator` genera un plan de pruebas derivado de los criterios de aceptación con trazabilidad 1:N y marca `⚠️ SIN COBERTURA` (nunca inventa escenarios, nunca valida con huecos). `axiom-security-reviewer` pasa de stub a cuerpo real (checklist de 10 familias de riesgo + severidad, solo-lectura/defensivo, gate opcional no bloqueante). Ver [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md).

### RF-AXM-038 Análisis de alcance opcional en el planner (`INC-20260715-planner-analysis-fanout`)

`axiom-role-planner` gana un análisis de alcance opcional, guiado por señales de complejidad: alcance simple → planifica directo; multi-capa con señales fuertes → análisis por dimensión (backend/frontend/qa/transversal) antes de escribir los planes de rol, con bloqueo-si-insuficiente. Portable y adapter-agnóstico (nunca exige spawnear subagentes). Ver [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md).

### RF-AXM-039 Canal de inyección por proyecto documentado (`INC-20260715-install-injection-guide`)

Manual `manuales/13_Skills_Agentes_y_Roles.md`: qué instala Axiom por rol de repo y cómo un proyecto inyecta su profundidad de stack (patrones, permisos, build/test) vía `axiom.config/skills-index/<role>.yaml`, el contexto técnico y las skills de rol — sin tocar el producto. Es la clave de "genérico sin perder funcionalidad". Ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md).