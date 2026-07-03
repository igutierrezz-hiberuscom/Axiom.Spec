# Increment: schemaVersion:2 emission cutover (D3 implementation)

Status: closed
Date: 2026-07-03

Closed by the validator-reviewer pass:
`Axiom.Spec/specs/increments/INC-20260702-schemaversion2-multirepo-primary-reconcile-validator/README.md`.
That increment independently re-verified every claim in this file (fresh
consumer-list re-audit, widened live CLI walkthrough with a second
adapter target and `repo attach`, independent version-dispatch source
trace, independent `axiom doctor` re-run, independent `git stash`
bisection) and found no blocking issue. See that file for the
independent evidence; this file's own closure rationale below is
superseded by that pass's result.

## Goal

Implement D3 from INC-01's original resolved decisions: re-enable
`axiom.yaml schemaVersion: 2` emission in `axiom init`, now that the
read side (`resolveProject`, `@axiom/doctor`'s MC-001/BC-001/BC-002) is
genuinely version-aware, and fix the two previously-unflagged consumer
gaps the audit increment
(`INC-20260702-schemaversion2-multirepo-primary-reconcile`) found and
required as part of the same cutover.

## Context

Parent chain: `INC-20260702-axiom-redesign-roadmap` ->
`INC-20260702-registry-manifest-schema-v2` (migration-engineer audit,
records D1/D2/D3) -> `...-design` (schema-writer) -> `...-impl`
(registry-engineer) -> `...-cli` (cli-implementer: attempted the D3
cutover, reverted) -> `...-resolver-fix` (made `resolveProject`
version-aware, judged cutover "now safe", deliberately did not flip
it) -> `...-validator` (made `@axiom/doctor` version-aware, closed
INC-01, again left D3 as a named follow-up) -> INC-20
(`versioning-reconcile`) -> INC-21 (`upgrade-reconcile`, second
consecutive increment blocked on the same root cause) ->
`INC-20260702-schemaversion2-multirepo-primary-reconcile`
(migration-engineer audit-only: re-confirmed the read side is safe,
found two new consumer-safety gaps, recommended splitting D3 and D1
into two sequential increments, D3 first) -> **this increment**
(cli-implementer: implements D3 per that audit's exact brief).

D1 (`defaultSingleRepoManifest`'s deprecation warning) is explicitly
**out of scope** here, per the audit's own scope-split rationale (D1's
warning should fire meaningfully only in the post-D3 world, and is a
distinct `schema-writer`/topology-implementer design decision). It is
the next role's job, in a separate, sequenced increment.

## Scope

1. Flip `Axiom/apps/cli/src/commands/init.ts`'s `buildAxiomYaml` to
   emit `schemaVersion: 2` (`AxiomYamlSchemaV2` shape:
   `projectId`/`name`/`repoId`/`role`/`mode`/`paths`), replacing the
   stale historical-revert comment.
2. Fix `Axiom/apps/cli/src/commands/configure.ts`'s
   `readAxiomYamlProjectName` to add a `schemaVersion === 2` branch
   (mirrors `repo.ts`'s already-correct `readProjectNameFromYaml`).
3. Fix `Axiom/packages/topology/src/loader.ts`'s `tryLoadTopologyHint`
   to also read a `schemaVersion: 2` document's top-level `name`/`mode`.
4. Run the real, live CLI walkthrough the resolver-fix increment asked
   for: `init` -> `join` -> `repo attach` -> topology resolution ->
   `configure --target copilot-vscode` against a v2 project with no
   `product.manifest.yaml`.
5. Update/add unit tests (v1 and v2 coverage) for the three changed
   files, plus a real, no-mock, end-to-end integration test.
6. Run full-monorepo `typecheck`/`build`/`test`, cross-check against the
   documented pre-existing baseline (~12 failed files / 13 failed tests
   / ~1598 passed), confirm no new regressions.

## Non-goals

- D1 (`defaultSingleRepoManifest`'s deprecation warning, `@axiom/doctor`
  TC-001's `warn` branch) — separate, sequenced follow-up increment.
- Q3/Q4 from the parent roadmap — untouched, remain open.
- Fixing the pre-existing `packages/cli-commands` build-configuration
  issue discovered during validation (see "Additional gap found"
  below) — unrelated to D3's scope, documented for awareness only.
- Migration tooling / `axiom upgrade` support for migrating existing
  v1 `axiom.yaml` documents to v2 — out of scope for this increment and
  the whole INC-01 chain (non-hard-removal posture; v1 remains
  supported indefinitely on the read side).

## Acceptance criteria

- [x] `buildAxiomYaml` emits `schemaVersion: 2` with the full
      `AxiomYamlSchemaV2` shape for both `installed-multi-repo` and
      `self-hosted`/`one-axiom-per-product` layouts.
- [x] `configure.ts`'s `readAxiomYamlProjectName` is version-aware
      (v1 `project.name` and v2 top-level `name`).
- [x] `topology/src/loader.ts`'s `tryLoadTopologyHint` is version-aware
      (v1 `project.name`/`project.mode` and v2 top-level `name`/`mode`).
- [x] A real, live CLI walkthrough was run and documented with
      concrete command output: `init` -> `resolveProject` ->
      `join` -> `loadTopology` (with `topology.yaml` absent, exercising
      the hint fallback) -> `configure --target copilot-vscode`
      against a v2 project with no `product.manifest.yaml`.
- [x] A negative test confirmed the pre-fix `configure.ts` would have
      hard-failed (exit 1) in the exact scenario the audit flagged.
- [x] Unit tests cover both v1 and v2 inputs for all three changed
      files.
- [x] A real (no-mock), repeatable, vitest-based end-to-end integration
      test exists covering the same walkthrough.
- [x] `npm run typecheck`, `npm run build`, and `npm test` were run.
      Baseline re-confirmed via stash-based bisection (moved new/changed
      files aside, re-ran the one suspicious failing test, confirmed
      identical failure on unmodified HEAD content).
- [x] Test result comparison shows zero new regressions: same 12 failed
      files / 13 failed tests before and after, by name-for-name
      comparison.

## Open questions

None blocking. Carried forward to the next role (D1 implementer):

- Exact deprecation-warning mechanism for `defaultSingleRepoManifest`
  (stderr log line vs. a new `TC-001`-adjacent `warn`-only
  `DoctorCheck` vs. both) — deliberately left to the D1 role, per the
  audit increment's own non-goals.

## Assumptions

- Inherited unchanged from the whole INC-01 chain: non-hard-removal
  posture (v1 `axiom.yaml` remains fully supported on the read side
  indefinitely; only newly-`init`-ed projects get v2).
- `repoId` for `init`-created projects is derived as
  `${slugifyProjectId(projectName)}-${DEFAULT_REPO_ROLE}` (e.g.
  `my-project-sdd`) — not specified verbatim by the schema-writer
  design (which only fixed the schema shape, not the exact `repoId`
  derivation for `init`'s single-repo case), but consistent with
  `@axiom/topology`'s `RepoRef.id`/`RoleAssignment.roleId` vocabulary
  and with the existing `slugifyProjectId` already used by `init.ts`
  for the user-level registry entry.
- The pre-existing `packages/cli-commands` build-configuration issue
  (see below) does not affect runtime correctness of the source changes
  in this increment — confirmed by running `npm run typecheck` (passes
  clean) and the full vitest suite (which uses esbuild/vite-node
  transform, not `tsc` emit, and is unaffected).

## Implementation notes

### Files changed

- `Axiom/apps/cli/src/commands/init.ts` — `buildAxiomYaml` now emits
  `schemaVersion: 2` for both layouts. Replaced the stale
  historical-revert comment with one explaining why the cutover is now
  safe, citing the resolver-fix/validator/audit increments.
- `Axiom/apps/cli/src/commands/configure.ts` — `readAxiomYamlProjectName`
  gained a `schemaVersion === 2` branch reading the top-level `name`
  field, mirroring `repo.ts`'s `readProjectNameFromYaml`.
- `Axiom/packages/topology/src/loader.ts` — `tryLoadTopologyHint`
  gained a `schemaVersion === 2` branch reading top-level `name`/`mode`
  before falling back to the v1 `project.name`/`project.mode` path.
- `Axiom/apps/cli/tests/init.test.ts` — Scenario 1's `axiom.yaml`
  shape assertions updated from v1 (`project.name`/`project.mode`) to
  v2 (`schemaVersion`/`projectId`/`name`/`repoId`/`role`/`mode`).
- `Axiom/apps/cli/tests/document-bootstrap.test.ts` — added Scenario
  1b (`readAxiomYamlProjectName` fallback, v1 and v2 branches), each
  as a real `runInit`/`runConfigure` execution, not a unit-level mock.
  Scenario 1 (pre-existing, unchanged) now itself exercises the
  previously-broken v2 fallback path end-to-end, since `runInit`
  emits v2 by default after this change.
- `Axiom/packages/topology/tests/topology.test.ts` — added two tests
  for `tryLoadTopologyHint`'s v2 branch (`installed-multi-repo` hint
  honored; non-`installed-multi-repo` v2 mode falls through to the
  single-repo default).
- `Axiom/apps/cli/tests/schemaversion2-e2e.test.ts` (new) — real,
  no-mock, end-to-end integration test chaining `runInit` ->
  `resolveProject` -> `runJoin` -> `loadTopology` (hint fallback) ->
  `runConfigure` (copilot-vscode, no `product.manifest.yaml`).
- `Axiom.Spec/general-spec.md` — updated the "Versioning" section to
  record the blocker as resolved (see "General spec integration").

### Real CLI walkthrough (concrete output)

Ran against the real built `apps/cli/dist/index.js` binary (Node,
Windows), in fresh temp directories, per the resolver-fix increment's
own request. Note on tooling: the monorepo's `apps/cli/dist/` was
locally incomplete due to a **pre-existing, unrelated** build
misconfiguration (see "Additional gap found" below) that silently
drops 8 files (`_shared.js`, `configure.js`, `components.js`,
`model.js`, `sync.js`, `upgrade.js`, `index-cmd.js`,
`validate-changes.js`) from `apps/cli`'s own `tsc -b` output. For the
manual walkthrough only, these 8 files were copied in from
`packages/cli-commands/dist/apps/cli/src/commands/` (which does
correctly build them, from the exact same source, as a side effect of
that package's own — also pre-existing — `include` list) as a local,
throwaway workaround; this was not committed and was deleted after the
walkthrough. It does not affect the source changes, the typecheck/build
success, or the vitest suite (which never depends on this `dist/`
output at all).

1. `axiom init --yes` in a fresh temp dir → `axiom.yaml`:
   ```
   schemaVersion: 2
   projectId: my-v2-project
   name: my-v2-project
   repoId: my-v2-project-sdd
   role: sdd
   mode: installed-multi-repo
   status: specification-and-external-runtime

   paths:
     control:        { path: .,                        product_runtime: false }
     specification:  { path: ../my-v2-project.spec,    product_runtime: false }
   ```
2. `axiom join --member user:test-user` → succeeded (`[axiom join]
   Miembro agregado.`) — this alone proves `resolveProject` resolved
   the v2 document to `status: 'resolved'` (a `join` on an unresolved
   project throws before doing anything).
3. `axiom topology show` with `axiom.spec/config/topology.yaml`
   **removed** (forcing the `tryLoadTopologyHint` fallback path,
   the exact scenario the audit flagged):
   ```
   Topología del proyecto ...\my-v2-project:
     mode: multi-repo
     sdd-repo: sdd-repo @ .
     spec-repo: spec-repo @ ../my-v2-project.spec
   ```
   Confirmed correct: pre-fix, `tryLoadTopologyHint` was v1-only and
   would have returned `null`, silently mis-resolving this to
   `defaultSingleRepoManifest` (`mode: single-repo`).
4. `axiom init --yes --target copilot-vscode` in a second fresh temp
   dir, with `axiom.spec/templates/copilot-instructions.template.md`
   and `axiom.spec/config/profiles.yaml` seeded (both required
   regardless of this increment's scope — `init` does not seed the
   copilot template itself, a separate, pre-existing, unrelated
   product gap not touched here), and critically **no**
   `axiom.spec/product.manifest.yaml` — then `axiom configure`:
   ```
   [axiom configure] OK.
     generatedFiles: 3
     externalDependencies: 4
     profile: ...\.sdd\my-v2-copilot-project\install-profile.json
   ```
   `.github/copilot-instructions.md` rendered with `{{project.name}}`
   resolved to `my-v2-copilot-project` (the v2 `axiom.yaml`'s
   top-level `name` field — no `product.manifest.yaml` exists in this
   project at all, so this is unambiguously the
   `readAxiomYamlProjectName` v2 fallback branch).
5. **Negative-test confirmation**: temporarily patched the compiled
   `readAxiomYamlProjectName` to the pre-fix v1-only implementation and
   re-ran step 4's `axiom configure` against an identical fresh v2
   project. Result: hard failure, exit code 1:
   ```
   [orchestrator] FAIL: command="configure-command" duration=12ms
   [axiom configure] Error: writeCopilotInstructions falló para
   adapterTarget=copilot-vscode: writeCopilotInstructions err
   (kind=missing-required-variable): resolveVariables failed:
   resolveVariables: missing required variable project.name
   (source: product.manifest.product.name o axiom.yaml#project.name).
   ```
   This is an exact reproduction of the audit's predicted blast
   radius. Restored the real fix and re-ran: exit code 0, same
   success as step 4. This confirms both that the bug was real and
   that the fix closes it.
6. `axiom doctor` against the v2 project: `MC-001`, `BC-001`, `BC-002`
   all `✓` (pass) — confirming the doctor's version-awareness holds
   against real `axiom init`-emitted v2 output, not just crafted test
   fixtures. (2 unrelated failures and 19 skips in this run are due to
   the minimal test fixture missing scaffolding unrelated to this
   increment — `mcp-manifest.yaml`, full `axiom.spec` scope — not
   caused by this change.)

All temp directories and the throwaway `dist/` patch were deleted
after the walkthrough.

### Real end-to-end integration test (repeatable, CI-safe)

`Axiom/apps/cli/tests/schemaversion2-e2e.test.ts` — same walkthrough
as above, expressed as a real vitest test using the actual `runInit`/
`runJoin`/`runConfigure`/`resolveProject`/`loadTopology` functions
against a real fresh temp directory (no `fs`/`yaml` mocking). Passes:

```
✓ apps/cli/tests/schemaversion2-e2e.test.ts (1 test) 93ms
```

### Additional gap found (documented, not fixed — out of scope)

While attempting to run the real CLI binary for the walkthrough,
discovered that `apps/cli/dist/` is **silently incomplete** after a
clean `npm run build` (`tsc -b`, exit 0, zero errors): 8 files under
`apps/cli/src/commands/` (`_shared.ts`, `configure.ts`, `components.ts`,
`model.ts`, `sync.ts`, `upgrade.ts`, `index-cmd.ts`,
`validate-changes.ts`) are never emitted to `apps/cli/dist/commands/`,
even though they are correctly resolved into `apps/cli`'s TS program
graph (confirmed via `--traceResolution`) and have zero compile
errors. Root cause, confirmed by direct trace (`--listFilesOnly`):
`Axiom/packages/cli-commands/tsconfig.json` has an unusual `include`
list that names these exact 8 `apps/cli/src/commands/*.ts` files
directly as its own compilation roots (in addition to `apps/cli`'s own
project-reference-based compilation of the same files), so TypeScript's
composite/incremental build graph treats them as already
"owned"/emitted by the `cli-commands` project and skips re-emitting
them into `apps/cli/dist/`. This means the current `dist/index.js`
binary cannot run standalone at all (`Cannot find module './_shared'`)
regardless of this increment's changes — **confirmed pre-existing** by
reproducing the identical gap after a full clean rebuild with our
changes stashed out (moved aside via `git stash`, since the changed
files are the ones importing the same paths). Not fixed here: it is a
cross-package build-tooling misconfiguration unrelated to D3's scope,
and `packages/cli-commands` is not one of this increment's authorized
files. Recommended as a small, separate, low-risk follow-up (likely:
remove the redundant `include` entries from `cli-commands/tsconfig.json`
since `apps/cli`'s own build already compiles those files, or
restructure the re-export barrel to not need direct compilation of
`apps/cli/src/...` sources).

## Validation

Real validation was executed (not the no-command fallback statement).

- `npm run typecheck` (`tsc -b`, full monorepo): clean, zero errors.
- `npm run build` (`tsc -b`, full monorepo): clean, zero errors. (See
  "Additional gap found" for a pre-existing, unrelated `dist/`
  completeness caveat that does not manifest as a build error.)
- `npm test` (vitest, full monorepo), before and after this increment's
  changes (verified via `git stash` on the four tracked changed
  source/test files, since the fifth is a new untracked file moved
  aside manually):
  - **Before** (unmodified HEAD content): 12 failed files / 13 failed
    tests / 1598 passed.
  - **After** (this increment's changes): 12 failed files / 13 failed
    tests / 1603 passed (+5: the new e2e test, 2 new topology v2-hint
    tests, 2 new configure v1/v2 fallback tests in
    `document-bootstrap.test.ts`).
  - **Failure set identical by name**, both runs:
    `apps/cli/tests/start.test.ts` (Scenario 4), `packages/agents/tests/catalog.test.ts`,
    `packages/doctor/tests/checks.test.ts` ("repositorio Axiom real"),
    `packages/model-routing/tests/{assignments,loader,opencode-projection,resolver}.test.ts`
    (4 files, canonical-policy-from-disk tests),
    `packages/project-resolution/tests/resolver.test.ts` ("resuelve
    scopes con absolutePath y exists correcto (v1)" — investigated in
    detail below, since it is in the exact package this increment
    depends on),
    `packages/skills/tests/catalog.test.ts`,
    `packages/telemetry/tests/audit-trail-sink.test.ts`,
    `packages/toolchain/tests/repair-add-gitignore.test.ts`,
    `packages/tui/tests/driver.test.ts` (2 tests, configure/upgrade
    TUI post-run summary text).
  - **Specific scrutiny of `resolver.test.ts`'s failure**: this is in
    `@axiom/project-resolution`, the exact package D3 depends on being
    version-aware, so it was investigated rather than assumed
    pre-existing. Root cause: the test's own fixture asserts
    `result.scopes['product'].exists` is `false` for a scope whose
    `path` is `.` (the project root itself, i.e. the temp dir the test
    just created) — trivially always `true` since the directory that
    was just created obviously exists. This is a pre-existing, broken
    test assertion unrelated to `axiom.yaml` schema version dispatch
    (confirmed: this increment never touched `resolver.ts`, and the
    failure reproduces identically — same message, same line — with
    this increment's changes stashed out).
- The manual real-binary CLI walkthrough (see above) is additional,
  qualitative, real-execution validation beyond the automated suites,
  per the task's explicit instruction that this must be checked via
  real execution given the history of a prior revert.

## Result

D3 is implemented, verified end-to-end via both a real CLI-binary
walkthrough and a real (no-mock) automated integration test, and
introduces zero regressions against the documented pre-existing
baseline (confirmed by name-for-name failure-set comparison across a
stash-based before/after bisection). Two previously-unflagged consumer
gaps (`configure.ts`'s `readAxiomYamlProjectName`,
`topology/loader.ts`'s `tryLoadTopologyHint`) were fixed as part of the
same cutover, per the audit's explicit instruction not to treat them
as separate work. One additional, unrelated, pre-existing
build-tooling gap (`packages/cli-commands`'s cross-package `include`
misconfiguration silently dropping 8 files from `apps/cli/dist/`) was
discovered during validation and is documented as a recommended
follow-up, not fixed here (out of this increment's authorized scope).

D1 (`defaultSingleRepoManifest`'s deprecation warning) remains
explicitly out of scope and unimplemented, per the audit's own
scope-split recommendation — a separate, sequenced follow-up increment.

## General spec integration

Updated `Axiom.Spec/general-spec.md`'s existing "Versioning"
(`@axiom/versioning`) section:

- Replaced the stale note that D3 "was already attempted and
  explicitly reverted" (a blocking fact) with a note that it is now
  **resolved**, summarizing what changed, why it was safe, the two
  consumer fixes, and the end-to-end verification performed.
- Updated the "second consecutive increment blocked on the same root
  cause" note (which named D3 as the blocker for INC-20/INC-21) to
  reflect that D3 has landed, while being explicit that D1 (the other
  half of "multi-repo primary") and topology-aware `axiom upgrade`
  wiring remain open, separate gaps — not overstating what this
  increment resolved.

This is consistent with this chain's own stated convention (the audit
increment deferred this exact update "once the D3 follow-up increment
lands and is verified end-to-end" — that condition is now met).

## Closure rationale

`Status: pending`, not `closed`, because per the task's own explicit
instruction: this is a critical, wide-blast-radius cutover with a
documented history of a prior revert (the original
`...-cli` increment), and the task explicitly asks for the next step
to be a validator-reviewer pass with extra scrutiny before closing.
All other closure-rule items are otherwise satisfied (clear goal and
acceptance criteria, changes implemented, real validation executed and
documented in detail, review against acceptance criteria completed,
general-spec integration performed) — the only reason this is not
`closed` is the deliberate extra-scrutiny gate requested for this
specific increment given its risk profile.

## Next step recommendation

**validator-reviewer**, with these exact inputs and extra scrutiny
given the wide blast radius and the history of a prior revert on this
exact thread:

1. Independently re-verify (do not re-trust) that `resolveProject`,
   `validateAxiomYamlContent`, and `@axiom/doctor`'s MC-001/BC-001/
   BC-002 remain version-aware and unregressed, by direct re-read of
   live source — same discipline the audit increment applied.
2. Independently re-run the real CLI walkthrough documented above
   (`init` -> `join` -> `topology show` with `topology.yaml` removed
   -> `configure --target copilot-vscode` with no
   `product.manifest.yaml`), ideally against a genuinely fresh clone/
   checkout rather than trusting this increment's own transcript.
3. Independently run `npm run typecheck`/`build`/`test` and re-confirm
   the 12-failed-file/13-failed-test baseline by name, with particular
   attention to whether the `packages/project-resolution` failure
   analyzed above is really pre-existing and unrelated (re-derive it
   independently rather than trusting this increment's stash-based
   bisection).
4. Confirm the three changed files
   (`apps/cli/src/commands/init.ts`, `apps/cli/src/commands/
   configure.ts`, `packages/topology/src/loader.ts`) and the new/
   updated tests are the only functional changes — no scope creep into
   `packages/cli-commands` (the discovered-but-unfixed build gap) or
   into D1 territory.
5. Decide whether the `packages/cli-commands` build-tooling gap
   documented here warrants its own follow-up increment now, or can
   wait — it does not block D3's correctness (confirmed: typecheck and
   the full vitest suite are both unaffected by it) but does mean the
   CLI binary cannot currently run standalone from a clean build,
   which may be independently important.
6. If all of the above hold, close this increment; if not, document
   specific corrections needed and set `Status: pending` with the
   concrete gap named.

**Separately, after this increment closes**: launch the D1 follow-up
increment (`defaultSingleRepoManifest` deprecation warning + TC-001
`warn` branch), per the audit increment's own next-step brief — now
finally unblocked, since D3 (this increment) has landed.
