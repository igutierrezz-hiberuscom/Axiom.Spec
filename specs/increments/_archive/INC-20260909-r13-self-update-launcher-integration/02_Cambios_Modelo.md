# 02 Cambios de Modelo

## Objetivo del documento

Definir las proyecciones, estados y bindings que necesita el Launcher para representar los contratos aportados por los incrementos 1–3 sin convertirse en autoridad alternativa. Son view models y envelopes de integración; no sustituyen los modelos canónicos de release, manifest, plan, journal o updater.

## Entidades o estructuras afectadas

### 1. `VersionEvidenceView`

Cada identidad se representa junto con **su propia** freshness y provenance; no existe una provenance global que pueda atribuirse por accidente a las tres versiones.

| Campo | Tipo conceptual | Regla |
|---|---|---|
| `version` | versión o `null` | `null` significa desconocida; nunca se sintetiza `0.0.0`. |
| `identityKind` | `published`, `downloaded`, `installed` o `project-runtime` | Determina qué autoridad puede demostrarla. |
| `freshness` | `EvidenceFreshnessView` | Calidad temporal de esta identidad concreta. |
| `provenance` | `ReleaseProvenanceView` o `null` | Evidencia propia de esta identidad; no se hereda de otra fila. |
| `validity` | `valid`, `invalid` o `unknown` | Resultado del contrato de identidad, no decisión del frontend. |
| `reason` | código tipado o `null` | Motivo sanitizado si falta o es inválida. |

Una identidad instalada puede tener provenance local sin remoto/ref; una publicada debe demostrar el remoto canónico; una descargada debe demostrar el artefacto/commit local correspondiente. Los campos no aplicables permanecen ausentes, no se copian de otra identidad.

### 2. `SelfUpdateInventoryView`

Proyección global y read-only del estado conocido:

| Campo | Tipo conceptual | Regla |
|---|---|---|
| `installationId` | string opaco | Derivado server-side; nunca elegido por el browser. |
| `publishedVersion` | `VersionEvidenceView` | Release más reciente demostrada por remoto canónico. |
| `downloadedVersion` | `VersionEvidenceView` | Release canónica disponible localmente según el incremento 1. |
| `installedVersion` | `VersionEvidenceView` | Versión observada de la instalación global activa. |
| `runtimeVersion` | `VersionEvidenceView` o `null` | Contexto opcional del proyecto; no reemplaza `installedVersion`. |
| `state` | `SelfUpdateInventoryState` | Estado primario tipado del contrato del incremento 1. |
| `lastKnownRelation` | `SelfUpdateRelationState` o `null` | Relación derivable de evidencia cacheada cuando `state=offline`. |
| `refresh` | `DiscoveryRefreshView` | Estado de conectividad/tentativa, separado de las evidencias. |
| `activeOperation` | resumen o `null` | Operación/journal reconciliado, si existe. |
| `capabilities` | flags tipados | Disponibilidad real de check/plan/apply/recover/cancel. |

Las tres evidencias son independientes. La UI no extrae una versión y luego le aplica freshness/provenance de otra.

### 3. Estado de relación, offline y precedencia

`SelfUpdateRelationState` describe la relación entre identidades válidas:

- `updated`;
- `update-available`;
- `checkout-behind`;
- `ahead`;
- `installed-misaligned`;
- `unknown`.

`SelfUpdateInventoryState` conserva la unión cerrada del incremento 1 y añade `offline` como estado primario de disponibilidad:

- cuando la consulta canónica actual es válida, `state` coincide con la relación calculada;
- cuando la consulta canónica falla por conectividad/timeout, `state=offline` y `lastKnownRelation` conserva la última relación demostrable, si existe;
- cuando no hay evidencia suficiente con conectividad disponible, `state=unknown`;
- cuando no hay evidencia y tampoco se pudo consultar la autoridad, `state=offline` y `lastKnownRelation=null`.

Ejemplo: una actualización conocida en cache con remoto inaccesible se representa como `state=offline` + `lastKnownRelation=update-available`, no como una elección ambigua entre ambos. Las capacidades server-side deciden si puede planificarse con artefactos locales; el frontend no infiere permiso a partir del estado.

### 4. `EvidenceFreshnessView`

Freshness propia de una identidad:

| Campo | Valores/regla |
|---|---|
| `kind` | `live`, `cached`, `stale`, `local` o `unknown`. |
| `observedAt` | Timestamp de esa evidencia; `null` si nunca existió. |
| `ageMs` | Antigüedad no negativa calculada server-side. |
| `source` | `canonical-remote`, `cache`, `local-artifact`, `active-installation` o `project-runtime`. |

`local` no significa live remoto. La UI no convierte `cached`/`stale` en `live` y muestra cada timestamp en su fila.

### 5. `ReleaseProvenanceView`

Proyección sanitizada y ligada a una única `VersionEvidenceView`:

- autoridad que la demostró;
- identificador normalizado del remoto sin userinfo, tokens ni secretos cuando aplique;
- ref/tag, commit exacto y versión demostrada cuando aplique;
- identidad de artefacto/entry point local cuando corresponda, sin exponer paths innecesarios;
- timestamp y método de observación;
- indicador de validación de schema/identidad;
- motivo tipado cuando la provenance no puede demostrarse.

Contenido arbitrario de Git, stderr, URLs con credenciales y paths locales innecesarios no forman parte de la respuesta HTTP.

### 6. `DiscoveryRefreshView`

Estado global de la tentativa de refresh, separado de las evidencias por identidad:

| Campo | Valores/regla |
|---|---|
| `connectivity` | `online`, `offline` o `unknown`. |
| `status` | `idle` o `in-flight`. |
| `requestId` | ID monotónico/opaco de la tentativa actual o `null`. |
| `deadlineAt` | Deadline de la tentativa activa o `null`. |
| `lastOutcome` | `success`, `timeout`, `cancelled`, `unavailable`, `invalid-provenance` o `never`. |
| `lastAttemptAt` | Timestamp del último intento o `null`. |

Timeout, `unavailable` o fallo de conectividad cambian `refresh.connectivity=offline`, fijan `state=offline` y conservan las evidencias por identidad. `cancelled` mantiene state/conectividad previos y solo actualiza `lastOutcome`; `invalid-provenance` mantiene conectividad online, fija relación/state `unknown` y bloquea apply.

### 7. `LauncherUpdatePlanView`

Adaptación de solo lectura del `UpdatePlan` sellado por el incremento 3:

- `planId` opaco y `planDigest` estable;
- `operationId` reservado e impredecible para fencing/idempotencia del handoff;
- `installationId` y evidencia de versión/commit instalados;
- versión/ref/commit target canónicos;
- precondiciones y warnings tipados;
- fases previstas y boundary irreversible;
- política elegida `close-only` o `close-and-restart`;
- capacidad de cancelación por fase;
- resumen de rollback/recovery;
- `createdAt` y expiración/revalidación exigida;
- confirmation grant entregado fuera del contenido visible, nunca persistido por el cliente.

El Launcher no recalcula el digest ni cambia target/operationId. Un plan actualizado crea una identidad nueva; no se parchea el anterior.

### 8. `LauncherConfirmationBinding`

Binding server-side que amplía el grant R-13.4 sin crear otro mecanismo:

| Dimensión | Valor ligado |
|---|---|
| Sesión | ID/hash de la sesión por proceso. |
| Instalación | `installationId` global derivado. |
| Operación | `operationId` reservado por el plan. |
| Acción | `apply`, `recover-resume` o `recover-rollback`. |
| Snapshot | Digest canónico del plan o recovery plan. |
| Target | versión/ref/commit exactos cuando aplica. |
| Lifecycle | `close-only` o `close-and-restart`. |
| Expiry | TTL de 120 segundos. |

El token es aleatorio, single-use, se almacena hasheado, no viaja en URL y se consume antes del handoff mutante. Sesión nueva, replay o cualquier mismatch produce rechazo. Un token consumido no se restaura aunque se pierda el ACK del helper.

### 9. `SelfUpdateOperationView`

La operación separa ejes para no mezclar estados locales de UI con fases/outcomes del runner:

| Campo | Tipo conceptual | Regla |
|---|---|---|
| `operationId` | string opaco | Reservado por el plan y clave idempotente del helper/journal. |
| `kind` | `apply`, `recover-resume` o `recover-rollback` | Acción exacta. |
| `planDigest` | string | Snapshot vinculado al grant. |
| `handoffState` | `not-started`, `launching`, `acknowledged`, `uncertain` o `failed` | Estado del protocolo Launcher↔helper. |
| `enginePhase` | `SelfUpdateEnginePhase` o `null` | Fase emitida por el runner, nunca inventada por UI. |
| `runnerOutcome` | `null`, `installed`, `unchanged`, `failed` o `recovery-required` | Outcome canónico del incremento 3. |
| `cancellationState` | `not-available`, `available`, `requested`, `accepted` o `rejected` | Eje independiente de la fase. |
| `relaunchState` | `not-requested`, `pending`, `succeeded` o `failed` | Lifecycle de Launcher, separado del outcome. |
| `sequence` | entero monotónico | Detecta eventos stale/gaps. |
| `startedAt`, `updatedAt` | timestamps | Observabilidad bounded. |
| `progress` | resumen sanitizado | Nunca stdout/stderr bruto. |
| `recovery` | `RecoveryView` o `null` | Estado derivado del journal. |

El error estable para `relaunchState=failed` es `relaunch_failed`; no se añade ese valor al outcome del engine. Una cancelación aceptada se presenta como tal solo después del ACK del runner y de reconciliar que no hubo activación.

### 10. `SelfUpdateEnginePhase`

Unión cerrada mapeada desde el runner:

- `accepted`;
- `acquiring-lock`;
- `revalidating`;
- `preparing`;
- `building`;
- `verifying`;
- `awaiting-launcher-exit`;
- `activating`;
- `persisting`;
- `rolling-back`;
- `recovering`;
- `succeeded`;
- `failed`;
- `recovery-required`.

`launching-helper` pertenece a `handoffState`; `cancel-requested` pertenece a `cancellationState`; `relaunching/relaunch-failed` pertenecen a `relaunchState`. No se exige porcentaje sintético.

### 11. Progreso y cotas temporales

- Cada evento JSON serializado tiene máximo **4 KiB**. Si el runner entrega uno mayor, el adaptador no encola el original: emite un evento fijo sanitizado `progress-truncated`, conserva el sequence y fuerza consulta del snapshot.
- El historial expuesto se limita a **100 eventos o 64 KiB**, lo que ocurra primero; al alcanzar el límite se conserva snapshot actual y se marca truncación.
- Handshake helper: default **5 s**, máximo server-side **10 s**.
- Drenaje/cierre HTTP/SSE: default **3 s**, máximo **5 s**.
- Espera del PID antes de activación: default **15 s**, máximo **30 s**; al vencer, no se activa y se registra outcome tipado/journal.
- Acknowledgement de relaunch: default **10 s**, máximo **30 s**; su timeout no cambia el outcome de instalación.
- Ninguna cota puede ser ampliada desde el browser. Shutdown cancela timers y procesos/handoffs aún no comprometidos.

### 12. `RecoveryView`

Proyección del journal durable del incremento 3:

- `journalId`/`operationId` opacos;
- último checkpoint y fase observada;
- instalación/target vinculados;
- evidencia disponible para `resume` y/o `rollback`;
- acciones permitidas y motivos de acciones deshabilitadas;
- necesidad de red o posibilidad de recovery offline;
- restart policy explícita, sin valor por defecto;
- warnings y next step sanitizados.

La UI nunca fabrica opciones ni elimina el journal. El runner decide si resume o rollback son seguros e idempotentes.

### 13. Envelopes HTTP

Las rutas nuevas reutilizan el envelope/versionado y la taxonomía del contrato CLI/control plane. Toda respuesta incluye:

- versión de schema;
- `requestId` u `operationId` según corresponda;
- `ok` y payload tipado o error estable;
- timestamp server-side;
- indicadores de stale/truncation cuando proceda.

No se añade un envelope incompatible con CLI ni se devuelven instancias de `Error`, stacks o stdout/stderr sin procesar.

## Contratos o estados afectados

### Estado de descubrimiento

```text
status local -> evidencias independientes + relación conocida/unknown
refresh inicia -> refresh.status=in-flight (sin borrar evidencias)
refresh éxito válido -> connectivity=online; state=relación actual; evidencias observadas se actualizan
refresh timeout/unavailable/conectividad -> connectivity=offline; state=offline; lastKnownRelation conserva relación cacheada
refresh invalid-provenance -> connectivity=online; state/relación=unknown; apply bloqueado
cancel manual/shutdown pre-commit -> refresh idle + outcome cancelled; conserva state/conectividad/evidencias previos
```

El refresh no atraviesa estados de operación y nunca desemboca en `apply`.

### Estado de plan y confirmación

```text
no-plan -> planning -> preview-ready(planDigest + operationId + grant)
planning -> error -> no-plan + error tipado
preview-ready -> editar/refresh/drift/expiry/session change -> invalidated
preview-ready -> confirmar -> grant consumido -> handoff launching
preview-ready -> cancelar -> no-plan
```

El cliente puede invalidar preventivamente; el servidor revalida siempre. Un grant consumido no vuelve a estado válido.

### Fencing del handoff

```text
operationId reservado + grant consumido
  -> helper intenta claim atómico de operationId/journal
  -> ACK durable -> handoffState=acknowledged
  -> timeout/pérdida de canal -> handoffState=uncertain -> reconciliar operationId + journal + updateOperationLock
       -> journal/claim existe: adjuntar a la operación existente
       -> ausencia demostrada + helper terminado: handoffState=failed; exigir plan/grant nuevos
       -> evidencia inconclusa: bloquear retry y mantener uncertain
ACK tardío del mismo operationId -> reconciliar idempotentemente; nunca crear segunda operación
```

La UI nunca vuelve directamente a plan listo después de un ACK ambiguo. El helper rechaza claims duplicados del mismo `operationId` devolviendo el snapshot existente.

### Estado de operación

```text
handoff acknowledged -> enginePhase accepted -> acquiring-lock -> revalidating
 -> preparing/building/verifying -> awaiting-launcher-exit -> activating -> persisting
 -> runnerOutcome installed|unchanged|failed|recovery-required

cancellationState available -> requested -> accepted|rejected
relaunchState not-requested|pending -> succeeded|failed
journal recovery-required -> recovery plan/confirm -> enginePhase recovering -> outcome
```

`relaunchState=failed` no prueba una instalación fallida. La versión instalada se vuelve a observar antes de decidir outcome.

### Exclusión mutua

- Un refresh puede coexistir con lectura de status, pero se coalesce con otro refresh.
- Un handoff `launching`, `uncertain` o `acknowledged` bloquea cualquier retry con otro operationId hasta reconciliarse.
- Un apply/recover activo bloquea nuevo plan mutante y cualquier segundo apply/recover.
- `recovery-required` bloquea apply nuevo hasta resolver el journal.
- El `updateOperationLock` y la autoridad de serialización permanecen en el runner del incremento 3; el `stateWriteLock` del 2 no autoriza apply/recover.

### Efectos permitidos por operación

| Operación | Red/ref/cache | Instalación activa | Manifest/journal | Helper |
|---|---|---|---|---|
| `status` | No | No | No | No |
| refresh de arranque / `check` | Solo fetch canónico y evidencia permitida | No | No journal transaccional | No |
| `plan` / recovery plan | No side effect adicional | No | No | No |
| `apply` confirmado | Según plan del engine | Sí, por engine | Sí, por engine | Sí |
| `recover` confirmado | Solo lo requerido por recovery | Sí, por engine | Sí, por engine | Sí |
| cancelación segura | Limpieza de staging del engine | No activación nueva | Checkpoint tipado | Helper activo hasta ACK |

## Notas de compatibilidad

- El contrato HTTP se añade sin debilitar las rutas existentes de R-13.4. Todas las rutas nuevas entran en las mismas tablas de método/schema/fields y middleware de seguridad.
- La UI puede degradarse a inventario read-only cuando engine/helper no estén disponibles; no oculta `engine_unavailable` ni ofrece botones ficticios.
- CLI y Launcher conservan envelopes de presentación distintos si es necesario, pero consumen el mismo resultado headless y la misma taxonomía.
- Manifest v1/v2, migración, escritura durable y `stateWriteLock` siguen bajo el incremento 2; ese lock evita lost updates del estado y Launcher nunca escribe `install.json` directamente.
- `UpdatePlan`, Git/staging, activation, journal, rollback, `updateOperationLock` y recovery del engine siguen bajo el incremento 3; ese lock serializa apply/recover. El helper externo, claim idempotente, ACK, fencing, espera del PID, reattach y relaunch son propiedad de R13-4 y consumen esa API. Ambos contratos pueden compartir primitivas de Core, pero tienen owner y responsabilidad distintos. Si el runner de R13-3 o el helper de R13-4 no soportan el handoff completo, G2 es `STOP`.
- La sesión cambia al reiniciar. La operación se reanuda por journal/`operationId`, no por cookie ni token antiguos.
- El progreso tolera reconexión, eventos perdidos y una versión nueva del Launcher; el snapshot durable es autoridad y los eventos son una optimización.
- Los paths y nombres de módulos HTTP concretos se fijan en implementación después del gate de dependencias; no son evidencia de capacidad actual.
