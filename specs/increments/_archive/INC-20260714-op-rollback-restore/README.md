# INC-20260714 — Operator-invokable checkpoint restore (`axiom rollback`)

> **Código**: INC-20260714-op-rollback-restore
> **Estado**: archived
> **Fecha**: 2026-07-14
> **Batch**: cierre de brechas post-KVP25 (gap 1/5, orden de dependencias 1→2→3→5→4)
> **Groundeo**: `INC-20260713-e2e-corner-case-catalog` (CC-VER-05, "Known gaps to build" P1 #1); memoria `project-axiom-kvp25-integration-test`.

## Goal

Exponer un comando de operador que **restaure el `ManagedState` desde un checkpoint
conocido**. Hoy `restoreCheckpoint` (`@axiom/versioning`) existe y se usa SOLO en el
rollback automático ante fallo dentro de `executeUpgrade`; `axiom upgrade --from-checkpoint <id>`
únicamente reutiliza un checkpoint como punto de restore para migrar **hacia adelante**.
Tras un upgrade fallido o arrepentido, un operador no tiene forma de restaurar un checkpoint
pre-upgrade a mano. Esta es la brecha **CC-VER-05** del catálogo.

## Owner decision (el diseño)

- **Comando primario nuevo: `axiom rollback`** (verbo claro de recuperación), NO un flag más
  en `upgrade` — evita dos caminos divergentes y mantiene `--from-checkpoint` con su
  semántica actual (migrar hacia adelante usando un checkpoint como punto de restore).
- Subformas:
  - `axiom rollback <checkpointId>` — restaura el `ManagedState` (y los demás archivos del
    manifest: `init.json`, `install-profile.json`) desde ese checkpoint.
  - `axiom rollback --list` — lista los checkpoints disponibles (id, `createdAt`, `reason`,
    nº de archivos) leyendo `listCheckpoints`.
  - `axiom rollback <id> --dry-run` — muestra qué archivos se restaurarían **sin mutar** nada.
- **NO se gatea por el orchestrator `upgrade-command`** (que exige `init.json`): el rollback
  es una acción de recuperación y gatearlo podría bloquear justamente la recuperación que se
  necesita. Sólo se exige que el proyecto resuelva (`withProjectContext`).
- Reusa la superficie ya pública de `@axiom/versioning` (`restoreCheckpoint`, `listCheckpoints`,
  `readManagedState`, `managedStatePath`) — no se cambia ninguna firma existente.

## Scope

1. **Núcleo testeable** `runRollback(args)` en un nuevo módulo `apps/cli/src/commands/rollback.ts`
   (espejo del patrón de `upgrade.ts`: función pura de la lógica + wrapper commander `registerRollback`).
   - `runRollback({ cwd, id?, list?, dryRun? })` corre dentro de `withProjectContext(cwd)` para
     obtener `rootPath` + `projectName`.
   - `--list` → devuelve los checkpoints (`listCheckpoints`).
   - id ausente y no `--list` → error claro ("indicá un checkpointId o usá --list"), exit 1, sin mutar.
   - `--dry-run <id>` → resuelve el checkpoint y devuelve los archivos del manifest sin mutar
     (id inexistente → error claro, sin mutar).
   - `<id>` real → captura el `ManagedState` previo (para el reporte), llama
     `restoreCheckpoint({ rootPath, projectName, id })`, y devuelve un `RollbackResult`
     (`{ id, restoredFiles, fromVersion, toVersion }`) donde `toVersion` es la versión del
     state ya restaurado.
2. **Registro** de `registerRollback(program)` en el punto donde se registran los demás
   subcomandos del CLI (junto a `registerUpgrade`).
3. **Higiene**: actualizar el mensaje de error obsoleto de `restoreCheckpoint`
   (`packages/versioning/src/checkpoints.ts`) que hoy dice "axiom upgrade --dry-run (siguiente
   lote)" para que apunte a `axiom rollback --list` (la superficie real ya existente).
4. **Tests** — unit/integration de `runRollback`: restaurar un checkpoint conocido revierte el
   `managed-state.json`; id inexistente → error claro y **cero mutación** (comprobar el archivo
   intacto); `--list` devuelve los checkpoints; `--dry-run` no muta. Regresión del mensaje de
   error si se toca `restoreCheckpoint`.
5. **e2e en sandbox aislado** (KVP25 en scratchpad, HOME hermético): `workspace setup` →
   `axiom upgrade` (crea checkpoint) → mutar `managed-state.json` a mano → `axiom rollback <id>`
   → el state queda revertido al snapshot; `axiom rollback <id-inexistente>` → error, sin cambio.

## Non-goals

- No cambiar la semántica de `axiom upgrade --from-checkpoint` (sigue migrando hacia adelante).
- No añadir fan-out cross-repo (eso es el gap 2, incremento separado).
- No tocar `executeUpgrade` ni el contrato de rollback automático ante fallo.
- No introducir un flag `--force`/`--yes` nuevo ni prompts interactivos (restore directo;
  el `--dry-run` cubre la previsualización).

## Acceptance

- `axiom rollback <id>` restaura el `ManagedState` desde ese checkpoint (managed-state.json
  vuelve al snapshot).
- `axiom rollback <id-inexistente>` → error claro, exit 1, **sin mutar** ningún archivo.
- `axiom rollback --list` lista los checkpoints disponibles.
- `axiom rollback <id> --dry-run` muestra los archivos a restaurar sin mutar.
- Sin regresión en `axiom upgrade` (`--from-checkpoint` intacto).
- Gate verde en `Axiom/`: `npm run build` → `npm test` → `npm run typecheck` →
  `node apps/cli/dist/index.js doctor` (módulo el flake conocido de timeout I/O 5000ms bajo carga).

## Result

**Estado: implementado y validado (gate verde). Ver rationale de `Status` en `metadata.yml` — se deja en `specified`→ debe promoverse a `implemented`/`integrated` por quien gestiona el metadata.yml del catálogo; el trabajo de código descripto abajo está cerrado.**

### Archivos cambiados (todos en `Axiom/`, el monorepo producto — NUNCA en `Axiom.SDD`)

- **Nuevo** `apps/cli/src/commands/rollback.ts` — núcleo testeable `runRollback(args)` +
  wrapper commander `registerRollback(program)`, mirror estructural de `upgrade.ts`.
  Exporta `RollbackArgs`, `RollbackRunResult`, `runRollback`, `registerRollback`,
  `formatRollbackErrorLine`.
- **Nuevo** `apps/cli/tests/rollback.test.ts` — 15 tests (6 `describe` blocks).
- `packages/versioning/src/checkpoints.ts` — higiene D-003: el mensaje de error de
  `restoreCheckpoint` para un id desconocido ahora referencia
  `` `axiom rollback --list` `` en vez del texto obsoleto
  `` `axiom upgrade --dry-run` (siguiente lote) ``. (No se tocó la lógica: la
  verificación de existencia del manifest sigue ocurriendo ANTES de cualquier
  escritura — cero mutación preservada.)
- `packages/cli-commands/tsconfig.json` — agregado
  `../../apps/cli/src/commands/rollback.ts` al `include` (junto a `upgrade.ts`).
- `apps/cli/tsconfig.json` — agregado `src/commands/rollback.ts` al `exclude`
  (single-ownership gotcha: `@axiom/cli-commands` es el único compilador de este
  archivo; `apps/cli` NO debe recompilarlo).
- `packages/cli-commands/src/index.ts` — re-export
  `export { registerRollback, runRollback, formatRollbackErrorLine } from '../../../apps/cli/src/commands/rollback';`
  junto al re-export de `registerUpgrade`.
- `apps/cli/src/index.ts` — `import { registerRollback } from '@axiom/cli-commands';`
  + `registerRollback(program);` registrado inmediatamente después de
  `registerUpgrade(program);`.

### Diseño as-built

Coincide con el diseño del owner (D-001/D-002/D-003) sin desviaciones:

- `runRollback({ cwd, id?, list?, dryRun? })` corre dentro de
  `withProjectContext(cwd, ...)` (rootPath + projectName), SIN `runOrchestrated`
  (D-002: rollback no se gatea por `upgrade-command`/`init.json`).
- `RollbackRunResult` es un discriminated union por `mode`:
  `{ mode: 'list', checkpoints }` | `{ mode: 'dry-run', id, files }` |
  `{ mode: 'restored', id, restoredFiles, fromVersion, toVersion }`.
- Orden de resolución dentro de `runRollback`: (1) `list` tiene prioridad y
  devuelve read-only; (2) sin `id` y sin `list` → `Error` claro (mensaje en
  español, sin mutar); (3) `dryRun` con `id` → resuelve el checkpoint vía
  `listCheckpoints` y devuelve su `files` SIN llamar a `restoreCheckpoint`
  (id inexistente → error explícito "no encontrado", cero mutación); (4) `id`
  real → captura `readManagedState` (pre), llama `restoreCheckpoint`, vuelve a
  leer el state (post) y arma el resultado. Un `id` desconocido en este camino
  real deja que `restoreCheckpoint` mismo lance (ya verifica existencia del
  manifest ANTES de escribir — cero mutación garantizada sin duplicar la
  validación).
- `registerRollback(program)`: `program.command('rollback [checkpointId]')`
  con `--list` y `--dry-run`; en error, imprime una sola línea de stderr vía
  `formatRollbackErrorLine(err)` (prefijo `[axiom rollback] Error: ...`, sin
  flags ad-hoc tipo `gateFailure`/`upgradeFailed` porque D-002 saca a rollback
  del orchestrator) y `process.exit(1)`.
- Sin cambios de firma en `@axiom/versioning` (`restoreCheckpoint`,
  `listCheckpoints`, `readManagedState`, `managedStatePath` se consumen tal
  cual). `axiom upgrade --from-checkpoint` intacto (no tocado).

### Tests añadidos (`apps/cli/tests/rollback.test.ts`, 15 tests, 0 regresiones)

1. Restore revierte `managed-state.json` al snapshot; `fromVersion`/`toVersion`
   reportados correctamente.
2. Id inexistente → rechaza (mensaje incluye el id) y el archivo queda
   **byte-idéntico** (cero mutación verificada).
3. `--list` devuelve los checkpoints creados (+ caso vacío sin checkpoints).
4. `--dry-run <id>` devuelve la lista de archivos del manifest sin restaurar
   (verificado con contenido byte-idéntico antes/después) + caso id
   inexistente con `--dry-run` (error, cero mutación).
5. Sin `id` y sin `--list` → error claro.
6. `formatRollbackErrorLine` formatea `Error` y valores no-`Error` con el
   prefijo `[axiom rollback] Error: ...`.

Ningún test existente fue debilitado ni borrado. `apps/cli/tests/upgrade.test.ts`
(formatUpgradeErrorLine) y toda la suite de `packages/versioning/tests/`
(incluido `checkpoints.test.ts`, que sólo verifica el PREFIJO
`Checkpoint "no-existe" no encontrado`, no el texto completo) pasan sin
modificación.

### e2e en sandbox aislado (KVP25, HOME hermético)

Ejecutado en una copia throwaway (`.../scratchpad/sandbox/work-gap1/`, borrada
al terminar) del baseline pristino (`.../scratchpad/sandbox/kvp25/`), que quedó
intacto (`git status` limpio, mismo HEAD). CLI invocada con
`USERPROFILE`/`HOME` apuntando al home hermético
(`.../scratchpad/sandbox/.axiom-home`) — el `~/.axiom` real (`projects.yml`)
quedó sin mtime nuevo, confirmando cero escritura ahí.

Transcript (resumido):

1. `workspace setup --name kvp25-gap1 --adopt-sdd . --adopt-spec ../Kvp.Spec --role backend:.../Kvp.Api --role frontend:.../Kvp.App --role e2e:.../Kvp.QA -y`
   → adopción aplicada (spec migrated: 59, control conformity OK, `axiom.yaml` v2 materializado).
2. `axiom upgrade --no-sync --no-doctor` → `OK. checkpointId: 2026-07-14T09-09-38.005Z-6806`
   (primer upgrade: el checkpoint cubre `managed-state.json` como "ausente al
   snapshot" porque el archivo aún no existía en disco — comportamiento
   documentado de `createCheckpoint`, no un bug; por eso se corrió un
   **segundo** `axiom upgrade` para obtener un checkpoint con contenido real).
3. Segundo `axiom upgrade --no-sync --no-doctor` → `checkpointId: 2026-07-14T09-10-53.216Z-77d7`.
   `axiom rollback --list` confirma ambos checkpoints.
4. Mutación manual de `managed-state.json` (`runtime.version` →
   `"9.9.9-MUTATED-2"`, `lastCheckpointId` → `"tampered-2"`).
5. `axiom rollback <id> --dry-run` sobre el 2º checkpoint → lista los 3
   archivos del manifest, archivo confirmado **sin cambios** antes/después.
6. `axiom rollback <id>` (real, mismo checkpoint) →
   `OK. fromVersion: 9.9.9-MUTATED-2 → toVersion: 0.1.0`; contenido de
   `managed-state.json` verificado byte a byte revertido al snapshot
   (`runtime.version: "0.1.0"`, `lastCheckpointId` del checkpoint correcto).
7. `axiom rollback bogus-id` → `[axiom rollback] Error: Checkpoint "bogus-id"
   no encontrado en ... Verificá el id con \`axiom rollback --list\` ...`,
   exit code 1 confirmado, archivo verificado byte-idéntico (cero mutación).
8. `axiom rollback` (sin id ni `--list`) →
   `[axiom rollback] Error: indicá un checkpointId a restaurar, o usá
   \`axiom rollback --list\``.

Hallazgo documentado (no es un defecto de este incremento): el checkpoint del
**primer** `axiom upgrade` sobre un proyecto recién adoptado no incluye un
snapshot "real" de `managed-state.json` (el archivo aún no existe en disco en
ese instante — comportamiento ya documentado en `createCheckpoint`'s
skip-silencioso de paths ausentes). Restaurar ESE checkpoint específico es
un no-op para `managed-state.json` (el `runRollback`/`restoreCheckpoint`
igual reportan éxito, consistente con el contrato de `restoreCheckpoint`).
Los unit tests de este incremento reflejan esto explícitamente: siembran el
`managed-state.json` con `saveManagedState` ANTES de `createCheckpoint`, tal
como ocurre en producción a partir del segundo upgrade en adelante.

### Gate (`Axiom/`)

- `npm run build` → limpio, 0 errores.
- `npm test` → **278 test files / 2800 tests, todos PASS** (0 fallos, sin
  timeout flake en esta corrida).
- `npm run typecheck` → limpio, 0 errores.
- `node apps/cli/dist/index.js doctor` (sobre el propio repo `Axiom/`) →
  `Resultado: PASS` (`45/57 OK · 0 FALLO · 3 ADVERTENCIA · 9 OMITIDO` —
  advertencias/omitidos pre-existentes, no relacionados con este incremento).

### Revisión contra acceptance criteria

- `axiom rollback <id>` restaura el `ManagedState` — **cumplido** (unit tests
  + e2e).
- `axiom rollback <id-inexistente>` → error claro, exit 1, sin mutar —
  **cumplido** (unit test byte-idéntico + e2e exit code 1 verificado).
- `axiom rollback --list` — **cumplido**.
- `axiom rollback <id> --dry-run` — **cumplido** (sin mutar, verificado
  byte a byte).
- Sin regresión en `axiom upgrade` (`--from-checkpoint` intacto) —
  **cumplido**: no se tocó `upgrade.ts` ni `executeUpgrade`; suite de
  `packages/versioning/tests/upgrade.test.ts` (12 tests) y
  `apps/cli/tests/upgrade.test.ts` (4 tests) pasan sin cambios.
- Gate verde `Axiom/` — **cumplido** (ver arriba).

### Desviaciones del spec

Ninguna. Todos los puntos de "wiring" de single-ownership (tsconfig
include/exclude, re-export en `cli-commands`, import+registro en
`apps/cli/src/index.ts`) y las decisiones D-001/D-002/D-003 se implementaron
tal cual estaban especificadas.

## General spec integration

No se integró contenido nuevo a `Axiom.Spec/general-spec.md`. El único
conocimiento estable generado — la superficie CLI `axiom rollback` y el
patrón single-ownership de `@axiom/cli-commands` — ya está documentado in-situ
(comentarios de cabecera en `rollback.ts`, `tsconfig.json` de
`cli-commands`/`apps/cli`) y en la memoria persistente del agente
(`build_cli_commands_single_ownership`). No hay una decisión de producto o
convención NUEVA (distinta de lo ya conocido) que amerite un párrafo en
`general-spec.md`; este incremento es una extensión aditiva de una superficie
ya diseñada (0019-A, rollback-first contract), no una nueva arquitectura.
