# Rol de plan: builder

## Rol

- nombre del rol: builder
- repositorio(s) asignado(s): `axiom-code-builder` como rol lógico de implementación sobre el repo runtime **Axiom**
- artefacto rector: `INC-20260909-r13-self-update-transactional-updater`
- posición: incremento **3/5** de la secuencia obligatoria **1 → 2 → 3 → 4 → 5**

## Objetivo del rol en este plan

Sustituir el apply simulado por un runner Git global transaccional y recuperable que cumpla ACC-078 y cierre `apply/recover` de ACC-079. El builder debe entregar código, tests y evidencia, pero no UI de Launcher, pipeline final de release ni transiciones de artifacts.

El trabajo se ejecuta fail-closed: un target no publicado, plan alterado, lock ambiguo, estado Git local, fallo de verificación o rollback no probado nunca se convierte en `installed`.

## Mapa previsto de archivos y símbolos

Las rutas siguientes son destinos de diseño para hacer el plan accionable; no afirman que esos archivos existan hoy. La tarea B0 debe mapearlas a la estructura real del repo sin duplicar módulos ni comandos. Si el paquete propietario usa otro nombre, se modifica el archivo existente equivalente y la evidencia registra el mapping.

| Ruta prevista | Símbolos/responsabilidad |
|---|---|
| `packages/core/src/self-update/contracts.ts` | `PublishedReleaseTarget`, `UpdatePlan`, `UpdateOutcome`, `UpdateResult`, `UpdatePhase`, `UpdateDiagnostic`, `ProgressEvent` |
| `packages/core/src/self-update/paths.ts` | `resolveUpdatePaths`, `canonicalizeManagedPath`, `assertOwnedPath`, `assertAtomicSibling` |
| `packages/core/src/self-update/process-runner.ts` | `spawnChecked`, `terminateProcessTree`, `ProcessFailure` |
| `packages/core/src/self-update/release-target.ts` | `resolvePublishedReleaseTarget`, `revalidatePublishedReleaseTarget` |
| `packages/core/src/self-update/planner.ts` | `previewGlobalUpdate`, `sealUpdatePlan`, `verifyPlanDigest`, `comparePlanPreconditions` |
| `packages/core/src/self-update/lock.ts` | `acquireGlobalUpdateLock`, `inspectLockOwner`, `releaseGlobalUpdateLock` |
| `packages/core/src/self-update/git-preflight.ts` | `inspectGitInstall`, `assertSafeGitState`, `assertTargetFastForward` |
| `packages/core/src/self-update/journal.ts` | `UpdateJournalStore`, `appendJournalTransition`, `loadRecoverableTransaction` |
| `packages/core/src/self-update/staging.ts` | `prepareReleaseStage`, `materializeTargetCommit`, `cleanupOwnedStage` |
| `packages/core/src/self-update/build.ts` | `installAndBuildCandidate`, `assertCandidateLockfile` |
| `packages/core/src/self-update/verify.ts` | `verifyCandidateByAbsolutePath`, `verifyActiveEntry`, `observeExactVersion` |
| `packages/core/src/self-update/activation.ts` | `prepareActiveEntrySwap`, `swapActiveEntryAtomically`, `restoreActiveEntry` |
| `packages/core/src/self-update/manifest.ts` | `writeObservedInstallManifest`, `snapshotInstallManifest`, `restoreInstallManifest` |
| `packages/core/src/self-update/recovery.ts` | `rollbackGlobalUpdate`, `recoverGlobalUpdate`, `inspectTransactionalReality` |
| `packages/core/src/self-update/runner.ts` | `applyGlobalUpdate`, `runGlobalSelfUpdate`, outcome mapping y eventos |
| `packages/core/src/self-update/index.ts` | exports públicos headless para CLI y Launcher 4 |
| `apps/cli/src/commands/self-update.ts` | destino previsto del wiring del comando existente; usar la ruta real identificada por B0 y no crear un comando duplicado |
| `packages/core/src/self-update/*.test.ts` | unitarias de contratos, paths, procesos, plan, lock, journal, activation y recovery |
| `packages/core/src/self-update/__tests__/transaction.integration.test.ts` | integración con repos Git locales, fault injection y dos updaters |
| `apps/cli/src/commands/self-update.test.ts` | mapping CLI de outcomes/códigos y eliminación del falso éxito |

## Alcance incluido

- Inventariar contratos reales de los incrementos 1/2 y el apply simulado actual.
- Implementar tipos, adapters, planner, runner, lock, journal, staging, build, verificación, activación, manifest, rollback y recovery.
- Exponer API headless sin dependencias de Launcher.
- Cablear el comando existente y retirar cualquier ruta que reporte éxito sin instalación observada.
- Crear tests deterministas con repos Git locales y fault injection en cada frontera.
- Ejecutar build, tests focalizados/completos, matriz Windows/POSIX y review contra criterios.
- Preparar el evidence packet para habilitar incrementos 4 y 5.

## Fuera de alcance

- Crear el instalador inicial o adoptar instalaciones no gestionadas.
- Implementar ventanas, botones, progreso visual o relaunch del Launcher 4.
- Implementar publicación/promoción final de release y documentación del incremento 5.
- Reparar repositorios del usuario mediante stash/reset/clean/merge/rebase.
- Añadir un segundo shim/binlink o un comando paralelo de self-update.
- Editar metadata/status/receipts/índices o ejecutar transiciones Core como parte de este rol.
- Extender el cambio a workspaces de proyectos o configuración Git global.

## Dependencias

### Obligaciones previas

- Incremento 1/5 de R13 aceptado y con identidad/rutas de instalación global disponibles.
- Incremento 2/5 de R13 aceptado y con contrato de estado/release/preview disponible.
- ACC-078 y ramas apply/recover de ACC-079 aprobadas tal como se expresan en el incremento origen.
- Toolchain del repo Axiom funcional con baseline de `npm run build` y `npx vitest run`.
- Primitiva de reemplazo atómico demostrable para el único entry en Windows y POSIX.

Si falla cualquiera, B0 registra `STOP`; el builder no inventa sustitutos ni avanza a mutaciones.

### Dependientes habilitados

- Incremento 4/5: importa `UpdatePlan`, runner, eventos y outcomes; no recibe Git/filesystem internals.
- Incremento 5/5: consume evidence packet, versión observada y resultados de validación; no se implementa su pipeline aquí.

## Tareas ordenadas

Las tareas se ejecutan en orden. Cada una exige tests verdes y review de su frontera antes de avanzar; no se paralelizan cambios que muten el mismo contrato.

### B0 — Readiness, inventario y baseline

**Archivos/símbolos:** sin cambios funcionales al inicio; localizar módulo del comando existente, manifest/installation resolver, release provider y entry global. Mapearlos al cuadro anterior.

**Trabajo:**

1. Confirmar evidencia aceptada de incrementos 1/2 y secuencia 1 → 2 → 3 → 4 → 5.
2. Encontrar el fake apply y todos sus callers; identificar qué path escribe versión o emite éxito.
3. Identificar un único entry global, manifest, raíces gestionadas y convenciones de atomic writes.
4. Ejecutar baseline build/tests y guardar fallos preexistentes.
5. Confirmar que no se necesita ampliar blast radius.

**Evidencia/STOP:** mapping de archivos/símbolos, caller map, baseline y decisión de adapter atómico. `STOP` si hay dos entries autoritativos, dependencias no aceptadas o ninguna estrategia atómica viable.

### B1 — Contratos cerrados y separación de instalación inicial

**Archivos/símbolos:** `contracts.ts`, `index.ts`; `UpdatePlan`, `UpdateOutcome`, `UpdateResult`, `GlobalSelfUpdater`, `InitialInstaller`.

**Trabajo:** modelar solo `installed | unchanged | failed | recovery-required`, snapshots readonly, fases, errores y eventos. `GlobalSelfUpdater` debe rechazar ausencia de instalación con `initial-install-required`; no llama al instalador.

**Tests:** exhaustividad de outcomes, deep immutability, serialización estable, no import desde Launcher y probe de instalación ausente sin writes.

**Fault boundary:** `contracts.after-install-probe`; el fallo deja filesystem intacto y outcome `failed`.

### B2 — Paths y process runner multiplataforma

**Archivos/símbolos:** `paths.ts`, `process-runner.ts`; funciones del mapa.

**Trabajo:** canonicalizar raíces/entry/lock/journal/staging; validar same-volume, ownership, marker, symlink/reparse y case. Ejecutar procesos sin shell, cwd absoluto, timeout, abort signal, output acotado y termination del árbol.

**Tests:** Windows drive/case/UNC/reparse fixtures, POSIX symlink/permisos, paths con espacios, escapes, spawn ENOENT, timeout, signal y exit no cero.

**Fault boundaries:** `paths.after-canonicalize`, `process.after-spawn`, `process.before-timeout-kill`, `process.after-child-exit`. Ninguna rama construye strings shell.

### B3 — Release target y plan inmutable

**Archivos/símbolos:** `release-target.ts`, `planner.ts`; `resolvePublishedReleaseTarget`, `sealUpdatePlan`, `verifyPlanDigest`, `comparePlanPreconditions`.

**Trabajo:** consumir exclusivamente proveedor validado 1/2; resolver release/ref/commit; observar versión/entry/manifest/Git; generar payload canónico + digest y deep-freeze. Apply revalida en estructura separada.

**Tests:** release no publicada, ref móvil, commit distinto, plan tampered, serialization round-trip, drift entre preview/apply y target ya activo.

**Fault boundaries:** `target.after-release-resolve`, `target.after-ref-resolve`, `plan.before-seal`, `plan.after-seal`. No se abre journal mutable ni se escribe manifest.

### B4 — Lock global y dos updaters

**Archivos/símbolos:** `lock.ts`; `acquireGlobalUpdateLock`, `inspectLockOwner`, `releaseGlobalUpdateLock`.

**Trabajo:** creación exclusiva por installId; record sanitizado; timeout de espera; liberación condicionada al operation id; reclaim solo con propietario no vivo demostrado y CAS. Apply y recover usan la misma API.

**Tests:** dos procesos/updaters, lock vivo, lock dudoso, PID reciclado/host distinto, fallo de release y retry. La edad sola nunca elimina.

**Fault boundaries:** `lock.before-create`, `lock.after-create`, `lock.before-release`, `lock.after-owner-check`. Si ownership queda incierto, `recovery-required`.

### B5 — Git preflight fail-closed

**Archivos/símbolos:** `git-preflight.ts`; `inspectGitInstall`, `assertSafeGitState`, `assertTargetFastForward`.

**Trabajo:** inspeccionar remote, branch/upstream, HEAD, target tag/ref/commit, tracked/staged/untracked, detached y same/behind/ahead/diverged/unrelated. Emitir comandos argv no interactivos y sanitizar remote.

**Tests con repos Git locales:** clean-behind permitido; dirty tracked/staged/untracked, detached, branch/upstream incorrectos, ahead, diverged, unrelated, remote/ref/tag/commit inválidos bloqueados. Snapshot de `git status`, HEAD y archivos idéntico tras fallo.

**Fault boundaries:** `preflight.after-remote`, `preflight.after-status`, `preflight.after-ref`, `preflight.after-ancestry`. Nunca ejecutar stash/reset/clean/merge/rebase.

### B6 — Journal durable y snapshots previos

**Archivos/símbolos:** `journal.ts`; `UpdateJournalStore`, `appendJournalTransition`, `loadRecoverableTransaction`.

**Trabajo:** schema/checksum/sequence, temp + flush + atomic replace, snapshots de manifest/entry, transiciones monotónicas e idempotentes. El journal se abre antes de fetch/staging y no guarda secretos.

**Tests:** truncado, checksum/schema desconocido, fallo write/flush/rename, replay, transición fuera de orden y varios journals. Artefactos corruptos se preservan.

**Fault boundaries:** `journal.before-open`, `journal.after-temp-write`, `journal.after-flush`, `journal.after-replace`, y equivalentes para cada transición. Antes de activación, entry/manifest siguen idénticos.

### B7 — Fetch y staging gestionado

**Archivos/símbolos:** `staging.ts`; `prepareReleaseStage`, `materializeTargetCommit`, `cleanupOwnedStage`.

**Trabajo:** fetch del remoto autorizado; revalidar ref→commit; crear candidate root con marker/transaction id; materializar commit exacto aislado. No mutar el checkout activo. Cleanup solo con marker, realpath contenido, transaction id correcto y no-actividad.

**Tests:** fetch failure/timeout/signal, tag movido entre preview/apply, commit ausente, marker ajeno, symlink/reparse escape, path activo y retry. Verificar que no hay `git reset`/`clean`/stash.

**Fault boundaries:** `fetch.before-spawn`, `fetch.after-fetch`, `stage.after-root-create`, `stage.after-marker`, `stage.after-checkout`, `cleanup.before-owned-delete`. Un path dudoso se conserva y diagnostica.

### B8 — npm ci, build completo y candidato reproducible

**Archivos/símbolos:** `build.ts`; `assertCandidateLockfile`, `installAndBuildCandidate`.

**Trabajo:** comprobar lockfile del commit, ejecutar `npm ci` y luego `npm run build` (o el build completo real descubierto en B0) en cwd candidato. No reutilizar node_modules/dist de otra release.

**Tests:** lockfile ausente/modificado, npm spawn/timeout/signal/exit, build spawn/timeout/signal/exit, output previo contaminado y path con espacios.

**Fault boundaries:** `deps.before-npm-ci`, `deps.after-npm-ci`, `build.before-run`, `build.after-run`. Todo fallo preactivación devuelve `failed` con activo intacto.

### B9 — Verificación absoluta pre y post activación

**Archivos/símbolos:** `verify.ts`; `observeExactVersion`, `verifyCandidateByAbsolutePath`, `verifyActiveEntry`.

**Trabajo:** ejecutar `--version`, `--help`, load/import y doctor/smoke por ruta absoluta; normalizar solo terminadores permitidos y exigir igualdad exacta. Repetir versión + smoke sobre entry activo tras swap.

**Tests:** PATH apunta a binario señuelo, mismatch de versión, help/load/doctor fallan, timeout/signal/spawn, stdout extraño y candidate path con espacios.

**Fault boundaries:** `verify.before-version`, `verify.after-version`, `verify.after-help`, `verify.after-load`, `verify.after-doctor`, y equivalentes post-swap. Nada se activa si falla la suite candidata.

### B10 — Activación atómica del entry único

**Archivos/símbolos:** `activation.ts`; `prepareActiveEntrySwap`, `swapActiveEntryAtomically`, `restoreActiveEntry`.

**Trabajo:** confirmar entry único, snapshotear tipo/target/bytes/permisos, preparar sibling en mismo filesystem y reemplazar con adapter atómico. No unlink-first; si la plataforma no garantiza atomicidad, fallar antes. Restauración usa snapshot y la misma garantía.

**Tests:** observador concurrente nunca ve entry ausente/segundo autoritativo, fallo antes/durante/después del swap, permisos, Windows/POSIX y resultado ambiguo inspeccionado.

**Fault boundaries:** `activation.after-snapshot`, `activation.after-prepare`, `activation.before-swap`, `activation.after-swap`, `activation.before-restore`, `activation.after-restore`. Después de swap, cualquier fallo inicia rollback.

### B11 — Manifest observado y commit transaccional

**Archivos/símbolos:** `manifest.ts`, `journal.ts`; `writeObservedInstallManifest`, `restoreInstallManifest`, transición `committed`.

**Trabajo:** tomar la versión de verificación post-swap; escribir manifest temp + flush + replace; persistir release id/commit en campos propios; verificar lectura; después escribir journal `committed`. Nunca copiar expectedVersion al campo observado.

**Tests:** expected/observed diferentes, manifest previo/ausente, write/flush/rename/readback failure e interrupción postmanifest/precommit.

**Fault boundaries:** `manifest.before-temp`, `manifest.after-write`, `manifest.after-flush`, `manifest.after-replace`, `commit.before-journal`, `commit.after-journal`. Sin commit durable, la política es rollback al snapshot previo.

### B12 — Rollback y recovery idempotentes

**Archivos/símbolos:** `recovery.ts`; `inspectTransactionalReality`, `rollbackGlobalUpdate`, `recoverGlobalUpdate`.

**Trabajo:** rollback inverso manifest → entry → candidato release/commit/deps/build; verificar versión/manifest previos; recovery bajo lock compara journal y realidad. Commit válido se reconoce; cualquier transacción no comprometida vuelve al snapshot. Cleanup conservador.

**Tests:** interrupción en todas las fases, candidate activo/premanifest, manifest target/precommit, commit/precleanup, journal corrupto, rollback interrumpido, recover repetido y retry apply con plan nuevo.

**Fault boundaries:** `rollback.before-manifest`, `rollback.after-manifest`, `rollback.before-entry`, `rollback.after-entry`, `rollback.before-stage-cleanup`, `recovery.after-observe`, `recovery.before-decision`, `recovery.after-decision`. Fallo que impida probar consistencia = `recovery-required`.

### B13 — Runner, outcomes, cancelación y eventos

**Archivos/símbolos:** `runner.ts`, `index.ts`; `applyGlobalUpdate`, `runGlobalSelfUpdate`, progress sink.

**Trabajo:** orquestar B3–B12, mapear cada error por fase, evitar throws sin tipar en frontera pública, emitir eventos monotónicos, respetar abort y garantizar liberación condicional de lock. Target ya activo sano = `unchanged` sin build/swap.

**Tests:** tabla exhaustiva phase × failure × expected outcome; cancelación pre/post mutación, doble señal, lock release y diagnostics sanitizados.

**Fault boundaries:** `runner.before-phase` y `runner.after-phase` para cada fase además de los puntos específicos. El injector entra por DI solo en tests y no por entorno de producción.

### B14 — Wiring CLI y API para Launcher posterior

**Archivos/símbolos:** comando real localizado en B0 (destino previsto `apps/cli/src/commands/self-update.ts`), `index.ts`.

**Trabajo:** reemplazar fake apply por runner; preview imprime/serializa el plan; apply consume el plan exacto según contrato existente; recover usa el helper. Mapear outcomes a códigos de salida y mensajes. Exportar API sin UI.

**Tests:** ningún camino devuelve éxito por escribir una versión; JSON/humano derivan del mismo result; `failed` y `recovery-required` son no cero; fake consumer importa API sin Launcher. Buscar y retirar callers del falso apply sin eliminar comportamiento ajeno.

**Fault boundary:** `cli.after-result-before-render`; un fallo de render no reejecuta apply ni altera outcome/journal.

### B15 — Matriz integral, plataformas y evidence packet

**Archivos/símbolos:** tests del mapa; fixtures temporales locales.

**Trabajo:** ejecutar la matriz E2E completa en Windows y POSIX; dos updaters; todos los estados Git; build/version/manifest; rollback/retry; paths adversos. Medir fingerprints fuera de blast radius y demostrar entry único.

**Comandos:**

```text
npm run build
npx vitest run packages/core/src/self-update
npx vitest run apps/cli
npx vitest run
```

Además, el harness ejecuta por ruta absoluta `--version`, `--help`, load/import y `doctor` tanto antes como después del swap.

**Evidencia:** resultados por SO, tabla de fault boundaries, journals sanitizados representativos, snapshots pre/post, baseline comparison y lista exacta de archivos cambiados.

### B16 — Review, rollback de implementación e integración estable

**Archivos/símbolos:** diff completo y artifacts de evidencia; no ampliar código salvo correcciones de review.

**Trabajo:**

1. revisión semántica independiente contra todos los AC;
2. revisión de seguridad/no clobber y blast radius;
3. ensayo del rollback de código antes de release;
4. confirmar que revertir wiring deshabilita apply con error explícito y no revive falso éxito;
5. confirmar compatibilidad diagnóstica/recovery con schemas emitidos;
6. identificar conocimiento estable para specs generales sin copiar historial;
7. documentar API que habilita 4/5 y evidencia que habilita 5/5.

**Gate:** solo `GO` si build, suite completa, plataformas, fault matrix y review están verdes. Un fallo nuevo, evidencia ausente o dependencia no aceptada mantiene `STOP`. Cierre/archivado se realiza después mediante Axiom/Core, nunca desde este rol.

## Matriz obligatoria de fault injection

El siguiente catálogo debe existir como unión tipada de test; “antes” y “después” son fronteras separadas. La expectativa se valida además de inyectar la excepción.

| Frontera | Inyección mínima | Estado obligatorio |
|---|---|---|
| instalación/plan | antes/después de probe, resolve, ref y seal | sin writes; `failed` |
| lock | antes/después de create, owner-check y release | un solo owner; ambigüedad → `recovery-required` |
| journal | temp/write/flush/replace de apertura y transición | activo intacto pre-swap; journal recuperable o preservado |
| preflight | después de remote/status/ref/ancestry | Git idéntico; `failed` |
| fetch/stage | spawn/fetch/root/marker/materialize | activo intacto; solo cleanup acreditado |
| dependencias | antes/después de `npm ci` | activo intacto; `failed` |
| build | antes/después del build | activo intacto; `failed` |
| verify candidate | version/help/load/doctor | no swap; `failed` |
| prepare activation | snapshot/sibling/before swap | entry previo visible; `failed` |
| swap | inmediatamente después o resultado ambiguo | rollback probado o `recovery-required` |
| verify active | version/smoke post-swap | rollback probado o `recovery-required` |
| manifest | temp/write/flush/replace/readback | rollback entry+manifest o `recovery-required` |
| commit | antes/después de journal committed | antes: rollback; después: `installed` si estado coherente |
| rollback | antes/después de manifest, entry y stage | continuar idempotente o `recovery-required` |
| recovery | observe/decision/action/final verify | misma decisión al reintentar; nunca borrado ciego |
| cleanup | ownership check y delete acreditado | commit no se revierte; path dudoso se conserva con warning |
| render CLI | después del result | no reejecutar ni cambiar la transacción |

## Validaciones y evidencias esperadas

- Build TypeScript completo sin errores nuevos.
- Unitarias de cada adapter/contrato y tabla exhaustiva de outcomes.
- Integración con repos Git locales sin red pública.
- Dos updaters reales/procesos compitiendo por el mismo lock.
- Matriz dirty/detached/ahead/diverged y comprobación de no clobber.
- Fault injection en todas las fronteras de la tabla.
- Fallos de build/version/manifest, rollback, recovery repetido y retry.
- Evidencia Windows/POSIX con paths con espacios y escapes.
- Invocaciones absolutas de version/help/load/doctor y manifest con versión observada.
- Diff limitado al core/adapters/tests necesarios, sin UI Launcher ni pipeline/docs final.
- Revisión independiente con decisión explícita por ACC y gate STOP/GO.

## Bloqueos y observaciones

- El builder no puede declarar `GO` basándose solo en mocks de filesystem: swap y path semantics requieren evidencia en ambos SO.
- Una primitiva “casi atómica” o unlink-first es un bloqueo, no una optimización posterior.
- Un lock viejo no equivale a abandonado; un directorio con nombre esperado no equivale a owned.
- Si el layout real difiere del mapa previsto, el mapping B0 debe conservar responsabilidades y evitar módulos duplicados; esto no autoriza ampliar alcance.
- Los valores exactos de timeout se fijan en configuración/test conforme a convenciones existentes, pero todos los procesos deben tener límite y diagnóstico.
- No se afirma que el runtime actual tenga estos símbolos ni que las validaciones ya hayan corrido.
- La integración estable, transición y archivado ocurren solo después de la implementación/review y mediante Axiom/Core.
