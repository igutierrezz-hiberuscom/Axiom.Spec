# Increment: schemaVersion:2 emission cutover (D3 validator-reviewer pass)

Status: closed
Date: 2026-07-03

## Goal

Independently re-verify, with extra scrutiny given this exact thread's
history of one prior revert, that the D3 (`axiom.yaml schemaVersion: 2`)
cutover implemented by
`INC-20260702-schemaversion2-multirepo-primary-reconcile-impl`
(cli-implementer) is genuinely safe to close — not by re-trusting the
implementer's or the audit's own accounts, but by independently
re-deriving the consumer list, independently re-running a widened live
CLI walkthrough, independently re-verifying the version-dispatch logic
by direct source trace, and independently re-running the full validation
suite with a stash-based bisection.

## Context

Parent chain: `INC-20260702-axiom-redesign-roadmap` ->
`INC-20260702-registry-manifest-schema-v2` (migration-engineer audit,
records D1/D2/D3) -> `...-design` (schema-writer) -> `...-impl`
(registry-engineer) -> `...-cli` (cli-implementer: attempted the D3
cutover, **reverted** it after finding `resolveProject` only read v1 —
the exact incident this validator pass exists to not repeat) ->
`...-resolver-fix` (made `resolveProject` version-aware, judged cutover
"now safe", deliberately did not flip it) -> `...-validator` (made
`@axiom/doctor` version-aware, closed INC-01, again left D3 as a named
follow-up) -> INC-20 (`versioning-reconcile`) -> INC-21
(`upgrade-reconcile`) -> `INC-20260702-schemaversion2-multirepo-primary-
reconcile` (migration-engineer audit-only: re-confirmed the read side
is safe, found two new consumer-safety gaps — `configure.ts`'s
`readAxiomYamlProjectName`, `topology/loader.ts`'s
`tryLoadTopologyHint` — recommended splitting D3 and D1, D3 first) ->
`...-impl` (cli-implementer: flipped `init.ts` to emit `schemaVersion:
2`, fixed both newly-found consumer gaps, ran a real CLI walkthrough
plus a negative-test reproduction, ran full validation, explicitly left
`Status: pending` and requested this exact validator-reviewer pass with
extra scrutiny) -> **this increment** (validator-reviewer).

## Scope

1. Independently re-derive the full consumer list for `axiom.yaml`
   (fresh grep across the entire `Axiom/` monorepo for
   `AXIOM_CONFIG_FILENAME`/`axiom.yaml`/`project.name`/`project.mode`/
   `.project.` patterns), not just re-check the prior audit's list.
2. Independently re-run the end-to-end CLI walkthrough against fresh
   real temp directories, widened beyond the implementer's own
   transcript: at least 2 different adapter targets, plus `repo attach`
   for multi-repo-adjacent behavior.
3. Independently verify `configure.ts`'s and `loader.ts`'s fixes by
   tracing the live version-dispatch logic, not by re-running the
   implementer's tests.
4. Re-run `axiom doctor` against a real, independently-created v2
   project and confirm MC-001/BC-001/BC-002.
5. Check for fixture drift: any existing test fixture anywhere in the
   monorepo that hardcodes a v1 `axiom.yaml` sample and asserts it is
   what a fresh `init` produces.
6. Decide whether the newly-found `cli-commands`/`dist` build gap
   duplicates the existing `BUG-20260702-cli-commands-tsconfig-missing-
   emit` ticket, or is a distinct issue.
7. Independently run `npm run typecheck`/`build`/`test` (full
   monorepo) and independently bisect the `project-resolution` failure
   via `git stash`, not by trusting the implementer's own bisection.
8. Record the closure verdict (closed, or blocking issue found) with
   the same seriousness as the original revert decision.
9. Update `Axiom.Spec/general-spec.md`'s versioning/topology sections.

## Non-goals

- No further code changes beyond what was already implemented by the
  `-impl` increment (this pass found no blocking issue requiring a
  fix — see Result).
- D1 (`defaultSingleRepoManifest`'s deprecation warning) — remains a
  separate, still-open follow-up, explicitly out of scope here, same as
  every prior increment in this chain.
- Fixing the `cli-commands`/`dist` build-tooling gap — out of scope;
  decision recorded is to update the existing bug ticket, not to fix it
  in this pass.
- Q3/Q4 from the parent roadmap — untouched, remain open.

## Acceptance criteria

- [x] A fresh, independent grep-based consumer audit was performed
      across the entire `Axiom/` monorepo (not just re-citing the prior
      audit's list), and every new hit was individually inspected and
      classified (real consumer vs. false positive).
- [x] At least one widened walkthrough dimension beyond the
      implementer's own transcript was independently exercised: a
      second adapter target (`github-copilot`, not `copilot-vscode`)
      and `repo attach` for multi-repo-adjacent behavior.
- [x] `configure.ts`'s `readAxiomYamlProjectName` and `loader.ts`'s
      `tryLoadTopologyHint` were read and traced directly against
      `repo.ts`'s already-correct reference implementation, confirming
      byte-for-byte-equivalent dispatch logic (not just "tests pass").
- [x] `axiom doctor` was independently re-run against a project this
      reviewer created via `axiom init` (not the implementer's), and
      MC-001/BC-001/BC-002 all pass.
- [x] Fixture-drift check performed: no test in the monorepo hardcodes
      a v1 `axiom.yaml` sample as `init`'s expected output.
- [x] The `cli-commands`/`dist` build gap was checked against the
      existing `BUG-20260702-cli-commands-tsconfig-missing-emit` ticket
      and a same-root-cause-vs-distinct-issue decision was recorded.
- [x] `npm run typecheck`, `npm run build`, `npm test` were
      independently re-run by this reviewer; results (12 failed files /
      13 failed tests / 1603 passed) were compared name-for-name against
      the implementer's own reported numbers.
- [x] The `project-resolution` resolver failure was independently
      bisected via this reviewer's own `git stash`, not by trusting the
      implementer's bisection narrative.
- [x] A closure verdict was recorded with explicit rationale, treating
      this with the same seriousness as the original revert decision.

## Open questions

None blocking. D1's exact deprecation-warning mechanism remains
deliberately undecided, carried forward unchanged from the audit
increment's own non-goals — not this reviewer's job to resolve.

## Assumptions

- Inherited unchanged from the whole INC-01 chain: non-hard-removal
  posture (v1 `axiom.yaml` remains fully supported on the read side
  indefinitely).
- The real `Axiom/axiom.yaml` (workspace-root manifest describing the
  3-repo `Axiom`/`Axiom.Spec`/`Axiom.SDD` topology) is confirmed, again,
  to be a different, unrelated schema from the `axiom.yaml` this whole
  chain concerns (scaffolded product projects' `AxiomYamlSchema`/
  `AxiomYamlSchemaV2`).
- This reviewer's local `~/.axiom/registry.json` is an unmigrated
  registry v1 file (a pre-existing condition of the reviewer's own
  machine, unrelated to this increment's scope) — this caused some
  walkthrough commands (`init`'s auto-register, `repo attach`'s final
  registry write) to emit an expected, unrelated "registry v1 not
  migrated" warning/failure. This does not affect `axiom.yaml`
  resolution, topology resolution, or `configure` — all of which
  succeeded independently of registry state, and is called out
  explicitly wherever it appears in the walkthrough transcript below so
  it is not mistaken for a D3 regression.

## Implementation notes

### 1. Independent, exhaustive consumer re-audit

Ran fresh greps across the entire `Axiom/` monorepo (`packages/` and
`apps/`) for `AXIOM_CONFIG_FILENAME`, `axiom\.yaml`, `\.project\.(name|
mode)`, and the broader `\.project\b`/`project:` patterns — not
restricted to the prior audit's or implementer's own file lists.

**New files surfaced by the wider grep, individually inspected and
ruled out (false positives / non-consumers, not previously listed by
name in the prior audit but confirmed safe):**

- `packages/mcp-tools/src/implementation-context-handler.ts` — only
  imports types (`PlanMetadata`, etc.); the `.project` grep hit was
  `PlanMetadata`-related, not `axiom.yaml` parsing. Not a consumer.
- `apps/cli/src/commands/validate-changes.ts` — delegates entirely to
  `loadTopology`/plan metadata; never parses `axiom.yaml` directly. Not
  an independent consumer (transitively safe via `@axiom/topology`,
  already audited).
- `apps/cli/src/commands/app-api.ts` — uses `getProjectV2`/
  `resolveProjectRoot`/`loadTopology`, all already-normalized-shape
  consumers. Not a raw `axiom.yaml` parser.
- `packages/filesystem-truth/src/discovery.ts` /
  `packages/filesystem-truth/src/index.ts` — only path discovery
  (`fs.existsSync(configPath)`); never parses YAML content or reads
  `project.name`/`mode`. Not version-sensitive.
- `packages/core/src/types.ts`, `packages/core/src/paths.ts`,
  `packages/user-workspace/src/registry-types.ts`,
  `packages/document-bootstrap/src/canonical-agents-md.ts`,
  `apps/cli/src/commands/model.ts`,
  `packages/document-bootstrap/src/variables.ts` — comment-only or
  type-doc-only references to `project.name`/`axiom.yaml`, OR (for
  `registry-types.ts`) a **different, unrelated** `schemaVersion` field
  belonging to the registry v1/v2 document (`~/.axiom/registry.json` /
  `projects.yml`), explicitly and correctly disambiguated in that
  file's own `AD-09` comment. `variables.ts`'s `axiomYamlProjectName`
  parameter is a plain resolved `string` fed by `configure.ts`'s
  already-fixed helper — not an independent parser.

**No new genuine consumer gap was found** beyond the two the audit
increment already identified and the implementer already fixed
(`configure.ts`'s `readAxiomYamlProjectName`, `loader.ts`'s
`tryLoadTopologyHint`). The exhaustive re-sweep is a **negative
result** — the same conclusion the audit reached, now confirmed via an
independently-run search rather than re-citing the audit's own list.

### 2. Version-dispatch logic: direct trace, not test-trust

Read `configure.ts`'s `readAxiomYamlProjectName` and `loader.ts`'s
`tryLoadTopologyHint` in full, side by side with `repo.ts`'s
already-correct `readProjectNameFromYaml` (the reference
implementation cited by both). Confirmed: both fixes implement
byte-for-byte-equivalent dispatch logic — `schemaVersion === 2` checked
first, top-level `name`/`mode` read on that branch, fallthrough to v1
`project.name`/`project.mode` otherwise, `try`/`catch` returning
`undefined`/`null` on any parse failure. Also re-read
`packages/project-resolution/src/resolver.ts` (unchanged since the
resolver-fix increment; still dispatches via
`validateAxiomYamlContent`) and `packages/doctor/src/checks.ts`'s
MC-001/BC-001/BC-002 (unchanged since the validator increment;
`usesInstalledMultiRepo` still checks both `rawProjectMode` (v1) and
`rawTopLevelMode` (v2)). No regression found in any of the four
already-version-aware consumers.

### 3. Widened live CLI walkthrough (independently run, real temp dirs)

Built the monorepo fresh (`npm run build`, clean, zero errors), applied
the same throwaway `packages/cli-commands/dist/` file-copy workaround
the implementer documented (to work around the pre-existing,
unrelated `BUG-20260702-cli-commands-tsconfig-missing-emit` gap and run
the real standalone CLI binary), ran the walkthrough in genuinely fresh
temp directories this reviewer created, then deleted the workaround
afterward. Node v24.13.1, Windows.

1. `axiom init --yes` in a fresh temp dir → `axiom.yaml`:
   ```
   schemaVersion: 2
   projectId: walkthrough1
   name: walkthrough1
   repoId: walkthrough1-sdd
   role: sdd
   mode: installed-multi-repo
   ...
   ```
   Matches the implementer's claimed shape exactly.
2. `axiom join --member user:reviewer --no-register` → succeeded
   (`[axiom join] Miembro agregado.`), independently confirming
   `resolveProject` resolves this reviewer's own freshly-created v2
   document to `status: 'resolved'`.
3. `axiom topology show` with `axiom.spec/config/topology.yaml`
   **removed** (forcing the `tryLoadTopologyHint` fallback):
   ```
   mode: multi-repo
   sdd-repo: sdd-repo @ .
   spec-repo: spec-repo @ ../walkthrough1.spec
   ```
   Confirms the v2 hint branch resolves correctly for an
   independently-created project, not just the implementer's own.
4. **Widened beyond the implementer's own transcript**: ran
   `axiom configure` against a **second, independently-created**
   fresh temp dir with default target **`github-copilot`** (not
   `copilot-vscode`, which is all the implementer tested), with
   `copilot-instructions.template.md` and `profiles.yaml` seeded and
   critically **no** `product.manifest.yaml`:
   ```
   [axiom configure] OK.
     generatedFiles: 2
     externalDependencies: 1
   ```
   `.github/copilot-instructions.md` rendered with `{{project.name}}`
   resolved to the v2 `axiom.yaml`'s top-level `name` — confirms
   `writeCopilotForTarget`'s fallback path (which applies to both
   `copilot-vscode` and `github-copilot`) works for the untested
   target too, not just the one the implementer checked.
5. **Independent negative-test reproduction** (own patch, own temp
   dir, not the implementer's): patched the compiled
   `readAxiomYamlProjectName` in `apps/cli/dist/commands/configure.js`
   to the pre-fix v1-only implementation, re-ran `axiom configure`
   against a fresh v2 project (target `copilot-vscode`, no
   `product.manifest.yaml`) → reproduced the exact predicted hard
   failure:
   ```
   [axiom configure] Error: writeCopilotInstructions falló para
   adapterTarget=copilot-vscode: ... missing required variable
   project.name (source: product.manifest.product.name o
   axiom.yaml#project.name).
   ```
   Restored the real fix, re-ran: exit code 0, success. Confirms both
   that the bug was real and that the fix genuinely closes it —
   independently, not by trusting the implementer's own transcript.
6. **`repo attach` for multi-repo-adjacent behavior** (new coverage,
   not in the implementer's own walkthrough): ran `axiom repo attach
   code-repo --role code` from the resolved v2 project root. Result:
   the command correctly resolved the v2 project and reached the
   registry-write stage (failing only on this reviewer's own machine's
   pre-existing unmigrated registry v1 state — an unrelated,
   environment-specific condition, not a D3 regression). This confirms
   `repo attach`'s project-resolution path is unaffected by the D3
   cutover for a v2 project.
7. `axiom doctor` against the walkthrough-1 v2 project (independently
   created by this reviewer, not reused from the implementer's run):
   `MC-001`, `BC-001`, `BC-002` all `✓` (pass). 2 unrelated failures
   (`GC-002` empty skills lockfile, `MEMORY`'s `mcp-manifest.yaml`
   missing) and 19 skips are minimal-fixture scaffolding gaps, not
   caused by this increment — same caveat the implementer's own run
   noted, now independently confirmed on a different project instance.

All temp directories and the throwaway `dist/` patch were deleted
after the walkthrough; the real `configure.js` was restored and
re-verified working before cleanup.

### 4. Fixture-drift check

Searched for `runInit` usages across all `*.test.ts` files (9 call
sites) and for any hand-written `axiom.yaml` fixture content in test
files. `apps/cli/tests/init.test.ts` (the one file that specifically
asserts `axiom.yaml`'s shape post-`init`) already asserts
`schemaVersion` is `2` — no drift. Every other `runInit` call site
(`sync.test.ts`, `context.test.ts`, `configure.test.ts`, `join.test.ts`)
uses `runInit` purely as setup and asserts on unrelated derived state
(`init.json`, registry entries, `install-profile.json`), never on
`axiom.yaml`'s own shape — safe regardless of version, since
`resolveProject` is version-agnostic downstream. Files that DO write
v1 `axiom.yaml` content directly as test fixtures
(`packages/doctor/tests/*.test.ts`, `apps/cli/tests/bootstrap.test.ts`)
do so deliberately to exercise the still-fully-supported v1 read path
— legitimate, not drift, consistent with the chain's non-hard-removal
posture. **No fixture drift found.**

### 5. `cli-commands`/`dist` build gap: same root cause, existing ticket updated

Independently reproduced the gap (clean build, exit 0, zero errors,
yet 8 files missing from `apps/cli/dist/commands/`). Checked
`Axiom.Spec/bugs/BUG-20260702-cli-commands-tsconfig-missing-emit/
README.md` (filed during INC-08, predates this chain): **same root
cause** — `packages/cli-commands/tsconfig.json`'s cross-`include` of
`apps/cli/src/commands/*.ts` files. That ticket named 6 affected files
(`_shared`, `configure`, `sync`, `upgrade`, `model`, `components`); the
implementer's finding named 8 (the same 6 plus `index-cmd` and
`validate-changes`, both added to the codebase after the bug was
originally filed). **Decision: this is a rediscovery, not a distinct
issue.** Updated the existing bug ticket (not creating a new one) to
record the rediscovery, the 2 additional affected files, and a note
that this has now surfaced three times across three separate
increments — a reasonable signal for future prioritization, but not
escalated into its own ticket per the task's framing (same root cause,
same fix scope, same ticket).

### 6. Independent validation re-run

- `npm run typecheck` (`tsc -b`, full monorepo): clean, zero errors —
  independently re-run, not re-trusted.
- `npm run build` (`tsc -b`, full monorepo): clean, zero errors.
- `npm test` (vitest, full monorepo), independently re-run: **12
  failed files / 13 failed tests / 1603 passed** — matches the
  implementer's reported post-change numbers exactly, name-for-name,
  independently confirmed by extracting and diffing the failing test
  list.
- **Independent stash-based bisection** (this reviewer's own, not the
  implementer's): stashed out `init.ts`, `configure.ts`, `loader.ts`,
  and their test changes; moved the new `schemaversion2-e2e.test.ts`
  aside; re-ran `packages/project-resolution/tests/resolver.test.ts`
  in isolation against unmodified HEAD content. Result: **identical
  failure** (`resuelve scopes con absolutePath y exists correcto (v1)`,
  same assertion, same line 107, `expected true to be false`) — 24
  passed / 1 failed, matching the full-suite run exactly. Root cause
  independently confirmed: the test's own `VALID_AXIOM_YAML` fixture
  declares `scopes.product.path: .` (the project root itself), which
  the test's own assertion (`scopes['product'].exists` should be
  `false`) contradicts — the root dir trivially always exists. This is
  a pre-existing, broken test assertion, confirmed unrelated to
  `axiom.yaml` schema-version dispatch by direct reproduction on
  unmodified source. Restored the stash afterward; re-confirmed
  `typecheck` clean post-restore.
- Confirmed via `git status` that only the implementer's documented
  files were modified/added for this increment
  (`init.ts`/`configure.ts`/`loader.ts` + their tests +
  `schemaversion2-e2e.test.ts`); the wider set of ~100 other modified/
  untracked files in the working tree is pre-existing, accumulated,
  uncommitted state from the rest of the increment chain (this
  workspace has not committed between increments), not scope creep
  introduced by D3.

## Validation

Real validation was executed (not the no-command fallback statement).

- `npm run typecheck`: clean, zero errors.
- `npm run build`: clean, zero errors.
- `npm test`: 12 failed files / 13 failed tests / 1603 passed —
  independently confirmed identical to the implementer's reported
  numbers, name-for-name.
- Independent `git stash`-based bisection of the one failure in the
  package this increment depends on
  (`packages/project-resolution/tests/resolver.test.ts`): confirmed
  pre-existing, reproduces identically on unmodified HEAD content.
- Real CLI walkthrough, independently run, widened beyond the
  implementer's own coverage (second adapter target `github-copilot`,
  `repo attach`, an independent negative-test reproduction, an
  independent `axiom doctor` run) — see Implementation notes §3.

## Result

**No blocking issue found.** Independent re-verification, with extra
scrutiny given this thread's history of one prior revert, confirms:

- The consumer-safety audit is exhaustive: a fresh, independently-run
  grep sweep across the entire monorepo found no new genuine consumer
  gap beyond the two the audit increment already identified and the
  implementer already fixed.
- `configure.ts`'s and `loader.ts`'s fixes are correct by direct
  source trace (not test-trust), byte-for-byte equivalent to the
  already-correct reference implementation in `repo.ts`.
- The live CLI walkthrough reproduces correctly on independently
  created projects, widened to a second adapter target
  (`github-copilot`) and `repo attach`, beyond what the implementer
  tested.
- `axiom doctor`'s MC-001/BC-001/BC-002 pass against an
  independently-created real v2 project.
- No fixture drift exists anywhere in the monorepo.
- The `cli-commands`/`dist` build gap is a rediscovery of an existing,
  already-tracked bug (`BUG-20260702-cli-commands-tsconfig-missing-
  emit`), now updated with the 2 additional affected files — not a
  distinct issue, not blocking, not fixed here.
- `npm run typecheck`/`build`/`test` are independently confirmed clean
  / at the exact same pre-existing baseline, with the one
  package-adjacent failure (`project-resolution/tests/resolver.test.ts`)
  independently bisected and confirmed pre-existing and unrelated.

**D3 is genuinely resolved** — not merely attempted a second time. The
original revert's root cause (an unaudited consumer breaking silently)
has now been checked exhaustively, twice, by two different roles using
two different methods (the audit's structured consumer-by-consumer
review, and this pass's fresh independent grep sweep), with no further
gap surfacing either time.

## General spec integration

Updated `Axiom.Spec/general-spec.md`'s "Versioning" section: confirmed
D3 is closed (not just implemented-pending-review), and updated the
note about D1 to make explicit it is the sole remaining open half of
"multi-repo primary" now that D3 has been independently validated.

## Closure rationale

`Status: closed`. All closure-rule items are satisfied:

- Goal and acceptance criteria are clear and every acceptance criterion
  above is checked and evidenced independently (not re-trusted from
  either prior increment).
- Changes were implemented (by the `-impl` increment); this pass
  required no further code changes, since no blocking issue was found.
- Available validation was executed independently
  (`typecheck`/`build`/`test`, stash-based bisection, widened live CLI
  walkthrough).
- Review against acceptance criteria and against the original intent
  (extra scrutiny given the prior-revert history) was completed.
- Stable knowledge was integrated into `Axiom.Spec/general-spec.md`.
- The result is documented clearly, with an explicit, evidenced
  closure verdict treating this with the same seriousness as the
  original revert decision.

Per the task's own explicit instruction, closing this increment also
closes the overall D3 thread from INC-01's original resolved decisions
(the multi-increment chain spanning the audit, the implementation, and
this validator pass).

## Next step recommendation

**D3 is genuinely unblocked now.** Two viable next steps, in order of
this reviewer's recommendation:

1. **Recommended first: return to Phase G's INC-22** ("Configure/
   upgrade/repair operations on existing installs" per the roadmap),
   since it was explicitly blocked on this exact root cause by two
   prior audit increments (INC-20 `versioning-reconcile`, INC-21
   `upgrade-reconcile`, both of which called this "the blocking root
   cause" for their own scope) and D3 is now closed, not just
   attempted. This unblocks the longest-waiting, most-cited dependency
   chain in the roadmap.
2. **After INC-22, or in parallel if capacity allows: the D1
   follow-up** (`defaultSingleRepoManifest`'s deprecation warning +
   `@axiom/doctor` TC-001 `warn` branch), per the original audit
   increment's own scope-split recommendation — now finally
   meaningful to implement, since D3's v2/multi-repo path genuinely
   exists for users to opt into (the audit's own stated precondition
   for D1's warning not being premature/noisy).

This reviewer recommends INC-22 first: it has two independent prior
increments explicitly waiting on this exact thread, while D1 has none
(it is a standalone, self-contained improvement with no other
increment blocked on it). Prioritizing the longer dependency chain
first reduces total blocked work across the roadmap.
