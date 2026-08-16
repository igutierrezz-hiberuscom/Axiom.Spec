# Panel de telemetría/auditoría en el launcher web

> **Código**: INC-20260811-acc-032-launcher-telemetry
> **Estado**: closed
> **Fecha de creación**: 2026-08-11
> **Tipo de cambio**: agregar
> **Acción de auditoría**: ACC-032
> **Plan padre**: `PLAN-REVISION-INTEGRAL-AXIOM`

## Objetivo

Añadir un apartado de telemetría/auditoría al launcher web (`axiom app`) que
explote los datos que hoy solo se escriben y se verifican, para que el operador
pueda ver qué hace el runtime y qué ha aprendido, sin tocar la terminal.

## Contexto y motivación

El audit trail se escribe siempre (cada mutación) pero su contenido solo se
explota por `axiom learn --from-audit` (opt-in, no conectado automáticamente) y
por `axiom context` (bloque `recentLessons`). Los contadores del bus
(`getCounters`) no tienen superficie pública. El launcher ya tiene paneles
Doctor y ADO, por lo que un panel de telemetría encaja en la superficie
existente.

## Alcance tipado

- `Axiom/apps/cli/src/commands/app-launcher*.ts` (endpoints del panel).
- `Axiom/apps/cli/static/launcher/` (front del panel).
- `Axiom/packages/telemetry` (`auditTrailVerify`, `AuditTrailSink`,
  `getTelemetryBus`) — solo lectura.
- `Axiom/packages/memory` (`extractLessons`, `recallLessons`) — solo lectura.
- `Axiom/apps/cli/src/commands/{audit,learn,context}.ts` — reutilizar lógica,
  no duplicarla.

## Decisiones cerradas

- El launcher solo LEE el audit trail; la única mutación es la captura de una
  lección explícita, que debe ser confirm-gated.
- Se reutiliza `auditTrailVerify` y el patrón de lectura de `learn.ts`/
  `extractLessons`; no se duplica lógica de negocio.
- El panel muestra: estado del audit trail (`compliant`/`absent`/`violation`,
  SHA-256, line count, retention violations, external rewrite), actividad
  agregada (conteo por `capabilityId`/`signalType`, eventos recientes) y
  lecciones (`axiom learn list` + captura confirm-gated).

## No incluido

- No escribir en el audit log desde el launcher.
- No duplicar la lógica de negocio de telemetría/memoria.
- No añadir un nuevo engine; solo superficies sobre engines existentes.

## Criterios de aceptación

- [x] El launcher expone un panel de telemetría/auditoría read-only.
- [x] El panel muestra el estado del audit trail y la actividad agregada.
- [x] El panel muestra lecciones y permite capturar una lección explícita con
      preview/confirmación.
- [x] No hay regresiones en los paneles existentes (Doctor, ADO).
- [x] La validación disponible (build/tests) pasa sin regresiones nuevas.

## Resultado de implementación

Implementado por el ejecutor del incremento (2026-08-11).

### Cambios

- `Axiom/apps/cli/src/commands/app-launcher-telemetry.ts` (nuevo): endpoints
  THIN del panel — `GET /launcher/telemetry` (estado del audit trail vía
  `auditTrailVerify`, actividad agregada por `capabilityId`/`signalType` +
  eventos recientes, contadores del bus vía `getTelemetryBus().getCounters()`,
  lecciones vía `runLearnList`) y `POST /launcher/telemetry/lessons` (captura
  de lección explícita confirm-gated vía `runLearnCapture`). Sin nuevo engine.
- `Axiom/apps/cli/src/commands/app-api.ts`: routing + dispatcher de los dos
  endpoints.
- `Axiom/apps/cli/static/launcher/transport.js`: mensajes `requestTelemetry` /
  `captureLesson`.
- `Axiom/apps/cli/static/launcher/index.html`: tab "Telemetría" + sección del
  panel.
- `Axiom/apps/cli/static/launcher/panels.js`: render del panel (audit trail,
  actividad, lecciones) + formulario de captura preview → confirm.
- `Axiom/apps/cli/static/launcher/launcher.js`: `switchView` incluye la vista
  `telemetry`.
- `Axiom/apps/cli/tests/launcher-telemetry.test.ts` (nuevo): 5 tests del panel.

### Decisiones aplicadas

- El launcher solo LEE el audit trail; la única mutación es la captura de una
  lección explícita, confirm-gated (preview hasta `confirmed:true`).
- No se duplica lógica de negocio: se reutilizan `auditTrailVerify`,
  `getTelemetryBus`, `runLearnList` y `runLearnCapture`.
- No se añade un nuevo engine; solo superficies sobre engines existentes.

### Validación

- `npm run build` (tsc -b): OK.
- `npx vitest run` sobre las suites del launcher (app-launcher, launcher-panels,
  launcher-ado-workflow, launcher-doctor, launcher-front-no-vscode,
  launcher-push, launcher-onboarding, launcher-ado-bridge, launcher-ado-peripherals,
  launcher-execution-mode, launcher-telemetry): todas pasan (144 tests).
- Sin regresiones nuevas; los fallos no aplican (ninguno introducido).

### Criterios

Todos los criterios de aceptación se cumplen (CA-1 a CA-4).