# Increment: SDD artifact freshness (auto-fetch + staleness warning)

Status: closed
Date: 2026-07-24

## Goal

Detect when an agent is about to read/edit a STALE SDD artifact (an
increment/bug/plan folder that has newer committed changes on the remote of
the shared `<project>.axiom`/spec repo), via a read-only auto-fetch +
staleness comparison, and **warn** (never silently proceed on stale, never
hard-block either). Plus ensure SDD-artifact writes push SCOPED to just that
artifact's folder/affected files — never a repo-wide `add -A`.

This is **INC-W7** of Cluster W in the Axiom evolution plan
(`giggly-imagining-moore.md`, "Cluster W — worktrees (depende de A5 + T1)"),
the LAST worktree item. It is an explicit, user-approved graduation to the
full product lifecycle (per `Axiom.SDD/AGENTS.md`'s "Explicit Bootstrap
Limits"): the user's own words, verbatim intent, ask for this — worktrees
share the central `<project>.axiom` repo, so periodic pulls / auto-fetch are
desirable to detect when an agent is working on an outdated increment/bug/plan
that has new remote changes; each worktree pushes ONLY its own
increment/bug folder (or just the affected files).

## Context

`packages/workflow/src/git/git-services.ts` already has `commitSync`
(stage + commit locally, push only when `push:true`, and ALREADY scopes to
`paths` via `git add -- <paths>` when given, falling back to `git add -A`
only when `paths` is omitted — D-GIT-003/existing precedent), `gitStatus`
(read-only porcelain), and `createGitRunner`/`GitRunner` (an injectable seam
over `spawnSync('git', ...)`). `packages/mcp-tools/src/artifact-handlers.ts`
exposes `spec.incrementRead`/`planRead`/`bugRead` (all funnel through a
shared `readArtifact(kind, input)` backed by
`@axiom/workflow`'s `loadArtifactMetadata`); `git-handlers.ts` exposes
`sdd.gitCommitSync` (wraps `commitSync` behind the preview/confirm contract)
and `sdd.transitionApply` (`transition-handlers.ts`) mutates a PER-WORKFLOW-
KIND state file (`workflow-state.json`, keyed by `workflowId` like
`'increment'`, not by a specific artifact instance id — see Assumption 2).
Nothing today periodically re-syncs the shared `<project>.axiom`/spec clone
with its own remote, so an agent could silently read/edit an increment/bug/
plan folder that is already superseded upstream (e.g. another worktree's
agent already pushed a newer version of the same artifact).

This is GIT-level artifact freshness (does the LOCAL clone's on-disk
increment/bug/plan folder lag its own remote-tracking branch) — DISTINCT
from INC-W4's code-intel/provider INDEX freshness (whether a structural
search index over CODE is stale). No code from W4 is touched here.

## Scope

- `packages/workflow/src/git/git-runner.ts`: additive `GitRunnerOptions`
  (`{ timeoutMs?: number }`) on `createGitRunner(options?)` — time-bounds a
  real runner's `spawnSync` calls. `createGitRunner()` (no args) is
  byte-for-byte unchanged for every existing caller.
- `packages/workflow/src/git/artifact-freshness.ts` (new): `checkArtifactFreshness(args)`
  — best-effort, GIT-level freshness of one-or-more artifact paths against
  their upstream. NEVER throws, NEVER returns an error — always resolves to
  `{ status: 'fresh'|'stale'|'unknown', ... }` (see Implementation notes for
  the exact algorithm/signature).
- `packages/workflow/src/git/index.ts` + `packages/workflow/src/index.ts`:
  barrel exports for the above.
- `packages/mcp-tools/src/types.ts`: additive `McpToolWarning` + optional
  `warnings?: ReadonlyArray<McpToolWarning>` on `McpToolResult`'s `ok:true`
  branch. Every pre-existing handler/call site is unaffected (optional,
  additive; omitted entirely when there is nothing to warn about, never
  `warnings: []`, so exact-equality tests like
  `expect(result).toEqual({ ok:true, data:null })` keep passing).
- `packages/mcp-tools/src/artifact-handlers.ts`: `readArtifact` (backing
  `getPlan`/`getIncrement`/`getBug`) attaches a `stale-artifact` warning when
  `checkArtifactFreshness` reports `'stale'` for that artifact's folder.
  `GetArtifactInput` gains an optional test-only `gitRunner`.
- `packages/mcp-tools/src/git-handlers.ts`: `gitCommitSync` attaches the same
  warning kind when the paths it is about to commit are themselves stale
  against the remote (reuses the SAME `paths` the commit is scoped to).
- `packages/mcp-server/src/server.ts`: `toolCallOkResult` appends warnings as
  an EXTRA `content[]` text block (never replaces/reorders `content[0]`, the
  pure JSON data payload every existing test parses) so a real MCP client
  actually sees the advisory, not just an in-memory field nobody reads.
- Tests: new `packages/workflow/tests/artifact-freshness.test.ts` (fake
  `GitRunner` cases + one real two-clone `git`-backed case); additions to
  `packages/workflow/tests/git-services.test.ts` (scoped-add pathspec,
  never `-A`, assertion), `packages/mcp-tools/tests/artifact-handlers.test.ts`,
  `packages/mcp-tools/tests/git-handlers.test.ts`, and one real-git test in
  `packages/mcp-server/tests/server.test.ts`.

## Non-goals

- INC-W4's code-intel/provider index freshness — untouched.
- The worktree lifecycle itself (W1-W6: worktree add/remove, per-worktree
  isolation/`Execution`, provisioning, provider/index isolation, harvest,
  mode selection) — untouched.
- Any MANDATORY git hook (pre-commit/post-checkout/etc.) — explicitly
  forbidden by the brief; auto-fetch is on-demand (at read/edit time) only.
- A caching/interval layer that throttles repeat fetches (e.g. "only fetch
  once every N minutes") — not requested; each read/edit's fetch is already
  cheap (single scoped branch, time-bounded) and the two local-only
  short-circuits (no git repo / no upstream configured) cost zero network
  calls, so this was judged unnecessary speculative complexity for now.
  Documented as a future consideration if real-world usage shows repeated
  fetches are costly.
- A new `axiom doctor` check — the brief's doctor mention was conditional
  ("if you add a freshness check there"); doctor checks in this codebase are
  fast/offline (e.g. `GW-001`'s hash comparison), while this is a
  best-effort NETWORK call — a meaningfully different operational profile.
  Documented as a future consideration, not implemented now (see Assumption 6).
- Migrating Axiom's own repos (`Axiom.SDD`/`Axiom.Spec`) to any new model.
- Any change to `apps/cli/src/commands/app-launcher-panels.ts`'s
  `apiCommitSync` (the launcher's REST/browser-panel surface) — it already
  passes `body.paths` straight through to `commitSync` unchanged (no `-A`
  bug there either); the brief scopes the warning wiring to the MCP surface
  (`incrementRead`/`bugRead`/`planRead`/`gitCommitSync`), a different,
  agent-facing consumption surface. Documented as a future consideration.
- `sdd.transitionApply` was NOT wired with a freshness warning directly —
  see Assumption 2 for why its input shape doesn't carry an artifact path.

## Acceptance criteria

- [x] A new git-level freshness service exists in `@axiom/workflow`'s git
      module, given a repo root + one-or-more artifact paths, that performs
      a READ-ONLY, time-bounded, best-effort `git fetch` of just the
      artifact's current upstream branch (never `--all`) and reports
      `fresh | stale | unknown` (+ `remoteCommitsAhead` when determinable).
- [x] `fresh` artifact (no remote commits touching the path ahead of HEAD) →
      no warning.
- [x] `stale` artifact (remote has newer commits touching the path) → a
      `stale-artifact` warning is surfaced, never a hard error/block.
- [x] No-remote / offline / no-upstream-configured / not-a-git-repo / path-
      escape / empty-paths → `unknown`, degrades gracefully, NEVER throws,
      NEVER attaches a warning (avoids false positives when we simply
      couldn't tell).
- [x] The warning surfaces at the point of read (`spec.incrementRead`/
      `spec.bugRead`/`spec.planRead`) AND at the point of write
      (`sdd.gitCommitSync`, keyed off the same scoped `paths`) — reaching
      the actual MCP `tools/call` response (`content[]`), not just an
      in-memory field.
- [x] Scoped push is verified end-to-end: `commitSync`'s `add` command uses
      `git add -- <paths>` (never `-A`) whenever `paths` is supplied, and
      `sdd.gitCommitSync`/`apiCommitSync` both pass a caller's `paths`
      through unchanged — asserted by a dedicated pathspec test, not just
      inferred from reading the code.
- [x] No mandatory git hook introduced anywhere.
- [x] `npm run build` (`tsc -b`) passes.
- [x] Targeted `vitest run` for `packages/workflow`, `packages/mcp-tools`,
      `packages/mcp-server`, `packages/doctor` — all pass; pre-existing vs
      new failures classified.
- [x] New unit tests use an injectable fake `GitRunner`; at least one test
      uses a REAL local two-clone `git` "remote" (`file://`-equivalent local
      clone, never a real network remote, never this workspace repo).
- [x] `@axiom/cli-commands` single-ownership preserved (not touched);
      `@axiom/tui` stayed generic (not touched).

## Assumptions

Each is a deliberate, narrowest-change resolution of an ambiguity the task
left open (per the "resolve ambiguity yourself, existing-precedent-first"
guardrail):

1. **`checkArtifactFreshness` NEVER returns a `Result`-style error — it
   always resolves to a value** (unlike `commitSync`/`gitStatus`/`roleBranch`,
   which return `Result<T, GitServiceError>`). This deliberately deviates from
   this file's sibling-function convention: the whole point of this function
   is "detect, don't block" / "never fail the read/edit" (brief, verbatim) —
   forcing every caller to unwrap an error branch that by design should never
   fire adds ceremony without safety benefit. Every failure mode (no repo, no
   upstream, fetch failure/timeout, path-escape, empty paths) folds into
   `status: 'unknown'` + a `reason` string instead.
2. **`sdd.transitionApply` was NOT wired with a freshness warning.**
   `TransitionApplyInput` operates on `workflow-state.json`, ONE record PER
   WORKFLOW KIND (`workflowId` like `'increment'`/`'bug'`/`'plan'`), not per
   specific artifact instance id — there is no artifact folder path to check
   freshness against in that shape (unlike `metadata.yml`'s folder-per-
   instance artifact store). Rather than bolt an artifact-path concept onto a
   tool whose job is generic FSM-state transition, the write-side warning is
   covered by `sdd.gitCommitSync` instead (which DOES carry `paths` — the
   exact shape needed, and the tool that actually pushes the artifact's
   folder). The brief's own phrasing ("`transitionApply`/`gitCommitSync`
   write path", "and/or") accommodates covering the write side via either.
3. **`checkArtifactFreshness`'s artifact-path parameter is `paths:
   ReadonlyArray<string>` (plural, mirrors `CommitSyncArgs.paths`), not a
   single `artifactPath: string`.** The brief's prose uses the singular ("an
   artifact path"), which fits the READ side (one increment/bug/plan folder)
   exactly, but the WRITE side (`gitCommitSync`) can scope a commit to
   MULTIPLE paths at once. Making the primitive plural-first lets the write-
   side wiring reuse the exact same `paths` array the commit itself uses (one
   fetch, one `rev-list` call with an OR'd pathspec) instead of either
   forcing multiple fetches or duplicating the primitive. The read side
   simply passes a single-element array.
4. **A `stale` artifact never blocks; a warning is ADDITIVE on the `ok:true`
   branch, never `ok:false`.** Verbatim from the brief ("Warn — do NOT hard-
   block... an agent can still proceed"). `McpToolResult`'s `warnings` field
   is optional and omitted (not `[]`) when empty, preserving every existing
   exact-equality test.
5. **Fetch is scoped to exactly the artifact's current upstream branch**
   (`git fetch <remote> <branch>`, resolved via `git for-each-ref
   --format=%(upstream:remotename|short)`), never `--all`/every ref — "don't
   fetch the world" (brief, verbatim). A repo with no upstream configured for
   the current branch short-circuits to `unknown` with ZERO network calls
   (only two cheap local `for-each-ref` reads) — the common case for a
   detached/local-only sandbox or most unit tests.
6. **No new `axiom doctor` check.** The brief's doctor mention was
   conditional ("... + doctor if you add a freshness check there"). Doctor
   checks in this codebase (e.g. `GW-001`) are fast, local-only comparisons;
   this feature's core operation is a best-effort NETWORK call, a materially
   different operational profile for a check that might run in CI/pre-
   commit contexts expecting near-instant, offline-safe results. Documented
   as a future consideration rather than implemented now.
7. **`createGitRunner`'s new `options?: { timeoutMs?: number }` parameter is
   OPTIONAL and additive** — every existing zero-arg call site
   (`roleBranch`/`commitSync`/`gitStatus`/`worktreeAdd`/`worktreeRemove`/all
   their tests) is byte-for-byte unchanged; only `checkArtifactFreshness`'s
   own default (no caller-injected `gitRunner`) constructs a runner with a
   timeout, keeping the "time-bounded fetch" guarantee local to this feature.

## Implementation notes

### `checkArtifactFreshness` signature

```ts
export type ArtifactFreshnessStatus = 'fresh' | 'stale' | 'unknown';

export interface ArtifactFreshnessArgs {
  readonly repoRoot: string;
  readonly paths: ReadonlyArray<string>;   // mirrors CommitSyncArgs.paths
  readonly gitRunner?: GitRunner;          // test-only injectable
  readonly fetchTimeoutMs?: number;        // default 5000; ignored if gitRunner supplied
}

export interface ArtifactFreshnessResult {
  readonly repoRoot: string;
  readonly paths: ReadonlyArray<string>;
  readonly status: ArtifactFreshnessStatus;
  readonly remoteCommitsAhead?: number;    // set only when determinable
  readonly remoteRef?: string;             // e.g. 'origin/main'
  readonly fetchOk: boolean;               // true only if `git fetch` itself ran + exited 0
  readonly reason?: string;                // set whenever status is 'unknown'
}

export function checkArtifactFreshness(args: ArtifactFreshnessArgs): ArtifactFreshnessResult;
```

### Algorithm (every step best-effort; ANY failure short-circuits to `unknown` + a `reason`, never throws)

1. Empty `paths` → `unknown` (`reason: 'no-paths-given'`), zero git calls.
2. Path-escape guard (`resolveInsideRepo`, same discipline as `commitSync`)
   for every path → `unknown` (`reason: 'path-escape'`) on violation, zero
   git calls beyond the guard itself (pure path math, no I/O).
3. `git rev-parse --is-inside-work-tree` → not a repo → `unknown`
   (`reason: 'no-git-repo'`).
4. `git rev-parse --abbrev-ref HEAD` → empty/`HEAD` literal (detached) →
   `unknown` (`reason: 'detached-head'`).
5. `git for-each-ref --format=%(upstream:remotename) refs/heads/<branch>`
   + `--format=%(upstream:short)` → either empty → `unknown`
   (`reason: 'no-upstream-configured'`), zero network calls so far.
6. `git fetch --quiet <remote> <branch>` (READ-only from the working tree's
   perspective — updates only the remote-tracking ref, never the local
   branch/index/working files) via a runner that is TIME-BOUNDED
   (`spawnSync`'s own `timeout`, default 5000ms) unless the caller injected
   their own `gitRunner`. Fetch failure/timeout → `unknown`
   (`reason: 'fetch-failed'`, `fetchOk: false`, `remoteRef` still reported).
7. `git rev-list --count HEAD..<remote>/<branch> -- <paths...>` → parses to
   `remoteCommitsAhead`; `>0` → `stale`, `0` → `fresh`. An unparseable/failed
   count → `unknown` (`reason: 'rev-list-failed'`/`'rev-list-unparseable'`).

### Warning wiring

- `artifact-handlers.ts`'s `readArtifact(kind, input)` (kind ∈
  `increment`/`bug`/`plan` only — `adr`/`decision` untouched, per the brief's
  explicit tool list) resolves the artifact's folder via the existing
  `resolveArtifactDir` + `path.relative(projectRoot, artifactDir)`, then
  calls `checkArtifactFreshness({ repoRoot: projectRoot, paths: [relPath],
  gitRunner: input.gitRunner })`. `status === 'stale'` → one `stale-artifact`
  warning attached to the `ok:true` result; anything else → no warning
  (field omitted entirely).
- `git-handlers.ts`'s `gitCommitSync` reuses `input.paths` (when non-empty)
  for the SAME check — a stale scoped commit gets the same warning kind,
  computed BEFORE reporting the result (both in preview and confirmed mode,
  so an agent sees it even before deciding to actually commit).
- `mcp-server/src/server.ts`'s `toolCallOkResult(data, warnings?)` appends
  warnings as an EXTRA `content[]` element (`content[1]`), never touching
  `content[0]` (the pure JSON payload every existing `JSON.parse(result.
  content[0].text)` test depends on) — verified with a real two-clone `git`
  test exercising the FULL `tools/call` dispatch.

### Scoped push verification (no code changes needed beyond a dedicated test)

`commitSync` already does `paths.length > 0 ? ['add', '--', ...paths] :
['add', '-A']` (pre-existing, INC-20260711-git-services). `sdd.gitCommitSync`
(`git-handlers.ts`) and the launcher's `apiCommitSync`
(`app-launcher-panels.ts`) both already forward a caller's `paths` through
unchanged. The gap closed here is a DEDICATED test asserting the exact `add`
argv recorded by a fake runner (never inferred from reading the source) —
see Validation.

## Validation

Ran from `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (`tsc -b`) — **PASSED**, clean, exit 0.
- `npx vitest run packages/workflow` — **243/243 passed** (17 files), incl.
  the new `artifact-freshness.test.ts` (13 tests) and the extended
  `git-services.test.ts` (18 tests, +3 new scoped-push pathspec tests).
- `npx vitest run packages/mcp-tools` — **97/97 passed** (12 files), incl.
  the extended `artifact-handlers.test.ts` (16 tests, +4 new) and
  `git-handlers.test.ts` (11 tests, +4 new).
- `npx vitest run packages/mcp-server` — **64/64 passed** (6 files), incl.
  the extended `server.test.ts` (30 tests, +2 new real-git end-to-end
  warning-surfacing tests).
- `npx vitest run packages/doctor` — **182/182 passed** (18 files),
  unaffected (this package was not touched).
- Extra, beyond the required scope (this feature touches a shared
  `@axiom/mcp-tools` type + `implementation-context-handler.ts`, which
  internally calls `getPlan`/`getIncrement`): `apps/cli/tests/
  launcher-front-no-vscode.test.ts` + `apps/cli/tests/e2e/
  workspace-mcp.e2e.test.ts` (9/9), `apps/cli/tests/e2e/
  north-star-bundle.e2e.test.ts` (1/1, exercises `buildImplementationContext`),
  `packages/launcher` (42/42, only a type-only import of
  `McpToolCapabilityId`) — all passed.
- **No pre-existing failures encountered** in any of the above — every test
  file touched by this increment was 100% green before AND after, so there
  was nothing to classify as pre-existing-vs-new; all new tests are
  additions, none were modified/removed.

## Result

Implemented. All 9 acceptance criteria met (see checkboxes above). Summary:

- New `checkArtifactFreshness(args)` in `@axiom/workflow`'s git module
  (`packages/workflow/src/git/artifact-freshness.ts`) — best-effort,
  never-throws, never-errors GIT-level freshness of one-or-more artifact
  paths, via a single time-bounded scoped `git fetch` + `git rev-list
  --count`. Verified against BOTH a fake injectable `GitRunner` (9 unit
  cases covering every degrade path) and a REAL two-clone local "remote"
  (2 real-git tests: stale-then-fresh-after-merge, and no-remote-configured).
- Wired as a `stale-artifact` `McpToolWarning` (new, additive, optional field
  on `McpToolResult`) into `spec.incrementRead`/`spec.bugRead`/`spec.planRead`
  (`artifact-handlers.ts`) and `sdd.gitCommitSync` (`git-handlers.ts`,
  reusing the commit's own scoped `paths`). Confirmed the warning reaches
  the ACTUAL `tools/call` JSON-RPC response as an extra `content[]` element
  (`mcp-server/src/server.ts`'s `toolCallOkResult`), via a real two-clone
  end-to-end test — not just an in-memory field nobody reads.
- Confirmed (with a dedicated pathspec-asserting test, not just code
  reading) that `commitSync`'s scoped push guarantee already held and now
  has explicit regression coverage: `paths` supplied → `git add -- <paths>`,
  NEVER `-A`; `paths` omitted → the deliberate whole-repo `add -A` fallback,
  unchanged.
- No mandatory git hook introduced. `createGitRunner` gained an additive,
  optional `timeoutMs` — every pre-existing zero-arg call site is
  byte-for-byte unchanged.
- `@axiom/cli-commands` and `@axiom/tui` were not touched.

## General spec integration

Integrado en la spec canónica al cierre del batch INC-20260724-* (integración
única de toda la tanda). Ficheros que recibieron el conocimiento de este
incremento (una única sección — se eliminó la duplicada que este README
tenía):

- `04_Flujos_SDD_y_Ciclo_de_Vida.md` — freshness de artefactos SDD: auto-fetch
  best-effort + warning `stale-artifact` en read/edit (nunca bloquea); push
  acotado (`git add -- <paths>`, nunca `-A`).
- `06_Integraciones_y_Capacidades.md` — warning MCP `stale-artifact`
  (`McpToolResult.warnings`) en `spec.incrementRead`/`bugRead`/`planRead` y
  `sdd.gitCommitSync`.
- `07_Gobierno_y_Seguridad.md` — push acotado (nunca repo-wide); sin hook git
  obligatorio.
- `08_Glosario.md` — `checkArtifactFreshness` / warning `stale-artifact`.
- `02_Requisitos_No_Funcionales.md` — NFR-AXM-018 (best-effort/never-block,
  sin hooks).
- `01_Requisitos_Funcionales.md` — RF-AXM-048.
