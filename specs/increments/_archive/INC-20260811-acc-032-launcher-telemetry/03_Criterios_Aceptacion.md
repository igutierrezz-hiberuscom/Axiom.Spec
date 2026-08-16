# Criterios de aceptación — INC-20260811-acc-032-launcher-telemetry

## CA-1: Panel de telemetría/auditoría

- [x] El launcher expone un panel de telemetría/auditoría read-only.
- [x] El panel muestra el estado del audit trail (`compliant`/`absent`/
      `violation`, SHA-256, line count, retention violations, external rewrite).

## CA-2: Actividad agregada

- [x] El panel muestra actividad agregada por `capabilityId`/`signalType` y
      eventos recientes.

## CA-3: Lecciones

- [x] El panel muestra lecciones (`axiom learn list`).
- [x] Permite capturar una lección explícita con preview/confirmación.

## CA-4: Sin regresiones

- [x] No hay regresiones en los paneles existentes (Doctor, ADO).
- [x] La validación disponible (build/tests) pasa sin regresiones nuevas.