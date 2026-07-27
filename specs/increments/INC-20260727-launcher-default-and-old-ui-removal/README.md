# Increment: Launcher as default `axiom app` front + old operator-UI removal

Status: closed
Date: 2026-07-27

## Goal

`axiom app` must open the LAUNCHER (`static/launcher/`, the current,
working front — Crear/Registro/ADO&Git/Instalar-Unirse, adapter selector,
doctor gate) by default instead of the OLD, buggy root operator UI
(`static/index.html` + `app.js` — "Axiom App" with a Topología/Roles/
Toolchain/Memoria/MCPs/Workflows tab bar, a stuck "Confirmar acción" modal
on load, and a `kvp25 (undefined)` render bug). The old UI and its
exclusive dead code must be deleted, not just bypassed.

## Context

`apps/cli/src/commands/app.ts` starts a local HTTP server
(`apps/cli/src/commands/app-api.ts`) and serves static assets from
`apps/cli/static/`. Two fronts coexisted there:

- `apps/cli/static/index.html` + `app.js` + `style.css` + `sw.js` +
  `manifest.json` — the ORIGINAL PWA shell from `0025 Lote A`
  (`PLAN-INC-0025`), superseded by the launcher but never removed. This is
  what `axiom app` opened by default (bare `GET /`).
- `apps/cli/static/launcher/` (`index.html`, `launcher.css`, `launcher.js`,
  `panels.js`, `transport.js`) — the REAL, current front (ported from
  KVP25's sdd-launcher, `INC-20260711-sdd-launcher-p3-front-server` and
  every `INC-20260711/14/15-*` launcher batch since), served at
  `/launcher/`. `05_Interfaces_Operativas.md` already documents this as
  "el front re-alojado se sirve bajo `/launcher/`" — the spec was already
  accurate; the CLI's default open behavior was not.

## Scope

- `apps/cli/src/commands/app.ts`: `appStart` opens (and the CLI banner
  prints) `${url}/launcher/` instead of the bare server root. `--port`/
  `--no-open` unaffected.
- `apps/cli/src/commands/app-api.ts`: `GET /` and `GET /index.html` now
  return `302 Location: /launcher/` (belt-and-suspenders — the old root
  `index.html` no longer exists on disk either). Removed, as PROVABLY DEAD
  (consumed ONLY by the deleted `app.js`, confirmed by a repo-wide
  consumer search before deletion — see "Implementation notes"):
  - Routes + handlers: `GET /api/projects/:id/topology`
    (`apiGetTopology`), `GET /api/projects/:id/roles` (`apiGetRoles`,
    distinct from `/launcher/roles/register|assign`), `GET
    /api/projects/:id/toolchain` (`apiGetToolchain`), `GET
    /api/projects/:id/memory` (`apiGetMemory`), `GET
    /api/projects/:id/mcps` (`apiGetMcps`), `GET
    /api/projects/:id/workflows/:workflowId` (`apiGetWorkflowState`).
  - Routes only (function kept — see below): `POST
    /api/projects/:id/commands/preview` (`buildCommandPreview` + its
    `buildPreview`/`BuildPreviewOptions`/`PreviewResult` helpers — DELETED,
    zero other callers), `POST /api/projects/:id/commands/execute` (the
    ROUTE dispatch was removed; `executeSubcommand` — the FUNCTION — is
    KEPT because `app-launcher.ts`'s `apiExecuteLauncherAction` calls it
    directly, in-process, for the launcher's own `transition` actions).
  - Now-unused imports/consts tied only to the removed code:
    `loadWorkflowState` (`@axiom/workflow`), `runToolchainShow`,
    `runRolesList`, `runMcpList` (`defaultMcpManifestPath` was already
    unused before this increment — left untouched, unrelated),
    `runMemoryShow`, `runTopologyShow`, `MCP_MANIFEST_REL`,
    `parseCommandBody`/`CommandRequestBody` (only caller was the two
    removed routes), `ExecuteResult` (already-unused type, part of the
    same dead preview/execute documentation cluster).
- `apps/cli/static/`: deleted `index.html`, `app.js`, `style.css`,
  `sw.js`, `manifest.json` (the old shell's entire PWA scaffolding —
  confirmed unreferenced by `static/launcher/*`, see below).
- `apps/cli/tests/app.test.ts`: replaced the 5 dead-route scenarios
  (topology ×2, commands/preview, commands/execute ×2) with 4 new
  scenarios covering the redirect + shipped-static regression, using the
  REAL `static/` dir (not the tests' usual empty tmp dir) so the assertions
  exercise the actual shipped files.

## Non-goals

- `apps/cli/static/index.html`'s OTHER endpoints that are NOT exclusive to
  it were deliberately KEPT (not touched, listed with why in
  "Implementation notes"): `GET /api/projects/:id` (`apiGetProject`), `GET
  /api/projects/:id/plugins` (`apiGetPlugins`), `GET /api/help`
  (`apiGetHelp`) — none of these were actually fetched by the old
  `app.js` (verified by reading its source before deletion); removing them
  would be scope creep unrelated to the old-UI removal.
- No change to any `/launcher/*` route, the launcher's own front assets,
  or the `Launcher`/adapter-routing abstractions.
- No change to `executeSubcommand`'s behavior/signature (only its HTTP
  route wrapper is gone; the function and its normalizers are untouched
  and still exercised by `app-launcher.test.ts` et al.).
- Did not edit `Axiom.Spec/specs/00..08` (explicit instruction for this
  increment) — see "General spec integration".

## Acceptance criteria

- [x] `axiom app` opens `${url}/launcher/` (verified with the built CLI:
      banner prints `.../launcher/`, browser-open call receives the same
      URL); `--no-open`/`--port` still work.
- [x] `GET /` and `GET /index.html` return `302` with `Location:
      /launcher/`.
- [x] `GET /launcher/` still serves 200 + the launcher HTML (title `Axiom
      Launcher`) and its assets (`launcher.css`/`launcher.js`/`panels.js`/
      `transport.js`) unaffected.
- [x] `apps/cli/static/index.html`, `app.js`, `style.css`, `sw.js`,
      `manifest.json` deleted; `GET /app.js` (and the other 3) → 404 against
      the real static dir.
- [x] Old-UI-exclusive backend endpoints removed with an explicit
      consumer-search rationale per endpoint (done — see "Implementation
      notes"); nothing shared or ambiguous was deleted.
- [x] `npm run build` (`tsc -b`) passes.
- [x] `apps/cli/tests/app.test.ts` and every app/app-api/launcher test file
      green; broad `apps/cli/tests` safety pass green.
- [x] Runtime route check against the BUILT CLI (`node
      apps/cli/dist/index.js app`), not just unit tests: `GET /` → 302
      `/launcher/`, `GET /launcher/` → 200 `<title>Axiom Launcher</title>`,
      `GET /app.js` → 404.

## Open questions

None blocking.

## Assumptions

1. **`sw.js`/`manifest.json` are old-UI-exclusive, not shared PWA
   plumbing.** Verified before deleting: `static/launcher/index.html`/
   `launcher.js`/`panels.js`/`transport.js` contain zero references to
   `manifest`, `serviceWorker`, or `sw.js` (grepped); `sw.js`'s own
   `SHELL_ASSETS` list only names the old shell's own files
   (`/`, `/index.html`, `/app.js`, `/style.css`, `/manifest.json`); the
   launcher registers no service worker. Safe to delete alongside
   `index.html`/`app.js`/`style.css`.
2. **`GET /api/projects/:id` (singular, `apiGetProject`), `GET
   /api/projects/:id/plugins`, and `GET /api/help` are NOT old-UI-exclusive
   endpoints**, even though they are also not currently consumed by the
   launcher's `transport.js`. `app.js`'s source (read in full before
   deletion) never fetches any of them — its 6 tabs map to exactly
   topology/roles/toolchain/memory/mcps/workflows, and its preview/execute
   forms map to `commands/preview`/`commands/execute`. Since Part C's
   mandate is "endpoints consumed ONLY by the deleted old UI", these three
   are out of scope; deleting them would be an unrelated, unrequested
   cleanup (deferred, not silently dropped).
3. **`executeSubcommand` (the exported function) is kept, only its direct
   HTTP route is removed.** Confirmed via a repo-wide grep that
   `app-launcher.ts`'s `apiExecuteLauncherAction` (the launcher's OWN
   `/launcher/execute` handler) calls `executeSubcommand` directly, in
   -process, for every `kind: 'transition'` action in
   `LAUNCHER_ACTION_COMMAND` (increment/bug/role/plan/qa-e2e lifecycle
   actions). Deleting the function would have broken the launcher's real
   execute path — this is exactly the "shared, keep it" case Part C called
   out, not a dead endpoint.
4. **`buildCommandPreview` (the exported function backing
   `commands/preview`) has no other caller** (confirmed by a repo-wide
   grep: only `app-api.ts`'s own now-removed route and the now-removed test
   assertion referenced it) — `app-launcher.ts`'s craft/preview path
   (`/launcher/craft`) has its own separate, adapter-aware prompt-crafting
   logic and never called `buildCommandPreview`. Safe to delete in full,
   including its private helpers.

## Implementation notes

### Consumer-search evidence (before deleting each endpoint)

For every candidate endpoint, ran a repo-wide grep for its wrapper
function name (`apiGetTopology`, `apiGetRoles`, `apiGetToolchain`,
`apiGetMemory`, `apiGetMcps`, `apiGetWorkflowState`, `buildCommandPreview`,
`executeSubcommand`) across `apps/cli/src`, `apps/cli/tests`, `packages/`,
and `apps/cli/static/launcher/*.js` (the launcher's own `fetch`/message
paths), plus a direct read of the deleted `app.js` to enumerate exactly
which endpoints it called. Results:

| Endpoint (route) | Wrapper fn | Old app.js caller? | Any other caller? | Action |
|---|---|---|---|---|
| `GET .../topology` | `apiGetTopology` | yes (`renderTopology`) | no | removed (route+fn) |
| `GET .../roles` | `apiGetRoles` | yes (`renderRoles`) | no | removed (route+fn) |
| `GET .../toolchain` | `apiGetToolchain` | yes (`renderToolchain`) | no | removed (route+fn) |
| `GET .../memory` | `apiGetMemory` | yes (`renderMemory`) | no | removed (route+fn) |
| `GET .../mcps` | `apiGetMcps` | yes (`renderMcps`) | no | removed (route+fn) |
| `GET .../workflows/:id` | `apiGetWorkflowState` | yes (`renderWorkflows`) | no | removed (route+fn) |
| `POST .../commands/preview` | `buildCommandPreview` | yes (`previewAndExecute`) | no | removed (route+fn) |
| `POST .../commands/execute` | `executeSubcommand` | yes (`previewAndExecute`) | **yes** — `app-launcher.ts`'s `apiExecuteLauncherAction` (in-process call, not HTTP) | route removed, **fn kept** |
| `GET /api/projects/:id` | `apiGetProject` | no | n/a (out of scope) | kept, untouched |
| `GET .../plugins` | `apiGetPlugins` | no | n/a (out of scope) | kept, untouched |
| `GET /api/help` | `apiGetHelp` | no | n/a (out of scope) | kept, untouched |

### Files changed

- `apps/cli/src/commands/app.ts` — `appStart` now opens
  `${url}/launcher/`; `printReadyBanner` (called from `registerApp`'s
  action) now receives `${result.url}/launcher/`. `AppStartResult.url`
  itself is unchanged (still the bare server URL) to avoid any risk to
  other callers of `appStart`.
- `apps/cli/src/commands/app-api.ts` — new `RouteMatch` kind
  `redirectToLauncher`, matched for `GET|HEAD /` and `GET|HEAD
  /index.html`, dispatched as `res.writeHead(302, { Location:
  '/launcher/' })`. Removed the 6 dead endpoint functions + the
  `buildCommandPreview` cluster + `parseCommandBody`/`CommandRequestBody`
  + the two dead routes/dispatcher cases; kept `executeSubcommand` and its
  normalizers (still exported, still used by `app-launcher.ts`). Updated
  the module-level doc comment to record the removal and the
  redirect.
- `apps/cli/static/index.html`, `apps/cli/static/app.js`,
  `apps/cli/static/style.css`, `apps/cli/static/sw.js`,
  `apps/cli/static/manifest.json` — deleted.
- `apps/cli/tests/app.test.ts` — Scenarios 3-7 (topology ×2,
  commands/preview, commands/execute ×2) replaced by Scenarios 3-6
  (redirect for `/` and `/index.html`, `/launcher/` still serves 200 with
  the real static dir, the 4 deleted old-UI files 404 against the real
  static dir). `buildCommandPreview` import removed (function no longer
  exists). `httpRequest` test helper extended to also return response
  `headers` (needed to assert `Location`).

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0.
- `npx vitest run apps/cli/tests/app.test.ts apps/cli/tests/app-launcher.test.ts
  apps/cli/tests/app-plugins.test.ts apps/cli/tests/launcher-front-no-vscode.test.ts
  apps/cli/tests/launcher-doctor.test.ts apps/cli/tests/launcher-onboarding.test.ts
  apps/cli/tests/launcher-panels.test.ts apps/cli/tests/launcher-push.test.ts
  apps/cli/tests/launcher-ado-workflow.test.ts apps/cli/tests/launcher-ado-bridge.test.ts
  apps/cli/tests/launcher-execution-mode.test.ts apps/cli/tests/e2e/launcher.e2e.test.ts`
  — **12 files / 118 tests passed** (single run, no flakes/retries needed).
- `npx vitest run apps/cli/tests` (full safety pass) — **128 files / 1212
  tests passed**, zero failures, single run.
- Runtime check against the BUILT CLI (`node apps/cli/dist/index.js app
  --no-open --port 4599`, real `AXIOM_HOME`, then `curl` against the
  chosen port 4600):
  - `GET /` → `302`, `Location: /launcher/`.
  - `GET /index.html` → `302`, `Location: /launcher/`.
  - `GET /launcher/` → `200`, body contains `<title>Axiom
    Launcher</title>`.
  - `GET /app.js`, `/style.css`, `/sw.js`, `/manifest.json` → `404` each.
  - `GET /launcher/launcher.js` → `200` (launcher assets unaffected).
  - `GET /api/health` → `200` (API unaffected).
  - Process stopped cleanly afterward.

No pre-existing failures were encountered in any touched or adjacent
scope.

## Result

Implemented. `axiom app` now opens and prints the launcher URL
(`.../launcher/`) by default; `GET /` and `GET /index.html` 302-redirect
to `/launcher/` server-side as a second layer of defense. The old operator
UI (`static/index.html` + `app.js` + `style.css` + `sw.js` +
`manifest.json`) and its 6 exclusive dead backend endpoints
(topology/roles/toolchain/memory/mcps/workflow-state as routes, plus the
`commands/preview` route+function and the `commands/execute` route) are
fully deleted, each confirmed dead via an explicit repo-wide consumer
search (documented above) before removal. `executeSubcommand` (the
function, still used in-process by the launcher's own execute path) and
three endpoints not actually tied to the old UI (`projects/:id`,
`plugins`, `/api/help`) were deliberately left untouched.

## General spec integration

No edits made to `Axiom.Spec/specs/00..08` — explicitly out of scope per
this increment's instructions. No new stable knowledge needs integration
beyond what is already recorded: `05_Interfaces_Operativas.md` already
documents "el front re-alojado se sirve bajo `/launcher/` desde el server
`axiom app`" (from `INC-20260711-sdd-launcher-p3-front-server`), which was
already accurate — this increment only made the CLI's default
open/redirect behavior consistent with that pre-existing, correct spec
statement, and deleted the leftover 0025-era shell it superseded. If a
future spec pass wants to note explicitly that the pre-launcher root PWA
shell no longer exists, that is a documentation-only follow-up, not a
behavior change.
