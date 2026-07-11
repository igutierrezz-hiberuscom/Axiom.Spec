# INC-20260711-per-role-review

## Goal

Move write-scope review from being conceptually done **only at ARCHIVE time from
the SPEC repo** (where validating all role repos is hard) to two complementary
surfaces:

1. **Per-role review at `role-complete`, inside that role's OWN repo.** When
   `axiom-role complete` runs in role X's repo, review that repo's real changes
   (its own `git diff`) against the plan's `allowedWriteScope` for that repo,
   and gate the completion on the result.
2. **Aggregate whole-increment validation runnable from the SPEC repo.** One
   command that reads the plan, resolves EVERY target repo to an absolute path
   via `LocalBindings`, diffs each, validates per-repo write-scope, and produces
   a consolidated per-repo + overall report.

Neither surface removes the pre-existing archive step; they add earlier and
broader validation points.

## Scope

- **A — Per-role review (`axiom-role complete`).** New module
  `apps/cli/src/commands/_role-review.ts` (`runRoleReview`) invoked as an
  explicit gate inside `runRoleSubcommand` when `subcommand === 'complete'`.
  - Identifies the current repo's topology id + roles (new exported helper
    `identifyCurrentRepoIdentity` in `_repo-affinity.ts`, reusing the guard's
    identity logic).
  - Loads the active approved plan from the role repo's OWN local project
    context (`loadWorkflowState('plan')` + `loadArtifactMetadata`).
  - Builds a single `RepoChangeSet` for the current repo via `runGitDiff(cwd)`,
    narrows the plan to `{ targetRepos:[thisRepo], allowedWriteScope:[entry] }`,
    and runs the shared `validateWriteScope` primitive.
  - FAILs the completion (exit 1, state stays `in-progress`) on any write-scope
    violation, unless `--no-review` / `--force` is passed.
  - Degrades to an explicit SKIP (completion proceeds) when the repo is not
    identifiable, there is no approved plan available locally, or the plan has
    no `allowedWriteScope` entry for this repo — never crashes.
- **B — Aggregate validation (`axiom validate changes --all-repos`).** New
  `runValidateChangesAggregate` + `formatAggregateReport` in
  `validate-changes.ts`. Reads the plan, resolves every `targetRepo` to an
  absolute path via `LocalBindings`, diffs each, runs `validateWriteScope` once
  over all resolved repos, and emits a consolidated report (per-repo pass/fail +
  explicit UNRESOLVED list + overall).
- **P1 gap close.** `buildRepoChangeSets` (in `packages/doctor/src/write-scope.ts`)
  now resolves repo paths via `resolveRepoPath(ref, bindings, projectRoot)`
  (bindings-preferred) instead of the projectRoot-relative-only heuristic. A new
  `buildTargetRepoChangeSets` resolves an arbitrary target-repo list and reports
  unresolvable repos explicitly.

## Non-goals

- **Cross-repo workflow-state sharing.** The per-role path does NOT reach into
  other repos to fetch the plan; it only uses what is available locally. In a
  true multi-repo workspace where the plan state/metadata live in the spec repo
  and are NOT mirrored into the role repo, the per-role review degrades to SKIP
  (documented, non-blocking). The aggregate-from-spec command is the surface for
  full multi-repo validation. (Open question Q-001.)
- No change to the archive step, the workflow state machines, the artifact
  metadata shape, or the `checkPlanIsApproved` / `checkRepoAffinity` gates.
- No new persistent tracking state; diffs are computed against `HEAD` +
  untracked, exactly as the pre-existing `validate changes` / `WS-001` do.

## Review model (per-role-at-complete vs aggregate-from-spec vs archive)

| Surface | Runs where | Runs when | Repos validated | Gating |
|---------|-----------|-----------|-----------------|--------|
| **Per-role review** (this increment, A) | role X's repo | `axiom-role complete` | current repo only (its own git) | FAILs completion (exit 1) on violation unless `--no-review`/`--force`; SKIPs gracefully when no local plan/scope |
| **Aggregate validation** (this increment, B) | SPEC repo | `axiom validate changes --plan <id> --all-repos`, on demand | every `targetRepo`, resolved via `LocalBindings` | read-only report; exit 1 if any violation OR any unresolved repo |
| **Single-repo `validate changes`** (pre-existing, INC-08) | any repo | on demand | topology repos relative to projectRoot | read-only report; exit 1 on violation |
| **`WS-001` doctor / archive-time** (pre-existing) | SPEC/control | `axiom doctor` / around archive | topology repos relative to projectRoot | advisory check |

All four share the SAME comparison primitive (`validateWriteScope`,
`@axiom/workflow`) and the SAME repo-change-set/known-generated-path helpers
(`@axiom/doctor`) — there is no duplicated diff-vs-scope logic. The per-role
review is an early LOCAL gate; the aggregate is the broad multi-repo sweep; the
archive/doctor checks remain as the pre-existing safety net.

## Acceptance criteria

- **AC1 (per-role in-scope):** `axiom-role complete` in a role repo whose diff is
  inside its `allowedWriteScope` → review passes, role transitions to `archived`,
  exit 0.
- **AC2 (per-role out-of-scope):** diff OUTSIDE scope → review FAILS, completion
  blocked (state stays `in-progress`), exit 1, message lists the violation(s).
- **AC3 (per-role bypass):** same out-of-scope diff with `--no-review` (or
  `--force`) → completes with exit 0 and a warning.
- **AC4 (per-role graceful skip):** no approved plan / no `allowedWriteScope` for
  this repo / unidentifiable repo → review SKIPs with an explicit note, role
  completes (exit 0). Existing single-repo/dogfood `complete` behavior unchanged.
- **AC5 (aggregate multi-repo):** two role repos changed, one violating its
  scope → consolidated report shows per-repo results (✓/✗) + overall FAIL,
  exit 1.
- **AC6 (aggregate unresolved):** a target repo with no binding and a missing
  path → reported as UNRESOLVED (not silently skipped), overall exit 1.
- **AC7 (aggregate happy/degrade):** single-repo or all-in-scope → overall OK,
  exit 0; missing plan → exit 1 with a clear message.
- **AC8 (no regression):** `buildRepoChangeSets` bindings upgrade keeps every
  pre-existing write-scope test green.

## Result

Implemented and green. See the "Result" note below and the code map.

**Files (grouped):**
- Spec: this folder (`metadata.yml`, `README.md`).
- Doctor (shared primitives): `packages/doctor/src/write-scope.ts` (bindings-preferred
  `buildRepoChangeSets`; new `buildTargetRepoChangeSets` + `UnresolvedRepo`/
  `AggregateRepoChanges`), `packages/doctor/src/index.ts` (exports).
- Per-role review (A): `apps/cli/src/commands/_repo-affinity.ts` (new
  `identifyCurrentRepoIdentity`/`CurrentRepoIdentity`),
  `apps/cli/src/commands/_role-review.ts` (new `runRoleReview`),
  `apps/cli/src/commands/axiom-role.ts` (gate wiring + `--no-review`/`--force`).
- Aggregate (B): `apps/cli/src/commands/validate-changes.ts`
  (`runValidateChangesAggregate` + `formatAggregateReport` + `--all-repos`).
- Tests: `apps/cli/tests/_role-review.test.ts`,
  `apps/cli/tests/axiom-role.test.ts` (complete-review scenarios),
  `apps/cli/tests/validate-changes.test.ts` (aggregate scenarios),
  `packages/doctor/tests/write-scope.test.ts` (bindings + target-repo resolution).

**Review-result format (per-role):**
```
[axiom role review] OK. Repo 'api-repo' (plan 'PLAN-...'): 3 archivo(s) revisado(s), sin violaciones de write-scope.
```
or on failure (blocks completion):
```
[axiom role review] FAIL. Repo 'api-repo' (plan 'PLAN-...'): 1 violación(es) de write-scope:
  - [outside-scope] 'docs/x.md' en repo 'api-repo' está fuera del scope declarado (paths: [src/**]).
[axiom role] Completion BLOQUEADA por el review de write-scope. Corregí los cambios fuera de scope o re-ejecutá con `--no-review` (o `--force`) para omitir el review.
```

**Aggregate report format:**
```
[axiom validate changes] AGGREGATE. Plan 'PLAN-...' — validación multi-repo: 3 repo(s) objetivo (2 resuelto(s), 1 no resuelto(s)).
  ✓ api-repo: sin violaciones (2 archivo(s) revisado(s)).
  ✗ web-repo: 1 violación(es):
      - [outside-scope] 'e2e/x.spec.ts' en repo 'web-repo' está fuera del scope declarado (paths: [src/**]).
  ⚠ mobile-repo: NO RESUELTO — El repo objetivo 'mobile-repo' no se pudo resolver a una ruta existente ...
  RESULTADO GLOBAL: FAIL — 1 violación(es), 1 repo(s) no resuelto(s).
```

## Pending / follow-ups

- **Q-001 (cross-repo plan state):** for per-role review to run (not SKIP) in a
  true multi-repo workspace, the plan's approved workflow-state + metadata must
  be readable from the role repo. Today they live in the spec repo. A future
  increment can either mirror the plan artifact into each role repo at
  `axiom-role start`, or teach the per-role path to resolve the spec repo via
  `LocalBindings` (explicitly opting into one cross-repo read). Until then the
  aggregate-from-spec command is the multi-repo validation surface.
