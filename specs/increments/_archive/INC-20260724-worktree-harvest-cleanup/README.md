# Increment: Worktree harvest + safe cleanup orchestrator

Status: closed
Date: 2026-07-24

## Goal

Provide a single orchestrator, `harvestAndCleanupExecution`, that safely tears
down a finished worktree `Execution`: preserve everything worth keeping
(logs, evidence, outputs) by copying it to a CENTRAL location that survives
worktree removal, then remove the worktree and its derived code-intel
indexes — in a strict order that never loses harvestable data and never
destroys uncommitted work.

This is **INC-W5** of Cluster W in the Axiom evolution plan
(`giggly-imagining-moore.md`, "Cluster W — worktrees"), depending on
**INC-W1** (`INC-20260724-git-worktree-services`, closed — `worktreeAdd`/
`worktreeRemove`), **INC-W2** (`INC-20260724-worktree-isolation-execution`,
closed — the `Execution` entity, execution-scoped paths, `ExecutionStore`),
and **INC-W4** (`INC-20260724-worktree-provider-isolation`, closed —
`teardownWorktreeCodeIntel`). It is an explicit, user-approved graduation to
the full product lifecycle (per `Axiom.SDD/AGENTS.md`'s "Explicit Bootstrap
Limits" — worktree support is a deliberate exception this plan requests, not
speculative architecture).

## Context

Three prior increments built the primitives this increment composes, but
nothing wires them together yet:

- **INC-W1** gave `worktreeRemove` (`packages/workflow/src/git/git-services.ts`):
  removes a registered git worktree, REFUSES a dirty one (uncommitted
  changes) unless `force: true`, and never touches the main repo root.
- **INC-W2** gave the `Execution` entity (`packages/isolation/src/
  execution.ts`) and `ExecutionStore` (`packages/persistence/src/
  execution-store.ts`). Critically, `ExecutionStore` is rooted at the SOURCE
  repo (`createExecutionStore(rootPath)` — "the natural choice is the SOURCE
  repo root... so `execution.json` and `Execution.logsPath`/`evidencePath`
  survive worktree removal (needed for harvest, INC-W5)" — per that
  increment's own `workspace-worktree-provision.ts` doc comment). So
  `execution.logsPath`/`execution.evidencePath` are ALREADY the CENTRAL,
  survives-removal destination — this increment does not need to invent a
  new central location, only copy into the one that already exists.
- **INC-W3** (`provisionWorktreeExecution`,
  `apps/cli/src/commands/workspace-worktree-provision.ts`) scaffolds a
  SEPARATE, EPHEMERAL, worktree-LOCAL mirror of the same
  `{config,mcp,outputs,logs,evidence,local}` shape (via
  `buildExecutionScopedPaths(execution.id, execution.worktreePath)`) INSIDE
  the worktree, for an agent session to write into during the run. Its own
  doc comment says explicitly: "Reconciling/harvesting the worktree-local
  copy into the store's central location is explicitly INC-W5's job (harvest
  + cleanup), not this increment's."
- **INC-W4** gave `teardownWorktreeCodeIntel`
  (`packages/providers/src/code-intel/teardown.ts`): a synchronous,
  best-effort, single-target-only primitive that deletes a worktree's
  derived `.cmm`/`.serena` index/cache directories. Its own doc comment
  frames itself as "the LAST of those 4 steps" of a future harvest/cleanup
  orchestration — this increment IS that orchestration.

The one thing none of W1/W2/W4 solve on their own: `.cmm`/`.serena` are
UNTRACKED, non-ignored directories inside the worktree. If they are still
present when `worktreeRemove` runs its dirty check (`git status
--porcelain`), they make the worktree look "dirty" even when there is no
REAL uncommitted work — which would force every caller to pass `force:
true`, defeating the exact guard INC-W1 built to protect uncommitted work.
Deleting them (INC-W4's teardown) BEFORE calling `worktreeRemove` (INC-W1)
removes this false-dirty signal so the ONLY thing `worktreeRemove`'s dirty
check can still catch is genuinely uncommitted work.

## Scope

- `packages/persistence/src/execution-harvest.ts` (new) — the orchestrator:
  `harvestAndCleanupExecution`, plus its option/result/error types and the
  `KillProcessesFn` seam.
- `packages/persistence/src/index.ts` — barrel exports for the above.
- `packages/persistence/package.json` — new dependency on `@axiom/providers`
  (needed to import `teardownWorktreeCodeIntel` directly; verified
  dependency-direction-safe: `@axiom/providers` and its own dependencies —
  `capability-model`, `install-profiles`, `project-resolution`,
  `tool-routing` — do not depend on `@axiom/persistence` or
  `@axiom/workflow`, so this new edge introduces no cycle).
- `packages/persistence/tsconfig.json` — matching `paths`/`references` entry
  for `@axiom/providers` (TS project-references build).
- `packages/persistence/tests/execution-harvest.test.ts` (new) — unit tests
  (fakes/injected seams) + one real-git integration test.

## Non-goals

- Wiring this orchestrator into any user-facing CLI command, MCP tool, or
  launcher flow — INC-W6 ("Selección de modo + default de instalación").
  `harvestAndCleanupExecution` is exposed as a plain, callable function for
  W6 to invoke; nothing in `apps/cli` is touched by this increment.
- SDD-artifact freshness / auto-fetch (increment/bug/plan staleness) —
  INC-W7; unrelated to this increment's runtime-state harvest.
- Re-opening INC-W1 (`worktreeRemove`'s own guard logic), INC-W2 (`Execution`
  shape, `ExecutionStore`'s persistence format), or INC-W4
  (`teardownWorktreeCodeIntel`'s own removal contract) beyond calling them as
  black boxes. No behavioral change to any of the three.
- A real "tracked long-running process" registry: today's providers spawn
  short-lived per-invoke subprocesses (per INC-W4's context), so there is
  nothing real to kill yet. This increment defines the seam
  (`killProcesses?: KillProcessesFn`) and a safe, honest default (reports
  `attempted: false` — no registry exists to consult), not a process
  tracker.
- Wiring telemetry (`@axiom/telemetry`) into the execution-scoped harvest
  tree. Telemetry today writes to a PROJECT-scoped overlay
  (`<projectRoot>/.axiom-state/local/telemetry.log`, via
  `LocalOnlySink`/`localOnlyLogPath`), not the execution-scoped
  `{logs,evidence,outputs}` tree `buildExecutionScopedPaths` defines. Making
  telemetry execution-scoped (so a worktree's own telemetry log would need
  harvesting too) is a separate, future increment — out of scope here (see
  Assumptions).
- Migrating Axiom's own repos (`Axiom.SDD`/`Axiom.Spec`) to any worktree
  model — out of scope for the whole plan.

## Acceptance criteria

- [x] `harvestAndCleanupExecution(execution, { store, force?, killProcesses?,
      dryRun? })` runs, in this STRICT order: (a) best-effort kill of tracked
      processes, (b) harvest (copy worktree-local `{logs,evidence,outputs}`
      to the central, survives-removal location), (c) teardown derived
      code-intel indexes (`teardownWorktreeCodeIntel`), (d) `worktreeRemove`.
      Proven by a unit test asserting call order via injected seams
      (`killProcesses`, `copyDirSync`, `removeDirSync`, `gitRunner`), not by
      reading the implementation.
- [x] Harvest copies `logs`/`evidence`/`outputs` from the worktree-local
      execution scope into `Execution.logsPath`/`evidencePath` (and the
      sibling central `outputs` dir) — proven to survive AFTER the worktree
      is physically removed (real-git integration test).
- [x] A dirty worktree (real uncommitted work, not just leftover derived
      indexes) is a HARD STOP: `worktreeRemove`'s refusal propagates as an
      error, the execution's state is NOT advanced to `removed`, and nothing
      already harvested is lost (the error result carries what was
      harvested).
- [x] `force: true` overrides the dirty refusal (proxied straight to
      `worktreeRemove`) and the run completes, advancing state to `removed`.
- [x] `dryRun: true` reports the planned sequence and mutates nothing: no
      file copy, no `teardownWorktreeCodeIntel` call, no `store.update`/
      `store.close` call. (`worktreeRemove` itself is still invoked with
      `dryRun: true`, so its OWN read-only guards — including the dirty
      check — still run for an accurate preview; only mutation is skipped.)
- [x] A harvest copy failure (e.g. simulated I/O error) degrades to a
      warning and does NOT block teardown/removal of an otherwise-clean,
      finished run.
- [x] `Execution.state` transitions observed, in order, for a full
      successful run: `created` (or later) `-> harvested -> removed`.
- [x] Teardown runs BEFORE `worktreeRemove` so leftover `.cmm`/`.serena`
      (untracked, non-ignored) never causes a FALSE dirty-worktree refusal —
      proven by a real-git integration test that seeds untracked `.cmm`/
      `.serena` directories and asserts the run still succeeds.
- [x] `npm run build` (`tsc -b`) passes.
- [x] Targeted `vitest run` for `packages/workflow`, `packages/persistence`,
      `packages/isolation`, `packages/providers`, `packages/telemetry`
      passes; pre-existing failures (if any) are classified separately from
      new ones.
- [x] No `apps/cli` file is touched (W6 owns that wiring). Verified: the full
      `apps/cli` suite (119 files / 1111 tests) was still run for regression
      confidence (transitive dependency on `@axiom/persistence`) and passes
      unchanged from INC-W4's own last-recorded baseline (119/1111, +0/+0).

## Open questions

None blocking — the brief bakes in the final orchestrator signature and
strict step order. See Assumptions for the ambiguities resolved
(existing-precedent-first) rather than left open.

## Assumptions

1. **The orchestrator lives in `@axiom/persistence`**
   (`execution-harvest.ts`), mirroring the exact precedent of
   `execution-worktree.ts` in the same package — a "compose helper" with no
   caller yet, described in its own doc comment as "Pensado para que W3 lo
   invoque directamente" (here: for W6 to invoke). `@axiom/persistence`
   already depends on `@axiom/workflow` (for `worktreeAdd`) and
   `@axiom/isolation` (for `Execution`/`buildExecutionScopedPaths`); this
   increment adds the one missing edge, `@axiom/providers` (for
   `teardownWorktreeCodeIntel`), verified acyclic (see Scope).
2. **No new `repoRoot`/`sourceRootPath` parameter.** The brief's FINAL
   signature is `harvestAndCleanupExecution(execution, { store, force?,
   killProcesses?, dryRun? })` — no separate repo-root argument.
   `worktreeRemove` needs one (its `repoRoot`), so it is DERIVED from
   `Execution.logsPath` by reversing `buildExecutionScopedPaths`'s known
   layout (`<repoRoot>/.axiom-state/executions/<id>/logs`), with a shape
   assertion (`.axiom-state`/`executions`/`<id>` segments must match) that
   fails clearly instead of silently resolving a nonsense path.
3. **Central `outputs` destination is derived, not stored.** `Execution`
   only persists `logsPath`/`evidencePath` (INC-W2's own deliberate
   choice), not an `outputsPath`. Since `buildExecutionScopedPaths` makes
   `outputs` a SIBLING of `logs`/`evidence` under the same `execRoot`, the
   central outputs destination is `path.join(path.dirname(execution.
   logsPath), 'outputs')` — no new persisted field, no schema change to
   `Execution`.
4. **Harvest categories are exactly `logs`, `evidence`, `outputs`** — the
   three named explicitly in the brief's Change section. Telemetry is
   explicitly NOT included (see Non-goals #5): it is project-scoped today,
   not execution-scoped, so there is nothing under the execution scope to
   copy for it yet.
5. **`killProcesses` is a caller-injectable seam
   (`(execution) => KillProcessesResult | Promise<KillProcessesResult>`)**,
   defaulting to an honest no-op (`{ attempted: false, killed: [],
   warnings: [] }`) — per the brief, "today providers spawn per-invoke
   short-lived subprocesses, so there may be nothing long-running." This is
   the "clean seam for tracked long-running MCP processes" the brief asks
   for, not a process registry (none exists yet to build on).
6. **`teardownWorktreeCodeIntel` has no preview mode**, so during `dryRun`
   it is never called at all (only described in the planned-steps report);
   during an applied run it always runs for real, UNCONDITIONALLY, before
   `worktreeRemove` — even when the worktree might otherwise be dirty for
   unrelated reasons. This is safe because teardown only ever deletes
   reconstructible index caches (`.cmm`/`.serena`), never real work.
7. **A dry-run preview of `worktreeRemove`'s dirty check reflects the state
   BEFORE teardown runs** (since teardown is skipped in `dryRun`), so a
   preview MAY report `wasDirty: true` (or even a `dirty-worktree` error)
   solely because of not-yet-removed derived index directories, when the
   equivalent APPLIED run would succeed (teardown removes them first). This
   is a documented, accepted limitation of previewing a multi-step mutation
   sequence — not solved by a speculative double-pass simulation.
8. **Bookkeeping (`store.update`/`store.close`) failures never block the
   physical steps.** The invariant this increment protects is "never lose
   harvestable data, never destroy uncommitted work" — a failed state-label
   write after a REAL successful harvest/removal is reported as a warning
   (`advancedToHarvested`/`advancedToRemoved: false`), not treated as an
   overall failure, since the dangerous physical operations already
   completed correctly.
9. **Only a `dirty-worktree` refusal from `worktreeRemove` is a hard stop
   with a dedicated error kind.** Any OTHER `worktreeRemove` failure
   (`not-a-worktree`, `invalid-worktree-path`, `git-command-failed`) also
   stops the run (state is never advanced to `removed`) but is reported
   under a more generic `worktree-remove-failed` error kind, since the
   brief singles out dirtiness specifically as the guarded case.

## Implementation notes

Files changed (all in `Axiom/`, the product monorepo):

- `packages/persistence/src/execution-harvest.ts` (new) —
  `harvestAndCleanupExecution` + types (`KillProcessesFn`,
  `KillProcessesResult`, `HarvestCategory`, `HarvestCopyOutcome`,
  `HarvestCategoryResult`, `HarvestAndCleanupOptions`,
  `HarvestAndCleanupResult`, `HarvestAndCleanupError`) + private helpers
  (`centralExecRootOf`, `deriveRepoRootFromExecution`,
  `defaultCopyDirSync`, `defaultKillProcesses`).
- `packages/persistence/src/index.ts` — barrel exports for the above.
- `packages/persistence/package.json` — new dependency `"@axiom/providers":
  "*"`.
- `packages/persistence/tsconfig.json` — matching `paths`/`references` entry
  for `@axiom/providers`.
- `packages/persistence/tests/execution-harvest.test.ts` (new) — 8 tests,
  ALL exercising a REAL `git init` + real `worktreeAdd` (mirroring
  `execution-worktree.test.ts`'s own style — no fake git simulator);
  targeted fake seams (`killProcesses`, `copyDirSync`,
  `teardownOptions.removeDirSync`, a `gitRunner` that wraps — never
  replaces — the real one) are used only for order-tracking/failure
  simulation. The 8th test is the brief's explicitly required end-to-end
  scenario (`worktreeAdd` -> seed fake logs/evidence/outputs -> commit ->
  `harvestAndCleanup` -> assert preserved centrally / worktree gone / indexes
  gone), plus proves teardown-before-remove avoids a false dirty refusal.

### Orchestrator signature + strict step order

```ts
async function harvestAndCleanupExecution(
  execution: Execution,
  options: {
    store: ExecutionStore;
    force?: boolean;
    killProcesses?: (execution: Execution) => KillProcessesResult | Promise<KillProcessesResult>;
    dryRun?: boolean;
    // test/advanced pass-through seams:
    gitRunner?: GitRunner;
    copyDirSync?: (sourceDir: string, destDir: string) => void;
    teardownOptions?: TeardownWorktreeCodeIntelOptions;
  },
): Promise<Result<HarvestAndCleanupResult, HarvestAndCleanupError>>
```

Strict order and where each primitive comes from:

1. **(a) kill tracked processes** — `options.killProcesses` (caller-injected
   seam) or `defaultKillProcesses` (honest no-op: `{attempted:false,
   killed:[],warnings:[]}` — no process registry exists today, per INC-W4's
   own Context). Best-effort; never invoked for real during `dryRun`.
2. **(b) harvest** — pure `fs` copy (`fs.cpSync`-based `defaultCopyDirSync`,
   same pattern as `apps/cli/src/commands/eject.ts`'s
   `copyArtifactIntoExport`), from `buildExecutionScopedPaths(execution.id,
   execution.worktreePath)` (`@axiom/isolation`, INC-W2 — the worktree-LOCAL
   scope) to `execution.logsPath`/`evidencePath` (already central, INC-W2)
   plus a derived central `outputs` sibling. Best-effort per category.
3. **(c) teardown** — `teardownWorktreeCodeIntel(execution, ...)`
   (`@axiom/providers`, INC-W4) called directly, unconditionally (except in
   `dryRun`, where it has no preview mode so it is simply never invoked).
4. **(d) worktreeRemove** — `worktreeRemove({repoRoot, worktreePath:
   execution.worktreePath, force, dryRun, gitRunner})` (`@axiom/workflow`,
   INC-W1), called for real in BOTH modes (it has its own native
   preview/confirm contract). A `dirty-worktree` (or any other) failure is a
   HARD STOP: the function returns `err(...)` immediately, before any state
   transition to `removed`.

Only on a successful (applied) step (d) does the function call
`store.close(execution.id)` (soft-close to `'removed'` — INC-W2, already
idempotent).

### How the 3 hard guarantees are enforced/tested

- **Harvest-before-delete**: encoded structurally — step (b)'s code runs to
  completion (including the `store.update(...,{state:'harvested'})` call)
  BEFORE step (c)/(d) are ever reached; there is no conditional path that
  skips (b) or reorders it after (c)/(d). Tested via a call-order spy
  (`execution-harvest.test.ts`, "strict step order" — injected
  `killProcesses`/`copyDirSync`/`removeDirSync`/`gitRunner` all push into one
  shared `order[]` array; assertions compare indices, not just "both
  happened").
- **Dirty-worktree hard stop**: `removeResult.error.kind === 'dirty-worktree'`
  short-circuits to `return err(...)` immediately — no code path after that
  branch can reach the `store.close` call. Tested with REAL uncommitted work
  left in a real worktree (not `.cmm`/`.serena`, which teardown already
  cleared) — asserts `result.ok === false`, worktree and its uncommitted
  file both still exist, harvested content already exists centrally, and
  the store's `state` never becomes `'removed'`.
- **`dryRun` mutates nothing**: every mutating call is behind an `if
  (dryRun)` branch that ONLY pushes a human-readable string to
  `plannedSteps` — `killProcesses` is never invoked, `copyDirSync` is never
  invoked (only `fs.existsSync` reads), `teardownWorktreeCodeIntel` is never
  invoked, `store.update`/`store.close` are never invoked. The one call NOT
  gated behind `if (dryRun)` is `worktreeRemove` itself, invoked WITH
  `dryRun: true` — it has its own native preview mode (reads git status,
  mutates nothing). Tested: worktree, its `.git`, and its local harvest
  files are all still present afterward; the central destination was never
  created; the store's state is unchanged.

### Central harvest destination + how it survives removal

`Execution.logsPath`/`evidencePath` are already rooted at the SOURCE repo
(`ExecutionStore`'s own `rootPath`, per INC-W2's design — "the natural
choice is the SOURCE repo root... so `execution.json` and
`Execution.logsPath`/`evidencePath` survive worktree removal"), i.e.
`<repoRoot>/.axiom-state/executions/<id>/{logs,evidence}` — a directory tree
that lives OUTSIDE the worktree entirely, so `git worktree remove
<worktreePath>` (which only ever deletes `<worktreePath>` itself) cannot
touch it. The central `outputs` destination is derived as a sibling:
`path.join(path.dirname(execution.logsPath), 'outputs')` (no new persisted
field on `Execution`). `worktreeRemove`'s own `repoRoot` argument (needed
by INC-W1 but absent from this increment's final signature) is derived by
reversing the same known layout from `execution.logsPath`
(`deriveRepoRootFromExecution`), guarded by a shape assertion that fails
loudly instead of silently computing a nonsense path.

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0:

  ```
  > axiom-product@0.1.0 build
  > tsc -b
  ```

  (no output, no errors — includes the new `@axiom/persistence` ->
  `@axiom/providers` project-reference edge building cleanly, confirmed
  acyclic.)

- `npx vitest run packages/persistence` (isolated first pass):

  ```
   Test Files  4 passed (4)
        Tests  50 passed (50)
     Start at  09:18:42
     Duration  22.69s
  ```

  Includes the new `packages/persistence/tests/execution-harvest.test.ts`
  (8 tests, all passing).

- `npx vitest run packages/workflow packages/persistence packages/isolation
  packages/providers packages/telemetry` (the full requested scope):

  ```
   Test Files  41 passed (41)
        Tests  447 passed (447)
     Start at  09:19:22
     Duration  28.42s (transform 3.56s, setup 6ms, collect 12.25s, tests 99.89s, environment 18ms, prepare 19.84s)
  ```

  Zero failures. No pre-existing failures were encountered in this run (all
  41 files green), so no pre-existing/new classification was needed.

- `npx vitest run apps/cli` (full suite, regression confidence — no
  `apps/cli` file was changed by this increment, but `@axiom/persistence`
  gained a new dependency edge and `apps/cli` consumes it transitively):

  ```
   Test Files  119 passed (119)
        Tests  1111 passed (1111)
     Start at  09:20:01
     Duration  56.99s (transform 21.73s, setup 21ms, collect 173.28s, tests 275.64s, environment 69ms, prepare 61.42s)
  ```

  119/1111 — byte-identical to INC-W4's own last-recorded baseline
  (119 files / 1111 tests), zero regressions, zero new failures.

No pre-existing failures were encountered or needed classification in any
validation run for this increment.

## Result

Implemented. `harvestAndCleanupExecution` (new,
`packages/persistence/src/execution-harvest.ts`) composes INC-W1's
`worktreeRemove`, INC-W2's `Execution`/`ExecutionStore`/
`buildExecutionScopedPaths`, and INC-W4's `teardownWorktreeCodeIntel` into
the strict `kill -> harvest -> teardown -> remove` sequence the brief
specifies, with harvest ALWAYS running before any deletion and a dirty
worktree ALWAYS being a hard stop (never silently overridden, never losing
already-harvested data). `dryRun` reports a human-readable planned sequence
while mutating nothing (delegating to `worktreeRemove`'s own native preview
mode for an accurate read of whether the run would be refused). The one
real design gap this increment closed itself — `worktreeRemove` needing a
`repoRoot` the final signature does not carry — is solved by reversing
`buildExecutionScopedPaths`'s known layout from `Execution.logsPath`, guarded
by a shape assertion. A real-git integration test proves the single most
important correctness property beyond simple composition: leftover,
untracked `.cmm`/`.serena` directories would normally make `worktreeRemove`
refuse removal as "dirty" — but because teardown (step c) always runs
BEFORE the removal's dirty check (step d), the run succeeds anyway, and the
harvested logs/evidence/outputs are provably still readable from the
central location after the worktree is entirely gone. `@axiom/persistence`
gained one new, verified-acyclic dependency (`@axiom/providers`); no
`apps/cli` file was touched (INC-W6 owns wiring this into a user-facing
command).

## General spec integration

Integrado en la spec canónica al cierre del batch INC-20260724-* (integración
única de toda la tanda). Ficheros que recibieron el conocimiento de este
incremento:

- `04_Flujos_SDD_y_Ciclo_de_Vida.md` — `harvestAndCleanupExecution` en orden
  estricto kill→harvest→teardown→remove; harvest sobrevive al borrado;
  worktree sucio = hard stop.
- `07_Gobierno_y_Seguridad.md` — cleanup seguro (harvest antes de borrar,
  nunca `force` por defecto).
- `08_Glosario.md` — `harvestAndCleanupExecution`.
- `02_Requisitos_No_Funcionales.md` — NFR-AXM-019 (limpieza segura por
  worktree).
- `01_Requisitos_Funcionales.md` — RF-AXM-047.
