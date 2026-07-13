# Increment: Epic close — ADO-suggestions + git role-branch + commit-sync panels

> **Código**: INC-20260711-epic-close-panels
> **Resuelve**: the ONLY remaining item of the epic `INC-20260711-sdd-launcher-core-port`
> (the front panels for ADO suggestions + git role-branch + commit-sync).
> **Estado**: archived
> **Fecha**: 2026-07-13

## Goal

Ship the three remaining operator panels of the re-homed sdd-launcher front — **ADO
suggestions**, **git role-branch**, and **git commit-sync** — as thin FRONT-END panels
plus thin `axiom app` server endpoints over engines that **already shipped** in earlier
phases of the epic. This is the LAST remaining item of
`INC-20260711-sdd-launcher-core-port`; **after this increment the epic is fully
delivered**.

## Scope (the 3 panels over shipped engines)

1. **ADO suggestions panel.** New `GET /api/projects/:id/launcher/ado-suggestions`
   (optional `?type=<WorkItemType>`). It builds the tracker via the shipped
   `createWorkItemTracker(config, ports)` factory (`@axiom/tracker-ado`) and returns
   suggested work items via `IWorkItemTracker.listWorkItems` (WIQL). With
   `tracker.kind != 'ado'` (→ `NullTracker`) it returns a **graceful, network-free**
   `{ configured: false, reason: "ADO no configurado…", suggestions: [] }` (NO error, NO
   network). The front panel renders the list (read-only; display-only — no link
   mutation in this slice).
2. **Role-branch panel.** New `POST /api/projects/:id/launcher/role-branch`
   (`{ branchName, confirmed? }`). Without `confirmed` → **preview**: the planned git
   command(s) + porcelain status via the shipped git-services (`gitStatus` + `roleBranch`
   `dryRun`), mutating nothing. With `confirmed:true` → applies `roleBranch` in the pinned
   repo. **NEVER pushes** (roleBranch is local-only). The front panel shows the planned
   branch + status and arms → confirms.
3. **Commit-sync panel.** New `POST /api/projects/:id/launcher/commit-sync`
   (`{ message, push?, paths?, confirmed? }`). Without `confirmed` → **preview** (status +
   planned commit, dryRun). With `confirmed:true` → commits **LOCALLY** via the shipped
   `commitSync`. Pushes **only** when the explicit `push` toggle is set **AND** confirmed
   (two guards); default is **no push**. The front panel shows status + planned commit,
   a push toggle (default off), and arms → confirms.

All three endpoints follow the established `axiom app` GET-read / POST-execute-with-
`confirmed` contract. A confirmed git apply broadcasts a `registryChanged` push over the
existing `LauncherEventHub` (SSE) so the front stays live. The three panels are removed
from the front's `deferred` advertisement (they are no longer partial).

## Non-goals

- **No new engine.** ADO suggestions reuse `createWorkItemTracker` + `listWorkItems`; the
  git panels reuse `@axiom/workflow`'s `roleBranch`/`commitSync`/`gitStatus` (the same
  functions the `sdd.gitRoleBranch` / `sdd.gitCommitSync` MCP handlers wrap). No git code
  and no tracker code is re-implemented.
- **Cost dashboard (copilot-usage) and deployment/trace panels: OUT of Axiom scope.**
  DISCARDED by owner decision 2026-07-11 — not revived, not deferred, not tracked as
  pending.
- No confirm-gated ADO "link" action in this slice (display-only suggestions, bounded).
- No push unless the explicit push toggle is set AND the mutation is confirmed.
- No rewrite of the `axiom app` server, the transport shim, or the shipped engines.

## Acceptance

1. **ADO suggestions — NullTracker path.** `GET .../launcher/ado-suggestions` with
   `tracker.kind: none` returns `{ configured: false, reason: <"ADO no configurado"…>,
   suggestions: [] }`, status 200, and makes NO network call.
2. **ADO suggestions — real ado tracker.** With a fake `ado` tracker (injected fake HTTP
   transport, the `@axiom/tracker-ado` test pattern) the endpoint returns the WIQL
   suggestions (id + title + url).
3. **Role-branch.** POST without `confirmed` → preview (planned `git checkout -b …` +
   status), no branch created. POST with `confirmed:true` in a temp git repo → the branch
   is created/checked out and **no push** ran. A non-git dir → a clear `not-a-repo` error,
   not a crash.
4. **Commit-sync.** POST without `confirmed` → preview (planned `git add`/`git commit` +
   status), no commit. POST with `confirmed:true` in a temp git repo → a **local** commit,
   `pushed: false`. Push happens only with `push:true` AND `confirmed`.
5. **No-VSCode-API.** RISK-004 stays green: the shipped front (including the new panel
   assets) has zero `acquireVsCodeApi` and zero `--vscode-*`.
6. **Gate green:** `npm run build` → `npm test` → `npm run typecheck` → `npm run doctor`
   in `Axiom/`, with no existing test weakened.

## Result

**COMPLETE (shipped, green). This CLOSES the epic `INC-20260711-sdd-launcher-core-port`
— all three remaining panels ship; nothing is left partial.** Status: `archived`.

- **3 panels + thin endpoints, each over an ALREADY-SHIPPED engine (no new engine):**
  - **ADO suggestions** — `GET /api/projects/:id/launcher/ado-suggestions?type=…`
    (`apps/cli/src/commands/app-launcher-panels.ts` → `apiAdoSuggestions`). Reuses the
    shipped `createWorkItemTracker` factory (`@axiom/tracker-ado`) + `IWorkItemTracker.
    listWorkItems` (WIQL). `tracker.kind != 'ado'` → `NullTracker` → a **network-free**
    `{ configured:false, reason:"ADO no configurado…", suggestions:[] }` (NOT an error).
    Display-only list (no link mutation — bounded).
  - **Role-branch** — `POST /api/projects/:id/launcher/role-branch` (`apiRoleBranch`).
    Reuses `@axiom/workflow` `roleBranch` + `gitStatus`. Preview (dryRun) without
    `confirmed`; applies in the pinned repo with `confirmed:true`. **Never pushes.**
  - **Commit-sync** — `POST /api/projects/:id/launcher/commit-sync` (`apiCommitSync`).
    Reuses `@axiom/workflow` `commitSync` + `gitStatus`. Preview without `confirmed`;
    commits **LOCALLY** with `confirmed:true`; pushes ONLY with `push:true` AND
    `confirmed` (two guards; **no-push-by-default**). A confirmed git apply broadcasts
    `registryChanged` over the existing `LauncherEventHub` (SSE).
- **Routes** wired in `apps/cli/src/commands/app-api.ts` following the launcher route
  pattern (GET-read / POST-execute-with-`confirmed`). The three panels are removed from
  the front's `deferred` advertisement (`app-launcher.ts` → `deferred: []`).
- **Front:** new `apps/cli/static/launcher/panels.js` + panel DOM in `index.html` +
  styles in `launcher.css`, wired through the EXISTING transport shim (`transport.js`
  gains the `loadAdoSuggestions` / `previewRoleBranch` / `createRoleBranch` / `commitSync`
  message types → the panel endpoints). **Zero VSCode APIs** (RISK-004): the shipped
  front (including `panels.js`) has no `acquireVsCodeApi` and no `--vscode-*`.
- **Tests:** `apps/cli/tests/launcher-panels.test.ts` (14 tests — NullTracker reason,
  fake-`ado`-tracker suggestions via injected fake transport, role-branch/commit-sync
  preview→confirm in TEMP git repos asserting NO push, non-git errors, 400s). RISK-004
  test extended to cover `panels.js` + the new transport routes.
- **Real e2e** (built dist server, scratch home + temp git repo): ADO suggestions
  (NullTracker → configured:false), role-branch preview→confirm (branch `role/e2e-be`
  created, no remote), commit-sync preview→confirm (local commit `e2e: work`, pushed:
  false, no remote). All green.
- **Gate:** `npm run build` ✓ · `npm test` **2731 passed** (270 files) ✓ ·
  `npm run typecheck` ✓ · `npm run doctor` **PASS** (0 failures; IX-001 metadata
  validity ✓). No existing test weakened.
