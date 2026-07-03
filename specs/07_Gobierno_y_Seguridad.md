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

## Límite de dogfooding (check `DF-001`) — roadmap de rediseño, cerrado

Regla: "Axiom se desarrolla con Axiom, pero Axiom no contiene su propia factoría interna como parte del producto instalable."

- `runDogfoodingBoundaryChecks` (`Axiom/packages/doctor/src/checks.ts`, id de check `DF-001`, categoría `dogfooding`) comprueba que ningún repo con rol `code` (`TopologyManifest.roleCodeRepositories`) tenga una dependencia física — una dependencia local de `package.json` (`file:`/`link:`/path relativo) o un literal de path `require`/`import` — que resuelva dentro del path de filesystem resuelto del repo con rol `sdd` o `spec`. Estrictamente unidireccional: que `sdd`/`spec` referencien a `code` para contexto es esperado y está fuera de alcance; solo se marca `code -> sdd`/`code -> spec`.
- Parametrizado por rol por diseño, no hardcodeado por nombre: se apoya en `sddRepo`/`specRepo`/`roleCodeRepositories` de `TopologyManifest`, así que funciona de forma significativa sobre los nombres de repo de cualquier proyecto de terceros, no solo sobre el layout de este workspace.
- Postura fail-open, con muchos `skip` (sin estado `warn`): se salta cuando el proyecto no está `resolved`, cuando `topology.yaml` falla al cargar, cuando `manifest.mode === 'single-repo'` (no es posible ningún límite cross-repo), o cuando `roleCodeRepositories` está vacío. Un `fail` requiere un match real y concreto de path resuelto; si no, `pass`.
- Escaneo mínimo suficiente (sin parseo AST, sin resolución de alias de bundler, sin recorrido transitivo de `node_modules`): un escaneo de dependencies/devDependencies/optionalDependencies de `package.json` acotado a los globs `workspaces` declarados propios del repo, más un grep acotado de literales de path `require(...)`/`from '...'`, excluyendo paths generados conocidos vía `aggregateKnownGeneratedPathGlobs`.
- Este propio repo `Axiom` no tiene `axiom.spec/config/topology.yaml`, así que `DF-001` reporta `skip` (no `pass`) al correr aquí hoy — un hueco esperado e intencional ligado a que este workspace todavía no se declara a sí mismo como `mode: multi-repo`, no un defecto. Levantar un `topology.yaml` real para este workspace queda como trabajo futuro diferido (reflejo de D1, ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md)).

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

Todo check de doctor futuro para validez de `mcp.yml`, o para el registro de capability/provider de `@axiom/mcp-tools`, queda diferido — sin instalador/scaffold que escriba `capabilities.yaml`/`providers.yaml`/`mcp.yml` a ningún proyecto real todavía, no hay call site de producción que comprobar.

## Trazabilidad de write-scope y ownership (ángulo de gobierno)

`validateWriteScope` (ver [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md)) es el mecanismo de gobierno que impide que un plan mute paths fuera de su `allowedWriteScope` declarado, o que repos con rol `sdd` reciban cambios no declarados (`unexpected-sdd-change`), o que se manipulen paths generados/cache conocidos (`generated-cache-tampering`). Es el mecanismo concreto que hace cumplir en runtime la separación de ownership entre repos que exige esta sección de gobierno — dos superficies (`axiom validate changes` y el check `WS-001`) comparten una única primitiva sin lógica duplicada.

## Regla conocida de build (no duplicada aquí)

`Axiom/packages/cli-commands/tsconfig.json` tiene un defecto de tooling de build que rompe `--help` para varios comandos CLI transitivamente dependientes de ese paquete. Rastreado íntegramente en `Axiom.Spec/bugs/BUG-20260702-cli-commands-tsconfig-missing-emit` (status: pending) — no se duplica el detalle aquí.