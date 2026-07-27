# Increment: Worktree provisioning

Status: closed
Date: 2026-07-24

## Goal

A provisioning function that materializes into a freshly-created git worktree
everything an AI-agent session needs to operate against it: the portable
`.axiom` surface, an MCP config pointing at the `<project>.axiom` unified
`axiom` broker (INC-A5), per-worktree code-intel MCP config (serena/cmm), and
the execution-scoped `.axiom-state` layout (INC-W2) — reusing existing
materialization primitives (`configure`/`sync`/`workspace-mcp`/
`workspace-code-intel`/adapters), never reimplementing them. Portable content
must be separated from local/secret content (secrets are never copied).

This is **INC-W3** of Cluster W in the Axiom evolution plan
(`giggly-imagining-moore.md`, "Cluster W — worktrees"), depending on **INC-W1**
(`INC-20260724-git-worktree-services`, closed — `worktreeAdd`/`worktreeRemove`)
and **INC-W2** (`INC-20260724-worktree-isolation-execution`, closed — the
`Execution` entity, execution-scoped paths, `ExecutionStore`, and the
`createExecutionForWorktree` compose helper built SPECIFICALLY for this
increment to call), and on **INC-A5** (the unified `axiom` MCP broker). It is
an explicit, user-approved graduation to the full product lifecycle (per
`Axiom.SDD/AGENTS.md`'s "Explicit Bootstrap Limits" — worktree support is a
deliberate exception this plan requests, not speculative architecture).

## Context

Today, an `axiom configure` / `axiom sync` run materializes a project's
adapter surface (AGENTS.md / skills-lock / etc. — `sync.ts`'s
`materializeAdapterOutputs`, one branch per adapter target) and
`workspace-mcp.ts` + `native-mcp-config.ts` materialize NATIVE per-adapter MCP
config files (`.mcp.json`, `opencode.json`, etc.), all against a SINGLE
project root at a time. INC-A5 added `buildAxiomMcpBrokerEntry` (in
`workspace-mcp.ts`) — a helper explicitly built as "the plumbing a future
caller will need" but with NO caller yet. INC-W1 added local-only
`worktreeAdd`/`worktreeRemove` git primitives. INC-W2 added the `Execution`
entity (`id`, `projectId`, `artifactRef`, `repoId`, `branch`, `worktreePath`,
`state` — a 7-value enum including `created` and `provisioned`), the
`ExecutionStore` (CRUD, atomic tmp+rename), execution-scoped paths
(`buildExecutionScopedPaths(executionId, rootPath)` ->
`.axiom-state/executions/<id>/{config,mcp,outputs,logs,evidence,local}`), and
`createExecutionForWorktree` — a compose helper (`worktreeAdd` +
`ExecutionStore.create`) explicitly built for, and left unwired until, this
increment.

A freshly-created git worktree shares the source repo's `.git` but checks out
ONLY tracked files — `.axiom-state/` is (and remains) gitignored
project-wide, so a new worktree always starts with an EMPTY `.axiom-state/`
even when `axiom.yaml`/`axiom.config/*.yaml`/`axiom.spec/templates/*` are
tracked and arrive "for free" via the checkout. The gap this increment fills
is exactly what git checkout does NOT provide: the resolved, non-secret
project config (`init.json`/`install-profile.json`/`workspace.json`) needed
for the adapter generators to run against the worktree at all, the
worktree-pointed native MCP config, and the execution-scoped runtime-state
directories.

`@axiom/cli-commands` single-ownership (INC-20260703): a specific subset of
`apps/cli/src/commands/*.ts` (including `sync.ts`, `configure.ts`) is compiled
ONLY by the `@axiom/cli-commands` package (its `tsconfig.json` `include`s
those exact files by absolute path and `apps/cli`'s own `tsconfig.json`
`exclude`s them) — any OTHER, regular `apps/cli` file that needs something
from one of those must import it via the `@axiom/cli-commands` package barrel,
never by relative path, or the build double-compiles the same source file
under two project references.

## Scope

- `apps/cli/src/commands/sync.ts`: exported (previously private)
  `materializeAdapterOutputs` + its result type `MaterializeAdapterOutputsResult`
  — no behavior change, additive `export` keyword only.
- `packages/cli-commands/src/index.ts`: re-exports `materializeAdapterOutputs`
  + its type from `sync.ts`, so the new (regular, non-owned) `apps/cli` module
  below can import it through the package barrel per the single-ownership
  rule.
- `apps/cli/src/commands/workspace-worktree-provision.ts` (new): the
  provisioning function `provisionWorktreeExecution` + its argument/result
  types + the `PORTABLE_AXIOM_STATE_FILENAMES` allowlist constant. A REGULAR
  (non-cli-commands-owned) `apps/cli` file — composes `materializeAdapterOutputs`
  (via the `@axiom/cli-commands` barrel), `buildAxiomMcpBrokerEntry` +
  `resolveMcpLaunchCommand` (`./workspace-mcp`, relative import — also a
  regular file), `writeNativeMcpConfig` + `NATIVE_MCP_TARGETS`
  (`./native-mcp-config`, relative import), `buildCodeIntelNativeServers` +
  `readEnabledProviders` (`@axiom/providers`), and `buildExecutionScopedPaths`
  (`@axiom/isolation`). NOT wired into any CLI command, commander
  registration, or MCP tool — a plain, callable, exported function only
  (INC-W6's job).
- `apps/cli/tests/workspace-worktree-provision.test.ts` (new): 10 hermetic
  unit tests (no real git).
- `apps/cli/tests/e2e/worktree-provision.e2e.test.ts` (new): 1 real-git
  integration test chaining `createExecutionForWorktree` (W1+W2) ->
  `provisionWorktreeExecution` (this increment) -> `worktreeRemove` (W1).

## Non-goals

- Per-worktree provider/index RUNTIME isolation (`serena`/`cmm` process
  lifecycle, caching, teardown) — INC-W4. This increment only materializes
  the STATIC MCP config entries pointing at the worktree path; nothing about
  running/caching/tearing down the underlying tools.
- Harvest + safe cleanup orchestration (kill processes -> harvest evidence ->
  `git worktree remove` + delete derived indexes) — INC-W5. This increment's
  execution-scoped `.axiom-state` layout inside the worktree is EPHEMERAL
  scratch; reconciling/harvesting it into the `ExecutionStore`'s central
  location is explicitly W5's job.
- Execution-mode selection (in-place vs. worktree), install-time default,
  wiring `provisionWorktreeExecution` into any user-facing command or MCP
  tool — INC-W6. This increment exposes a CALLABLE FUNCTION only.
- Freshness / auto-fetch / staleness detection for SDD artifacts — INC-W7.
- The `.github/copilot-instructions.md` (B4) writer (`configure.ts`'s
  `writeCopilotForTarget`, needing `product.manifest.yaml`/`providers.yaml`/
  local-overlay reads) — out of scope; this increment reuses
  `materializeAdapterOutputs` (the SAME adapter-surface step `axiom sync`
  itself performs), not `configure`'s superset. A worktree provisioned by
  this increment gets exactly what a bare `axiom sync` would generate for its
  adapter target.
- `engram` code-intel/MCP wiring — out of scope; `buildCodeIntelNativeServers`
  only ever recognizes `cmm`/`serena` (`CODE_INTEL_PROVIDER_IDS`), so an
  `engram` entry in `workspace.json#providers` is silently ignored by
  construction, matching the plan's "ONLY: ... cmm/serena" framing for W3.
- Migrating Axiom's own repos (`Axiom.SDD`/`Axiom.Spec`) to any worktree
  model — out of scope for the whole plan.

## Acceptance criteria

- [x] `provisionWorktreeExecution` materializes, into
      `execution.worktreePath`: (1) the portable `.axiom` surface via
      `materializeAdapterOutputs`; (2) a native MCP config entry for the
      `axiom-mcp-broker` (via `buildAxiomMcpBrokerEntry`); (3) native MCP
      config entries for the project's enabled code-intel providers
      (`cmm`/`serena`) pointed at the WORKTREE path (via
      `buildCodeIntelNativeServers`); (4) the execution-scoped `.axiom-state`
      directory layout (via `buildExecutionScopedPaths`) — all reused, none
      reimplemented.
- [x] Only 3 explicitly-named, non-secret files
      (`init.json`/`install-profile.json`/`workspace.json`) are ever copied
      from the source project's `.axiom-state/<projectId>/` into the
      worktree's own `.axiom-state/<projectId>/`; `.axiom-state/local/**` (and
      any other project's `.axiom-state/<otherProjectId>/`) is NEVER read or
      copied. The execution-scoped `local` subfolder created inside the
      worktree is always EMPTY.
- [x] Best-effort: every sub-step is individually guarded; a sub-failure
      (missing/corrupt `install-profile.json`, a `writeNativeMcpConfig`
      warning, an unsupported adapter target, a `store.update` failure)
      degrades to a string in `result.warnings` and the function completes —
      it NEVER throws and NEVER rejects.
- [x] No-clobber: a pre-existing, unrelated native MCP server entry in
      `.mcp.json`/`opencode.json`/etc. is preserved (merge, not replace); a
      pre-existing copy of a portable `.axiom-state` file at the destination
      is never overwritten by the seeding step.
- [x] Created-gating: provisioning only runs when `execution.state ===
      'created'` AND `execution.worktreePath` exists on disk; any other state
      (or a missing worktree directory) short-circuits as a documented no-op
      (`attempted:false`, `skippedReason` set, nothing written).
- [x] `Execution` advances `created -> provisioned` via `store.update` ONLY
      when a `store` is passed to `provisionWorktreeExecution`; omitted by
      default, and NOT wired into any user-facing command (INC-W6's job).
- [x] `npm run build` (`tsc -b`) passes.
- [x] Targeted `vitest run` for `apps/cli/tests` (full suite, 118 files / 1110
      tests) passes with zero failures; `packages/isolation`,
      `packages/persistence`, `packages/adapters`, `packages/providers`,
      `packages/workflow` pass except one PRE-EXISTING, confirmed-flaky test
      in `packages/workflow/tests/git-worktree-services.test.ts` (unrelated
      file, not touched by this increment — see Validation).
- [x] Unit tests cover: happy path (all 4 materialization steps + their
      contents); secrets/local never copied (with a recursive
      file-content scan proving a planted "secret" string appears nowhere
      under the worktree); no-clobber (both the native MCP file and the
      copied state file); best-effort (missing profile, and a corrupt
      worktree-side copy that makes `materializeAdapterOutputs` throw
      internally); created-gating (wrong state, missing worktree dir);
      `Execution` state advancement (with/without `store`).
- [x] An integration test chains a REAL `worktreeAdd` (via
      `createExecutionForWorktree`) -> `provisionWorktreeExecution` -> asserts
      a working `.axiom` surface + MCP config in the worktree -> a REAL
      `worktreeRemove` succeeds afterward.
- [x] `@axiom/cli-commands` single-ownership preserved (`materializeAdapterOutputs`
      consumed via the package barrel, not by relative path); `@axiom/tui`
      untouched (stays generic).
- [x] No worktree creation, provisioning call, or state transition wired into
      any CLI command or MCP tool (INC-W6 territory) — `provisionWorktreeExecution`
      has test callers only in this increment.

## Open questions

None blocking. See Assumptions for the ambiguities resolved (existing-precedent-first,
per the "resolve ambiguity yourself" guardrail) rather than left open.

## Assumptions

1. **Allowlist, not a directory copy or a blacklist**: exactly 3 files are
   copied by name from the source's `.axiom-state/<projectId>/` —
   `init.json`, `install-profile.json`, `workspace.json`
   (`PORTABLE_AXIOM_STATE_FILENAMES`). This is a deliberately SAFER default
   than "copy the folder minus `local/`": a future file added under
   `.axiom-state/<projectId>/` is excluded by construction unless explicitly
   added to the allowlist, rather than being copied by default.
2. **The execution-scoped `.axiom-state` layout is materialized INSIDE the
   worktree** (`buildExecutionScopedPaths(execution.id, execution.worktreePath)`),
   as an EPHEMERAL, worktree-local scratch area — DISTINCT from wherever the
   caller's `ExecutionStore` is rooted (a decision INC-W2 left to the store's
   constructor caller, not redefined here). The natural choice for the
   store (used by this increment's own integration test) is the SOURCE repo
   root, so `execution.json`/`logsPath`/`evidencePath` survive worktree
   removal (needed for harvest, INC-W5) and are listable without entering
   every worktree. Reconciling the two is explicitly INC-W5's job.
3. **Created-gating trusts the CALLER-supplied `execution.state` snapshot**;
   it does NOT re-fetch from `store` even when one is passed (since `store`
   is optional, the function can't always re-fetch). The caller is
   responsible for passing a fresh snapshot (e.g., immediately after
   `store.create()`/`store.get()`) — consistent with `Execution` being a
   plain value object everywhere else in this codebase, not a live reference.
4. **`axiomRepoPath` defaults to `sourceRootPath` when omitted**: resolving
   the REAL `<project>.axiom` repo path is a `@axiom/topology` concern
   (out of this increment's scope to reimplement); today's common
   single-repo topology (code repo === axiom/control repo, as this very
   product repo demonstrates) makes `sourceRootPath` a correct default. A
   future caller with a split topology passes the real path explicitly.
5. **The axiom broker entry + code-intel entries are merged into ONE native
   MCP config file** (via a single `writeNativeMcpConfig` call with a
   combined `servers` array), mirroring exactly how `writeWorkspaceNativeMcpConfigs`
   already combines shared servers + per-repo code-intel servers for the
   multi-repo case — not two separate writes/files.
6. **No B4 (`copilot-instructions.md`) materialization**: `sync.ts`'s
   `materializeAdapterOutputs` (reused here) is the SAME step `axiom sync`
   itself calls, which does not include `configure.ts`'s B4 branch (needs
   `product.manifest.yaml`/`providers.yaml`/local-overlay reads). A
   worktree gets exactly what `axiom sync` would generate; adding B4 parity
   is future work if a worktree specifically needs that file too.
7. **`materializeAdapterOutputs` exported from `sync.ts` via the
   `@axiom/cli-commands` barrel**, not duplicated: the ONLY change to
   `sync.ts` is the `export` keyword (plus its result type) — zero behavior
   change to `axiom sync` itself.
8. **`workspace-worktree-provision.ts` lives as a regular `apps/cli` file**,
   not added to `@axiom/cli-commands`'s `include` list: nothing outside
   `apps/cli/tests` needs to consume it yet (no TUI/MCP-tools caller in this
   increment), so there is no single-ownership reason to move it.

## Implementation notes

Files changed (all in `Axiom/`, the product monorepo):

- `apps/cli/src/commands/sync.ts` — `export` added to
  `MaterializeAdapterOutputsResult` and `materializeAdapterOutputs`; no other
  change.
- `packages/cli-commands/src/index.ts` — re-exports `materializeAdapterOutputs`
  + its type from `sync.ts`.
- `apps/cli/src/commands/workspace-worktree-provision.ts` (new) —
  `provisionWorktreeExecution`, `ProvisionWorktreeExecutionArgs`/
  `ProvisionWorktreeExecutionResult`, `PORTABLE_AXIOM_STATE_FILENAMES`,
  internal helpers (`copyPortableStateFileNoClobber`, `isNativeMcpTarget`,
  `readResolvedInstallProfile`, `toWarningMessage`).
- `apps/cli/tests/workspace-worktree-provision.test.ts` (new) — 10 tests.
- `apps/cli/tests/e2e/worktree-provision.e2e.test.ts` (new) — 1 test (11 total).

### `provisionWorktreeExecution` signature

```ts
async function provisionWorktreeExecution(args: {
  execution: Execution;                              // @axiom/isolation (INC-W2)
  sourceRootPath: string;                             // repo the worktree was cut FROM
  axiomRepoPath?: string;                              // default: sourceRootPath
  store?: ExecutionStore;                              // @axiom/persistence (INC-W2)
  launchCommand?: McpLaunchBase;                       // test seam
  enabledCodeIntelProvidersOverride?: readonly string[]; // test seam
  codeIntelOptions?: CodeIntelNativeServerOptions;     // test seam
}): Promise<{
  executionId: string;
  worktreePath: string;
  attempted: boolean;
  skippedReason?: string;
  adapterSurface: { attempted: boolean; generatedFilesCount: number };
  mcpConfig: { attempted: boolean; path?: string; serverIds: readonly string[] };
  executionStateDirs: readonly string[];
  warnings: readonly string[];
  advancedToProvisioned: boolean;
}>
```

### Design decisions (one-line reasons)

- **Allowlist copy (3 named files) instead of a directory copy** — see
  Assumption 1; safety-first against unexpected future secrets.
- **`readResolvedInstallProfile` is a small LOCAL helper**, not a reuse of
  `sync.ts`'s private `readInstallProfileSync` — it has slightly different
  semantics (copy-then-read vs. read-and-degrade) and matches this
  codebase's OWN precedent of each command file owning a tiny local
  JSON-read helper (`configure.ts`/`sync.ts` both already do this
  independently for `init.json`).
- **A single, TS-narrowing `isNativeMcpTarget` guard** bridges the plain
  `ResolvedInstallProfile.adapterTarget: string` into `writeNativeMcpConfig`'s
  typed `target: AdapterTarget` parameter, so an unsupported target is
  skipped BEFORE the call (with an explicit warning) rather than relying
  solely on `writeNativeMcpConfig`'s own internal default-case warning.
- **One outer try/catch wraps the whole materialization body**
  (belt-and-suspenders) in addition to per-step try/catches: every failure
  mode found during design was already individually guarded, but the
  "provisioning must never throw" contract is strong enough to warrant the
  extra layer.
- **State advancement is a separate, final step**, run regardless of how
  many warnings were collected earlier — best-effort means the OVERALL
  attempt still completed, not that every sub-step was warning-free.

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — PASSED, clean, exit 0.
- `npx vitest run apps/cli/tests/workspace-worktree-provision.test.ts
  apps/cli/tests/e2e/worktree-provision.e2e.test.ts` — **11/11 passed**
  (2 test files).
- `npx vitest run apps/cli/tests` (the FULL apps/cli suite) — **118 test
  files / 1110 tests passed**, zero failures (includes the two files the
  brief flagged as known-flaky-under-parallel-load,
  `context.test.ts`/`workspace-setup.test.ts` — both passed clean in this
  run).
- `npx vitest run packages/isolation packages/persistence packages/adapters
  packages/providers packages/workflow` — 1 failure on the first run:
  `packages/workflow/tests/git-worktree-services.test.ts >
  worktreeAdd > attaches an EXISTING (not-checked-out-elsewhere) branch, no
  '-b'` — root cause: that PRE-EXISTING test (from INC-W1, never touched by
  this increment) asserts `plannedCommands.join(' ')` does NOT contain the
  substring `'-b'`, but its helper builds the worktree path as
  `wt-${Math.random().toString(36)...}`, and the random suffix
  occasionally starts with `b` right after the `wt-` prefix (e.g.
  `wt-breg2v9bppe`), which itself contains the substring `-b` — a false
  positive from a naive substring check colliding with random test-fixture
  data, NOT a real `-b` flag in the planned git command. Confirmed
  flaky-not-broken by re-running the SAME file in isolation immediately
  after: **22/22 passed clean**. This file and bug are entirely unrelated
  to this increment (no file this increment touches was in the failing
  path) — classified as a PRE-EXISTING, probabilistic flake, not a
  regression.

## Result

Implemented. `provisionWorktreeExecution` (new,
`apps/cli/src/commands/workspace-worktree-provision.ts`) composes 4 existing
materialization primitives — `materializeAdapterOutputs` (`sync.ts`, newly
exported), `buildAxiomMcpBrokerEntry` + `writeNativeMcpConfig`
(`workspace-mcp.ts`/`native-mcp-config.ts`), `buildCodeIntelNativeServers`
(`@axiom/providers`), and `buildExecutionScopedPaths` (`@axiom/isolation`) —
into a single, best-effort, no-clobber, created-gated function that
materializes everything an AI-agent session needs into a freshly-created
worktree, while allowlisting exactly 3 non-secret files as the only content
ever copied from the source project's `.axiom-state/`. It is a plain,
exported, callable function with zero CLI/MCP wiring, ready for INC-W6 to
invoke. `createExecutionForWorktree` (INC-W2's previously-unwired compose
helper) now has its first real caller, exercised end-to-end in this
increment's own integration test.

## General spec integration

Integrado en la spec canónica al cierre del batch INC-20260724-* (integración
única de toda la tanda). Ficheros que recibieron el conocimiento de este
incremento:

- `06_Integraciones_y_Capacidades.md` — `provisionWorktreeExecution`
  materializa la superficie `.axiom` portable + MCP unificado + code-intel
  por-worktree + `.axiom-state` execution-scoped (reusando primitivas).
- `04_Flujos_SDD_y_Ciclo_de_Vida.md` — paso de provisioning en el flujo de
  ejecución en worktree.
- `07_Gobierno_y_Seguridad.md` — portable-only, secretos nunca copiados
  (3 ficheros no-secretos).
- `08_Glosario.md` — `provisionWorktreeExecution`.
- `02_Requisitos_No_Funcionales.md` — NFR-AXM-017 (no filtración de secretos).
- `01_Requisitos_Funcionales.md` — RF-AXM-047.
