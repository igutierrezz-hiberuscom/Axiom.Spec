# Increment: Per-worktree provider isolation (code-intel)

Status: closed
Date: 2026-07-24

## Goal

Make code-intel providers (`cmm` structural, `serena` symbolic) run ISOLATED
PER WORKTREE: each worktree gets its own index/cache, pointed at the worktree
path, never sharing a mutable graph across worktrees; freshness is checked
per worktree before critical analysis; and derived indexes are torn down when
a worktree is removed.

This is **INC-W4** of Cluster W in the Axiom evolution plan
(`giggly-imagining-moore.md`, "Cluster W — worktrees"), depending on **INC-W2**
(`INC-20260724-worktree-isolation-execution`, closed — the `Execution`
entity + execution-scoped paths), **INC-W3**
(`INC-20260724-worktree-provisioning`, closed — `provisionWorktreeExecution`
materializes per-worktree code-intel STATIC MCP config), and **INC-T1**
(`INC-20260724-cmm-replaces-graphify-codegraph`, closed — `cmm` is the sole
structural provider, `serena` symbolic, `cmm-freshness.ts` +
`.cmm/sync-state.json`). It is an explicit, user-approved graduation to the
full product lifecycle (per `Axiom.SDD/AGENTS.md`'s "Explicit Bootstrap
Limits" — worktree support is a deliberate exception this plan requests, not
speculative architecture).

## Context

Before this increment, every code-intel `ProviderClient` invocation
(`cmm-client.ts`, `serena-client.ts`, both built on
`shared.ts`'s `createCodeIntelProviderClient`) spawned its local MCP server
subprocess with `cwd: ctx.projectRoot` — hardcoded to the PROJECT root, with
no notion of "this call is running against a worktree". `cmm`'s freshness
pre-check (`cmm-freshness.ts`'s `ensureCmmFreshness`, from INC-T1) was called
with `ctx.projectRoot` too, so its `.cmm/sync-state.json` marker
(`resolveCmmSyncStatePath(projectRoot)`) was always read/written under the
project root — meaning two worktrees of the same project would, if ever
invoked live, have shared/collided on the SAME `.cmm/` index at the project
root (a mutable graph neither worktree actually owned). There was no primitive
to delete a worktree's derived code-intel state either.

INC-W3 already solved the STATIC half of this problem:
`provisionWorktreeExecution` calls `buildCodeIntelNativeServers(providers,
execution.worktreePath, ...)` (`native-mcp-launch.ts`), so a worktree's
GENERATED `.mcp.json`/`opencode.json` entries already point an IDE-launched
`cmm`/`serena` server at the worktree path via `--project <worktreePath>`.
What was missing was the RUNTIME seam: `invokeCapabilityLive` ->
`ProviderClient.invoke(capabilityId, input, ctx)` (`packages/providers/src/
invoke.ts`) forwards `ctx: ProviderInvokeContext` unchanged to whatever client
handles the call — and `ctx` (`types.ts`) had no field at all for "this call
is worktree-scoped", so `resolveLaunch`/`buildArgs`/the freshness pre-check
inside `cmm-client.ts`/`serena-client.ts` had no way to know to target
anything other than `ctx.projectRoot`.

Investigation also confirmed several places were ALREADY root-agnostic and
needed no change:
- `cmm-freshness.ts`'s `resolveCmmSyncStatePath`/`readCmmSyncState`/
  `writeCmmSyncState`/`ensureCmmFreshness` all take a plain `projectRoot:
  string` parameter — really just "the root to scope the marker under".
  Feeding a worktree path in already produces a worktree-scoped marker, with
  zero signature change.
- `native-mcp-launch.ts`'s `serenaMcpArgs`/`cmmMcpArgs` are likewise
  root-agnostic — INC-W3 already proved this by passing `worktreePath` in for
  the static-config case.
- `apps/cli/src/commands/workspace-code-intel.ts`'s `initializeCodeIntelIndexes`
  takes `repoPath: string` — already usable against a worktree path by a
  future caller, with zero change.
- `packages/providers/src/invoke.ts`'s `invokeCapabilityLive` and
  `packages/providers/src/project-registry.ts`'s `buildProjectProviderRegistry`
  both only thread `ctx`/register clients generically — neither needed a
  change either.

This confirmed the gap was narrow and precisely located: `types.ts` (the
`ctx` shape) + `cmm-client.ts`/`serena-client.ts` (the only two places that
read `ctx.projectRoot` directly for `cwd`/tool-argument/freshness purposes) +
a brand-new teardown primitive (nothing existed to delete a worktree's
derived `.cmm/`/`.serena` state).

## Scope

- `packages/providers/src/types.ts` — additive `ProviderInvokeContext.worktreeRoot?: string`.
- `packages/providers/src/code-intel/shared.ts` — new exported
  `resolveCodeIntelRoot(ctx)` helper (`ctx.worktreeRoot ?? ctx.projectRoot`),
  the single resolution function both clients call.
- `packages/providers/src/code-intel/cmm-client.ts` — `resolveLaunch`
  (`cwd`/`args`), the `code.knowledgeGraph` `buildArgs`'s `projectPath`, and
  the freshness pre-check's root argument to `ensureCmmFreshness` all switch
  from `ctx.projectRoot` to `resolveCodeIntelRoot(ctx)`.
- `packages/providers/src/code-intel/serena-client.ts` — `resolveLaunch`
  (`cwd`/`args`) switches from `ctx.projectRoot` to `resolveCodeIntelRoot(ctx)`.
- `packages/providers/src/code-intel/cmm-freshness.ts` — doc-comment-only
  addendum (no functional change) documenting why it is already
  worktree-aware for free.
- `packages/providers/src/code-intel/teardown.ts` (new) — the teardown
  primitive: `teardownWorktreeCodeIntel`, `CodeIntelTeardownTarget`,
  `TeardownWorktreeCodeIntelOptions`, `TeardownWorktreeCodeIntelResult`,
  `SERENA_CACHE_DIRNAME`, `DEFAULT_CODE_INTEL_TEARDOWN_DIRNAMES`.
- `packages/providers/src/index.ts` — barrel exports for the above.
- `apps/cli/src/commands/workspace-code-intel.ts` — doc-comment-only
  addendum (no functional change) confirming `initializeCodeIntelIndexes` is
  already worktree-capable.
- `packages/providers/tests/fixtures/stub-code-intel-mcp-server.mjs` —
  additive: the stub's successful tool-call response now also includes
  `cwd: process.cwd()`, so a test can assert the REAL OS-level working
  directory a spawned subprocess was launched with.
- `packages/providers/tests/code-intel-worktree-scope.test.ts` (new) — 10 tests.
- `packages/providers/tests/code-intel-teardown.test.ts` (new) — 8 tests.
- `apps/cli/tests/e2e/worktree-provider-isolation.e2e.test.ts` (new) — 1
  real-git integration test chaining `worktreeAdd`+`Execution` (W1+W2) ->
  `provisionWorktreeExecution` (W3) -> simulated per-worktree `cmm` index ->
  `teardownWorktreeCodeIntel` (this increment) -> `worktreeRemove` (W1).

## Non-goals

- Re-opening the provider SET — `cmm`/`serena` remain the only two
  code-intel providers (INC-T1, ADR-0031). No new provider id introduced.
- The full harvest + safe-cleanup ORCHESTRATION (kill processes -> harvest ->
  `git worktree remove` -> delete derived indexes) — INC-W5.
  `teardownWorktreeCodeIntel` is only the LAST of those 4 steps, provided
  here as a standalone, callable primitive with no orchestration, no
  process-killing, and no git operation.
- Execution-mode selection (in-place vs. worktree), install-time default,
  wiring per-worktree invocation into any user-facing command or MCP tool —
  INC-W6. Nothing in this increment constructs a `worktreeRoot`-carrying
  `ProviderInvokeContext` from a real command/agent session; that remains a
  future caller's job (most naturally INC-W5/W6, which already know an
  `Execution.worktreePath`).
- Freshness / staleness detection for SDD ARTIFACTS (increment/bug/plan
  documents) — INC-W7; unrelated to this increment's code-intel INDEX
  freshness (INC-T1's `cmm` auto-sync, made worktree-aware here).
- A git hook of any kind for reindexing/freshness — freshness/reindex stays
  on-demand (provider pre-check / doctor), matching INC-T1's own "no
  mandatory git hook" decision, unchanged here.
- `axiom doctor` changes — no doctor check today enumerates code-intel
  index/cache paths or worktree state; none was added (see Assumptions).
- Migrating Axiom's own repos (`Axiom.SDD`/`Axiom.Spec`) to any worktree
  model — out of scope for the whole plan.

## Acceptance criteria

- [x] A worktree-scoped invoke targets the worktree cwd (not `projectRoot`):
      proven by a REAL subprocess spawn (`code-intel-worktree-scope.test.ts`)
      against the stub MCP server, asserting the OS-level `process.cwd()` the
      subprocess reports equals the worktree root, not the project root.
- [x] Two worktrees resolve to DISTINCT index/cache paths (no sharing): proven
      both at the pure-function level (`resolveCodeIntelRoot` unit tests) and
      behaviorally (two real `createCmmProviderClient` invocations, scoped to
      two different worktree roots, produce two independent
      `.cmm/sync-state.json` markers — neither touches the other or the
      project root's own marker).
- [x] Project-scoped default is UNCHANGED when no worktree scope is given:
      every pre-existing test in `code-intel-clients.test.ts` (and the rest
      of `packages/providers/tests`) passes unmodified; new tests explicitly
      cover the "no `worktreeRoot`" fallback path for both cmm and serena.
- [x] The freshness marker is worktree-scoped: `ensureCmmFreshness` (reused
      verbatim, zero signature change) is called with
      `resolveCodeIntelRoot(ctx)`, which resolves to `ctx.worktreeRoot` when
      present — proven by real `createCmmProviderClient(...).invoke(...)`
      calls that persist `.cmm/sync-state.json` under the worktree root and
      leave the project root's own marker untouched.
- [x] Teardown removes the worktree's `.cmm`/serena cache and is a safe no-op
      when absent: `teardownWorktreeCodeIntel` unit-tested for both.
- [x] Teardown never touches the main repo's index: proven structurally (the
      primitive only ever computes `path.join(targetPath, dirname)` for the
      ONE target given — no implicit shared-root fallback exists in the
      implementation to touch) and by test (seed main-repo + 2 worktrees,
      tear down 1 worktree, assert the other worktree AND the main repo
      survive byte-for-byte).
- [x] An integration test exercises a REAL worktree (W1) -> provision (W3) ->
      two worktrees with independent simulated cmm index dirs -> teardown
      one -> the other's (and the main repo's) index survives ->
      `worktreeRemove` succeeds for both afterward.
- [x] `npm run build` (`tsc -b`) passes.
- [x] Targeted `vitest run` for `packages/providers`, `packages/isolation`,
      `packages/doctor` passes (34 files / 319 tests, zero failures); the
      full `apps/cli` suite (119 files / 1111 tests) passes with zero
      failures, including the two files the brief flagged as
      known-flaky-under-parallel-load (`context.test.ts`/
      `workspace-setup.test.ts` — both passed clean in this run).
- [x] Windows-specific: the teardown primitive never throws even when a
      directory removal fails or silently no-ops (locked file) — covered via
      an injectable `removeDirSync` seam (deterministic simulation, since a
      real OS-level lock is unreliable to reproduce portably), and Node's own
      `maxRetries`/`retryDelay` remediation is wired into the real default
      implementation.
- [x] `@axiom/cli-commands` single-ownership preserved — no `apps/cli`
      command file was touched except a doc-comment addendum;
      `@axiom/tui` untouched (stays generic).
- [x] The provider SET is unchanged (`cmm`/`serena` only) — no new provider
      id, no doctor change, no routing change.

## Open questions

None blocking. See Assumptions for the ambiguities resolved
(existing-precedent-first, per the "resolve ambiguity yourself" guardrail)
rather than left open.

## Assumptions

1. **A single `worktreeRoot?: string` field on `ProviderInvokeContext`**, not
   a nested `ProviderWorktreeScope`/`Execution` object: the brief asked for
   "a worktree-scoped root" (singular), and `@axiom/providers` has no
   dependency on `@axiom/isolation` today (verified via `package.json`) — a
   plain string keeps the seam decoupled, matching `types.ts`'s own
   documented "deliberately minimal" philosophy for this context type.
2. **`resolveCodeIntelRoot` lives in `code-intel/shared.ts`**, not as a
   generic seam-level concept in `types.ts`: only the code-intel providers
   need worktree-scoping today (per the brief's explicit framing —
   `cmm`/`serena`, not `filesystem`); `shared.ts` is already documented as
   "Shared helpers for the code-intel `ProviderClient`s", the natural home.
   Avoids introducing a general "effective root" concept with no other
   consumer yet (no speculative architecture).
3. **`cmm-freshness.ts` needed ZERO functional/signature change**: every one
   of its functions already takes a plain, arbitrary `projectRoot: string` —
   feeding it `resolveCodeIntelRoot(ctx)` (which may resolve to a worktree
   path) from the ONE call site in `cmm-client.ts` makes the whole freshness
   system worktree-aware for free. Confirmed by reading the file in full
   before editing (per the brief's "read first" instruction) — this is a
   deliberate no-change decision, not an oversight; documented with a doc
   comment addendum for future readers.
4. **`workspace-code-intel.ts` needed ZERO functional change** for the same
   reason: `initializeCodeIntelIndexes(args: { repoPath: string, ... })` was
   already root-agnostic (confirmed by reading its only real caller in
   `workspace-setup.ts`, which passes `roleRepo.path` — never anything
   `projectRoot`-specific). A doc-comment addendum records this was verified,
   not skipped.
5. **`teardownWorktreeCodeIntel` accepts `string | { worktreePath: string }`**
   (duck-typed), not an imported `@axiom/isolation` `Execution` type: this
   satisfies the brief's literal "`teardownWorktreeCodeIntel(worktreePath |
   execution)`" signature request without adding a new package dependency —
   a real `Execution` value structurally satisfies the type without an
   import, and a test proves this directly.
6. **`teardownWorktreeCodeIntel` is SYNCHRONOUS**, unlike the async
   `provisionWorktreeExecution` (W3): it only ever does local filesystem
   removal (no subprocess, no network) — nothing actually asynchronous to
   await. A future async orchestrator (W5) can call it as one more
   (synchronous) step trivially. Matches `writeCmmSyncState`/
   `readCmmSyncState`'s existing sync precedent in the same folder.
7. **Default teardown dirnames are exactly `.cmm` (imported constant,
   `CMM_INDEX_DIRNAME`) and `.serena` (new, documented ASSUMPTION —
   `SERENA_CACHE_DIRNAME`)**, extensible via `options.extraDirnames`: mirrors
   the SAME "confirmed cmm convention, documented-assumption serena
   convention, both overridable" pattern `cmm-client.ts`/`native-mcp-
   launch.ts` already established — no local Serena checkout was available
   in this environment to independently verify its exact on-disk cache
   dirname, so it is flagged the same way, not silently assumed.
8. **`removeDirSync` is an injectable seam on `teardownWorktreeCodeIntelOptions`**,
   defaulting to `fs.rmSync(dir, { recursive: true, force: true, maxRetries:
   3, retryDelay: 100 })`: reproducing a REAL OS-level file lock
   deterministically and portably in a unit test is unreliable (Node's own
   recursive-`rm` implementation already retries around several
   Windows-lock-shaped failures, and `chmod`-based simulations are not
   guaranteed to actually block removal). An injectable primitive — mirroring
   this codebase's OWN existing seam conventions (`gitRunner`, `now`,
   `syncCommand`) — gives a 100%-deterministic test of the "never throws,
   reports a warning" contract instead.
9. **No `axiom doctor` change**: searched `packages/doctor/src` for any
   existing worktree/code-intel-index-path coupling before concluding none
   was needed — the only hit was an unrelated comment in `checks.ts`
   referencing `initializeCodeIntelIndexes`'s best-effort discipline by
   analogy, not an actual dependency on its paths. The brief's FINAL decision
   list also does not list doctor work for this increment (unlike the plan's
   original file-list draft, which is superseded by the FINAL decisions).
10. **No wiring of a real caller that sets `ctx.worktreeRoot`**: this
    increment provides the PLUMBING (context field, resolution helper,
    client updates, teardown primitive) and proves it end-to-end against
    REAL worktrees in its own integration test — but does not wire any
    command, MCP tool, or agent-facing call site to actually construct a
    worktree-scoped `ProviderInvokeContext` during a live session. That is
    squarely INC-W5/W6 territory (the orchestrator that actually knows when
    an agent is "running inside worktree X").

## Implementation notes

Files changed (all in `Axiom/`, the product monorepo):

- `packages/providers/src/types.ts` — additive `ProviderInvokeContext.worktreeRoot?: string`.
- `packages/providers/src/code-intel/shared.ts` — new `resolveCodeIntelRoot`.
- `packages/providers/src/code-intel/cmm-client.ts` — `resolveLaunch`,
  `code.knowledgeGraph`'s `buildArgs`, and the freshness pre-check now call
  `resolveCodeIntelRoot(ctx)`.
- `packages/providers/src/code-intel/serena-client.ts` — `resolveLaunch` now
  calls `resolveCodeIntelRoot(ctx)`.
- `packages/providers/src/code-intel/cmm-freshness.ts` — doc-comment
  addendum only.
- `packages/providers/src/code-intel/teardown.ts` (new) —
  `teardownWorktreeCodeIntel` + types + constants.
- `packages/providers/src/index.ts` — barrel exports.
- `apps/cli/src/commands/workspace-code-intel.ts` — doc-comment addendum only.
- `packages/providers/tests/fixtures/stub-code-intel-mcp-server.mjs` —
  additive `cwd: process.cwd()` in the stub's success response.
- `packages/providers/tests/code-intel-worktree-scope.test.ts` (new).
- `packages/providers/tests/code-intel-teardown.test.ts` (new).
- `apps/cli/tests/e2e/worktree-provider-isolation.e2e.test.ts` (new).

### `resolveCodeIntelRoot` — the one resolution function

```ts
export function resolveCodeIntelRoot(ctx: ProviderInvokeContext): string {
  return ctx.worktreeRoot ?? ctx.projectRoot;
}
```

Both `cmm-client.ts`'s and `serena-client.ts`'s `resolveLaunch` call this for
`cwd` (and, for `serena`, the `--project` arg baked into `args` too); `cmm`'s
`code.knowledgeGraph` `buildArgs` calls it for `projectPath`; `cmm`'s
freshness pre-check calls it as the root passed to `ensureCmmFreshness`. One
function, four call sites, zero duplication of the `?? ` fallback logic.

### `teardownWorktreeCodeIntel` signature

```ts
type CodeIntelTeardownTarget = string | { readonly worktreePath: string };

function teardownWorktreeCodeIntel(
  target: CodeIntelTeardownTarget,
  options?: {
    extraDirnames?: readonly string[];
    removeDirSync?: (dirPath: string) => void; // test seam
  },
): {
  targetPath: string;
  removed: readonly string[];
  absent: readonly string[];
  warnings: readonly string[];
};
```

### Design decisions (one-line reasons)

- **`worktreeRoot` as a flat string field, not a nested scope object** — see
  Assumption 1; keeps `@axiom/providers` free of a new `@axiom/isolation`
  dependency.
- **`resolveCodeIntelRoot` in `code-intel/shared.ts`, not `types.ts`** — see
  Assumption 2; only code-intel providers need this today.
- **Zero changes to `cmm-freshness.ts`/`workspace-code-intel.ts` beyond doc
  comments** — see Assumptions 3–4; both were ALREADY root-agnostic, so the
  worktree-awareness came entirely from the call site (`cmm-client.ts`)
  passing a different root, not from changing either module's contract.
- **Duck-typed `CodeIntelTeardownTarget` union** — see Assumption 5; matches
  the brief's literal signature ask with zero new package dependency.
- **Synchronous teardown primitive** — see Assumption 6; no async work exists
  inside it.
- **`.serena` flagged as a documented ASSUMPTION, `.cmm` reused from the
  already-confirmed `CMM_INDEX_DIRNAME` constant** — see Assumption 7;
  mirrors this codebase's existing "confirmed vs. assumed, both overridable"
  discipline for these exact two tools.
- **Injectable `removeDirSync` test seam** — see Assumption 8; deterministic
  over relying on real OS file-lock behavior.
- **Stub MCP server fixture extended additively (`cwd` field added to its
  JSON response)** rather than creating a brand-new fixture file: every
  existing assertion in this codebase's tests reads specific fields off the
  parsed JSON, never a full-object equality check, so this is provably
  non-breaking (confirmed by re-running every existing test that uses this
  fixture).

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0, no output.

- `npx vitest run packages/providers packages/isolation packages/doctor`:

  ```
   Test Files  34 passed (34)
        Tests  319 passed (319)
     Start at  08:53:08
     Duration  12.26s (transform 3.37s, setup 6ms, collect 32.59s, tests 39.09s, environment 19ms, prepare 16.84s)
  ```

  Includes the 2 new test files added by this increment:
  `packages/providers/tests/code-intel-worktree-scope.test.ts` (10 tests) and
  `packages/providers/tests/code-intel-teardown.test.ts` (8 tests). Zero
  failures, zero pre-existing failures encountered.

- `npx vitest run apps/cli/tests/e2e/worktree-provider-isolation.e2e.test.ts`:

  ```
   ✓ apps/cli/tests/e2e/worktree-provider-isolation.e2e.test.ts (1 test) 2655ms
   Test Files  1 passed (1)
        Tests  1 passed (1)
  ```

- `npx vitest run apps/cli/tests/workspace-code-intel.test.ts
  apps/cli/tests/workspace-worktree-provision.test.ts
  apps/cli/tests/e2e/worktree-provision.e2e.test.ts
  apps/cli/tests/workspace-mcp.test.ts apps/cli/tests/workspace-setup.test.ts`:

  ```
   Test Files  5 passed (5)
        Tests  89 passed (89)
  ```

- `npx vitest run apps/cli/tests` (the FULL apps/cli suite, for maximum
  regression confidence given the doc-comment-only touch to
  `workspace-code-intel.ts`):

  ```
   Test Files  119 passed (119)
        Tests  1111 passed (1111)
     Start at  08:54:14
     Duration  53.77s (transform 19.42s, setup 25ms, collect 169.11s, tests 250.68s, environment 69ms, prepare 61.68s)
  ```

  118 -> 119 files / 1110 -> 1111 tests versus INC-W3's own last-recorded
  baseline — exactly +1 file / +1 test (this increment's new e2e test), zero
  regressions. Both tests the brief flagged as known-flaky-under-parallel-load
  (`context.test.ts` / `workspace-setup.test.ts`) passed clean in this run.

No pre-existing failures were encountered or needed classification in this
increment's validation runs.

## Result

Implemented. `ProviderInvokeContext` gained an additive `worktreeRoot?:
string` field; `resolveCodeIntelRoot` (new, `code-intel/shared.ts`) is the
single function that resolves it against the pre-existing `projectRoot`
fallback. Both `cmm-client.ts` and `serena-client.ts` now call it for their
spawned subprocess's `cwd` (and, for `cmm`, the `projectPath` tool argument
and the freshness pre-check's root) — when a worktree-execution context is
given, the subprocess, the tool's own "which project" argument, and the
on-disk index/cache marker (`.cmm/sync-state.json`, serena's own `.serena/`
cache — both implicitly rooted at whatever path is launched against) all
bind to the SAME worktree path, so two worktrees never share a mutable
structural index. `cmm-freshness.ts` and `workspace-code-intel.ts` needed no
functional change at all — both were already root-agnostic, confirmed by
reading them in full before editing and documented with addendum comments.
`teardownWorktreeCodeIntel` (new) is a synchronous, best-effort, safe-no-op,
single-target-only primitive that deletes a worktree's derived `.cmm`/
`.serena` state — the last step of INC-W5's future harvest/cleanup
orchestration, provided here standalone. An integration test exercises a
REAL git worktree pair end-to-end: `worktreeAdd`+`Execution` (W1+W2) ->
`provisionWorktreeExecution` (W3) -> two independent simulated cmm indexes ->
`teardownWorktreeCodeIntel` on one -> the other (and the main repo) survives
-> `worktreeRemove` succeeds for both.

## General spec integration

Integrado en la spec canónica al cierre del batch INC-20260724-* (integración
única de toda la tanda). Ficheros que recibieron el conocimiento de este
incremento:

- `06_Integraciones_y_Capacidades.md` — aislamiento de providers por worktree
  (`ProviderInvokeContext.worktreeRoot`, `resolveCodeIntelRoot`, índice/caché
  por worktree) + `teardownWorktreeCodeIntel`.
- `07_Gobierno_y_Seguridad.md` — un índice por worktree (nunca compartido);
  teardown nunca toca el índice del repo principal.
- `08_Glosario.md` — `teardownWorktreeCodeIntel`.
- `02_Requisitos_No_Funcionales.md` — NFR-AXM-019 (aislamiento por worktree).
- `01_Requisitos_Funcionales.md` — RF-AXM-047.
