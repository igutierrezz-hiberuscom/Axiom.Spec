# Increment: Reconcile contextual TUI shell and detection — independent validation (validator-reviewer)

Status: closed
Date: 2026-07-02

## Goal

Independently re-verify the tui-developer implementation of the
addendum-14 detection heuristic (`INC-20260702-tui-shell-detection-reconcile-impl`)
against its own acceptance criteria and against the audit brief
(`INC-20260702-tui-shell-detection-reconcile`), without trusting either
prior spec's prose claims at face value. This is the third and final
role of INC-04 of
`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`.

## Context

Two prior roles completed:

1. **migration-engineer audit**
   (`Axiom.Spec/specs/increments/INC-20260702-tui-shell-detection-reconcile/README.md`):
   confirmed addendum-14 checks 1/2 already worked via `discoverAxiomRoot`,
   checks 3/4 were unimplemented, no shell rewrite needed, produced a
   5-item implementation brief and OQ1/OQ2/OQ3.
2. **tui-developer implementation**
   (`Axiom.Spec/specs/increments/INC-20260702-tui-shell-detection-reconcile-impl/README.md`):
   implemented `findGitRoot`/`findGitRootAxiomConfig`
   (`@axiom/filesystem-truth`), `findByAncestorRepoPathV2`
   (`@axiom/user-workspace`), `resolveProjectWithFallback`
   (`@axiom/project-resolution`), and the `tui.ts` fallback control-flow
   branch + `initialMessage` driver plumbing, with OQ1 deferred, OQ2
   resolved (new function in `@axiom/user-workspace`), OQ3 resolved
   (build the fallback UX now).

This increment does not re-implement anything; it independently re-derives
every material claim from the live code and from a fresh test run,
including bisecting the baseline via `git stash` again (not trusting the
prior run's numbers), and makes an explicit judgment call on whether
`general-spec.md` should now be created.

## Scope

- Re-read `resolveProjectWithFallback` in
  `Axiom/packages/project-resolution/src/resolver.ts` and independently
  confirm check order and short-circuit correctness.
- Re-read `findGitRoot`/`findGitRootAxiomConfig` in
  `Axiom/packages/filesystem-truth/src/discovery.ts` and independently
  assess the unbounded-walk design decision for safety
  (termination, pathological performance, consistency with
  `git rev-parse --show-toplevel` semantics).
- Re-read `findByAncestorRepoPathV2` in
  `Axiom/packages/user-workspace/src/registry.ts` and independently
  verify the ancestor-vs-false-positive path comparison logic.
- Re-read the new `Axiom/apps/cli/src/commands/tui.ts` control flow and
  independently confirm the true-failure path is unchanged and that
  `ambiguous`/`invalid-config` do not trigger the new fallback.
- Confirm `mcp-inventory`/`memory-inventory` and the listed untouched
  packages/files genuinely have no diff (via `git diff --stat`, not
  prose trust).
- Run `npm run typecheck`, `npm run build`, `npm test` for real, and
  independently re-derive the baseline via `git stash`/`git stash pop`
  rather than accepting the tui-developer's quoted baseline number as-is.
- Make an explicit, reasoned recommendation on whether
  `Axiom.Spec/general-spec.md` should be created now.
- Assess INC-04-as-a-whole for closure and close it if warranted.

## Non-goals

- No new code changes (this is a pure verification pass; see one
  clarification below under Implementation notes — no source edits were
  required because no bug was found).
- No re-litigation of OQ1/OQ2/OQ3 (already resolved by the user before
  the tui-developer pass).
- No implementation of `mcp-inventory`/`memory-inventory` promotion
  (tracked separately, see Result below).
- No work on INC-05 itself (only a handoff flag for whoever picks it up).

## Acceptance criteria

- [x] Check order (cwd → parent-walk → registered-ancestor-path →
      git-root) independently confirmed against addendum-14's stated
      order, with short-circuit correctness verified line-by-line.
- [x] `findGitRoot`'s unbounded-walk design independently assessed for
      safety (termination, performance, consistency with
      `git rev-parse --show-toplevel`); confirmed safe, no narrowing fix
      needed.
- [x] `findByAncestorRepoPathV2`'s path-comparison logic independently
      verified to reject the `/repos/foo` vs `/repos/foobar` false-positive
      class.
- [x] `tui.ts`'s true-failure path and `ambiguous`/`invalid-config`
      non-triggering independently confirmed unchanged, by reading the
      control flow directly (not by re-running only the new tests).
- [x] `mcp-inventory`/`memory-inventory` and the listed untouched
      packages/files independently confirmed via `git diff --stat`
      (zero diff).
- [x] Full monorepo validation (`typecheck`, `build`, `test`) executed
      independently; baseline re-derived via a fresh `git stash`
      bisection, not assumed from the prior spec's numbers.
- [x] Explicit `general-spec.md` recommendation made with reasoning
      (not defaulted).
- [x] INC-04-as-a-whole closure assessed against its own roadmap entry
      and closed with an explicit closure summary, naming deferred items.
- [x] Result documented; next step recommendation given, including an
      explicit INC-05/INC-03 overlap flag.

## Open questions

None outstanding. All of INC-04's own open questions (OQ1/OQ2/OQ3, from
the audit increment) were resolved before this pass started.

## Assumptions

- The migration-engineer and tui-developer specs' file lists
  (Scope/Implementation notes sections) are treated as claims to verify,
  not as ground truth — every file-existence and behavior claim in this
  spec was independently re-derived from the live repository state at
  the time of this review, not copied from the prior specs.
- "Independently re-derived" for the test baseline means a fresh
  `git stash` / `git stash pop` cycle was run in this session, producing
  its own JSON test report, rather than trusting the tui-developer
  spec's quoted `1231 passed` / `1254 passed` numbers as given.

## Implementation notes

No source code changes were made in this increment. All four independent
reviews below confirmed the tui-developer implementation as correct with
no bugs found, so no corrective fix was required.

### 1. Detection chain order and short-circuit correctness

Read `resolveProjectWithFallback` in
`Axiom/packages/project-resolution/src/resolver.ts` line-by-line
(not summarized from the spec). Confirmed:

- Line 176: `const direct = resolveProject(startPath)` — this call alone
  covers addendum-14 checks 1 and 2 (`discoverAxiomRoot`'s own logic
  checks cwd at depth 0, then walks parents up to
  `MAX_TRAVERSAL_DEPTH`).
- Line 177-179: `if (direct.status !== 'not-found') return { resolution:
  direct, detectionMethod: 'direct' }` — this is the correct short-circuit
  for checks 1/2 succeeding, AND (critically) it also correctly
  short-circuits the `ambiguous`/`invalid-config` cases, which is the
  exact behavior the audit's own reasoning required (an `axiom.yaml` was
  already found, just invalid — searching elsewhere would not fix that).
- Lines 187-206: check 3 (`findByAncestorRepoPathV2`) only runs after the
  line-177 short-circuit did NOT return, i.e. only on genuine
  `not-found`. Inside, it only returns a `registered-ancestor-path`
  result if `ancestorResult.ok && ancestorResult.value !== null` AND a
  matching repo entry is found AND the re-resolution of that repo's path
  via `resolveProject(matchedRepo.path)` is itself not `not-found`
  (line 199) — three genuine guard conditions, not a single optimistic
  return.
- Lines 208-215: check 4 (`findGitRootAxiomConfig`) is reached only if
  check 3's block did not already return, i.e. only if check 3 genuinely
  found nothing usable. Same re-resolve-and-verify pattern as check 3.
- Line 218: if all of the above fall through, the original `direct`
  result (the true `not-found`) is returned unchanged.

**Verdict**: order matches addendum-14 exactly (cwd → parent → registered
path → git-root), and each check is only attempted after the prior ones
have genuinely failed — no incorrect short-circuiting found. Confirmed
correct, no fix needed.

### 2. `findGitRoot` unbounded-walk safety assessment

Read `findGitRoot` in `Axiom/packages/filesystem-truth/src/discovery.ts`
(lines 84-99) directly. The loop:

```
for (;;) {
  const gitPath = path.join(current, '.git');
  if (fs.existsSync(gitPath)) return current;
  const parent = path.dirname(current);
  if (parent === current) return null; // filesystem root
  current = parent;
}
```

Independent safety assessment (not accepting the tui-developer's
Assumptions section as sufficient on its own):

- **Termination**: `path.dirname(current) === current` is the standard,
  well-defined sentinel for "filesystem root reached" on both POSIX
  (`path.dirname('/') === '/'`) and Windows (`path.dirname('C:\\') ===
  'C:\\'`). This is the exact same sentinel already used by
  `discoverAxiomRoot` one function above it in the same file, so it is
  not a novel or untested termination condition — it is a proven pattern
  in this codebase. No infinite-loop risk.
- **Pathological performance**: the loop cost is bounded by the number of
  directory levels between `startPath` and the filesystem root, which is
  a `fs.existsSync` call per level — the same cost shape as
  `discoverAxiomRoot`'s existing bounded walk, just without the
  early-exit at 10 levels. Even for unusually deep trees (e.g. 30-40
  levels, well beyond any realistic project layout), this is a handful
  of synchronous stat calls, not a performance concern in practice. No
  pathological blowup exists because each level does O(1) work and the
  filesystem depth is bounded by the OS itself (Windows historically
  ~260 chars per path segment budget, POSIX typically much deeper but
  still finite).
- **Consistency with `git rev-parse --show-toplevel`**: real Git has no
  artificial depth cap on this walk either — it stops at the first
  `.git` found walking upward, or at `GIT_CEILING_DIRECTORIES`/filesystem
  root if none is found. This implementation's behavior (walk to
  filesystem root, stop at first `.git` file-or-directory) is the
  correct unshelled equivalent, and the file-or-directory check
  (`fs.existsSync(gitPath)`, not `fs.statSync(gitPath).isDirectory()`)
  correctly handles worktrees/submodules where `.git` is a `gitdir:`
  pointer file rather than a real directory — this was verified by
  reading the exact `fs.existsSync` call, which does not discriminate
  file vs directory, so both cases are correctly matched.
- **Contrast with `discoverAxiomRoot`'s bound**: `discoverAxiomRoot`'s
  10-level cap exists specifically because `axiom.yaml` is an
  Axiom-specific convention with a bounded expected nesting depth
  relative to a project's own root — it is not a general filesystem-walk
  safety limit, it is a scope-appropriateness heuristic for that one
  file's convention. `findGitRoot` has no equivalent reason to bound
  itself, since a real Git working tree's depth is unrelated to where
  `axiom.yaml` happens to live. The tui-developer's Assumptions section
  correctly identified this distinction; this review independently
  reaches the same conclusion by inspecting the actual code and
  reasoning about it from first principles, not by trusting that
  explanation.

**Verdict**: the unbounded-walk design is safe and consistent with real
`git`'s own behavior. No narrowing fix (e.g. an upper bound) is
warranted — adding one would reintroduce exactly the problem the
tui-developer's tests caught (a depth-capped `findGitRoot` made check 4
practically unreachable in any scenario distinct from checks 1/2).
Confirmed correct as designed, no fix needed.

### 3. `findByAncestorRepoPathV2` correctness

Read `findByAncestorRepoPathV2` in
`Axiom/packages/user-workspace/src/registry.ts` (lines 894-915) directly.
The match predicate per registered repo:

```
const registered = path.resolve(repo.path);
if (registered === target) return true;
const registeredWithSep = registered.endsWith(path.sep)
  ? registered
  : registered + path.sep;
return target.startsWith(registeredWithSep);
```

Independent trace against the stated false-positive case
(`/home/user/my-project-2` must not match a registered
`/home/user/my-project`):

- `registered = '/home/user/my-project'` (POSIX example; same logic
  applies with `\` on Windows via `path.sep`).
- `registeredWithSep = '/home/user/my-project/'` (separator appended
  because it did not already end with one).
- `target = '/home/user/my-project-2'`.
- `target.startsWith('/home/user/my-project/')` → **false**, because the
  character immediately after the shared prefix in `target` is `-2`, not
  `/`. The exact-equality check (`registered === target`) is also false.
  Correctly rejected.
- Contrast with the true-positive case
  (`/home/user/my-project/sub/dir`): `target.startsWith('/home/user/
  my-project/')` → **true**. Correctly accepted.
- Contrast with the exact-match case (`target === registered`, no
  descendant): handled by the first branch (`registered === target`)
  before the separator logic is even reached, so a bare exact match
  does not depend on `endsWith`/`startsWith` string quirks at all.

This is prefix-of-path-segments matching, not naive substring matching —
confirmed by direct trace, not by trusting the spec's own claim that it
"is not a naive substring match." The `resolveProjectWithFallback`
caller in `project-resolution/src/resolver.ts`
(`normalizeForAncestorCompare`) independently re-implements the exact
same algorithm to identify *which* registered repo matched (since
`findByAncestorRepoPathV2` returns the whole project entry, not the
specific matching repo) — this review traced both implementations and
confirmed they use the same normalization (`path.resolve` +
conditional trailing-separator append) and the same comparison logic
(`registered === target || target.startsWith(registeredWithSep)`), so
there is no drift between the registry-side match and the
resolver-side re-derivation of which repo matched.

**Verdict**: confirmed correct, no bug found, no fix needed.

### 4. `tui.ts` fallback control-flow correctness

Read `Axiom/apps/cli/src/commands/tui.ts` lines 264-297 directly (the
default-mode branch, the only branch this increment touches per its own
non-goals). Confirmed:

- The fallback is only attempted inside `if (resolution.status ===
  'not-found')` (line 275) — a status of `ambiguous` or `invalid-config`
  never enters this block at all, so those statuses fall straight
  through to the exact same error-construction code (lines 291-296) that
  existed before this increment, unconditionally. This was confirmed by
  reading the code structure directly: the `if` at line 275 wraps only
  a `return` statement for the success case; there is no alternate code
  path for `ambiguous`/`invalid-config` that could accidentally swallow
  them into the new fallback logic.
- The error object construction (`new Error(...)`, `err.projectNotFound
  = true`, `throw err`) at lines 291-296 is textually identical to what
  the pre-existing (pre-INC-04) code did for the true-not-found case —
  confirmed by comparing this file's current content against the
  migration-engineer audit's own quoted line range for the original
  behavior ("apps/cli/src/commands/tui.ts lines 212-219" in the audit
  spec) and confirming the message format, the `projectNotFound` flag
  name, and the throw mechanism are unchanged in substance (only line
  numbers shifted due to the new code inserted above).
- Also verified via the new `apps/cli/tests/tui.test.ts`, specifically
  the `ambiguous` scenario (`no reintenta el fallback si axiom.yaml
  existe pero es inválido`), which writes an `axiom.yaml` missing
  `project.name` (producing `status: 'ambiguous'` per
  `resolveFromV1`) and asserts `projectNotFound: true` is still thrown.
  This is a genuine, non-trivial test — it does not merely check "no
  crash," it specifically proves the fallback was not attempted for a
  non-`not-found` status. Read the test body directly to confirm the
  assertion is meaningful, not a rubber-stamp.

**Verdict**: the true-failure path is confirmed unchanged, and
`ambiguous`/`invalid-config` are confirmed not swallowed into the new
fallback path. No bug found, no fix needed.

### 5. Untouched-package confirmation

Ran `git status --porcelain` and `git diff --stat` directly against the
live `Axiom` working tree (not read from either prior spec's prose).
Result — full changed-file list:

```
 M apps/cli/src/commands/tui.ts
 M packages/filesystem-truth/src/discovery.ts
 M packages/filesystem-truth/src/index.ts
 M packages/filesystem-truth/tests/discovery.test.ts
 M packages/project-resolution/package.json
 M packages/project-resolution/src/index.ts
 M packages/project-resolution/src/resolver.ts
 M packages/project-resolution/tests/resolver.test.ts
 M packages/project-resolution/tsconfig.json
 M packages/tui/src/driver.ts
 M packages/user-workspace/src/index.ts
 M packages/user-workspace/src/registry.ts
 M packages/user-workspace/tests/registry-v2.test.ts
?? apps/cli/tests/tui.test.ts
```

- `git diff -- apps/cli/src/commands/tui.ts packages/tui/src/driver.ts |
  grep -i "mcp-inventory\|memory-inventory"` returned no matches (exit
  code 1) — confirmed `mcp-inventory`/`memory-inventory` are genuinely
  untouched (OQ1 deferral respected), not just absent from the prior
  specs' prose.
- `git diff --stat -- packages/topology packages/config-validation
  packages/doctor packages/orchestrator apps/cli/src/commands/init.ts
  apps/cli/src/commands/join.ts apps/cli/src/commands/repo.ts
  apps/cli/src/commands/projects.ts apps/cli/src/commands/app-api.ts`
  produced zero output — confirmed none of `@axiom/topology`,
  `@axiom/config-validation`, `@axiom/doctor`, `@axiom/orchestrator`, or
  the 5 named INC-01-cutover CLI files were modified.

**Verdict**: confirmed exactly as claimed, independently.

### 6. Independent validation run

Validation command discovery (per `Axiom.SDD/AGENTS.md`): same as the
tui-developer pass — `Axiom/package.json` defines `npm run typecheck`
(`tsc -b`), `npm run build` (`tsc -b`), and `npm test` (`vitest run`).
All three were run for real in this session.

- `npm run typecheck`: clean, zero errors.
- `npm run build`: clean, zero errors.
- `npm test` (full monorepo, JSON reporter for exact comparison):
  **12 failed test files / 13 failed tests / 1254 passed** (of 1267
  total).

**Independent baseline re-derivation** (fresh `git stash push -u` /
`git stash pop` in this session — not reusing the tui-developer's
quoted numbers): with all INC-04 changes stashed, `npm test` (JSON
reporter) produced **12 failed test files / 13 failed tests / 1231
passed** (of 1244 total). The failed-file list and failed-test-name list
were diffed programmatically between the baseline run and the
post-implementation run: **identical set of 12 files and 13 fully
qualified test names in both runs**, including the one pre-existing
`project-resolution` failure
(`resolveProject resuelve scopes con absolutePath y exists correcto
(v1)`) and the one pre-existing `tui/driver` failures. Zero new
failures, zero newly-fixed failures, net +23 passing tests
(1254 - 1231 = 23), matching the tui-developer spec's claim exactly.

Additionally ran a targeted regression check on the specific screens the
tui-developer spec flagged as needing direct confirmation:
`npx vitest run packages/tui/tests/router.test.ts
packages/tui/tests/projects.test.ts packages/tui/tests/topology.test.ts`
→ 3 files, 32 tests, all passed. (No dedicated `model-list.test.ts` file
exists; `model-list` behavior is covered within `driver.test.ts`, whose
one pre-existing failure is unrelated to `model-list` — it is the
`upgrade`/`configure` preview-flow assertions listed above, confirmed
present in both baseline and post-implementation runs.)

**Verdict**: all validation numbers independently confirmed, not merely
re-quoted.

## Validation

`npm run typecheck`, `npm run build`, and `npm test` were run directly in
this session (see Section 6 above for full detail), plus an independent
`git stash`-based baseline re-derivation and a targeted
router/projects/topology regression re-run. This satisfies
`Axiom.SDD/AGENTS.md`'s validation-discovery order (package scripts
found and used); no fallback statement is needed.

## Result

Independently re-verified all four implementation claims from
`INC-20260702-tui-shell-detection-reconcile-impl` by reading the live
code directly (not trusting prior spec prose) and found **no bugs**:

1. `resolveProjectWithFallback`'s check order and short-circuiting are
   correct.
2. `findGitRoot`'s unbounded walk is safe (proper termination, no
   pathological performance, correct match to `git`'s own
   `--show-toplevel` semantics) and is the right design — no narrowing
   fix was warranted.
3. `findByAncestorRepoPathV2`'s path-prefix comparison correctly
   distinguishes ancestor/descendant relationships from sibling-prefix
   false positives (traced directly against the
   `/repos/foo` vs `/repos/foobar` case).
4. `tui.ts`'s true-failure path is unchanged and `ambiguous`/
   `invalid-config` are not swallowed into the new fallback.

Independently confirmed `mcp-inventory`/`memory-inventory` and all
named out-of-scope packages/files are genuinely untouched (via
`git diff --stat`, not prose).

Independently re-ran the full validation suite and re-derived the
baseline via a fresh `git stash` bisection in this session: baseline
`1231 passed / 13 failed / 12 failed files`, post-implementation
`1254 passed / 13 failed / 12 failed files` — identical failing-test
identity set in both runs, net +23 passing tests, zero regressions.

## General spec integration

**Explicit recommendation: continue deferring `Axiom.Spec/general-spec.md`
creation.** Reasoning (not a default):

- `general-spec.md` still does not exist in `Axiom.Spec` (confirmed
  directly — `Axiom.Spec/specs/` contains only the `00_...` through
  `08_...` numbered documents plus `increments/`/`bugs/`/`archive/`, no
  `general-spec.md`), consistent with every prior increment in this
  workspace (roadmap, INC-01 chain, INC-02 chain, INC-03 chain all noted
  the same absence).
- The parent roadmap
  (`INC-20260702-axiom-redesign-roadmap/README.md`) defines 24
  increments across 7 phases; only INC-01 through INC-04 (Phase A fully,
  Phase B partially — INC-05 not started) are closed as of this
  increment. That is roughly 17% of the roadmap's own increment count,
  and none of Phases C/D/E/F/G have started at all.
  Phase E (MCP) and Phase B's own Q4 (capability/provider model
  reconciliation) are explicitly gated on open architectural questions
  (Q1, Q2, Q4, Q5) that the roadmap itself states are genuine
  architecture decisions, not implementation details. I independently
  checked whether Q1/Q2/Q5 were formally closed anywhere in the INC-01
  chain's own spec files (`grep` across all
  `INC-20260702-registry-manifest-schema-v2*` files) and found **no
  explicit resolution recorded** for those question IDs outside the
  roadmap document itself — INC-01 shipped a working `schemaVersion: 2`
  in practice, but the roadmap's own named open questions were not
  closed as formal decisions in the implementation chain's specs. This
  means the single-repo-vs-multi-repo architectural tension the roadmap
  flagged as "must resolve before Phase E/G can be scoped precisely" is
  still open in the record, even though downstream work has proceeded
  pragmatically.
- Creating `general-spec.md` now, after only Phase A + a slice of Phase
  B, risks one of two failure modes: (a) a thin document that mostly
  restates the roadmap's own already-recorded state (low marginal
  value, duplicates content that already lives in the roadmap and
  INC-01/02/03/04 closure summaries), or (b) prematurely locking in
  answers to Q1/Q2/Q5 by omission (e.g. writing "the registry is
  `~/.axiom/projects.yml`, YAML, `repos: {role: path}` map" as settled
  fact in a "stable spec" document before the roadmap's own blocking
  questions are formally closed) — exactly the same risk the roadmap's
  own "General spec integration" section already flagged for
  `03_Modelo_Operativo_y_Datos.md`'s "sin cerrar el naming final"
  placeholder.
- This is the same judgment INC-01's, INC-02's, and INC-03's own
  validator-reviewer passes reached, each with matching reasoning (file
  doesn't exist yet, no natural closure point reached). This increment
  does not default to that precedent without re-examining it — the
  re-examination (checking Phase completion percentage and the Q1/Q2/Q5
  closure status directly) reaches the same conclusion independently.

**When to revisit**: the most natural next closure point is either (a)
the end of Phase B (INC-05 closes) — a reasonable "installation +
detection are both settled" checkpoint — or (b) whenever Q1/Q2/Q5 are
explicitly, formally resolved (not just pragmatically worked around),
since those answers determine the shape of the very first
`general-spec.md` sections that would need to exist (registry format,
repo topology model, manifest schema). Recommend the next
validator-reviewer or migration-engineer pass that closes INC-05
explicitly re-ask this same question rather than assuming either answer.

## Next step recommendation

**INC-04 is closed as a whole.** See closure summary below.

Recommend starting **INC-05 — AGENTS.md as canonical contract + adapter
reconciliation** (`@axiom/document-bootstrap` + `packages/adapters/*`)
next, per the parent roadmap's Phase B sequencing (INC-05 depends on
INC-03, already closed).

**Explicit overlap flag for whoever picks up INC-05**: the parent
roadmap's original INC-05 scope (written before INC-03 landed) describes
the diff as "today the canonical source for generated files is
`axiom.yml` + `install-profile.json` + per-adapter generator, not
`AGENTS.md` as a single upstream contract... `document-bootstrap` is
scoped to one target (copilot instructions)... it is not yet a general
'generate all adapters from one canonical file' pipeline." This is now
**partially stale**: INC-03's own implementation chain
(`INC-20260702-installer-wizard-reconcile-agents-md/README.md`) already
built a canonical `AGENTS.md` generator
(`writeCanonicalAgentsMd` in `@axiom/document-bootstrap`, per that
increment's own closure notes referenced in
`INC-20260702-installer-wizard-reconcile-validator/README.md`) as part
of `axiom init`'s wizard reconciliation work. INC-05's migration-engineer
step should start by auditing `writeCanonicalAgentsMd`'s current
contract and scope directly, rather than re-deriving "does a canonical
AGENTS.md generator exist" from scratch — the answer is very likely
"yes, partially, already," and INC-05's real remaining diff is probably
narrower than the roadmap's original framing: reconciling the 6 adapter
generators (`packages/adapters/*`) to actually *consume*
`writeCanonicalAgentsMd`'s output as their upstream source (or confirm
they already do, or decide a shim), not building the generator itself.
This flag is raised here specifically so it is not rediscovered from
scratch by INC-05's migration-engineer.

## INC-04 closure summary

**INC-04 — Reconcile contextual TUI shell and detection (`@axiom/tui`)**
is closed as a whole, across its 3 constituent role-specs:

1. `INC-20260702-tui-shell-detection-reconcile` (migration-engineer
   audit) — confirmed the roadmap's INC-04 hypothesis, sharpened it with
   two material refinements (menu diff granularity, addendum-14 check-3
   feasibility gated on INC-01), produced OQ1/OQ2/OQ3 and a 5-item
   implementation brief.
2. `INC-20260702-tui-shell-detection-reconcile-impl` (tui-developer
   implementation) — implemented all 4 addendum-14 checks, composed
   them in `resolveProjectWithFallback`, wired the fallback UX into
   `tui.ts`, added 20 new tests across 4 packages, ran full validation
   for real with a `git stash`-derived baseline.
3. `INC-20260702-tui-shell-detection-reconcile-validator` (this
   increment) — independently re-verified every material claim from
   both prior roles by reading the live code directly (not the prior
   specs' prose), found **zero bugs**, independently re-derived the
   test baseline via a fresh `git stash` cycle and confirmed identical
   results, and made an explicit (not defaulted) `general-spec.md`
   deferral recommendation.

**All of INC-04's roadmap-level acceptance criteria are met**:
addendum-14's 4-check heuristic is fully implemented and independently
verified correct; the "no shell rewrite needed" judgment held through
implementation (confirmed by the additive nature of every diff: new
functions, one new optional driver field reusing an existing router
field, one new control-flow branch guarded by an existing status
check); full monorepo validation was run and independently
re-confirmed with zero regressions; the roadmap's own dependency
ordering (INC-01, INC-02, INC-03 all closed before INC-04 started) was
respected.

**Deferred items, named explicitly** (not silently dropped):

- **`mcp-inventory`/`memory-inventory` menu promotion (OQ1)**: tracked
  at
  `Axiom.Spec/specs/increments/INC-20260702-tui-menu-promote-inventory-screens/README.md`,
  status `pending (not started — deferred placeholder)`. Confirmed in
  this pass that both screens remain genuinely untouched by INC-04's
  implementation.
- **`projects` screen v1→v2 registry migration**: flagged by the
  original migration-engineer audit (Section 1 of that spec) as a
  functional gap (`loadProjectsData`/`onProjectSelect` in `driver.ts`
  still call `listProjects`/`useProject`, the v1 registry functions, not
  `listProjectsV2`/`useProjectV2`). Explicitly out of scope for
  `INC-20260702-tui-shell-detection-reconcile-impl` per its own
  Non-goals section. Not yet tracked as its own increment file — flagged
  here for whoever next touches the `projects` screen or the
  `user-workspace` v1/v2 registry boundary to open as a small, scoped
  increment rather than rediscovering it.
- **Menu additions for increment/bug/plan creation and "guided
  implementation"**: flagged by the audit's implementation brief item 3
  as out of scope for the detection-focused tui-developer pass. Not yet
  tracked as its own increment file — depends partly on
  `@axiom/orchestrator`'s still-stubbed intent commands (INC-02's own
  scope), so it is reasonable to defer opening that increment file until
  INC-02's stubs are filled in further, rather than opening a spec file
  that would immediately block on unstarted work.
- **`general-spec.md` creation**: explicitly deferred (see reasoning
  above), not silently skipped.

Status: **closed**.
