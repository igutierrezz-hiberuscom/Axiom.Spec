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

## NFR-AXM-007 Escritura atómica e idempotencia

Toda escritura de superficies generadas (skills, agents, components, `install-profile.json`, adapters) debe hacerse vía patrón tmp+rename, y debe ser idempotente byte a byte cuando el input no cambia. Verificado en `@axiom/skills`, `@axiom/agents`, `@axiom/document-bootstrap`, `@axiom/persistence`.

## NFR-AXM-008 Aislamiento project-scoped

Ninguna ruta, cache, MCP binding o entrada de memoria debe cruzar el `projectId` de origen. Verificado en `@axiom/isolation` (path-guard, `DEFAULT_ALLOWED_MCP_SERVERS`) y en las reglas GATE 0024 (`@axiom/memory`: spec prevalece sobre memoria en conflicto).

## NFR-AXM-009 Auditabilidad de telemetría

Cada overlay operacional (`local-only`, `standard`, `enterprise`) debe declarar señales mínimas de telemetría (`minimumSignals`) y sinks habilitados; `axiom sync` debe abortar antes de mutar si el gate no se satisface. Verificado en `Axiom/docs/configuration/telemetry-and-isolation.md` y `axiom audit`.

## NFR-AXM-010 Cobertura runtime verificable por doctor

Cada capa nueva (adapters, agents, toolchain) debe tener un GATE verificable por `@axiom/doctor` (p. ej. GATE 0031: los 6 adapter packages deben tener `src/generator.ts` + `dist/index.js` materializados, o falla). No basta con documentación: el doctor debe poder comprobarlo en runtime.

Complementariamente, la suite e2e `apps/cli/tests/e2e/` (`INC-20260708-workspace-e2e-tests`) da una garantía de calidad end-to-end para el setup de workspace + MCP: crea in-process un workspace multi-repo completo (repo de control + spec + rol custom, migración de registry legado v1→v2, base de spec, skills por repo de código, ambos adapters) **Y** lanza los servers reales `axiom mcp serve` sdd+spec como procesos hijo, probando que responden `initialize`/`tools/list`/`tools/call` sobre stdio JSON-RPC usando exactamente el `command`/`args` que el propio setup genera en `.axiom/mcp.yml`. Es cobertura test-only (sin cambio de producto): verifica que la config MCP generada es efectivamente lanzable, no solo declarativamente correcta. `INC-20260708-adapters-depth` amplió esta suite con un e2e de adapters (`adapters.e2e.test.ts`) que materializa in-process los 8 `ADAPTER_TARGETS` en un workspace y verifica el set completo de ficheros declarado en `GENERATED_FILES_BY_TARGET` por target (ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md)).

**Estado de la suite: verde (0 fallos).** Supersede cualquier nota previa de "fallos conocidos": tras `INC-20260708-product-repo-self-bootstrap` (que llevó los fallos de 10 a 4 sembrando la baseline canónica ausente) e `INC-20260708-fix-longstanding-test-failures` (que cerró los 4 restantes — todos defectos de test, no de producto: asserts obsoletos y un test no-hermético dependiente del reloj real), la suite completa del repo `Axiom/` pasa sin fallos. Los incrementos posteriores de esta tanda solo añadieron tests (0 regresiones en cada cierre). Nota de taxonomía reconciliada: el model-routing tiene **7 slots** canónicos (`increment, bug, plan, implementation, qa-e2e, review, archive`), no 10 — cualquier cifra "10 slots" en documentación previa es obsoleta.

## NFR-AXM-011 Evolución de schema solo aditiva

Todo cambio de schema persistido (registro de usuario, manifiesto `axiom.yaml`, `metadata.yml` de artefactos, `mcp.yml`) debe convivir de forma aditiva junto al formato anterior, seleccionado por presencia de fichero o por campo `schemaVersion`, en vez de reemplazar/renombrar destructivamente. Verificado en la migración de registro (`~/.axiom/projects.yml` `schemaVersion: 2` añadido junto a `~/.axiom/registry.json` `schemaVersion: 1`, ambos leídos/escritos por el mismo paquete) y en el cutover de `axiom.yaml` (`schemaVersion: 1` y `2` ambos soportados por `resolveProject` y los checks `MC-001`/`BC-001`/`BC-002` de `@axiom/doctor`, nunca resolviendo mal uno como el otro).

## NFR-AXM-012 Sin caché persistente hasta que el volumen real lo justifique

Ningún subsistema debe introducir una caché persistente en disco (p. ej. un índice de artefactos) sin una necesidad de rendimiento medida o un consumidor concreto que la requiera. Verificado: `axiom index rebuild`/`validate` son wrappers de escaneo directo sobre `listArtifacts`, sin fichero `.axiom/cache/*.index.json`; ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md).

## NFR-AXM-013 Disciplina de alcance mínimo primero

Toda extensión del producto debe auditar primero el paquete existente que ya cubre parcialmente la necesidad, y extenderlo de forma aditiva, en vez de construir una infraestructura paralela. Principio aplicado de forma sistemática por el roadmap de rediseño de 23 incrementos (cada incremento llevó un migration-engineer como primer subagente, con la tarea explícita de auditar antes de escribir contrato nuevo) — ver `specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`.

## NFR-AXM-014 Aislamiento por construcción

Los mecanismos de aislamiento (project-scoping, MCP allowlist, límite de dogfooding) deben derivarse de la resolución de topología/proyecto en vez de hardcodear nombres de repo o rutas, de forma que funcionen correctamente sobre cualquier proyecto/topología de terceros, no solo sobre este workspace. Verificado en el check `DF-001` (dogfooding, parametrizado por rol vía `TopologyManifest`, ver [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md)) y en el aislamiento de `mcp.yml` por scoping de filesystem dentro del repo de spec de cada proyecto (ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)).

## NFR-AXM-015 Operaciones incrementales sobre instalaciones existentes (hueco de 7 operaciones — parcialmente cerrado)

`axiom configure` es single-shot (re-aplica el perfil persistido completo) y no cubre añadir/quitar incrementalmente los elementos de un workspace ya inicializado. El "hueco de 7 operaciones" (referenciado desde [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md) RF-AXM-022 y [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md)) enumera las mutaciones incrementales ausentes: añadir/quitar repo, rol, adapter y provider/tool-MCP.

**Parcialmente cerrado — `INC-20260708-incremental-operations`**: las 4 operaciones de tipo ADD ya existen como comandos CLI idempotentes y no-clobber que resuelven el proyecto desde cwd y reusan el motor multi-repo existente (`runWorkspaceSetup` + helpers), sin duplicar lógica de scaffolding: `axiom repo add`, `axiom adapter add <target>`, `axiom provider add <id>`, `axiom role add <roleId> --path <path>` (ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md) y [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)). Las 4 operaciones de tipo REMOVE (`repo/adapter/provider/role remove`) quedan explícitamente **diferidas** como follow-up simétrico, no implementadas. Es una capacidad genuinamente nueva del runtime del producto (no tooling de desarrollo).

## NFR-AXM-016 Economía de tokens y foco del agente vía tuning por adapter (`INC-20260715-adapter-agent-tuning`)

Los prompts que el launcher genera deben poder pedir al agente un modo de trabajo terse y económico en tokens sin cambiar la lógica de negocio. Verificado: `agentTuning` por adapter (`verbosity`/`personality`/`model?`) inyecta un preámbulo determinista ("responde solo lo necesario, minimiza tokens") en el prompt pregenerado; de serie `verbosity:'low'`+`personality:'pragmatic'`. Es prompt-shaping puro y opcional (sin `agentTuning` no hay preámbulo → retrocompatible), desacoplado de model-routing y selección de providers. Capacidad en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md), superficie en [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md).