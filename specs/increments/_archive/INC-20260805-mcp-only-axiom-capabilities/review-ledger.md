# Review ledger: INC-20260805-mcp-only-axiom-capabilities

Date: 2026-08-05
Flow: increment
Route: sdd

## Initial independent review

| id | lens | location | severity | status | evidence |
| --- | --- | --- | --- | --- | --- |
| REVIEW-001 | review | `Axiom/packages/capability-model/src/schemas.ts` | WARNING | open | El schema permitía una definición `domain: axiom` dentro del mapa provider-routed, aunque el contrato pretendía separar ambos mapas. |
| REVIEW-002 | review | `Axiom/packages/config-validation/src/schemas.ts` | WARNING | open | `mcpOnlyCapabilities` no estaba declarado y Zod podía eliminarlo al normalizar el YAML. |
| REVIEW-003 | review | `Axiom/packages/capability-model/src/loader.ts` | CRITICAL | open | Un id MCP-only desconocido podía descartarse silenciosamente al construir el modelo. |
| REVIEW-004 | review | `Axiom/packages/capability-model/src/types.ts`, `Axiom/packages/providers/tests/fixtures.ts`, `Axiom/packages/tool-routing/src/schemas.ts` | WARNING | open | El nuevo mapa obligatorio no estaba reflejado en todos los consumidores tipados ni en el schema de contexto. |
| REVIEW-005 | review | `README.md` | WARNING | open | El README tenía palabras sin tilde en la documentación de resultado. |

## Resolution plan

Se reparan los cinco hallazgos en la misma unidad antes de cerrar el
incremento. No se acepta pérdida silenciosa de ids ni normalización que
elimine el mapa MCP-only. La revisión final se ejecutará después del apply y
de la validación focalizada.

## Final review

Date: 2026-08-05
Pass: N=1
Scope: README y ledger actuales del incremento, más el diff actual de los
archivos modificados dentro del scope.

| id | lens | location | severity | status | evidence |
| --- | --- | --- | --- | --- | --- |
| REVIEW-001 | review | `Axiom/packages/capability-model/src/schemas.ts:15,81` | WARNING | verified | `ProviderRoutedCapabilityDomainEnum` solo admite `sdd`, `spec`, `code` y `memory`; la prueba focalizada rechaza `domain: axiom` dentro de `capabilities`. |
| REVIEW-002 | review | `Axiom/packages/config-validation/src/schemas.ts:186,218` | WARNING | verified | `mcpOnlyCapabilities` tiene schema propio, conserva la forma al validar YAML y rechaza ids desconocidos; la suite de `config-validation` pasó dentro de la suite focalizada. |
| REVIEW-003 | review | `Axiom/packages/capability-model/src/loader.ts:115,135,298` | CRITICAL | verified | `addCapability` registra un issue de carga cuando `makeCapabilityDefinition` no puede construir una definición y el loader devuelve `load-error`; la regresión del loader pasó. |
| REVIEW-004 | review | `Axiom/packages/capability-model/src/types.ts:148`, `Axiom/packages/providers/tests/fixtures.ts:116`, `Axiom/packages/tool-routing/src/schemas.ts:89` | WARNING | verified | `CapabilityModel`, los fixtures tipados y `RouteToolContextSchema` incluyen `mcpOnlyCapabilities`; build y suites focalizadas pasan. |
| REVIEW-005 | review | `Axiom.Spec/specs/increments/INC-20260805-mcp-only-axiom-capabilities/README.md:117,123,128` | WARNING | verified | Se corrigieron las tildes de `genérico`, `está` e `integración`, y se eliminó la frase duplicada de la sección Result. |

## Final validation observed

- `npm run build`: pasó.
- Suite focalizada de `capability-model`, `config-validation`,
  `install-profiles`, `mcp-tools` y `mcp-server`: 11 archivos y 145 pruebas
  pasaron después de la reparación.
- Suite focalizada de registro/servidor/broker MCP: 3 archivos y 44 pruebas
  pasaron.
- `checkCandidateFreeze(INC-20260805-mcp-only-axiom-capabilities)` con
  `AXIOM_TEST_FORCE_JSON=1`: `{"ok":true}`.
- `apps/cli/tests/freeze.test.ts`: 7 pruebas pasaron.
- La prueba directa del loader pasó 2 archivos y 25 pruebas antes de esta
  validación final; el build completo también pasó.
- Receipt verify final: `receipts/2026-08-05T15-53-49.873Z-verify-success.json`,
  `status: success`, hash `aa11722b5ccc4b026b3fe0a0f35cac0e83432c955f39a94f1e4fb6af4ee5eddc`.

## Compliance and risks

La separación de schemas, la preservación de `mcpOnlyCapabilities`, la
propagación del mapa a los consumidores tipados y el error explícito del
loader para definiciones inválidas están verificadas. La documentación
añadida está corregida.

## Recommendation

`closed`: REVIEW-001 a REVIEW-005 están verificados en el diff actual y la
integración estable en `specs/00..08` y `context/**` está completada.

## Suggested commit message

`review: recheck MCP-only axiom capabilities increment`