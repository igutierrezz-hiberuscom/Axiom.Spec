# Increment: Worktree mode selection (execution-mode default + CLI/launcher wiring)

Status: closed
Date: 2026-07-24

## Goal

Let a user run an increment/bug/plan implementation EITHER in-place (today's
behavior) OR in a dedicated git worktree, with the DEFAULT chosen by the
architect at install time and overridable per run — wiring together **W1**
(`worktreeAdd`/`worktreeRemove`, `INC-20260724-git-worktree-services`), **W2**
(`Execution`/`ExecutionStore`/`createExecutionForWorktree`,
`INC-20260724-worktree-isolation-execution`), **W3**
(`provisionWorktreeExecution`, `INC-20260724-worktree-provisioning`), and
**W5** (`harvestAndCleanupExecution`, `INC-20260724-worktree-harvest-cleanup`).

This is **INC-W6** of Cluster W in the Axiom evolution plan
(`giggly-imagining-moore.md`, "Cluster W — worktrees") — the USER-FACING
WIRING increment that makes worktree execution actually usable, depending on
W1 and W3 (per the plan) and, transitively, on W2/W5 (the compose helpers
this increment is the first real caller of). It is an explicit, user-approved
graduation to the full product lifecycle (per `Axiom.SDD/AGENTS.md`'s
"Explicit Bootstrap Limits" — worktree support is a deliberate exception this
plan requests, not speculative architecture).

## Context

W1–W5 built every primitive needed for worktree-mode execution but
deliberately left them UNWIRED, each with a comment along the lines of "INC-W6
wires the user-facing call site":

- `createExecutionForWorktree` (`@axiom/persistence`) composes `worktreeAdd`
  (`@axiom/workflow`) + `ExecutionStore.create`.
- `provisionWorktreeExecution` (`apps/cli/src/commands/workspace-worktree-provision.ts`)
  materializes the portable `.axiom` surface + MCP config + execution-scoped
  `.axiom-state` layout into a worktree.
- `harvestAndCleanupExecution` (`@axiom/persistence`) orchestrates
  kill-processes → harvest → teardown → `worktreeRemove`, in that strict
  order, never forcing a dirty worktree by default.
- `@axiom/workflow`'s `renderBranchName`/`GIT_ACTION_HANDLERS['role-branch']`
  already compute the in-place branch name (`role/<id>-<slug>` by default)
  used by `axiom-role start --create-branch`.

None of these had a caller from a real user-facing command. Separately,
`axiom-role.ts`'s `runRoleSubcommand` was fully synchronous, and
`ResolvedInstallProfile` (`@axiom/install-profiles`, persisted at
`.axiom-state/<projectId>/install-profile.json` by `@axiom/installer`'s
`installProfile`, written by `axiom configure`) had no notion of an
execution-mode default at all.

## Scope

- `packages/install-profiles/src/{types,constants,composer,index}.ts`: new
  `ExecutionMode = 'in-place' | 'worktree'` type, `EXECUTION_MODES`,
  `DEFAULT_EXECUTION_MODE` ('in-place'), and a new required
  `ResolvedInstallProfile.executionMode` field (composer always defaults it —
  see Assumption 1).
- `packages/installer/src/{types,installer}.ts`: `InstallArgs.executionMode?`
  + `resolveExecutionModeForInstall` (explicit arg → preserve the value
    already persisted in a previous `install-profile.json` → default) + a new
  `InstallerError` kind `invalid-execution-mode`.
- `apps/cli/src/commands/configure.ts`: `--execution-mode <in-place|worktree>`
  flag, forwarded to `installProfile`.
- `apps/cli/src/commands/_execution-mode.ts` (new): `resolveExecutionMode`/
  `readInstallTimeExecutionMode` — the CLI/launcher-side "brain" that resolves
  the effective mode (override → install default → `'in-place'`).
- `apps/cli/src/commands/_worktree-execution.ts` (new): `buildWorktreeBranchName`,
  `buildUniqueWorktreePath`, `resolveRepoIdentityForExecution`,
  `startWorktreeExecution` (composes `createExecutionForWorktree` +
  `provisionWorktreeExecution`), `closeWorktreeExecution` (idempotent wrapper
  over `harvestAndCleanupExecution`).
- `apps/cli/src/commands/axiom-role.ts`: `RoleSubcommandArgs.executionMode`
  (input) + `RoleSubcommandResult.executionMode`/`.executionId` (output,
  surfaced from `vars`) on the existing SYNCHRONOUS `runRoleSubcommand`
  (unchanged signature/behavior for every existing caller); a NEW exported
  ASYNC `runRoleSubcommandAsync` that composes the worktree start/close flow
  around it; `--worktree`/`--in-place` CLI flags on `start`.
- `packages/launcher/src/action-catalog.ts`: an optional `executionMode`
  select field (`''|'in-place'|'worktree'`) on the `back-new`/`front-new`
  role-start actions only (NOT `e2e-new`, which maps to the separate `qa-e2e`
  workflow).
- `apps/cli/src/commands/app-launcher.ts`: `LAUNCHER_ACTION_COMMAND`'s
  `back-new`/`front-new` `needs` + `buildCliArgs` + `buildLauncherCommandPreview`
  translate the field into `--worktree`/`--in-place` in the PREVIEW command
  only (see Assumption 4 — the real execute path is untouched).
- `apps/cli/src/commands/sync.ts` + `apps/cli/src/index.ts`: added
  `executionMode: 'in-place'` to the two pre-existing MINIMAL fallback
  `ResolvedInstallProfile` object literals (required by the new field; no
  behavior change — those loaders never read `executionMode`).
- New tests: `packages/install-profiles/tests/execution-mode.test.ts`,
  `packages/installer/tests/execution-mode.test.ts`,
  `apps/cli/tests/execution-mode.test.ts`,
  `apps/cli/tests/worktree-execution.test.ts`,
  `apps/cli/tests/axiom-role-worktree.test.ts`,
  `apps/cli/tests/e2e/axiom-role-worktree.e2e.test.ts`,
  `apps/cli/tests/launcher-execution-mode.test.ts`; additions to
  `packages/launcher/tests/action-catalog.test.ts`.

## Non-goals

- Re-opening W1–W5 internals beyond composition (no change to `worktreeAdd`/
  `worktreeRemove`, `ExecutionStore`, `provisionWorktreeExecution`'s own
  materialization logic, or `harvestAndCleanupExecution`'s step order/guards).
- SDD-artifact freshness / auto-fetch (W7) — untouched.
- Migrating Axiom's own repos (`Axiom.SDD`/`Axiom.Spec`) to any worktree
  model — out of scope for the whole plan.
- Redirecting the EXISTING write-scope review (`_role-review.ts`) or
  functional-verify (`_functional-verify.ts`) to validate the WORKTREE's code
  instead of `projectRoot` — both still validate `args.projectRoot` exactly as
  before this increment (see Assumption 5 — a documented, deliberately
  deferred follow-up, not a silent gap).
- Wiring worktree mode into the web launcher's REAL execute path
  (`apiExecuteLauncherAction` → `executeSubcommand`) — the launcher exposes
  the CHOICE (craft/preview only); the CLI (`axiom-role start --worktree`) is
  the authoritative executor (see Assumption 4).
- An explicit `--artifact-kind` CLI flag — `Execution.artifactRef.kind` is
  inferred from the role's `id` prefix (see Assumption 6).
- Touching the `qa-e2e` workflow/`axiom-qa-e2e.ts` — out of this increment's
  `role`-workflow scope.
- Adding an execution-mode wizard step to `axiom workspace setup`'s multi-repo
  TUI wizard — `workspace-setup.ts` doesn't call `installProfile` today
  either; a future increment can add this (see Assumption 2).

## Acceptance criteria

- [x] An architect-chosen `executionMode` ('in-place' | 'worktree') setting
      exists at the install-time config layer, defaulting to 'in-place' when
      never configured.
- [x] The mode is exposed in the CLI role-start flow (`--worktree`/
      `--in-place`) and in the launcher (an `executionMode` field on
      `back-new`/`front-new`), both defaulting to the install setting and
      overridable per invocation.
- [x] Worktree mode composes, in order: parameterized branch name (reused
      in-place naming) + a unique worktree path (execution-id-shaped token) →
      `worktreeAdd` (via `createExecutionForWorktree`, W1+W2) → provisioning
      (W3). On `complete`, `harvestAndCleanupExecution` (W5) runs.
- [x] Branch + worktree naming is parameterized (overridable `branchTemplate`/
      `worktreesRoot`) and always yields a unique worktree path per execution
      — proven for two concurrent executions on the SAME id/slug (real fs
      check) and for two different roles in the same repo.
- [x] In-place stays the default AND fully unchanged: no flags + no install
      config → `runRoleSubcommandAsync`'s output is byte-for-byte identical to
      `runRoleSubcommand`'s (message, exitCode, toState); no `.worktrees`
      sibling directory is ever created; `worktreeStart`/`worktreeClose` are
      `undefined`.
- [x] Closing a worktree run uses W5's safe path — a dirty worktree is a
      surfaced refusal, `force` is never set by real CLI flags.
- [x] `npm run build` (`tsc -b`) passes.
- [x] Unit tests: executionMode default resolves to the install setting; an
      explicit per-run override wins; worktree mode composes
      worktreeAdd+Execution+provision (asserted against a real, hermetic
      temp-git fixture); in-place mode is provably unchanged; naming yields
      unique worktree paths for concurrent executions.
- [x] An integration test (real git) proves: role-start in worktree mode →
      worktree + Execution + provisioned surface (execution-scoped
      `.axiom-state` layout) exist → `complete` → harvest+cleanup runs →
      worktree is gone, evidence/logs are preserved centrally (verified by
      reading back planted content from the SOURCE repo's
      `Execution.evidencePath`/`logsPath` after worktree removal).
- [x] Guard tests updated in lockstep: `packages/doctor` (install-profile
      checks read `install-profile.json` loosely, unaffected), `packages/launcher`
      (`action-catalog.test.ts` extended), `packages/installer`/
      `packages/install-profiles` (new field covered), `apps/cli` role-start
      tests (`axiom-role-git.test.ts` untouched and still green — the new
      flow is fully additive).

## Open questions

None blocking. See Assumptions for every ambiguity resolved
(existing-precedent-first, per the "resolve ambiguity yourself" guardrail).

## Assumptions

1. **`executionMode` lives in `install-profile.json`
   (`ResolvedInstallProfile.executionMode`), not `workspace.json`.** Both were
   viable ("install-profile / workspace config" per the brief). Chosen
   `install-profile.json` because: (a) it is the EXACT file
   `provisionWorktreeExecution` (W3) already reads for the rest of a
   worktree's resolved config (`adapterTarget`), so W6 adds zero new file
   reads to that pipeline; (b) worktree provisioning ALREADY hard-depends on
   `install-profile.json` existing (W3 warns and skips MCP-config
   materialization otherwise), so tying the execution-mode default to the
   same file doesn't add a new dependency, it aligns with an existing one.
   The composer (`resolveInstallProfile`) itself never composes
   `executionMode` from the (functionalProfile, overlay, adapterTarget)
   triple — it always defaults to `'in-place'` — because there is no
   compositional input to derive it from; the REAL resolution
   (explicit-arg → preserve-previous → default) lives in `@axiom/installer`'s
   `installProfile`, mirroring exactly how `generatedFiles`/
   `externalDependencies` are ALSO not composed but augmented one layer up.
   Since `install-profile.json` is fully REWRITTEN on every `axiom configure`
   run (unlike `workspace.json`, which `configure` only touches when
   `--providers` is passed), `installProfile` reads back the PREVIOUS file's
   `executionMode` when the caller omits the flag, so a plain `axiom
   configure` (for unrelated reasons) never silently resets the architect's
   choice.
2. **`axiom workspace setup` (the multi-repo TUI wizard) is NOT wired to set
   `executionMode`** — it does not call `@axiom/installer`'s `installProfile`
   today for ANY field (it writes `workspace.json`/`init.json` directly,
   never `install-profile.json`), so adding an execution-mode wizard step
   there would be a larger, separate lift. `axiom configure --execution-mode`
   is the (single, already-precedented) surface for this in this increment;
   a future increment can extend the wizard.
3. **`init.json`/`InitRecord`/`ProfileTriple` are untouched** —
   `executionMode` is not part of the (functionalProfile, overlay,
   adapterTarget) triple `axiom init` writes; threading a 4th field through
   that widely-used, heavily-tested shape was judged out of proportion to
   this increment's scope.
4. **The web launcher's `back-new`/`front-new` `executionMode` field is
   PREVIEW-only.** `apiExecuteLauncherAction` → `executeSubcommand` →
   `runRoleSubcommand` is UNTOUCHED (`app-api.ts` was not modified at all):
   it already does not wire `--create-branch`/`--commit`/`--confirm` for
   role-start today, so wiring worktree materialization there — an ASYNC,
   heavier operation — while git side-effects still don't run there would be
   an inconsistent, higher-risk asymmetry. `buildCliArgs` DOES include
   `executionMode` in its returned args (harmless — `executeSubcommand`'s
   `role` case never reads that key), so `buildLauncherCommandPreview` can
   special-case it into `--worktree`/`--in-place` for the DISPLAYED
   CLI-equivalent command, while the actual "execute" button still only
   performs the state transition (exactly as before this increment). The CLI
   (`axiom-role start --worktree`) is the authoritative executor.
5. **Write-scope review (`_role-review.ts`) and functional-verify
   (`_functional-verify.ts`) still validate `args.projectRoot` (the SOURCE
   repo), not the worktree, on `complete`.** Redirecting them would need
   `RoleReviewDeps`/`verifyDeps.repoRoot` to default to
   `execution.worktreePath` when in worktree mode — a correctness
   improvement, but one that (a) is not required by any acceptance criterion
   here (the criteria are about compose/config/close, not scope-review
   correctness), and (b) for the review specifically depends on
   `_repo-affinity.ts`'s topology-based identity resolution, a heavier,
   separate lift. Documented explicitly as a deferred follow-up, not a silent
   gap.
6. **`Execution.artifactRef.kind` is inferred from the role's `id` prefix**
   (`bug*` → `'bug'`, `plan*` → `'plan'`, else `'increment'`) —
   `axiom-role.ts` has no explicit "increment vs bug vs plan" input today;
   adding a new `--artifact-kind` CLI flag was judged out of proportion to
   this increment's scope. Purely descriptive `Execution` metadata; never
   branches W1–W5 behavior.
7. **The worktree PATH token is generated via the SAME `generateUniqueExecutionId`
   generator (`@axiom/isolation`) that later mints `Execution.id`, but the two
   are NOT guaranteed to be the same literal string.** `ExecutionStore.create()`
   always generates its OWN id internally (no caller-supplied `id` field —
   accepting one would mean reopening W2's internals). Both are unique by
   construction (collision-checked against real fs state); they are linked via
   `Execution.worktreePath`, not by string equality.
8. **The worktree lives in a SIBLING `<repoBasename>.worktrees/` directory**
   by default (parameterized via `WorktreeNamingOptions.worktreesRoot`) — no
   existing convention was established by W1–W5 (naming was explicitly W6's
   job); this mirrors Axiom's own dot-suffix sibling convention
   (`<name>.spec`, `<name>.axiom`).
9. **`runRoleSubcommand` stays 100% synchronous and additive-only** (new
   OPTIONAL input `executionMode`, new OPTIONAL outputs `executionMode`/
   `executionId`) — the actual async worktree materialization/teardown lives
   in a SEPARATE exported wrapper, `runRoleSubcommandAsync`, called by the CLI
   action handler (already `async`). This keeps every existing caller (MCP
   `sdd.transitionApply`, `app-api.ts`'s `executeSubcommand`, every
   pre-existing test) byte-for-byte unchanged, and avoids a breaking
   sync→async signature change on a widely-used function.

## Implementation notes

### Where `executionMode` lives and how it resolves

- **Storage**: `ResolvedInstallProfile.executionMode` (`@axiom/install-profiles`),
  persisted at `<repoRoot>/.axiom-state/<projectId>/install-profile.json` by
  `@axiom/installer`'s `installProfile` — set via `axiom configure
  --execution-mode <in-place|worktree>`.
- **Default/resolution** (`apps/cli/src/commands/_execution-mode.ts`):
  `resolveExecutionMode(projectRoot, override?)` → `override` if given, else
  `readInstallTimeExecutionMode(projectRoot)` (scans
  `.axiom-state/*/install-profile.json`, best-effort, defaults to
  `DEFAULT_EXECUTION_MODE` = `'in-place'` on any miss/corruption — never
  throws).
- **Override surfaces**: CLI `axiom-role start --worktree`/`--in-place`
  (mutually exclusive, validated in `addRoleSubcommand`'s action handler);
  launcher `back-new`/`front-new`'s `executionMode` select field (preview
  only — see Assumption 4).

### The worktree start flow (`runRoleSubcommandAsync`, `subcommand: 'start'`)

1. `resolveExecutionMode(projectRoot, executionModeOverride)`.
2. `runRoleSubcommand({ ...args, executionMode: resolvedMode })` — the
   UNCHANGED sync state machine (plan-approved gate, repo-affinity, the
   `role-start` transition), which additionally stashes
   `vars.executionMode = resolvedMode`. If this fails or the mode is
   `'in-place'`, return its result AS-IS (no worktree branch taken).
3. `startWorktreeExecution` (`_worktree-execution.ts`):
   a. `buildWorktreeBranchName(vars, branchTemplate)` →
      `renderBranchName('role', ...)` (same default `role/<id>-<slug>` as
      in-place `--create-branch`).
   b. `buildUniqueWorktreePath(projectRoot, vars, namingOptions)` — a unique,
      collision-checked path under `<repoBasename>.worktrees/`.
   c. `createExecutionForWorktree({ store, worktree: {repoRoot, worktreePath,
      branchName}, projectId, artifactRef, repoId })` (W1+W2) — `worktreeAdd`
      then `ExecutionStore.create`.
   d. `provisionWorktreeExecution({ execution, sourceRootPath, store })`
      (W3).
4. Persists `vars.executionId` onto the role's workflow-state record
   (`persistExecutionId` — a small re-load+patch+re-save, best-effort).

### The close flow (`runRoleSubcommandAsync`, `subcommand: 'complete'`)

1. `runRoleSubcommand(args)` — UNCHANGED (review gate → functional-verify gate
   → `role-complete` transition → save), still validating `args.projectRoot`
   (see Assumption 5).
2. If the result's `executionMode === 'worktree'` and `executionId` is set:
   `createExecutionStore(projectRoot).get(executionId)` → `closeWorktreeExecution`
   → `harvestAndCleanupExecution` (W5: kill → harvest → teardown →
   `worktreeRemove`, NEVER `force` by default) → idempotent no-op if the
   `Execution` is already `'removed'`.

### Naming scheme + uniqueness proof

- Branch: `renderBranchName('role', branchTemplate, {id, slug})`, default
  `role/<id>-<slug>` — IDENTICAL to in-place.
- Worktree path: `<worktreesRoot>/<slug(id)>[-<slug(slug)>]-<token>`, where
  `<token>` = `generateUniqueExecutionId((candidate) =>
  pathExists(join(worktreesRoot, prefix + '-' + candidate)))`.
  `apps/cli/tests/worktree-execution.test.ts` proves: (a) a forced first-candidate
  collision triggers a retry that yields a different candidate; (b) two
  "concurrent" requests for the SAME `id`/`slug`, with the first candidate
  actually materialized on disk between calls, get DISTINCT, non-colliding
  paths (real `fs.existsSync` check, no mocks); (c) two different roles in
  the same repo get distinct paths.

### In-place unchanged

`apps/cli/tests/axiom-role-worktree.test.ts`'s first test asserts
`runRoleSubcommandAsync` (no flags, no install config) produces an output
whose `message`/`exitCode`/`toState` are IDENTICAL to a direct
`runRoleSubcommand` call, `worktreeStart`/`worktreeClose` are `undefined`, and
no `<repo>.worktrees` sibling directory is ever created.

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0 (verified twice, before
  and after the full test pass).
- `npx vitest run packages/install-profiles packages/installer` — **7 files /
  65 tests passed** (includes the 2 new `execution-mode.test.ts` files: 3 +
  5 tests).
- `npx vitest run apps/cli/tests/execution-mode.test.ts
  apps/cli/tests/worktree-execution.test.ts
  apps/cli/tests/axiom-role-worktree.test.ts
  apps/cli/tests/e2e/axiom-role-worktree.e2e.test.ts
  apps/cli/tests/launcher-execution-mode.test.ts` — **5 files / 36 tests
  passed** (one initial failure in the e2e test — a TEST-FIXTURE gap, not a
  product bug: the throwaway temp repo never wrote a `.gitignore`, so planted
  evidence/log files inside the worktree were untracked-and-dirty; fixed by
  committing them inside the throwaway worktree before `complete`, mirroring
  `execution-harvest.test.ts`'s own `commitAll` precedent — re-ran green).
- `npx vitest run packages/launcher packages/mcp-server packages/doctor` —
  **30 files / 286 tests passed** (guard tests: `action-catalog.test.ts`
  extended with 3 new tests; `install-profiles.test.ts`/other doctor checks
  unaffected, since they read `install-profile.json` loosely).
- `npx vitest run apps/cli/tests` (the FULL apps/cli suite) — **124 test
  files / 1147 tests passed**, zero failures (includes the two files the
  brief flagged as known-flaky-under-parallel-load,
  `context.test.ts`/`workspace-setup.test.ts` — both passed clean in this
  run; includes every new/updated file from this increment).
- `npx vitest run packages/persistence packages/isolation packages/workflow`
  (extra safety pass — no source changes there, but this increment imports
  from all three) — **23 files / 310 tests passed**, including
  `git-worktree-services.test.ts` (previously flagged flaky-under-parallel-load
  by a sibling increment; clean here).

No pre-existing failures were encountered in any touched or adjacent scope.

## Result

Implemented. The architect now configures a project-wide default execution
mode via `axiom configure --execution-mode <in-place|worktree>` (persisted to
`install-profile.json`, defaulting to `'in-place'` and preserved across
unrelated re-configures); `axiom-role start` resolves it (install default →
per-run `--worktree`/`--in-place` override) via the new async
`runRoleSubcommandAsync`, which composes W1+W2 (`createExecutionForWorktree`)
→ W3 (`provisionWorktreeExecution`) on start and W5
(`harvestAndCleanupExecution`) on `complete`, using parameterized,
collision-proof branch/worktree naming. The web launcher surfaces the same
choice (preview-only) on `back-new`/`front-new`. In-place remains the default
and is provably unchanged (byte-for-byte identical output, no new
side-effects) when worktree mode is never selected. All 5 previously-unwired
Cluster W primitives (`worktreeAdd` via `createExecutionForWorktree`,
`ExecutionStore`, `provisionWorktreeExecution`, `harvestAndCleanupExecution`)
now have their first real, user-facing caller.

## General spec integration

Integrado en la spec canónica al cierre del batch INC-20260724-* (integración
única de toda la tanda). Ficheros que recibieron el conocimiento de este
incremento:

- `05_Interfaces_Operativas.md` — `axiom configure --execution-mode`,
  `axiom-role start --worktree`/`--in-place`, campo `executionMode` del
  launcher (preview-only).
- `04_Flujos_SDD_y_Ciclo_de_Vida.md` — resolución del modo + wiring
  start/complete (compone W1/W2/W3 y W5).
- `03_Modelo_Operativo_y_Datos.md` — `executionMode` en `install-profile.json`.
- `08_Glosario.md` — `executionMode` (in-place | worktree).
- `01_Requisitos_Funcionales.md` — RF-AXM-047.
