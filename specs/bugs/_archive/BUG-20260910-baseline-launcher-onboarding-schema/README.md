# baseline launcher onboarding request schema

> **Código**: BUG-20260910-baseline-launcher-onboarding-schema
> **Estado**: Reportado
> **Fecha de reporte**: 2026-09-10
> **Severidad**: (pendiente de completar)
> **Prioridad**: (pendiente de completar)

## Resumen del defecto

Los endpoints de onboarding del Launcher rechazan cuerpos válidos con `400`
antes de aplicar las validaciones semánticas de paths solapados, paths
absolutos y adopción. El schema runtime no coincide con el contrato consumido
por los tests.

## Contexto conocido

Fallaron ocho escenarios de `apps/cli/tests/launcher-onboarding-migration.test.ts`:
previews y confirmaciones de setup/adopt devuelven `400`; casos que esperan
mensajes `superpone`, `absoluto` u `otro proyecto` reciben `El body contiene
campos no permitidos.`

## Clasificación funcional

Contrato de entrada HTTP/Launcher y validación de onboarding; independiente de
R13-3 y de su API headless.

## Comportamiento actual

El parser de requests aplica un allow-list que descarta campos que el contrato
de setup/adoption necesita, impidiendo llegar al validador de negocio.

## Comportamiento esperado

Los cuerpos válidos se aceptan en preview y confirmación, los inválidos se
rechazan con el diagnóstico semántico esperado y preview/dry-run no modifica
filesystem ni registry.

## Reproducción

(pendiente de completar)

### Precondiciones

(pendiente de completar)

### Pasos

(pendiente de completar)

### Resultado observado

HTTP `400` en casos que esperan `200` o `409`, y mensajes de campos no
permitidos en lugar de overlap/absolute/ownership.

## Superficie de regresión

Handlers HTTP del Launcher y esquemas/normalizadores de workspace setup y
adoption.

## Estructura mínima del bug

Mantener preview y confirmación sobre los mismos contratos; no mover lógica de
workspace al runner self-update.

## Material adicional

Log: `%TEMP%\\axiom-r13-baseline-20260910.log`.

Suite: `Axiom/apps/cli/tests/launcher-onboarding-migration.test.ts`.

## Trazabilidad y fuentes

Baseline global del 2026-09-10; los fallos se clasifican fuera de R13-3.

## Estado de validación humana

Pendiente de reproducción aislada, corrección del schema y validación de
read-only/idempotencia.
