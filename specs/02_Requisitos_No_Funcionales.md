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

## NFR-AXM-011 Evolución de schema solo aditiva

Todo cambio de schema persistido (registro de usuario, manifiesto `axiom.yaml`, `metadata.yml` de artefactos, `mcp.yml`) debe convivir de forma aditiva junto al formato anterior, seleccionado por presencia de fichero o por campo `schemaVersion`, en vez de reemplazar/renombrar destructivamente. Verificado en la migración de registro (`~/.axiom/projects.yml` `schemaVersion: 2` añadido junto a `~/.axiom/registry.json` `schemaVersion: 1`, ambos leídos/escritos por el mismo paquete) y en el cutover de `axiom.yaml` (`schemaVersion: 1` y `2` ambos soportados por `resolveProject` y los checks `MC-001`/`BC-001`/`BC-002` de `@axiom/doctor`, nunca resolviendo mal uno como el otro).

## NFR-AXM-012 Sin caché persistente hasta que el volumen real lo justifique

Ningún subsistema debe introducir una caché persistente en disco (p. ej. un índice de artefactos) sin una necesidad de rendimiento medida o un consumidor concreto que la requiera. Verificado: `axiom index rebuild`/`validate` son wrappers de escaneo directo sobre `listArtifacts`, sin fichero `.axiom/cache/*.index.json`; ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md).

## NFR-AXM-013 Disciplina de alcance mínimo primero

Toda extensión del producto debe auditar primero el paquete existente que ya cubre parcialmente la necesidad, y extenderlo de forma aditiva, en vez de construir una infraestructura paralela. Principio aplicado de forma sistemática por el roadmap de rediseño de 23 incrementos (cada incremento llevó un migration-engineer como primer subagente, con la tarea explícita de auditar antes de escribir contrato nuevo) — ver `specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`.

## NFR-AXM-014 Aislamiento por construcción

Los mecanismos de aislamiento (project-scoping, MCP allowlist, límite de dogfooding) deben derivarse de la resolución de topología/proyecto en vez de hardcodear nombres de repo o rutas, de forma que funcionen correctamente sobre cualquier proyecto/topología de terceros, no solo sobre este workspace. Verificado en el check `DF-001` (dogfooding, parametrizado por rol vía `TopologyManifest`, ver [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md)) y en el aislamiento de `mcp.yml` por scoping de filesystem dentro del repo de spec de cada proyecto (ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)).