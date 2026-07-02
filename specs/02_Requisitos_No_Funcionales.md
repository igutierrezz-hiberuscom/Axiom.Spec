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