# INC-20260730-phase-receipts: Receipt Phase

## Metadata

- **ID**: INC-20260730-phase-receipts
- **Status**: closed
- **Goal**: Asegurar el gobierno verificable mediante la emisión de "receipts" (recibos) inmutables en JSON al finalizar cada fase del ciclo de vida de `increment`/`bug`.
- **Scope**: `@axiom/workflow` (`packages/workflow/src/receipts.ts`, ya existente), `apps/cli/src/commands/axiom-increment.ts`, `apps/cli/src/commands/axiom-bug.ts`, `apps/cli/src/commands/phase.ts` (sin cambios, verificado que sigue funcionando).
- **Non-goals**: No modificar la manera en la que se ejecutan las fases/transiciones (el receipt es un side-effect posterior, nunca altera el resultado). No reescribir historial pasado — el receipt hand-made preexistente en `receipts/2026-07-30T11-36-13.199Z-design-success.json` se deja intacto.

## Acceptance Criteria

1. Las funciones de ciclo de vida en `packages/workflow` (o el equivalente en el CLI de phases) emiten un receipt JSON validado al final. → Cumplido: `writePhaseReceipt` (ya existía) ahora se invoca automáticamente al final de cada sub-command de `axiom-increment`/`axiom-bug`, además de seguir disponible manualmente vía `axiom phase receipt`.
2. Si una fase falla, emite un receipt marcando el error. → Cumplido: toda transición con `exitCode === 1` (transición inválida, re-`create` rechazado, `verify` bloqueado por functional-verify) emite un receipt `status: 'failure'`.
3. Todos los tests relevantes pasan. → Cumplido, ver `## Validación y review`.

## Implementation Plan

### Fase 1: Auditoría del estado existente

1. Confirmar que `writePhaseReceipt` (`packages/workflow/src/receipts.ts`) ya existe, funciona, y está exportado por el barrel de `@axiom/workflow` — no reinventar.
2. Confirmar que el único punto de entrada previo era el comando manual `axiom phase receipt` (`apps/cli/src/commands/phase.ts`) — sin emisión automática en ningún lugar.
3. Ubicar el(los) choke point(s) reales del ciclo de vida: `runIncrementSubcommand` (`axiom-increment.ts`) y `runBugSubcommand` (`axiom-bug.ts`), ambos wrappers delgados sobre `applyTransition` + `saveWorkflowState` + sync de `metadata.yml`.

### Fase 2: Wiring de emisión automática

4. Renombrar la lógica existente a `runIncrementSubcommandCore`/`runBugSubcommandCore` (privadas, no exportadas) SIN tocar una sola línea de su cuerpo.
5. Exportar un wrapper público homónimo (`runIncrementSubcommand`/`runBugSubcommand`) que llama al core, calcula y escribe el receipt best-effort, y retorna el resultado del core sin modificarlo.
6. Implementar `emitIncrementPhaseReceipt`/`emitBugPhaseReceipt`: política de fase real, id resoluble, dry-run gate, archive-move gate, try/catch best-effort (ver `## Decisiones de implementación`).

### Fase 3: Cobertura de tests

7. `packages/workflow/tests/receipts.test.ts` (nuevo): unit tests de `writePhaseReceipt` — creación de carpeta, naming, shape, hash, sanitización, ausencia de colisión entre múltiples receipts.
8. `apps/cli/tests/axiom-increment-receipts.test.ts` y `apps/cli/tests/axiom-bug-receipts.test.ts` (nuevos): emisión automática en éxito, en fallo (transición inválida, re-`create`), en dry-run (`verify --preview`, sin receipt), y en sub-command desconocido (sin receipt).

## Decisiones de implementación

- **Choke point elegido**: `runIncrementSubcommand` (`axiom-increment.ts:332` antes de este incremento) y su análogo `runBugSubcommand` (`axiom-bug.ts:268` antes de este incremento) — es el único lugar donde el CLI ejecuta `applyTransition` para los 8 (`increment`) / 4 (`bug`) sub-comandos reales. El comando manual `axiom phase receipt` (`phase.ts`) se deja sin tocar y sigue funcionando de forma independiente.
- **Vocabulario de fase — resuelto deliberadamente**: el Scope original del increment nombraba fases SDD (`design, tasks, apply, verify, knowledge, freeze, close`), pero el CLI real no tiene esos nombres — tiene 8 transiciones de `increment` (`create, refine, specify, change, plan, plan-approve, verify, archive`) y 4 de `bug` (`create, fix-plan, verify, archive`). Se decidió **emitir receipts con el nombre REAL de la transición ejecutada** (el `command` de `workflows.yaml`, e.g. `increment-verify`, `bug-archive`), no inventar un mapeo hacia el vocabulario SDD genérico: es lo que efectivamente corrió y es verificable 1:1 contra el state machine y `workflows.yaml`. `PhaseName` en `receipts.ts` ya tolera esto (`'design' | 'tasks' | ... | string`), sin necesidad de cambios en el tipo.
- **No-goal respetado ("no modificar cómo se ejecutan las fases")**: se implementó como un WRAPPER, no como código intercalado en el cuerpo de la función original. `runIncrementSubcommandCore`/`runBugSubcommandCore` son copias EXACTAS (sólo renombradas, no exportadas) de la lógica previa; el wrapper público llama al core, y SÓLO DESPUÉS calcula y escribe el receipt a partir del `result` ya calculado. Esto garantiza por construcción que ningún resultado de transición puede verse alterado por la lógica de receipts.
- **Postura de fallo del receipt-writing — best-effort/never-block**: si `writePhaseReceipt` lanza (permisos, FS de solo lectura, `resolveSpecArtifactRelPath`/`resolveArtifactDir` con un caso inesperado), se atrapa con try/catch, se reporta por `stderr` con el prefijo `[axiom increment]`/`[axiom bug]`, y el `result`/`exitCode` que ya calculó el core NUNCA se modifica. Es la misma convención que ya usa este mismo archivo para otros side-effects no críticos (hooks `pre-start`/`post-start` fire-and-forget, `archiveArtifactDir` en `syncIncrementMetadata` que sólo emite un `WARN` por stderr sin bloquear la transición).
- **Dry-run/preview respetado**: `verify --preview`/`--dry-run` (sólo existe en `increment`, no en `bug`) retorna `exitCode: 0` pero NO aplica la transición (`fromState === toState`, mensaje con `PREVIEW`). Emitir un receipt `success` ahí sería falso (afirmaría que una fase corrió cuando no corrió). Se gatea explícitamente: `emitIncrementPhaseReceipt` retorna temprano si `rawArgs.verifyPreview === true`, sin excepción.
- **Resolución del ID/carpeta del increment — descubrimiento importante durante la implementación**: `axiom-increment create`/`axiom-bug create` NO usan el `--id` humano tipeado por el caller como nombre de carpeta del artefacto folder-per-instancia (INC-06) — usan un `vars.metadataId` generado por `generateUniqueArtifactId` (formato `INC-YYYYMMDD-HHMMSS-xxxxxx`), completamente distinto del `--id` humano. Un primer intento de este incremento usó el `--id` humano para resolver la carpeta del receipt y creó una carpeta FANTASMA junto a la carpeta real de `metadata.yml`, rompiendo `apps/cli/tests/spec-scope-convergence.test.ts` (que cuenta carpetas bajo `increments/`). Se corrigió: la resolución prioriza `vars.metadataId` (co-localiza el receipt con `metadata.yml`/`README.md` reales) y sólo cae al `--id`/`vars.id` humano cuando NO hay `metadataId` — que es exactamente el caso de TODOS los increments reales de este propio repositorio (incluido este mismo), que se autoran a mano vía este workflow spec-first y nunca pasan por `axiom-increment create`. Esta caída coincide con el comportamiento ya establecido del comando manual `axiom phase receipt --increment <id>`, que siempre trata el `--increment` como el nombre literal de carpeta.
- **`archive` mueve físicamente la carpeta — gate adicional descubierto por tests**: `archiveArtifactDir` (invocado dentro del core, antes de que corra el wrapper de receipts) mueve la carpeta completa de `<kind>/<id>/` a `<kind>/_archive/<id>/` al llegar a `archived`. Sin un gate, el receipt de la fase `archive` se escribía en la carpeta PRE-archive (recreándola como fantasma), mientras los receipts previos viajaban al archive junto con el resto de la carpeta. Se corrigió: cuando `result.toState === 'archived'` y `resolveArchivedArtifactDir(...)` ya existe en disco, el receipt se escribe ahí; si el move best-effort falló (caso raro, `archiveArtifactDir` sólo emite un WARN sin bloquear), se cae de vuelta a la carpeta pre-archive (chequeo por existencia real en filesystem, no por asunciones de timing).
- **Alcance de la política "sin id/transición conocida → sin receipt"**: sub-commands desconocidos (no mapean a ninguna transición de `workflows.yaml`) nunca emiten receipt — no hay una fase real que nombrar y no se inventa una. Fallas estructurales tempranas (validación de args, config de `workflows.yaml` ausente/corrupta, guard de repo-affinity) SÍ emiten receipt de fallo cuando el `--id` explícito está disponible (son "una fase intentada y bloqueada"), pero se omiten silenciosamente si no hay ningún ID resoluble en absoluto (no hay carpeta atribuible sin inventar una).

## Validación y review

- `npm run build` (desde `Axiom/`): pasa (`tsc -b`), sin errores en ningún paquete/app.
- `npx vitest run packages/workflow apps/cli`: **150 archivos de test, 1511/1512 tests pasan**. La única falla (`apps/cli/tests/launcher-panels.test.ts > POST /launcher/role-branch > confirmed → creates + checks out the branch in a temp repo, NO push`, timeout a 5000ms) es un **flake de contención bajo suite completa, NO atribuible a este incremento**: no toca ningún archivo relacionado (`launcher-panels.ts`, git role-branch), y re-ejecutado en aislamiento (`npx vitest run apps/cli/tests/launcher-panels.test.ts`) pasa limpio (14/14) en 8.7s. Mismo patrón de debilidad de aislamiento de tests ya documentado para `packages/memory/tests/engram-backend.test.ts` (fuera del alcance declarado de este incremento).
- Tests nuevos añadidos por este incremento, ejecutados de forma aislada tras la corrección del bug de folder fantasma: `packages/workflow/tests/receipts.test.ts` (8/8), `apps/cli/tests/axiom-increment-receipts.test.ts` (6/6), `apps/cli/tests/axiom-bug-receipts.test.ts` (5/5) — los 19 tests nuevos pasan.
- Se re-verificó explícitamente que ningún test PRE-EXISTENTE quedó roto por el wiring: `apps/cli/tests/axiom-increment.test.ts` (10/10), `apps/cli/tests/axiom-bug.test.ts` (6/6), `apps/cli/tests/axiom-increment-metadata.test.ts` (4/4), `apps/cli/tests/axiom-bug-metadata.test.ts` (3/3), y en particular `apps/cli/tests/spec-scope-convergence.test.ts` (3/3) — el test que expuso y confirmó la corrección del bug de carpeta fantasma (`metadataId` vs `--id` humano).
- Las dos fallas pre-existentes fuera de alcance declaradas antes de empezar (`packages/install-profiles/tests/composer.test.ts` × 5, `packages/memory/tests/engram-backend.test.ts` × 1 por contención) NO están en el scope de `packages/workflow apps/cli` y por lo tanto no aparecen en esta corrida — no se tocó ningún archivo de esos dos paquetes.
- El comando manual `axiom phase receipt` (`apps/cli/src/commands/phase.ts`) queda completamente sin modificar (verificado con `git status`/`git diff`: cero cambios en ese archivo ni en `packages/workflow/src/receipts.ts`) — sigue funcionando exactamente igual que antes.
- Review de consistencia manual: se confirmó que `packages/workflow/src/index.ts` ya exportaba `writePhaseReceipt`/`PhaseName`/`PhaseStatus`/`PhaseReceipt` desde antes de este incremento (sin cambios necesarios en el barrel).

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

Lo aportado por ESTE incremento quedó en: RF-AXM-059 (`01`), NFR-AXM-023 §2 y su hueco de excepciones (`02`), artefacto `receipts/*.json` (`03`), receipt automático por transición (`04`), nota de vocabulario de `phase` (`05`), gate de recibos (`07`), términos `PhaseReceipt`/emisión best-effort (`08`), resumen de tanda (`00`).

### Contexto técnico (`Axiom.Spec/context/**`)

**Sí aplicó.** Documentos actualizados por este incremento: `architecture/03-ciclo-de-vida-cli-y-orquestacion.md` (wrapper vs core, y dónde vive el `safeParse`), `references/01-inventario-de-packages.md` (fila `@axiom/workflow`), `references/02-historial-de-incrementos.md`, `references/03-riesgos-y-brechas-conocidas.md` (el receipt no cubre excepciones que escapen del core).

El pase de contexto no fue solo aditivo: se corrigió el punto 5 de `context/TECHNICAL_CONTEXT.md`, que declaraba `TC-011` como bloqueo vigente y citaba "3425/3427 tests" — `npm run doctor` da hoy `PASS` (46/61 OK, 0 fallos) y la suite vigente es 3489 tests / 3483 verdes / 6 rojos preexistentes.

## Estado de cierre

El incremento se marca `closed`: el goal es claro, los acceptance criteria están definidos y se verificaron uno a uno, los cambios se implementaron en `apps/cli/src/commands/axiom-increment.ts` y `apps/cli/src/commands/axiom-bug.ts` (wiring automático) sin tocar `packages/workflow/src/receipts.ts` ni `apps/cli/src/commands/phase.ts` (ambos ya cumplían su parte), la validación disponible (`npm run build` + `npx vitest run packages/workflow apps/cli`) se ejecutó y quedó documentada con números reales, la review contra los acceptance criteria se completó (incluyendo el hallazgo y corrección de dos bugs reales de resolución de carpeta descubiertos por los propios tests nuevos), y la integración a `general-spec.md` queda explícitamente diferida al pase de batch — sin bloquear el cierre de este incremento individual, siguiendo el mismo patrón que otros incrementos hermanos de este lote. No se archiva la carpeta (pendiente del pase de batch).
