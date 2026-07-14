# INC-20260714 — Launcher UX redesign + ADO workflow surface

> **Código**: INC-20260714-launcher-ux-ado
> **Estado**: archived
> **Fecha**: 2026-07-14
> **Batch**: mejora de front solicitada junto al cierre de brechas post-KVP25 (se ejecuta tras los 5 gaps)
> **Groundeo**: epic `INC-20260711-sdd-launcher-core-port` (+ P3 front-server, P4 launcher, front-longtail, epic-close-panels, ado-peripheral — todos archivados); fuente read-only `C:/repos/KVP25 Workspace/Kvp.Sdd/tools/sdd-launcher`.

## Contexto (qué YA existe — no reconstruir)

El core del sdd-launcher YA está re-homeado en Axiom (epic cerrado):
- `@axiom/launcher`: prompt engine (`buildPrompt`/`craftPrompt`), catálogo de acciones
  (familias `{incremento,bug,plan,back,front,e2e}` × modos `{new,execute,close,archive}` con
  `visibleWhen`), routing adapter-agnóstico, `Launcher` abstraction.
- `axiom app` HTTP server + endpoints en `app-launcher.ts`: `GET /launcher/data`,
  `POST /launcher/craft` (preview prompt, read-only), `POST /launcher/execute`
  (scaffold/transición, preview/confirm), `GET /launcher/registry`, `POST /launcher/launch`,
  `GET /launcher/events` (SSE push).
- Front navegador en `apps/cli/static/launcher/{index.html,launcher.css,launcher.js,panels.js,transport.js}`
  — framework-free, theme-aware, **cero APIs de VSCode** (RISK-004), servido por `axiom app`.
- `@axiom/tracker` (puerto) + `@axiom/tracker-ado` (impl real: WIT create/get/update/**transition
  (state)**/link/childTask/listWorkItems/map*; **estimate** OriginalEstimate/RemainingWork;
  sprints; git branch/PR/commit linking; attachments; **role-close hours scheduling**) — swap
  `tracker.kind: none|ado` config-only; `NullTracker` default; tests con `FakeHttpTransport` (nunca red).
- 3 paneles de operación en el front (epic-close-panels): sugerencias ADO (read-only),
  git role-branch, commit-sync.

**Descartado por el owner (2026-07-11) — NO re-añadir:** cost-dashboard (copilot-usage) y
deployment/trace panels (fuera de Axiom).

## Goal

Que el front de Axiom (a) **se parezca al launcher de KVP25** en pulido y usabilidad
("interfaz bonita de fácil uso"), (b) ayude a los miembros a **crear incrementos/bugs/planes/
implementaciones** olvidándose de la operativa (estructura scaffoldeada por scripts ya existentes,
prompts pregenerados listos para lanzar en la herramienta elegida), y (c) dé **mucha ayuda con
ADO**: crear work items, cambios de estado, estimaciones, marcar horas, generar ramas y enlazar
todo — pidiendo sólo los datos necesarios, desde una UI clara.

## Scope

### P0 — Rediseño UX guiado del front (must-ship)
`apps/cli/static/launcher/{index.html,launcher.css,launcher.js,panels.js}` — **reutiliza los
endpoints existentes** (data/craft/execute/registry/launch/events), NO cambia el server para esto:
- Layout pulido y jerarquía visual clara (tipografía, spacing, estados, light/dark ya existente).
  Referencia estética: el sdd-launcher de KVP25 (`getLauncherHtml.ts`/`media/launcher.js`,
  read-only). Sin frameworks, sin deps nuevas, **cero APIs de VSCode** (mantener RISK-004).
- **Flujo guiado** para "¿qué querés hacer?": crear incremento / bug / plan / implementación
  (familias back/front/e2e) → elegir herramienta/adapter → formulario mínimo (campos dinámicos del
  catálogo + `visibleWhen`) → **preview del prompt pregenerado** con **copiar a 1 clic** + lanzar →
  opción de **ejecutar** (scaffold de la estructura vía los run-functions existentes,
  preview/confirm) → registro en vivo (SSE). El objetivo: el miembro "sólo introduce la información
  necesaria" y obtiene el prompt/estructura sin operar a mano.
- Registro (increments/bugs/plans) legible y accionable.

### P1 — Superficie de workflow ADO (endpoints + UI)
Exponer la superficie de escritura de `@axiom/tracker-ado` (ya existente) como endpoints del
launcher en `app-launcher.ts`, y construir su UI. Operaciones (las que pidió el usuario):
- **Crear work item** (`createWorkItem`/`map*`), **cambio de estado** (`transitionWorkItem`),
  **estimación** (estimate → OriginalEstimate/RemainingWork), **marcar horas** (role-close
  scheduling / CompletedWork+RemainingWork), **generar rama** + **enlazar** (branch/PR/commit
  ArtifactLink), y enlazar el work item al artefacto Axiom (externalRef).
- **Contrato de seguridad**: mismo patrón que `/launcher/execute` — sin mutación sin
  `confirmed:true` (preview primero). **Config-gated**: si `tracker.kind !== 'ado'`, la UI muestra
  un estado claro "ADO no configurado" y no intenta llamar a la red. Con `ado` configurado, delega
  al tracker real.
- **Tests SIN red**: usar `NullTracker` (kind:none) y/o el `FakeHttpTransport` de tracker-ado para
  los endpoints; **nunca** `dev.azure.com`.

## Non-goals

- No reintroducir cost-dashboard ni deployment/trace (descartados por el owner).
- No introducir un framework de front, bundler, ni dependencia nueva; no APIs de VSCode.
- No llamadas de red reales en tests (Fake/Null only).
- No reescribir `@axiom/launcher`, `@axiom/tracker-ado`, ni el server `axiom app` (se extiende
  aditivamente).
- No una extensión VSCode nueva (el front corre en navegador vía `axiom app`, invariante del epic).

## Realismo de alcance

P0 (rediseño UX) es el must-ship y satisface el grueso del pedido ("interfaz bonita", crear
incrementos/bugs/planes/implementaciones con prompts pregenerados y selección de herramienta). P1
(workflow ADO) se entrega tan lejos como cierre verde con tests Fake/Null; cualquier operación ADO
que no entre limpia se documenta como follow-up (no se deja el gate en rojo, no se simula
funcionalidad).

## Acceptance

- El front rediseñado corre en un navegador contra `axiom app`, **cero APIs de VSCode**, guía al
  usuario a crear increment/bug/plan/implementación con prompt pregenerado + copiar + (opcional)
  ejecutar el scaffold; registro en vivo por SSE. `launcher.e2e.test.ts` (y sus equivalentes) verde.
- Endpoints ADO nuevos: sin `confirmed` → preview sin mutar; con `ado` no configurado → estado
  claro sin red; con Fake/Null → crean/transicionan/estiman/enlazan según corresponda (test).
- Sin dependencias nuevas; sin red en tests; RISK-004 (cero `acquireVsCodeApi`/`--vscode-*`) intacto.
- Gate verde en `Axiom/`: `npm run build` → `npm test` → `npm run typecheck` →
  `node apps/cli/dist/index.js doctor` (módulo el flake 5000ms).

## Result

**Estado: implementado (P0 + P1 shipped, gate verde). Ver detalle abajo.**

### P0 — Rediseño UX guiado (shipped completo)

Rediseño completo de `apps/cli/static/launcher/{index.html,launcher.css,launcher.js,panels.js}`.
NO se tocó ningún endpoint existente para P0 (sólo se reutilizan `data/craft/execute/registry/
launch/events`):

- **Sistema visual**: nueva escala tipográfica/espaciado (`--ax-space-*`), cards con elevación
  suave (`--ax-shadow`/`--ax-shadow-md`), botones pill, badges de estado, todo sobre las
  `--ax-*` vars existentes (mantiene light/dark vía `prefers-color-scheme`). Cero frameworks,
  cero deps nuevas. Referencia estética (read-only, no copiada): el sdd-launcher de KVP25
  (`getLauncherHtml.ts`/`media/launcher.js`) — se tomó el lenguaje visual (sidebar/cards/pills/
  badges/spinners), no el código.
- **Flujo guiado de 3 pasos** ("¿qué querés hacer?"): paso 1 elegí familia (incremento/bug/plan/
  back/front/e2e, con labels amigables); paso 2 elegí modo (nuevo/ejecutar/cerrar/archivar,
  filtrado a lo que la familia tiene); paso 3 formulario dinámico (motor `visibleWhen` sin
  cambios). Preview del prompt pregenerado con botón **Copiar** prominente (Clipboard API
  directa) + Lanzar + Ejecutar opcional (preview→confirm intacto).
- **3 vistas** conmutables por tabs (Crear / Registro / ADO & Git) en vez de 3 columnas fijas —
  más foco por tarea. Registro en vivo (increments/bugs/plans) vía el endpoint existente + push
  SSE `registryChanged` (sin cambios de contrato).
- Paneles existentes (sugerencias ADO, rama de rol, commit-sync) preservados 1:1 (mismos ids,
  mismos tipos de mensaje) dentro de la nueva vista "ADO & Git".

### P1 — Superficie de workflow ADO (shipped completo, las 6 operaciones)

Nuevo archivo `apps/cli/src/commands/app-launcher-ado.ts` (aditivo, no reescribe
`app-launcher.ts`/`app-launcher-panels.ts`) con 6 endpoints delegando a `@axiom/tracker-ado`:

- `POST /api/projects/:id/launcher/ado/work-items` → `createWorkItem`
- `POST .../work-items/:workItemId/state` → `transitionWorkItem`
- `POST .../work-items/:workItemId/estimate` → `updateWorkItem({estimate})`
- `POST .../work-items/:workItemId/hours` → `sprints.applyRoleCloseScheduling`
- `POST .../work-items/:workItemId/git-link` → `git.linkArtifact` (branch/pullRequest/commit)
- `POST .../work-items/:workItemId/link-axiom` → **reutiliza `runExternalRefAdd`** (el escritor
  de `metadata.yml#externalRefs` ya existente en `artifact-metadata-cli.ts`) para enlazar el
  work item ADO con el artefacto Axiom (increment/bug/plan) — 100% local, sin red.

Todas: **config-gated** (`tracker.kind !== 'ado'` → `{configured:false, reason}`, sin red, sin
error, mismo patrón que `apiAdoSuggestions`) y **confirm-gated** (`confirmed:true` requerido
para mutar; sin él, `{executed:false, preview}`). "Generar rama" reutiliza el endpoint
`role-branch` ya existente (no se duplicó lógica de git); el nuevo `git-link` es lo que enlaza
esa rama (o un PR/commit) al work item.

UI: `transport.js` mapea los 6 nuevos tipos de mensaje (`adoWorkItemCreate/State/Estimate/
Hours`, `adoGitLink`, `adoLinkAxiom`) a sus endpoints; `panels.js` agrega la card "Workflow ADO"
(6 mini-formularios preview→confirm) con una línea de estado que refleja `configured`/
`trackerKind` (poblada automáticamente al configurar proyecto, reutilizando `loadAdoSuggestions`
— sin llamada de red adicional cuando `kind≠ado`).

**Ninguna operación quedó diferida** — las 6 pedidas por el usuario entraron limpias con tests
Fake/Null.

### Archivos cambiados

- `apps/cli/static/launcher/{index.html,launcher.css,launcher.js,panels.js,transport.js}` (P0 +
  wiring P1)
- `apps/cli/src/commands/app-launcher-ado.ts` (nuevo — los 6 endpoints P1)
- `apps/cli/src/commands/app-api.ts` (rutas + dispatcher para los 6 endpoints, aditivo)
- `apps/cli/tests/launcher-ado-workflow.test.ts` (nuevo — 16 tests)

### Tests

- Nuevo `apps/cli/tests/launcher-ado-workflow.test.ts` (16 tests): NullTracker (kind:none) vía
  HTTP real para las 6 rutas (`configured:false`, sin red, 200) + validación 400; tracker `ado`
  FAKE (`HttpTransport` inyectado, **nunca** `dev.azure.com`) para preview→confirm de las 6
  operaciones, incluida `link-axiom` verificando que escribe un `externalRef` real en el
  `metadata.yml` de un incremento scaffoldeado (vía el lane existente `increment-new` execute).
- `launcher.e2e.test.ts`, `app-launcher.test.ts`, `launcher-panels.test.ts`, `launcher-push.test.ts`,
  `launcher-front-no-vscode.test.ts` — verdes sin cambios (contrato de transporte y mensajes
  preservado 1:1).

### Smoke HTTP (transcript resumido)

Servidor headless (`appStart` del dist, homeDir/staticDir temporales, mismo patrón que los
tests) contra un proyecto de fixture (sin `tracker.json` → `kind:none`):

```
GET /launcher/index.html -> 200 11691 bytes
  contains <title>Axiom Launcher</title>: true
  contains view-switch tabs: true
  contains ADO workflow card: true
GET /launcher/{launcher.css,launcher.js,panels.js,transport.js} -> 200 (todos)

GET /launcher/data -> 200 actions: 19 adapters: [claude-code, github-copilot, cli]

POST /launcher/ado/work-items {title:"Smoke WI", confirmed:true} -> 200
  {"configured":false,"provider":"none","trackerKind":"none",
   "reason":"...ADO no configurado (tracker.kind='none', local-only)... Sin red, sin error."}

GET /launcher/ado-suggestions -> 200
  {"configured":false,"provider":"none","trackerKind":"none","suggestions":[]}
```

### Gate (`Axiom/`)

- `npm run build` → OK (tsc -b, 0 errores).
- `npm test` → **281 archivos de test, 2832 tests, 0 fallos** (una sola pasada, sin necesidad de
  re-run de aislamiento — no hubo flakes en esta corrida).
- `npm run typecheck` → OK (tsc -b, 0 errores).
- `node apps/cli/dist/index.js doctor` → `PASS` (45/57 OK · 0 FALLO · 3 ADVERTENCIA · 9 OMITIDO —
  mismo baseline preexistente, ninguna advertencia nueva introducida).

### Invariantes verificados

- Cero `acquireVsCodeApi` / `--vscode-*` en `apps/cli/static/launcher/*` (grep manual + test
  `launcher-front-no-vscode.test.ts`, ambos limpios).
- Cero dependencias npm nuevas (ningún `package.json` tocado).
- Cero mutaciones git (sólo lectura: `git status`/`git diff --stat` para verificar el diff).
- Cero llamadas de red en tests (NullTracker o `HttpTransport` fake inyectado en todos los casos
  ADO; nunca `dev.azure.com`).
- Preview/confirm intacto en cada endpoint mutante nuevo y en los ya existentes.

## General spec integration

No se requiere integración adicional en `general-spec.md`: el conocimiento estable (contrato
launcher, `IWorkItemTracker`, tracker config, RISK-004) ya estaba consolidado por el epic
`INC-20260711-sdd-launcher-core-port` y sus increments hijos. Este increment es una extensión
aditiva (rediseño de front + 6 endpoints ADO nuevos) que no cambia ninguna decisión canónica ya
documentada.

## Nota de verificación del orquestador (2026-07-14)

Durante la re-verificación independiente (smoke visual con `axiom app` + navegador) se detectó y
corrigió un bug de pulido de CSS en el cambio de vista por tabs: `.view[hidden] { display: none }`
(especificidad 0,1,1) quedaba pisado por `#view-create { display: grid }` (id, 1,0,0), así que al
cambiar a la tab "ADO & Git" la vista "Crear" seguía visible. Fix (1 palabra en
`apps/cli/static/launcher/launcher.css`): `.view[hidden] { display: none !important; }`. Verificado
en navegador: con "ADO & Git" activa, `view-create`/`view-registry` quedan `display:none` y solo
`view-ops` visible. Cambio CSS-only (no afecta build/test/typecheck/doctor). Smoke confirmó también
el wizard guiado (familias increment/bug/plan/backend/frontend/qa-e2e → modos), la selección de
adapter+launcher, y la tarjeta Workflow ADO (6 formularios) con el estado config-gated
"no configurado, sin red" cuando `tracker.kind=none`.
