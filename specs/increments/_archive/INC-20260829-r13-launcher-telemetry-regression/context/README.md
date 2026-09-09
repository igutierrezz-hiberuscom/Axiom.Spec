# Context

## Propósito

Conservar el contrato técnico estable de tail bounded, orden y scopes de telemetría R-13 y la evidencia reproducible de ACC-076.

## Hechos estables reconciliados

- `readAuditTrailTail({ projectRoot, maxEvents, maxBytes })` es la API sancionada para la ventana bounded/validated; el launcher no lee `audit.log` directamente.
- La proyección devuelve `schemaVersion: 2`, separa `projectMetrics` de `processMetrics` (`scope: process-wide`) y deriva los 20 eventos recientes newest-first desde la ventana tail.
- Dos roots producen snapshots independientes; corrupción/IO se reporta de forma tipada y `lessons` permanece ausente.
- La matriz `apps/cli/tests/r13-acc-076-matrix.test.ts` cubre ACC-075 y el agregado ACC-076 con fixtures loopback/fake ADO, sin red externa.

## Evidencia

Resultado exacto: 35 casos PASS, 0 FAIL, 0 TIMEOUT; 14 checks de ausencia de mutación y un test de resumen que afirma esos conteos.

## Qué no debe vivir aquí

No deben vivir metadata, status, IDs, links, receipts manuales, índices generados, secretos ni la spec normativa completa. El lifecycle se gestiona con Axiom Core.
