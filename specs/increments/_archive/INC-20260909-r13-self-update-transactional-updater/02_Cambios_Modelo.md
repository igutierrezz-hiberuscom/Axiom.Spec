# 02 Cambios de Modelo

## Objetivo del documento

Definir las estructuras y máquinas de estado necesarias para que `preview`, `apply` y `recover` compartan una transacción verificable. Los nombres son contratos de diseño propuestos para la implementación del incremento; no afirman la existencia actual de archivos o símbolos.

## Entidades o estructuras afectadas

### Target de release publicado

```ts
type PublishedReleaseTarget = Readonly<{
  releaseId: string;
  expectedVersion: string;
  remoteIdentity: string;
  immutableRef: string;
  commit: string;
  validationEvidenceId: string;
}>;
```

`PublishedReleaseTarget` solo puede construirse desde el proveedor validado de releases entregado por los incrementos 1 y 2. `immutableRef` debe resolver a `commit` tanto al planificar como al aplicar. La estructura no contiene credenciales ni acepta branch/ref arbitrarias proporcionadas por UI.

### Snapshot de instalación y preflight

```ts
type InstallSnapshot = Readonly<{
  installId: string;
  installRoot: string;
  activeEntry: string;
  manifestPath: string;
  observedVersion: string;
  activeCommit: string;
  activeReleaseId?: string;
  entryFingerprint: string;
  manifestFingerprint: string | "absent";
}>;

type GitPreflight = Readonly<{
  remoteIdentity: string;
  branch: string;
  upstream: string;
  detached: boolean;
  headCommit: string;
  targetRef: string;
  targetCommit: string;
  trackedDirty: boolean;
  stagedDirty: boolean;
  untrackedDirty: boolean;
  relation: "same" | "behind-fast-forward" | "ahead" | "diverged" | "unrelated";
}>;
```

Los fingerprints permiten detectar drift entre preview y apply sin almacenar contenido sensible. `observedVersion` procede de ejecución absoluta del entry. Los estados dirty, detached, ahead, diverged y unrelated son bloqueantes.

### Plan inmutable

```ts
type UpdateStrategy = "managed-staging" | "proven-ff-only";

type UpdatePlan = Readonly<{
  schemaVersion: 1;
  planId: string;
  createdAt: string;
  install: InstallSnapshot;
  target: PublishedReleaseTarget;
  preflight: GitPreflight;
  strategy: UpdateStrategy;
  paths: UpdatePaths;
  commands: readonly PlannedCommand[];
  preconditions: readonly PlanPrecondition[];
  digest: string;
}>;
```

El payload se serializa de forma canónica, se calcula el digest sin el campo `digest`, se añade el digest y se congela profundamente. `preview` devuelve esa representación. `apply` verifica schema y digest y utiliza el mismo objeto lógico; la revalidación produce observaciones separadas y no reescribe el plan. Si hay drift, el plan se rechaza y debe generarse otro.

### Paths gestionados

```ts
type UpdatePaths = Readonly<{
  installRoot: string;
  releasesRoot: string;
  stagingRoot: string;
  activeEntry: string;
  manifestPath: string;
  lockPath: string;
  journalRoot: string;
}>;
```

Todas las rutas son absolutas y canónicas. `UpdatePaths` se deriva de `installId`, no de cwd. La abstracción debe modelar case/volumen en Windows, permisos y symlinks en POSIX, reparse points, espacios y límites de raíz. Staging y el sibling temporal del entry deben estar en el filesystem que garantice el swap atómico.

### Lock global

```ts
type UpdateLockRecord = Readonly<{
  schemaVersion: 1;
  installId: string;
  operationId: string;
  planDigest?: string;
  pid: number;
  hostId: string;
  acquiredAt: string;
}>;
```

El lock se adquiere mediante creación exclusiva o primitiva equivalente. Su reclamación requiere comprobar propietario y ejecutar compare-and-swap; `mtime` o edad no son prueba suficiente. `apply` y `recover` comparten lock y política.

### Journal transaccional

```ts
type UpdatePhase =
  | "started"
  | "locked"
  | "preflight-validated"
  | "target-fetched"
  | "candidate-prepared"
  | "dependencies-installed"
  | "candidate-built"
  | "candidate-verified"
  | "activation-prepared"
  | "entry-activated"
  | "active-verified"
  | "manifest-persisted"
  | "committed"
  | "rollback-started"
  | "rollback-completed"
  | "recovery-required";

type UpdateJournal = Readonly<{
  schemaVersion: 1;
  transactionId: string;
  installId: string;
  planDigest: string;
  phase: UpdatePhase;
  previous: PreviousStateSnapshot;
  candidate: CandidateState;
  activation?: ActivationSnapshot;
  manifest?: ManifestSnapshot;
  lastError?: ProcessOrUpdateError;
  sequence: number;
  checksum: string;
}>;
```

Cada transición incrementa `sequence` y se persiste mediante archivo temporal único, flush y rename/reemplazo atómico. El journal conserva suficiente evidencia para rollback sin incluir secretos ni stdout/stderr completos. Una transición solo se publica después de que el hecho correspondiente sea durable; las operaciones deben ser idempotentes ante repetición.

### Candidato, activación y manifest

```ts
type CandidateState = Readonly<{
  transactionId: string;
  ownedRoot: string;
  ownershipMarker: string;
  releaseId: string;
  commit: string;
  lockfileFingerprint: string;
  buildFingerprint?: string;
  absoluteEntry?: string;
  observedVersion?: string;
}>;

type ActivationSnapshot = Readonly<{
  activeEntry: string;
  previousKind: "file" | "symlink" | "binlink" | "absent";
  previousTarget?: string;
  previousBytesFingerprint?: string;
  preparedSibling: string;
  activatedFingerprint?: string;
}>;

type InstallManifest = Readonly<{
  schemaVersion: 1;
  installId: string;
  observedVersion: string;
  releaseId: string;
  commit: string;
  activeEntry: string;
  transactionId: string;
  observedAt: string;
}>;
```

`InstallManifest.observedVersion` es el único campo de versión instalada y se obtiene de `--version` sobre el entry activo después del swap. El valor esperado continúa en plan/journal como expectativa y nunca sustituye a la observación. La implementación debe snapshotear bytes y existencia previa del manifest para restauración exacta.

`ActivationSnapshot` modela un solo entry autoritativo. El reemplazo se prepara como sibling y se hace con la primitiva de reemplazo atómico de la plataforma; no se admite unlink-then-create. Si el entry es un binlink con representación específica de plataforma, el adapter debe tratarlo como una única unidad de activación verificable.

### Procesos y errores

```ts
type ProcessFailureKind = "spawn" | "timeout" | "signal" | "exit";

type ProcessFailure = Readonly<{
  kind: ProcessFailureKind;
  phase: UpdatePhase;
  executable: string;
  args: readonly string[];
  cwd: string;
  durationMs: number;
  exitCode?: number;
  signal?: string;
  systemCode?: string;
  stdoutTail?: string;
  stderrTail?: string;
}>;
```

Los argumentos y tails se sanitizan y limitan. El ejecutor usa `spawn` sin shell, cwd absoluto, entorno controlado, timeout por fase y terminación acotada del árbol de procesos. Una excepción nativa se traduce siempre a un error tipado.

### Resultados y API headless

```ts
type UpdateOutcome = "installed" | "unchanged" | "failed" | "recovery-required";

type UpdateResult = Readonly<{
  outcome: UpdateOutcome;
  operationId: string;
  planDigest?: string;
  observedVersion?: string;
  releaseId?: string;
  phase: UpdatePhase;
  diagnostics: readonly UpdateDiagnostic[];
}>;

type UpdateRunner = Readonly<{
  previewGlobalUpdate(input: PreviewInput): Promise<UpdatePlan>;
  applyGlobalUpdate(plan: UpdatePlan, options?: RunOptions): Promise<UpdateResult>;
  recoverGlobalUpdate(input: RecoveryInput, options?: RunOptions): Promise<UpdateResult>;
  runGlobalSelfUpdate(input: RunInput, options?: RunOptions): Promise<UpdateResult>;
}>;
```

`RunOptions` admite cancelación, timeouts, progress sink y dependencias inyectables. El `FaultInjector` solo se expone al composition root de tests. La API no importa Launcher ni componentes visuales. La instalación inicial dispone de otro contrato y nunca se elige implícitamente desde estos métodos.

## Contratos o estados afectados

### Máquina principal de apply

`started`, `locked` y el preflight son fases/eventos de ejecución anteriores al journal transaccional. Mientras posee el lock, apply crea como primer registro durable `preflight-validated`, antes de fetch o staging. El lock conserva su propio record de ownership y puede liberarse con seguridad aunque un rechazo de plan/preflight no haya abierto journal.

| Estado/fase | Invariante | Siguiente acción válida | Fallo esperado |
|---|---|---|---|
| `started` | Plan y digest válidos; ninguna mutación | adquirir lock | `failed` |
| `locked` | escritor exclusivo probado | revalidar release/instalación/Git | `failed`; lock se libera |
| `preflight-validated` | mundo coincide con plan | crear journal y preparar/fetch target | `failed` sin tocar activo |
| `target-fetched` | ref resuelve al commit exacto | materializar candidato propio | `failed` sin tocar activo |
| `candidate-prepared` | candidato contenido y marcado | `npm ci` desde lockfile | `failed` sin tocar activo |
| `dependencies-installed` | dependencias corresponden al lockfile | build completo | `failed` sin tocar activo |
| `candidate-built` | salidas pertenecen al target | verificación absoluta | `failed` sin tocar activo |
| `candidate-verified` | version/help/load/doctor válidos | preparar sibling de activación | `failed` sin tocar activo |
| `activation-prepared` | entry anterior snapshotteado | swap atómico | `failed` sin tocar activo o rollback si la primitiva es ambigua |
| `entry-activated` | target puede estar visible | verificar entry activo | rollback; si no se prueba, `recovery-required` |
| `active-verified` | entry activo sano y versión exacta | persistir manifest observado | rollback ante fallo |
| `manifest-persisted` | entry y manifest coherentes | escribir `committed` durable | rollback ante interrupción previa a commit |
| `committed` | transacción instalada y durable | liberar lock y cleanup conservador | `installed`; fallo de cleanup solo añade diagnóstico |

El caso target ya activo se comprueba bajo lock usando commit, release, entry y versión observada. Si todo está sano, no abre fases mutables y termina `unchanged`. Un manifest que diga target sin evidencia del binario no autoriza ese outcome.

### Máquina de rollback

1. Persistir `rollback-started` si el journal sigue escribible.
2. Restaurar exactamente el manifest previo si fue tocado.
3. Restaurar el entry previo mediante primitiva atómica y verificarlo por ruta absoluta.
4. Invalidar la candidatura como activa y tratar release/commit, dependencias y build candidatos como recursos de la transacción.
5. Eliminar solo recursos cuya propiedad, transaction id, realpath, contención y no-actividad se demuestren; conservar cualquier recurso dudoso.
6. Comparar versión observada y manifest con `PreviousStateSnapshot`.
7. Persistir `rollback-completed`; el `apply` que falló devuelve `failed`.
8. Ante cualquier imposibilidad de demostrar restauración, persistir cuando sea posible `recovery-required` y devolver ese outcome.

El rollback no usa `git reset`, `stash`, `clean` ni borrado recursivo sobre una raíz no autenticada.

### Máquina de recovery

`recover` adquiere el lock global, valida schema/checksum y observa entry, manifest, candidato, commit y marcadores antes de decidir:

| Realidad observada | Decisión |
|---|---|
| Journal terminal `committed`, entry/manifest/versión coherentes | reconocer `installed`; cleanup conservador opcional |
| Journal terminal `rollback-completed`, estado previo sano | `unchanged` |
| Sin commit y entry previo aún activo | completar rollback/cleanup propio y devolver `unchanged` |
| Sin commit y candidate entry activo | restaurar entry y manifest previos; `unchanged` si queda probado |
| Manifest target escrito pero sin commit durable | restaurar snapshot previo; no promover silenciosamente |
| Journal corrupto, ownership ambiguo o entry no atribuible | no borrar ni activar; `recovery-required` |
| Rollback falla o una señal interrumpe recovery | `recovery-required` |

La segunda ejecución sobre un estado ya reconciliado produce el mismo outcome lógico y no repite una mutación dañina.

### Semántica de outcomes

- `installed`: solo después de `committed` y verificación post-activación.
- `unchanged`: target ya estaba activo y sano, o recovery dejó/restauró de forma probada el estado previo; no implica que el intento fallido haya instalado nada.
- `failed`: apply no instaló el target y verificó que el estado anterior sigue sano; incluye lock ocupado, preflight inválido y fallos pre-activación.
- `recovery-required`: entry, manifest, lock o journal no permiten probar una configuración coherente, o rollback/recovery no concluyó.

No existen outcomes alternativos como `success`, `updated`, `rolled-back` o `partial`; el detalle vive en diagnósticos y fase.

### Fronteras de fault injection

El modelo incluye puntos deterministas antes y después de: lock, validación del plan, apertura de journal, preflight, fetch, materialización del candidato, `npm ci`, build, verificación, preparación del entry, swap, verificación activa, escritura del manifest, commit, cada paso de rollback y cleanup. Antes del swap, el entry/manifest deben permanecer idénticos. Después del swap y antes de commit, el sistema debe restaurar y probar el snapshot o devolver `recovery-required`.

## Notas de compatibilidad

- Los schemas de plan, lock, journal y manifest se versionan desde `1`; una versión desconocida se rechaza sin mutación. Recovery debe conservar el artefacto desconocido para inspección.
- El adapter de paths debe producir la misma identidad lógica en Windows y POSIX sin aplicar reglas POSIX a drive letters, UNC, case o reparse points.
- Los archivos temporales de journal, manifest y entry se crean en el mismo directorio/volumen que su destino y con permisos no más amplios que el original.
- Las instalaciones antiguas sin identidad/manifest gestionados se derivan al instalador inicial; no se migran como efecto lateral.
- El runner no cambia el contrato de publicación de releases ni crea tags. Solo consume evidencia validada.
- El incremento 4 depende de tipos y eventos headless, no de detalles internos de Git/filesystem. El incremento 5 depende de outcomes/evidencias estables, no del journal como API pública.
- El rollback de código de este incremento debe conservar lectura diagnóstica de schemas ya emitidos o deshabilitar apply con error explícito; nunca debe volver a reportar un apply simulado como éxito.
