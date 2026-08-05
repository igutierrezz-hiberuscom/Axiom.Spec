# 07 Gobierno y Seguridad

## Gobierno

1. una única fuente de verdad por capa;
2. no mezclar runtime, spec y tooling en la misma responsabilidad (regido por `Axiom.SDD/AGENTS.md` para este workspace);
3. no promover estructuras legacy como canónicas por comodidad;
4. límites explícitos de bootstrap (`Axiom.SDD/AGENTS.md`): no introducir sin pedido explícito índices obligatorios, sistemas de metadata complejos, carpetas de lifecycle enterprise, integraciones de work items (Azure DevOps/Jira) como dependencia obligatoria, abstracciones MCP obligatorias, lógica de Workbench, frameworks pesados de multi-agente, jerarquías autogeneradas profundas, ni scripts que no existan aún.

### Ownership documental de decisiones

`Axiom.Spec/decisions/` es el hogar canónico de los ADR y decisiones estructurales del workspace. Las referencias activas deben apuntar a esa ruta; `Axiom/docs/` puede conservar documentación operativa o histórica, pero no es el hogar actual de esos ADR. Esta regla es independiente de `Axiom/axiom.spec/`, que sigue siendo baseline product-owned del runtime y no una segunda fuente canónica de la spec (ADR-0032).

## Seguridad operativa y compliance (verificado en runtime)

1. **GATEs verificables por doctor**: GATE 0010 (overlay `enterprise` exige `audit-trail-sink`), GATE 0031 (9 adapter packages deben tener `src/generator.ts` + `dist/index.js`), GATE 0024 (memoria no es fuente de verdad; spec prevalece en conflicto), GATE 0033 (agents como contratos materializables, sin ejecución).
2. **Aislamiento project-scoped**: `@axiom/isolation` aplica path-guard y una lista de MCP servers permitidos por defecto; ninguna cache, binding o entrada de memoria debe cruzar `projectId`. El server MCP ejecutable **hace cumplir** este aislamiento a nivel de handler, no solo por convención — ver "Aislamiento MCP por proyecto (enforced)" más abajo.
3. **Audit trail**: `axiom audit` verifica SHA-256, conteo de líneas, retención y detección de reescritura externa, devolviendo `compliant` | `absent` | `violation` (exit 1 en violación).
4. **Telemetría por overlay**: `telemetry-sinks.yaml` define `dataSensitivityBoundaries` por overlay (tags permitidos/redactados, nivel de redacción, flujo cross-project, ventana de retención) y una lista de sinks (`null-sink`, `log-sink`, `remote-sink`, `audit-trail-sink`). `axiom sync` aborta antes de mutar si el overlay exige señales mínimas y no hay sink habilitado que las cubra. No hay documentado un mecanismo formal de opt-out completo de telemetría (verificado como ausente, no como implementado).
5. **Doctor como gate de gobierno mínimo**: valida presencia y validez de `axiom.yaml`, `integrations.yaml`, `policy-as-code.yaml`, protección de `.axiom-state/local/` frente a versionado accidental, y aislamiento project-scoped.
6. **Policy-as-code**: `policy-as-code.yaml` concentra `sensitivityTags`, `artifactLifecycle` (transiciones y verificación antes de archive), reglas de `tools`/`compliance` (actitud ante herramientas faltantes o MCP no aprobados), `projectIsolation` y `doctorValidation`.

## Seguridad de launcher, plugins y superficies sustituidas (2026-08-04)

- El launcher es una capa fina sobre CLI/workflow: onboarding, adopción,
	lifecycle y plugins no duplican escritores ni máquinas de estados.
- Las acciones de plugins solo pueden ejecutarse mediante handlers registrados
	en una allowlist estática. El campo declarativo `command` es una etiqueta de
	compatibilidad; nunca se interpreta como shell, script, path o permiso.
- El catálogo HTTP usa una proyección explícita y elimina propiedades
	desconocidas, `sourcePath`, credenciales y opciones no declaradas. Los
	valores de fields se validan contra tipo, required, opciones y campos
	permitidos antes de alcanzar tracker, filesystem o red.
- Las mutaciones local/external requieren preview y `confirmed: true`; el
	default Azure DevOps usa `NullTracker` sin red. Mensajes de proveedor y
	`externalRefs` se redactan antes de volver a la UI.
- La antigua TUI pública y su acción implícita fueron retiradas tras una
	matriz de paridad. La CLI headless, el launcher y MCP son las superficies
	vigentes; los runners compartidos no se eliminan por retirar una interfaz.

## Reglas de cierre (incrementos y bugs, vigentes en este workspace)

Un incremento o bug solo puede marcarse `closed` si: el objetivo/comportamiento esperado es claro, existen acceptance criteria, se implementaron cambios (o hay justificación explícita de no-code), se ejecutó la validación disponible, se revisó contra el intent original y los acceptance criteria, y se integró conocimiento estable en `Axiom.Spec`. Si falta cualquiera de estos puntos, el estado debe quedar `pending` con motivo explícito — nunca `closed` por comodidad.

## Límite de dogfooding (check `DF-001`) — roadmap de rediseño, cerrado

Regla: "Axiom se desarrolla con Axiom, pero Axiom no contiene su propia factoría interna como parte del producto instalable."

- `runDogfoodingBoundaryChecks` (`Axiom/packages/doctor/src/checks.ts`, id de check `DF-001`, categoría `dogfooding`) comprueba que ningún repo con rol `code` (`TopologyManifest.roleCodeRepositories`) tenga una dependencia física — una dependencia local de `package.json` (`file:`/`link:`/path relativo) o un literal de path `require`/`import` — que resuelva dentro del path de filesystem resuelto del repo con rol `sdd` o `spec`. Estrictamente unidireccional: que `sdd`/`spec` referencien a `code` para contexto es esperado y está fuera de alcance; solo se marca `code -> sdd`/`code -> spec`.
- Parametrizado por rol por diseño, no hardcodeado por nombre: se apoya en `sddRepo`/`specRepo`/`roleCodeRepositories` de `TopologyManifest`, así que funciona de forma significativa sobre los nombres de repo de cualquier proyecto de terceros, no solo sobre el layout de este workspace.
- Postura fail-open, con muchos `skip` (sin estado `warn`): se salta cuando el proyecto no está `resolved`, cuando `topology.yaml` falla al cargar, cuando `manifest.mode === 'single-repo'` (no es posible ningún límite cross-repo), o cuando `roleCodeRepositories` está vacío. Un `fail` requiere un match real y concreto de path resuelto; si no, `pass`.
- Escaneo mínimo suficiente (sin parseo AST, sin resolución de alias de bundler, sin recorrido transitivo de `node_modules`): un escaneo de dependencies/devDependencies/optionalDependencies de `package.json` acotado a los globs `workspaces` declarados propios del repo, más un grep acotado de literales de path `require(...)`/`from '...'`, excluyendo paths generados conocidos vía `aggregateKnownGeneratedPathGlobs`.
- Este propio repo `Axiom` no tiene un `axiom.config/topology.yaml` explícito, así que `DF-001` reporta `skip` (no `pass`) al correr aquí hoy — un hueco esperado e intencional ligado a que este workspace todavía no se declara a sí mismo como `mode: multi-repo`, no un defecto. Levantar un `topology.yaml` real para este workspace queda como trabajo futuro diferido (reflejo de D1, ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)).

## Checks de doctor — categorías establecidas por el roadmap de rediseño

`@axiom/doctor`'s forma `DoctorCheck { id, category, description, status, evidence }` (`pass | fail | warn | skip`) se usa de forma consistente en toda categoría. Doctor es **puramente diagnóstico** — detecta y reporta, nunca muta el filesystem (confirmado por lectura directa: cero rutas de código `fix`/`repair`/`autofix` en `checks.ts`). Categorías añadidas por este roadmap, sumadas al conjunto preexistente (boundaries, policies, manifests, isolation, capability-model, install-profiles, gateway, tool-routing, topology, coherencia de QA-lane):

- `MC-001`/`BC-001`/`BC-002` — checks de manifest/boundary, conscientes de versión para `axiom.yaml` `schemaVersion: 1` y `2`.
- `WS-001` (categoría `write-scope`) — valida el plan activo contra `allowedWriteScope`; se salta limpiamente cuando no hay ningún plan activo (ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md) para la primitiva `validateWriteScope`, que es la misma que consume `axiom validate changes`).
- `IX-001` (categoría `index`/`artifacts`) — confirma que todo `metadata.yml` bajo `{increments,bugs,plans,adr,decisions}/*/` parsea correctamente, vía `listArtifacts`; se salta limpiamente si el scope de spec no resuelve.
- `TC-010` — obligatorio: falla cuando `skills-catalog.yaml` está ausente.
- `TC-012` (`skills-role-index-validity`) — opcional: se salta cuando `skills-index/` está ausente.
- `TC-013` (`technical-context-index-validity`) — opcional: se salta cuando `technical-context/indexes/` está ausente.
- `TR-001..004` — smoke-test de `routeTool` vía fixture en memoria.
- `DF-001` (categoría `dogfooding`) — ver arriba.

`CC-004` no valida únicamente el subconjunto que ya aparece en
`providers.yaml`: compara las 16 capabilities provider-routed canónicas con
su declaración en `capabilities.yaml`, su estado, su clase de cumplimiento y
los providers que las sirven. Las capabilities MCP-only `axiom.*` se
comprueban en la superficie MCP y no se consideran huérfanas del registry
tradicional. La severidad es `fail` para requeridas activas sin provider,
`warn` para opcionales o post-MVP sin provider y no bloqueante para
`disabled`/`unavailable`, siempre con evidencia visible.

### Gobierno del versionado de toolchain (`INC-20260730-toolchain-versioning`)

Las checks `TC-020..TC-023` hacen visible el estado de reproducibilidad sin convertir una dependencia externa ausente en una mutación automática:

- `TC-020` valida la presencia y forma del lockfile; su ausencia es una condición informativa para un proyecto que todavía no ha fijado tools.
- `TC-021` compara cada versión locked con el canal y la versión declarados por el catálogo; una incompatibilidad es warning.
- `TC-022` conserva en la ruta síncrona la indicación de que hace falta un probe real y, en `doctor --deep`, compara la versión instalada contra la locked. El deep check es never-fail: un binario ausente, una versión no obtenida o una diferencia producen warning/skip, no fail.
- `TC-023` valida que cada entrada locked tenga versión y uno de los canales `stable`, `candidate` o `edge`.

El lockfile es estado local project-scoped, su escritura es atómica y el upgrade usa checkpoint/rollback. La política de gobierno prohíbe presentar `plan` o `upgrade` como instaladores: Axiom no descarga, sustituye ni hace rollback de binarios externos.

Todo check de doctor futuro para validez de `mcp.yml`, o para el registro de capability/provider de `@axiom/mcp-tools`, queda diferido. El setup de workspace multi-repo (`runWorkspaceSetup`, INC-20260705-workspace-mcp-generation) ya escribe un `mcp.yml` real a un proyecto, pero valida en línea reusando `validateMcpProjectConfig` (no vía un check de doctor); no existe todavía instalador/scaffold que escriba `capabilities.yaml`/`providers.yaml` a un proyecto real, así que para esos no hay call site de producción que un check de doctor deba comprobar.

## Trazabilidad de write-scope y ownership (ángulo de gobierno)

`validateWriteScope` (ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md)) es el mecanismo de gobierno que impide que un plan mute paths fuera de su `allowedWriteScope` declarado, o que repos con rol `sdd` reciban cambios no declarados (`unexpected-sdd-change`), o que se manipulen paths generados/cache conocidos (`generated-cache-tampering`). Es el mecanismo concreto que hace cumplir en runtime la separación de ownership entre repos que exige esta sección de gobierno — dos superficies (`axiom validate changes` y el check `WS-001`) comparten una única primitiva sin lógica duplicada.

Hasta `INC-20260710-plan-role-split` (P1-5), esta primitiva era genérica pero no tenía nada real que hacer cumplir en la práctica: `axiom-plan create` siempre dejaba el `allowedWriteScope` del plan vacío. Cerrado: `axiom-plan create` ahora deriva el role-split del plan de `topology.yaml#roles`/`#assignments` (o de un `--roles` explícito) y puebla `targetRepos`/`allowedWriteScope` con los repos que esos roles poseen — ver "`PlanMetadata.roles`" en [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md). `validateWriteScope` en sí no cambió; el gap era exclusivamente que el producer nunca poblaba lo que el consumer ya sabía leer.

Regla de ownership complementaria en el scaffolding: **guarda no-clobber del setup de workspace** (`runWorkspaceSetup`, INC-20260705-workspace-multirepo-setup-engine). Al parametrizar un directorio existente, el motor nunca sobrescribe un `axiom.yaml` preexistente que pertenece a OTRO proyecto: si el fichero parsea con un `projectId` distinto (o es v1/no-parseable-pero-presente), su escritura se salta y se registra como warning, nunca como error — Axiom no reclama la propiedad de un repo ya gobernado por otro proyecto. Ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md).

Los pasos aditivos de la segunda tanda de setup de workspace (INC-20260705-*) heredan la misma postura de seguridad de ownership: todos son **best-effort** (un fallo de generación de adapter, de baseline de skills o de base de spec nunca hace fallar el setup global — se degrada a warning) y ninguno clobbera contenido preexistente. La baseline de skills y la base de spec están además **gateadas por creación**: solo corren cuando el repo destino (control para skills, spec para la base) se crea recién en la misma llamada; un repo ya existente se salta silenciosamente. La base de spec aplica adicionalmente un guardado per-file (skip + warning si el fichero destino ya existe), de modo que jamás reescribe una spec o un contexto técnico que un repo ya tuviera. Las autoskills por repo de código (round 3, `INC-20260705-workspace-code-repo-skills`) heredan la misma postura: best-effort por repo (el fallo de un repo no salta los demás), gateadas por el `created` propio de cada repo de rol, sin clobber de un repo preexistente. Ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md) y [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

Robustez de registro sin pérdida de ownership ni de datos (round 3, `INC-20260705-workspace-setup-registry-robustness`): un fallo de registro o un registry legado v1 sin migrar **ya no aborta silenciosamente el install** — el scaffolding local completa siempre, best-effort, y `registryRegistered` refleja la realidad sin ocultar un install medio-completado. La auto-migración v1→v2 (`migrateLegacyRegistryV1ToV2`) **nunca borra datos de usuario**: preserva el `registry.json` legado renombrándolo a `registry.json.migrated` en vez de eliminarlo, de modo que la evolución de schema es aditiva y reversible por el usuario. Ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md).

## Aislamiento MCP por proyecto (enforced) — INC-20260708-mcp-project-isolation-hardening

El server MCP ejecutable (`@axiom/mcp-server`, lanzado por proyecto vía `axiom mcp serve --kind <sdd|spec|memory|axiom> --project-root <path>`, ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md)) ahora **hace cumplir** el aislamiento project-scoped a nivel de código, no solo por convención. Esto es una garantía de ownership/seguridad de aislamiento equivalente a la que `@axiom/memory` ya aplica sobre `projectId` (GATE 0024), llevada a la capa de input-builders del MCP.

- **Vulnerabilidad cerrada (fuga cross-project)**: los input-builders por capability resolvían la identidad de proyecto con el patrón `str(args,'projectId') ?? context.projectId` (y equivalentes para `projectRoot`/`rootPath`/`specScopeAbsolutePath`), de modo que un llamador (el agente conectado al server del proyecto A) podía pasar un `projectId`/`projectRoot` ajeno en los argumentos de `tools/call` y leer los datos del proyecto B a través del mismo proceso servidor. Era una fuga real cross-project (datos, existencia de proyectos, nombres, paths de repos).
- **Pinning al `context` propio**: cada campo que identifica proyecto (`projectId`, `homeDir`, `projectRoot`, `specScopeAbsolutePath`, `sddScopeAbsolutePath`) se **fija (pin)** al `context` que el server resolvió al arrancar (`resolveMcpServerContext({ projectRoot })`). El valor del llamador ya no se usa nunca verbatim: si lo omite, se usa el del `context`; si lo pasa igual, la llamada procede; si lo pasa **distinto**, se rechaza.
- **Rechazo con `isError`**: un valor cross-project que difiera del `context` devuelve un `tools/call` con `isError: true` y el mensaje `"Cross-project access blocked: this MCP server is scoped to project '<id>'. Launch the MCP server for the target project instead."`. El rechazo solo dispara ante un **mismatch real** contra un `context` resuelto — si `context.projectId` no resuelve (proyecto no registrado), no hay opinión de contexto con la que entrar en conflicto y se conserva el contrato preexistente de "campo requerido faltante".
- **`sdd.projectRegistryRead` acotado al proyecto propio**: esta capability ya no enumera el registro machine-wide (`listProjectsV2`); su salida se filtra a la entrada cuyo `id === context.projectId` (o `[]` si no resuelve). Un server project-scoped enumerando proyectos ajenos era en sí mismo una fuga.
- **Selectores dentro-de-proyecto intactos**: los argumentos que nombran algo INTERNO al proyecto ya fijado (`id`, `planId`, `skillId`, `specRelPath`, `role`/`roleOrKind`, `taskTags`, `includeStale`, `contextBudget`, `targetRepoId`) siguen siendo caller-supplied, sin cambios funcionales.
- **Alcance y límites**: los artefactos de config en disco ya eran project-scoped; el registro es metadata-only. La corrección es completa a nivel de handler (no queda ninguna ruta donde un identificador de proyecto del llamador se use verbatim para leer datos). El guard de aislamiento en el arranque vía `@axiom/isolation` (`checkMcpAllowed`/`assertProjectIsolation`) se **difirió** deliberadamente: requeriría sintetizar un `ProjectResolution` a partir de un `McpServerContext` (adapter especulativo) y no es necesario para cerrar la vulnerabilidad confirmada — el pinning a nivel de handler es el fix no negociable y suficiente.

## Postura de gobierno de la tanda INC-20260708-* (auto-validación, LOCAL-only, aislamiento preservado)

- **Auto-validación del producto: estado histórico y revalidación R-04** (`INC-20260708-product-repo-self-bootstrap` + `INC-20260708-fix-longstanding-test-failures`): aquella tanda dejó la baseline canónica de `Axiom/` materializada y registró doctor/readiness y suite verde. El registro del 2026-07-30 quedó superado: entonces `npm run doctor` y `npm run readiness:first-project` fallaban en `TC-011` por un `bundleHash` stale de `axiom-reviewer`, y la ejecución global independiente del review reportó 3425/3427 tests; ambos hechos pertenecen a fotografías históricas. El estado de 2026-08-02 también queda como histórico: desde R-04, `CC-004` devuelve `FAIL` de forma intencional y explicada porque el repo dogfooded sirve 7/16 capabilities provider-routed; `readiness:first-project` hereda ese fallo. Ver [00_Resumen_Ejecutivo.md](00_Resumen_Ejecutivo.md), [02_Requisitos_No_Funcionales.md](02_Requisitos_No_Funcionales.md) y [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).
- **Postura LOCAL-only / bbdd-local en toda la capa de providers**: la restricción no-negociable del operador ("todas las herramientas siempre en local con bbdd en local") se hace cumplir por construcción en `@axiom/providers` — el guard `LOCAL_ONLY`/`isLocalTarget` rechaza/degrada cualquier config de provider que apunte a un endpoint no-local, y los clientes de code-intel (`cmm`/`serena`) y el backend de memoria (`engram`, SQLite local vía MCP stdio) se lanzan siempre como procesos LOCALES. No existe ninguna ruta de ejecución de provider remota/cloud en el seam. Una tool local ausente degrada limpiamente (`not-installed`), nunca es un `fail` de gobierno (`PS-001` del doctor es `warn`, no `fail`, para "habilitado pero no instalado" — una tool de dev ausente es un hecho de entorno, no un defecto del proyecto). Ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).
- **Aislamiento project-scoped preservado en todas las superficies nuevas**: GATE 0024 (memoria) se conserva y se EXTIENDE al nuevo backend engram con un segundo pin a nivel de proceso (`engram mcp --project=<projectId>`), y el input-builder `memory.*` fija SIEMPRE `projectRoot` al `context` del server (nunca lee un id/path del llamador — un pin más estricto que el de `sdd.*`/`spec.*`, porque la memoria no tiene ningún caso de uso legítimo cross-project). El aprendizaje continuo hereda el aislamiento gratis (todo persist/recall pasa por un `MemoryBackend` scope-bound). La selección de providers, las reglas, las autoskills y las operaciones incrementales son todas project-scoped (resuelven el proyecto desde cwd/`workspace.json`) y best-effort/no-clobber, heredando la misma postura de ownership (nunca reclaman ni clobberan un repo gobernado por otro proyecto). Ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md) y [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md).
- **Sin arquitectura especulativa** (límites de bootstrap): esta tanda rechazó explícitamente el motor de instintos de ECC (scoring/herencia/promoción), un motor de hooks de sesión propio, y la fabricación de instrumentación de tool-calls en vivo — el aprendizaje continuo y los delegation triggers son deterministas/puros sobre inputs reales o explícitos, y los hooks de sesión quedan como snippet documentado opt-in, nunca auto-aplicado.

## Regla conocida de build (no duplicada aquí)

`Axiom/packages/cli-commands/tsconfig.json` tiene un defecto de tooling de build que rompe `--help` para varios comandos CLI transitivamente dependientes de ese paquete. Rastreado íntegramente en `Axiom.Spec/bugs/BUG-20260702-cli-commands-tsconfig-missing-emit` (status: pending) — no se duplica el detalle aquí.

## Onboarding/ADO desde el launcher: mutación segura y secreto fuera del repo (2026-07-15) — tanda INC-20260715-*

- **Mutación confirm-gated en todas las superficies nuevas del launcher**: los endpoints de onboarding (`install`/`join`/`roles register`/`roles assign`) y el puente ADO en creación NO mutan sin `confirmed:true` (preview primero), heredando el contrato de `/launcher/execute`. El explorador de carpetas y la detección de tracker son read-only, best-effort y no-crash (nunca tumban el server ni el front). `INC-20260715-launcher-onboarding`, `INC-20260715-launcher-ado-bridge`.
- **PAT de Azure DevOps nunca en el repo**: el token de acceso personal se resuelve en runtime (variable de entorno declarada en `tracker.json#auth.patEnvVar`, o env de usuario Windows, o `SecretStore` bajo `axiom.ado.<org>.<project>.pat`, o prompt interactivo persistido en el store); nunca se escribe en `tracker.json` ni en ningún fichero versionado. El plugin sólo se considera configurado con `kind:'ado'` + `enabled` + org + project (`isRealTrackerRequested`); cualquier otra combinación degrada a local-only sin red. Detalle operable en [manuales/12_Plugin_Azure_DevOps.md](manuales/12_Plugin_Azure_DevOps.md); capacidad en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).
- **Sin arquitectura especulativa** (límites de bootstrap): el puente ADO en creación NO acopla el ciclo de vida del incremento/bug (no auto-crea; ofrece un paso confirm-gated de un clic) y no modifica los paquetes de tracker; el tuning de agente es prompt-shaping puro (no toca model-routing/providers). Azure DevOps se mantiene como plugin opcional y no bloqueante.

## Gates de revisión, QA y seguridad instaladas (2026-07-15) — tanda INC-20260715-*

- **Revisión por fase de solo lectura** (`axiom-phase-reviewer`): revisa spec/plan/código y devuelve OK|KO con ledger de hallazgos; NUNCA escribe código/spec, NUNCA hace `commit`/`push`. Es un gate de calidad entre fases, no un flujo de autoría.
- **Revisión de seguridad defensiva y no bloqueante** (`axiom-security-reviewer`, ya con cuerpo real): solo-lectura, defensiva (sin exploits ni ejecución de código), **ofusca cualquier secreto** (p. ej. `AKIA****XXXX`) y no expone PII; checklist de 10 familias de riesgo + severidad. Es un gate OPCIONAL: un hallazgo no bloquea el ciclo por defecto, lo señala.
- **QA-validation trazable** (`axiom-qa-validator`): deriva casos de los criterios de aceptación con trazabilidad 1:N, marca `⚠️ SIN COBERTURA` y no da por validado con huecos — refuerza la trazabilidad criterio→prueba sin inventar escenarios.
- **Consolidación atómica y confirm-gated** (`axiom-spec-integrator`): la integración a la spec canónica y el archivado son todo-o-nada y confirm-gated; nunca pierde información ni inventa; no hace `commit`/`push`. `axiom-tech-context` solo DETECTA spec-drift, no decide.
- **Disciplina de write-scope reforzada**: `axiom-role-planner` declara un `allowedWriteScope` mínimo por rol que `axiom-phase-reviewer` (lente code) y las validaciones de cambios contrastan; escribir fuera de scope es un hallazgo bloqueante.

## Gobierno del modelo `<project>.axiom`, worktrees y stack externo (2026-07-24) — tanda INC-20260724-*

Postura de gobierno/seguridad de la graduación a *full product lifecycle*. Formas de datos en [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md); flujos en [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md).

- **Legacy intacto por construcción + provenance para salir de Axiom** (`INC-20260724-adopt-creates-axiom-repo` / `-provenance-lifecycle-manifest` / `-export-eject-rollback`): Axiom **nunca escribe** en los repos legacy (adopción crea un `<project>.axiom` nuevo; probado byte-for-byte en tests). La trazabilidad de ownership se persiste en `metadata.yml` (`origin.source` migrated|axiom-native, `lifecycle.state` migrated|migrated-and-modified|axiom-native, `managedBy`, `exportPolicy.rollbackEligible`) + el manifest global `<project>.axiom/migration/migration-manifest.yaml`, de modo que `axiom eject` puede volcar exactamente lo que Axiom creó/modificó (rollback-eligible) **sin tocar legacy** (default dry-run).
- **Sin hooks git obligatorios** (`INC-20260724-cmm-replaces-graphify-codegraph` / `-rtk-skill-invoked` / `-sdd-artifact-freshness`): ninguna capacidad de esta tanda instala un hook git. El auto-sync de `cmm`, el uso de RTK y el auto-fetch de freshness de artefactos son todos **on-demand y best-effort**, nunca cableados a pre-commit/post-checkout ni a ningún hook.
- **Exclusiones never-compress** (`INC-20260724-rtk-skill-invoked`): la skill `axiom-terminal-output-efficient` prohíbe explícitamente comprimir con RTK contenido crítico — memoria Engram, la spec y sus increments/bugs, ADRs/decisions, y evidencia de compliance/seguridad — y prohíbe invocar RTK desde un hook o wrapper global. El contrato es la propia skill (mismo precedente que las demás "Reglas absolutas" del catálogo, no enforced por doctor).
- **Higiene de supply-chain de AutoSkills** (`INC-20260724-autoskills-lock-hygiene`): las skills instaladas por AutoSkills son distinguibles (`provenance: autoskills`) y fechadas (`installedAt`), y pasan por un gate de policy allow/deny/licencia (`axiom.config/autoskills-policy.yaml`, opcional, default allow-all; degrada sin bloquear). Una skill denegada se salta con razón visible; corre solo por-code-repo en install-time.
- **Aislamiento por worktree y no-filtración de secretos** (`INC-20260724-worktree-provisioning` / `-worktree-provider-isolation`): el provisioning es **portable-only** — copia exactamente 3 ficheros no-secretos (`init.json`/`install-profile.json`/`workspace.json`); `.axiom-state/local/**` y cualquier otro proyecto **nunca** se leen ni copian (probado con un scan recursivo de un "secreto" plantado). Cada worktree tiene su propio índice/caché de code-intel (nunca un grafo mutable compartido); `teardownWorktreeCodeIntel` solo borra el estado derivado del worktree indicado y **nunca** toca el índice del repo principal.
- **Cleanup seguro del worktree** (`INC-20260724-worktree-harvest-cleanup` / `-worktree-close-correctness`): orden estricto **kill → harvest → teardown → remove**; harvest SIEMPRE precede a cualquier borrado (los datos harvesteados sobreviven al borrado del worktree). Un worktree con trabajo real sin integrar es **hard stop** — nunca se fuerza por defecto. El cierre neutraliza solo los ficheros que el propio provisioning generó (registrados en `Execution.provisionedPaths`) antes del dirty check, de modo que trabajo genuino sigue bloqueando; si el cierre hace hard-stop, un rollback compensatorio evita dejar el rol `archived` junto a un worktree huérfano.
- **Push acotado, nunca repo-wide** (`INC-20260724-sdd-artifact-freshness`): la escritura de artefactos SDD hace `git add -- <paths>` acotado a la carpeta del incremento/bug (nunca `git add -A`) — cada worktree/ejecución empuja solo lo suyo, sin arrastrar otros artefactos.
- **Aislamiento MCP preservado en el broker unificado** (`INC-20260724-unified-axiom-mcp`): el broker `axiom` mantiene los pins project-scoped por-campo de los brokers previos (cada input-builder fija `projectRoot`/`specScopeAbsolutePath` al contexto del server); el subconjunto de escritura es exactamente `sdd.transitionApply` + `sdd.gitCommitSync` (no expone `sdd.gitRoleBranch`).
- **Genericidad sin fuga de gobierno**: todo el contenido nuevo es adapter/stack-agnóstico; la profundidad específica de cada proyecto se inyecta como DATO por proyecto (`skills-index/<role>.yaml` + contexto técnico), no en el producto — sin hardcodear reglas de un stack ni credenciales. Ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md) y [manuales/13_Skills_Agentes_y_Roles.md](manuales/13_Skills_Agentes_y_Roles.md).
## Gobierno verificable de flujos desatendidos (2026-08-02) — tanda `INC-20260730-*`

La tanda cierra el bloque de gobierno de la ejecución desatendida sobre tres gates y un catálogo de errores (detalle funcional en RF-AXM-057..061, propiedades en NFR-AXM-023):

- **Gate de evidencia** — `@axiom/memory` rechaza fail-closed cualquier `save` sin `rationale`/`source` no triviales, antes de tocar disco o de invocar engram por MCP, y desde los tres puntos de entrada del backend para que no sea evitable. Un agente no puede persistir contexto inventado sin declarar su origen.
- **Gate de freeze** — el orquestador congela el candidate y verifica el hash antes de delegar un `apply`, de modo que el subagente trabaja sobre exactamente los inputs revisados.
- **Gate de recibos** — cada transición deja un receipt JSON con hash, en éxito y en fallo, antes de dar un incremento por verificado o de integrar su conocimiento.
- **Errores tipados** — la recuperación automática se decide sobre `error.code` de un catálogo cerrado, nunca sobre el texto del mensaje.

### Propagación normativa a las 7 superficies de `axiom-autopilot` (`INC-20260730-autopilot-integration`)

Las tres directivas (scope tipado y determinista al delegar, verificación de freeze antes de un apply delegado, captura y verificación de recibos antes de integrar conocimiento) son **requisito formal, no sugerencia**, y están presentes en las **siete** superficies del orquestador: `.agents/skills/axiom-autopilot/SKILL.md` (raíz del workspace), `Axiom.SDD/.agents/skills/axiom-autopilot/SKILL.md`, `Axiom.SDD/.github/skills/axiom-autopilot/SKILL.md`, `Axiom.SDD/.github/agents/axiom-autopilot.agent.md`, `Axiom.SDD/.github/prompts/axiom-autopilot.prompt.md`, `.claude/skills/axiom-autopilot.md` y `.claude/commands/axiom-autopilot.md`.

El defecto que esto corrige es de **propagación, no de redacción**: las directivas existían únicamente en la copia de la raíz del workspace, mientras que las fuentes distribuibles bajo `Axiom.SDD/` — que son las que se instalan en un proyecto adoptante — no las tenían. Un proyecto que adoptara Axiom recibía por tanto un orquestador sin ninguno de los tres gates. Regla derivada: **una directiva normativa que solo vive en la copia local del workspace no está adoptada**; la fuente distribuible es la que define lo que reciben los adoptantes.
