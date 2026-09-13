# 07 Gobierno y Seguridad

## Gobierno

1. una única fuente de verdad por capa;
2. no mezclar runtime, spec y tooling en la misma responsabilidad (regido por `Axiom.SDD/AGENTS.md` para este workspace);
3. no promover estructuras legacy como canónicas por comodidad;
4. límites explícitos de bootstrap (`Axiom.SDD/AGENTS.md`): no introducir sin pedido explícito índices obligatorios, sistemas de metadata complejos, carpetas de lifecycle enterprise, integraciones de work items (Azure DevOps/Jira) como dependencia obligatoria, abstracciones MCP obligatorias, lógica de Workbench, frameworks pesados de multi-agente, jerarquías autogeneradas profundas, ni scripts que no existan aún.

### Ownership documental de decisiones

`Axiom.Spec/specs/decisions/` es la raíz canónica de las decisiones estructurales gestionadas y `Axiom.Spec/specs/adr/` la de los ADR gestionados del workspace. Las referencias activas deben apuntar a esas rutas; `Axiom/docs/` puede conservar documentación operativa o histórica, pero no es el hogar actual de esos artefactos. Esta regla es independiente de `Axiom/axiom.spec/`, que sigue siendo baseline product-owned del runtime y no una segunda fuente canónica de la spec (ADR-0032).

### Fuente y distribución del manual runtime

`Axiom/docs/**` es la única fuente del manual operativo del producto runtime.
`Axiom.Spec/specs/manuales/**`, cuando existe, pertenece a la instalación
canónica del workspace y no se fusiona ni se distribuye. Los proyectos que
instalan, adoptan o actualizan Axiom reciben una copia bajo `docs/axiom/` en el
repositorio autoral; el materializador común conserva el alcance bajo el root,
usa escritura atómica y mantiene `manifest.json` con `sourceHash` y hashes por
archivo. Las ediciones locales no se pisan: quedan `stale` y la nueva versión
se separa bajo `.stale/`.

### Gate documental de cierre

Las transiciones gobernadas de archive consumen una declaración estructurada de
revisión documental. El alcance puede enumerar documentos explícitos o paths
afectados; cada entrada declara `updated`, `unchanged` o `unreviewed` con motivo.
En modo `warning`, un documento no revisado produce aviso; en modo `block`,
impide la transición. Preview no escribe. El gate comparte el runner común de
CLI, launcher y MCP y no crea una segunda vía de archivado.

## Seguridad operativa y compliance (verificado en runtime)

1. **GATEs verificables por doctor**: GATE 0031 (8 adapter packages activos deben tener `src/generator.ts` + `dist/index.js`), GATE 0024 (memoria no es fuente de verdad; spec prevalece en conflicto), GATE 0033 (agents como contratos materializables, sin ejecución). El audit trail local es transversal y no depende de un gate enterprise.
2. **Aislamiento project-scoped**: `@axiom/isolation` aplica path-guard y una lista de MCP servers permitidos por defecto; ninguna cache, binding o entrada de memoria debe cruzar `projectKey` (`projectId` v2 o slug estable v1). El server MCP ejecutable **hace cumplir** este aislamiento a nivel de handler, no solo por convención — ver "Aislamiento MCP por proyecto (enforced)" más abajo.
	El mismo principio se aplica a checkpoints, markers de toolchain y
	selección de providers: buscar un alias legacy no autoriza restaurar o leer
	literalmente en ese alias. Los destinos se remapean al namespace canónico y
	las operaciones de worktree reciben `Execution.projectId` explícito.
3. **Audit trail**: `axiom audit` verifica SHA-256, conteo de líneas, retención y detección de reescritura externa, devolviendo `compliant` | `absent` | `violation` (exit 1 en violación).
4. **Telemetría local**: `telemetry-sinks.yaml` define la política `local-only` y una lista de sinks (`local-log`, `local-audit-trail`). El `AuditTrailSink` es append-only, mantiene sidecar SHA-256 y aplica `P365D` por defecto; `axiom audit` lee de forma read-only. No existe gate de señales ligado a un overlay enterprise retirado.
5. **Doctor como gate de gobierno mínimo**: valida presencia y validez de `axiom.yaml`, `integrations.yaml`, `policy-as-code.yaml`, protección de `.axiom-state/local/` frente a versionado accidental, y aislamiento project-scoped.
6. **Policy-as-code**: `policy-as-code.yaml` concentra `sensitivityTags`, `artifactLifecycle` (transiciones y verificación antes de archive), reglas de `tools`/`compliance` (actitud ante herramientas faltantes o MCP no aprobados), `projectIsolation` y `doctorValidation`.

7. **Compatibilidad controlada de resolver**: `ProjectResolution.mode` solo
	expone `local-only`. `gateway` y `hybrid` se aceptan como input raw para
	migración v1/v2, pero no son permisos, providers, rutas ni estados actuales.

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
	`externalRefs` se redactan antes de volver a la UI. En lifecycle,
	`--force`, `--no-review` y `--no-verify` sólo afectan sus gates locales
	documentados y nunca sustituyen la confirmación explícita exigida por una
	transición `requiresApproval`.
- La antigua TUI pública y su acción implícita fueron retiradas tras una
	matriz de paridad. La CLI headless, el launcher y MCP son las superficies
	vigentes; los runners compartidos no se eliminan por retirar una interfaz.

## Reglas de cierre (incrementos y bugs, vigentes en este workspace)

Un incremento o bug solo puede marcarse `closed` si: el objetivo/comportamiento esperado es claro, existen acceptance criteria, se implementaron cambios (o hay justificación explícita de no-code), se ejecutó la validación disponible, se revisó contra el intent original y los acceptance criteria, y se integró conocimiento estable en `Axiom.Spec`. Si falta cualquiera de estos puntos, el estado debe quedar `pending` con motivo explícito — nunca `closed` por comodidad.

## Límite de dogfooding (check `DF-001`) — roadmap de rediseño, cerrado

Regla: "Axiom se desarrolla con Axiom, pero Axiom no contiene su propia factoría interna como parte del producto instalable."

- `runDogfoodingBoundaryChecks` (`Axiom/packages/doctor/src/checks.ts`, id de check `DF-001`, categoría `dogfooding`) comprueba que ningún repo de `code` (`TopologyManifest.codeRepos`) tenga una dependencia física — una dependencia local de `package.json` (`file:`/`link:`/path relativo) o un literal de path `require`/`import` — que resuelva dentro de la autoridad `axiomRepo` o de los repos fuente `legacyRepos` que el proyecto haya declarado. El escaneo sigue siendo estrictamente unidireccional: solo se marca code → authority/legacy.
- Parametrizado por rol por diseño, no hardcodeado por nombre: se apoya en `axiomRepo`/`codeRepos`/`legacyRepos` de `TopologyManifest`, de modo que funciona sobre los nombres de repo de cualquier proyecto de terceros, no solo sobre este workspace.
- La comprobación exige que `loadTopology` haya resuelto una autoridad schema 2 válida; una autoridad ausente, pointer inválido, YAML malformado, schema no soportado o semántica inválida no se convierte en una topología vacía ni en un fallback local. Fuera de una topología multi-repo válida, el check puede reportar `skip` conforme a su contrato; un match real de path sigue siendo `fail`.
- Escaneo mínimo suficiente (sin parseo AST, sin resolución de alias de bundler, sin recorrido transitivo de `node_modules`): un escaneo de dependencies/devDependencies/optionalDependencies de `package.json` acotado a los globs `workspaces` declarados propios del repo, más un grep acotado de literales de path `require(...)`/`from '...'`, excluyendo paths generados conocidos vía `aggregateKnownGeneratedPathGlobs`.
- En este workspace la autoridad schema 2 está en `Axiom.Spec/axiom.config/topology.yaml`; `Axiom/axiom.yaml` apunta a ella como repo `code` y `Axiom.SDD` figura como `legacy` read-only. El check ya no presupone ni necesita un `topology.yaml` local dentro de `Axiom/`.

## Checks de doctor — categorías establecidas por el roadmap de rediseño

`@axiom/doctor`'s forma `DoctorCheck { id, category, description, status, evidence }` (`pass | fail | warn | skip`) se usa de forma consistente en toda categoría. Doctor es **puramente diagnóstico** — detecta y reporta, nunca muta el filesystem (confirmado por lectura directa: cero rutas de código `fix`/`repair`/`autofix` en `checks.ts`). Categorías añadidas por este roadmap, sumadas al conjunto preexistente (boundaries, policies, manifests, isolation, capability-model, install-profiles, tool-routing, topology, coherencia de QA-lane):

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

La validez de `mcp.yml` se comprueba en línea durante setup/proyección mediante
`validateMcpProjectConfig` y el filtro project-bound; no existe un check de
doctor dedicado para el registro de capability/provider de
`@axiom/mcp-tools`. El filtro bloquea la proyección nativa si el proyecto, el
manifest, `enabled` o `targetRepo` no pueden confirmarse. No existe todavía un
instalador/scaffold que escriba `capabilities.yaml`/`providers.yaml` a un
proyecto real, así que no se inventa un check de doctor para esos artefactos.

## Trazabilidad de write-scope y ownership (ángulo de gobierno)

`validateWriteScope` (ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md)) es el mecanismo de gobierno que impide que un plan mute paths fuera de su `allowedWriteScope` declarado, o que repos con rol `sdd` reciban cambios no declarados (`unexpected-sdd-change`), o que se manipulen paths generados/cache conocidos (`generated-cache-tampering`). Es el mecanismo concreto que hace cumplir en runtime la separación de ownership entre repos que exige esta sección de gobierno — dos superficies (`axiom validate changes` y el check `WS-001`) comparten una única primitiva sin lógica duplicada.

Hasta `INC-20260710-plan-role-split` (P1-5), esta primitiva era genérica pero no tenía nada real que hacer cumplir en la práctica: `axiom-plan create` siempre dejaba el `allowedWriteScope` del plan vacío. Cerrado: `axiom-plan create` ahora deriva el role-split del plan de `topology.yaml#roles`/`#assignments` (o de un `--roles` explícito) y puebla `targetRepos`/`allowedWriteScope` con los repos que esos roles poseen — ver "`PlanMetadata.roles`" en [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md). `validateWriteScope` en sí no cambió; el gap era exclusivamente que el producer nunca poblaba lo que el consumer ya sabía leer.

La guarda de ownership de workspace es ahora una frontera común, no un warning
local de scaffolding. `workspace setup`, `repo add` y `role add` rechazan antes
de toda escritura un `axiom.yaml` foráneo, inválido o ambiguo; `workspace adopt`
solo puede preservar como `skipped` una identidad válida de otro proyecto. El
preflight enumera tanto recursos estructurales como destinos derivados, valida
create flags, IDs reservados, ownership y solapamientos bidireccionales, y trata
solo `ENOENT` como ausencia. `ENOTDIR`, `EACCES`, `EIO` o una observación
indeterminada son fallos tipados, no permiso para inventar un target.

Apply mantiene locks de proyecto, registry solicitado y recursos en orden
canónico durante recovery, replan y comprobación de precondiciones. El journal
persistido bajo `.axiom-state/<projectKey>/structural-transactions/<operationId>/`
permite rollback/roll-forward por hashes; contenido desconocido deja
`recovery-required` y bloquea la siguiente mutación. No existe éxito parcial
estructural. `axiom.yaml` solo reemplaza `AXIOM:MANAGED`, preservando byte a byte
las regiones humanas; una conversión markerless solo se acepta con identidad
inequívoca y sin conflicto.

Adapters, skills, reglas, MCP, catálogos y base de spec son outputs derivados:
se ejecutan únicamente después de `committed`, dentro de límites de ownership ya
prevalidados. Sus averías se aíslan como warnings tipados y no revierten ni
falsean la unidad estructural. Los comandos granulares de repair reutilizan el
catálogo único de steps y no habilitan capacidades ni mutan registry o
`workspace.json`.

Robustez del catálogo user-level (`INC-20260829-r13-user-project-registry`): `~/.axiom/projects.yml` es el único formato activo y se valida completo antes de exponer o mutar datos. IDs ambiguos, identidades divergentes y paths canónicamente equivalentes ya poseídos por otro proyecto fallan sin escribir; una reejecución solo es idempotente cuando conserva identidad y ownership. Las mutaciones están serializadas por un lock local acotado y el reemplazo usa un temporal propietario, flush/fsync, validación y rename atómico. El reclaim comprueba generación y fencing: un reclamador retrasado no puede borrar la generación sucesora y la ventana entre crear el lock y publicar su owner queda protegida por un lease de inicialización. La recuperación elimina únicamente claims, leases o temporales propios y huérfanos; nunca borra contenido ajeno. Ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md).

## Aislamiento MCP por proyecto (enforced) — ACC-030 sobre los guards previos

El server MCP ejecutable (`@axiom/mcp-server`, lanzado por proyecto vía `axiom mcp serve --kind axiom --project-root <path>`, ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md)) **hace cumplir** el aislamiento project-scoped a nivel de código, no solo por convención. Esto es una garantía de ownership/seguridad de aislamiento equivalente a la que `@axiom/memory` ya aplica sobre `projectId` (GATE 0024), llevada a la capa de input-builders del MCP.

- **Vulnerabilidad cerrada (fuga cross-project)**: los input-builders por capability resolvían la identidad de proyecto con el patrón `str(args,'projectId') ?? context.projectId` (y equivalentes para `projectRoot`/`rootPath`/`specScopeAbsolutePath`), de modo que un llamador (el agente conectado al server del proyecto A) podía pasar un `projectId`/`projectRoot` ajeno en los argumentos de `tools/call` y leer los datos del proyecto B a través del mismo proceso servidor. Era una fuga real cross-project (datos, existencia de proyectos, nombres, paths de repos).
- **Pinning al `context` propio**: cada campo que identifica proyecto (`projectId`, `homeDir`, `projectRoot`, `specScopeAbsolutePath`, `sddScopeAbsolutePath`) se **fija (pin)** al `context` que el server resolvió al arrancar (`resolveMcpServerContext({ projectRoot })`). El valor del llamador ya no se usa nunca verbatim: si lo omite, se usa el del `context`; si lo pasa igual, la llamada procede; si lo pasa **distinto**, se rechaza.
- **Rechazo con `isError`**: un valor cross-project que difiera del `context` devuelve un `tools/call` con `isError: true` y el mensaje `"Cross-project access blocked: this MCP server is scoped to project '<id>'. Launch the MCP server for the target project instead."`. El rechazo solo dispara ante un **mismatch real** contra un `context` resuelto — si `context.projectId` no resuelve (proyecto no registrado), no hay opinión de contexto con la que entrar en conflicto y se conserva el contrato preexistente de "campo requerido faltante".
- **`sdd.projectRegistryRead` acotado al proyecto propio**: esta capability ya no enumera el registro machine-wide (`listProjectsV2`); su salida se filtra a la entrada cuyo `id === context.projectId` (o `[]` si no resuelve). Un server project-scoped enumerando proyectos ajenos era en sí mismo una fuga.
- **Selectores dentro-de-proyecto validados**: los argumentos que nombran algo INTERNO al proyecto ya fijado (`id`, `planId`, `skillId`, `specRelPath`, `role`/`roleOrKind`, `taskTags`, `includeStale`, `contextBudget`, `targetRepoId`) siguen siendo caller-supplied, pero los IDs deben ser segmentos seguros y `specRelPath`/paths de skills no pueden escapar de las raíces registradas del proyecto.
- **Alcance y límites**: los artefactos de config en disco ya eran project-scoped; el registro es metadata-only. La corrección es completa a nivel de handler (no queda ninguna ruta donde un identificador de proyecto del llamador se use verbatim para leer datos). El guard de aislamiento en el arranque vía `@axiom/isolation` (`checkMcpAllowed`/`assertProjectIsolation`) se **difirió** deliberadamente: requeriría sintetizar un `ProjectResolution` a partir de un `McpServerContext` (adapter especulativo) y no es necesario para cerrar la vulnerabilidad confirmada — el pinning a nivel de handler es el fix no negociable y suficiente.

## Filtro MCP por proyecto y target (ACC-029)

La proyección a configuraciones nativas tiene una barrera anterior al writer:
`filterProjectBoundMcpServers` confirma el `projectId` en el registry v2,
valida `mcp.yml` y `mcp-manifest.yaml`, reconcilia el id lógico `axiom` con
`axiom-mcp-broker`, y compara `enabled`, tipo, scope y `targetRepo`. Ante un
error devuelve cero servidores y un warning accionable. El provisioning de
worktrees usa la misma barrera y deriva el `targetRepo` real del binding, por
lo que no existe un camino paralelo que pueda escribir un broker sin binding.

Los writers nativos mantienen un allowlist de IDs gestionados por Axiom y
eliminan solo esos IDs cuando dejan de estar permitidos; los servidores y
claves ajenos del usuario se conservan. Un JSON ilegible queda intacto con
warning. Codex y Antigravity, cuyos ficheros son user-globales, no reciben
escritura ni recomendación automática sin binding seguro. Esta combinación
evita que un MCP de KVP25 aparezca como disponible para EMT por compartir la
máquina o una configuración global.

## Postura de gobierno de la tanda INC-20260708-* (auto-validación, LOCAL-only, aislamiento preservado)

- **Auto-validación del producto: estado histórico y revalidación R-04** (`INC-20260708-product-repo-self-bootstrap` + `INC-20260708-fix-longstanding-test-failures`): aquella tanda dejó la baseline canónica de `Axiom/` materializada y registró doctor/readiness y suite verde. El registro del 2026-07-30 quedó superado: entonces `npm run doctor` y `npm run readiness:first-project` fallaban en `TC-011` por un `bundleHash` stale de `axiom-reviewer`, y la ejecución global independiente del review reportó 3425/3427 tests; ambos hechos pertenecen a fotografías históricas. La revalidación R-04 muestra 13/16 capabilities provider-routed servidas; las tres opcionales restantes son `warn` no bloqueante y las capabilities MCP-only se validan aparte. Ver [00_Resumen_Ejecutivo.md](00_Resumen_Ejecutivo.md), [02_Requisitos_No_Funcionales.md](02_Requisitos_No_Funcionales.md) y [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).
- **Postura LOCAL-only / bbdd-local en toda la capa de providers**: la restricción no-negociable del operador («todas las herramientas siempre en local con bbdd en local») se hace cumplir por construcción en `@axiom/providers`: `LOCAL_ONLY`/`isLocalTarget` rechazan o degradan config de provider no-local; `cmm`/`serena` se lanzan siempre como procesos locales y su ausencia sigue siendo un `warn` de `PS-001`, no un fallo de gobierno. La memoria es una excepción deliberada a esa degradación: Engram local es obligatorio para `@axiom/memory`; TC-024 falla si `engram --version` no responde y toda operación de memoria devuelve un error tipado sin persistencia alternativa.
- **Aislamiento project-scoped preservado en todas las superficies nuevas**: GATE 0024 (memoria) usa Engram con un pin a nivel de proceso (`engram mcp --project=<projectId> --tools=agent`) y guards que rechazan IDs cruzados; el input-builder `memory.*` fija siempre `projectRoot` al `context` del server. No hay backend JSON ni captura automática de decisiones, bugs o lecciones. Las decisiones y bugs confirmados se guardan solo mediante la acción explícita de skill `axiom memory add`; las selecciones de providers, reglas, autoskills y operaciones incrementales permanecen project-scoped y best-effort/no-clobber.
- **Sin arquitectura especulativa** (límites de bootstrap): esta tanda rechazó explícitamente el motor de instintos de ECC (scoring/herencia/promoción), un motor de hooks de sesión propio y la fabricación de instrumentación de tool-calls en vivo. La retirada de `axiom learn` conserva el audit trail como trazabilidad, no como fuente automática de memoria; los delegation triggers siguen siendo deterministas y puros sobre inputs explícitos.

## Regla conocida de build (no duplicada aquí)

`Axiom/packages/cli-commands/tsconfig.json` tuvo un defecto histórico de tooling de build que rompía `--help` para varios comandos CLI transitivamente dependientes de ese paquete. El bug legacy `BUG-20260702-cli-commands-tsconfig-missing-emit` fue cerrado el 2026-08-06 y su corrección está documentada en el incremento archivado `INC-20260804-cli-commands-package-output`; la antigua carpeta top-level se retiró al limpiar raíces legacy. No se duplica el detalle aquí.

## Onboarding/ADO desde el launcher: mutación segura y secreto fuera del repo (2026-07-15) — tanda INC-20260715-*

- **Mutación confirm-gated en todas las superficies nuevas del launcher**: los endpoints de onboarding (`install`/`join`/`roles register`/`roles assign`) y el puente ADO en creación NO mutan sin `confirmed:true` (preview primero), heredando el contrato de `/launcher/execute`. El explorador de carpetas y la detección de tracker son read-only, best-effort y no-crash (nunca tumban el server ni el front). `INC-20260715-launcher-onboarding`, `INC-20260715-launcher-ado-bridge`.
- **PAT de Azure DevOps nunca en el repo**: el token de acceso personal se resuelve en runtime (variable de entorno declarada en `tracker.json#auth.patEnvVar`, o env de usuario Windows, o `SecretStore` bajo `axiom.ado.<org>.<project>.pat`, o prompt interactivo persistido en el store); nunca se escribe en `tracker.json` ni en ningún fichero versionado. El plugin sólo se considera configurado con `kind:'ado'` + `enabled` + org + project (`isRealTrackerRequested`); cualquier otra combinación degrada a local-only sin red. Detalle operable en el manual runtime `Axiom/docs/cli/external-sync.md`; capacidad en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).
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
- **Aislamiento MCP preservado en el broker unificado** (`ACC-030`): el broker `axiom` mantiene los pins project-scoped por-campo y expone la unión completa del registry, incluidas `sdd.transitionApply`, `sdd.gitRoleBranch` y `sdd.gitCommitSync`; todas las mutaciones conservan preview/confirmación y sus guards existentes.
- **Genericidad sin fuga de gobierno**: todo el contenido nuevo es adapter/stack-agnóstico; la profundidad específica de cada proyecto se inyecta como DATO por proyecto (`skills-index/<role>.yaml` + contexto técnico), no en el producto — sin hardcodear reglas de un stack ni credenciales. Ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md) y el manual runtime `Axiom/docs/usage/README.md`.
## Gobierno verificable de flujos desatendidos (2026-08-02) — tanda `INC-20260730-*`

La tanda cierra el bloque de gobierno de la ejecución desatendida sobre tres gates y un catálogo de errores (detalle funcional en RF-AXM-057..061, propiedades en NFR-AXM-023):

- **Gate de evidencia** — `@axiom/memory` rechaza fail-closed cualquier `save` sin `rationale`/`source` no triviales, antes de tocar disco o de invocar engram por MCP, y desde los tres puntos de entrada del backend para que no sea evitable. Un agente no puede persistir contexto inventado sin declarar su origen.
- **Gate de freeze** — el orquestador congela el candidate y verifica el hash antes de delegar un `apply`, de modo que el subagente trabaja sobre exactamente los inputs revisados.
- **Gate de recibos** — cuando el caller habilita `receipt`, `runGovernedTransition` deja el receipt JSON desde la ruta común, en éxito o rechazo aplicable; previews no lo emiten y un fallo de escritura es best-effort/no bloqueante. El orquestador verifica los recibos requeridos antes de dar un incremento por verificado o de integrar su conocimiento.
- **Errores tipados** — la recuperación automática se decide sobre `error.code` de un catálogo cerrado, nunca sobre el texto del mensaje.

### Propagación normativa a las 7 superficies de `axiom-autopilot` (`INC-20260730-autopilot-integration`)

Las tres directivas (scope tipado y determinista al delegar, verificación de freeze antes de un apply delegado, captura y verificación de recibos antes de integrar conocimiento) son **requisito formal, no sugerencia**, y están presentes en las **siete** superficies del orquestador: `.agents/skills/axiom-autopilot/SKILL.md` (raíz del workspace), `Axiom.SDD/.agents/skills/axiom-autopilot/SKILL.md`, `Axiom.SDD/.github/skills/axiom-autopilot/SKILL.md`, `Axiom.SDD/.github/agents/axiom-autopilot.agent.md`, `Axiom.SDD/.github/prompts/axiom-autopilot.prompt.md`, `.claude/skills/axiom-autopilot.md` y `.claude/commands/axiom-autopilot.md`.

El defecto que esto corrige es de **propagación, no de redacción**: las directivas existían únicamente en la copia de la raíz del workspace, mientras que las fuentes distribuibles bajo `Axiom.SDD/` — que son las que se instalan en un proyecto adoptante — no las tenían. Un proyecto que adoptara Axiom recibía por tanto un orquestador sin ninguno de los tres gates. Regla derivada: **una directiva normativa que solo vive en la copia local del workspace no está adoptada**; la fuente distribuible es la que define lo que reciben los adoptantes.

## Gobierno R-10 de transiciones y evidencia

El grafo canónico de `workflows.yaml` se resuelve fail-closed: sólo su ausencia permite usar el asset distribuido; una configuración presente ilegible o con schema no soportado no puede degradarse silenciosamente a otro workflow. Las superficies CLI, launcher, MCP e integrate no mantienen grafos o gates alternos.

`runGovernedTransition` exige confirmación explícita para toda transición `requiresApproval`; bypasses locales (`--force`, `--no-review`, `--no-verify`) no la sustituyen. La única aprobación de plan es `draft → plan-approved`, y el inicio de roles comprueba state y metadata del plan. Para archive de increment/bug, la decisión QA común es fail-closed cuando inline o requerida: sólo evidencia `passed` permite persistir; parallel puede continuar con aviso. Preview no escribe, y una operación fallida recupera o declara inconsistencia en vez de ocultar éxito parcial.

Los receipts son evidencia de observabilidad co-localizada con el artefacto y se verifican explícitamente en cierres: su escritura best-effort no convierte un fallo durable en éxito. Los cambios de estado, integración y movimiento a `_archive` se ejecutan únicamente por Axiom/Core; las referencias o decisiones históricas se preservan, y una superación nueva se registra por Core sin reescribir el antecedente.

Las Decisions tienen un límite deliberado del modelo Core: `axiom-decision` permite crear y enlazar con planes o incrementos, pero no aceptar, cerrar ni superseder una Decision. Por ello `DEC-20260818-134600-3jfjak` permanece `proposed` y solo está enlazada por Core al correctivo R-10; ese vínculo aporta trazabilidad, no una supersesión formal de la decisión histórica 0015.


## Control plane del launcher: autorización y límites (R-13, ACC-070..ACC-072)

La seguridad del launcher se implementa en el runtime mediante un estado server-side por proceso (`AsyncLocalStorage` para el contexto y un marcador no falsificable para autorización). Los grants conservan hashes de token, digest de payload, sesión, proyecto, acción, diagnóstico y expiración; los consumidos permanecen solo dentro de una ventana acotada para distinguir replay. Grants y deliveries tienen límites deterministas de memoria y purga/evicción por TTL/retención.

Los endpoints de plugins, onboarding, paneles ADO/Git y lifecycle convergen en preview→confirmación y handlers allowlisted; no aceptan una autorización HTTP inventada. La autorización se marca únicamente después del consumo atómico del grant y antes del bridge/runner. El alias plugin-scoped de `/launcher/execute` es compatibilidad explícita; los targets ambiguos se rechazan antes de consumir tokens. El transporte HTTP permanece configurado/allowlisted y local-only según la sección de integraciones; Doctor sigue siendo diagnóstico, no permiso. Los endpoints de onboarding (`workspace/setup`, `workspace/adopt`) aceptan en su schema cerrado de body el campo opcional `confirmed`, que es ignorado autorizativamente sobre HTTP (solo aplica a callers in-process sin contexto de request); la ejecución real exige preview grant de un solo uso (INC-20260913-baseline-group-a-onboarding-allowlist).

## Gobierno y seguridad del launcher R-13 (2026-09-08)

El control plane exige loopback literal, sesión por proceso, same-origin, headers seguros, body/schema acotados y errores sanitizados. Los tokens de preview se ligan a sesión, proyecto, acción y payload normalizado, tienen TTL y consumo atómico single-use; replay, carrera, edición, expiry y mismatch fallan cerrado sin mutar artefactos. Required-fields se comprueba antes de alcanzar filesystem, tracker, bridge o red.

El routing declarativo no permite fallback para una acción lifecycle conocida; el fallback queda reservado a acciones sintéticas explícitamente fuera del catálogo. ADO no es una autorización ni una dependencia: es un bridge opcional posterior al resultado local. La matriz ACC-076 demuestra negativos y ausencia de mutación con 35 PASS, 0 FAIL y 0 TIMEOUT; el timeout no se considera éxito.