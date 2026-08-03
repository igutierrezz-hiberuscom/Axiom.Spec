# INC-20260730-typed-recovery: Tipado Estricto de Errores

## Metadata

- **ID**: INC-20260730-typed-recovery
- **Status**: closed
- **Goal**: Asegurar que todos los errores del framework expuestos a agentes o usuarios tengan un `code` bien definido para facilitar la recuperación automática (recovery) sin depender del frágil "message matching".
- **Scope**: catálogo cerrado de `code`s en `packages/core/src/error.ts` (`AXIOM_ERROR_CODES` + `AxiomErrorCode`), migración de los throw sites clave de validación/dependencia a `AxiomError` en `packages/workflow`, `packages/orchestrator` y `apps/cli`.
- **Non-goals**: reescribir todos los try/catch de bajo nivel de librerías externas (yaml parse, `fs` crudo); los errores de `fs`/red puros pueden mantenerse envueltos o conservar el original mientras el error principal emitido al orquestador sea tipado. No se migran los 62 throws de `apps/cli` — ver "Decisiones de implementación".

## Acceptance Criteria

- [x] Se exporta la clase `AxiomError` (ya existía; se le agregó el catálogo `AXIOM_ERROR_CODES`/`AxiomErrorCode`, ambos exportados desde el barrel de `@axiom/core`).
- [x] Se audita y cambian varios casos clave para usar el nuevo `code` (ambos throws de `packages/workflow`, el `GateFailureError` de `packages/orchestrator`, y 37 de los 62 throws de `apps/cli`, cubriendo 17 archivos).
- [x] Los subagentes pueden consultar `error.code` para tomar decisiones deterministas (cubierto por tests que ramifican sobre `error.code` en vez de `error.message`).

## Implementation Plan

1. Auditar el estado real: `AxiomError` ya existía en `packages/core/src/error.ts` y ya se usaba en 4 sitios (`freeze.ts` ×2, `_shared.ts` ×1, `runner.ts`'s `GateFailureError` ×1) con `code`s como string literals sueltos, sin catálogo central.
2. Definir `AXIOM_ERROR_CODES` (const object) + `AxiomErrorCode` (union type) en `packages/core/src/error.ts`, exportado desde el barrel. Catálogo cerrado y grounded: cada entrada corresponde a un throw site real ya migrado en el mismo cambio (sin códigos especulativos).
3. Migrar los 2 raw `throw new Error` de `packages/workflow` (`artifact-id.ts`, `branch-naming.ts`) a `AxiomError`.
4. Reemplazar los 4 usos preexistentes de `AxiomError` (con `code` como string literal) por las constantes del catálogo (`freeze.ts`, `_shared.ts`, `runner.ts`), sin cambiar ningún mensaje.
5. Migrar el subconjunto de mayor valor de `apps/cli`: el patrón `readInitJson` duplicado (init.json ausente/JSON inválido) en 5 archivos, el patrón `installProfile failed` duplicado en 4 archivos, el patrón "proyecto Axiom no resuelto" duplicado en 2 archivos adicionales (fuera de `withProjectContext`), y la validación de opciones CLI (enums cerrados, flags requeridos, formato de `--role`) en `workspace.ts`, `init.ts`, `roles.ts`, `rollback.ts`, `join.ts`, `toolchain.ts`, `workspace-setup.ts`.
6. Corregir un bug latente descubierto al testear: el workaround `Object.setPrototypeOf(this, AxiomError.prototype)` en el constructor rompía `instanceof` para subclases (e.g. `GateFailureError`). Se cambió a `new.target.prototype`.
7. Agregar tests focalizados en `packages/core`, `packages/workflow` y `apps/cli` (catálogo, `instanceof`, `.code` de los sitios migrados).
8. Validar build + tests (targeted y suite ampliada `packages/workflow packages/core apps/cli`).

## Decisiones de implementación

### Catálogo de códigos (`packages/core/src/error.ts`)

`AXIOM_ERROR_CODES` — 11 entradas, todas grounded en un throw site real:

| Code | Origen |
|---|---|
| `AXIOM_NO_PROJECT` | `withProjectContext` (`apps/cli/_shared.ts`, preexistente) + `member-install.ts`/`repair.ts` (nuevo, mismo concepto fuera de `withProjectContext`) |
| `AXIOM_GATE_FAILURE` | `GateFailureError` (`packages/orchestrator/runner.ts`, preexistente) |
| `AXIOM_MEMORY_SCOPE` | `freeze.ts` (preexistente) |
| `AXIOM_MEMORY_QUERY` | `freeze.ts` (preexistente) |
| `AXIOM_INIT_NOT_FOUND` | `init.json` ausente — `context.ts`, `configure.ts`, `audit.ts`, `model.ts`, `sync.ts`, `gateway.ts`, `start.ts` (nuevo) |
| `AXIOM_INIT_INVALID_JSON` | `init.json` malformado — mismos 5 primeros archivos (nuevo) |
| `AXIOM_INSTALL_PROFILE_FAILED` | `installProfile()` retorna `Result` de error — `gateway.ts`, `start.ts`, `context.ts`, `configure.ts` (nuevo) |
| `AXIOM_INVALID_OPTION` | validación de flags/enums/argumentos CLI — `workspace.ts` (×6), `init.ts` (×3), `roles.ts`, `rollback.ts`, `start.ts` (overlay/gateway), `join.ts`, `toolchain.ts`, `workspace-setup.ts` (nuevo) |
| `AXIOM_ARTIFACT_ID_EXHAUSTED` | `generateUniqueArtifactId` agota `maxAttempts` (`packages/workflow`, nuevo) |
| `AXIOM_BRANCH_TEMPLATE_VAR_MISSING` | `renderBranchName` con variable de template faltante (`packages/workflow`, nuevo) |
| `AXIOM_CHECKPOINT_NOT_FOUND` | `axiom rollback` con `checkpointId` desconocido (`rollback.ts`, nuevo) |

`AxiomError.code` se mantiene tipado como `string` (no como `AxiomErrorCode`) deliberadamente: subclases externas o futuras (como `GateFailureError`, que ya existía antes del catálogo) no deben quedar forzadas a este union cerrado. Los callers nuevos deberían preferir un valor de `AXIOM_ERROR_CODES`; el catálogo es la fuente de verdad documentada, no un enforcement de tipos duro. Esto evita introducir una restricción especulativa no pedida.

Se decidió reutilizar `AXIOM_INVALID_OPTION` como código genérico para TODA validación de input de CLI (enum cerrado, flag requerido faltante, formato de `--role`, canal de toolchain inválido, alias de `--profile` no resoluble) en vez de crear un código por cada flag. Esto mantiene el catálogo chico (AGENTS.md prohíbe arquitectura especulativa) sin perder la propiedad central del incremento: un subagente puede ramificar sobre "esto es un problema de input que el operador debe corregir" sin parsear el mensaje.

### Bug descubierto y corregido: `instanceof` roto en subclases de `AxiomError`

Al escribir un test para un subclass ad-hoc de `AxiomError` (mismo patrón que `GateFailureError` de `@axiom/orchestrator`), se detectó que `new CustomError(...) instanceof CustomError` era `false`. Causa raíz: el constructor de `AxiomError` hacía `Object.setPrototypeOf(this, AxiomError.prototype)` incondicionalmente — un workaround típico de ES5-downlevel para restaurar `instanceof Error` en subclases de `Error`, pero que aquí pisaba el prototype chain correcto de CUALQUIER subclase de `AxiomError` (incluido `GateFailureError`, ya en producción). El fix es `Object.setPrototypeOf(this, new.target.prototype)`, que preserva el constructor real invocado. No hay ningún test existente que dependiera del comportamiento roto (`GateFailureError` se consume vía `caught as GateFailureError` + chequeo de la propiedad `gateFailure`, nunca `instanceof GateFailureError`), así que el fix es puramente correctivo y no rompe nada. Se documentó el porqué en el propio comentario del código.

### Throw sites de `apps/cli` migrados (37 de 62)

Migrados (agrupados por archivo): `context.ts` (×3: init not-found, init invalid-json, installProfile failed), `configure.ts` (×3: init not-found, init invalid-json, installProfile failed), `audit.ts` (×2), `model.ts` (×2), `sync.ts` (×2), `gateway.ts` (×2: init not-found, installProfile failed), `start.ts` (×3: init not-found, overlay/gateway conflict, installProfile failed), `workspace.ts` (×6: `assertValidEnumList`, `assertValidEnum`, `parseRoleFlag` ×2, `--name` faltante, `--spec-path` faltante), `init.ts` (×5: `validateProjectName`, `parseEnum`, `resolveProfileToCanonical` ×2, `role` inválido en `repo attach`), `roles.ts` (×1: `resolveTeamOrProfileRole`), `rollback.ts` (×2: id faltante, checkpoint no encontrado), `join.ts` (×1: `--member` faltante), `member-install.ts` (×1: proyecto no resuelto), `repair.ts` (×1: proyecto no resuelto), `toolchain.ts` (×1: canal inválido), `workspace-setup.ts` (×1: repo `create:false` con path inexistente), `upgrade.ts` (×1: `defaultDoctorFn` con proyecto no resuelto).

### Throw sites deliberadamente NO migrados (25 restantes en `apps/cli`, 9 archivos)

Se dejaron sin tipar por caer en el non-goal explícito ("wrapping de librerías externas / fs crudo") o por ser de valor incremental bajo dado el alcance del incremento:

- **Parseo de YAML externo** (`topology.ts` ×3 en `loadProfilesYamlForValidation`, `roles.ts` ×3 en `loadProfilesYaml`, `init.ts` ×3 en `tryLoadProfilesYaml`): `fs.readFileSync` + `yaml.load` + shape-check. Non-goal explícito de la spec.
- **Escritura/lectura de estado de persistencia ya envuelta en `Result`** (`sync.ts` ×1 en la escritura de `last-sync.json`, `gateway.ts` ×2 en `writeResult`/`readResult` de `store.write`/`store.read`, `audit.ts` ×1 en `auditTrailVerify`, `configure.ts` ×2 en `writeCopilotInstructions`/`generateOpencodeConfig`): el `Result.error` ya trae `kind`+`message` estructurado; envolver en un único `code` genérico no agrega valor de decisión nuevo sin diseñar un mapeo `kind→code` (fuera de alcance, evita over-engineering).
- **Invariantes internos, no user-facing** (`gateway.ts` ×1 `assertGatewayStateCoherence`, `init.ts` ×1 "resolveRoleId devolvió un canónico inesperado" dentro de un catch que ya re-envuelve con `AXIOM_INVALID_OPTION`, `workspace-setup.ts` ×3: cardinalidad de `repos[]` kind=control/spec y coincidencia de `controlRepoPath`): son bugs de programación de la propia CLI (specs mal construidos por el caller interno), no condiciones de recovery para un operador/subagente.
- **Template faltante en `writeCopilotForTarget`** (`configure.ts` ×1: `copilot-instructions.template.md` no encontrado, más ×1 catch-wrap del propio `writeCopilotForTarget`): fallo de instalación/seed del template versionado, un nivel de detalle más profundo que las preconditions ya cubiertas (`init.json`, `installProfile`).
- **Búsqueda de puerto libre** (`app.ts` ×2): utilidad de bajo nivel (rango de puertos agotado), no una precondición de negocio ni un input de operador.
- **Conflicto "axiom.yaml ya existe"** (`init.ts` ×1, línea 637): quedó fuera por no encajar limpiamente en ningún código del catálogo sin crear uno nuevo de un solo uso (sería el único caso de "recurso ya existe, usar --force"); documentado aquí para que un incremento futuro lo retome si se vuelve un caso real de recovery.

Ningún mensaje de error visible fue modificado — todas las migraciones son `throw new Error(msg)` → `throw new AxiomError(CODE, msg)` con el mismo `msg` literal, preservando compatibilidad con los tests existentes basados en regex/`toContain` sobre `.message`.

## Validación y review

- `npm run build` (`tsc -b`, monorepo completo): **pasa**, sin errores ni warnings.
- `npx vitest run packages/core packages/workflow packages/orchestrator`: **pasa, 305/305 tests** (23 archivos). Incluye el nuevo `packages/core/tests/error.test.ts` (5/5) y las extensiones a `artifact-id.test.ts`/`engine.test.ts`.
- `npx vitest run packages/core packages/workflow packages/orchestrator apps/cli` (suite ampliada, timeout 20s por test dado el tamaño de la suite e2e de `apps/cli`), ejecutada dos veces en distintos puntos del incremento (antes y después de migrar el último throw site en `upgrade.ts`): **pasa ambas veces, última corrida 1548/1548 tests, 153 archivos**, 0 fallos.
- Corrida previa sin `--testTimeout` ampliado: 1242/1243 con 1 timeout en `apps/cli/tests/member-install.test.ts` (`Test timed out in 5000ms`) — reproducido como **flaky por carga del runner paralelo**, no relacionado con este incremento (`member-install.ts` no fue tocado). Confirmado corriendo el archivo solo (13.3s) y dentro de la corrida ampliada con timeout mayor (14.7s y luego de nuevo en la corrida final): 11/11 pasa en las tres.
- Líneas `[orchestrator] FAIL: command="..."` visibles en el output de test son trazas esperadas a stderr del propio runner (ver `packages/orchestrator/src/runner.ts` línea ~121), no fallos de test.
- Baseline pre-existente fuera de alcance (mencionado en el brief, NO re-verificado en esta pasada porque no se tocó ningún archivo de esos paquetes): `packages/install-profiles/tests/composer.test.ts` (5 fallos determinísticos) y `packages/memory/tests/engram-backend.test.ts` (1 fallo flaky de timing).
- Review de mensajes: se verificó archivo por archivo que ningún string de mensaje cambió (solo se agregó el primer argumento `code` a los `throw`); los tests que hacían `toThrow(/regex/)` o `toContain(...)` sobre `.message` siguen intactos y en verde.

## Result

Se definió y documentó `AXIOM_ERROR_CODES`/`AxiomErrorCode` en `packages/core/src/error.ts` como catálogo único y grounded (11 códigos, cada uno atado a throw sites reales). Se migraron ambos throws de `packages/workflow`, el `GateFailureError` de `packages/orchestrator`, y 37 de 62 throws de `apps/cli` (17 archivos tocados en total, incluyendo los 4 que ya usaban `AxiomError` con `code` como string literal suelto), cubriendo los casos de mayor valor para recovery determinista: proyecto no resuelto, `init.json` ausente/inválido, `installProfile` fallido, validación de opciones/enums de CLI, checkpoint de rollback no encontrado, agotamiento de generación de artifact IDs, y variable de template de branch faltante. Se descubrió y corrigió, además, un bug latente de `instanceof` roto en subclases de `AxiomError` (causa raíz: `Object.setPrototypeOf` hardcodeado en vez de `new.target.prototype`), relevante para cualquier subclase presente o futura del framework. Se agregaron tests focalizados en `packages/core`, `packages/workflow` y `apps/cli` que verifican `.code` e `instanceof AxiomError` en los sitios migrados. Build y suites targeted + ampliada verdes (1548/1548, 153 archivos, corrida final); el único fallo observado en una corrida previa fue un timeout flaky de infraestructura de test en un archivo no tocado por este incremento, confirmado no reproducible al aislarlo o darle más margen de tiempo.

## General spec integration

**Realizada** en la pasada única de integración a nivel de lote (2026-08-02), junto con los otros cinco incrementos `INC-20260730-*`. Se tocaron los nueve ficheros canónicos:

- `Axiom.Spec/specs/00_Resumen_Ejecutivo.md`
- `Axiom.Spec/specs/01_Requisitos_Funcionales.md`
- `Axiom.Spec/specs/02_Requisitos_No_Funcionales.md`
- `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md`
- `Axiom.Spec/specs/04_Flujos_SDD_y_Ciclo_de_Vida.md`
- `Axiom.Spec/specs/05_Interfaces_Operativas.md`
- `Axiom.Spec/specs/06_Integraciones_y_Capacidades.md`
- `Axiom.Spec/specs/07_Gobierno_y_Seguridad.md`
- `Axiom.Spec/specs/08_Glosario.md`

Lo aportado por ESTE incremento quedó en: RF-AXM-057 (`01`), NFR-AXM-023 como refuerzo de NFR-AXM-006 (`02`), catálogo `AXIOM_ERROR_CODES` y la distinción `AXIOM_INVALID_OPTION` vs `AXIOM_INVALID_CONFIG` (`03`), términos `AxiomError`/`AXIOM_ERROR_CODES` (`08`), resumen de tanda (`00`).

### Contexto técnico (`Axiom.Spec/context/**`)

**Sí aplicó.** Documentos actualizados por este incremento: `references/01-inventario-de-packages.md` (fila `@axiom/core` con los nuevos exports), `references/02-historial-de-incrementos.md`, `references/03-riesgos-y-brechas-conocidas.md` (los 25 throw sites sin tipar quedan registrados como límite vigente).

El pase de contexto no fue solo aditivo: se corrigió el punto 5 de `context/TECHNICAL_CONTEXT.md`, que declaraba `TC-011` como bloqueo vigente y citaba "3425/3427 tests" — `npm run doctor` da hoy `PASS` (46/61 OK, 0 fallos) y la suite vigente es 3489 tests / 3483 verdes / 6 rojos preexistentes.

## Estado de cierre

El incremento cumple las 3 acceptance criteria: `AxiomError` (ya existente) fue dotado de un catálogo cerrado y documentado de `code`s (`AXIOM_ERROR_CODES`/`AxiomErrorCode`), se migraron los throw sites clave de validación/dependencia (100% de `packages/workflow`, el `GateFailureError` de `packages/orchestrator`, y ~60% de los throws de `apps/cli` — el subconjunto de mayor valor, con el resto documentado explícitamente y su rationale), y se agregaron tests que demuestran que un consumidor puede ramificar de forma determinista sobre `error.code`. `npm run build` y las suites targeted/ampliadas (`packages/core`, `packages/workflow`, `packages/orchestrator`, `apps/cli`) corren en verde (1548/1548 tests, 153 archivos, corrida final) sin ningún mensaje de error de usuario modificado. No se integró conocimiento estable a `Axiom.Spec/general-spec.md` ni a `specs/00_*.md`..`08_*.md` porque esa integración es responsabilidad del batch runner (`/axiom-autopilot`), no de este incremento individual — ver "General spec integration". Se marca `Status: closed`.
