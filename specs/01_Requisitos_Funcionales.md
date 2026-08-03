# 01 Requisitos Funcionales

## Meta-requisitos del workspace de desarrollo (dogfooding)

### RF-AXM-001 Separación por ownership

Axiom debe distinguir claramente el repo de runtime (`Axiom/`), el repo de spec (`Axiom.Spec/`) y el repo SDD (`Axiom.SDD/`). Vigente hoy: `Axiom.SDD/AGENTS.md` fija esta separación como regla operativa activa para construir el propio producto.

### RF-AXM-002 Workflow SDD sobre Axiom

Axiom debe permitir que la especificación, planificación, implementación y validación del propio producto se ejecuten con sus propias reglas operativas (bootstrap lifecycle de `Axiom.SDD/AGENTS.md`: entender → localizar/crear spec → refinar → implementar → validar → revisar → cerrar → integrar conocimiento estable).

### RF-AXM-003 Modelo de topología de repos (multi-repo, dentro del producto)

El runtime debe conocer qué repos forman parte de un proyecto gestionado, cómo se llaman, qué alias usan y dónde se resuelven. Implementado hoy vía `@axiom/topology` (`topology.yaml`: `repoRefs` + `roleAssignments` + QA lanes) y expuesto por `axiom topology`/`axiom repo`.

### RF-AXM-004 Resolución multi-surface

La topología de repos y el proyecto activo deben poder resolverse tanto desde workspace local (`@axiom/filesystem-truth`, `@axiom/project-resolution`) como desde las surfaces externas soportadas por adapters y MCP. El runtime actual expone servers MCP ejecutables y acotados al proyecto (`@axiom/mcp-server`) para los kinds `sdd`, `spec`, `memory` y `axiom`, además del catálogo gestionado por `@axiom/toolchain` y los comandos `axiom mcp`. La resolución MCP no es un acceso genérico ilimitado: el server fija el proyecto y rechaza referencias a otro scope.

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

### RF-AXM-056 Versionado reproducible de herramientas externas (`INC-20260730-toolchain-versioning`)

El runtime debe permitir fijar y revisar versiones de tools externas sin convertir el catálogo en una orden implícita de instalación:

- `axiom.config/toolchain-catalog.yaml` debe aceptar schema 2 con `versionExtractor`, canales `stable`/`candidate`/`edge` y restricciones opcionales de compatibilidad; el loader mantiene compatibilidad de lectura con schema 1.
- `.axiom-state/<project>/toolchain.lock` debe usar schema 1 y registrar, por tool, `id`, `version` y `channel`, con metadatos opcionales del probe (`probeCommand`, `probeOutput`, `probedAt`). Su escritura debe ser atómica y project-scoped.
- `extractVersion()` debe extraer la versión desde la salida del probe usando la regex declarada por la entrada del catálogo. Las tools sin contrato local de probe no deben recibir una ejecución inventada.
- `axiom toolchain show` debe exponer versión instalada, versión locked y canal; `axiom toolchain plan` debe mostrar el diff sin escribir y admitir `--channel` y `--id` repetido; `axiom toolchain upgrade` debe admitir `--dry-run` y exigir `--yes` para persistir.
- `upgrade` solo debe modificar el lockfile. Debe crear checkpoint y restaurarlo, eliminando el nuevo lockfile si no existía uno previo, ante un fallo de escritura o de verificación posterior.
- Doctor debe exponer TC-020 (lockfile), TC-021 (compatibilidad), TC-022 (drift) y TC-023 (canal). La ruta síncrona conserva semántica no bloqueante; la comparación real de versiones queda en `doctor --deep` y sus probes solo pueden producir `pass`, `warn` o `skip`.

### RF-AXM-014 Model routing por slot (`axiom model`)

El CLI debe permitir ver, fijar, quitar y resetear el modelo asignado a cada slot operativo (`increment`, `bug`, `plan`, `implementation`, `qa-e2e`, `review`, `archive`), con fallback declarado por `SupportLevel` del target (`multi-mode`, `single-mode`, `fallback-only`) y proyección opcional a `.opencode/model-routing.json`. Fuente: `Axiom/docs/cli/model.md`, package `@axiom/model-routing`.

### RF-AXM-015 Gestión de components y skills

El CLI debe exponer catálogo, instalación/desinstalación con checkpoint (`axiom components`) y registro/drift de skills materializadas contra `.opencode/skills-lock.yaml` (`axiom skills`). Fuente: `Axiom/docs/cli/components.md`, `skills.md`.

### RF-AXM-016 Generación de surfaces por adapter target

El runtime debe poder generar configuración específica por adapter target mediante una función `generate<Target>Config` de firma común (`Promise<Result<GeneratorResult, AdapterGeneratorError>>`). Desde la tanda `INC-20260726-*` (ver RF-AXM-049/RF-AXM-050) el vocabulario canónico es de **10** `ADAPTER_TARGETS` y **9** de ellos tienen paquete `@axiom/adapters-<target>` dedicado (`opencode`, `claude-code`, `github-copilot`, `vscode`, `cursor`, `litellm`, `codex`, `antigravity`, `visual-studio-2026`); `copilot-vscode` es el único target declarado en MVP que sigue sin paquete dedicado (usa `writeCopilotInstructions`). El nivel `SupportLevel` de model-routing (`fallback-only` para la mayoría de targets) es un eje SEPARADO de "tiene generador dedicado" y no cambió en esa tanda.

### RF-AXM-017 Superficie de comandos operativos más amplia que la documentada

La superficie operativa se define por los comandos registrados en `apps/cli/src/index.ts` y por sus funciones `run*`, no por un conteo histórico de ficheros. Incluye, entre otras, las superficies de workspace multi-repo, workflow (`axiom increment`, `axiom bug`, `axiom plan`, `axiom role`), adapters, providers, MCP, memoria, toolchain, bootstrap, adopción y launcher. Los comandos que todavía no tengan documentación propia deben citarse con el contrato verificable de código y tests; la mera existencia de un fichero no convierte una capacidad en contrato estable. La vía real del workflow usa los comandos CLI y `@axiom/workflow`; no debe documentarse el catálogo interno de intent commands no cableado como punto de entrada de usuario.

### RF-AXM-023 Punto de entrada TUI, menú de bootstrap sin proyecto y wizard de `init` guiado

- `axiom` sin subcomando debe abrir la TUI (acción por defecto de Commander en `apps/cli/src/index.ts`, reusando `runTuiCli`). Los subcomandos, `--help` y `--version` matchean antes que la acción por defecto y quedan sin cambios.
- `axiom`/`axiom tui` en una carpeta SIN proyecto Axiom (ni por resolución directa ni por fallback) y con input interactivo (TTY) debe abrir un menú de bootstrap estilizado (screen `setup` de `@axiom/tui`): inicializar Axiom en esta carpeta · actualizar Axiom (self-update) · ver proyectos registrados · salir. Elegir un proyecto registrado hace `chdir` + abre la TUI operativa sobre él. En no-TTY (CI, pipes) debe preservarse el error `projectNotFound` con exit 1.
- Elegir "inicializar" debe lanzar un WIZARD guiado (ver RF-AXM-024 para el comportamiento vigente hoy, que reemplazó el wizard single-repo original de este requisito). El wizard es UI aditiva: no reemplaza a `axiom init` no-interactivo (RF-AXM-006). El detalle de UI, el split de capas y las etiquetas viven en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md) — no se duplican aquí.

### RF-AXM-024 Setup de workspace multi-repo desde el wizard de la TUI (`INC-20260705-*`)

La acción "Inicializar Axiom en esta carpeta" de la screen `setup` debe preparar, en UNA operación, un workspace Axiom multi-repo completo — no solo inicializar el repo actual. Reemplaza el wizard single-repo de 6 pasos de RF-AXM-023 (`INC-20260703-tui-init-wizard`); el comando `axiom init`/`runInit` (RF-AXM-006) sigue siendo single-repo y sin cambios.

- El wizard debe recoger, en orden y con defaults sensatos pre-cargados: `name` (texto libre, del que se deriva el `projectId`) → `sddPath` → `specPath` → `roles` (texto libre, separado por comas, con roles arbitrarios y cero o más valores) → un path por rol seleccionado → `profile` → `overlay` → `adapters` (multi-select de los 10 `ADAPTER_TARGETS`) → `providers` (multi-select de los 3 providers seleccionables, sin preselección) → un único resumen de confirmación. Los defaults de paths son rutas hermanas derivadas del proyecto y también se aceptan rutas absolutas.
- Solo al confirmar, el CLI debe ensamblar un `WorkspaceSetupSpec` y llamar a `runWorkspaceSetup` (`INC-20260705-workspace-multirepo-setup-engine`), que scaffoldea/parametriza el repo de control (SDD), el repo de Spec y los N repos de rol; escribe un `axiom.yaml` por repo con un mapa `paths` recíproco y consciente de rol (los repos "se conocen entre sí"); escribe un `axiom.config/topology.yaml` y `.axiom-state/local/topology-bindings.yaml` en el repo de control; y registra TODOS los repos bajo un único proyecto en el registro v2 (`~/.axiom/projects.yml`) en una llamada idempotente (`upsertProjectReposV2`), con una guarda no-clobber que nunca sobrescribe un `axiom.yaml` de otro proyecto. Tras el registro, genera best-effort la config MCP (SDD MCP + Spec MCP, `INC-20260705-workspace-mcp-generation`). El detalle de datos vive en [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md); el de UI y capas en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md); el de MCP en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).
- Cancelar (en cualquier step o en la confirmación) no debe llamar a `runWorkspaceSetup` y debe volver al menú `setup` sin efectos; un fallo de `runWorkspaceSetup` debe capturarse, reportarse y volver al menú `setup` sin crashear la TUI.
- El step `target` (select único) se sustituyó por `adapters` (multi-select de los 10 `ADAPTER_TARGETS`, default `['opencode']`); los adapters multi-seleccionados se instalan en TODOS los repos del workspace (control + spec + cada rol), su selección se persiste en `<controlRepo>/.axiom-state/<projectId>/workspace.json`, y la proyección MCP cubre los targets con configuración nativa de proyecto. Nueve targets tienen paquete `@axiom/adapters-<target>` dedicado; `copilot-vscode` usa el escritor de instrucciones compartido. El nivel de routing (`multi-mode`, `single-mode`, `fallback-only`) es independiente de esta cobertura.
- El step `providers` persiste la selección en `<controlRepo>/.axiom-state/<projectId>/workspace.json#providers`. Solo `cmm` y `serena` son providers seleccionables de code-intel; `engram` se resuelve mediante el backend de memoria. Una selección vacía conserva el fallback `filesystem` y no implica que se hayan habilitado tools externas.
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

- **`axiom configure`** re-aplica el perfil persistido (`profileTriple` de `init.json`) y, cuando se proporciona `--providers <csv>`, actualiza además la selección project-scoped de providers en `workspace.json#providers`. La operación de reconfiguración no añade ni elimina repos, adapters, roles o tools por sí sola; esas mutaciones tienen superficies separadas y aditivas: `axiom repo add`, `axiom adapter add <target>`, `axiom provider add <id>` y `axiom role add <roleId> --path <path>`. Las operaciones REMOVE siguen diferidas, conforme a NFR-AXM-015.
- **`axiom upgrade`** avanza `ManagedState` vía la cadena de migraciones registrada, con checkpoint rollback-first (ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)).
- **`axiom repair`** (general, top-level, `apps/cli/src/commands/repair.ts`) es solo composición: ejecuta `runDoctorChecks`, agrupa hallazgos `fail`/`warn` por categoría, y despacha exactamente 4 categorías conocidas-como-corregibles a funciones ya existentes — `install-profiles` -> `runConfigure`, `artifact-index` -> `runIndexRebuild`, `toolchain` -> `runToolchainRepair`, `memory` (coherencia de bindings MCP, `TC-007`) -> `runMcpRepair`. Toda otra categoría se reporta como "no auto-corregible; requiere revisión manual" — no se escribió lógica de corrección nueva para ninguna de ellas. Soporta `--dry-run`. Es distinto de dos subcomandos previos, narrows y específicos de dominio: `axiom toolchain repair` (re-deriva el estado de detección de una sola tool) y `axiom mcp repair` (verifica una entrada de manifest MCP, no instala físicamente el MCP).
- `axiom repair` se une a `upgrade` en los conjuntos `REQUIRES_CONFIRMATION`/`REQUIRES_PREVIEW` del driver de TUI (preview dry-run, confirmación Y/n, luego despacho), ya que ambos mutan el filesystem.
- **`axiom index rebuild`/`validate`/`list`** — ver RF-AXM-018.

## Requisitos funcionales de la tanda INC-20260708-* (cerrado)

### RF-AXM-025 Ejecución y selección de providers LOCALES (`INC-20260708-provider-runtime-execution-seam` / `-code-intel-providers-wired` / `-wizard-configure-provider-selection`)

El runtime debe poder EJECUTAR (no solo declarar) los providers LOCALES resueltos por `@axiom/tool-routing`, y el proyecto debe poder ELEGIR cuáles habilita. Verificado: `@axiom/providers` aporta `invokeCapabilityLive` (resuelve vía `routeTool`, ejecuta el `ProviderClient` registrado, camina el fallback y nunca lanza) y `createStdioMcpClient` (cliente MCP stdio LOCAL). El conjunto seleccionable es exactamente `cmm`, `serena` y `engram`; `CODE_INTEL_PROVIDER_IDS` contiene `cmm` y `serena`, mientras `engram` usa el backend `MemoryBackend` y no un `ProviderClient` ficticio. `cmm` cubre las capabilities estructurales `code.knowledgeGraph` y `code.structureAnalysis`; `serena` cubre `code.semanticNavigation`; ambos degradan a filesystem cuando la tool local no está disponible. `codegraph` y `graphify` ya no son seleccionables, registrables ni enrutables.

La selección se hace en el wizard (step `providers`) y en `axiom configure --providers <csv>`, se persiste en `workspace.json#providers` (nunca recortando el registry canónico de `providers.yaml`) y se materializa vía `buildProjectProviderRegistry`/`resolveMemoryBackend`. El doctor la reporta con `PS-001`: una tool habilitada pero no instalada produce `warn`, no `fail`. Ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md) y [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md).

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

## Requisitos funcionales de la tanda INC-20260724-* (cerrado)

Graduación a *full product lifecycle* para proyectos instalados (los repos propios de Axiom no se migran). Ver [00_Resumen_Ejecutivo.md](00_Resumen_Ejecutivo.md).

### RF-AXM-040 Modelo de repo único gestionado `<project>.axiom` (`INC-20260724-topology-single-axiom-repo` / `-adopt-creates-axiom-repo`)

La topología soporta un único repo gestionado `<project>.axiom` (`schemaVersion: 2`: `axiomRepo`/`codeRepos`/`legacyRepos`, discriminadores `kind`/`mode`), retrocompatible con `sddRepo`/`specRepo` (auto-map + warning `deprecated-legacy-shape`). La adopción CREA un `<project>.axiom` hermano nuevo y migra la spec/context legacy dentro (formato Axiom), dejando los repos legacy byte-for-byte intactos (registrados `legacyRepos[]`, `mode: read-only-source`). Ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md).

### RF-AXM-041 Provenance/lifecycle + manifest de migración (`INC-20260724-provenance-lifecycle-manifest`)

Cada artefacto lleva metadata de origen/ciclo de vida (`origin.source`, `lifecycle.state`, `managedBy`, `exportPolicy.rollbackEligible`); editar un migrado lo transiciona a `migrated-and-modified` automáticamente. Manifest global en `<project>.axiom/migration/migration-manifest.yaml`. Aditivo/retrocompatible.

### RF-AXM-042 Salida de Axiom (`axiom eject`) (`INC-20260724-export-eject-rollback`)

`axiom eject` vuelca los artefactos rollback-eligible (`axiom-native` + `migrated-and-modified`) a una carpeta de export con reporte + manifest, dry-run por defecto, sin escribir jamás en legacy. Ver [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md) y [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md).

### RF-AXM-043 MCP unificado de `<project>.axiom` (`INC-20260724-unified-axiom-mcp`)

Un solo broker `axiom` expone los reads de spec/increment/bug/adr/technical-context + edición de estados (`transitionApply`) + push (`gitCommitSync`) + reads de topología/manifest/adoption-state; los brokers `sdd`/`spec`/`memory` se mantienen por retrocompat; Engram sigue aparte. Ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

### RF-AXM-044 `cmm` como único proveedor estructural de code-intel (`INC-20260724-cmm-replaces-graphify-codegraph`)

`cmm` (ADR-0031) es el único proveedor estructural (`graphify`/`codegraph` retirados); `serena` sigue simbólico; fallback siempre a filesystem; freshness/auto-sync on-demand sin hooks. Ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

### RF-AXM-045 RTK y concisión como skills (`INC-20260724-rtk-skill-invoked` / `-concision-skills-policy`)

RTK es usable solo vía la skill `axiom-terminal-output-efficient` (sin hooks/wrapper global; con exclusiones never-compress). La skill `axiom-concision-discipline` adopta la filosofía de concisión de Caveman sin instalar Caveman. Ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md) y [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md).

### RF-AXM-046 Higiene del lock de AutoSkills (`INC-20260724-autoskills-lock-hygiene`)

Las skills instaladas por AutoSkills llevan `provenance: autoskills` + `installedAt` y pasan por un gate de policy allow/deny/licencia opcional (default allow-all). Corre por-code-repo en install-time. Ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md) y [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md).

### RF-AXM-047 Ejecución en worktree seleccionable (`INC-20260724-git-worktree-services` … `-worktree-mode-selection` / `-worktree-close-correctness`)

Un incremento/bug/plan puede ejecutarse in-place o en un git worktree dedicado; el default lo elige el arquitecto en la instalación (`executionMode`) y es overridable por run (`axiom-role start --worktree`/`--in-place`). Incluye primitivas git worktree, entidad `Execution` + `ExecutionStore`, provisioning portable, aislamiento de providers/índices por worktree, harvest+cleanup seguro (kill→harvest→teardown→remove, dirty=hard stop) y cierre correcto. Ver [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md).

### RF-AXM-048 Freshness de artefactos SDD (`INC-20260724-sdd-artifact-freshness`)

Al leer/editar un increment/bug/plan, un auto-fetch best-effort detecta desfase con el remoto y emite un warning `stale-artifact` (nunca bloquea); el push va acotado a la carpeta del artefacto (nunca `-A`). Sin hook git obligatorio. Ver [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md).

## Requisitos funcionales de la tanda INC-20260726-* (paridad de alcance de adapters + front de onboarding del launcher, cerrado)

Tanda de 7 incrementos (spec-first, gate verde por incremento) que lleva el conjunto de adapters a **paridad de primera clase** (registro canónico, generadores, MCP nativo, routing y prompt del launcher), convierte el launcher web en un front de onboarding config-rich y añade probes de runtime opt-in a `doctor`. Ver [00_Resumen_Ejecutivo.md](00_Resumen_Ejecutivo.md) y [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md); no funcionales asociados en [02_Requisitos_No_Funcionales.md](02_Requisitos_No_Funcionales.md) (NFR-AXM-020/021).

### RF-AXM-049 Registro canónico de adapter targets (10 ids, fuente única reconciliada) (`INC-20260726-adapter-registry-canonical`)

El vocabulario `AdapterTarget`/`ADAPTER_TARGETS` es ahora exactamente **10 ids**: `opencode, copilot-vscode, claude-code, antigravity, visual-studio-2026, cursor, github-copilot, litellm, vscode, codex` (`DEFAULT_TARGET` sigue `opencode`). `codex` es un target nuevo reconocido; `vscode` pasa a ser un target explícito de primera clase (distinto de `copilot-vscode`, con paquete propio `@axiom/adapters-vscode` y generador real cableado — antes era invisible a `init`/`workspace-adapters`). El vocabulario está alineado a **fuente única** a través de la lista de targets del CLI (`init.ts`), el registro de allowed-targets del composer de install-profiles (`default-profiles.ts` — `allowedTargets` de AMBOS profiles), la lista headline `axiom.yaml#capabilities.adapters` (8 ids: `claude-code, github-copilot, vscode, opencode, cursor, antigravity, visual-studio-2026, codex`; `copilot-vscode`/`litellm` siguen como targets internos, fuera del headline) y el support-matrix de model-routing. El invariante de reconciliación (los 5 sitios que toda alta de target debe tocar) vive en NFR-AXM-021.

### RF-AXM-050 Generadores de adapter dedicados para codex/antigravity/visual-studio-2026 (`INC-20260726-adapter-generators`)

`codex`, `antigravity` y `visual-studio-2026` ganan paquete `@axiom/adapters-<target>` de primera clase (single-file, merge-preserving, mismo contrato `Result<T, AdapterGeneratorError>` + máquina de 6 estados de preservación `TEAM:CUSTOM` que `@axiom/adapters-claude-code`), reemplazando el fallback thin-canonical: `codex` → `.codex/AGENTS.md`, `antigravity` → `.antigravity/AGENTS.md`, `visual-studio-2026` → `.vs/AXIOM.md` (path sin cambios). Los paquetes de adapter dedicados suben de 6 a **9** (verificado por el check `TC-009`/GATE 0031, ver NFR-AXM-010); el único target declarado en MVP que sigue sin paquete dedicado es `copilot-vscode`. El nivel `'fallback-only'` de `SUPPORT_MATRIX` para estos 3 targets es un eje SEPARADO ("proyección de model-routing per-slot") y queda sin cambios. El helper `writeThinCanonicalAgentsMd` y sus constantes quedan en el código, no referenciados, como fallback disponible para un futuro target sin generador dedicado.

### RF-AXM-051 Paridad de MCP nativo + superficie portable `.axiom` incondicional (`INC-20260726-adapter-mcp-parity`)

La proyección de config MCP nativa (`writeNativeMcpConfig`) cubre ahora todo target con un schema nativo verificado o de asunción documentada, y da una nota informativa honesta al resto — **nunca un schema inventado**. `NATIVE_MCP_TARGETS` (gate de "se escribe fichero de proyecto") = **7 ids**: `claude-code, cursor, copilot-vscode, github-copilot, opencode, vscode, visual-studio-2026`. `vscode` → `.vscode/mcp.json` (mismo fichero/shape que `copilot-vscode`/`github-copilot`); `visual-studio-2026` → nuevo `.vs/mcp.json` (mismo shape stdio, path propio para no colisionar nunca con el `.mcp.json` de claude-code; es una **ASUNCIÓN documentada y override-able**, no re-verificada de forma independiente). Un segundo conjunto, `NATIVE_MCP_INFORMATIVE_TARGETS` = `codex`/`antigravity`: su config MCP real es **user-global** (`~/.codex/config.toml` `[mcp_servers]`; `~/.gemini/config/mcp_config.json` `mcpServers`) — Axiom no escribe fichero de proyecto para ellos, solo una nota informativa nombrando la ubicación real + los servers Axiom lanzables (`sdd-mcp-server`/`spec-mcp-broker`). `litellm` (y cualquier target futuro sin schema) mantiene el fallback genérico. Todo caller que despache sobre `NATIVE_MCP_TARGETS` debe además chequear `NATIVE_MCP_INFORMATIVE_TARGETS` o la nota se vuelve código muerto (hueco encontrado y cerrado en `workspace-worktree-provision.ts`). La superficie portable `.axiom/{agents,commands,skills}/` (`materializeProcessSurfaces`) se confirma adapter-agnóstica e **incondicional** (todo repo la recibe con independencia de los adapters seleccionados, probado con `adapters: []`), de modo que cada adapter es descubrible por ella sin necesidad de un formato de skill nativo por-IDE.

### RF-AXM-052 Paridad de routing de adapters del launcher (`INC-20260726-launcher-adapter-routing-parity`)

`AXIOM_ADAPTER_ROUTING.adapters` (`@axiom/launcher`, la tabla que decide qué adapters puede elegir el launcher web al craftear un prompt) lista ahora **9 entradas** — los 8 adapters headline (`claude-code, github-copilot, vscode, opencode, cursor, antigravity, visual-studio-2026, codex`) + `cli` — todas reusando el mismo `skillRoutingMap()`/`cliRoutingMap()`, de modo que cada acción de cada adapter resuelve al id de skill real (`axiom-sdd-orchestrator` o, para `review-*`, `axiom-phase-reviewer`) + el mcpTool `sdd.transitionApply`, NUNCA el fallback genérico `@axiom` de portapapeles (antes solo 3 de esos ids eran ruteables). Es una tabla DISTINTA del registro `ADAPTER_TARGETS` (aquélla gatea `axiom init --target`/materialización; ésta gatea el picker del launcher). El endpoint `apiGetLauncherData` expone los 9 con un `label` legible (`ADAPTER_LABELS`); el front rendea el selector dinámicamente y prefiere `label` sobre el id. Adapter-neutral: solo difieren las cosméticas de cabecera/mención.

### RF-AXM-053 Enriquecimiento de contexto del prompt del launcher (`INC-20260726-launcher-prompt-context-enrichment`)

Todo prompt que el launcher craftea lleva ahora, de forma adapter-neutral: **DÓNDE leer** (las rutas repo-relativas reales de la carpeta/README/metadata del artefacto en la spec, resueltas con `resolveArtifactDir` + `resolveSpecArtifactRelPath` de `@axiom/workflow` — la MISMA resolución que usa el endpoint de registro, sin esquema de path a mano; o, cuando el id aún no existe, una instrucción clara "id no asignado" en vez de una ruta falsa) y un bloque **"Herramientas y ubicacion"** que nombra los MCP servers gestionados (`AXIOM_MANAGED_MCP_SERVERS` = `sdd-mcp-server`/`spec-mcp-broker`, override-able vía `CraftPromptOptions.mcpServers`), el `mcpTool` de mutación confirmada (`sdd.transitionApply`) y la skill a aplicar — derivados del mismo `RoutingTarget` que la capa de routing ya resolvía (dato existente recién superficializado, no un concepto de routing nuevo). El bloque es byte-idéntico entre adapters skill-routed; el adapter `cli` omite solo la línea de skill (target `command`-kind, sin skillId). Sin duplicación: una ruta literal de spec/metadata aparece a lo sumo una vez por prompt.

### RF-AXM-054 Probes de runtime opt-in en doctor (`axiom doctor --deep`) (`INC-20260726-doctor-runtime-probes`)

`axiom doctor` gana un superset asíncrono OPT-IN, `--deep` (`runDoctorChecksDeep`), que añade probes de runtime reales sobre el árbol síncrono de checks de configuración (sin tocarlo): `TC-018-<toolId>` (probe funcional best-effort por tool declarada, reusando el contrato `resolveProbeCommand`/`probeToolInstalled` de `@axiom/toolchain` — `--version` para serena/cmm/engram; hace `skip` honesto de tools sin contrato de binario como `rtk`/`caveman`, nunca fabrica un probe) y `TC-019-<serverId>` (un handshake `initialize` JSON-RPC MCP **real** contra el `command`/`args` de lanzamiento de cada server gestionado, leído de `.axiom/mcp.yml`, reusando `createStdioMcpClient` de `@axiom/providers` y el handler `initialize` de `@axiom/mcp-server`), categoría nueva `runtime-probes`. El endpoint del gate de doctor del launcher (`GET .../launcher/doctor`) admite el mismo opt-in vía `?deep=1`/`?deep=true`, con el path síncrono rápido como default. Disciplina aditiva/never-fail en NFR-AXM-020.

### RF-AXM-055 Front de onboarding config-rich del launcher (`INC-20260726-launcher-onboarding-config-front`)

Las tarjetas install/join del launcher (`axiom app`) cubren ahora la superficie completa de onboarding — `name`, `path`, `profile`, `overlay`, `layout`, rol de este repo, adapter primario + adapters adicionales, tools y execution-mode — con el gate preview→confirm preservado exactamente. Todo parámetro con ruta real invocable queda **realmente cableado**: `runInit` (scaffold base) → `generateWorkspaceAdapters` (adapter primario + adicionales, salida real por-adapter — cierra el hueco de que un install nunca materializaba output por-adapter) → `runConfigure` cuando se da execution-mode (best-effort; el fallo se reporta en `executionMode.warning`, nunca hace rollback de lo ya aplicado). Las **tools** se superficializan honestamente con una nota "pendiente" explícita (`tools.applied: false`, apuntando a `axiom toolchain add --id <id>`) — NO se cablean ni se fingen aplicadas, porque no existe hoy una ruta limpia de instalación de toolchain desde un `init` recién hecho (diferido). La asignación de roles de equipo/código es alcanzable en el flujo de join reusando el endpoint existente `apiLauncherRolesAssign` (auto-select del proyecto recién onboardeado, sin endpoint nuevo). `ADAPTER_LABELS` pasa a un fichero compartido `apps/cli/src/commands/_adapter-labels.ts`, cubriendo la UNIÓN de `ADAPTER_TARGETS` y `AXIOM_ADAPTER_ROUTING`.
### RF-AXM-057 Errores tipados con `code` estable para recovery determinista (`INC-20260730-typed-recovery`)

`@axiom/core` exporta `AxiomError` (subclase de `Error` con un campo `code: string` de solo lectura) y `AXIOM_ERROR_CODES`, un **catálogo cerrado y documentado** cuyas entradas están todas ancladas a un throw site real — no se enumeran códigos especulativos. Los subagentes y el orquestador ramifican sobre `error.code` en vez de hacer message-matching sobre `error.message` (frágil, pensado para humanos). La migración cubre los dos throws crudos de `packages/workflow` (`generateUniqueArtifactId` agotado, variable de template ausente en `renderBranchName`), `GateFailureError` de `packages/orchestrator`, y **37 de los 62** throw sites de `apps/cli` — los 25 restantes quedan deliberadamente sin tipar y **enumerados uno a uno con su razón** en el README del incremento (wrapping de YAML externo, errores ya estructurados como `Result`, invariantes internos no user-facing, utilidad de búsqueda de puerto). `AxiomError.code` se mantiene tipado como `string`, no como el union cerrado `AxiomErrorCode`, para no forzar a subclases externas preexistentes al catálogo. **Ningún mensaje de error visible cambió**: toda migración es `throw new Error(msg)` → `throw new AxiomError(CODE, msg)` con el mismo `msg` literal.

Bug corregido en el mismo incremento: el constructor de `AxiomError` hacía `Object.setPrototypeOf(this, AxiomError.prototype)` de forma incondicional, lo que **pisaba silenciosamente el prototype chain de CUALQUIER subclase** (incluida `GateFailureError`, ya en producción) y rompía `instanceof SubclaseError`. Corregido a `new.target.prototype`.

### RF-AXM-058 Evidencia obligatoria y fail-closed en memoria persistente (`INC-20260730-engram-evidence`)

Persistir una `MemoryEntry` a través de `@axiom/memory` exige `rationale` (por qué se registró) y `source` (de dónde viene) con **longitud > 3 tras `trim()`**; las cadenas vacías o de solo espacios cuentan como ausentes. El rechazo es fail-closed y ocurre **antes de cualquier I/O** (escritura a disco o llamada MCP a engram), devolviendo la variante nueva `MemoryError { kind: 'missing-evidence', field: 'rationale' | 'source' }` — se extiende el idioma `Result` que los callers ya manejan en vez de lanzar. El guard `validateMemoryEvidence()` (puro, en `packages/memory/src/evidence.ts`) se invoca desde **tres puntos** — `saveMemory()`, `createInMemoryBackend().save()` y `createEngramBackend().save()` — porque `MemoryBackend` es una interfaz duck-typed exportada y varios tests y callers invocan `backend.save()` directamente; enforcement en un único sitio sería evitable eligiendo otro backend.

Fuera del gate por diseño, documentado: la **ruta de lectura/recall** (`searchResultToEntry`/`buildEntryFromSearchResult`) sigue reconstruyendo entries con `rationale`/`source` vacíos cuando el resultado de búsqueda no los trae — endurecerla rompería `load()`/`query()` sobre entradas preexistentes sin evidencia. `saveSessionSummary` queda exento: `MemorySessionSummary` no tiene esos campos y su entry interna usa literales fijos que el caller no controla.

### RF-AXM-059 Receipts de fase emitidos automáticamente (`INC-20260730-phase-receipts`)

`writePhaseReceipt` (`@axiom/workflow`) deja de ser alcanzable solo por el comando manual `axiom phase receipt`: **toda transición real de ciclo de vida emite ahora su receipt JSON automáticamente**, tanto en éxito (`exitCode === 0` → `success`) como en fallo (transición inválida, `verify` bloqueado por functional-verify, re-`create` rechazado → `failure`). El wiring vive en los dos únicos choke points, `runIncrementSubcommand` y `runBugSubcommand`, convertidos en **wrappers públicos delgados** sobre un `…Core` privado que conserva la lógica byte-idéntica: el core calcula el resultado, el wrapper emite el receipt y devuelve el resultado **sin modificarlo**, satisfaciendo por construcción el non-goal "no cambiar cómo se ejecutan las fases".

Tres reglas de emisión, todas deliberadas: (1) el `phase` del receipt es el **nombre real de la transición** (`increment-verify`, `bug-archive`…), el mismo `command` de `workflows.yaml` — no un mapeo inventado al vocabulario `design/tasks/apply/...` que no existe en la CLI; (2) `verify --preview`/`--dry-run` **no** emite receipt, porque no se aplicó ninguna transición y afirmar que una fase corrió sería falso; (3) la escritura es **best-effort/never-block** — cualquier excepción se reporta por stderr y jamás altera `result`/`exitCode`, misma postura que los hooks y `archiveArtifactDir`.

La carpeta destino prioriza `vars.metadataId` (la carpeta real folder-per-artifact) sobre el `--id` humano, y detecta el caso `archive`, donde la carpeta ya fue movida físicamente a `_archive/` antes de que corra el wrapper.

### RF-AXM-060 Freeze de candidate como gate de `apply` (`INC-20260730-candidate-freeze`)

`axiom freeze --increment <id>` congela un candidate escribiendo `candidate-freeze.json` en la carpeta del incremento, con `hash` combinado sha256 sobre `memoryHash` (entries de memoria filtradas por `entry.increment === <id>`) y `specsHash` (el `README.md` del incremento). La API de validación `checkCandidateFreeze(incrementId, cwd) → { ok, reason? }` está **cableada al ciclo real**: `axiom-increment` la consulta antes de delegar un apply, y devuelve `ok: false` con razón legible cuando falta el freeze o cuando los inputs mutaron respecto del hash congelado.

**Limitación documentada, no cerrada**: pese a que el Scope original hablaba de congelar "memoria local, repo specs y **dependencias**", `hashCandidateInputs` hashea únicamente memoria filtrada + `README.md`. No cubre lockfiles (`package-lock.json`, `toolchain.lock`) ni otros artefactos del incremento como `metadata.yml`. Ampliar el algoritmo invalidaría silenciosamente todo artefacto ya congelado del repo de spec, por lo que se registra como limitación explícita en vez de sobre-declarar cobertura.

### RF-AXM-061 Validación fail-closed de `profiles.yaml` contra el schema canónico (`INC-20260730-exact-scope`)

Los **tres** loaders ad-hoc de `profiles.yaml` de `apps/cli` — `roles.ts#loadProfilesYaml`, `init.ts#tryLoadProfilesYaml` y `topology.ts#loadProfilesYamlForValidation`, cada uno con su propio `fs.readFileSync` + `yaml.load` + chequeo de shape a mano — convergen en `validateInstallProfilesYamlContent()` (`@axiom/config-validation`), que valida contra el `InstallProfilesYamlSchema` **ya existente y hasta ahora sin consumidores**. `apps/cli` pasa a depender realmente de `@axiom/config-validation` (el path de tsconfig y la project reference ya existían; faltaba la entrada en `package.json`). Un fichero presente pero estructuralmente inválido es ahora fail-closed con `AxiomError(AXIOM_INVALID_CONFIG)` y error por campo.

Matiz preservado deliberadamente por call site: **fail-closed significa rechazar input inválido, no volver fatal un fichero opcional legítimamente ausente**. Los dos loaders que devolvían `null` ante ausencia lo siguen haciendo, y `topology.ts` mantiene su fallback a `DEFAULT_PROFILES` (contrato de `INC-20260710-dynamic-team-roles`). Solo cambia el tratamiento del fichero presente-pero-inválido.
