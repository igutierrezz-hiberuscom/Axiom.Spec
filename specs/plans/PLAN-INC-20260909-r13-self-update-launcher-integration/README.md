# Plan de integración de self-update en el Launcher

> **Código**: PLAN-INC-20260909-r13-self-update-launcher-integration
> **Estado**: draft
> **Artefacto origen**: INC-20260909-r13-self-update-launcher-integration
> **Versión de spec**: v1
> **Versión de plan**: p1

## Resumen ejecutivo

Este plan implementará, cuando sus precondiciones den `GO`, el cuarto de cinco incrementos de self-update R-13.5. Añadirá al Launcher una superficie global para inventario, refresh, plan, apply y recover sobre los runners headless que deberán estar entregados y aceptados por los incrementos 1–3. Reutilizará el control plane R-13.4 y coordinará un helper externo, sin duplicar el motor Git ni ejecutar una actualización al arrancar.

La secuencia global obligatoria es:

1. release identity/discovery;
2. estado persistente y contrato CLI;
3. updater transaccional/recuperable;
4. **integración Launcher de este plan**;
5. certificación de release, E2E y documentación final.

El Launcher actual no contiene rutas, panel ni handlers de self-update. Los paths y símbolos de este documento son puntos de cambio **probables** basados en el runtime observado; se confirmarán contra el output real del incremento 3 antes de editar código.

## Objetivo técnico

Entregar un adaptador Launcher seguro y observable que:

- renderice status local inmediatamente y programe un fetch canónico best-effort/no bloqueante/acotado;
- muestre `publishedVersion`, `downloadedVersion`, `installedVersion`, freshness, provenance y estados tipados;
- exponga `status`, `check`, `plan`, `apply` y `recover` mediante el mismo runner que la CLI;
- aplique preview→confirmación single-use vinculada a plan/instalación/target/restart policy;
- lance un helper externo y no actualice módulos cargados por el proceso Launcher;
- presente progreso bounded, cancelación segura, cierre/reinicio explícito, reanudación y recovery;
- conserve auth, same-origin, schemas, límites, headers y no-mutación de R-13.4;
- produzca evidencia focal hermética para el incremento 5.

## Precondiciones y gates STOP/GO

| Gate | Evidencia requerida | GO | STOP |
|---|---|---|---|
| **G0 — secuencia** | Contratos y evidencia aceptada de incrementos 1, 2 y 3. | Las tres dependencias son consumibles y compatibles. | Falta una dependencia o existe cambio incompatible no resuelto. |
| **G1 — runner compartido** | API headless real para status/check/plan/apply/recover, errors, progress, cancel y journal. | La CLI **ya consume** la misma API sin wrapper mutante paralelo. | La CLI solo “podría” migrarse, usa runners legacy exclusivos o habría que copiar lógica al Launcher. |
| **G2 — helper** | Boundary externo con operationId reservado, claim/journal idempotente, ACK durable, fencing de ACK incierto, espera del PID, reattach y relaunch seguro aportados por 3. | Puede inyectarse/probarse sin shell ni engine in-process y duplicados devuelven la operación existente. | Falta claim/fencing/recovery o el Launcher tendría que inventar semántica del engine. |
| **G3 — baseline R-13.4** | Suites focales de auth/origin/schema/limits/grants/SSE. | Todas las garantías R-13.4 heredadas están verdes. Un fallo preexistente solo puede tolerarse si se demuestra ajeno a estas garantías, tiene owner/evidencia y no afecta el candidate. | Cualquier suite/garantía heredada relevante está roja o sin ejecutar. |
| **G4 — diseño de ruta** | Schemas cerrados, binding de grants y threat review. | Cada mutación tiene método, schema, límites y grant explícitos. | Ruta mutante evita middleware, acepta remote/path del browser o carece de binding. |
| **G5 — candidate** | Tests focales, build/typecheck, no-mutación, review independiente. | Sin hallazgos bloqueantes y con evidence package trazable. | Fallo nuevo, red externa, duplicación de engine o evidencia incompleta. |
| **G6 — handoff a 5** | Matriz y limitaciones documentadas. | Incremento 5 puede certificar sin reinterpretar resultados. | Se intenta declarar release/cierre final desde este incremento. |

Un `STOP` se resuelve en el incremento propietario; no se parchea localmente con una implementación alternativa. Este plan no cambia status ni ejecuta transiciones Core.

## Alcance incluido

### WP0 — Baseline y freeze de interfaces

1. Leer outputs vigentes de los incrementos 1–3 y registrar versiones de schema/API, no copiar su prose.
2. Confirmar los entry points que usa la CLI y construir una tabla de paridad operación→runner→result/error.
3. Ejecutar baseline focal R-13.4 y self-update. Las suites que prueban auth, Origin, schemas, límites, grants y SSE deben estar verdes; un fallo preexistente solo es no bloqueante con evidencia de que es ajeno, owner explícito y cero impacto sobre el candidate.
4. Verificar que el checkout de trabajo no contiene cambios ajenos en los paths que se editarán.
5. Emitir decisión `GO/STOP` para G0–G3 antes de diseñar adaptadores definitivos.

### WP1 — Adaptador headless y estado del Launcher

Paths probables:

- `apps/cli/src/commands/app-self-update.ts` — módulo nuevo de adaptación; nombres tentativos `apiGetSelfUpdateStatus`, `apiCheckSelfUpdate`, `apiPlanSelfUpdate`, `apiApplySelfUpdate`, `apiRecoverSelfUpdate`, `apiCancelSelfUpdate`, `apiGetSelfUpdateOperation`.
- `apps/cli/src/commands/app.ts` — scheduling posterior al bootstrap, cancelación en shutdown y coordinación de cierre/relaunch; símbolos existentes relevantes `registerApp`, `appStart`, `openBrowser`.
- `apps/cli/src/commands/self-update.ts` — wrapper CLI que debe consumir el mismo runner; no será fuente de lógica copiada.
- Paquete headless entregado por el incremento 3 — ubicación por confirmar; `packages/user-workspace/src/self-update.ts` es una referencia legacy posible, no una decisión de ownership.

Tareas:

1. Crear una capa delgada que transforme request/context en inputs del runner y sus resultados en view models/envelopes.
2. Derivar installation/home/prefix/remote/ref server-side; no aceptar esos valores del browser.
3. Programar después del bootstrap un refresh coalesced con deadline por defecto 10 s, máximo 30 s y abort en shutdown.
4. Separar status local del refresh. La primera respuesta nunca espera red.
5. Mapear por separado `publishedVersion`, `downloadedVersion` e `installedVersion`, cada una con freshness/provenance propias; mantener `refresh` y `lastKnownRelation` en ejes distintos y no rederivar SemVer, target, outcome o recovery.
6. Reconciliar `activeOperation`, handoff incierto y journal al arrancar antes de habilitar apply.

### WP2 — Rutas HTTP y control plane

Path principal probable: `apps/cli/src/commands/app-api.ts`.

Símbolos existentes a extender sin bypass:

- `RouteMatch` y `matchRoute`;
- `CLOSED_BODY_FIELDS` y `BODY_FIELD_TYPES`;
- `readJsonBody`, `handleApiRequest`, `responseHeaders`/`sendJson`;
- `createAppServer` y estado por proceso.

Contrato de rutas candidato, sujeto a G4:

| Método/ruta probable | Operación | Mutación permitida |
|---|---|---|
| `GET /api/self-update/status` | status local + operación reconciliada | Ninguna. |
| `POST /api/self-update/check` | refresh canónico coalesced | Solo refs/cache/evidencia autorizadas. |
| `POST /api/self-update/plan` | preview + grant | Ninguna sobre instalación/journal. |
| `POST /api/self-update/apply` | consumir grant + helper handoff | Sí, exclusivamente mediante helper/runner. |
| `POST /api/self-update/recover/plan` | preview de resume/rollback + policy explícita + grant | Ninguna sobre instalación/journal. |
| `POST /api/self-update/recover` | consumir recovery grant y ejecutar acción previsualizada | Sí, exclusivamente mediante helper/runner. |
| `POST /api/self-update/operations/:id/cancel` | cancel cooperativo | Solo si runner lo declara seguro. |
| `GET /api/self-update/operations/:id` | snapshot/progreso | Ninguna. |
| Stream global de operación, ruta por confirmar | eventos bounded | Ninguna; auth/origin y caps R-13.4. |

Los nombres son candidatos, pero preview y ejecución de recovery deben seguir siendo dos requests/schemas inequívocos; no se admite un único body ambiguo que pueda mutar durante preview. La ruta definitiva se confirma en G4 tras comprobar colisiones y patrones reales.

Implementación de seguridad:

1. Todas las rutas pasan por cookie de sesión, Host exacto y same-origin; no CORS/preflight.
2. POST exige JSON, body máximo 256 KiB y read timeout 5 s.
3. Tablas de schemas cerrados rechazan campos/tipos desconocidos antes del runner.
4. `launcher-security.ts:issuePreviewGrant`/`consumePreviewGrant` se reutilizan o generalizan sin segundo token store. El binding añade installation, `operationId`, acción, plan digest, target y restart policy; TTL 120 s y single-use se conservan.
5. Apply/recover consume el grant antes de spawn. Recovery preview y execute tienen schemas/rutas separados y grant distinto de apply.
6. Progreso por polling/SSE exige auth y no recibe token en URL. Si usa SSE, conserva heartbeat 15 s, máximo 8 clientes, cola 64 KiB y cleanup/backpressure.
7. Cada evento serializado se limita a 4 KiB; el oversized se sustituye por `progress-truncated` y fuerza snapshot sin encolar contenido no confiable.
8. Errores 5xx, child output y provenance se sanitizan; API usa `no-store` y headers R-13.4.

### WP3 — Helper, handoff y lifecycle

El motor y formato de journal pertenecen al incremento 3. Este WP integra su boundary:

1. Reservar `operationId` impredecible dentro del plan y ligarlo al grant; el browser no lo elige ni puede cambiarlo.
2. Spawn con executable/path absolutos validados, array de argumentos, `shell: false`, entorno mínimo y sin datos controlados por browser.
3. El helper reclama atómicamente `operationId` y materializa journal/digest antes del ACK. Un claim repetido devuelve snapshot existente, no crea otro updater.
4. Handshake default 5 s/máximo 10 s. Spawn fallido antes de claim exige plan/grant nuevos. Timeout o canal perdido después de posible claim produce `handoff_uncertain`, bloquea retries y reconcilia journal/`updateOperationLock`/proceso; ACK tardío se adjunta al mismo ID.
5. Solo ausencia demostrada de journal/claim más helper terminado permite declarar fallo pre-handoff. El grant ya consumido nunca revive.
6. Tras ACK durable, seguir ejes separados `enginePhase`, `cancellationState`, `relaunchState` y `runnerOutcome`; historial máximo 100 eventos o 64 KiB, evento máximo 4 KiB.
7. Propagar cancel solo con `cancellable: true`; no matar el proceso ni declarar cancelado sin ACK y reconciliación.
8. En `awaiting-launcher-exit`, drenar HTTP/SSE 3 s default/5 s máximo. El helper espera el PID 15 s default/30 s máximo; timeout aborta antes de activación y deja journal/outcome tipado.
9. Respetar policy ligada al grant: `close-only` o `close-and-restart`. Recovery obliga a elegirla también sin default.
10. Para relaunch, usar entry point observado/validado, nunca strings del browser. Esperar ACK 10 s default/30 s máximo; timeout fija `relaunchState=failed` sin reescribir outcome de instalación.
11. En proceso nuevo, crear sesión nueva y reconciliar journal/handoff. Ante crash conservar journal y mostrar `recovery-required`; nunca borrar estado ni auto-recover.

### WP4 — Static frontend y accesibilidad

Paths probables actuales:

- `apps/cli/static/launcher/index.html` — punto de montaje/nav de sección global;
- `apps/cli/static/launcher/launcher.js` — estado reactivo, request sequencing, invalidación de plan/confirmación y render;
- `apps/cli/static/launcher/panels.js` — panel/controles de self-update si encaja en su composición;
- `apps/cli/static/launcher/transport.js` — wrappers API, auth por cookie y stream/poll;
- `apps/cli/static/launcher/launcher.css` — estados, diálogo, focus y reduced motion.

Símbolos existentes a respetar: `invalidateConfirmation`, `api`, `withConfirmation`, `openEventStream` y manejo monotónico de requests; se confirmarán en el candidate.

Tareas:

1. Implementar sección global con tres evidencias por identidad, estado/lastKnownRelation, refresh, actions, plan, handoff, operación y recovery.
2. Modelar por separado relation/offline, refresh, plan, handoff, engine phase, cancellation, relaunch y runner outcome.
3. Exigir restart policy sin default antes de plan, también para recovery.
4. Invalidar plan/grant ante edición, refresh, drift, expiry, reload o sesión nueva.
5. Usar textos/acciones explícitos: nunca “Aceptar” genérico ni apply directo desde check.
6. Implementar cancelación segura, no-cancel tras boundary y cierre de pestaña sin cancel implícito.
7. Reconciliar tras restart y priorizar recovery.
8. Cumplir teclado, focus trap/restore, `aria-live` acotado, labels no dependientes de color y reduced motion.
9. Evitar DOM/log growth: historial y mensajes bounded.

### WP5 — Pruebas herméticas

#### HTTP/control plane

Crear o extender suites probables:

- `apps/cli/tests/app-self-update.test.ts`;
- `apps/cli/tests/app.test.ts`;
- `apps/cli/tests/launcher-security.test.ts`;
- `apps/cli/tests/r13-acc-078-launcher-matrix.test.ts` o nombre equivalente;
- `apps/cli/tests/launcher-front-no-vscode.test.ts` / suite frontend vigente.

Matriz mínima:

- bootstrap responde mientras fetch está pendiente;
- fetch success/timeout/cancel/offline/coalescing y remoto único;
- status/check/plan no mutan instalación;
- auth/Host/Origin/método/content-type/schema/body/timeout/headers;
- grant ausente/expiry/replay/session-installation-action-payload mismatch y carrera de dos applies;
- stream/poll auth, Host/Origin, cliente lento y reconexión;
- state/lastKnownRelation, refresh y freshness/provenance independiente por identidad;
- recovery preview/execute separados, policy sin default y grant ligado a journal/action/operationId;
- evento >4 KiB, cola/clientes/historial excedidos, gaps y snapshot de reconciliación;
- engine unavailable, plan stale, busy/`lock_conflict` del updateOperationLock, handoff uncertain, recovery-required y errores sanitizados.

#### Helper y filesystem

Usar por test:

- remoto Git bare local con dos releases/refs y commits deterministas;
- clone/worktree temporal;
- `HOME`, npm prefix, installation root, cache y journal temporales;
- entry point **real de producción** del helper para al menos un E2E completo;
- fixture ejecutable separado solo para fault injection determinista (ACK perdido/tardío, crash, timeout) y clocks inyectables;
- puertos loopback efímeros;
- guard que rechace red no-loopback.

Casos:

- spawn pre-claim, handshake success, ACK perdido, ACK tardío, timeout y claim idempotente del mismo operationId;
- fake clock/process tests para cotas fetch 10/30 s, handshake 5/10 s, drain 3/5 s, PID wait 15/30 s y relaunch ACK 10/30 s;
- helper espera salida de PID antes de la activación observable;
- `close-only` no relanza y `close-and-restart` sí;
- browser close/reload no cancela ni duplica;
- cancel antes del boundary y rechazo después;
- helper/Launcher crash y reattach por journal;
- resume/rollback autorizado y recovery offline/no disponible;
- relaunch failed con instalación verificada;
- snapshots de HEAD/branch/worktree/shim/manifest/journal antes/después;
- ningún `checkout`, `merge`, `pull`, `reset` o `stash` en bootstrap/status/check/plan.

No se ejecutará red externa ni se dependerá del GitHub real, npm registry, home del usuario o instalación global de la máquina.

#### Accesibilidad reproducible

1. Confirmar en G0 qué harness DOM/browser existente puede ejecutar interacción real; no inventar una dependencia.
2. Automatizar, si el harness lo permite, tab order, apertura/cierre de diálogo, focus trap/restore, invalidación, live-region agrupada y media query `prefers-reduced-motion`.
3. Verificar que badge/warnings tienen texto/icono y no dependen solo de color mediante assertions DOM/CSS disponibles.
4. Si algún aspecto no puede automatizarse, ejecutar protocolo manual reproducible con browser/herramienta/versión, pasos, resultado y evidencia sanitizada. Una captura estática no acredita teclado o foco.
5. Todo fallo de accesibilidad funcional bloquea G5 igual que una regresión de seguridad.

### WP6 — Review, validación e integración

1. Ejecutar tests focales tras cada concern y luego gates completos.
2. Ejecutar review independiente por comportamiento contra ACC-077/078 y todos los criterios.
3. Revisar específicamente que no haya SemVer/Git/manifest/`stateWriteLock`/`updateOperationLock`/rollback duplicado en Launcher.
4. Revisar threat model: CSRF/same-origin, replay/race, path/remote injection, command injection, child lifecycle, log/token leakage, DoS por fetch/eventos y restart spoofing.
5. Clasificar cada fallo como nuevo o preexistente con evidencia. Auth/origin/schema/limits/grants/SSE heredados deben estar verdes; solo un fallo ajeno, con owner y cero impacto demostrado, puede no bloquear.
6. Preparar resumen de rutas, tests, resultados, limitaciones y plataformas para el incremento 5.
7. Solo tras implementación/validación, reconciliar conocimiento estable en las specs canónicas propietarias mediante el workflow permitido. No copiar historia ni tocar índices/metadata manualmente.

## Alcance excluido

- Resolver releases/semver, escribir manifest v2, implementar `stateWriteLock`/`updateOperationLock` o implementar Git/staging/activation/rollback/journal.
- Mantener los runners legacy como segunda implementación.
- Añadir auto-apply, auto-recover o apply al cerrar el Launcher.
- Permitir remoto/ref/path/home/prefix/executable elegidos desde HTTP.
- Relajar loopback, cookie, same-origin, schemas, body limits, headers o grants.
- Diseñar modo remoto, work item integrations o abstracciones enterprise.
- Crear release workflow/CI, packaging final, docs de usuario/troubleshooting finales o certificación multiplataforma de cierre; son 5/5.
- Modificar metadata, receipts, índices o status desde este plan.

## Roles impactados

- **builder (`axiom-code-builder`)**: implementa adaptadores, API, static UI, helper lifecycle y tests sin entrar al motor.
- **reviewer independiente**: valida intención, seguridad, no-mutación, no-duplicación, accesibilidad y paridad CLI/Launcher.
- **owner de incrementos 1–3**: resuelve incompatibilidades de contrato en su artefacto propietario; no se corrigen localmente.
- **owner del incremento 5**: consume evidence/handoff para release CI, E2E final y documentación.

El detalle operativo del builder está en [`role-builder.md`](./role-builder.md).

## Estrategia E2E

La prueba integra servidor HTTP real, static assets reales, runner/helper de producción detrás de seams y filesystem/Git temporales. El remoto canónico se emula con un bare repo local; la suite bloquea red externa. Se prueban dos releases, cache fresca/stale, instalación alineada/desalineada, operación happy path, no-op, fallos, cancel, restart y recovery.

La evidencia debe distinguir:

- effects permitidos de fetch sobre refs/cache;
- ausencia de cambios en bootstrap/status/plan y rechazos de seguridad;
- handoff antes de cierre;
- espera del PID antes de activación;
- versión realmente observada tras relaunch;
- journal y rollback/recovery aportados por el incremento 3.

La matriz final de release/plataformas pertenece al incremento 5; aquí se ejecutan plataformas disponibles y se registra explícitamente lo no ejecutado.

## Seguridad

- Preservar auth, Host/Origin exactos, no CORS, no-store y headers R-13.4.
- Usar schemas cerrados y límites antes de invocar runner/helper.
- Bind de grant a sesión + instalación + `operationId` + acción + plan digest + target + restart policy; consumo single-use pre-spawn.
- Derivar todos los paths/remotos/refs/entry points en servidor/runner.
- Spawn sin shell, args separados, cwd/env controlados, operationId reservado y claim idempotente.
- Aplicar cotas server-side: fetch 10/30 s, handshake 5/10 s, drain 3/5 s, PID wait 15/30 s y relaunch ACK 10/30 s.
- No exponer token, cookie, credentials, stdout/stderr bruto, paths sensibles ni operation secrets.
- Coalescer fetch, limitar evento a 4 KiB, clientes/historial/cola y abortar/recolectar en shutdown.
- Tratar contenido Git, mensajes del helper y outputs como datos no confiables antes de UI/log.
- Nunca ofrecer force/reset/stash/ignore preflight.

## Rollback del cambio de integración

El rollback de producto lo ejecuta el engine del incremento 3; el Launcher solo lo observa o solicita mediante recovery plan confirmado.

Para revertir una regresión de la **integración Launcher**:

1. impedir nuevas operaciones mediante capability/route coherente, sin dejar botones que apunten a rutas ausentes;
2. preservar CLI, runner, helper y journals para que una operación ya aceptada pueda terminar o recuperarse;
3. no matar helpers, borrar journal ni revertir una instalación sana por un fallo exclusivo de UI/relaunch;
4. retirar/revertir en conjunto panel, transport y handlers después de comprobar que no quedan operaciones activas;
5. validar que status CLI y recover siguen disponibles;
6. ejecutar de nuevo baseline R-13.4 y self-update.

Se incluirá una prueba de compatibilidad con journal creado antes del rollback de UI. Cualquier rollback que cambie engine o schema vuelve al incremento propietario.

## Riesgos y dependencias

| Riesgo | Mitigación / gate |
|---|---|
| Contrato 3 no disponible o cambia | G0–G2; STOP y corrección en dependencia. |
| Duplicación CLI/Launcher | Un runner headless, review de imports/comandos y tests de paridad. |
| Fetch bloquea arranque o queda huérfano | Scheduling posterior, deadline, coalescing, abort/cleanup y test con promesa pendiente. |
| CSRF/replay/race | R-13.4, binding ampliado, consumo atómico y carrera probada. |
| Actualizar módulos cargados | Helper externo, ack durable y espera explícita de PID. |
| ACK perdido/tardío crea duplicado | operationId reservado, claim idempotente, `handoff_uncertain`, fencing y reconciliación antes de retry. |
| Pérdida de operación al reiniciar | Journal/snapshot autoritativos y reattach con sesión nueva. |
| Progreso/log sin cota | Evento 4 KiB, historial 100/64 KiB, caps de transporte y backpressure. |
| Cancelación causa estado parcial | Capability del runner y rechazo tras boundary. |
| Relaunch se confunde con instalación | Outcomes separados y re-observación de installedVersion. |
| Tests tocan máquina/red real | Git/home/prefix/ports temporales y network guard. |
| UI oculta offline/unknown | Matriz de estados, provenance/edad y acciones honestas. |
| Rollback de UI abandona helper | Capability shutdown, preservación de journal y test de compatibilidad. |

## Cambios en contexto técnico

Durante implementación puede actualizarse el contexto local de este plan con mapa confirmado de rutas/símbolos, versiones de contratos 1–3, matriz test/evidencia, resultados sanitizados, findings y handoff a 5.

La integración estable se revisa tema por tema contra owners candidatos:

| Tema | Owner candidato |
|---|---|
| Inventario, estados y no-auto-apply | `specs/01_Requisitos_Funcionales.md` |
| Preview/apply/recover, fencing, cierre/restart | `specs/04_Flujos_SDD_y_Ciclo_de_Vida.md` |
| Superficie Launcher | `specs/05_Interfaces_Operativas.md` |
| Runner/helper compartidos | `specs/06_Integraciones_y_Capacidades.md` |
| Auth/origin/grants/limits/child security | `specs/07_Gobierno_y_Seguridad.md` |
| Límite arquitectónico u operativo durable | `context/**`, solo si emerge conocimiento estable nuevo |

Los owners se confirman al integrar. Si el owner ya cubre el contrato sin cambio, la evidencia registra “no requiere integración”; no se duplica. Nunca se editan índices generados ni metadata/status manualmente.

## Consolidación y archivado

Este plan no archiva ni cambia status. El cierre futuro requiere:

1. criterios del incremento revisados contra implementación;
2. gates G0–G6 con evidencia;
3. validaciones focales y completas sin fallos nuevos, con todas las garantías R-13.4 heredadas en verde;
4. review independiente sin findings bloqueantes;
5. integración canónica estable cuando aplique;
6. handoff claro a 5/5.

El archivado/transición, si corresponde, se realizará exclusivamente mediante Axiom Core. La finalización de este incremento habilita, pero no completa, `INC-20260909-r13-self-update-release-certification`.

## Validaciones y gates

Desde `C:\repos\Axiom Workspace\Axiom`, ajustando nombres de tests nuevos a los finalmente creados:

```powershell
# Baseline y suites focales
npx vitest run apps/cli/tests/self-update.test.ts packages/user-workspace/tests/self-update.test.ts
npx vitest run apps/cli/tests/launcher-security.test.ts apps/cli/tests/app.test.ts apps/cli/tests/app-launcher.test.ts apps/cli/tests/r13-acc-076-matrix.test.ts

# Integración nueva
npx vitest run apps/cli/tests/app-self-update.test.ts apps/cli/tests/r13-acc-078-launcher-matrix.test.ts

# Helper/install script si sigue formando parte del boundary entregado por 3
node --test scripts/install-global.test.mjs

# Gates de compilación y regresión
npm run typecheck
npm run build
npx vitest run
git diff --check
```

Además:

- ejecutar network guard y confirmar cero conexiones externas;
- inspeccionar handles/procesos temporales al terminar;
- comprobar snapshots de no-mutación;
- revisar diff limitado a paths previstos;
- documentar comandos exactos, conteos, plataforma y fallos preexistentes/nuevos.

No se declarará PASS para un comando no ejecutado ni se usará la evidencia del incremento 5 por adelantado.

## Fuentes y supuestos

Fuentes:

- incremento origen y sus documentos 01–04;
- incrementos 1, 2 y 3 de la serie;
- ACC-077/078 y secuencia en `PLAN-REVISION-INTEGRAL-AXIOM.md`;
- baseline archivada R-13.4/ACC-070..076;
- inspección read-only de planificación (2026-09-09) sobre `apps/cli/src/commands/app*.ts`, `launcher-security.ts`, `self-update.ts`, `apps/cli/static/launcher/*` y `packages/user-workspace/src/self-update.ts`.

Esa inspección no fija un candidate freeze ni commit dentro de este plan: todos los paths/símbolos se tratan como candidatos hasta que G0 registre worktree/commit y los revalide. La afirmación de ausencia de self-update describe ese snapshot de planificación, no una garantía histórica.

Supuestos que se validan en G0–G2:

- el incremento 3 entregará runner/helper headless con plan, `updateOperationLock`, progreso, cancelación y recovery;
- el incremento 1 identificará inequívocamente remoto/ref canónicos y las tres versiones;
- el incremento 2 entregará estado/envelopes y `stateWriteLock` para persistencia/migración;
- el control plane R-13.4 seguirá siendo la única baseline de seguridad;
- no hay requisito de compatibilidad con una ruta self-update Launcher previa, porque no existe actualmente.
