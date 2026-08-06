# Incremento: builder unico y politica local-only

Status: closed
Date: 2026-08-06
Requested work items: ACC-017, ACC-018, ACC-020 (ACC-021 absorbida en ACC-018)
Runtime repository: `Axiom/`
Canonical spec repository: `Axiom.Spec/`

## Goal

Reducir el contrato operativo del runtime a una unica configuracion funcional
`builder` y una unica politica local-only, ambas implicitas. El producto debe
seguir permitiendo elegir el target de adaptador y los roles de equipo o fase,
pero no debe pedir ni aceptar la seleccion de perfiles funcionales, overlays o
gateway. El audit trail local queda habilitado como capacidad transversal y los
servidores MCP reales se conservan.

## Context

La auditoria R-04 verifico que `product-owner`, `builder`, `standard`,
`enterprise`, `axiom-gateway` y `generated-snapshots` forman un eje de
configuracion mas amplio que el comportamiento ejecutable que el MVP necesita.
El perfil funcional controla capabilities, providers y alcance de mutacion; el
overlay controla discovery, gateway y telemetria. La implementacion real del
gateway es declarativa y no abre un daemon o broker remoto; `generated-snapshots`
tampoco tiene un backend ejecutable identificado. En cambio, el
`AuditTrailSink`, `axiom audit`, el broker MCP y sus handlers tienen codigo y
pruebas reales.

## Scope

### Incluido

- Fijar `builder` como el unico perfil funcional efectivo y eliminar
  `product-owner`, `analista`, `arquitecto` y los selectores equivalentes.
- Fijar local-only como politica efectiva de discovery y permisos, eliminando
  `standard`, `enterprise`, `gateway-first`, `axiom-gateway`,
  `gateway-state.json`, `axiom gateway start/status` y flags gateway.
- Retirar `generated-snapshots` del registry, tipos, fallbacks, routing,
  doctor, providers, fixtures y configuracion activa.
- Mantener los targets de adaptador, los roles de equipo (`backend`,
  `frontend`, `qa-e2e`) y las fases del workflow como ejes independientes.
- Normalizar estados existentes que contengan `functionalProfile`,
  `operationalOverlay` u otros valores antiguos a `builder` y `local-only`
  antes de componer o leer el estado actual. Los archivos antiguos no deben
  bloquear una instalacion o un `start` por esos identificadores.
- Mantener `AuditTrailSink`, `axiom audit`, el log append-only y su sidecar de
  integridad. La unica retencion activa sera `P365D`, conservando el mayor
  window existente y evitando crecimiento local ilimitado.
- Mantener `axiom-mcp-broker`, `sdd-mcp-server`, `spec-mcp-broker`, sus handlers
  y sus pruebas sin convertirlos en providers gateway.
- Actualizar pruebas focalizadas y eliminar pruebas que afirmen superficies
  retiradas, sustituyendolas por pruebas de compatibilidad y politica unica.

### Excluido

- Retirar targets de adaptador o roles de implementacion del equipo.
- Retirar el backend JSON local de memoria, `engram` o los handlers MCP reales.
- Introducir un daemon gateway, un broker remoto o una politica enterprise
  nueva.
- Completar capacidades aspiracionales que no tengan implementacion ejecutable.
- Integrar todavia la spec general `Axiom.Spec/specs/00..08` o `context/**`;
  esa consolidacion la hara el autopilot una sola vez al final.
- Reconciliar toda la documentacion operativa; queda en ACC-019 y en el
  incremento documental posterior.

## Acceptance criteria

- [x] `init`, `configure`, `workspace` y el launcher no exponen flags,
      aliases ni selectores de perfil funcional u overlay; el target de
      adaptador continua disponible.
- [x] Los estados nuevos materializan siempre `builder` y `local-only`, y
      los estados antiguos con `product-owner`, aliases, `standard` o
      `enterprise` se leen y normalizan sin conservar esos valores como
      contrato activo.
- [x] `@axiom/install-profiles`, `@axiom/installer`, `@axiom/core`,
      `@axiom/user-workspace`, CLI y tipos compartidos ya no requieren el
      triple seleccionable; la composicion valida la politica unica y los
      diez targets canonicos siguen resolviendo.
- [x] `providers.yaml`, capability model, config validation y tool routing no
      declaran ni resuelven `axiom-gateway`, `gateway`, `gateway-first` o
      `generated-snapshots`; la resolucion efectiva queda en filesystem/local.
- [x] No se registra `axiom gateway`, no se crea `gateway-state.json` y
      `start`/`sync` no aceptan `--gateway` ni `--no-gateway`.
- [x] `AuditTrailSink` y `axiom audit` siguen funcionando para mutaciones y
      verificaciones con una unica retencion `P365D`; los eventos no se
      bloquean por ausencia de un overlay enterprise.
- [x] Los tres brokers/handlers MCP reales se mantienen registrados y sus
      pruebas siguen pasando sin depender de `axiom-gateway`.
- [x] Doctor, readiness, build y las suites focalizadas del alcance pasan; los
      fallos preexistentes quedan clasificados y no se ocultan.
- [x] No quedan afirmaciones activas en runtime/configuracion que presenten
      `product-owner`, `standard`, `enterprise`, gateway o
      `generated-snapshots` como opciones vigentes.

## Risks and mitigations

- Los estados persistidos tienen formas historicas distintas. Se resolveran en
  un helper de normalizacion en el borde de lectura y se probaran fixtures de
  cada forma antes de cambiar el writer.
- Algunos consumidores usan `functionalProfile` y `overlay` como campos de
  trazabilidad. Se conservan los campos si son necesarios para compatibilidad,
  pero sus valores efectivos se limitan a `builder` y `local-only`.
- El barrel y los artefactos `dist/` pueden contener texto residual. Se
  regeneraran con `npm run build` y no se editaran manualmente.
- El workflow actual puede crear candidates bajo `Axiom/axiom.spec` cuando el
  cwd es el runtime, mientras freeze/receipts buscan `Axiom.Spec/specs`; el
  autopilot mantendra este README como candidate canonico y registrara la
  discrepancia como deuda de tooling, sin cambiarla dentro de este incremento.

## Open questions

Ninguna bloqueante. La retencion unica se fija en `P365D` por ser la ventana
existente mas larga y por limitar el crecimiento de disco; cambiarla a retencion
indefinida requeriria una decision independiente sobre almacenamiento.

## Assumptions

- Los diez adapter targets canonicos siguen siendo seleccionables porque son
  una dimension distinta del perfil funcional.
- `engram` sigue siendo un provider opcional de memoria local/project-scoped y
  no se confunde con `generated-snapshots`.
- La ausencia de gateway no elimina el audit trail ni el broker MCP.

## Implementation notes

Se centralizo la compatibilidad en `@axiom/install-profiles` mediante
`normalizeProfileTriple` y `normalizeInstallProfile`: los readers convierten
estados antiguos a `builder` + `local-only`, mientras los writers nuevos ya no
aceptan el triple seleccionable. `resolveInstallProfile` recibe solo el target
y conserva los diez targets canonicos.

La registry activa queda en cuatro providers locales (`filesystem`, `serena`,
`cmm`, `engram`) y un discovery profile `local-only`. Se eliminaron gateway,
generated snapshots, el check `GW-001`, el modulo/comando CLI gateway, el
subcomando `roles add` y los flags de start; las operaciones `axiom.*` quedan
en la frontera MCP y no se desvían al routing genérico.

El audit trail permanece append-only con sidecar SHA-256, `axiom audit` sigue
siendo read-only y la ventana única es `P365D`, incluso si falta el YAML o se
instancia el sink directamente. Los brokers MCP y sus handlers no se
modificaron funcionalmente. Los ejemplos MCP se actualizaron para resolver con
filesystem/engram. `workspace.json`, `context status` y `member install`
normalizan también los estados persistidos legacy antes de escribirlos.

## Validation

Ejecutado en `Axiom/`:

- `npm run build`: pasa.
- `npx vitest run`: 326 archivos y 3313 pruebas pasan con `testTimeout: 30000`
  fijado en `Axiom/vitest.config.ts` para la suite I/O-heavy. La reducción
  frente a 3323 corresponde a diez casos de combinaciones legacy de
  perfiles/overlays retiradas durante la remediación del review.
- Suites focalizadas de install-profiles, installer, capability-model,
  tool-routing, doctor, components, telemetry, CLI, MCP, providers y topology:
  pasan.
- El barrido de runtime/configuracion no encontró opciones activas antiguas;
  las referencias restantes en `normalize.ts`, composer y routing son filtros
  de compatibilidad de lectura para estados persistidos.
- `npm run doctor`: PASS, 45/60 OK, 0 fallos, 4 advertencias y 11 omitidos.
  `CC-004` queda como warning honesto por tres capabilities opcionales sin
  provider (`code.symbolSearch`, `code.referenceSearch`, `code.impactAnalysis`).
- `npm run readiness:first-project`: PASS; secuencia `init -> configure ->
  toolchain repair -> sync -> start -> audit -> doctor`.
- Validación posterior al review: 3 archivos y 22 pruebas focalizadas de
  catálogos de agents/skills y contrato de review pasan; la suite completa
  estándar vuelve a quedar verde después de sincronizar los bundleHash.

No se identificaron fallos preexistentes en la suite completa de esta pasada.

## Result

Implementacion de runtime completada y validada. El incremento queda `closed`
tras la revisión independiente, la verificación de receipts, la integración
canónica y la validación operativa final.

La ejecución del lifecycle desde `Axiom/` sigue materializando un candidato
auxiliar bajo `Axiom/axiom.spec/increments/`, mientras freeze/receipts y la
integración canónica usan `Axiom.Spec/specs/increments/`. Se conserva el
artefacto product-owned y se deja registrada esta incoherencia de tooling; no
se elimina de forma destructiva.

## General spec integration

Integrada en una única pasada del autopilot sobre `Axiom.Spec/specs/00..08` y
`Axiom.Spec/context/**`.
