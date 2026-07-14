# INC-20260714 — Cross-repo `axiom upgrade` fan-out

> **Código**: INC-20260714-cross-repo-upgrade-fanout
> **Estado**: archived
> **Fecha**: 2026-07-14
> **Batch**: cierre de brechas post-KVP25 (gap 2/5, orden 1→2→3→5→4; depende del gap 1 por compartir `upgrade.ts`)
> **Groundeo**: `INC-20260713-e2e-corner-case-catalog` (CC-VER-09, "Known gaps to build" P1 #2); `INC-20260714-op-rollback-restore` (gap 1); memoria `project-axiom-kvp25-integration-test`.

## Goal

Hoy `axiom upgrade` resuelve UN solo proyecto desde el cwd (`withProjectContext(cwd)`) y sólo
migra/checkpointea ESE repo. Un proyecto multi-repo debe poder actualizar **todos sus repos**
en una sola invocación desde el repo de control. Esta es la brecha **CC-VER-09**.

## Owner decision (el diseño)

- **Fan-out orquestado desde el repo de control (`role: sdd`).** Cuando el proyecto resuelto es
  **multi-repo** (`loadTopology(rootPath).mode === 'multi-repo'`) y el comando se ejecuta desde el
  repo de control, `axiom upgrade` itera **todos los repos de la topología** (`sddRepo`,
  `specRepo`, `roleCodeRepositories`), resuelve su path on-disk (`resolveRepoPath` +
  `loadLocalBindings`, `@axiom/topology`), y migra/checkpointea **cada repo**.
- **Per-repo vía `executeUpgrade` directo (`@axiom/versioning`).** El fan-out corre DENTRO del
  gate de entrada del control (`runOrchestrated('upgrade-command', control)` — que exige
  `init.json` SÓLO en el control). Para cada repo llama a `executeUpgrade({ rootPath: <repoPath>,
  projectName, targetVersion, runSync:false, runDoctor:false })`. `executeUpgrade` **no** exige
  `init.json` (inicializa el `ManagedState` con `readOrInitManagedState` y crea el checkpoint
  pre-upgrade), así que NO hace falta fabricar `init.json` en los repos de rol/spec. Cada repo
  obtiene su propio `ManagedState` + checkpoint bajo su propio `<repoPath>/.axiom-state/…`.
  → decisión "ManagedState/checkpoint **por repo**" (la opción permitida por el enunciado).
- **`sync` + `doctor` una sola vez** al final, desde el control (nivel workspace), no N veces —
  respetando `--no-sync` / `--no-doctor`.
- **Sin regresión / opt-out.** Fan-out sólo se dispara en topología **multi-repo** desde el
  control. Un proyecto single-repo (la mayoría de los tests existentes) NUNCA lo dispara → su
  comportamiento es byte-idéntico. Se añade `--repo-only` para forzar el comportamiento
  single-repo de hoy incluso en un proyecto multi-repo (y desde un repo no-control, `upgrade`
  sigue siendo per-repo como hoy).
- **Reporte por-repo** (`repoId`, `role`, `path`, `fromVersion`→`toVersion`, nº migraciones,
  `checkpointId`, `ok|failed`). Si un repo falla, `executeUpgrade` ya restauró SU checkpoint;
  el comando reporta el estado por-repo y sale con código ≠ 0. Sinergia con el gap 1: cada repo
  ya migrado se puede revertir con `axiom rollback <checkpointId>` (por-repo).
- **`--dry-run` fan-out:** previsualiza el plan por-repo (`previewUpgrade` por repo), sin mutar.

## Scope

- `apps/cli/src/commands/upgrade.ts` — únicamente aquí:
  - Nuevo flag `--repo-only` + arg `repoOnly?: boolean` en `UpgradeArgs`.
  - Detección de fan-out dentro de `runUpgrade` (tras `withProjectContext` + entrar al gate del
    control): `loadTopology(rootPath).mode === 'multi-repo'` && repo de control && !repoOnly.
  - Orquestación fan-out: enumerar repos de topología, `resolveRepoPath` cada uno (dedupe por
    path absoluto; control primero), `executeUpgrade`/`previewUpgrade` por repo con
    `runSync:false,runDoctor:false`, recolectar `PerRepoUpgradeResult[]`.
  - `sync`+`doctor` una vez al final (control), salvo flags.
  - Formatters de salida para el reporte por-repo (mirror de `formatResult`/`formatPlan`).
  - Tipo de retorno extendido (unión): además de los casos single-repo actuales, un caso
    `{ dryRun:false, fanout:true, repos: PerRepoUpgradeResult[], syncRun, doctorRun }` y el dry-run
    fan-out equivalente.
- `@axiom/topology` y `@axiom/versioning` se **reutilizan sin cambios** (ya son deps de
  `@axiom/cli-commands`, el owner de `upgrade.ts`). Si aparece una necesidad puntual (p. ej.
  exportar un helper), hacerla mínima y aditiva.
- **Tests** — unit/integration de fan-out en un proyecto multi-repo sintético: todos los repos
  migran/checkpointean; per-repo report correcto; `--repo-only` → single-repo; single-repo →
  byte-idéntico (regresión); dry-run fan-out no muta; fallo en un repo → report + exit≠0 sin
  abortar la contabilidad. NO debilitar tests existentes de `upgrade`.
- **e2e en sandbox aislado** (KVP25, HOME hermético): `workspace setup` del multi-repo →
  `axiom upgrade` desde el control → los 5 repos reportan checkpoint (`.axiom-state/.../checkpoints`
  presente en cada uno); `axiom upgrade --repo-only` toca sólo el control.

## Non-goals

- No fabricar `init.json` en repos de rol/spec (no hace falta: `executeUpgrade` es gate-free).
- No cambiar la semántica de `--from-checkpoint` (gap 1) ni de `executeUpgrade`.
- No cambiar el comportamiento single-repo (byte-idéntico).
- No un `ManagedState` único global compartido entre repos (se elige per-repo).

## Acceptance

- En un proyecto multi-repo, `axiom upgrade` desde el control migra/checkpointea **todos** los
  repos de la topología, con reporte por-repo.
- `axiom upgrade --repo-only` (o desde un repo no-control) conserva el modo per-repo de hoy.
- Un proyecto single-repo es byte-idéntico al comportamiento previo (sin fan-out).
- `--dry-run` en multi-repo previsualiza por-repo sin mutar.
- Gate verde en `Axiom/`: `npm run build` → `npm test` → `npm run typecheck` →
  `node apps/cli/dist/index.js doctor` (módulo el flake conocido 5000ms).

## Result

**Estado: implementado y validado.**

### Archivos cambiados (`Axiom.SDD`, working tree — sin commits)

- `apps/cli/src/commands/upgrade.ts` — ÚNICO archivo con la lógica nueva:
  - `UpgradeArgs.repoOnly?: boolean` + flag `--repo-only`.
  - `resolveFanoutTargets(resolution, repoOnly)`: decide si el fan-out aplica
    (`resolution.role === 'sdd'` && `loadTopology(rootPath).mode === 'multi-repo'`
    && `!repoOnly`) y, si aplica, enumera `[sddRepo, specRepo,
    ...roleCodeRepositories]` resueltos a path absoluto (`resolveRepoPath` +
    `loadLocalBindings`, con fallback a bindings vacíos si `loadLocalBindings`
    falla), deduplicados por path absoluto (normalizado a lower-case en
    `win32`) y con el control primero.
  - `runUpgrade`: rama fan-out (real y dry-run) ANTES de la rama single-repo
    existente (que queda intacta, línea por línea). Real: `executeUpgrade`
    per-repo con `runSync:false,runDoctor:false`, try/catch por repo
    (`PerRepoUpgradeResult` ok|fail), luego `sync`/`doctor` UNA vez desde el
    control. Dry-run: `previewUpgrade` per-repo, sin mutar.
  - Tipos nuevos exportados: `PerRepoUpgradePlanEntry`, `PerRepoUpgradeResultOk`,
    `PerRepoUpgradeResultFail`, `PerRepoUpgradeResult`. `UpgradeRunResult` es
    ahora una unión de 4 casos (single-repo × 2 + fan-out × 2); cada variante
    declara los campos de las otras 3 como `?: undefined` para que código
    legacy que sólo chequea `dryRun` (sin chequear `fanout`) siga
    compilando/funcionando (`.plan`/`.result` quedan como `T | undefined`).
  - `formatFanoutPlan`/`formatFanoutResult` (mirror de `formatPlan`/
    `formatResult`) + wiring en la `action` de `registerUpgrade`: discrimina
    primero por `fanout`, imprime el reporte por-repo, y hace
    `process.exit(1)` si algún repo falló (D-004).
- `apps/cli/tests/e2e/adopt-upgrade.e2e.test.ts` — 2 líneas: se agregó
  `repoOnly: true` a las 2 llamadas a `runUpgrade`. **Motivo (hallazgo, ver
  abajo)**: el fixture de este test (control+spec en paths distintos, sin
  repos de rol) YA calificaba como `mode: 'multi-repo'` bajo la nueva regla
  D-001, así que hubiese disparado fan-out y roto `dry.plan` (el test es
  sobre el gate de `init.json`, no sobre fan-out) — `repoOnly: true` lo fija
  al comportamiento single-repo pre-existente, sin debilitar ninguna
  aserción.
- `packages/tui/src/driver.ts` — wrapper aditivo alrededor de `cliRunUpgrade`
  (antes asignado directo a `runners.upgrade`): la TUI no soporta fan-out
  todavía (fuera de alcance de este incremento); el wrapper narrowea el
  resultado y lanza un error claro si `fanout: true` en vez de romper la
  compilación o renderizar campos `undefined` silenciosamente. Necesario
  porque `UpgradeRunResult` (el tipo real que `cliRunUpgrade` retorna) ganó 2
  variantes no asignables al tipo delgado `FlowRunners['upgrade']`.
- `apps/cli/tests/upgrade-fanout.test.ts` (nuevo) — 5 escenarios (ver
  Validation).

### Diseño as-built (idéntico a D-001..D-004 de `metadata.yml`)

Sin desviaciones de las decisiones D-001..D-004. Un matiz encontrado durante
la implementación (no una desviación, un detalle de interacción):
`loadTopology` cae a `defaultInstalledMultiRepoManifest` (mode `'multi-repo'`)
incluso SIN `topology.yaml` propio cuando `axiom.yaml` v2 declara
`mode: 'installed-multi-repo'` (hint de `tryLoadTopologyHint`) — por eso un
"single-repo" v2 genuino en los tests necesita `mode: 'one-axiom-per-product'`
en su `axiom.yaml` para no activar ese fallback. Documentado en el comentario
de `writeAxiomYamlV2` del nuevo test file.

### Tests

- Nuevo `apps/cli/tests/upgrade-fanout.test.ts` (5 tests, fixture multi-repo
  sintético con control+spec+backend en 3 tmpdirs distintos, topology.yaml +
  topology-bindings.yaml a mano — mismo patrón que `repo-affinity.test.ts`):
  1. Fan-out real: los 3 repos migran/checkpointean; reporte control-first;
     `sync`/`doctor` corren UNA vez (no 3).
  2. `--repo-only`: sólo el repo de control se toca.
  3. Single-repo (sin `topology.yaml`, `mode: one-axiom-per-product`): nunca
     dispara fan-out aun con `role: sdd`.
  4. `--dry-run` fan-out: 0 checkpoints creados, planes por-repo retornados.
  5. Fallo en un repo (path de backend reemplazado por un archivo → ENOTDIR):
     ese repo se reporta `ok:false`, sdd/spec migran igual, y
     `repos.some(r => !r.ok)` (la condición de exit≠0 real) es `true`.
- `apps/cli/tests/e2e/adopt-upgrade.e2e.test.ts` (1 test, ajustado con
  `repoOnly: true`, ver arriba) y `apps/cli/tests/upgrade.test.ts` (4 tests,
  sin cambios) — verdes, sin debilitar ninguna aserción.
- `packages/versioning/tests/upgrade.test.ts` (12 tests) y `packages/tui`
  completo (14 archivos) — verdes sin cambios.

### e2e en sandbox aislado (KVP25, HOME hermético)

Copia throwaway `.../scratchpad/sandbox/work-gap2/` (baseline `kvp25/`
intacto, verificado post-run: timestamps sin cambios, `axiom.yaml` ausente).
`npm run build` corrido antes de invocar `dist/index.js`.

1. `workspace setup --name kvp25-gap2 --adopt-sdd . --adopt-spec <Kvp.Spec>
   --role backend:<Kvp.Api> --role frontend:<Kvp.App> --role e2e:<Kvp.QA> -y`
   desde `Kvp.Sdd` → adopción aplicada (`spec migrated: 59`), `axiom.yaml`
   v2 (`role: sdd`, `mode: installed-multi-repo`) + `topology.config/
   topology.yaml` (`mode: multi-repo`, 3 `roleCodeRepositories`) +
   `.axiom-state/kvp25-gap2/init.json` materializados en el control.
2. `axiom upgrade --no-sync --no-doctor` desde el control →
   `[axiom upgrade] fan-out: 5 repo(s), 0 fallo(s).` con las 5 entradas
   (`sdd-repo`, `spec-repo`, `backend`, `frontend`, `e2e`) en `[OK]`, cada
   una con su propio `checkpointId` distinto. Confirmado en disco: los 5
   repos (`Kvp.Sdd`, `Kvp.Spec`, `Repos KVP25/Kvp.Api`, `.../Kvp.App`,
   `.../Kvp.QA`) tienen `.axiom-state/kvp25-gap2/checkpoints/<id>/` con el
   id exacto reportado.
3. `axiom upgrade --repo-only --no-sync --no-doctor` desde el control →
   output single-repo de siempre (`[axiom upgrade] OK.`, sin texto
   "fan-out"). Conteo de checkpoints ANTES→DESPUÉS: `Kvp.Sdd` 1→2 (nuevo
   checkpoint), los otros 4 repos 1→1 (sin cambios) — confirma que
   `--repo-only` sólo toca el repo de control.

Throwaway `work-gap2/` borrado al terminar (mismo patrón que gap 1). Cero
mutaciones git en ningún repo; `~/.axiom` real y `C:\repos\KVP25 Workspace`
reales nunca tocados (todo corrió con `USERPROFILE`/`HOME` apuntando al
`.axiom-home` hermético del sandbox).

### Validación (gate en `Axiom/`)

- `npm run build` → limpio (`tsc -b`, sin output/errores).
- `npm run typecheck` → limpio (`tsc -b`, sin output/errores). Nota: `tsc -b`
  raíz NO incluye `apps/cli/tests/**` en su project graph (sólo `src/**/*`
  por paquete) — los tests se type-checkean implícitamente al ejecutarlos
  con `vitest run` (transpilación esbuild, sin chequeo de tipos estricto),
  no en este comando.
- `npm test` (`vitest run`, suite completa): **279 archivos de test, 2805
  tests** → 277 archivos/2803 tests verdes en la corrida completa + 2 fallos
  puntuales, AMBOS confirmados como flakes preexistentes y NO relacionados
  con este cambio (no tocan `upgrade`/`topology`/`versioning`/`tui`):
  - `packages/memory/tests/engram-backend.test.ts` — "Test timed out in
    5000ms" (el flake conocido mencionado en el brief) → re-corrido en
    aislamiento: **15/15 verdes**.
  - `packages/telemetry/tests/bus.test.ts` — assertion de timing
    (`expect(elapsed).toBeLessThan(10)`, sensible a carga de la máquina) →
    re-corrido en aislamiento: **11/11 verdes**.
  - Con ambos aislados: **279/279 archivos, 2805/2805 tests verdes**.
- `node apps/cli/dist/index.js doctor` → `Resultado: PASS` (45/57 OK, 0
  FALLO, 3 ADVERTENCIA, 9 OMITIDO — advertencias/omitidos pre-existentes, no
  relacionados con este incremento).

### Revisión contra criterios de aceptación

- ✅ Multi-repo desde el control migra/checkpointea TODOS los repos, con
  reporte por-repo (e2e sandbox + Scenario 1 del test file).
- ✅ `--repo-only` (y, por diseño, correr desde un repo no-control) conserva
  el modo per-repo de hoy (e2e sandbox + Scenario 2).
- ✅ Single-repo es byte-idéntico (sin fan-out) — Scenario 3 + el test e2e
  pre-existente `adopt-upgrade.e2e.test.ts` (ajustado con `repoOnly: true`
  para aislar la intención original del test del nuevo fan-out; ver
  hallazgo arriba) + `upgrade.test.ts` sin cambios, ambos verdes.
- ✅ `--dry-run` en multi-repo previsualiza por-repo sin mutar — Scenario 4.
- ✅ Gate verde en `Axiom/` (build/test/typecheck/doctor), flake de 5000ms
  aislado y confirmado no relacionado.

### Integración a `general-spec.md`

No se integró conocimiento nuevo a `Axiom.Spec/general-spec.md`. El diseño
de fan-out (D-001..D-004) ya vive consolidado en este mismo increment file
(fuente canónica del gap CC-VER-09), y el catálogo de corner cases
(`INC-20260713-e2e-corner-case-catalog`) es el lugar correcto para marcar
CC-VER-09 como cerrado — no se duplica contenido en `general-spec.md` por
ahora. Único matiz reusable descubierto (la interacción
`axiom.yaml#mode: installed-multi-repo` → `loadTopology` fallback a
`multi-repo` SIN `topology.yaml` propio) queda documentado in-situ (código +
este Result) por ser un detalle de implementación de bajo nivel, no una
decisión de producto que amerite `general-spec.md`.

Status final: **closed** (goal claro, acceptance criteria cumplidos,
cambios implementados, validación disponible ejecutada y reportada, revisión
contra acceptance hecha, integración a general-spec decidida explícitamente
como no aplicable con motivo).
