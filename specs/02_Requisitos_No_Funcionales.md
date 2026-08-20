# 02 Requisitos No Funcionales

## NFR-AXM-001 Trazabilidad

Cada cambio del producto debe ser trazable entre spec, workflow SDD y runtime.

## NFR-AXM-002 Baja ambigüedad

La estructura documental y operativa debe minimizar dobles interpretaciones sobre qué repo es fuente de verdad de cada cosa.

## NFR-AXM-003 Portabilidad

La configuración operativa de repos y adapters debe poder funcionar en distintos entornos y surfaces.

## NFR-AXM-004 Auditabilidad

Los YAML y manifests operativos deben ser legibles, versionables y verificables.

## NFR-AXM-005 Migración controlada

La transición desde la estructura actual debe poder hacerse por fases sin romper el runtime existente en el primer corte. Verificado: `@axiom/versioning` implementa checkpoints (últimos 5 preservados) y rollback automático ante fallo de migración; `axiom upgrade --dry-run` permite previsualizar sin mutar.

## NFR-AXM-006 Sin excepciones para control de flujo

Todo el dominio y las capas de I/O deben modelar fallos con `Result<T, E>` (`@axiom/core`) en vez de excepciones, para que los llamadores compongan sin `try/catch` disperso. Verificado en `@axiom/persistence`, `@axiom/installer`, `@axiom/adapters-*`.

## NFR-AXM-024 Diagnóstico de cobertura sin PASS falso

Los checks de cobertura de capabilities deben usar un catálogo de referencia
independiente de las declaraciones observadas en los providers. Las
capabilities requeridas activas sin provider deben producir un fallo visible;
las opcionales o post-MVP deben producir una advertencia no bloqueante; las
capabilities MCP-only no deben convertirse en falsos huérfanos del registry
tradicional. La evidencia debe distinguir una capability no declarada de una
capability declarada pero no servida.

## NFR-AXM-007 Escritura atómica e idempotencia

Toda escritura de superficies generadas (skills, agents, components, `install-profile.json`, adapters) debe hacerse vía patrón tmp+rename, y debe ser idempotente byte a byte cuando el input no cambia. Verificado en `@axiom/skills`, `@axiom/agents`, `@axiom/document-bootstrap`, `@axiom/persistence`.

Para las instrucciones generadas de Copilot, la idempotencia debe reemplazar
solo el bloque `AXIOM:GENERATED`; el preámbulo, la cola humana y
`TEAM:CUSTOM` se conservan byte a byte. La migración de la ruta legacy
`.vscode/copilot-instructions.md` escribe primero `.github/copilot-instructions.md`,
elimina la fuente solo después de confirmar el destino y conserva ambas copias
con advertencia cuando el contenido humano no se puede reconciliar.

## NFR-AXM-008 Aislamiento project-scoped

Ninguna ruta, cache, MCP binding o entrada de memoria debe cruzar el `projectKey` de origen. `projectKey` es `projectId` en schema v2 y el slug estable de `project.name` en v1. Verificado en `@axiom/isolation` (path-guard, `DEFAULT_ALLOWED_MCP_SERVERS`) y en las reglas GATE 0024 (`@axiom/memory`: spec prevalece sobre memoria en conflicto).

## NFR-AXM-025 Namespace único de estado runtime

Todo estado ligado a un proyecto debe escribir bajo `.axiom-state/<projectKey>/`.
El nombre API `config` no crea un segmento físico adicional. `.axiom-state/local/`
queda reservado para datos repo/operador-locales; `executions/<executionId>/`,
`axiom.config/` y `~/.axiom/` son fronteras separadas. Lecturas legacy deben
tener precedencia determinista, migración atómica e idempotente, advertencia
ante copias conflictivas y ningún writer activo duplicado.

La garantía incluye la restauración semántica: un checkpoint legacy no solo se
localiza por alias, sino que remapea sus destinos al namespace canónico antes
de completar el restore. Detección y repair de toolchain, selección de
providers de worktree y checks de doctor reciben identidad de proyecto
explícita para no caer en scans globales.

## NFR-AXM-026 Compatibilidad de resolución sin reintroducción de modos

`ProjectResolution.mode` solo puede contener `local-only`. `gateway` y
`hybrid` se toleran como valores raw de entrada para migración, pero nunca
como estados efectivos, rutas de provider, permisos o superficies operativas.

## NFR-AXM-009 Auditabilidad de telemetría

La política local-only debe mantener el audit trail habilitado por defecto: log append-only, sidecar SHA-256 y retención `P365D`. `axiom audit` es read-only y `axiom sync` no aplica gates de señales pertenecientes a overlays retirados; un fallo real de configuración o generación sí debe abortar antes de mutar.

## NFR-AXM-010 Cobertura runtime verificable por doctor

Cada capa nueva (adapters, agents, toolchain) debe tener un GATE verificable por `@axiom/doctor` (p. ej. GATE 0031 / check `TC-009`: los **8** adapter packages activos deben tener `src/generator.ts` + `dist/index.js` materializados, o falla — eran 6 hasta `INC-20260726-adapter-generators`, que añadió `codex`/`antigravity`/`visual-studio-2026`; `litellm` fue retirado). No basta con documentación: el doctor debe poder comprobarlo en runtime.

Complementariamente, la suite moderna de `@axiom/mcp-server` da una garantía end-to-end del broker unificado: construye un proyecto adoptado, crea `createMcpServer({ serverKind: 'axiom' })` con binding válido y prueba sobre el dispatcher JSON-RPC real el descubrimiento `server/discover`, el catálogo `tools/list` y llamadas `tools/call` de dominios `sdd.*`, `spec.*` y `axiom.*`. La config generada por workspace queda cubierta por `workspace-mcp.test.ts`: una única entrada `axiom-mcp-broker` con `axiom mcp serve --kind axiom --project-root <control>`. Las antiguas suites de procesos separados `sdd`/`spec` permanecen comentadas como historia y no constituyen contrato vigente. `INC-20260708-adapters-depth` amplió la cobertura con un e2e de adapters (`adapters.e2e.test.ts`) que materializa in-process los `ADAPTER_TARGETS` en un workspace y verifica el set completo de ficheros declarado en `GENERATED_FILES_BY_TARGET` por target (ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md)); desde `ACC-025..ACC-029` son **8** targets activos (antes 10, con LiteLLM retirado y sin aceptación pública de `copilot-vscode`).

**Registro histórico de validación (2026-07-30; superado el 2026-08-02):** el requisito de cobertura runtime sigue vigente, pero aquella ejecución global no debe confundirse con el estado actual. El cierre histórico de `INC-20260708-product-repo-self-bootstrap` e `INC-20260708-fix-longstanding-test-failures` registró una suite sin fallos en ese momento; la ejecución independiente del review reportó 328/330 archivos y 3425/3427 tests, con fallos en `packages/doctor/tests/agents.test.ts` y `packages/mcp-tools/tests/capability-routing-roundtrip.test.ts`, fuera del alcance de ese incremento. Después se añadió una prueba focalizada de `extractVersion()` y ambos fallos se reprodujeron aisladamente. En esa fecha, `npm run doctor` y `npm run readiness:first-project` también fallaban en `TC-011` por un `bundleHash` stale de `axiom-reviewer`.

**Registro histórico verificado el 2026-08-02:** `npm run doctor` devolvía `PASS` (46/61 OK, 0 fallos, 3 advertencias, 12 omitidos). La medición global de aquella fecha registraba 3489 tests, 3483 verdes y 6 fallos preexistentes caracterizados. La revalidación R-04 del 2026-08-06 devuelve `doctor PASS` (45/60, 0 fallos) y readiness PASS; `CC-004` sirve 13/16 capabilities y deja warning por `code.symbolSearch`, `code.referenceSearch` y `code.impactAnalysis`. Nota de taxonomía reconciliada: el model-routing tiene **7 slots** canónicos (`increment, bug, plan, implementation, qa-e2e, review, archive`), no 10.

## NFR-AXM-011 Evolución de schema solo aditiva

Todo cambio de schema persistido (registro de usuario, manifiesto `axiom.yaml`, `metadata.yml` de artefactos, `mcp.yml`) debe convivir de forma aditiva junto al formato anterior, seleccionado por presencia de fichero o por campo `schemaVersion`, en vez de reemplazar/renombrar destructivamente. Verificado en la migración de registro (`~/.axiom/projects.yml` `schemaVersion: 2` añadido junto a `~/.axiom/registry.json` `schemaVersion: 1`, ambos leídos/escritos por el mismo paquete) y en el cutover de `axiom.yaml` (`schemaVersion: 1` y `2` ambos soportados por `resolveProject` y los checks `MC-001`/`BC-001`/`BC-002` de `@axiom/doctor`, nunca resolviendo mal uno como el otro).

## NFR-AXM-012 Sin caché persistente hasta que el volumen real lo justifique

Ningún subsistema debe introducir una caché persistente en disco (p. ej. un índice de artefactos) sin una necesidad de rendimiento medida o un consumidor concreto que la requiera. Verificado: `axiom index rebuild`/`validate` son wrappers de escaneo directo sobre `listArtifacts`, sin fichero `.axiom/cache/*.index.json`; ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md).

## NFR-AXM-013 Disciplina de alcance mínimo primero

Toda extensión del producto debe auditar primero el paquete existente que ya cubre parcialmente la necesidad, y extenderlo de forma aditiva, en vez de construir una infraestructura paralela. Principio aplicado de forma sistemática por el roadmap de rediseño de 23 incrementos (cada incremento llevó un migration-engineer como primer subagente, con la tarea explícita de auditar antes de escribir contrato nuevo) — ver `specs/increments/_archive/INC-20260702-axiom-redesign-roadmap/README.md`.

## NFR-AXM-014 Aislamiento por construcción

Los mecanismos de aislamiento (project-scoping, MCP allowlist, límite de dogfooding) deben derivarse de la resolución de topología/proyecto en vez de hardcodear nombres de repo o rutas, de forma que funcionen correctamente sobre cualquier proyecto/topología de terceros, no solo sobre este workspace. Verificado en el check `DF-001` (dogfooding, parametrizado por rol vía `TopologyManifest`, ver [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md)) y en el aislamiento de `mcp.yml` por scoping de filesystem dentro del repo de spec de cada proyecto (ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)).

## NFR-AXM-015 Operaciones incrementales sobre instalaciones existentes (hueco de 7 operaciones — parcialmente cerrado)

`axiom configure` es single-shot (re-aplica el perfil persistido completo) y no cubre añadir/quitar incrementalmente los elementos de un workspace ya inicializado. El "hueco de 7 operaciones" (referenciado desde [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md) RF-AXM-022 y [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md)) enumera las mutaciones incrementales ausentes: añadir/quitar repo, rol, adapter y provider/tool-MCP.

**Parcialmente cerrado — `INC-20260708-incremental-operations`**: las 4 operaciones de tipo ADD ya existen como comandos CLI idempotentes y no-clobber que resuelven el proyecto desde cwd y reusan el motor multi-repo existente (`runWorkspaceSetup` + helpers), sin duplicar lógica de scaffolding: `axiom repo add`, `axiom adapter add <target>`, `axiom provider add <id>`, `axiom role add <roleId> --path <path>` (ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md) y [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)). Las 4 operaciones de tipo REMOVE (`repo/adapter/provider/role remove`) quedan explícitamente **diferidas** como follow-up simétrico, no implementadas. Es una capacidad genuinamente nueva del runtime del producto (no tooling de desarrollo).

## NFR-AXM-016 Economía de tokens y foco del agente vía tuning por adapter (`INC-20260715-adapter-agent-tuning`)

Los prompts que el launcher genera deben poder pedir al agente un modo de trabajo terse y económico en tokens sin cambiar la lógica de negocio. Verificado: `agentTuning` por adapter (`verbosity`/`personality`/`model?`) inyecta un preámbulo determinista ("responde solo lo necesario, minimiza tokens") en el prompt pregenerado; de serie `verbosity:'low'`+`personality:'pragmatic'`. Es prompt-shaping puro y opcional (sin `agentTuning` no hay preámbulo → retrocompatible), desacoplado de model-routing y selección de providers. Capacidad en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md), superficie en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md).

## NFR-AXM-017 Legacy intacto y no filtración de secretos (`INC-20260724-*`)

Axiom nunca escribe en los repos legacy de un proyecto adoptado (probado byte-for-byte); la adopción crea un `<project>.axiom` nuevo y los legacy quedan como fuentes read-only. El provisioning de un worktree es portable-only: copia exactamente 3 ficheros no-secretos (`init.json`/`install-profile.json`/`workspace.json`) y **nunca** lee ni copia `.axiom-state/local/**` ni otros proyectos (secretos nunca copiados). El push de artefactos SDD va acotado (`git add -- <paths>`, nunca `-A`). Ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md) y [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md).

## NFR-AXM-018 Sin hooks git obligatorios; operaciones externas best-effort/never-block (`INC-20260724-*`)

Ninguna capacidad de la graduación a full lifecycle instala un hook git. El auto-sync de `cmm`, el uso de RTK, el auto-fetch de freshness de artefactos, el provisioning y el teardown de code-intel por worktree son todos **on-demand y best-effort**: nunca lanzan, degradan a un warning/`unknown` y jamás bloquean la operación del usuario. Complementa NFR-AXM-006 (sin excepciones para control de flujo). Ver [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md) y [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md).

## NFR-AXM-019 Aislamiento y limpieza segura por worktree (`INC-20260724-*`)

Cada worktree ejecuta una `Execution` distinta con estado execution-scoped (`.axiom-state/executions/<id>/…`) y su propio índice/caché de code-intel (nunca un grafo mutable compartido; `teardownWorktreeCodeIntel` nunca toca el índice del repo principal). El cierre sigue el orden estricto kill→harvest→teardown→remove con harvest siempre antes de cualquier borrado; un worktree con trabajo real sin integrar es hard stop (nunca `force` por defecto), y un fallo de cierre no deja estado inconsistente (rollback compensatorio). El schema evoluciona de forma aditiva (topología `schemaVersion: 2`, campos de metadata opcionales), extendiendo NFR-AXM-011. Ver [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md).

## NFR-AXM-020 Probes de runtime de doctor: aditivos, opt-in, never-fail (`INC-20260726-doctor-runtime-probes`)

Los probes de runtime que `axiom doctor --deep` añade (`TC-018-*`, `TC-019-*`, categoría `runtime-probes`, ver RF-AXM-054 en [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md)) son estrictamente ADITIVOS: solo pueden emitir `pass`/`warn`/`skip`, NUNCA `fail` (el helper `fail` no se les expone por diseño). `runDoctorChecksDeep` delega en `runDoctorChecks` (no re-lista los checks, para que las dos listas no diverjan) y su `summary.failed` es siempre idéntico al del árbol síncrono — el gate de doctor del launcher y todos los checks/tests/callers preexistentes quedan byte-idénticos salvo que se opte explícitamente por `--deep`/`?deep=1`. Ambos probes son inyectables (`probeFn`/`McpProbeFn`), así que la suite de tests nunca spawnea un proceso real. La fuente del liveness MCP es **exclusivamente** `.axiom/mcp.yml` (nunca un comando sintetizado): un proyecto sin él (p. ej. layouts self-hosted que no corrieron `axiom workspace setup`) recibe un `warn` honesto, no una adivinanza. Complementa NFR-AXM-018 (best-effort/never-block).

## NFR-AXM-021 Vocabulario de adapter targets de fuente única + honestidad de MCP nativo (`INC-20260726-adapter-*`)

Toda alta de un adapter target DEBE tocar 5 sitios, todos compile- o runtime-enforced, o el target se rompe en silencio: (a) `ADAPTER_TARGETS` de `init.ts` + sus re-declaraciones inline; (b) `adapterTargets` de `default-profiles.ts` + los `allowedTargets` de AMBOS profiles; (c) `SUPPORT_MATRIX`/`MVP_TARGETS` del support-matrix; (d) `ADAPTER_LABELS` de `_adapter-labels.ts`; (e) el switch de despacho de `workspace-adapters.ts` (guarda de exhaustividad tipada `never`). El primero se enforcea por typing estructural de TS, el switch por la guarda `never`, el resto por tests — así una futura alta no puede olvidar un sitio en silencio. Además, un target que deba ser elegible en el launcher debe recibir su entrada en `AXIOM_ADAPTER_ROUTING` (RF-AXM-052). La config MCP nativa nunca se inventa: un target sin schema verificado recibe una nota informativa o fallback, jamás un fichero fabricado.

## NFR-AXM-022 Reproducibilidad y aislamiento del versionado de toolchain (`INC-20260730-toolchain-versioning`)

El versionado de tools debe conservar separación de responsabilidades y fallar de forma acotada:

- El catálogo global (`axiom.config/toolchain-catalog.yaml`) y el manifest habilitado (`axiom.config/toolchain.yaml`) son declarativos; el lockfile (`.axiom-state/<projectKey>/toolchain.lock`) es estado local generado, project-scoped e ignorado junto con `.axiom-state/`.
- La persistencia del lockfile debe ser atómica (`tmp` + `rename`), y un upgrade fallido debe dejar exactamente el lockfile previo o restaurar la ausencia previa. No se permite modificar binarios externos como efecto lateral.
- `plan` debe ser una operación pura. La mera presencia de una tool en el catálogo no debe implicar una alta, descarga o instalación; la CLI planifica el conjunto declarado o lockeado y permite restringirlo con `--id`.
- Los probes de versión deben ser locales, inyectables en tests y best-effort. La ruta síncrona de doctor mantiene su semántica existente; `doctor --deep` puede añadir advertencias o saltos de probe, pero nunca nuevos fallos duros.
## NFR-AXM-023 Gobierno verificable de la ejecución desatendida (tanda `INC-20260730-*`)

Un flujo desatendido (`/axiom-autopilot` y sus subagentes) debe dejar **evidencia comprobable** de lo que ejecutó, y no debe poder avanzar sobre estado no verificado. Tres propiedades, todas fail-closed o best-effort según corresponda:

1. **Evidencia en memoria** — nada se persiste en la memoria del proyecto sin `rationale` y `source` no triviales (RF-AXM-058). Un agente no puede "inventar contexto" y guardarlo sin justificar de dónde salió. El rechazo ocurre antes de cualquier I/O y no es evitable eligiendo otro backend.
2. **Recibos de fase** — toda transición de ciclo de vida emite un receipt JSON inmutable con `hash` sha256 sobre su contenido, en éxito y en fallo (RF-AXM-059). La emisión es **best-effort/never-block**: certifica sin poder romper el ciclo que certifica. Esta es una asimetría deliberada — un receipt ausente degrada la auditabilidad, pero un receipt que hace fallar un `apply` correcto sería peor.
3. **Congelado de candidate** — el estado de entrada de un candidate se congela y se verifica antes de delegar un apply (RF-AXM-060), de modo que el apply sea determinista respecto de los inputs que se revisaron.

Refuerza `NFR-AXM-006` (sin excepciones para control de flujo): con `AXIOM_ERROR_CODES` (RF-AXM-057) la recuperación automática se decide sobre un `code` estable, no sobre el texto de un mensaje.

**Hueco conocido y no cerrado**: el receipt se emite tras retornar el core de la transición, por lo que cubre los fallos con `exitCode === 1` pero **no** una excepción que escape del core — ese camino no deja receipt. Es defendible (una excepción es un crash, no un desenlace de fase) pero es un límite real del gobierno verificable y queda registrado como tal, no como cobertura completa.
