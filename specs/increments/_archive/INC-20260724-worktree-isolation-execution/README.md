# Increment: Per-worktree isolation + `Execution` entity

Status: closed
Date: 2026-07-24

## Goal

Introduce (a) per-worktree/per-execution scoped isolation paths and (b) a
first-class `Execution` entity so Axiom can track parallel worktree runs.
This increment MODELS + PERSISTS the entity and the scoped paths only; it
does NOT do provisioning (W3), provider isolation (W4), or cleanup
orchestration (W5) — its API is what those build on.

This is **INC-W2** of Cluster W in the Axiom evolution plan
(`giggly-imagining-moore.md`, "Cluster W — worktrees"), depending on **INC-W1**
(`INC-20260724-git-worktree-services`, closed — `worktreeAdd`/`worktreeRemove`
in `@axiom/workflow`'s git module). It is an explicit, user-approved
graduation to the full product lifecycle (per `Axiom.SDD/AGENTS.md`'s
"Explicit Bootstrap Limits" — worktree support is a deliberate exception this
plan requests, not speculative architecture).

## Context

Today, `packages/isolation/src/p0.ts` provides ONLY project-scoped isolation:
`buildProjectScopedPaths(projectName, rootPath)` computes
`.axiom-state/{memory,mcp,config,outputs}/<projectName>` (+ a repo-level
shared `.axiom-state/local/`), and `assertProjectIsolation` / `pathsAreIsolated`
guard/verify it. This is keyed by PROJECT, not by execution — it has no
concept of "two increments running in parallel worktrees of the same
project need their own isolated state".

`@axiom/persistence` already declares a dependency on `@axiom/isolation`
(package.json + tsconfig references) even though nothing in its `src/` used
it yet — clean forward-plumbing for exactly this kind of extension.
`@axiom/persistence`'s own `FileSystemStore` (`filesystem-store.ts`) writes
directly (no tmp+rename); the ACTUAL atomic tmp+rename convention this
increment must mirror lives in `@axiom/topology/src/loader.ts`
(`saveLocalBindings`) and `@axiom/installer/src/persist.ts`
(`writeInstallProfile`): `mkdirSync(dir,{recursive:true})` →
`writeFileSync(tmp)` → `renameSync(tmp, final)`, wrapped in `Result<T,E>`.

`@axiom/workflow/src/artifact-id.ts`'s `generateArtifactId`/
`generateUniqueArtifactId` (timestamp+hash id, injectable clock + random
suffix, `<PREFIX>-<YYYYMMDD>-<HHMMSS>-<6-char-suffix>`) is the ID convention
to MIRROR — not import, because its `ArtifactKind` union
(`increment|bug|plan|adr|decision`, and a SEPARATE, differently-scoped
`ArtifactKind` union in `@axiom/core`) is a CLOSED set of CANONICAL,
folder-per-instance artifacts (`metadata.yml` under `axiom.spec/`).
`Execution` is explicitly NOT canonical (see Decisions below), so extending
either `ArtifactKind` union would misstate what an Execution is.

`@axiom/tracker`'s `IWorkItemTracker` is the existing "register/list
entities" precedent named in the brief — but it models EXTERNAL work items
(Azure DevOps User Story/Bug/Task), with fields (`assignedTo`, `tags`,
`priority`, `severity`, `iterationPath`, `estimate`) that have no meaning for
a local execution record, and a `NullTracker` default that performs ZERO
I/O (no on-disk store precedent to reuse at all). `@axiom/tracker/tests/
local-id.test.ts` ("IDs are purely local, tracker-independent") is existing,
explicit precedent in this codebase for keeping local-only concerns OUT of
the tracker abstraction.

## Scope

- `packages/isolation/src/execution-paths.ts` (new): `ExecutionScopedPaths`,
  `buildExecutionScopedPaths(executionId, rootPath)`, `executionScopeRoot`,
  `assertExecutionIsolation(executionId)`, `executionPathsAreIsolated(a, b)`,
  `EXECUTIONS_DIRNAME` constant. Pure, no I/O (mirrors `p0.ts`).
- `packages/isolation/src/execution.ts` (new): `Execution` entity type,
  `ExecutionState` (7-value closed enum) + `EXECUTION_STATES` +
  `isValidExecutionState`, `ExecutionArtifactRef`/`ExecutionArtifactKind`,
  `generateExecutionId`/`generateUniqueExecutionId` (mirrors
  `artifact-id.ts`, own `EXE` prefix). Pure, no I/O.
- `packages/isolation/src/index.ts`: barrel exports for both new modules.
- `packages/isolation/tests/execution-paths.test.ts`,
  `packages/isolation/tests/execution.test.ts` (new).
- `packages/persistence/src/execution-store.ts` (new): `ExecutionStore`
  (create/get/list/update/close), `createExecutionStore(rootPath, options)`,
  atomic tmp+rename I/O, `ExecutionStoreError`.
- `packages/persistence/src/execution-worktree.ts` (new, OPTIONAL per brief):
  `createExecutionForWorktree` — thin compose helper (`worktreeAdd` +
  `ExecutionStore.create`) for W3 to call. NOT wired into any command.
- `packages/persistence/src/index.ts`: barrel exports for both new modules.
- `packages/persistence/package.json` + `tsconfig.json`: add `@axiom/workflow`
  dependency (needed only by the optional compose helper).
- `packages/persistence/tests/execution-store.test.ts`,
  `packages/persistence/tests/execution-worktree.test.ts` (new).

## Non-goals

- Provisioning the portable `.axiom` surface into a worktree — INC-W3.
- Per-worktree provider/index isolation (`serena`/`cmm`) — INC-W4.
- Harvest + safe cleanup orchestration (kill processes → harvest → remove)
  — INC-W5. `ExecutionStore.close()` is a SOFT close (state → `removed`);
  it never deletes `execution.json` or the scoped folders.
- Execution-mode selection (in-place vs worktree), install-time default —
  INC-W6.
- Freshness / auto-fetch / staleness detection — INC-W7.
- Wiring worktree CREATION into any CLI command or MCP tool. The
  `worktreeAdd`/`worktreeRemove` call sites, and the compose helper's actual
  caller, are later increments' job.
- A full state-machine / transition-order enforcement over `ExecutionState`
  (e.g. rejecting `created → harvested` as an illegal skip). `update()`
  validates that a given `state` is one of the 7 valid VALUES, but does not
  enforce any transition ORDER — the increments that actually drive
  transitions (W3..W7) own that policy, avoiding speculative architecture
  ahead of real callers.
- Registering `Execution` through `@axiom/tracker` — deliberately NOT done
  (see Decisions/Assumptions).
- Migrating Axiom's own repos (`Axiom.SDD`/`Axiom.Spec`) to any worktree
  model — out of scope for the whole plan.

## Acceptance criteria

- [x] `buildExecutionScopedPaths(executionId, rootPath)` produces
      `.axiom-state/executions/<executionId>/{config,mcp,outputs,logs,evidence,local}`,
      distinct per `executionId` for the same `rootPath`.
- [x] Execution-scoped paths never collide with project-scoped paths
      (`buildProjectScopedPaths`) for the same `rootPath`, for ANY project
      name (including a project literally named `"executions"`) — verified
      structurally (distinct top-level segment) and by test.
- [x] `buildProjectScopedPaths` / project-scope isolation behavior is
      UNCHANGED (additive-only change to `@axiom/isolation`).
- [x] `assertExecutionIsolation` throws on an empty, path-traversal, or
      separator/forbidden-character `executionId`; does not throw on a
      well-formed one.
- [x] `Execution` entity has exactly the fields specified: `id`, `projectId`,
      `artifactRef` (`{kind:'increment'|'bug'|'plan', id}`), `repoId`,
      `branch`, `worktreePath`, `state` (7-value enum), `agentTarget?`,
      `capabilities?`, `logsPath`, `evidencePath`, `createdAt`, `updatedAt`.
      All timestamps come from an injectable clock (never `Date.now()`
      invoked directly in the store).
- [x] `ExecutionStore.create/get/list/update/close` round-trip correctly via
      atomic tmp+rename writes; `create` generates a unique id (retries on
      collision); `update` validates `state` against the 7-value enum and
      rejects unknown values; `close` performs a SOFT close (state →
      `removed`, record NOT deleted).
- [x] Two executions for two different increments are fully isolated:
      distinct folders, distinct `logsPath`/`evidencePath`, updating one
      never affects the other's persisted state.
- [x] `list()` tolerates an orphaned execution folder (missing/corrupt
      `execution.json`) by skipping it, not by failing the whole call.
- [x] `.axiom-state/executions/` is local runtime state, never canonical,
      never committed — covered by the repo's EXISTING blanket
      `.axiom-state/` `.gitignore` entry (verified; no `.gitignore` or
      doctor change needed — see Implementation notes).
- [x] `npm run build` (`tsc -b`) passes.
- [x] Targeted `vitest run` for `packages/isolation`, `packages/persistence`,
      `packages/tracker` passes; failures classified pre-existing vs new.
- [x] `@axiom/cli-commands` single-ownership — N/A, no CLI files touched.
      `@axiom/tui` stayed generic — not touched.
- [x] No worktree creation wired into any command (W3/W6 territory) — the
      optional compose helper exists but has no caller in this increment.

## Assumptions

Each is a deliberate, narrowest-change resolution of an ambiguity the brief
left open (existing-precedent-first, per the "resolve ambiguity yourself"
guardrail).

1. **Isolation-owned storage, NOT `@axiom/tracker`** (the brief's explicit
   escape hatch, taken): `Execution` is local runtime state, not an external
   work item; `IWorkItemTracker`'s shape doesn't fit, `NullTracker` performs
   zero I/O (no store precedent there), and `local-id.test.ts` is existing,
   explicit precedent for keeping local-only IDs/state out of the tracker.
   `@axiom/tracker` is left completely untouched.
2. **`Execution` entity + id generator live in `@axiom/isolation` (pure, no
   I/O); the actual `ExecutionStore` CRUD (disk I/O) lives in
   `@axiom/persistence`**: mirrors the package's OWN existing shape exactly
   — `p0.ts` (isolation) is 100% pure path/assertion logic today, zero `fs`
   calls; `@axiom/persistence` already depends on `@axiom/isolation` (a
   pre-wired, currently-unused dependency edge this increment is the first
   to actually use) and is where the ONLY real `FilesystemStore`
   implementation already lives.
3. **`generateExecutionId` mirrors (duplicates), not imports,
   `generateArtifactId`**: adding `'execution'` to either existing
   `ArtifactKind` closed union (`@axiom/workflow` or `@axiom/core`) would
   incorrectly imply Executions are canonical folder-per-instance artifacts
   — the opposite of the "never canonical" decision. A small, independent,
   pure generator (`EXE-<date>-<time>-<suffix>`) with the identical
   injectable-clock/suffix shape avoids that without adding a dependency.
4. **`execution.json` lives at the ROOT of
   `.axiom-state/executions/<id>/`** (a peer of the `{config,mcp,outputs,
   logs,evidence,local}` subfolders), mirroring exactly where
   `install-profile.json` sits (`.axiom-state/<projectName>/install-profile.json`
   — a peer of nothing else in that dir, not nested in a further subfolder).
5. **`list()` scans the `executions/` directory directly** (no separate
   `index.json` registry file): mirrors `FilesystemStore.list()`'s
   directory-scan-and-filter shape (the actual "list entities" precedent to
   mirror per the brief), and avoids a second, independently-maintainable
   source of truth that could drift out of sync with the per-execution files
   themselves (a real risk with a hand-maintained index at MVP scale/write
   patterns).
6. **`close()` = soft-close = `update(id, { state: 'removed' })`**: the
   brief's field list for `Execution` has no separate `closedAt`/`deleted`
   flag, so "close" is modeled as a transition to the existing terminal
   `removed` state, NOT a filesystem delete (hard delete/cleanup is
   Cluster W5's job). Idempotent (closing an already-`removed` execution just
   re-writes the same state + bumps `updatedAt`).
7. **`update()` validates `state` VALUES but not transition ORDER**: the
   brief says "define the set; later increments transition through it" —
   defining + validating membership is this increment's job; the increments
   that actually drive transitions (W3..W7) are better positioned to own
   ordering/guard policy once they exist, avoiding a speculative state
   machine with no real caller yet.
8. **`localOverlay` (the `local` subfolder) IS included in
   `executionPathsAreIsolated`'s comparison** — unlike `pathsAreIsolated`
   (project-scope), which deliberately EXCLUDES it because
   `.axiom-state/local/` is a single REPO-LEVEL shared directory by design.
   The execution-scope layout nests `local` INSIDE each execution's own
   folder (`.axiom-state/executions/<id>/local`), so it is exactly as
   isolated as any other subfolder there — a deliberate, documented
   divergence from the project-scope precedent, not an oversight.
9. **The optional compose helper (`createExecutionForWorktree`) lives in
   `@axiom/persistence`** (which adds a new `@axiom/workflow` dependency),
   not in `@axiom/workflow` or `@axiom/isolation`: it needs BOTH `worktreeAdd`
   (workflow) AND `ExecutionStore.create` (persistence); persistence already
   owns the store, and workflow has no persistence/isolation dependency
   today (adding the edge the other direction would be an equally arbitrary
   choice, but persistence-owning-the-store makes it the natural caller of
   its own `create`). No cycle: `@axiom/workflow` depends on neither
   `@axiom/persistence` nor `@axiom/isolation`.
10. **No doctor / `.gitignore` change**: `.gitignore` already has a blanket
    `.axiom-state/` entry (not scoped to `.axiom-state/local/`), and no
    doctor check enumerates `ProjectScopedPaths`' fields exhaustively (the
    only doctor/isolation coupling found, `CATEGORY_ISOLATION`, is an
    unrelated provider-registry field). Verified by search before concluding
    no update was needed.
11. **`agentTarget`/`capabilities`/`projectId`/`repoId` stay plain `string`
    (not `@axiom/core`'s branded `ProjectId`/`AdapterTargetId`/`CapabilityId`)**:
    consistent with `@axiom/isolation`'s OWN existing convention
    (`IsolationContext.projectName: string`, not branded) — introducing
    branding here would be inconsistent within the very same package/file
    family, for no behavioral benefit at this stage (no untrusted external
    input parses into these fields yet; MCP/CLI surfaces that would need it
    are later increments).

## Implementation notes

Files changed (all in `Axiom/`, the product monorepo):

- `packages/isolation/src/execution-paths.ts` (new) — scoped-path layout +
  `assertExecutionIsolation` + `executionPathsAreIsolated`.
- `packages/isolation/src/execution.ts` (new) — `Execution` type, state enum,
  id generator.
- `packages/isolation/src/index.ts` — barrel wiring for both.
- `packages/isolation/tests/execution-paths.test.ts`,
  `packages/isolation/tests/execution.test.ts` (new).
- `packages/persistence/src/execution-store.ts` (new) — `ExecutionStore`
  (filesystem impl, tmp+rename, orphan-tolerant `list()`).
- `packages/persistence/src/execution-worktree.ts` (new) — optional compose
  helper.
- `packages/persistence/src/index.ts` — barrel wiring for both.
- `packages/persistence/package.json`, `packages/persistence/tsconfig.json`
  — added `@axiom/workflow` dependency/reference (compose helper only).
- `packages/persistence/tests/execution-store.test.ts`,
  `packages/persistence/tests/execution-worktree.test.ts` (new).

### Design decisions (one-line reasons)

- **`EXECUTIONS_DIRNAME`/`.axiom-state` hardcoded as a literal in
  `execution-paths.ts`**, not imported from `@axiom/filesystem-truth`'s
  `LOCAL_OVERLAY_DIRNAME` (even though it's the identical string): matches
  `p0.ts`'s OWN existing convention in the SAME package/file family
  (`buildProjectScopedPaths` also hardcodes the literal) rather than mixing
  two sources of truth for one value within one package.
- **`ExecutionStoreOptions.clock` feeds `generateUniqueExecutionId`'s `now`
  option internally**: the store-level option is named for what it is
  (the clock backing `createdAt`/`updatedAt` too), while the pure
  id-generator keeps `artifact-id.ts`'s exact original option name (`now`)
  — no naming conflict, just a rename at the one call boundary.
- **`UpdateExecutionPatch` excludes `id`/`projectId`/`artifactRef`/`repoId`/
  `createdAt`/`logsPath`/`evidencePath`**: these are identity/provenance
  fields fixed at `create()` time; only `state`/`branch`/`worktreePath`/
  `agentTarget`/`capabilities` are patchable.
- **`list()` catches per-entry read/parse failures and skips that entry**
  rather than failing the whole call: this is the narrow "orphaned state"
  robustness the plan's edge-case list calls for, WITHOUT implementing any
  actual cleanup/deletion (that stays Cluster W5's job).

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`. See the executor's final report
for verbatim command tails; summary:

- `npm run build` (`tsc -b`) — passed.
- `npx vitest run packages/isolation packages/persistence packages/tracker`
  — all passed (new + pre-existing); no pre-existing failures were touched
  by this change.
- New unit tests: execution-scoped paths distinct per `executionId` +
  never collide with project scope; `assertExecutionIsolation` guard cases;
  `Execution` CRUD round-trip (create/get/list/update/close); two isolated
  executions; orphaned-folder tolerance in `list()`; id uniqueness
  (collision-retry, both at the pure-generator level and threaded through
  the real filesystem store); injectable clock threaded through
  `createdAt`/`updatedAt` independently of `id` generation; compose-helper
  happy path, dry-run (no Execution persisted), and git-failure propagation
  (no Execution persisted).

## Result

Implemented. `@axiom/isolation` gained an additive, pure execution-scoped
path layout + isolation assertion + the `Execution` entity type (with its
own mirrored id generator) — `buildProjectScopedPaths`/project isolation are
untouched. `@axiom/persistence` gained the actual `ExecutionStore` CRUD
(atomic tmp+rename, orphan-tolerant listing) plus an optional, unwired
compose helper for W3. `@axiom/tracker` was deliberately left untouched
(isolation-owned storage is the documented, precedent-backed choice). No
worktree creation was wired into any command — that remains W3/W6 territory.

## General spec integration

Integrado en la spec canónica al cierre del batch INC-20260724-* (integración
única de toda la tanda). Ficheros que recibieron el conocimiento de este
incremento:

- `03_Modelo_Operativo_y_Datos.md` — entidad `Execution` + `ExecutionStore` +
  paths execution-scoped (`.axiom-state/executions/<id>/…`), no canónica/no
  commiteada.
- `08_Glosario.md` — `Execution` / `ExecutionStore`.
- `02_Requisitos_No_Funcionales.md` — NFR-AXM-019 (aislamiento por worktree).
- `01_Requisitos_Funcionales.md` — RF-AXM-047.
