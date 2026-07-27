# Increment: Git worktree services (`worktreeAdd` / `worktreeRemove`)

Status: closed
Date: 2026-07-24

## Goal

Add local-only git **worktree** primitives to `@axiom/workflow`'s git module
(`worktreeAdd`, `worktreeRemove`), mirroring the existing `roleBranch` /
`commitSync` service shape — same `GitServiceResultBase`
(`mode: 'preview'|'applied'` + `plannedCommands`) dry-run/preview contract,
same guard style, same injectable `GitRunner` seam — so later increments in
Cluster W (per-worktree isolation + `Execution` entity, provisioning,
provider isolation, harvest/cleanup, mode selection, freshness) can build a
full worktree execution lifecycle on top of these primitives.

This is **INC-W1** of Cluster W in the Axiom evolution plan
(`giggly-imagining-moore.md`, "Cluster W — worktrees (depende de A5 + T1)").
Cluster W is a fresh, mostly independent subsystem; this increment is its
foundation. It is an explicit, user-approved graduation to the full product
lifecycle (per `Axiom.SDD/AGENTS.md`'s "Explicit Bootstrap Limits" — worktree
support is a deliberate exception this plan requests, not speculative
architecture).

## Context

Today, `packages/workflow/src/git/git-services.ts` has `roleBranch`
(checkout in-place, existing-branch detection), `commitSync` (stage + commit
locally, push only when `push:true`), and `gitStatus` (read-only porcelain),
all behind an injectable `GitRunner` seam (`git-runner.ts`), all supporting
`dryRun` (preview planned commands, mutate nothing), and all guarded
(`isValidBranchName`, `resolveInsideRepo` — a path-escape guard requiring
scoped paths to resolve INSIDE the repo root). These are registered as named
actions (`role-branch` / `commit-sync`) in `git-actions.ts`'s
`GIT_ACTION_HANDLERS`, dispatched by the generic `runTransitionActions`
runner (`transition-effects.ts`) — a transition can declare an action; a
handler previews when `ctx.confirmed` is false and applies when true. There
is no `git worktree` support at all yet.

A git worktree is the INVERSE case of `resolveInsideRepo`'s path guard:
whereas `commitSync`'s scoped paths must resolve INSIDE the repo, a
worktree's target directory must resolve OUTSIDE the repo's working tree (a
separate/sibling directory sharing the repo's `.git`). This increment adds a
dedicated guard, `assertWorktreePathOutsideRepo`, for that inverse case, and
two new services on top of it.

## Scope

- `packages/workflow/src/git/git-services.ts`:
  - New guard `assertWorktreePathOutsideRepo(repoRoot, worktreePath)` —
    rejects an empty path, the main repo root itself, or a path resolving
    INSIDE the repo's working tree.
  - New `worktreeAdd(args)` — `git worktree add <path> -b <branch>` (new
    branch) or `git worktree add <path> <branch>` (attach an existing local
    branch, detected the same way `roleBranch` detects one). Refuses an
    existing NON-EMPTY target directory. Supports `dryRun`.
  - New `worktreeRemove(args)` — `git worktree remove <path>` (+ `--force`
    when `force:true`; + optional trailing `git worktree prune` when
    `prune:true`). Refuses a path that is not a currently-registered
    worktree (`git worktree list`); refuses a DIRTY worktree unless
    `force:true`. Supports `dryRun`.
  - New `GitServiceError` kinds: `invalid-worktree-path`,
    `worktree-path-exists`, `not-a-worktree`, `dirty-worktree`.
  - New result shapes `WorktreeAddResult` / `WorktreeRemoveResult` (both
    extend `GitServiceResultBase`).
  - New internal helpers: `canonicalize` (realpath-when-possible path
    canonicalization — see Implementation notes / Learned), `listWorktrees` /
    `findRegisteredWorktree` (parse + query `git worktree list --porcelain`).
- `packages/workflow/src/git/git-actions.ts`: `WORKTREE_ADD_ACTION` /
  `WORKTREE_REMOVE_ACTION` named actions + handlers, registered into
  `GIT_ACTION_HANDLERS` alongside `role-branch`/`commit-sync`. `GitActionContext`
  gains `worktreePath` / `force` / `prune`.
- `packages/workflow/src/git/index.ts` + `packages/workflow/src/index.ts`:
  barrel exports for everything above.
- Tests: new `packages/workflow/tests/git-worktree-services.test.ts` (22
  tests — pure guard tests, fake-runner guard/wiring tests, and real-`git
  init`-in-a-temp-repo integration tests for both services).

## Non-goals

- Per-worktree isolation / the `Execution` entity (`packages/isolation`) —
  INC-W2.
- The provisioning script that materializes a portable `.axiom` surface into
  a worktree — INC-W3.
- Per-worktree provider/index isolation (`serena`/`cmm`) — INC-W4.
- Harvest + safe cleanup orchestration (kill processes → harvest evidence →
  remove) — INC-W5.
- Execution-mode selection (in-place vs worktree), install-time default,
  branch/worktree naming/parametrization — INC-W6.
- Freshness / auto-fetch / staleness detection for SDD artifacts — INC-W7.
- Any CLI subcommand or MCP tool surface for worktrees (e.g. an
  `axiom-role start --worktree` flag, or `sdd.gitWorktreeAdd` /
  `sdd.gitWorktreeRemove` MCP tools mirroring `git-handlers.ts`) — judged OUT
  of this increment's explicit scope ("ONLY: the two git worktree services +
  guards + action-taxonomy/barrel wiring + tests"); see Assumption 4.
- Deleting the branch used by a removed worktree, or any other git-history
  mutation beyond the worktree administrative metadata itself.
- Migrating Axiom's own repos (`Axiom.SDD`/`Axiom.Spec`) to any worktree
  model — out of scope for the whole plan.

## Acceptance criteria

- [x] `worktreeAdd` creates a NEW branch (`-b`) when the branch does not
      exist, or attaches an EXISTING local branch (no `-b`) when it does —
      detection mirrors `roleBranch`.
- [x] `worktreeAdd` / `worktreeRemove` both support `dryRun`: report
      `mode: 'preview'` + non-empty `plannedCommands`, mutate nothing (no
      directory created/removed, no branch created).
- [x] Path guards reject: the main repo root itself, a path resolving INSIDE
      the repo, and (for `worktreeAdd`) an existing NON-EMPTY target
      directory. An existing EMPTY directory is ALLOWED (git itself accepts
      this).
- [x] `worktreeRemove` NEVER removes the main repo/working tree (app-level
      guard, not left to git's own refusal), refuses to remove a DIRTY
      worktree unless `force:true`, and rejects a path that is not a
      currently-registered worktree of the repo.
- [x] Both services are LOCAL-ONLY: no fetch/push, no git hooks beyond git's
      own worktree machinery.
- [x] Registered into the `git-actions.ts` named-action taxonomy
      (`WORKTREE_ADD_ACTION` / `WORKTREE_REMOVE_ACTION` in
      `GIT_ACTION_HANDLERS`) and exported through both barrels
      (`git/index.ts`, package `index.ts`).
- [x] Windows paths with spaces work correctly (verified with a real
      `git init` + `worktree add`/`remove` cycle on a path containing a
      space).
- [x] `npm run build` (`tsc -b`) passes.
- [x] Targeted `vitest run packages/workflow` passes (227/227, including 22
      new tests); every failure classified pre-existing vs new (none were
      pre-existing — see Validation).
- [x] `@axiom/cli-commands` single-ownership preserved — N/A, this increment
      touched no CLI files.
- [x] `@axiom/tui` stayed generic — not touched.

## Assumptions

Each is a deliberate, narrowest-change resolution of an ambiguity the task
left open (per the "resolve ambiguity yourself" guardrail):

1. **Wired into `GIT_ACTION_HANDLERS`** (rather than "export the services
   directly" — the brief's stated fallback): the pattern fits with zero
   friction (same confirm-gating, same `GitActionContext`/`gitRunner`
   injection shape as `role-branch`/`commit-sync`), and it costs nothing
   extra for a later increment (W2/W6) to declare a `worktree-add` /
   `worktree-remove` action on a transition without inventing the wiring
   itself. No transition declares these actions yet — that is deliberately
   left to a lifecycle-owning later increment (see Non-goals).
2. **No MCP tool surface added** (`mcp-tools/src/git-handlers.ts` left
   untouched, despite the outer brief's "if procede" note): the increment's
   own Scope boundaries say "ONLY: the two git worktree services + guards +
   action-taxonomy/barrel wiring + tests" — MCP tool exposure is a
   CLI/MCP-surface concern that belongs with the increment that actually
   drives worktree lifecycle (W2/W3, or the unified MCP of INC-A5), not with
   the bare primitives. Verified `git-handlers.ts`'s existing tests (7/7,
   part of `packages/mcp-tools`) are unaffected — it only imports
   `roleBranch`/`commitSync`/`gitStatus`.
3. **`assertWorktreePathOutsideRepo` rejects "inside repo" uniformly for
   BOTH `worktreeAdd` and `worktreeRemove`** (not just "never remove the
   main repo root"): the brief's guard list states the outside-repo
   constraint once, ahead of the add-specific and remove-specific bullets,
   framing it as a guard shared by both operations — applied literally as
   written.
4. **`branchName` already checked out elsewhere is NOT a bespoke guard**:
   if a caller asks `worktreeAdd` to attach a branch that's currently
   checked out in another worktree (or the main repo), git itself refuses
   the mutating command; that surfaces as an ordinary `git-command-failed`
   error, consistent with how the rest of this module lets git's own
   state-dependent refusals bubble up (e.g. `commitSync` doesn't
   pre-validate whether a commit will conflict — it lets git fail and wraps
   the failure).
5. **A dirty-status check that itself fails to run (e.g. the worktree
   directory vanished out-of-band) is treated as "can't verify clean" and
   handled the SAME as "dirty"** (requires `force`) — the conservative
   direction, and it keeps `worktreeRemove` from ever silently proceeding
   against a worktree it couldn't actually inspect.
6. **No `startPoint` argument for `worktreeAdd`** (which commit a NEW branch
   is based on): not requested by the brief's FINAL decisions, and adding it
   would be scope creep beyond "just the primitives" — new branches are
   created from the current `HEAD` of `repoRoot`, git's own default.

## Implementation notes

Files changed (all in `Axiom/`, the product monorepo):

- `packages/workflow/src/git/git-services.ts` — `assertWorktreePathOutsideRepo`,
  `worktreeAdd`, `worktreeRemove`, `WorktreeAddArgs`/`WorktreeAddResult`,
  `WorktreeRemoveArgs`/`WorktreeRemoveResult`, 4 new `GitServiceError` kinds,
  internal `canonicalize`/`listWorktrees`/`findRegisteredWorktree` helpers.
- `packages/workflow/src/git/git-actions.ts` — `WORKTREE_ADD_ACTION`,
  `WORKTREE_REMOVE_ACTION`, `handleWorktreeAdd`, `handleWorktreeRemove`,
  `resolveWorktreePath` helper, `GitActionContext.worktreePath`/`.force`/
  `.prune`, both registered into `GIT_ACTION_HANDLERS`.
- `packages/workflow/src/git/index.ts` — barrel exports.
- `packages/workflow/src/index.ts` — package-level barrel exports (this
  package re-exports the git module via an explicit named list, not
  `export *`, so both had to be updated).
- `packages/workflow/tests/git-worktree-services.test.ts` (new) — 22 tests.

### Learned: Windows short-name (8.3) path aliasing broke worktree lookup

`worktreeRemove` needs to confirm `worktreePath` is a currently-registered
worktree by cross-checking `git worktree list --porcelain`'s output. The
first implementation compared `path.resolve(worktreePath)` (the caller's
string) against `path.resolve(entry.path)` (git's reported string) — and
**every** real-`git`-backed `worktreeRemove` test failed
(`not-a-worktree`), even though the worktree unquestionably existed (the
preceding `worktreeAdd` in the same test had just succeeded).

Root cause: on this machine, `os.tmpdir()` returns the Windows **short
(8.3) name** form of the user's home directory
(`C:\Users\IGUTIE~1\AppData\Local\Temp`), while `git worktree list
--porcelain` always reports the **OS-canonical long name** with forward
slashes (`C:/Users/igutierrezz/AppData/Local/Temp/...`). `path.resolve`
normalizes slash direction but has **no knowledge of the filesystem** — it
cannot know `IGUTIE~1` and `igutierrezz` name the same directory, so the
string comparison never matched. Confirmed via a standalone diagnostic
script before patching (see Validation).

Fix: a `canonicalize(p)` helper that does `path.resolve(p)` then attempts
`fs.realpathSync.native(resolved)` (the **native** realpath — the default
JS `fs.realpathSync` does **NOT** expand Windows short names in this
environment; `.native` does), falling back to the plain resolved path when
the target doesn't exist yet or realpath otherwise fails (never throws).
Applied in `assertWorktreePathOutsideRepo` (so its returned, canonicalized
path flows through to everything downstream) and again in
`findRegisteredWorktree` (canonicalizing git's reported side too, since
that string never passed through the guard). This is a genuine
Windows-environment gotcha, not a test-only artifact — worth remembering for
any future code that string-compares a caller-supplied path against a
git-reported one.

### Design decisions (one-line reasons)

- **Array-of-arrays `mutatingArgs` + `for` loop**, even where there's always
  exactly one command (`worktreeAdd`): matches `roleBranch`'s exact shape,
  keeping every git-service function structurally uniform.
- **`WorktreeAddResult`/`WorktreeRemoveResult` live in the "Shared result
  shapes" section** (not inline with their Args), mirroring where
  `RoleBranchResult`/`CommitSyncResult`/`GitStatusResult` already live.
- **fs existence/stat checks wrapped in try/catch**, mapped to
  `worktree-path-exists`: the rest of the module never throws (git failures
  always come back as a `Result`); raw `fs.*Sync` calls are a new throw
  surface this module didn't have before, so they're contained the same way.
- **`worktreeExisted: true` only for a pre-existing EMPTY directory**: a
  non-empty existing path is rejected before this field is ever computed,
  so the two states (rejected vs. `worktreeExisted:true`) are mutually
  exclusive by construction — documented on the field itself.

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0 (both before and after
  the `canonicalize` fix).
- Targeted `npx vitest run packages/workflow` — **227/227 tests passed**
  across 16 files (includes the new `git-worktree-services.test.ts`, 22
  tests). First run surfaced 6 NEW failures, all traced to the single
  short-name/long-name path-aliasing root cause above (none were
  pre-existing — this test file, and the bug it caught, are both new in
  this increment); after the `canonicalize` fix, re-run: 0 failures.
- Diagnostic script (throwaway, run via `node -e`, not committed) confirmed
  the root cause empirically before patching: constructed a real temp repo
  + worktree via `os.tmpdir()`, printed `git worktree list --porcelain`'s
  raw output side-by-side with `path.resolve`, and confirmed
  `fs.realpathSync.native` (but not the default `fs.realpathSync`) resolves
  the short-name alias.
- Extra sanity check beyond the required scope: `npx vitest run
  packages/mcp-tools` (the direct consumer of this git module via
  `git-handlers.ts`) — **89/89 tests passed**, including
  `git-handlers.test.ts` (7/7) unaffected by these additive changes.

Test breakdown of the new file (22 tests): 4 pure-guard
(`assertWorktreePathOutsideRepo`), 9 `worktreeAdd` (5 real-`git`-backed:
new-branch happy path, existing-branch attach, dry-run preview, existing-empty-
dir allowed, space-in-path add+remove cycle; 4 fake-runner guard tests:
invalid branch name, main-repo-root, inside-repo, existing-non-empty-dir),
7 `worktreeRemove` (5 real-`git`-backed: clean removal, dirty refusal,
forced dirty removal, dry-run preview, prune option; 2 fake-runner:
not-a-worktree, main-repo-root refusal), 2 `git-actions` wiring smoke tests
(fake-runner, proving `GIT_ACTION_HANDLERS` dispatch + confirm-gating for
both new actions).

## Result

Implemented. All acceptance criteria met. `worktreeAdd`/`worktreeRemove` are
live in `@axiom/workflow`'s git module, on the identical preview/confirm +
`plannedCommands` contract as `roleBranch`/`commitSync`, registered as
named actions in `GIT_ACTION_HANDLERS`, and exported through both barrels.
No lifecycle, Execution entity, provisioning, provider isolation, harvest,
mode selection, or freshness work was touched (deferred to INC-W2..W7 per
Scope). A genuine Windows path-aliasing bug (short vs. long username form)
was found and fixed during validation, not just worked around in tests.

## General spec integration

Integrado en la spec canónica al cierre del batch INC-20260724-* (integración
única de toda la tanda). Los primitivos `worktreeAdd`/`worktreeRemove` se
integran como base del ciclo de ejecución en worktree (no como superficie
propia):

- `04_Flujos_SDD_y_Ciclo_de_Vida.md` — `worktreeAdd`/`worktreeRemove` como
  primitivas (dry-run/preview, guards de path fuera-del-repo, nunca el repo
  principal, rechaza worktree sucio salvo `force`, local-only) dentro del
  flujo de ejecución en worktree.
- `07_Gobierno_y_Seguridad.md` — local-only, guards de cleanup seguro.
- `01_Requisitos_Funcionales.md` — RF-AXM-047.
