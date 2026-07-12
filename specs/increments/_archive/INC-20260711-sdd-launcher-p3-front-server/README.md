# Increment: P3 — Local server endpoints + front-end transport shim + `Launcher`

> **Código**: INC-20260711-sdd-launcher-p3-front-server
> **Implementa**: Phase **P3** of epic `INC-20260711-sdd-launcher-core-port`
> **Estado**: archived (core slice shipped green; kept long-tail — server push channel +
> `plan-new`/`plan-execute` execute mapping — RESOLVED in `INC-20260711-front-longtail`;
> cost-dashboard + deployment-trace DISCARDED, out of Axiom scope by owner decision 2026-07-11)
> **Fecha**: 2026-07-11
> **Referencias externas (READ-ONLY)**: `C:/repos/KVP25 Workspace/Kvp.Sdd/tools/sdd-launcher/{media/launcher.js, src/extension.ts, src/services/launchBridge.ts}`

## Goal

Run the prompt-crafting front-end **from anywhere**, served by the **existing** `axiom app`
HTTP server (reuse, do not rebuild). Re-home sdd-launcher's front behind a ~100-line
transport shim; add a `Launcher` abstraction that replaces `launchBridge`.

## Scope

- **P3.1 — Server endpoints.** Extend `apps/cli/src/commands/app-api.ts` (via a new
  `app-launcher.ts` module) to expose the launcher's core message types as HTTP endpoints,
  mapped onto the existing GET-read / POST-execute-with-`confirmed` contract and reusing
  `loadAppPlugins` / `executeSubcommand`:
  - `GET  /api/projects/:id/launcher/data` — families/actions from the `@axiom/launcher`
    catalog (`createLauncherActions`) + adapters (`AXIOM_ADAPTER_ROUTING`) + roles + launcher kinds.
  - `POST /api/projects/:id/launcher/craft` — craft/preview a prompt via the `@axiom/launcher`
    prompt engine + adapter routing (`craftPrompt`). Read-only: NO mutation, shows the exact
    prompt + the mapped CLI command.
  - `POST /api/projects/:id/launcher/execute` — execute an action (scaffold/transition).
    Delegates to the existing P1 `run*Subcommand` helpers via `executeSubcommand`. No mutation
    without `confirmed:true` (preview-only otherwise).
  - `GET  /api/projects/:id/launcher/registry` — list increments/bugs/plans (`listArtifacts`).
  - `POST /api/projects/:id/launcher/launch` — launch the crafted prompt via the `Launcher`
    abstraction (server-side HTTP send; clipboard delegated to the browser).
- **P3.2 — Front-end transport shim + re-home.** `apps/cli/static/launcher/` served by the
  existing static host. `transport.js` exposes `window.AxiomBridge.postMessage()` → `fetch`
  to the endpoints and re-dispatches server responses as `window` `message` events (same
  contract shape as a VSCode webview, so a full `launcher.js` port drops in unchanged). The
  working-core UI (`launcher.js` + `launcher.css` + `index.html`) covers project/adapter/
  launcher selection, action nav from the catalog, dynamic form with `visibleWhen`, live
  prompt preview, execute/confirm, and the registry table.
- **P3.3 — `Launcher` abstraction.** `packages/launcher/src/launcher.ts`:
  `Launcher.launch(request) => Promise<LaunchResult>` with `ClipboardLaunch`,
  `HttpLaunch(agentEndpoint)`, optional `VSCodeLaunch`, and a `createLauncher(kind, opts)`
  factory. Pure + dependency-injected; the clipboard path is testable with no running agent.
- **P3.4 — E2E lane (local-only).** Drive `axiom app` end-to-end: open action → craft prompt
  → preview → confirm → local-only artifact created, with NO live ADO (tracker.kind:none /
  NullTracker).

## Non-goals

- No rebuild of the `axiom app` HTTP server, the P1 CLI subcommands, or the `@axiom/workflow`
  lifecycle — the endpoints are thin wrappers that delegate.
- Not carrying forward the earlier discardable HTML front (epic D-002).
- No faithful port of the full ~5359-LOC `launcher.js` in this increment (see D3-002):
  advanced panels are deferred.
- No live Azure DevOps: the E2E lane is local-only (`NullTracker`).

## Acceptance (from PLAN.md P3)

1. The front runs in a plain browser against `axiom app` with **zero VSCode APIs** present;
   preview shows the exact prompt/command **without** executing.
2. `Launcher` selection is user-visible and testable (clipboard path asserted with no running agent).
3. The earlier discardable HTML front is **not** carried forward.
4. Corner cases: POST without `confirmed` → preview only, no mutation; unknown action →
   graceful error; server serves the static front + responds to the core endpoints; adapter
   with no routing entry → default mention (fallback from P4).

## E2E lane

`parallel` lane (epic §9). `apps/cli/tests/e2e/launcher.e2e.test.ts` drives the real
`axiom app` server over HTTP against a local-only fixture (`topology single-repo`,
`workflows.yaml` seeded, no tracker/ADO):

1. `GET  /launcher/data` → the catalog is served (families × modes actions present).
2. `POST /launcher/craft` (adapter `claude-code`, action `increment-new`) → returns the exact
   prompt + the mapped CLI command; **no artifact is created** (preview only).
3. `POST /launcher/execute` **without** `confirmed` → `executed:false` + preview; **no mutation**.
4. `POST /launcher/execute` with `confirmed:true` → a local-only increment artifact folder is
   created under the spec root; `workflow-state.json` advances to `specifying`; `externalRefs`
   empty (no ADO).

## Result

- **FULL (shipped, green):**
  - Server endpoints `launcher/{data,craft,execute,registry,launch}` on the existing server,
    honoring preview-vs-confirmed and delegating to P1 / `executeSubcommand`.
  - Transport shim (`window.AxiomBridge`) + working-core front (action nav, dynamic form with
    `visibleWhen`, live prompt preview, execute/confirm, registry table, project/adapter/
    launcher selection). Zero `acquireVsCodeApi`, zero `--vscode-*` (asserted by test).
  - `Launcher` abstraction (`ClipboardLaunch`/`HttpLaunch`/`VSCodeLaunch` + `createLauncher`),
    unit-tested (clipboard asserted with no agent; http via injected fetch).
  - E2E lane test (open → craft → preview → confirm → local artifact, no ADO).
  - Server-endpoint tests; front no-VSCode assertion test.
- **LONG-TAIL — RESOLVED / DISCARDED (`INC-20260711-front-longtail`):**
  - ✅ **Server push channel** — implemented as **SSE** (`GET .../launcher/events`,
    `text/event-stream`; no new dependency, no WebSocket handshake). The transport shim opens the
    stream and re-dispatches pushes as `window` `message` events, with a graceful fetch fallback
    when `EventSource` is unavailable. A confirmed execute pushes `registryChanged`.
  - ✅ **`plan-new` / `plan-execute` execute mapping** — `plan-new` scaffolds a plan via
    `runPlanCreate`; `plan-execute` delegates to the real `plan-approve` transition. Both honor
    the preview/execute/**confirmed** contract (no mutation without `confirmed:true`).
  - ❌ **Cost dashboard and deployment trace: DISCARDED — out of Axiom scope by owner decision
    2026-07-11; not deferred/pending.** (Future revisit possible but untracked.)
- **STILL PARTIAL (epic remaining pointer, not this phase):** faithful port of the remaining
  advanced `launcher.js` panels — ADO suggestions, role-branch/commit-sync git panels.
