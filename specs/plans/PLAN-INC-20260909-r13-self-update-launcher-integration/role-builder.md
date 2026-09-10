# Rol de plan: builder

## Rol

- nombre del rol: builder
- repositorio principal asignado: `Axiom` (runtime; rol lógico `axiom-code-builder`)
- repositorio de especificación: `Axiom.Spec`, únicamente para la trazabilidad e integración canónica autorizadas
- posición en la secuencia: incremento 4/5, posterior a identity, state/CLI y updater transaccional

## Objetivo del rol en este plan

Implementar la integración futura de self-update en el Launcher como adaptador del runner headless compartido con CLI. El builder debe entregar bootstrap no bloqueante, inventario/provenance, rutas seguras, preview→confirmación, helper externo, progreso bounded, cancelación, cierre/reinicio y recovery sin duplicar el motor Git. La existencia del plan no implica que la capacidad ya exista.

## Alcance incluido

1. **Gate de dependencias**
   - Confirmar outputs y evidencia de los incrementos 1, 2 y 3.
   - Confirmar que la CLI **ya consume** la API headless real de status/check/plan/apply/recover y helper; “podría consumirla” no basta para `GO`.
   - Comparar schemas/errors/progress/cancel/journal y claim idempotente con este incremento.
   - Registrar `STOP` antes de código si falta un contrato; no crear sustitutos locales.

2. **Wiring de servidor**
   - Añadir el adaptador Launcher en un módulo focal, probablemente `apps/cli/src/commands/app-self-update.ts`.
   - Extender `app-api.ts` mediante `RouteMatch`, `matchRoute`, schemas cerrados y `handleApiRequest`, sin bypass del middleware.
   - Programar refresh canónico después del bootstrap con coalescing/deadline/abort.
   - Derivar installation, remote, ref, paths y executable server-side.
   - Reconciliar operación/journal al arrancar.

3. **Control plane y confirmación**
   - Reutilizar cookie/auth, Host/Origin, no-CORS, headers, body 256 KiB, timeout 5 s y errores sanitizados.
   - Reutilizar `issuePreviewGrant`/`consumePreviewGrant` con binding a installation, operationId, action, plan digest, target y restart policy.
   - Consumir grant antes de spawn/handoff; aplicar single-use y race tests.
   - Proteger status/progress/recovery con sesión; no poner secretos o token en URL.

4. **Helper y lifecycle**
   - Reservar `operationId` en plan/grant y exigir claim/journal idempotente antes del ACK.
   - Spawn externo sin shell, con args/env/cwd controlados; handshake 5 s default/10 s máximo.
   - Tratar ACK perdido/tardío como `handoff_uncertain`: bloquear retry y reconciliar operationId/journal/`updateOperationLock`/proceso. Un token consumido no revive.
   - Separar handoff, engine phase, cancellation, relaunch y runner outcome; evento máximo 4 KiB e historial 100/64 KiB.
   - Propagar cancel solo en fase segura y esperar ACK/reconciliación.
   - Coordinar drain 3/5 s, PID wait 15/30 s y `close-only`/`close-and-restart`; relaunch ACK 10/30 s.
   - Separar fallo de relaunch de outcome de instalación y soportar sesión nueva/reattach.

5. **Static UI**
   - Integrar panel global en `apps/cli/static/launcher/{index.html,launcher.js,panels.js,transport.js,launcher.css}` según encaje real.
   - Mostrar cada versión con freshness/provenance propia, `state`/`lastKnownRelation` y refresh separado.
   - Implementar policy sin default, plan inmutable con operationId, diálogo explícito, invalidación anti-race y botones honestos.
   - Implementar progreso, handoff incierto, cancelación, restart, resume, offline y recovery bifásico con policy explícita y accesibilidad completa.
   - No almacenar confirmation token en URL/localStorage/logs.

6. **Pruebas y evidencia**
   - Crear suites HTTP/control-plane/frontend/helper focales.
   - Usar remotos Git bare, clones, homes, prefixes, ports y journals temporales.
   - Ejecutar al menos un E2E con el entry point real del helper; usar fixture ejecutable solo para ACK perdido/tardío, crash y timeout.
   - Bloquear red externa y aislar filesystem/procesos de la máquina.
   - Probar snapshots de no-mutación, paridad CLI/Launcher y deadlines con fake clock/procesos controlados.
   - Validar accesibilidad con harness DOM/browser existente o protocolo manual reproducible; capturas estáticas no bastan.
   - Ejecutar build, typecheck, tests focales/completos y diff check.

## Fuera de alcance

- Implementar o corregir release identity, comparador SemVer, manifest v2, `stateWriteLock`, `updateOperationLock`, motor Git, staging, build candidato, activation, rollback o journal.
- Copiar funciones legacy de `apps/cli/src/commands/self-update.ts` dentro del Launcher.
- Añadir auto-apply/auto-recover, remoto/path elegible desde UI o actualizaciones in-process.
- Relajar el control plane R-13.4 o crear un segundo confirmation store.
- Definir release CI, publicación, packaging, matriz final o documentación de usuario del incremento 5.
- Modificar metadata, receipts, índices o status manualmente.
- Tocar archivos no relacionados para “limpieza” oportunista.

## Dependencias

Obligatorias:

- `INC-20260909-r13-self-update-release-identity`: remote/ref canónicos, `published/downloaded/installed`, provenance, freshness y estados.
- `INC-20260909-r13-self-update-state-cli-contract`: schema, envelopes, taxonomía, gramática/runners CLI y `stateWriteLock` de persistencia.
- `INC-20260909-r13-self-update-transactional-updater`: `UpdatePlan`, engine, helper, `updateOperationLock`, progress/cancel, journal, rollback y recovery.
- R-13.4/ACC-070..076: auth, same-origin, schemas, límites, headers, SSE y grants.

Posterior:

- `INC-20260909-r13-self-update-release-certification` consume las pruebas y limitaciones de este rol.

## Secuencia de ejecución

1. **STOP/GO inicial**: inspeccionar contratos y ejecutar baseline; no editar runtime antes de G0–G3.
2. **Diseño mínimo**: fijar view models, schemas, rutas y bindings; threat review y G4.
3. **Adaptador y scheduler**: status local, refresh no bloqueante/coalesced y reconciliation.
4. **API segura**: rutas, schemas, grants y progreso autenticado.
5. **Helper lifecycle**: handshake, cancel, PID wait, close/relaunch, reattach/recovery.
6. **Frontend**: estados, plan/confirm, operation/recovery y accesibilidad.
7. **Pruebas focales**: unitarias, HTTP, static y helper con fixtures herméticos.
8. **Regresión completa**: typecheck/build/Vitest/helper tests/diff-check.
9. **Review independiente**: comportamiento, seguridad, no-mutación, no-duplicación y rollback.
10. **Integración/handoff**: conocimiento estable y evidence package para 5/5, sin transición manual.

Cada fase solo continúa si la anterior mantiene `GO`. Un fallo del engine se enruta al incremento 3; no se absorbe en el adaptador.

## Rutas y símbolos probables

| Área | Path/símbolo observado o candidato | Uso previsto |
|---|---|---|
| App lifecycle | `apps/cli/src/commands/app.ts`: `registerApp`, `appStart`, `openBrowser` | Refresh post-bootstrap, shutdown y relaunch coordination. |
| Router | `apps/cli/src/commands/app-api.ts`: `RouteMatch`, `matchRoute`, `CLOSED_BODY_FIELDS`, `BODY_FIELD_TYPES`, `handleApiRequest`, `createAppServer` | Rutas/schemas/middleware. |
| Security | `apps/cli/src/commands/launcher-security.ts`: `issuePreviewGrant`, `consumePreviewGrant` | Grant single-use y binding ampliado. |
| Adapter | `apps/cli/src/commands/app-self-update.ts` (candidato) | Handlers delgados y mapeo de view models. |
| CLI wrapper | `apps/cli/src/commands/self-update.ts` | Consumidor par del runner, no código a copiar. |
| Shared runner | path entregado por incremento 3; por confirmar | Única autoridad de operaciones/engine. |
| UI | `apps/cli/static/launcher/index.html`, `launcher.js`, `panels.js`, `transport.js`, `launcher.css` | Panel, estado, transport, diálogo y accesibilidad. |
| Tests | `apps/cli/tests/app-self-update.test.ts` y matriz R-13.5 (candidatos) | HTTP, frontend, helper y no-mutación. |

No se crea un archivo candidato si el runtime vigente ya posee una abstracción propietaria adecuada. Los símbolos marcados como observados provienen de la inspección read-only de planificación y deben revalidarse en G0 contra worktree/commit registrado; hasta entonces no son evidencia contractual.

## Seguridad y rollback

Checklist antes de G5:

- [ ] No hay remote/ref/path/home/prefix/executable controlado por browser.
- [ ] Ninguna mutación evita auth, Origin, schema, límites o grant.
- [ ] Token no aparece en URL, DOM persistente, log o telemetry.
- [ ] Apply/recover consumen grant ligado a operationId antes de spawn y revalidan en runner.
- [ ] Claim/journal es idempotente; ACK perdido/tardío entra en fencing y bloquea retry.
- [ ] Spawn usa `shell: false`, args separados y environment mínimo.
- [ ] Fetch 10/30 s, handshake 5/10 s, drain 3/5 s, PID wait 15/30 s y relaunch ACK 10/30 s tienen cleanup.
- [ ] Evento máximo 4 KiB, historial 100/64 KiB y transporte conservan caps.
- [ ] Cierre de pestaña no cancela ni duplica operación.
- [ ] Helper espera salida de PID antes de activation.
- [ ] Rollback/recovery pertenecen al runner y su journal se preserva.
- [ ] Retirar UI/API no abandona una operación aceptada; CLI recover sigue disponible.

Si una regresión obliga a revertir la integración, deshabilitar nuevas operaciones de forma coherente, dejar terminar/recover helpers activos y revertir juntos panel+transport+handlers. Nunca borrar journal, matar un helper arbitrariamente ni rollbackear una instalación verificada solo por fallo de UI/relaunch.

## Validaciones y evidencias esperadas

### Evidencia funcional

- Matriz requisito/criterio/test con CA-LSI-01..46.
- Capturas textuales/DOM de cada state/freshness/operation sin datos sensibles.
- Evidencia interactiva de teclado, foco, live regions, reduced motion y señal no solo cromática mediante harness confirmado o protocolo manual reproducible con herramienta/versión.
- Trazas bounded de requestId/sequence, operationId claim, ACK perdido/tardío, PID wait y restart policy.
- Comparación de outputs semánticos CLI/Launcher para fixtures iguales.

### Evidencia de no-mutación

Antes/después de bootstrap, status, check, plan y rechazos:

- HEAD/branch/worktree;
- refs no autorizadas;
- installation root/shim;
- manifest y journal;
- procesos/handles restantes.

Check puede cambiar únicamente refs/cache permitidas y debe quedar distinguido en el snapshot.

### Comandos

Desde `C:\repos\Axiom Workspace\Axiom`:

```powershell
npx vitest run apps/cli/tests/launcher-security.test.ts apps/cli/tests/app.test.ts apps/cli/tests/app-launcher.test.ts apps/cli/tests/r13-acc-076-matrix.test.ts
npx vitest run apps/cli/tests/self-update.test.ts packages/user-workspace/tests/self-update.test.ts
npx vitest run apps/cli/tests/app-self-update.test.ts apps/cli/tests/r13-acc-078-launcher-matrix.test.ts
# Solo si G0 confirma que sigue siendo parte del boundary de producción
node --test scripts/install-global.test.mjs
npm run typecheck
npm run build
npx vitest run
git diff --check
```

Los nombres nuevos se ajustan a los archivos realmente creados. Registrar comando, plataforma, conteo, duración, resultado y clasificación de fallos. No afirmar PASS si no se ejecutó.

### Review independiente

El reviewer debe comprobar:

- coherencia con ACC-077/078 y dependencias 1–3;
- ausencia de afirmaciones/código que trate self-update Launcher como preexistente;
- no-duplicación de engine/semver/state/taxonomía;
- no-auto-apply, confirmación vinculada y fail-closed;
- seguridad HTTP/helper, cotas, sanitización y network isolation;
- UX de cancel/restart/recovery/offline y accesibilidad;
- estrategia de rollback y handoff a 5/5.

## Bloqueos y observaciones

Bloqueos que fuerzan `STOP`:

- dependencia 1, 2 o 3 no disponible/incompatible;
- CLI aún depende exclusivamente de runners legacy mutantes que no pueden compartirse;
- helper no sobrevive al cierre, carece de claim idempotente/fencing, no confirma journal o actualiza in-process;
- falta de binding extensible en confirmation grant;
- cualquier suite heredada de auth/origin/schema/limits/grants/SSE en rojo;
- tests requieren remoto/npm/home reales;
- no es posible distinguir installed/downloaded/published o outcome observado;
- cualquier bypass de same-origin/schema/limits;
- hallazgo bloqueante de review o fallo nuevo en build/tests.

Observaciones no bloqueantes deben documentarse con owner y follow-up. La release CI/docs finales, cobertura completa Windows/POSIX y publicación son handoff explícito al incremento 5, no deuda a ocultar en este rol.
