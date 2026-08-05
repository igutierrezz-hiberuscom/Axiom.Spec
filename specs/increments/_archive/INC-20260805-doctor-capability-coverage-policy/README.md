# Increment: política de cobertura de capabilities en Doctor

Status: closed
Date: 2026-08-05

## Goal

Convertir la intención de `CC-004` en una política explícita y verificable:
Doctor debe comparar el catálogo provider-routed con las capabilities
configuradas y con los providers declarados, excluyendo de forma explícita las
capabilities MCP-only y diferenciando la severidad según su obligación.

## Context

El bug de `CC-004` corrige el universo circular, pero por sí solo no decide
qué hacer con una capability requerida, opcional, post-MVP, deshabilitada o
servida por MCP. Esta política debe evitar tanto el PASS falso actual como los
falsos positivos que aparecerían si se exigiera un provider tradicional para
cada handler MCP.

## Scope

- `Axiom/packages/doctor/src/checks.ts`: implementar la comparación de
  catálogo, configuración y providers, y producir estados `fail`, `warn`,
  `pass` o `skip` coherentes.
- `Axiom/packages/doctor/tests/capability-model.test.ts`: fixtures reales y
  adversariales para cada clase de cobertura.
- `Axiom/packages/capability-model/src/constants.ts` y exports solo si el
  incremento MCP-only necesita exponer una clasificación estable.
- `Axiom/axiom.config/capabilities.yaml` únicamente si hace falta declarar de
  forma no ambigua una clase o estado ya acordado; no añadir providers
  ficticios para silenciar Doctor.

## Non-goals

- No crear providers nuevos ni implementar handlers MCP.
- No cambiar la selección de providers ni el algoritmo general de `routeTool`.
- No convertir una capability opcional ausente en un fallo bloqueante.
- No tratar una capability MCP-only como huérfana del provider registry.
- No modificar otras comprobaciones de Doctor salvo lo imprescindible para
  compartir helpers o evidencia.
- No integrar todavía la spec general ni `context/**`; lo hará el autopilot al
  final del lote.

## Acceptance criteria

- [x] `CC-004` usa un universo independiente del conjunto observado en
      `providers.yaml`.
- [x] Las capabilities MCP-only quedan excluidas de la obligación de provider
      tradicional mediante una regla explícita y testeada.
- [x] Una capability provider-routed requerida, declarada pero sin provider,
      produce `fail` y menciona su id.
- [x] Una capability provider-routed opcional o post-MVP sin provider produce
      `warn` o el estado no bloqueante documentado, nunca un PASS falso.
- [x] Una capability deshabilitada o no disponible no produce un fallo de
      cobertura activa sin evidencia de que deba estar operativa.
- [x] Una capability canónica ausente de `capabilities.yaml` se distingue de
      una capability declarada pero no servida por provider.
- [x] El proyecto real deja de mostrar `CC-004 PASS` engañoso y el resultado
      explica las capabilities faltantes y su severidad.
- [x] Las demás comprobaciones CC-001..CC-006 mantienen su contrato.
- [x] Build y pruebas focalizadas de Doctor/capability-model pasan.
- [x] Existe un phase receipt `verify` con la evidencia de validación ejecutada.

## Open questions

Ninguna bloqueante. Se adopta esta política: provider-routed requerida sin
provider = `fail`; provider-routed opcional o post-MVP sin provider = `warn`;
MCP-only = fuera de CC-004; deshabilitada/no disponible = no fallo activo,
pero visible si la evidencia es relevante.

## Assumptions

- `CANONICAL_CAPABILITY_IDS` representa el catálogo provider-routed después del
  incremento MCP-only.
- Las listas de `capabilities.yaml` aportan la clase de cumplimiento y
  `capabilityStates` aporta el estado operativo.
- `capabilities` y `applicableCapabilities` de un provider son equivalentes
  para calcular qué sirve el registry, con prioridad al campo disponible.

## Implementation notes

Se auditó la implementación existente y se mantuvo el cambio enfocado en
Doctor. `CC-004` valida y carga `capabilities.yaml`, toma como universo las 16
capabilities provider-routed canónicas y filtra de forma explícita
`MCP_ONLY_CAPABILITY_IDS` antes de comparar la cobertura declarada en
`providers.yaml`. La evidencia separa `Sin declarar`, `Requeridas sin provider`,
`Opcionales sin provider` e `Inactivas sin provider`; una capability requerida
activa produce `fail`, una opcional o post-MVP produce `warn`, y una
deshabilitada/no disponible queda visible sin fallo activo.

Se ampliaron las pruebas adversariales con la clase post-MVP. No se añadieron
providers ficticios ni se modificaron `config-validation`, installer, perfiles
MCP-only u otros paquetes fuera del alcance.

## Validation

- `npm run build`: paso (`tsc -b`).
- `npm exec vitest run packages/doctor/tests/capability-model.test.ts`: 1
  archivo, 21 pruebas pasaron.
- `npm exec vitest run packages/doctor/tests packages/capability-model/tests packages/config-validation/tests`:
  24 archivos, 266 pruebas pasaron.
- `get_errors` sobre `packages/doctor/src/checks.ts` y
  `packages/doctor/tests/capability-model.test.ts`: sin errores.
- Los casos `unavailable`, `capabilities.yaml` ausente y YAML inválido están
  cubiertos por pruebas dedicadas.
- Verificación directa del proyecto real: `CC-004` = `fail`, con 7 de 16
  capabilities servidas; seis requeridas y tres opcionales activas sin
  provider aparecen en la evidencia. Esto es el resultado esperado y elimina
  el PASS falso previo.
- `checkCandidateFreeze(INC-20260805-doctor-capability-coverage-policy)` con
  `AXIOM_TEST_FORCE_JSON=1`: `ok: true` antes de actualizar esta documentación;
  después se re-congeló el mismo candidato con `--force-json` y la comprobación
  final volvio a dar `ok: true`.
- Receipt verificable final: `receipts/2026-08-05T15-53-50.489Z-verify-success.json`
  (hash `af28bae85f85b0a0359f5e84f85cd41dd8de6800e06c6e4bde40d2050dafa95a`).

## Result

Implementación completada y validada. Doctor ya no obtiene el universo de
CC-004 desde las capabilities observadas en providers: compara el catálogo
canónico provider-routed con la declaración, compliance, estado y cobertura
real. Las capabilities `axiom.*` MCP-only no generan una obligación de provider
tradicional. El proyecto dogfooded queda correctamente en `fail` por las
capabilities activas sin provider, con evidencia útil para corregir la
configuración.

La integración estable en `Axiom.Spec/specs/00..08` y `context/**` quedó
consolidada durante la pasada única de R-04.

## General spec integration

Integrado en la pasada única de R-04:

- `specs/01_Requisitos_Funcionales.md`: requisito funcional de cobertura
  canónica y diagnóstico de `CC-004`.
- `specs/02_Requisitos_No_Funcionales.md`: NFR de ausencia de PASS falso y
  severidad por clase/estado.
- `specs/06_Integraciones_y_Capacidades.md`: política vigente de cobertura y
  resultado dogfooded 7/16.
- `specs/07_Gobierno_y_Seguridad.md`: regla de evidencia y severidad de
  Doctor.
- `context/operations/02-doctor-troubleshooting-y-telemetria.md`: contrato
  operativo de `CC-004` y errores de `capabilities.yaml`.

No se crearon ni eliminaron documentos de `context/**`; se actualizaron los
documentos propietarios existentes.

El artefacto queda archivado por el autopilot.
