# R-10 ACC-043 contrato QA único para archive

> **Código**: INC-20260817-r10-acc043-unify-qa-archive-gate
> **Tipo de cambio**: modificar
> **Fecha de creación**: 2026-08-17

## Resumen

Eliminar las dos implementaciones divergentes de `checkQaArchiveGate` y establecer una única evaluación de QA para todo archive gobernado de incrementos y bugs.

## Contexto y motivación

El runtime contiene un helper en `qa-archive-gate.ts` y otro en `axiom-qa-e2e.ts`. Difieren al interpretar ausencia de estado, `failed` y `cancelled`, y el runner actual recibe adaptadores que degradan errores a avisos. Por ello la semántica del archive depende de la ruta (CLI, launcher o integrate) en vez de la política de QA declarada.

## Alcance

### Incluido

- Un contrato único de evaluación QA, expresivo y tipado, consumido en la frontera común de `runGovernedTransition`.
- La clasificación explícita de evidencia QA en `passed`, `failed`, `cancelled` y `pending`, más un diagnóstico fail-closed cuando no pueda evaluarse una política requerida.
- Política uniforme: `qaLane: parallel` permite archive y expone el estado QA; QA `inline` o declarada requerida bloquea archive salvo evidencia `passed`.
- Cableado de increment, bugs, `integrate`, launcher y MCP a la misma decisión del runner, con preview no mutante que muestra la decisión.
- Retirada de las dos implementaciones locales y de sus tipos/exports obsoletos, sin duplicar el contrato en adaptadores.
- Pruebas combinatorias de la política y pruebas de integración por superficie afectada.

### Excluido

- Cambiar la máquina de estados pura, crear estados de lifecycle nuevos o redefinir la ejecución de `axiom-qa-e2e`.
- Cambiar la política de confirmación, receipts, archive compensatorio o aprobación de planes de ACC-041/042.
- Añadir configuración enterprise, un dispatcher externo o una nueva integración de QA.

## Decisiones funcionales cerradas

1. La decisión de QA pertenece a la ruta gobernada común y no a cada interfaz.
2. En modo paralelo, la transición no queda bloqueada por `pending`, `failed` ni `cancelled`; el resultado siempre identifica el estado observado. Solo `passed` satisface evidencia requerida.
3. En modo inline o cuando QA sea rol requerido, `pending`, `failed` y `cancelled` bloquean. La única evidencia satisfactoria es `passed`.
4. El mecanismo para reconocer la obligatoriedad reutiliza la configuración y metadata existentes; no introduce una fuente de policy nueva ni admite fallback silencioso ante una configuración requerida ilegible.
5. La consolidación de spec/contexto se hace al final del lote R-10, después de verificar el comportamiento real.

## Dudas abiertas

Ninguna de producto. La implementación elegirá la ubicación de módulo de menor nivel que no cree una dependencia circular; debe justificarla en la evidencia técnica.

## Estrategia de validación

Ejecutar la suite del nuevo contrato y las suites de `governed-transition-runner`, increment, bug, integrate, launcher y MCP que cubran archive/preview. Ejecutar build focalizado del package propietario si la interfaz pública cambia. La verificación independiente repetirá las pruebas afectadas y comprobará ausencia de las dos implementaciones anteriores.

## Trazabilidad y fuentes

Plan `PLAN-REVISION-INTEGRAL-AXIOM.md`, ACC-043; `apps/cli/src/commands/qa-archive-gate.ts`; `apps/cli/src/commands/axiom-qa-e2e.ts`; `packages/workflow/src/governed-transition-runner.ts`.

## Estado de validación humana (histórico, previo a la implementación)

Comportamiento decidido por el lote. Pendiente de implementación, validación independiente, receipts y consolidación final.
## Resultado de cierre

`QaArchiveDecision` es el único contrato QA del runner: distingue los cuatro estados, permite parallel con aviso y bloquea inline/QA requerida salvo `passed`; preview no persiste y CLI, bug, integrate, launcher y MCP comparten la decisión. Validación actual: focal R-10 `21/21` y `238/238`, build, suite global `330/330` y `3289/3289`, doctor PASS `45/60` (0 fallos) y readiness PASS. Receipt Core de verify: SHA-256 `cf2f7d31a9a45feb072d1542bc7000874515ff3843571da3b4766871dc38ef8e`.
## Cierre Core

Después de registrar `qa-e2e: archived` mediante `start → verify --run-validation → pass`, el Core archivó este incremento en `specs/increments/_archive/` mediante `specify → plan → plan-approve → verify → archive`; el receipt `2026-08-18T14-09-51.347Z-increment-archive-success.json` lo confirma. La cifra `3289/3289` anterior es una fotografía previa: la validación global final es `330/330` archivos y `3294/3294` tests, con build, doctor y readiness PASS.