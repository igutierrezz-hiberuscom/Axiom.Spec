# 07 Gobierno y Seguridad

## Gobierno

1. una única fuente de verdad por capa;
2. no mezclar runtime, spec y tooling en la misma responsabilidad (regido por `Axiom.SDD/AGENTS.md` para este workspace);
3. no promover estructuras legacy como canónicas por comodidad;
4. límites explícitos de bootstrap (`Axiom.SDD/AGENTS.md`): no introducir sin pedido explícito índices obligatorios, sistemas de metadata complejos, carpetas de lifecycle enterprise, integraciones de work items (Azure DevOps/Jira) como dependencia obligatoria, abstracciones MCP obligatorias, lógica de Workbench, frameworks pesados de multi-agente, jerarquías autogeneradas profundas, ni scripts que no existan aún.

## Seguridad operativa y compliance (verificado en runtime)

1. **GATEs verificables por doctor**: GATE 0010 (overlay `enterprise` exige `audit-trail-sink`), GATE 0031 (6 adapter packages deben tener `src/generator.ts` + `dist/index.js`), GATE 0024 (memoria no es fuente de verdad; spec prevalece en conflicto), GATE 0033 (agents como contratos materializables, sin ejecución).
2. **Aislamiento project-scoped**: `@axiom/isolation` aplica path-guard y una lista de MCP servers permitidos por defecto; ninguna cache, binding o entrada de memoria debe cruzar `projectId`.
3. **Audit trail**: `axiom audit` verifica SHA-256, conteo de líneas, retención y detección de reescritura externa, devolviendo `compliant` | `absent` | `violation` (exit 1 en violación).
4. **Telemetría por overlay**: `telemetry-sinks.yaml` define `dataSensitivityBoundaries` por overlay (tags permitidos/redactados, nivel de redacción, flujo cross-project, ventana de retención) y una lista de sinks (`null-sink`, `log-sink`, `remote-sink`, `audit-trail-sink`). `axiom sync` aborta antes de mutar si el overlay exige señales mínimas y no hay sink habilitado que las cubra. No hay documentado un mecanismo formal de opt-out completo de telemetría (verificado como ausente, no como implementado).
5. **Doctor como gate de gobierno mínimo**: valida presencia y validez de `axiom.yaml`, `integrations.yaml`, `policy-as-code.yaml`, protección de `.sdd/local/` frente a versionado accidental, y aislamiento project-scoped.
6. **Policy-as-code**: `policy-as-code.yaml` concentra `sensitivityTags`, `artifactLifecycle` (transiciones y verificación antes de archive), reglas de `tools`/`compliance` (actitud ante herramientas faltantes o MCP no aprobados), `projectIsolation` y `doctorValidation`.

## Reglas de cierre (incrementos y bugs, vigentes en este workspace)

Un incremento o bug solo puede marcarse `closed` si: el objetivo/comportamiento esperado es claro, existen acceptance criteria, se implementaron cambios (o hay justificación explícita de no-code), se ejecutó la validación disponible, se revisó contra el intent original y los acceptance criteria, y se integró conocimiento estable en `Axiom.Spec`. Si falta cualquiera de estos puntos, el estado debe quedar `pending` con motivo explícito — nunca `closed` por comodidad.