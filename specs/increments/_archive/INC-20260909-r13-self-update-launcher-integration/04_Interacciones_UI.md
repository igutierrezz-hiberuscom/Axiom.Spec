# 04 Interacciones UI

## Objetivo del documento

Especificar de forma exhaustiva la futura experiencia de self-update en el Launcher: información visible, estados, transiciones, preview/confirmación, cancelación, progreso, cierre/reinicio, offline, errores y recovery. Esta superficie aún no existe en el Launcher actual; el documento define el comportamiento que deberá implementarse y validar.

## Superficie UI afectada

### Ubicación y scope

El Launcher incorporará una sección global **“Actualización de Axiom”** accesible desde su navegación principal o área de estado. La sección:

- se identifica como perteneciente a la instalación global user-level;
- no depende del proyecto seleccionado ni queda oculta dentro de una acción project-scoped;
- puede mostrar la versión del runtime del proyecto como contexto separado;
- no se presenta como disponible hasta que el backend anuncie capability real;
- degrada a diagnóstico read-only si engine/helper no están disponibles.

### Estructura visual

1. **Resumen de estado**: badge primario tipado, `lastKnownRelation` si está offline y estado global del refresh.
2. **Inventario de versiones**: filas independientes para publicada, descargada e instalada; cada fila contiene su propio valor, freshness, timestamp, fuente y provenance. El runtime de proyecto es una cuarta fila claramente diferenciada.
3. **Provenance por identidad**: remoto/ref/commit sanitizados cuando aplican, fuente `canonical-remote/cache/local-artifact/active-installation` y momento observado. Nunca se aplica una única provenance global a todas las filas.
4. **Acciones**: “Comprobar ahora”, selector obligatorio de política de cierre, “Preparar plan”, “Revisar y aplicar” y, cuando corresponda, “Recuperar”.
5. **Plan**: snapshot inmutable del target, precondiciones, fases, warnings, cancelabilidad, rollback/recovery y expiración.
6. **Operación**: fase actual, historial acotado, estado de cancelación, aviso de cierre/reinicio y outcome.
7. **Errores/recovery**: error tipado, impacto, evidencia conservada y siguiente acción segura.

No se muestra stdout/stderr bruto, stack, token, cookie, URL con credenciales ni paths locales innecesarios.

## Flujo de interacción

### 1. Bootstrap y refresh de arranque

1. El bootstrap autenticado carga assets y solicita `status` local.
2. La sección renderiza de inmediato un skeleton acotado; no espera red para desbloquear el resto del Launcher.
3. Con la respuesta local muestra las tres evidencias por separado, el estado global del refresh y cualquier journal/operación pendiente.
4. El servidor agenda en background un único fetch del remoto canónico. La UI presenta “Comprobando en segundo plano” sin modal ni bloqueo global.
5. Al recibir un snapshot más nuevo, actualiza únicamente si su `requestId/sequence` supera al renderizado. Un resultado stale se ignora.
6. Si el fetch vence o falla por conectividad, conserva evidencias por fila, fija `state=offline` y muestra `lastKnownRelation`. Si se cancela manualmente, conserva state/conectividad previos; si la provenance es inválida con red disponible, usa `unknown` y bloquea apply.
7. Ningún resultado abre el diálogo de plan, selecciona restart policy, lanza helper o inicia apply.

El arranque no ejecuta `checkout`, `merge`, `pull`, cambio de branch ni instalación. El único side effect admisible es el fetch canónico y la evidencia/cache permitida.

### 2. “Comprobar ahora”

- Está disponible salvo que haya un refresh equivalente en curso; en ese caso muestra el mismo estado coalesced.
- Al pulsarlo, el botón cambia a “Comprobando…” y aparece “Cancelar comprobación” si el runner la declara cancelable.
- La cancelación detiene la tentativa cooperativamente, conserva state/conectividad/evidencias previos y registra `lastOutcome=cancelled`; no se etiqueta offline ni altera instalación.
- Un éxito actualiza evidencias. Timeout/unavailable muestra offline; provenance inválida muestra unknown/error y bloquea apply; cancelación conserva el estado previo.
- Check nunca prepara ni aplica una actualización. Si aparece `update-available`, se habilita “Preparar plan”, no “Actualizar ahora” directo.

### 3. Preparar plan

Antes de solicitar plan, el operador debe elegir una política sin valor preseleccionado:

- **“Cerrar al instalar; lo iniciaré manualmente”** (`close-only`);
- **“Cerrar y reiniciar Launcher al terminar”** (`close-and-restart`).

“Preparar plan” solo se habilita cuando:

- existe target canónico verificable;
- no hay operación apply/recover activa;
- no existe `recovery-required` pendiente;
- el engine está disponible;
- la policy de cierre fue elegida explícitamente.

Durante planning se mantiene visible el inventario que originó la acción. La respuesta presenta:

- versión/commit instalados y target;
- remoto/ref/commit canónicos;
- plan digest, `operationId` reservado, fecha y expiración en formato legible;
- precondiciones y warnings;
- fases previstas y punto tras el cual no se puede cancelar;
- alcance de rollback/recovery;
- policy de cierre/reinicio seleccionada.

Planning no muestra progreso de instalación ni cambia el badge a “Actualizando”. Si el estado cambia durante planning, la respuesta stale se descarta.

### 4. Preview y confirmación explícita

“Revisar y aplicar” abre un diálogo modal solo para un plan vigente. El diálogo:

- repite installed → target y commit/ref completos;
- indica que se actualizará la instalación global, no solo el proyecto;
- enumera cambios operativos: helper externo, posible preparación, cierre del Launcher y policy de reinicio;
- explica hasta qué fase se puede cancelar;
- muestra warnings y recovery disponible;
- no expone el confirmation token;
- inicia foco en el título, atrapa foco y devuelve foco al disparador al cerrar.

Acciones del diálogo:

- **“Volver”**: cierra sin side effect y conserva el plan mientras siga vigente.
- **“Cancelar plan”**: descarta localmente snapshot y grant.
- **“Aplicar, cerrar y no reiniciar”** para `close-only`, o **“Aplicar, cerrar y reiniciar”** para `close-and-restart`: única acción confirmatoria.

No se usa un botón genérico “Aceptar”. El grant está ligado a sesión, instalación, `operationId`, acción, plan digest, target y policy. Editar policy, recibir un refresh nuevo, cambiar de sesión, expirar, navegar/recargar o cancelar invalida la confirmación y obliga a regenerar plan/grant.

Al confirmar, los controles se deshabilitan contra doble click. El servidor consume el grant atómicamente; solo una petición concurrente puede continuar. Un rechazo devuelve el foco al plan y explica que debe revisarse otra vez.

### 5. Handoff al helper y progreso

Después de confirmar:

1. La UI entra en `handoffState=launching` y muestra “Iniciando helper seguro”. El `operationId` ya fue reservado por el plan; no lo genera el browser.
2. El helper reclama atómicamente `operationId` y materializa journal/digest antes de responder. Solo ese ACK durable cambia a `handoffState=acknowledged` y permite mostrar `enginePhase=accepted`.
3. Un fallo de spawn demostrado antes del claim cambia a `handoffState=failed`; el grant consumido no se recupera y cualquier retry exige plan/operationId/grant nuevos.
4. Si el ACK se pierde, vence o llega tarde, la UI cambia a `handoffState=uncertain`, mantiene Launcher abierto, bloquea reintentos y consulta por `operationId`, journal y `updateOperationLock`. Si aparece journal se adjunta a la operación existente; solo ausencia demostrada más helper terminado permite volver a planificar.
5. Un ACK tardío del mismo `operationId` se reconcilia idempotentemente. Nunca crea una segunda operación ni revive el token.
6. Tras ACK, el frontend sigue snapshots/eventos autenticados y mantiene separados `enginePhase`, `cancellationState`, `relaunchState` y `runnerOutcome`.
7. Cada evento serializado tiene máximo 4 KiB. Uno mayor se sustituye por `progress-truncated` y fuerza snapshot; nunca se inyecta el contenido original en DOM.
8. El historial conserva hasta 100 eventos o 64 KiB, muestra “historial truncado” si aplica y nunca inserta logs ilimitados.
9. No se fabrica porcentaje. Cuando no existe avance cuantificable se usa indicador indeterminado y texto de fase.
10. Un gap de sequence provoca una consulta de snapshot; no se adivinan eventos perdidos.

Mientras una operación esté activa:

- check/plan/apply/recover incompatibles quedan deshabilitados con razón;
- cerrar la pestaña no cancela;
- la vista muestra que la operación continuará mediante helper/journal;
- la navegación fuera de la sección no pierde seguimiento.

### 6. Cancelación

- “Cancelar actualización” aparece solo con `cancellable: true`.
- Pulsarlo abre una confirmación breve que distingue cancelar de cerrar la pestaña.
- Al confirmar, cambia `cancellationState` de `available` a `requested` y evita peticiones repetidas hasta ACK.
- Si el runner acepta, `cancellationState=accepted`; la UI solo muestra “Cancelado” tras reconciliar que no hubo activación y vuelve a status.
- Si la operación cruzó el boundary irreversible, el backend responde `cancellation_not_safe`, fija `cancellationState=rejected`, retira el botón y mantiene seguimiento.
- Nunca muestra “Cancelado” por timeout del browser sin outcome del helper.

### 7. Cierre y reinicio explícitos

La policy se eligió antes del plan y quedó ligada al grant. Cuando el helper llega a `awaiting-launcher-exit`:

- la UI anuncia que el helper está listo y que el proceso se cerrará conforme a la opción confirmada;
- el servidor termina entrega de estado y drena SSE/requests con default 3 s y máximo 5 s antes de cerrar;
- el helper espera la salida del PID con default 15 s y máximo 30 s; si vence, no activa y deja outcome/journal tipado;
- no hay reload in-process ni mezcla deliberada de módulos viejos/nuevos.

Para `close-only`, el helper no relanza. El último mensaje visible indica cómo iniciar Launcher manualmente; al volver, la nueva sesión reconcilia el journal.

Para `close-and-restart`, solo el helper puede relanzar después de verificar la instalación. La instancia nueva crea sesión nueva y debe confirmar relaunch dentro de 10 s por defecto/30 s máximo; luego carga status y muestra el `runnerOutcome`. Un timeout/fallo fija `relaunchState=failed`, muestra instalación observada y paso manual, y no se etiqueta automáticamente como rollback.

### 8. Reanudación y recovery

En cada bootstrap, antes de habilitar apply, status reconcilia journal:

- operación activa: muestra la fase/snapshot y reanuda observación;
- outcome terminal: muestra resumen y permite descartarlo solo según contrato del runner;
- `recovery-required`: presenta un banner prioritario, bloquea nuevo apply y abre sección Recovery;
- journal incompatible/corrupto: muestra error tipado y guidance; nunca lo borra ni sintetiza éxito.

Recovery lista exclusivamente acciones autorizadas por el runner:

- **Reanudar**: continuar desde checkpoint verificable;
- **Revertir**: restaurar la última instalación sana.

Cada opción exige elegir sin default `close-only` o `close-and-restart` y genera mediante una petición **preview** un recovery plan read-only con evidencia, efectos, requisito de red, cierre/reinicio y riesgos. Esa respuesta emite un grant nuevo ligado a action, journal, operationId y policy; una segunda petición **execute** confirmada lanza el helper. Preview nunca muta y no hay recovery automático al arranque.

### 9. Offline

- Con cache: el estado primario es `offline`; cada fila conserva su valor/freshness/provenance y el resumen muestra `lastKnownRelation`, timestamp/edad y acción de reintento. Plan/apply solo se habilitan si capabilities del runner demuestran evidencia/artifact local suficientes.
- Sin cache: `state=offline`, `lastKnownRelation=null` e identidades desconocidas; nunca se muestra “Estás actualizado”.
- Una operación/recovery ya preparada puede continuar offline solo si el runner lo autoriza. La UI no modifica esa decisión.
- El estado offline no oculta una operación o journal local.

## Estados visibles

### Estado de inventario

| Estado | Mensaje principal | Acción normal | Restricciones |
|---|---|---|---|
| `updated` | “La instalación coincide con la release publicada” | Comprobar | Plan deshabilitado salvo cambio posterior. |
| `update-available` | “Hay una actualización disponible” | Elegir policy y preparar plan | No aplicar directo. |
| `checkout-behind` | “La copia local está detrás de la release publicada” | Comprobar / inspeccionar | No implica que installed sea igual a downloaded. |
| `ahead` | “La instalación/copia está por delante de la release canónica” | Inspeccionar | Sin downgrade automático. |
| `installed-misaligned` | “La identidad instalada no coincide con su evidencia” | Diagnóstico/recovery si autorizado | Apply normal puede quedar bloqueado. |
| `unknown` | “No hay evidencia suficiente” | Comprobar | No concluye actualizado ni habilita apply. |
| `offline` | “No se pudo consultar el remoto canónico” | Reintentar | Muestra `lastKnownRelation`, si existe; capabilities deciden otras acciones. |

`offline` tiene precedencia como estado primario cuando falla la autoridad. La última relación (`update-available`, por ejemplo) aparece como dato secundario cacheado y nunca reemplaza ese badge.

### Freshness por identidad

| Estado | Tratamiento visual |
|---|---|
| `live` | Evidencia remota observada ahora/a las … con provenance propia. |
| `cached` | “Último dato válido de …”; no se presenta como live. |
| `stale` | Warning por fila con edad y acción global de reintento. |
| `local` | Evidencia del artefacto/instalación local; no implica observación remota. |
| `unknown` | Campo “Desconocido” y motivo de ausencia/invalidación. |

El refresh global se muestra aparte como `online/offline/unknown` + `idle/in-flight`; no sobrescribe la freshness de todas las filas.

### Estado de plan

| Estado | UI |
|---|---|
| `no-plan` | Solo selector policy y “Preparar plan” cuando procede. |
| `planning` | Botón ocupado; inventario original visible. |
| `preview-ready` | Snapshot completo y “Revisar y aplicar”. |
| `invalidated` | Banner “El estado cambió; prepara un plan nuevo”; apply retirado. |
| `expired` | Mismo tratamiento que invalidated, con motivo temporal. |

### Estado de handoff

| `handoffState` | Mensaje/controles |
|---|---|
| `not-started` | No existe operación mutante. |
| `launching` | “Iniciando helper seguro”; no retry/doble click. |
| `acknowledged` | Operación durable; se muestra `enginePhase`. |
| `uncertain` | “Verificando si la operación comenzó”; bloquea retry y reconcilia journal/`updateOperationLock`. |
| `failed` | Fallo pre-claim demostrado; exige plan/grant nuevos. |

### Fase del engine

| `enginePhase` | Mensaje/controles |
|---|---|
| `accepted` | “Operación aceptada por el helper”; cancel según capability. |
| `acquiring-lock` | “Esperando acceso exclusivo”; sin reintentos paralelos. |
| `revalidating` | “Comprobando que el plan sigue siendo válido”. |
| `preparing` / `building` / `verifying` | Fase textual y progreso solo si es verificable. |
| `awaiting-launcher-exit` | Aviso de cierre conforme a policy confirmada. |
| `activating` / `persisting` | Cancelación no disponible; no cerrar helper manualmente. |
| `rolling-back` / `recovering` | Warning claro y seguimiento; sin apply nuevo. |
| `succeeded` / `failed` / `recovery-required` | Mostrar `runnerOutcome` y evidencia observada. |

`cancellationState` (`available/requested/accepted/rejected`) se muestra aparte de `enginePhase`. `relaunchState` (`not-requested/pending/succeeded/failed`) aparece solo después del outcome; `failed` se comunica con código `relaunch_failed` y no reescribe `runnerOutcome`.

## Matriz de transiciones y acciones

| Desde | Evento | Hacia | Side effect permitido |
|---|---|---|---|
| bootstrap | status local | inventario visible | Ninguno. |
| inventario visible | refresh inicia | refresh `in-flight` | Fetch canónico acotado. |
| refresh activo | éxito | `state=relación` + evidencias actualizadas | Ref/cache permitida; nunca instalación. |
| refresh activo | timeout/unavailable/conectividad | `state=offline` + `lastKnownRelation` | Cancelar proceso y conservar evidencias por identidad. |
| refresh activo | cancel manual | state/conectividad previos + `lastOutcome=cancelled` | Cancelar proceso; ninguna mutación adicional. |
| refresh activo | provenance inválida | `state=unknown` + apply bloqueado | Error tipado; no remoto alternativo. |
| `update-available` | preparar plan | planning | Ninguno sobre instalación/journal. |
| planning | plan válido | preview-ready | Reservar operationId y emitir plan/grant temporal. |
| preview-ready | editar/refresh/expiry | invalidated | Revocar/descartar grant. |
| preview-ready | confirmar | handoff `launching` | Consumir grant y lanzar helper. |
| handoff `launching` | spawn falla antes de claim | `failed` | No cerrar; plan/grant nuevos. |
| handoff `launching` | ACK timeout/transport loss | `uncertain` | Bloquear retry y reconciliar operationId/journal/`updateOperationLock`. |
| handoff `launching/uncertain` | ACK durable o journal hallado | `acknowledged` | Adjuntar a la única operación existente. |
| cancellation `available` | confirmar cancel | `requested` | Señal cooperativa. |
| operación irreversible | intentar cancel | `rejected` | Error `cancellation_not_safe`; seguimiento continúa. |
| awaiting exit | policy confirmada | Launcher cerrado | Helper espera PID dentro de cota; luego engine. |
| nueva instancia | journal activo | operación reanudada | Lectura/reconexión con sesión nueva. |
| journal recovery | preview con policy | recovery preview-ready | Ninguna mutación. |
| recovery preview-ready | confirmar con grant | recovering | Helper/runner compartido. |

## Errores y respuesta de UI

| Clase tipada | Comportamiento |
|---|---|
| `unauthenticated` / `origin_rejected` | No reintento mutante; renovar bootstrap/sesión o mostrar rechazo seguro. |
| `invalid_request` / `body_too_large` | Error de cliente genérico; no invocar runner. |
| `offline` / `canonical_remote_unavailable` | Conservar cache y ofrecer reintento manual. |
| `invalid_provenance` | Bloquear plan/apply y mostrar evidencia no confiable. |
| `engine_unavailable` | Mantener inventario read-only y explicar dependencia no disponible. |
| `plan_stale` | Invalidar plan/grant y volver a planning. |
| `confirmation_missing/expired/replayed/mismatch` | Cero side effects; solicitar nuevo plan/confirmación. |
| `busy` / `lock_conflict` | Mostrar operación existente o reintento seguro; no loop automático. |
| `precondition_changed` | Mostrar drift y obligar a check/plan nuevos. |
| `helper_spawn_failed` | Fallo pre-claim demostrado: mantener abierto y exigir plan/grant nuevos. |
| `handoff_uncertain` | Bloquear retry/cierre y reconciliar operationId/journal/`updateOperationLock` hasta outcome concluyente. |
| `helper_unreachable` | Si ya hubo claim, tratar como handoff incierto u operación activa; nunca asumir que no empezó. |
| `cancellation_not_safe` | Retirar cancel, mantener progreso y explicar boundary. |
| `recovery_required` | Bloquear apply y priorizar Recovery. |
| `relaunch_failed` | Mostrar instalación observada y arranque manual. |
| `internal_error` | Mensaje sanitizado + requestId; sin stack ni datos sensibles. |

## Accesibilidad

- La sección usa headings, definición/tabla accesible para versiones y botones con nombres completos.
- Badges incluyen texto e icono; color nunca es el único portador de estado.
- El diálogo tiene nombre/descripción, focus trap, orden lógico, cierre por Escape solo antes de enviar la mutación y retorno de foco.
- Cambios de inventario se anuncian en `aria-live="polite"`; errores que requieren intervención usan `role="alert"`. Eventos de progreso se agrupan por fase para no saturar lectores.
- La barra o spinner incluye texto; con `prefers-reduced-motion` se elimina animación no esencial.
- Botones deshabilitados exponen un motivo cercano y legible, no solo `disabled` sin contexto.
- Targets y commits se pueden copiar/leer completos con teclado; abreviaturas visuales conservan accessible name completo.
- La cancelación y la acción destructiva no dependen de posición o color y tienen etiquetas distintas.

## No-mutación y seguridad visibles

- La UI no usa GET para mutaciones ni construye comandos Git.
- Tokens nunca aparecen en DOM persistente, URL, localStorage, telemetría o mensajes.
- Status, check y plan no se etiquetan como instalación y no muestran progreso de apply.
- Cerrar la pestaña no se representa como cancelación.
- El frontend no puede elegir remote, ref, checkout, home, prefix, executable o shim.
- Toda mutación pasa por sesión + Host/Origin + schema/límites + grant + lock/revalidación del runner.
- Un evento frontend no autoriza nada; el servidor valida snapshot y consume el grant.
- No se ofrece “forzar”, “ignorar dirty”, “resetear” ni “hacer stash”.

## Cascadas y comportamiento reactivo

- Un snapshot de inventario más nuevo invalida plan/grant y recalcula capabilities.
- Cambiar policy de restart invalida plan/grant; no se modifica el plan existente.
- Una operación activa deshabilita check/plan/apply/recover incompatibles en todas las vistas.
- `recovery-required` tiene prioridad visual y funcional sobre `update-available`.
- Un cambio de proyecto no reinicia refresh ni pierde operación porque el scope es global.
- Una sesión nueva descarta tokens cliente, pero reconcilia journal global.
- Responses/events stale se ignoran por requestId/sequence; gaps fuerzan snapshot.
- Al quedar offline, `state=offline`; cada identidad conserva evidencia/edad propia y `lastKnownRelation` se muestra aparte. Al recuperar conexión no se autoaplica nada.
- El outcome terminal fuerza una nueva lectura de installedVersion; la UI nunca asume que target solicitado equivale a instalación observada.
- La navegación o recarga no duplica helper ni operación; el backend devuelve el `activeOperation` existente.
