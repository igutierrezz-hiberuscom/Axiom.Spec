# Requisitos — INC-20260811-acc-032-launcher-telemetry

## REQ-1: Panel de estado del audit trail

El launcher debe exponer un panel read-only que muestre el resultado de
`axiom audit` (`auditTrailVerify`): status `compliant`/`absent`/`violation`,
SHA-256, line count, retention violations y external rewrite.

## REQ-2: Panel de actividad

El launcher debe leer el `audit.log` reciente y agregarlo de forma útil: conteo
por `capabilityId`/`signalType`, eventos más recientes y detección de patrones
repetidos, sin duplicar la lógica de negocio.

## REQ-3: Panel de lecciones

El launcher debe mostrar las lecciones derivadas (`axiom learn list`) y permitir
capturar una lección explícita (`axiom learn capture --text`) con
preview/confirmación, siguiendo el patrón confirm-gated del launcher.

## REQ-4: Solo lectura del audit trail

El launcher no escribe en el audit log. La única mutación es la captura de una
lección explícita, confirm-gated.