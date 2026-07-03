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

El CLI debe poder inicializar un proyecto validando el nombre (`^[a-z0-9][a-z0-9-]{0,62}$`), determinando el layout (`self-hosted` vs `installed-multi-repo`), generando `axiom.yaml` con el profile triple (`functionalProfile` + `operationalOverlay` + `adapterTarget`), creando `.sdd/local/` y `.sdd/<projectName>/`, y persistiendo `init.json`. Fuente: `Axiom/docs/cli/init.md`.

### RF-AXM-007 Registro de miembros (`axiom join`)

El CLI debe registrar miembros del proyecto por id (`user:alice`, `agent:sdd`, etc.) en `.sdd/<projectName>/members.yaml`, deduplicando por igualdad exacta. Fuente: `Axiom/docs/cli/join.md`.

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

- **Índice de skills por rol** (`SkillsRoleIndex`, `Axiom/packages/skills/src/role-index.ts`): `{schemaVersion: 1, role, repoKinds?, mandatory: [{id, path?, reason?}], available: [{id, path?, tags, summary?}]}`, versionado independientemente del `SkillsCatalog` preexistente (`Axiom/packages/skills/src/catalog.ts`), sin reemplazarlo ni remodelarlo. Un fichero por rol, en `<rootPath>/axiom.spec/config/skills-index/<role>.yaml`, junto a `axiom.spec/config/skills-catalog.yaml`. El catálogo markdown por rol vive en `<rootPath>/axiom.spec/target-axiom-skills/catalogs/<role>.md`. El check `TC-012` de `@axiom/doctor` se salta limpiamente cuando `skills-index/` no existe (opcional por proyecto, a diferencia del `skills-catalog.yaml` único obligatorio, `TC-010`, que falla si está ausente).
- **Índice de contexto técnico** (`TechnicalContextIndex`, `Axiom/packages/technical-context/src/technical-context-index.ts`): `{schemaVersion: 1, projectId?, role, repoKinds, mandatory: {always: [{id, path, reason?}], whenTags: [{tags: [...], documents: [{id, path, reason?}]}]}, available: [{id, path, summary?, tags}]}`. A diferencia del índice de skills, `repoKinds` es obligatorio y todo `path` de documento es obligatorio (sin catálogo externo de respaldo). Un fichero por rol/tipo-de-repo, en `<specScopeAbsolutePath>/technical-context/indexes/<role-o-kind>.index.yml`. `resolveMandatoryDocuments(index, taskTags)` devuelve `mandatory.always` más todo grupo `whenTags` cuyas `tags` sean subconjunto de `taskTags` (todas presentes, no cualquiera), deduplicado por `id`; un grupo con `tags` vacío nunca matchea. El check `TC-013` se salta limpiamente cuando `technical-context/indexes/` no existe. `TechnicalContextIndex.status?: 'draft' | 'reviewed'` es aditivo y opcional: presente y en `'draft'` en índices generados por `axiom bootstrap from-code`; no existe mecanismo para pasarlo a `'reviewed'` (paso de curación humana, deliberadamente no automatizado).
- Ni `TC-012` ni `TC-013` verifican que los `path` de documento resuelvan a ficheros existentes en disco — diferido deliberadamente.
- **Aclaración de nombrado "repo SDD"**: "el repo SDD" significa dos cosas distintas según el contexto: (1) el sentido de gobierno de este workspace — `Axiom.SDD`, que no contiene código de producto y es irrelevante para dónde van físicamente los artefactos de `@axiom/skills`; (2) el sentido de topología de un proyecto producto — `TopologyManifest.sddRepo`, que en el modo `single-repo` por defecto coincide con `specRepo` y `rootPath`. Toda spec futura debe indicar cuál sentido usa en vez de la frase desnuda.

### RF-AXM-021 Bootstrap (`axiom bootstrap from-code` y `from-legacy-sdd`)

Ambas rutas de bootstrap son deliberadamente de alcance mínimo: pasadas mecánicas de introspección/migración, no las cadenas literales de 7 subagentes que describe el documento fuente para cada vía.

**`axiom bootstrap from-code --level minimal|basic [--role <role>]`** (Nivel 0/1): `--level` es un enum cerrado — el Nivel 2 (Standard) en adelante requiere comprensión arquitectónica/de negocio genuina y queda explícitamente diferido. `--role` nombra el conjunto de documentos/índice (`'repo'` por defecto) — no existe auto-detección de rol basada en topología. Análisis (`analyzer.ts`): `detectStacks` (presencia de manifest/lockfile), `buildRepoMap` (directorios de primer nivel + tabla literal pequeña nombre-de-carpeta-a-propósito), `detectCommands` (`package.json#scripts`, copiado verbatim). Cada documento generado lleva un banner literal `<!-- AXIOM:DRAFT -->`. La población de índice solo llena `TechnicalContextIndex.available`; `mandatory.*` siempre queda vacío (paso de curación humana). El primitivo de escritura `writeGuardedFile`/`resolveGuardedPath` de `@axiom/document-bootstrap` (path-guard + escritura atómica tmp+rename) es reusable por cualquier generador futuro que necesite escribir un fichero nuevo de forma segura dentro de un `projectRoot`.

**`axiom bootstrap from-legacy-sdd <path> [--dry-run]`**: pasada única, mecánica, de escaneo-extracción-creación. Asume un repo legado con cero o más carpetas de primer nivel `increments/`/`bugs/`/`plans/`/`adr/`/`decisions/`. El escaneo (`scanner.ts`) es de solo lectura, nunca lanza excepción (fallos por entrada se acumulan en una lista `failures`, el escaneo continúa); el tipo se deriva exclusivamente del nombre de carpeta legado fijo (cero sniffing de contenido). La migración (`migrator.ts`) crea cada artefacto nuevo vía las primitivas existentes de `@axiom/workflow` — sin escritor nuevo; el contenido del README se migra verbatim con un banner de procedencia `<!-- AXIOM:MIGRATED -->`. Incremento/bug/plan por defecto quedan en `'draft'` salvo que el estado escaneado mapee a uno de los 9 valores reales de `WorkflowState`. Colisiones: skip-and-report, nunca sobrescribe, nunca aborta el lote; correr la misma migración dos veces es seguro. `--dry-run` no escribe nada en el filesystem. `workflow-state.json` deliberadamente no se toca. Quedan diferidos: la reestructuración del cuerpo de texto legado, la inferencia de ADR/Decision desde prosa no estructurada, un flag `--kind`, y la migración del propio contenido de `Axiom.Spec`.

### RF-AXM-022 Operaciones `configure`/`upgrade`/`repair`

- **`axiom configure`** es una operación single-shot, sin flags, de "re-aplicar todo el perfil persistido": relee `profileTriple` de `init.json`, re-ejecuta `installProfile()`, y (para targets `copilot-vscode`/`github-copilot`/`opencode`) re-materializa los ficheros generados del adapter. **No** soporta añadir/quitar incrementalmente un repo, rol, adapter o tool/MCP a una instalación existente — ver el hueco de 7 operaciones en [02_Requisitos_No_Funcionales.md](02_Requisitos_No_Funcionales.md).
- **`axiom upgrade`** avanza `ManagedState` vía la cadena de migraciones registrada, con checkpoint rollback-first (ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)).
- **`axiom repair`** (general, top-level, `apps/cli/src/commands/repair.ts`) es solo composición: ejecuta `runDoctorChecks`, agrupa hallazgos `fail`/`warn` por categoría, y despacha exactamente 4 categorías conocidas-como-corregibles a funciones ya existentes — `install-profiles` -> `runConfigure`, `artifact-index` -> `runIndexRebuild`, `toolchain` -> `runToolchainRepair`, `memory` (coherencia de bindings MCP, `TC-007`) -> `runMcpRepair`. Toda otra categoría se reporta como "no auto-corregible; requiere revisión manual" — no se escribió lógica de corrección nueva para ninguna de ellas. Soporta `--dry-run`. Es distinto de dos subcomandos previos, narrows y específicos de dominio: `axiom toolchain repair` (re-deriva el estado de detección de una sola tool) y `axiom mcp repair` (verifica una entrada de manifest MCP, no instala físicamente el MCP).
- `axiom repair` se une a `upgrade` en los conjuntos `REQUIRES_CONFIRMATION`/`REQUIRES_PREVIEW` del driver de TUI (preview dry-run, confirmación Y/n, luego despacho), ya que ambos mutan el filesystem.
- **`axiom index rebuild`/`validate`/`list`** — ver RF-AXM-018.