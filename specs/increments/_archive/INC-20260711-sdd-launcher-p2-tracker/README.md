# Increment P2 — `IWorkItemTracker` port + `NullTracker` + `@axiom/tracker-ado`

> **Código**: INC-20260711-sdd-launcher-p2-tracker
> **Implementa**: Phase **P2** of the epic `INC-20260711-sdd-launcher-core-port`
> **Estado**: archived (green gate; P2 acceptance core met)
> **Fecha**: 2026-07-11
> **Fuente (read-only)**: `C:/repos/KVP25 Workspace/Kvp.Sdd/tools/sdd-launcher/src/services/adoWorkflowService.ts` (2211 LOC) + `src/config/launcherProject.ts` (ADO config model).

## Goal

Make the work-item tracker a **decoupled, optional plugin**. Axiom runs
**local-only by default** (`NullTracker`); Azure DevOps is opt-in and fills the
existing declarative stub (`app-plugins-azure-devops.ts` + the
`axiom external-sync azure-devops …` commands). Swapping trackers must be
**config-only** — no core workflow code path branches on tracker kind.

## Scope (incluido)

- **P2.1** — New package `@axiom/tracker`: the provider-agnostic
  `IWorkItemTracker` interface, the `SecretStore` + `promptForSecret` ports,
  a **total no-op `NullTracker`** (the DEFAULT), a pure `TrackerConfig`
  resolver, and re-use of the existing `ExternalRef` shape (imported from
  `@axiom/workflow`) as the only persisted tracker state.
- **P2.2** — New package `@axiom/tracker-ado`: `AdoWorkItemTracker` implements
  `IWorkItemTracker`, porting the lifecycle-critical WIT surface of
  `adoWorkflowService.ts` behind an injected `HttpTransport` (default
  `NodeHttpsTransport`, raw `https` + `_apis` v7.1). The 2 VSCode touches
  (`context.secrets`, `showInputBox`) are replaced by the injected `SecretStore`
  + `promptForSecret`. A guard test asserts **no `vscode` import** (RISK-002).
  A `createWorkItemTracker(config, ports)` factory is the config→impl swap point.
- **P2.3** — Wire the tracker behind the declarative surface: the new
  `axiom external-sync azure-devops <create|list|mapping|mapping-set>` command
  drives `IWorkItemTracker`. Config `tracker: { kind, enabled, org, project,
  mappings }` (default `kind: none`) selects `NullTracker` vs `AdoWorkItemTracker`
  via the factory. No core workflow path changes on kind.
- **P2.4** — Confirm `generateArtifactId` already yields tracker-independent
  IDs, so `NullTracker` mode needs no ADO id (a test asserts this).

## Non-goals (excluido)

- No live network. All ADO tests use an in-memory fake `HttpTransport` /
  recorded-fixture doubles; **never** `dev.azure.com`.
- No rewrite of the pure `@axiom/workflow` state machine — it stays
  tracker-agnostic; the tracker is an injected side-effect, per RECONCILIATION.
- The **peripheral** ADO surface is NOT fully ported (see Result → PARTIAL).
- No front-end (P3), no prompt engine (P4).

## Acceptance (from PLAN.md P2)

- With `tracker.kind: none`, a full lifecycle completes end-to-end with **no
  network and no ADO id**; artifacts carry an **empty `externalRefs`**.
- With `tracker.kind: ado` (fake transport), the four mappings —
  `increment→User Story`, `bug→Bug`, `role→child Task`, `e2e→US+Task` — create
  and link work items and populate `ExternalRef`s. Auth resolution order
  (env → SecretStore → prompt) is unit-tested with fakes.
- Swapping trackers requires **only** config (assert via a test): the factory
  returns `NullTracker` for `none` and `AdoWorkItemTracker` for `ado`, and no
  core code path branches on kind.
- `@axiom/tracker-ado` has **no `vscode` import** (guard test).

## Ports

- **`IWorkItemTracker`** (provider-agnostic) — `createWorkItem`, `getWorkItem`,
  `updateWorkItem`, `transitionWorkItem`, `linkWorkItems`, `createChildTask`,
  `listWorkItems`, plus the 4 mapping helpers `mapIncrementToUserStory`,
  `mapBugToBug`, `mapRoleToTask`, `mapE2eToUsAndTask`. Persisted state is
  `ExternalRef { provider, type, id, url }` (reused from `@axiom/workflow`).
- **`SecretStore`** — `get(key)` / `store(key, value)`. Ships `EnvSecretStore`
  (env-backed read; store is a no-op) as the CLI-safe default.
- **`promptForSecret(opts)`** — terminal/HTTP prompt seam. Ships a
  non-interactive default (`noopPromptForSecret`) that returns `undefined`.
- **`HttpTransport`** (tracker-ado) — `request({ method, hostname, path,
  headers, body })`. Default `NodeHttpsTransport` uses raw `https.request`
  (faithful to the source). Tests inject a fake transport → no network.
- **`createWorkItemTracker(config, ports)`** (tracker-ado) — the config→impl
  swap point.

## Result

See the "Result" section at the bottom (FULL vs STUB and gate outcome), filled
after implementation.

---

### Result — what is FULL vs STUB

**FULL (faithfully ported, tested):**

- Auth resolution `resolvePat`: `env[patEnvVar]` → (Windows user-scoped env,
  injectable) → `SecretStore.get(secretKey)` → `promptForSecret` (then persisted
  to the SecretStore). Basic-auth header `Basic base64(":" + PAT)`.
- Raw `https` request layer (`requestJson`) against `dev.azure.com`
  `/{org}/{project}/_apis/... api-version=7.1`, behind the injected
  `HttpTransport` (default `NodeHttpsTransport`). `allowNotFound` → `undefined`.
- WIT work-item **create** (`POST _apis/wit/workitems/${type}`, JSON-patch:
  Title/State/AssignedTo/Tags/Priority/Severity/IterationPath/Description/
  ReproSteps/Estimate + parent `Hierarchy-Reverse`), **get** (`getWorkItemById`),
  **update** (PATCH fields), **state transition** (`transitionWorkItemToState`
  with optional reason/assignedTo/description), **related + child links**
  (`System.LinkTypes.Related` / `Hierarchy-Reverse`, dedup via relations read),
  **WIQL query** (`listWorkItems` by type, then batched field fetch).
- The 4 mappings: `increment→User Story`, `bug→Bug`, `role→child Task`
  (`ensureRoleTask` with title template + parent), `e2e→US+Task`
  (`ensureE2eWorkItems`: US under feature parent + related link to origin +
  child Task).
- `NullTracker` total no-op; `createWorkItemTracker` config swap; local-only ID
  independence (`generateArtifactId`).

**STUB / PARTIAL (deferred, with reason):**

- **Build definitions/artifacts/runs** (`getBuildDefinitionsByNames`,
  `getCompletedBuildsForDefinition`, `getBuildArtifacts`, `getBuild{WorkItems,
  Changes}`) — peripheral CI traceability, not on the increment/bug/role/e2e
  lifecycle. Deferred to keep the green gate and avoid ballooning the port.
- **Git commit/PR traceability + branch artifact links** (`getCommitWorkItems`,
  `getPullRequestsByCommit`, `getPullRequestWorkItems`, `addBranchArtifactLink`,
  `resolveGitRepository`) — these belong with P3's git-services port
  (`roleBranch`/`commitSync`/`gitSync`), not P2's tracker.
- **Sprint/iteration catalog + role-close scheduling** (`getProjectSprints`,
  `readSchedulingSnapshot`, `buildRoleCloseSchedulingOperations`,
  `resolveCreationValues` sprint enrichment) — the create path accepts an
  explicit `iterationPath`/`estimate`; automatic sprint discovery is deferred.
- **Attachments / repro-image upload** (`uploadInlineAttachments`,
  `downloadAttachment`, `getWorkItemDocumentSeed`, repro-image HTML) — the
  `ReproSteps`/`Description` HTML text fields ARE ported; inline image upload is
  deferred (needs the binary multipart path, peripheral to lifecycle).
- **Tag/feature/user-story catalogs & severity value lookups**
  (`getProjectTags`, `getProjectFeatures`, `getProjectUserStories`,
  `getBugSeverityValues`, `getWorkItemDetailsBatch`) — form-population helpers
  for the front-end (P3/P4), not lifecycle mutations.

**Gate (all green in `Axiom/`):**

- `npm run build` → PASS (`tsc -b`; the 2 new packages compile).
- `npm test` → **2477 passing** (238 files); baseline 2431 + 46 new P2 tests, no
  existing test weakened.
- `npm run typecheck` → PASS.
- `npm run doctor` → **PASS** (45/57 OK · 0 FAIL · 3 warn · 9 skip). The new
  packages do not trip any coverage check: `TC-009` (adapter-runtime-coverage)
  targets a fixed list under `packages/adapters/` (the trackers live elsewhere),
  and `PS-001` (provider-selection) / `DF-001` (dogfooding) pass unchanged.

**Wiring:** `packages/tracker` + `packages/tracker-ado` added to the root
`package.json` workspaces (auto via `packages/*`; `npm install` links them),
root `tsconfig.json` references, root `vitest.config.ts` aliases, and
`apps/cli`'s `package.json` deps + `tsconfig.json` paths/references. The
`axiom external-sync` command is registered in `apps/cli/src/index.ts`.
