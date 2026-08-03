# INC-20260730-candidate-freeze: Freeze de Candidate

## Metadata

- **ID**: INC-20260730-candidate-freeze
- **Status**: closed
- **Goal**: Implementar un comando `axiom freeze` global que congele el estado de un candidate/incremento (memoria local del proyecto + `README.md` del incremento en el spec repo) generando un snapshot hash, para que el CLI pueda verificar antes de aplicar cambios que el candidate no mutó desde que fue congelado.
- **Scope**: `apps/cli/src/commands/freeze.ts` (`registerFreezeCommand`, `hashCandidateInputs`, `checkCandidateFreeze`); wiring de la validación en `apps/cli/src/commands/axiom-increment.ts` (subcomando `change`); cobertura de tests en `apps/cli/tests/freeze.test.ts`.
- **Non-goals**: No congela estado de repos externos no vinculados. No reemplaza `knowledge freeze` (específico de la fase de harvest). No introduce hashing de lockfiles ni de otros artefactos por-incremento (e.g. `metadata.yml`) — ver limitación documentada en Acceptance Criteria §3 y en Validación.

## Acceptance Criteria

1. Ejecutar `axiom freeze --increment <id>` genera un archivo JSON inmutable (snapshot + hash) en el directorio del incremento. **Cumplido.** El archivo se llama `candidate-freeze.json` (no `.frozen.json` como sugería la redacción original del Scope — ver "Decisiones de implementación" para la justificación de mantener el nombre real ya implementado y con un artefacto real ya generado en este mismo directorio).
2. Se proporciona una API para validar el freeze. **Cumplido.** `checkCandidateFreeze(incrementId, cwd): Promise<{ ok: boolean; reason?: string }>`, ya wireada en `axiom-increment.ts` (subcomando `change`, el punto del ciclo de vida del incremento más cercano a "aplicar cambios").
3. **Honestidad sobre qué congela realmente `hashCandidateInputs`** (evaluación explícita pedida por este incremento, sin sobre-reclamar cobertura):
   - SÍ hashea: las entries de memoria del proyecto activo filtradas por `entry.increment === incrementId` (`memoryHash`) y el contenido de `specs/increments/<id>/README.md` en el spec repo (`specsHash`). `combinedHash = sha256(memoryHash-specsHash)`.
   - NO hashea (limitación documentada, no cerrada en este pase): otros artefactos por-incremento que sí existen como convención en otras partes del CLI (e.g. `metadata.yml` de `artifact-metadata-cli.ts`, cuando existe), ni lockfiles/dependencias externas (e.g. `package-lock.json`, `axiom.config/toolchain.lock` de INC-20260730-toolchain-versioning). El Scope original habla de congelar "memoria local, repo specs, dependencias"; lo que está implementado y probado cubre memoria + el propio `README.md` del incremento, no "dependencias" en sentido amplio.
   - Decisión: no se amplía el algoritmo de hashing en este pase. Cambiar qué se hashea invalidaría (silenciosamente) el hash de cualquier freeze ya generado en el spec repo — incluyendo el artefacto real `candidate-freeze.json` de este mismo incremento — sin que el beneficio (este incremento en particular no tiene `metadata.yml` ni lockfile propio) lo justifique. Se documenta como consideración futura, no como trabajo pendiente bloqueante de este incremento.

## Implementation Plan

Al recibir este incremento, la mayor parte de la implementación ya estaba hecha (comando CLI, hashing, API de validación, wiring en `axiom-increment.ts`). El plan se redujo a cerrar 3 brechas concretas detectadas en la auditoría previa:

1. **Cobertura de tests (brecha principal)**: crear `apps/cli/tests/freeze.test.ts` cubriendo: determinismo de `hashCandidateInputs`, cambio de hash al mutar el `README.md`, `checkCandidateFreeze` con archivo de freeze ausente, con hash mismatch tras mutación, con freeze válido no mutado, y regresión de `JSON.parse` (archivo corrupto no debe tirar excepción).
2. **Fix de `JSON.parse` sin guardar** en `checkCandidateFreeze` (`freeze.ts` línea ~64 original): mover la lectura+parseo del archivo de freeze a su propio `try/catch` que devuelve `{ ok:false, reason }` en vez de dejar que la excepción escape sin capturar.
3. **Revisión honesta de la AC** sobre qué congela `hashCandidateInputs` (memoria + README únicamente) frente a la redacción original del Scope ("memoria local, repo specs, dependencias"), documentando la brecha real en vez de sobre-reclamar cobertura.

No se tocó el resto del comando (`registerFreezeCommand`, el cálculo de hashes, el wiring en `axiom-increment.ts`) porque ya funcionaba correctamente y reescribirlo no aportaba valor frente al riesgo de romper el artefacto `candidate-freeze.json` real que ya existe en este directorio.

## Decisiones de implementación

- **Nombre del archivo (`candidate-freeze.json` vs `.frozen.json`)**: se mantiene `candidate-freeze.json`, el nombre ya implementado en código y ya materializado como artefacto real en este directorio (`candidate-freeze.json`, generado el 2026-07-30). Renombrarlo rompería ese artefacto y cualquier consumidor externo que ya lo busque por ese nombre. La redacción original del Scope ("`.frozen.json` (o similar)") ya contemplaba esta variación con "o similar"; se documenta acá como la resolución definitiva en vez de re-generar el archivo.
- **Fix del `JSON.parse` sin guardar**: se envolvió la lectura+parseo en su propio `try/catch`, separado del `try/catch` existente que ya envolvía la llamada a `hashCandidateInputs`. Se eligió no fusionar ambos `try/catch` en uno solo para mantener mensajes de error distinguibles (archivo corrupto vs. fallo de hashing) y para tocar el mínimo de líneas posible sobre código que ya funcionaba.
- **Ubicación y patrón de los tests nuevos** (`apps/cli/tests/freeze.test.ts`): se siguió el patrón ya establecido en `apps/cli/tests/memory.test.ts` (tmp dirs vía `mkdtempSync`, sin `topology.yaml` → manifest single-repo default donde `specRepo === projectRoot`, limpieza en `afterEach`). Toda memoria persistida en los tests usa `saveMemory` directo (no `runMemoryAdd`) para poder setear el campo `increment`, y siempre con `rationale`/`source` reales de más de 3 caracteres (GATE INC-20260730-engram-evidence) — de lo contrario el guardado sería rechazado fail-closed.
- **`process.chdir()` no usado en los tests**: el pool de vitest de este repo corre los tests en worker threads, donde `process.chdir()` lanza `ERR_WORKER_UNSUPPORTED_OPERATION` (confirmado empíricamente al escribir el test). `hashCandidateInputs`/`checkCandidateFreeze` reciben `cwd` explícito y no lo necesitan; para el único escenario que ejercita el comando Commander real (que sólo conoce `process.cwd()`, sin flag `--path`), se usó `vi.spyOn(process, 'cwd').mockReturnValue(root)` en su lugar — no requiere tocar `freeze.ts`.
- **`AXIOM_TEST_FORCE_JSON`**: se usa tal como ya estaba documentado en el propio código (`checkCandidateFreeze` ya lo leía desde `process.env`) para forzar el backend JSON de memoria en los escenarios que pasan por `checkCandidateFreeze` sin poder pasarle `forceJson` explícitamente (esa función no expone ese parámetro en su firma pública). Se limpia con `delete process.env.AXIOM_TEST_FORCE_JSON` en `afterEach`.
- **No se amplía el hashing de `hashCandidateInputs`** para cubrir lockfiles/`metadata.yml` — ver justificación en Acceptance Criteria §3.

## Validación y review

- `npm run build` (raíz del monorepo, `tsc -b`): **pasa**, sin output (compilación limpia de todos los packages/apps).
- `npx vitest run apps/cli`: **pasa completo — 133 archivos, 1261 tests, 0 fallos** (incluye el nuevo `apps/cli/tests/freeze.test.ts`, 7/7 tests verdes). `apps/cli/tests/launcher-panels.test.ts`, señalado como potencialmente flaky por contención bajo suite completa, corrió limpio dentro de esta corrida de `apps/cli` (14/14).
- Tests nuevos añadidos en `apps/cli/tests/freeze.test.ts` (7 escenarios):
  1. `hashCandidateInputs` es determinista (mismos inputs → mismo `combinedHash`/`memoryHash`/`specsHash` en dos corridas).
  2. El hash cambia (`specsHash` y `combinedHash`) cuando el `README.md` del incremento muta; `memoryHash` no cambia si la memoria no cambió.
  3. `checkCandidateFreeze` devuelve `ok:false` con reason clara cuando falta `candidate-freeze.json`.
  4. `checkCandidateFreeze` devuelve `ok:false` (hash mismatch) cuando el `README.md` mutó después de congelar.
  5. `checkCandidateFreeze` devuelve `ok:true` sobre un freeze válido y no mutado.
  6. Regresión del fix de `JSON.parse`: un `candidate-freeze.json` con JSON inválido/truncado NO lanza excepción — devuelve `{ ok:false, reason }` (verificado explícitamente con `try/catch` alrededor de la llamada en el test, comprobando `thrown === undefined`).
  7. El comando CLI real (`registerFreezeCommand` + Commander `parseAsync`) escribe `candidate-freeze.json` con el shape esperado (`incrementId`, `hash`/`memoryHash`/`specsHash` como hex sha256 de 64 caracteres, `timestamp` ISO válido), y el freeze recién escrito valida `ok:true` de inmediato vía `checkCandidateFreeze`.
- **No se corrió la suite completa del monorepo** (`3471 tests` de referencia mencionados en el brief) en este pase — solo se corrieron los dos comandos mandatorios (`npm run build` y `npx vitest run apps/cli`), que es donde vive todo el código y los tests de este incremento. Las fallas preexistentes conocidas (`packages/install-profiles/tests/composer.test.ts`, `packages/memory/tests/engram-backend.test.ts`) están fuera del árbol `apps/cli` y por lo tanto no aparecen en esta corrida ni fueron re-verificadas acá.
- **El artefacto real `Axiom.Spec/specs/increments/INC-20260730-candidate-freeze/candidate-freeze.json` NO cambió** durante este trabajo (verificado por `LastWriteTime` = 30/07/2026 13:38:18, igual al `timestamp` original `2026-07-30T11:38:18.778Z`). Todos los tests nuevos operan sobre directorios temporales (`mkdtempSync`), nunca sobre este spec repo real.
- **Nota meta, honesta y directamente relevante al propio tema de este incremento**: al reescribir este mismo `README.md` (paso 1, obligatorio del flujo) el `specsHash` congelado en `candidate-freeze.json` quedó desactualizado respecto del contenido real actual del archivo — se verificó explícitamente (`sha256` del `README.md` actual = `59ab5924308a9982279bcf07e55581515de66db0443b1a47dd2431fd4454d877`, distinto del `specsHash` congelado = `a93cd43c4cf2ae324457bc1d796c5124357531ecde416fa54cff08a43ea12444`). Es decir: si alguien corriera `checkCandidateFreeze('INC-20260730-candidate-freeze', ...)` contra este spec repo real ahora mismo, devolvería `ok:false` (hash mismatch) — comportamiento correcto y esperado de la propia feature que este incremento implementa, no un bug. No se regeneró el freeze (instrucción explícita de no tocar el artefacto salvo que un test real lo reescribiera); re-congelar este incremento en particular (`axiom freeze --increment INC-20260730-candidate-freeze`) queda como una acción de seguimiento natural, fuera del alcance de este pase.
- Review honesta de la AC: la AC #1 y #2 (archivo JSON inmutable + API de validación) están completamente cumplidas y ahora cubiertas por tests. La AC implícita del Scope ("congela ... dependencias") NO está cubierta — documentado explícitamente como limitación conocida, no como brecha oculta.

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

Lo aportado por ESTE incremento quedó en: RF-AXM-060 con su limitación explícita (`01`), NFR-AXM-023 §3 (`02`), artefacto `candidate-freeze.json` y la consecuencia de reescribir el README (`03`), gate de freeze antes del apply (`04`), comando `axiom freeze` (`05`), gate de freeze (`07`), términos `Candidate freeze`/`checkCandidateFreeze` (`08`), resumen de tanda (`00`).

### Contexto técnico (`Axiom.Spec/context/**`)

**Sí aplicó.** Documentos actualizados por este incremento: `architecture/03-ciclo-de-vida-cli-y-orquestacion.md`, `references/02-historial-de-incrementos.md`, `references/03-riesgos-y-brechas-conocidas.md` (el freeze no cubre dependencias; reescribir el README lo invalida).

El pase de contexto no fue solo aditivo: se corrigió el punto 5 de `context/TECHNICAL_CONTEXT.md`, que declaraba `TC-011` como bloqueo vigente y citaba "3425/3427 tests" — `npm run doctor` da hoy `PASS` (46/61 OK, 0 fallos) y la suite vigente es 3489 tests / 3483 verdes / 6 rojos preexistentes.

## Estado de cierre

`Status: closed`. Justificación frente a las reglas de cierre de `AGENTS.md`:

- Goal claro: sí (freeze determinista de memoria + spec del incremento, con API de validación).
- Acceptance criteria existen y están evaluados explícitamente (incluyendo la limitación honesta de AC §3).
- Cambios implementados: fix del `JSON.parse` sin guardar en `checkCandidateFreeze` + 7 tests nuevos en `apps/cli/tests/freeze.test.ts`. El resto del comando (ya funcional antes de este pase) se dejó intacto deliberadamente.
- Validación disponible ejecutada: `npm run build` (limpio) y `npx vitest run apps/cli` (133/133 archivos, 1261/1261 tests, 0 fallos).
- Review contra el intent y las AC: hecha, incluyendo la evaluación honesta de qué congela realmente `hashCandidateInputs` (memoria + README del incremento, NO lockfiles ni otros artefactos por-incremento).
- Integración de conocimiento estable en `general-spec.md`: diferida explícitamente al pase de batch (ver sección anterior) — no bloquea el cierre de este incremento individual porque la propia instrucción de orquestación de este lote define ese paso como consolidado al final, no por-incremento.
- Resultado documentado claramente: sí, en esta misma sección y en Validación y review.

No se archiva esta carpeta en este pase (queda para el cierre de batch), pero el incremento en sí se considera resuelto y cerrado.

## Nota post-archivado (2026-08-02)

El pase de integración a nivel de lote reescribió este `README.md`, lo que cambió su `specsHash` y dejó **stale por diseño** el `candidate-freeze.json` de este mismo incremento: `checkCandidateFreeze` reportaría `ok: false`. Es el mecanismo funcionando, no un fallo.

El re-congelado se intentó y **no fue posible**: `axiom freeze --increment INC-20260730-candidate-freeze` falla con "Directorio del incremento no existe" porque tanto `registerFreezeCommand` como `checkCandidateFreeze` resuelven sólo `specs/increments/<id>` y no `specs/increments/_archive/<id>/`. El artefacto congelado queda por tanto como registro histórico no comprobable.

Hallazgo registrado en `Axiom.Spec/context/references/03-riesgos-y-brechas-conocidas.md`. Arreglo natural para un incremento futuro: resolver también la ruta archivada usando `resolveArchivedArtifactDir` de `@axiom/workflow` — que ya existe y que `INC-20260730-phase-receipts` usa exactamente para este mismo problema al emitir receipts de `archive`.
