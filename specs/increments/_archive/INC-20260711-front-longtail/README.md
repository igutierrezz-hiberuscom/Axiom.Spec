# Increment: Front long-tail — push channel + plan-mapping + discard cleanup

> **Código**: INC-20260711-front-longtail
> **Resuelve**: the KEPT P3 front long-tail of `INC-20260711-sdd-launcher-p3-front-server`
> (phase P3 of the epic `INC-20260711-sdd-launcher-core-port`)
> **Estado**: in_progress
> **Fecha**: 2026-07-12

## Goal

Close out the P3 front long-tail with an owner decision applied: implement the two **kept**
items (a server **push channel** and the **`plan-new`/`plan-execute` execute mapping**), and
**remove** the two **discarded** items (copilot-usage cost dashboard, deployment/trace panels)
from every pending record — they leave Axiom scope, they are NOT deferred/pending.

## Scope

- **Push channel (SSE).** Add a server-push endpoint to the existing `axiom app` HTTP server
  (`GET /api/projects/:id/launcher/events`, `text/event-stream`) so the server can PUSH events
  (e.g. `registryChanged` after a confirmed execute) to the front. Implemented with **Server-Sent
  Events**, not a raw WebSocket: SSE rides the existing `http.Server`, needs no new dependency and
  no 101-Upgrade handshake, and is consumed by the browser `EventSource`. A per-server
  `LauncherEventHub` broadcasts to the subscribers of a project. The transport shim
  (`static/launcher/transport.js`) opens the stream on `configure` and re-dispatches pushes as
  `window` `message` events (same contract the front already consumes), with a **graceful fallback**
  to the existing request/response fetch path when `EventSource` is unavailable or the stream errors.
- **`plan-new` / `plan-execute` execute mapping.** Wire both from preview-only to real execute in
  `app-launcher.ts`, delegating to the P1 run-functions (no lifecycle reimplementation):
  - `plan-new` → scaffold a plan via `runPlanCreate` (`axiom-plan create`; metadata-only, not a
    state transition).
  - `plan-execute` → the plan's real transition `plan-approve` via the existing `executeSubcommand`
    (`runPlanApprove`).
  Both respect the preview/execute/**confirmed** contract: no mutation without `confirmed:true`.
- **Discard cleanup.** Remove cost-dashboard + deployment-trace from all pending/deferred records
  (P3 README long-tail, epic README/PLAN, `06_Integraciones_y_Capacidades.md`).

## Non-goals

- **Cost dashboard (copilot-usage) and deployment/trace panels: OUT of Axiom scope.** Discarded
  by owner decision 2026-07-11 ("lo que descarto queda fuera y no queda como pendiente, sale de
  axiom"). They are **not** deferred and **not** tracked as pending. A future revisit is possible
  but untracked.
- No faithful full-port of the remaining advanced launcher panels (ADO suggestions, git
  role-branch/commit-sync) — those remain PARTIAL as decided in P3; they are not part of this
  increment and remain in the epic's remaining pointer.
- No new heavy dependency; no rewrite of the `axiom app` server, the P1 subcommands, or the
  `@axiom/workflow` lifecycle.

## Acceptance

1. The server pushes an event over the push channel: an in-process test connects to
   `GET .../launcher/events`, triggers a confirmed execute, and receives a `registryChanged` push.
2. The transport shim opens the push channel and falls back to fetch when the channel is
   unavailable (asserted on the shipped shim wiring).
3. `plan-new` without `confirmed` → preview only (executable, `axiom-plan create`), no mutation;
   with `confirmed:true` → delegates to `runPlanCreate` and creates the plan artifact.
4. `plan-execute` without `confirmed` → preview only (executable, `axiom-plan approve`);
   with `confirmed:true` → delegates to the plan transition (`runPlanApprove`).
5. RISK-004 stays green: the shipped front has zero `acquireVsCodeApi` and zero `--vscode-*`.
6. Gate green: `npm run build` → `npm test` → `npm run typecheck` → `npm run doctor` in `Axiom/`.

## Result

**COMPLETE (shipped, green).** Status: `archived`.

- **Push channel — SSE (not WebSocket).** New `GET /api/projects/:id/launcher/events`
  (`text/event-stream`) on the existing `axiom app` server. A per-server `LauncherEventHub`
  (`apps/cli/src/commands/app-launcher.ts`) broadcasts to the subscribers of a project; a
  confirmed execute broadcasts `registryChanged`. Dependency-free, no 101-Upgrade handshake.
  The transport shim (`static/launcher/transport.js`) opens an `EventSource` on `configure`,
  re-dispatches pushes as `window` `message` events, and falls back to the existing fetch path
  when `EventSource` is absent or the stream errors. `launcher.js` consumes `registryChanged`
  (auto-refresh) + `pushChannel` (status).
- **`plan-new` / `plan-execute` execute mapping.** `plan-new` → `runPlanCreate` (scaffold,
  metadata-only); `plan-execute` → real `plan-approve` transition via `executeSubcommand`
  (`runPlanApprove`). Both honor the preview/execute/**confirmed** contract; a successful
  confirmed execute pushes `registryChanged`.
- **Discards removed** from all pending records: P3 README long-tail + status→archived, P3
  `metadata.yml` status→archived, epic `README.md`/`PLAN.md` P3 heading + status, and
  `06_Integraciones_y_Capacidades.md` remaining pointer. Cost dashboard + deployment trace are
  marked DISCARDED / out of Axiom scope, explicitly not deferred/pending.
- **Tests.** `apps/cli/tests/launcher-push.test.ts` (SSE push received in-process; shim
  fetch-fallback wiring; hub degradation). `apps/cli/tests/app-launcher.test.ts` updated for the
  new plan mapping. RISK-004 (`launcher-front-no-vscode.test.ts`) stays green.
- **Gate:** `npm run build` ✓ · `npm test` 2625 passed ✓ · `npm run typecheck` ✓ ·
  `npm run doctor` PASS (0 failures; IX-001 metadata validity ✓).
