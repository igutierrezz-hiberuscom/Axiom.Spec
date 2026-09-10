# 01 Requisitos

## Objetivo del documento

Definir el contrato funcional, de seguridad y no-mutación para integrar en el Launcher las superficies de self-update de ACC-077/078. Los términos **DEBE**, **NO DEBE** y **PUEDE** son normativos. Este contrato describe capacidad por implementar; no acredita que exista actualmente.

## Requisitos del incremento

### Dependencias y secuencia

- **REQ-DEP-01 — Posición global.** Este incremento DEBE ejecutarse como 4/5 después de `INC-20260909-r13-self-update-release-identity`, `INC-20260909-r13-self-update-state-cli-contract` e `INC-20260909-r13-self-update-transactional-updater`.
- **REQ-DEP-02 — Gate de entrada.** Antes de implementar, el builder DEBE comprobar que los tres contratos anteriores están disponibles, son compatibles y tienen evidencia aceptada. La ausencia de API headless, `UpdatePlan`, journal, progreso, cancelación o recovery produce `STOP`.
- **REQ-DEP-03 — Handoff.** El resultado DEBE exponer la integración y evidencia focal que necesita `INC-20260909-r13-self-update-release-certification`; no DEBE adelantar release CI ni documentación final.
- **REQ-DEP-04 — Autoridad y ownership.** Identidad/release pertenecen al incremento 1; manifest y `stateWriteLock` al 2; `UpdatePlan`, `updateOperationLock`, engine, journal y rollback al 3. R13-3 entrega runner headless, eventos tipados y cancelación; R13-4 posee el helper externo, claim/ACK/fencing, espera del PID, reattach y relaunch. Launcher y HTTP solo adaptan los contratos y no adquieren locks directamente.

### Bootstrap, descubrimiento e inventario

- **REQ-DIS-01 — Primer render local.** El Launcher DEBE quedar disponible y renderizar `status` desde evidencia local sin esperar una operación de red.
- **REQ-DIS-02 — Refresh de arranque.** Después del bootstrap, el servidor DEBE programar un `git fetch` asíncrono del remoto canónico validado por el incremento 1. La tarea será best-effort, cancelable en shutdown, con deadline por defecto de 10 segundos y máximo configurable de 30 segundos.
- **REQ-DIS-03 — No bloqueo.** Un fetch lento, fallido, cancelado o sin red NO DEBE retrasar la respuesta de bootstrap, impedir el uso del Launcher ni convertir el proceso en fallido.
- **REQ-DIS-04 — Límite de mutación.** El refresh solo PUEDE actualizar refs remotas/evidencia de descubrimiento y la cache permitida por los contratos previos. NO DEBE ejecutar `checkout`, `merge`, `pull`, `rebase`, `reset`, `stash`, cambio de branch, activación, instalación ni escritura de un outcome transaccional.
- **REQ-DIS-05 — Remoto único.** El fetch DEBE dirigirse exclusivamente al remoto/ref canónicos ya validados; no admite URL, remote name, ref ni credenciales proporcionados por el browser.
- **REQ-DIS-06 — Coalescing.** Solo PUEDE existir un refresh activo por instalación. Arranque y `check` concurrentes DEBEN compartir o coalescer la operación, no lanzar fetches ilimitados.
- **REQ-DIS-07 — Inventario separado.** La vista DEBE mostrar `publishedVersion`, `downloadedVersion` e `installedVersion` por separado. Valores ausentes se muestran como desconocidos, nunca como `0.0.0` ni inferidos de otra identidad.
- **REQ-DIS-08 — Evidencia.** Cada identidad DEBE incluir, cuando exista, freshness propia (`live`, `cached`, `stale` o `local`), `observedAt`, antigüedad y provenance de su autoridad (`canonical-remote`, `cache`, `local-artifact` o `active-installation`) con remoto/ref/commit solo cuando apliquen. URLs y diagnósticos se sanitizan.
- **REQ-DIS-09 — Estados tipados.** La vista DEBE consumir y presentar sin reinterpretación los estados `updated`, `update-available`, `checkout-behind`, `ahead`, `installed-misaligned`, `unknown` y `offline`.
- **REQ-DIS-10 — Resultado honesto del refresh.** Timeout, remoto unavailable o fallo de conectividad conservan evidencia previa por identidad, fijan `state=offline` y mantienen relación cacheada solo como `lastKnownRelation`. Cancelación manual conserva `state`/conectividad previos y solo registra `lastOutcome=cancelled`. Provenance inválida con conectividad fija relación `unknown`, `lastOutcome=invalid-provenance` y bloquea apply. Ningún caso se presenta como live ni concluye falsamente que no hay actualizaciones.

### Operaciones compartidas

- **REQ-RUN-01 — Un runner.** `status`, `check`, `plan`, `apply` y `recover` DEBEN invocar la misma API headless que usa la CLI. No se permiten implementaciones paralelas en handlers HTTP, scripts del frontend o comandos ad hoc.
- **REQ-RUN-02 — Semántica común.** Inputs normalizados, estado tipado, `UpdatePlan`, taxonomía de errores, lock, outcomes y recovery DEBEN mantener paridad semántica CLI/Launcher.
- **REQ-RUN-03 — Status.** `status` es una lectura local y NO DEBE iniciar fetch, crear plan, escribir manifest/journal, lanzar helper ni cambiar instalación.
- **REQ-RUN-04 — Check.** `check` PUEDE solicitar el refresh canónico acotado, pero NO DEBE crear/aplicar un plan ni modificar worktree activo, shim, instalación, manifest transaccional o journal de apply.
- **REQ-RUN-05 — Plan.** `plan` DEBE ser un preview read-only sellado por el runner. Incluye identidad actual/target, provenance, precondiciones, fases, riesgos, política de cierre/reinicio, digest estable y `operationId` reservado; no descarga dependencias, construye, activa ni abre una transacción.
- **REQ-RUN-06 — Plan inmutable.** `apply` DEBE consumir exactamente el plan previsualizado. Si cambian identidad, ref, commit, instalación, precondición o política de reinicio, el plan queda stale y se exige un nuevo preview.
- **REQ-RUN-07 — Engine unavailable.** Si el incremento 3 no está disponible, las operaciones mutantes fallan con `engine_unavailable`; la UI no debe emularlas.

### Confirmación y mutación

- **REQ-CON-01 — Nunca automática.** Detectar `update-available`, completar un fetch, abrir/cerrar el Launcher o vencer un timer NO DEBE ejecutar `apply` ni `recover`.
- **REQ-CON-02 — Preview obligatorio.** `apply` solo se habilita después de un `plan` válido presentado al operador.
- **REQ-CON-03 — Confirmación explícita.** La confirmación DEBE nombrar versión instalada, versión target, commit/ref, efectos, política de cierre/reinicio y posibilidad de cancelación. El botón final debe describir la acción, no usar un “Aceptar” genérico.
- **REQ-CON-04 — Binding.** El grant DEBE reutilizar el mecanismo R-13.4: aleatorio, TTL 120 s, single-use, comparación segura y binding a sesión, identidad global de instalación, `operationId`, acción, digest del plan, target y política `close-only` o `close-and-restart`.
- **REQ-CON-05 — Consumo.** El servidor DEBE consumir el grant atómicamente antes de lanzar el helper o producir cualquier side effect. Missing, expiry, replay o mismatch fallan cerrado.
- **REQ-CON-06 — Invalidation.** Editar cualquier opción, recibir un estado/provenance más nuevo, cambiar de sesión, recargar o cancelar invalida plan y confirmación en cliente; el servidor sigue siendo la autoridad final.
- **REQ-CON-07 — Recover protegido y bifásico.** `recover` DEBE separar preview de ejecución: primero presenta operación/journal y acción `resume` o `rollback`, exige elegir sin default `close-only` o `close-and-restart`, y genera un recovery plan/grant propio; solo una segunda petición confirmada puede mutar. No reutiliza token de apply ni mezcla preview/execute en un schema ambiguo.

### Helper, progreso, cancelación y restart

- **REQ-HLP-01 — Proceso externo.** `apply` y `recover` DEBEN ejecutarse en un helper externo al proceso Launcher. El helper no importa ni actualiza módulos desde el proceso que los tiene cargados.
- **REQ-HLP-02 — ID reservado y claim idempotente.** El plan DEBE reservar un `operationId` impredecible ligado al grant. Al confirmar, el helper reclama atómicamente ese ID y materializa journal/plan digest antes del ACK. Repetir el mismo ID devuelve la operación existente; nunca inicia una segunda.
- **REQ-HLP-02A — ACK incierto y fencing.** Si spawn falla antes de crear helper puede informarse `helper_spawn_failed`. Si ACK se pierde, vence o llega tarde, el estado es `handoff_uncertain`: se bloquean retries y se reconcilian `operationId`, journal, `updateOperationLock` y proceso. Solo ausencia demostrada más helper terminado permite declarar fallo pre-handoff; cualquier retry exige plan, `operationId` y grant nuevos. Un grant consumido jamás se reutiliza.
- **REQ-HLP-03 — Handoff durable.** La operación solo pasa a aceptada tras ACK que demuestra claim/journal durable. El helper DEBE delegar al runner del incremento 3 el `updateOperationLock` y la revalidación bajo ese lock. El `stateWriteLock` del incremento 2 solo protege persistencia/migración de estado. Si el incremento 3 no aporta claim idempotente/reattach, el gate es `STOP`; Launcher no lo implementa.
- **REQ-HLP-04 — Progreso acotado.** La UI recibe fases tipadas y mensajes sanitizados, no stdout/stderr bruto. Cada evento JSON tiene máximo 4 KiB; un evento mayor se sustituye por `progress-truncated` y fuerza snapshot. El historial se limita a 100 eventos o 64 KiB, lo que ocurra primero; cada evento se numera. Si se usa SSE, hereda heartbeat, máximo de clientes y backpressure de R-13.4.
- **REQ-HLP-04A — Deadlines.** Los defaults/máximos server-side son: handshake 5/10 s, drenaje HTTP/SSE 3/5 s, espera del PID 15/30 s y ACK de relaunch 10/30 s. El browser no puede ampliarlos. Timeout esperando PID aborta antes de activar; timeout de relaunch no cambia el outcome de instalación.
- **REQ-HLP-05 — Cancelación segura.** La cancelación solo se ofrece cuando el runner indique `cancellable: true`. Antes de activación solicita cancelación cooperativa y limpieza de staging; desde el boundary irreversible se rechaza con `cancellation_not_safe` y la UI mantiene seguimiento.
- **REQ-HLP-06 — Cerrar pestaña.** Cerrar o recargar el browser NO cancela una operación ya aceptada o incierta. El estado se recupera por `operationId`/journal al reconectar.
- **REQ-HLP-07 — Cierre explícito.** El plan obliga a elegir `close-only` o `close-and-restart`. El Launcher solo cierra después de ACK durable y de la confirmación vinculada a esa opción.
- **REQ-HLP-08 — Activación sin módulos cargados.** El helper DEBE esperar la terminación del PID Launcher antes del boundary de activación. No se permite reemplazar in-process los módulos activos.
- **REQ-HLP-09 — Reinicio explícito.** Solo `close-and-restart` autoriza al helper a relanzar. `close-only` termina sin relanzar y comunica cómo iniciar manualmente. Un fallo de relaunch no convierte una instalación verificada en rollback automático; se muestra como eje operativo separado.
- **REQ-HLP-10 — Reanudación.** Un Launcher nuevo DEBE reconciliar el journal y cualquier handoff incierto antes de ofrecer otro apply, mostrar progreso/outcome y reanudar observación sin reutilizar sesión o token anteriores.
- **REQ-HLP-11 — Recovery first.** Si el runner informa `recovery-required`, plan/apply normal quedan bloqueados y se prioriza `recover`. Nunca se borra ni ignora el journal desde la UI.

### Control plane, seguridad y límites

- **REQ-SEC-01 — Baseline R-13.4.** Toda ruta hereda bind loopback, cookie de sesión por proceso, Host exacto, Origin same-origin, ausencia de CORS, headers restrictivos, errores 5xx sanitizados y `Cache-Control: no-store`.
- **REQ-SEC-02 — Auth completa.** API, polling/SSE de progreso y recuperación requieren sesión válida. No se ponen token, `operationId` sensible, rutas locales ni credenciales en query strings.
- **REQ-SEC-03 — Schemas cerrados.** Cada route kind declara método, content type, campos admitidos y tipos. Campos desconocidos, métodos incorrectos o payloads incompatibles fallan antes de invocar runner/helper.
- **REQ-SEC-04 — Límites.** Mutaciones JSON conservan máximo 256 KiB y timeout de lectura 5 s. Fetch usa default/máximo 10/30 s; handshake 5/10 s; drenaje 3/5 s; espera del PID 15/30 s; relaunch ACK 10/30 s. Son cotas server-side no ampliables por el browser; toda espera se cancela/recolecta en shutdown.
- **REQ-SEC-05 — Diagnóstico mínimo.** Mensajes UI no exponen stack, comandos completos con secretos, contenido arbitrario del remoto ni paths innecesarios. Los detalles locales se redactan y acotan.
- **REQ-SEC-06 — Scope global.** El servidor deriva la instalación global desde contexto confiable. El browser no elige checkout, home, prefix, shim, executable ni remote.

### Calidad, observabilidad y pruebas

- **REQ-QLT-01 — Estado reconciliable.** Las respuestas incluyen identificadores/sequence suficientes para ignorar resultados stale y retomar operaciones sin duplicarlas.
- **REQ-QLT-02 — Accesibilidad.** La superficie es operable con teclado, mantiene foco, no depende solo de color, anuncia cambios relevantes con live regions acotadas y respeta reduced motion.
- **REQ-QLT-03 — No red externa en tests.** HTTP, fetch y helper se prueban con loopback, remoto Git bare, repos, homes y prefixes temporales. Cualquier intento de red externa debe fallar el test.
- **REQ-QLT-04 — Evidencia de no-mutación.** Las pruebas de bootstrap/status/check/plan y fallos de seguridad comparan worktree, branch/HEAD, shim, manifest y journal antes/después.
- **REQ-QLT-05 — Evidencia de paridad.** Los tests prueban que CLI y Launcher llaman el mismo runner/fixtures y producen estados/outcomes equivalentes para inputs equivalentes.
- **REQ-QLT-06 — Helper real y fault fixtures.** Los dobles ejecutables pueden inyectar ACK perdido/tardío, crash y timeout, pero al menos un E2E DEBE invocar el entry point de producción del helper con runner/filesystem temporales.
- **REQ-QLT-07 — Accesibilidad reproducible.** La validación DEBE incluir recorrido DOM/browser automatizado disponible o protocolo manual reproducible con herramienta/versión, cubriendo teclado, focus trap/restore, live regions, reduced motion y señalización no basada solo en color.

## Reglas de negocio relevantes

1. Hay una única instalación global user-level observada; un proyecto abierto no crea otra autoridad de self-update.
2. `publishedVersion`, `downloadedVersion`, `installedVersion` y runtime project-scoped son identidades distintas, cada una con freshness/provenance propias, y no se rellenan por inferencia.
3. Solo una release demostrada por el remoto canónico puede ser target.
4. Offline es el estado primario cuando falla la consulta canónica; una relación cacheada se conserva aparte como `lastKnownRelation` y no equivale a evidencia live.
5. La cache aporta evidencia y antigüedad por identidad; nunca sustituye provenance live sin indicarlo.
6. Un handoff incierto bloquea retries hasta reconciliar `operationId`, journal, lock y proceso; un token consumido no se reutiliza.
7. Preview y apply comparten exactamente el mismo plan y `operationId`. Cualquier drift exige replan.
8. Un grant autoriza una sola operación y no reemplaza auth, same-origin, `updateOperationLock` ni revalidación.
9. Apply/recover se serializan mediante el `updateOperationLock` del incremento 3; el `stateWriteLock` del 2 protege únicamente lecturas/escrituras/migración de estado.
10. La cancelación no puede atravesar una fase declarada irreversible.
11. `recovery-required` tiene prioridad sobre una actualización nueva.
12. Recovery también es bifásico, exige policy de restart explícita y grant propio.
13. La instalación realmente observada después de activar prevalece sobre el target solicitado.
14. Handoff, fase de engine, cancelación, relaunch y outcome son ejes separados; la UI no los colapsa ni transforma optimistamente.

## Fuera de alcance funcional

- Implementar descubrimiento/identidad, manifest v2 o motor transaccional.
- Definir comandos Git, layout de staging, algoritmo de rollback o formato de journal alternativos.
- Actualizar automáticamente o en background sin preview y confirmación.
- Soportar administración remota, múltiples instalaciones elegidas desde UI o remotos arbitrarios.
- Emitir logs de build completos por el browser.
- Certificar release CI, packaging, distribución npm o documentación final del operador.
- Cambiar lifecycle, metadata, receipts, índices o archivos canónicos fuera de este incremento.
