# Increment: Git services + script/action side-effect variant + git MCP tools

> **Código**: INC-20260711-git-services
> **Estado**: Implementado y gate-verde (build + suite vitest completa + typecheck + doctor PASS).
> **Fecha**: 2026-07-12
> **Tipo de cambio**: Nueva capacidad (porta KVP25 git services al core)
> **Epic padre**: `INC-20260711-sdd-launcher-core-port` (§5, D-004)
> **Referencias externas (read-only)**: `C:/repos/KVP25 Workspace/Kvp.Sdd/tools/sdd-launcher/src/services/{roleBranchService,gitSyncService}.ts`

---

## 1. Goal

Materialize the last cross-cutting slice of the sdd-launcher core port (D-004,
epic §5): bring KVP25's git services (`roleBranch` / `commitSync` / `gitStatus`)
into the Axiom core behind a `GitRunner` seam, expose them as (a) a new
**`script/action`** variant of the declared per-transition side-effect taxonomy
and (b) **confirm-gated git MCP action tools**, and wire an **opt-in** git
side-effect into the `role` workflow's `start` transition. This turns the MCP
surface + the workflow state machine into a real bidirectional control plane that
can *drive git* in a target/role repo — without ever mutating the workspace by
default.

## 2. Scope

### Incluido

- **`script/action` side-effect variant** — extend `TransitionEffects`
  (`packages/workflow/src/types.ts`) with an optional
  `actions?: NamedAction[]` (`{ name, args?, confirmRequired? }`), preserved
  through `workflows-loader.ts`, and a **pure action-dispatch runner**
  (`transition-effects.ts` `runTransitionActions`) that dispatches named actions
  to an injected registry of handlers. Additive + NO-OP: a transition declaring
  no actions behaves exactly as before.
- **Git services in core** behind a `GitRunner` seam
  (`packages/workflow/src/git/`): `roleBranch`, `commitSync` (default
  `push:false`), `gitStatus` (read-only porcelain). Path-escape guard: only
  operate inside the resolved target repo root. Registered as the `role-branch` /
  `commit-sync` named-action handlers (`git/git-actions.ts` `GIT_ACTION_HANDLERS`).
- **Role workflow wiring** — declare a `role-branch` action on the `role`
  workflow `start` transition (`default-workflows.ts` + `axiom.config/workflows.yaml`),
  consumed by a new OPT-IN `axiom-role start --create-branch [--commit] [--push]
  [--confirm]`. Default behavior (no flags) is unchanged: no git side-effect.
- **Confirm-gated git MCP tools** — `sdd.gitRoleBranch` and `sdd.gitCommitSync`,
  mirroring `sdd.transitionApply`: preview by default, mutate only with
  `{ confirmed: true }`, `projectRoot` pinned via `pinned()`, never push unless
  an explicit `push:true` is also set.

### Excluido (Non-goals)

- No auto-push on any default path. No branch/base-branch fetch/pull/rebase
  orchestration from KVP25's `bootstrapRoleBranch`/`gitSyncService` (which push
  eagerly) — only the local-only, confirm-gated subset is ported.
- No change to the default behavior of `axiom-role start` (git is behind a flag).
- No new package (`@axiom/git`) — git lives as a module in `@axiom/workflow`
  (D-GIT-001). No ADO/tracker work (that is the epic's P2, already shipped).
- `sdd.transitionApply` is NOT changed to run git (D-CRMW-003 stands: it is
  state-only; git is these new tools' / the CLI's responsibility).

## 3. SAFETY model (non-negotiable)

- The feature runs git in a **target/role repo**, never in the Axiom workspace
  repos. All git tests/e2e `git init` a **throwaway** repo under a fresh temp dir
  and operate only there; unit tests use an **injectable fake `GitRunner`** that
  records commands without touching disk.
- **Confirm-gated** (mirrors `sdd.transitionApply`'s `confirmed` model): without
  confirmation the tools/CLI **preview** (report the git commands that WOULD run
  + current porcelain status) and mutate nothing.
- **Never push by default**: `commitSync` is local-only; `push` requires an
  explicit `push:true` arg AND confirmation. Two independent guards (D-GIT-003).
- **Path-escape guard**: `commitSync` rejects any `paths` entry that resolves
  outside the target repo root; branch names with traversal/illegal ref chars are
  rejected.
- The `GitRunner` always runs with `cwd = <resolved target repo root>`, never
  `process.cwd()`, so a test running under the Axiom repo can never mutate it.

## 4. "script/action side-effect + git MCP tools" section

### 4.1 The `script/action` variant

```ts
interface NamedAction {
  readonly name: string;                              // e.g. 'role-branch' | 'commit-sync'
  readonly args?: Readonly<Record<string, unknown>>;
  readonly confirmRequired?: boolean;                 // gate; true for all git actions
}
interface TransitionEffects {
  readonly localYaml?: LocalYamlMutation | LocalYamlMutation[]; // P0 (unchanged)
  readonly tracker?: string;                                    // P0 (unchanged)
  readonly actions?: ReadonlyArray<NamedAction>;                // NEW (this increment)
}
```

`runTransitionActions(effects, ctx, registry)` (pure orchestration; generic over
the handler-context type so `@axiom/workflow` stays git-agnostic):

- no `actions` declared → `{ outcomes: [], noop: true }` (behavior preserved).
- for each action: unknown name → `unknown-action` outcome; `confirmRequired &&
  !ctx.confirmed` → the handler is invoked in PREVIEW mode (reports planned
  commands, mutates nothing); otherwise APPLY.

The pure P0 YAML runner (`applyTransitionEffects`) is untouched and stays a NO-OP
for `actions` (they never mutate YAML).

### 4.2 Git services + `GitRunner` seam

`GitRunner = (args, cwd) => { ok, code, stdout, stderr }` (default impl:
`spawnSync('git', …)`; injectable). Services return `Result<…, GitServiceError>`:

- `roleBranch(repoRoot, branchName, { dryRun, gitRunner })` — create/checkout the
  role branch (existing-branch handling ported from KVP25 `roleBranchService`),
  **never pushes**.
- `commitSync(repoRoot, message, { push=false, paths, dryRun, gitRunner })` —
  stage + commit locally; push only when `push:true` (and not dryRun).
- `gitStatus(repoRoot, { gitRunner })` — read-only porcelain (for previews).

### 4.3 Confirm-gated git MCP tools

`sdd.gitRoleBranch` / `sdd.gitCommitSync` (registered in
`@axiom/mcp-tools/registry.ts`, described in `@axiom/mcp-server/tool-sets.ts`,
pinned in `input-builders.ts`):

- Without `confirmed:true` → PREVIEW (`mode:'preview'`, planned commands, current
  status); mutate nothing.
- With `confirmed:true` → run via `GitRunner` in the **pinned** `projectRoot`;
  `gitCommitSync` pushes only if `push:true` is also passed.
- Cross-project `projectRoot` arg → `pinned()` rejection (reused verbatim).

## 5. Acceptance

- AC1: A transition declaring a `role-branch` action dispatches to the handler
  (fake runner records it); a transition with no actions is a runner NO-OP; a
  `confirmRequired` action is not applied without confirmation.
- AC2: In a real `git init` temp repo, `roleBranch` creates+checks out the branch;
  `commitSync` commits locally and does NOT push; `gitStatus` reads porcelain; a
  path-escape attempt is rejected.
- AC3: `axiom-role start --create-branch` without `--confirm` previews only (no
  branch created); with `--confirm` the branch is created in the temp role repo;
  default `axiom-role start` (no flag) is unchanged (no git) and all existing
  tests stay green.
- AC4: `sdd.gitRoleBranch` / `sdd.gitCommitSync` preview without confirm, mutate
  in the pinned temp repo when confirmed, never push without an explicit push
  arg, and reject a cross-project arg.
- AC5: Gate green — build + full vitest suite + typecheck + doctor, with the
  doctor dogfooding/write-scope boundary intact (git actions are runtime
  features, not workspace mutations).

## 6. Result

Implemented and gate-green. See `metadata.yml` decisions D-GIT-001..004.

- **`script/action` shape + runner**: `NamedAction` + `TransitionEffects.actions`
  in `packages/workflow/src/types.ts`; preserved through `workflows-loader.ts`;
  pure generic dispatch runner `runTransitionActions` (+ `hasTransitionActions`,
  `TransitionActionHandler`, `TransitionActionOutcome`) in
  `packages/workflow/src/transition-effects.ts`.
- **Git services**: `packages/workflow/src/git/{git-runner,git-services,git-actions}.ts`
  behind the injectable `GitRunner` seam; `commitSync` defaults `push:false`
  (no-push-by-default proven by test asserting the fake runner never records a
  `push`). Registered as `GIT_ACTION_HANDLERS` (`role-branch`, `commit-sync`).
- **Role opt-in**: `axiom-role start --create-branch [--commit] [--push]
  [--confirm]`; default unchanged. `role` `start` transition declares the
  `role-branch` action in both `default-workflows.ts` and
  `axiom.config/workflows.yaml` (kept in sync).
- **Git MCP tools**: `sdd.gitRoleBranch`, `sdd.gitCommitSync`
  (`packages/mcp-tools/src/git-handlers.ts`), registered + pinned; preview/confirm
  contract; no-push-by-default.
- **Proof git ran only in temp repos**: unit tests use a fake `GitRunner`;
  real-git tests `git init` throwaway `os.tmpdir()` mkdtemp repos and pass an
  explicit `repoRoot`; no git service ever defaults to `process.cwd()`.

## 7. Trazabilidad

- Parent epic: `INC-20260711-sdd-launcher-core-port` §5 (git services; script/action
  side-effect; MCP bidirectional control plane), decision D-004.
- Sibling: `INC-20260711-cross-repo-mcp-wiring` (the `sdd.transitionApply`
  confirm-gated MCP pattern these tools mirror).
- Source (read-only): KVP25 `roleBranchService.ts` / `gitSyncService.ts`.
