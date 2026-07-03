# Increment: Reconcile derived caches / index rebuild — validator-reviewer

Status: closed
Date: 2026-07-02

## Goal

Execute the **validator-reviewer** step of INC-07 of the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
Phase C): independently re-verify the index-engineer implementation against
the migration-engineer audit's brief, re-run real validation from a clean
state, and close INC-07 as a whole if everything holds up — or report back
with specific gaps if not.

## Context

Full INC-07 chain read before this review:

1. Parent roadmap — `Axiom.Spec/specs/increments/
   INC-20260702-axiom-redesign-roadmap/README.md` (INC-07 entry, Phase C,
   and its explicit dependents INC-08/INC-11/INC-12).
2. migration-engineer audit — `Axiom.Spec/specs/increments/
   INC-20260702-index-rebuild-reconcile/README.md`.
3. index-engineer implementation — `Axiom.Spec/specs/increments/
   INC-20260702-index-rebuild-reconcile-impl/README.md`.

This review does not trust either prior document's prose. Every claim below
was independently re-checked against the actual code in `Axiom` and against
fresh command output.

**Important scope note discovered during this review**: `Axiom`'s working
tree currently has substantial uncommitted changes unrelated to INC-07
(`packages/project-resolution/*`, `packages/filesystem-truth/*`,
`packages/user-workspace/*`, `packages/installer/src/registry.ts`,
`packages/tui/src/driver.ts`, `apps/cli/src/commands/tui.ts`, plus new
`apps/cli/tests/tui.test.ts` and `packages/workflow/src/artifact-id.ts` +
its test) — none of these are mentioned in either INC-07 spec's scope, and
grepping their diffs confirms they are about project-resolution fallback
(addendum-14 ancestor-path/git-root detection) and TUI/registry-v2 work,
not index/cache/list concerns. This review only assesses the files INC-07
itself declares in scope; the unrelated uncommitted work is explicitly out
of scope here and untouched by this review.

## Scope

- Independent line-by-line review of `listArtifacts` in
  `Axiom/packages/workflow/src/artifact-store.ts`.
- Independent review of the `list` subcommands
  (`apps/cli/src/commands/artifact-metadata-cli.ts`) and `index rebuild`/
  `index validate` (`apps/cli/src/commands/index-cmd.ts`), confirming they
  call `listArtifacts` (no parallel scan) and that `index-cmd.ts` performs
  no cache writes.
- Independent review of `@axiom/doctor`'s new `IX-001` check
  (`runArtifactIndexChecks` in `packages/doctor/src/checks.ts`), confirming
  it imports and reuses `listArtifacts` (not reimplemented) and matches the
  existing `DoctorCheck{id,category,description,status,evidence}` shape and
  `pass/fail/warn/skip` convention, and is wired into `runDoctorChecks`.
- `git diff`/`git status` re-confirmation that `@axiom/workflow`'s
  state-machine/state-store/hooks files are untouched and no `.axiom/cache`
  path exists anywhere in the working tree.
- Re-read of `packages/document-bootstrap/src/canonical-agents-md.ts`'s
  boilerplate promising `axiom index rebuild`/`axiom index validate`/
  `axiom doctor`, confirmed against the actual shipped command names and
  behavior.
- Fresh, independent `npm run typecheck` / `npm run build` / `npm test` run
  on the full monorepo, plus a `git stash`/`git stash pop` bisection to
  isolate INC-07's own test delta from other unrelated uncommitted work
  already present in the tree (restored afterward, verified byte-for-byte).
- Closure assessment for INC-07 as a whole (all three subagent steps).

## Non-goals

- No further code changes — this is a review-and-close step. No bugs were
  found that required a fix (see Result).
- No re-litigation of design choices already made and documented by the
  migration-engineer audit or index-engineer implementation (three
  kind-scoped `list` subcommands vs. a unified `artifact list --kind`
  command; cache-less `rebuild`/`validate`) — these were reviewed for
  internal consistency, not re-decided.
- No action on the unrelated uncommitted changes found in the working tree
  (project-resolution/filesystem-truth/user-workspace/tui/registry) — out
  of scope for INC-07, not modified or evaluated for correctness here.

## Acceptance criteria

- [x] `listArtifacts` independently confirmed to: (a) return an empty
      result without throwing when the artifact-kind folder is absent,
      (b) skip and report a malformed/unparseable `metadata.yml` as a
      `failure` without aborting the rest of the scan, (c) map
      `increment/bug/plan` to `increments/bugs/plans` correctly.
- [x] `list` subcommands and `index rebuild`/`index validate` independently
      confirmed to call `listArtifacts` (no duplicate scan logic), and
      `index-cmd.ts` confirmed to contain zero `writeFile`/`fs.write*`
      calls targeting any cache path.
- [x] `IX-001` independently confirmed to import `listArtifacts` from
      `@axiom/workflow` (not reimplemented), match the existing
      `DoctorCheck` shape/conventions, and be wired into `runDoctorChecks`.
- [x] `git diff` independently confirms `@axiom/workflow`'s state machine,
      `state-store.ts`, `hooks.ts`, `workflows-loader.ts` are untouched;
      only `artifact-store.ts`, `artifact-id.ts` (pre-existing, INC-06),
      `index.ts` (barrel), and `package.json` changed in that package.
      `git status` independently confirms zero `.axiom/cache` paths exist
      anywhere in the working tree.
- [x] Canonical `AGENTS.md` boilerplate (`canonical-agents-md.ts`) promise
      independently confirmed to match shipped command names and behavior
      (`axiom index validate`, `axiom index rebuild`, `axiom doctor`).
- [x] `npm run typecheck`, `npm run build` independently re-run: clean,
      zero errors.
- [x] `npm test` independently re-run: **12 failed files / 13 failed tests
      / 1320 passed (1333 total)** — matches the index-engineer report's
      post-change figures exactly.
- [x] `git stash`/`git stash pop` bisection performed and reverted cleanly
      (working tree restored to its exact pre-bisection state, verified).
- [x] INC-07 closure assessment completed, with deferred items explicitly
      named.

## Open questions

None blocking. One documentation-precision discrepancy was found in the
prior specs' baseline test-count claims (see "Result" — Finding V1); it
does not affect INC-07's functional correctness and does not block closure.

## Assumptions

- The pre-existing uncommitted changes found in the working tree
  (project-resolution/filesystem-truth/user-workspace/tui/registry,
  `artifact-id.ts`) are earlier, still-uncommitted work from a different
  increment (addendum-14 project-resolution fallback, registry v2, TUI
  wiring), not part of INC-07. This is inferred from: (a) neither INC-07
  spec's scope/file list mentions any of them, (b) their diff content is
  about project-resolution ancestor/git-root detection and registry v2
  migration, unrelated to artifact listing/indexing, (c) `artifact-id.ts`
  is imported by `artifact-store.ts` as a pre-existing INC-06 type
  (`ArtifactKind`), consistent with the INC-06 chain, not authored by
  INC-07. This review does not disturb or evaluate that other work.
- Per `Axiom.SDD/AGENTS.md`'s closure rules and this repo's own established
  chain pattern (INC-06's validator-reviewer step performed final
  consolidation), this document performs the INC-07-as-a-whole closure.

## Implementation notes

No implementation changes were made in this step — this is independent
verification only. Findings below.

### V1 — `listArtifacts` correctness: confirmed, no bugs found

Read `Axiom/packages/workflow/src/artifact-store.ts` in full (514 lines,
not excerpted). `listArtifacts` (lines 427-464):

- **(a) Absent folder** — `fs.readdirSync(kindDir, { withFileTypes: true })`
  wrapped in `try/catch`; catch branch returns
  `{ entries: [], failures: [] }` unconditionally, no re-throw. Confirmed
  correct: an absent `increments/`/`bugs/`/`plans/` folder (or an absent
  `axiom.spec/` scope entirely) cannot crash the scan.
- **(b) Malformed `metadata.yml`** — delegates to the existing, already
  battle-tested `loadArtifactMetadata` per subdirectory (no parsing logic
  duplicated); an `err(...)` result is pushed to `failures` and the `for`
  loop `continue`s to the next directory entry. Confirmed: one corrupt
  `metadata.yml` cannot abort the rest of the scan. A folder that exists
  but has no `metadata.yml` at all (`loadResult.value === undefined`) is
  correctly treated as a distinct, non-error case (silently skipped, not
  pushed to `failures`) — this matches `loadArtifactMetadata`'s own
  pre-existing `ok(undefined)`-for-absent-file contract and is a
  deliberate, documented design choice (see the impl spec's Assumptions),
  not an oversight.
- **(c) Kind-to-folder mapping** — `ARTIFACT_FOLDER` (lines 32-36) maps
  `increment→'increments'`, `bug→'bugs'`, `plan→'plans'`, exactly matching
  the folder-per-instance layout `<specPath>/{increments,bugs,plans}/<ID>/`
  established by INC-06 and used consistently by every other function in
  the file (`resolveArtifactDir`, etc.).

**No bugs found in `listArtifacts`.**

### V2 — CLI wiring: confirmed no duplicate scan, no cache writes

Read `Axiom/apps/cli/src/commands/index-cmd.ts` (239 lines) and
`Axiom/apps/cli/src/commands/artifact-metadata-cli.ts` (392 lines) in full.

- `runIndexRebuild` and `runIndexValidate` both call
  `listArtifacts(args.projectRoot, kind)` per kind (3x each, for
  `increment`/`bug`/`plan`) — no re-implementation of directory scanning
  or YAML parsing anywhere in `index-cmd.ts`.
- `runListCommand` (in `artifact-metadata-cli.ts`, shared by all three
  `addListSubcommand` call sites) also calls `listArtifacts` directly.
  Grepped both files for `readdirSync`/`readdir`/`glob`: zero matches
  outside `artifact-store.ts` itself — confirmed single scan primitive,
  no parallel implementation.
- Grepped `index-cmd.ts` explicitly for `writeFile|fs\.write|mkdirSync|
  createWriteStream`: **zero matches**. `runIndexRebuild`'s "rebuild" is
  read-only — it constructs a summary string/object from `listArtifacts`'
  return value and returns it; nothing is persisted to disk.
- Confirmed the three CLI files wire the shared list helper correctly:
  `axiom-increment.ts:627`, `axiom-bug.ts:500`, `axiom-plan.ts:530` each
  call `addListSubcommand(cmd, { ownKind: '<kind>', tag: TAG })`, and
  `apps/cli/src/index.ts` imports and calls `registerIndex(program)`
  (line 110 import, line 162 call site).

**No bugs found; no duplicate scan logic; no cache writes.**

### V3 — `@axiom/doctor` `IX-001` check: confirmed correct, reuses `listArtifacts`

Read `runArtifactIndexChecks` (`packages/doctor/src/checks.ts`, lines
2260-2315) and its wiring in `packages/doctor/src/index.ts` in full.

- `checks.ts` line 34: `import { listArtifacts, type ArtifactKind } from
  '@axiom/workflow';` — confirmed imported, not reimplemented.
- The function calls `listArtifacts(resolution.rootPath, kind)` for all
  three kinds inside a loop, aggregates `entries`/`failures` counts, and
  only then decides `pass`/`fail`/`skip` — no independent directory
  traversal.
- Shape match confirmed against `DoctorCheck` (`checks.ts` lines 40-46:
  `{ id, category, description, status, evidence? }`) and the `pass`/
  `fail`/`skip` helper signatures (lines 59-73): `runArtifactIndexChecks`
  calls `skip('IX-001', CATEGORY_ARTIFACT_INDEX, desc, reason)`,
  `fail('IX-001', CATEGORY_ARTIFACT_INDEX, desc, evidence)`, and
  `pass('IX-001', CATEGORY_ARTIFACT_INDEX, desc, evidence)` — argument
  order and types match every other check in the file exactly (spot-
  checked against `runAgentsCatalogCoverageCheck`'s equivalent calls).
- Skip condition (`resolution.status !== 'resolved'`) matches the same
  condition used by every other check category in `checks.ts` (verified
  by grep across the file — this is the file's established convention,
  not a new one invented for `IX-001`).
- `packages/doctor/src/index.ts`: `runArtifactIndexChecks` is imported
  (line 21), re-exported from the barrel (line 46), a
  `CATEGORY_ARTIFACT_INDEX = 'artifact-index'` constant is defined
  (line 58) matching the existing `CATEGORY_TOOL_ROUTING`/
  `CATEGORY_MEMORY`/etc. pattern, and `...runArtifactIndexChecks(resolution)`
  is spread into `runDoctorChecks`'s aggregate `checks` array (line 85),
  the last entry — correctly wired into the main aggregation, not
  orphaned.
- Test coverage independently confirmed in
  `packages/doctor/tests/checks.test.ts`: three dedicated `it(...)` blocks
  cover the `skip` (not-found project), `pass` (no metadata.yml / all
  valid), and `fail` (at least one corrupt metadata.yml) branches.

**No bugs found; correctly reuses `listArtifacts`; correctly wired.**

### V4 — Scope isolation: confirmed via direct `git diff`/`git status`

- `git diff --stat -- packages/workflow/` shows only `package.json` (+4/-1,
  adding the `js-yaml`/`@types/js-yaml` dependency used by
  `artifact-store.ts`, pre-existing from INC-06) and `src/index.ts` (barrel
  re-exports, +35 lines) as *modified* files; `artifact-store.ts` and
  `artifact-id.ts` (+ their tests) are new/untracked. **No
  `state-store.ts`, `hooks.ts`, or `workflows-loader.ts` appear anywhere in
  this diff** — confirmed untouched.
- `git status --short` filtered for `.axiom` and a filesystem search for
  any `.axiom` directory in the working tree (excluding `node_modules`)
  both returned **zero matches**. No `.axiom/cache/*.index.json` or any
  other cache artifact exists anywhere.

**Both non-goal checks confirmed clean.**

### V5 — Canonical `AGENTS.md` promise: confirmed fulfilled

Read `Axiom/packages/document-bootstrap/src/canonical-agents-md.ts` lines
70-78 directly:

```
Never edit generated index/cache files manually.

If an index appears stale or inconsistent, run:
- axiom index validate
- axiom index rebuild
- axiom doctor
```

All three commands now exist, under the exact names promised:
`axiom index validate` and `axiom index rebuild` are registered by
`registerIndex` (`index-cmd.ts`), and `axiom doctor` already existed and
now additionally covers the artifact-store/`metadata.yml` surface via
`IX-001`. Behavior is sane for a user following this advice literally:
`rebuild` re-scans and reports counts/failures (readable, non-destructive,
exit 0), `validate` re-scans and exits 1 if any `metadata.yml` fails to
parse (CI-usable signal), and `doctor` now flags the same parse-failures
as part of its broader health report. **The gap the migration-engineer
audit's Finding 1 identified is closed**: a user or agent following this
generated boilerplate today no longer hits a "command not found."

### V6 — Real validation: independently re-run, with one documentation-precision correction

Executed directly, not inferred from prior specs:

1. `npm run typecheck` (`tsc -b`, full monorepo) — **clean, zero errors**.
2. `npm run build` (`tsc -b`, full monorepo) — **clean, zero errors**.
3. `npm test` (`vitest run`, full monorepo, current working tree with all
   INC-07 changes present) — **12 failed files / 13 failed tests / 1320
   passed (1333 total)**. The failed-file list, independently enumerated,
   is byte-for-byte identical to the one the index-engineer's spec lists:
   `apps/cli/tests/start.test.ts`, `packages/agents/tests/catalog.test.ts`,
   `packages/doctor/tests/checks.test.ts` (the same pre-existing
   `resolveProject` returns `'ambiguous'` environment-dependent failure),
   `packages/model-routing/tests/{assignments,loader,opencode-projection,
   resolver}.test.ts` (4 files), `packages/project-resolution/tests/
   resolver.test.ts`, `packages/skills/tests/catalog.test.ts`,
   `packages/telemetry/tests/audit-trail-sink.test.ts`,
   `packages/toolchain/tests/repair-add-gitignore.test.ts`,
   `packages/tui/tests/driver.test.ts` (2 of its tests). None of these
   touch any INC-07 file. **This matches the index-engineer's reported
   post-change figure exactly.**
4. **`git stash`/`git stash pop` bisection**, done twice for precision:
   - First pass: stashed *all* uncommitted changes (including the
     unrelated project-resolution/tui/registry work also present in the
     tree) and re-ran `npm test`: **12 failed / 13 failed / 1231 passed**.
     This is a "remove everything uncommitted" baseline, not a clean
     "INC-07 only" baseline.
   - Second pass (more precise): stashed *only* the files INC-07 itself
     declares in scope (`artifact-store.ts` + its test, `index-cmd.ts` +
     its test, `artifact-metadata-cli.ts` + its test, the three
     `axiom-{increment,bug,plan}.ts` diffs, their new metadata tests,
     `apps/cli/src/index.ts`, `packages/doctor/{checks,index}.ts` +
     `checks.test.ts`, `packages/doctor/{package.json,tsconfig.json}`,
     `packages/workflow/{package.json,src/index.ts}`), leaving the other,
     unrelated uncommitted work untouched. Re-ran `npm test`: **12 failed
     / 13 failed / 1263 passed (1276 total)** — same failed-file set as
     both other runs. Then ran `git stash pop` and confirmed via
     `git status --short` (37 entries, matching the pre-bisection count
     exactly) that the working tree was restored losslessly.
   - **Both stash passes confirm the load-bearing claim**: the set of
     failing tests is identical with or without INC-07's changes present
     (12 files / 13 tests, same names). **Zero regressions attributable to
     INC-07.**
5. **Documentation-precision finding (not a functional defect)**: the
   isolated INC-07-only delta measured here is **1263 → 1320 = +57 net new
   passing tests**, not the index-engineer spec's stated "+22 net new
   (1298 → 1320)". The "1298" figure in both the migration-engineer audit's
   context and the index-engineer's own "baseline confirmed first" claim
   does not reproduce when isolating strictly by file scope as done here —
   most likely it was measured against a tree state where some of the
   other, unrelated uncommitted work (project-resolution/tui/registry) was
   also absent at the time, which is plausible given multiple increments
   share this same working tree without intermediate commits. This is a
   **test-count bookkeeping discrepancy in the prior specs' baseline
   claim, not a bug in the INC-07 implementation**: the actual code
   changes are correct (V1-V5 above), the typecheck/build are clean, and
   the failing-test set is unchanged with or without INC-07's own diff.
   Recorded here for accuracy; does not block closure.
6. Scoped re-run of only the touched/new INC-07 test files via
   `npx vitest run <7 files>`: **72 passed, 1 failed** (the same
   pre-existing `resolveProject` environment-dependent failure) —
   independently reproduces the index-engineer spec's scoped-run claim
   exactly.

## Validation

Real validation was executed (commands exist and were run, not
best-effort/inspection-only):

- `npm run typecheck` — clean.
- `npm run build` — clean.
- `npm test` (full monorepo) — 12 failed files / 13 failed tests / 1320
  passed (1333 total); independently reproduced.
- `npm test` scoped to the 7 touched/new INC-07 test files — 72 passed, 1
  pre-existing environment-dependent failure; independently reproduced.
- `git stash`/`git stash pop` bisection (two passes, described in V6) to
  isolate INC-07's own test delta; working tree fully restored afterward,
  verified via `git status --short` entry count before/after.
- Direct `git diff`/`git status` inspection (not reliance on prior specs'
  file lists) to confirm scope boundaries (V4).
- Direct source reads (not excerpts) of all five INC-07-declared code
  changes (V1-V3) plus the `canonical-agents-md.ts` boilerplate (V5).

## Result

**All of INC-07's acceptance criteria, across all three subagent steps,
are independently confirmed met.** No bugs were found in `listArtifacts`,
the `list`/`index rebuild`/`index validate` CLI commands, or the new
`IX-001` doctor check — all three pieces correctly share the single
`listArtifacts` scan primitive with no duplication, exactly as the
migration-engineer audit required and the index-engineer implementation
claimed. No `.axiom/cache/*.index.json` file or any persistent cache exists
anywhere in the working tree. `@axiom/workflow`'s state machine, state
store, hooks, and workflow loader are untouched. The canonical `AGENTS.md`
boilerplate's `axiom index rebuild`/`axiom index validate`/`axiom doctor`
promise is now genuinely fulfilled by matching, correctly-behaving commands.

One discrepancy was found and is recorded transparently: the prior specs'
claimed pre-INC-07 baseline of "1298 passed" does not reproduce under a
strict file-scope-isolated bisection (this review measured 1263 passed
with only INC-07's own files stashed out, or 1231 passed with everything
uncommitted stashed out) — the true INC-07-attributable delta is +57 net
new passing tests, not +22. This is a bookkeeping imprecision in how the
baseline was described across a working tree that also holds other,
unrelated uncommitted work — it does not indicate any regression or defect
in INC-07's own code, since the failing-test set (12 files / 13 tests) is
proven identical with or without INC-07's changes in both bisection passes.

## General spec integration

`Axiom.Spec/general-spec.md` does not exist in this repository (confirmed
directly — same finding as every prior spec in the INC-06/INC-07 chain).
The closest existing equivalents are `Axiom.Spec/specs/00_Resumen_Ejecutivo.md`
through `08_Glosario.md`, which are out of scope for this increment's
integration step (they are the source specification documents, not a
consolidated "stable knowledge" file this workflow is meant to populate).

Per `Axiom.SDD/AGENTS.md`'s Documentation Rules ("if no stable knowledge
needs integration, state that explicitly"): no separate `general-spec.md`
integration was performed, because none exists to integrate into. The
stable knowledge this increment establishes — `listArtifacts` as the one
direct-scan primitive for increments/bugs/plans, `axiom index rebuild`/
`axiom index validate` as cache-less wrappers over it, `IX-001` as the
doctor check reusing it, and the explicit deferral of a persistent
`.axiom/cache/*.index.json` file until a concrete consumer or measured
volume justifies it — is fully captured in this closed increment chain
(`INC-20260702-index-rebuild-reconcile{,-impl,-validator}`) and in the
parent roadmap's INC-07 entry, which remains the canonical reference point
for this repo's current documentation structure.

## Closure rationale

Per `Axiom.SDD/AGENTS.md`'s closure rules, this increment (the
validator-reviewer step) and **INC-07 as a whole** are both set to
`Status: closed`, because all required conditions are met:

- Goal and expected behavior were clear at every step (audit → build →
  verify).
- Acceptance criteria existed and were independently checked off for all
  three steps, not self-certified.
- Changes were implemented (index-engineer step) and independently
  re-verified against the actual code (this step), not trusted from prior
  documents' prose.
- Available validation (`typecheck`, `build`, `test`) was executed
  independently, including a two-pass `git stash` bisection to isolate the
  true test delta.
- Review against intent and acceptance criteria was completed, with one
  discrepancy found (test-count bookkeeping, not a functional defect) and
  documented transparently rather than silently accepted.
- Stable knowledge integration into `general-spec.md` was assessed; no
  such file exists in this repo, stated explicitly, consistent with every
  prior increment in this chain.
- Results are documented clearly in this file and its two predecessors.

## INC-07 closure summary (whole increment)

**INC-07 — Reconcile derived caches / index rebuild** is `closed`.

**Delivered** (per the roadmap's originally proposed subagent sequence,
migration-engineer → index-engineer → validator-reviewer, followed exactly):

1. `listArtifacts(projectRoot, kind, specRelPath?)` — the direct-scan bulk
   enumeration primitive that did not exist anywhere before this
   increment (migration-engineer's Finding 2), in
   `Axiom/packages/workflow/src/artifact-store.ts`.
2. `list` subcommands on `axiom-increment`, `axiom-bug`, `axiom-plan`
   (three kind-scoped subcommands, per the increment's own explicit
   instruction, not a unified `artifact list --kind` command).
3. `axiom index rebuild` and `axiom index validate` — cache-less, direct-
   scan CLI commands closing the shipped `AGENTS.md` boilerplate
   inconsistency (migration-engineer's Finding 1).
4. `@axiom/doctor`'s `IX-001` check (`artifact-index` category) — closing
   the doctor-coverage gap over the artifact-store surface
   (migration-engineer's Finding 3).

**Deferred items, named explicitly** (not silently dropped):

- **The actual `.axiom/cache/*.index.json` persistent cache file** — the
  literal on-disk cache format named in the source decision documents'
  §5.1-§5.4. Deferred per the migration-engineer audit's two-tier
  recommendation until either (a) a project has enough artifact instances
  that the direct-scan `listArtifacts` is empirically slow, or (b) a
  concrete consumer needs the cache for programmatic lookups the `list`
  output alone can't serve (the audit specifically named INC-14's future
  `get_implementation_context` MCP tool as the most plausible trigger).
  Building this now would be speculative infrastructure per
  `Axiom.SDD/AGENTS.md`'s explicit bootstrap limits — no volume problem
  and no consumer exist today to justify it.
- **`search.index`, `graph.index`, `state.index`** (source doc §5.2) —
  no concrete consumer or source data (no search index, no graph/state-
  transition-history concept) exists anywhere in `Axiom` today to derive
  these from. Out of scope until a concrete consumer appears.
- **Curated/versioned indexes** (`technical-context.index`, `skills.index`,
  `commands.index`, `rules.index`, `roles.index`, source doc §5.5) — a
  different concept from derived caches ("si decide comportamiento del
  agente, se versiona"). Explicitly belongs to INC-09/INC-10, not INC-07.
- **Any remaining source-doc-6.1 command name not covered by `rebuild`/
  `validate`** — the audit's scope review of §6.1 named `rebuild`,
  `repair`, and `diff` as the documented command surface alongside
  `validate`. `axiom index repair` and `axiom index diff` were not
  requested by either INC-07 spec's acceptance criteria and were not
  built; they remain unimplemented and are not promised anywhere in the
  currently-shipped `AGENTS.md` boilerplate (which only names `validate`
  and `rebuild`), so there is no shipped-documentation/behavior mismatch
  for them today. If a future increment's audit finds a concrete need
  (e.g. `AGENTS.md` boilerplate is later updated to mention them), that
  would be new, separately-scoped work, not a gap in INC-07's own closure.

## Next step recommendation

**INC-08 — Reconcile `axiom doctor` extension + write-scope validation**,
per the parent roadmap (`Axiom.Spec/specs/increments/
INC-20260702-axiom-redesign-roadmap/README.md`, Phase C), whose stated
dependencies (`INC-06, INC-07`) are now both satisfied. Concrete brief per
the roadmap's own text: audit `@axiom/doctor/src/checks.ts` (now including
the `IX-001`/`artifact-index` category added by this increment) to confirm
no `allowedWriteScope`-equivalent check exists yet, then add a new
`WS-00x` check category validating `allowedWriteScope` (already present on
`PlanMetadata` per INC-06's schema, addendum section 9) following the same
`pass/fail/warn/skip` + evidence convention this increment reused for
`IX-001`. Subagent sequence per the roadmap: migration-engineer →
schema-writer → cli-implementer/validator-reviewer.
