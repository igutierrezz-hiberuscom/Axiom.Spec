# Increment: Reconcile bootstrap-from-code — validator-reviewer pass

Status: closed
Date: 2026-07-02

## Goal

Independently verify the `bootstrap-implementer` role's implementation
(`Axiom.Spec/specs/increments/INC-20260702-bootstrap-from-code-reconcile-impl/README.md`)
against the migration-engineer audit's acceptance criteria
(`Axiom.Spec/specs/increments/INC-20260702-bootstrap-from-code-reconcile/README.md`),
by directly reading diffs/code (not re-trusting claims), independently
running an end-to-end smoke test of `axiom bootstrap from-code --level
minimal`, independently investigating the implementer's flagged
Windows `EPERM` rename-race claim, and independently re-running the full
validation suite. This is the final role of **INC-18** of the parent
roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
Phase F).

## Context

Read both prior specs in full before this pass. Key facts carried
forward:

- `writeGuardedFile`/`resolveGuardedPath`
  (`Axiom/packages/document-bootstrap/src/guarded-write.ts`, new)
  extract the path-guard + atomic-write primitive out of
  `writeCopilotInstructions` (`writer.ts`, refactored to call the
  extracted helpers).
- `TechnicalContextIndex` gained an additive `status?: 'draft' |
  'reviewed'` field (`Axiom/packages/technical-context/src/
  technical-context-index.ts`).
- New `Axiom/apps/cli/src/bootstrap-from-code/{analyzer,drafter,
  index-builder}.ts` + `Axiom/apps/cli/src/commands/bootstrap.ts`
  implement `axiom bootstrap from-code --level minimal|basic [--role
  <role>]`.
- Draft markers use a distinct `AXIOM:DRAFT` banner (not
  `document-bootstrap`'s `AXIOM:GENERATED`/`TEAM:CUSTOM`).
- The implementer reported a transient Windows `EPERM` rename race in an
  unrelated `packages/versioning/tests/checkpoints.test.ts`, and the
  already-tracked `packages/cli-commands` dist build defect (both
  claimed unrelated/pre-existing).
- Repository note: `Axiom` is a git repository with substantial
  **pre-existing, uncommitted** working-tree changes unrelated to INC-18
  (e.g. `packages/doctor`, `packages/adapters/*`, `apps/cli/src/commands/
  axiom-{bug,increment,plan}.ts`, `tui.ts`) predating this increment
  chain. This validator pass scoped its diff review to the files the
  bootstrap-implementer's own spec listed as changed
  (`document-bootstrap`, `technical-context`,
  `apps/cli/src/bootstrap-from-code/*`, `apps/cli/src/commands/
  bootstrap.ts`), confirmed via `git diff --stat` against each
  individually, not the full uncommitted working tree.

## Scope

- Read the `writer.ts` diff directly (`git diff`) and confirm the
  refactor's actual behavior against every real call site and every
  pre-existing test, not just the "zero behavior change" claim.
- Trace `resolveGuardedPath`'s escape-rejection logic directly (not just
  its test names) against `../../etc/passwd`-style attacks.
- Confirm `TechnicalContextIndex.status?` is genuinely additive by
  reading the validator's exact branch conditions and running its test
  suite plus its known callers (`@axiom/mcp-tools`, `@axiom/doctor`).
- Read `analyzer.ts`/`drafter.ts`/`index-builder.ts` in full; confirm the
  analysis is mechanical, every draft gets the `AXIOM:DRAFT` banner,
  `available[].path` entries resolve (by direct path arithmetic and by a
  real end-to-end run) to actually-written files, and
  `mandatory.always`/`mandatory.whenTags` are always empty.
- Run `axiom bootstrap from-code --level minimal` end-to-end via a
  temporary vitest-based smoke test against a real subdirectory of this
  monorepo (`packages/document-bootstrap`), since the built CLI
  (`dist/`) is confirmed broken independent of this increment.
- Independently re-run `packages/versioning/tests/checkpoints.test.ts`
  multiple times in isolation, and the full suite multiple times, to
  assess the implementer's `EPERM` claim rather than accepting a single
  observation.
- Run `npm run typecheck`, `npm run build`, `npm test` (full monorepo)
  fresh and compare against both the stated pre-INC-18 baseline (12
  failed files / 13 failed tests / 1513 passed) and the implementer's
  claimed post-implementation result (1559 passed).
- Assess INC-18 as a whole against its acceptance criteria and close it
  if satisfied, naming deferred items explicitly.
- Review/refine (not duplicate) the "Bootstrap-from-code (Level 0/1)"
  section already added to `Axiom.Spec/general-spec.md` by the
  implementer.

## Non-goals

- No new feature work, no Level 2+ implementation, no `axiom bootstrap
  promote` tooling — all explicitly deferred per the audit/implementer
  specs, re-confirmed as still out of scope here.
- No fix to the pre-existing, unrelated `packages/cli-commands`
  dist-build defect or the `packages/doctor`/`resolveProject`
  cwd-sensitive test — both confirmed pre-existing and out of this
  increment's scope.
- No attempt to resolve or commit the large body of unrelated
  pre-existing uncommitted changes found in the `Axiom` working tree
  (adapters, TUI, axiom-bug/increment/plan commands) — explicitly out of
  scope for this validator pass; not created by INC-18.
- Via B (`bootstrap-from-legacy-sdd`) remains INC-19, not started here.

## Acceptance criteria

- [x] `writer.ts`'s diff independently read; refactor confirmed
      byte-identical for every path actually exercised by its one real
      caller and its full pre-existing test suite (4/4 passing
      unchanged), with one latent (currently-unexercised) semantic
      difference documented for a hypothetical relative-`targetPath`
      caller.
- [x] Path-guard escape rejection independently traced and confirmed
      against `../../etc/passwd`-style and absolute-outside-root
      attacks; one minor false-positive edge case found and documented
      (safe direction, not a security defect).
- [x] `TechnicalContextIndex.status?` confirmed additive by reading the
      validator's exact conditional branches; existing 29-test suite
      passes; known callers (`@axiom/mcp-tools`, `@axiom/doctor`)
      confirmed to read the index structurally, not exhaustively.
- [x] `analyzer.ts`/`drafter.ts`/`index-builder.ts` read in full;
      analysis confirmed mechanical (fs existence checks, JSON parsing,
      no fabricated content — `buildSummaryParagraph` explicitly
      disclaims architectural/business understanding); `AXIOM:DRAFT`
      banner confirmed unconditional across both levels; referential
      integrity confirmed by direct path arithmetic and by an end-to-end
      run; `mandatory.*` confirmed hardcoded to empty arrays.
- [x] End-to-end smoke test executed and passed: `runBootstrapFromCode`
      against a real subdirectory of this monorepo, output validated
      through the real `validateTechnicalContextIndex` (not just
      `yaml.load`), referential integrity confirmed against the
      filesystem.
- [x] `EPERM` claim independently investigated: isolated re-runs (4x) of
      `checkpoints.test.ts` all pass; 3 consecutive full-suite runs all
      land at the same 12-failed-file/13-failed-test baseline with zero
      EPERM occurrences; confirmed no code/dependency relationship
      between `packages/versioning` and any file INC-18 touched.
- [x] Real validation independently executed: `typecheck` (clean),
      `build` (clean), `test` (full monorepo, 3 runs, consistently 1559
      passed / 13 failed / 12 failed files, matching the implementer's
      claim).
- [x] INC-18 (audit + impl + validator) assessed as a whole against its
      acceptance criteria and closed, with deferred items named
      explicitly.
- [x] `Axiom.Spec/general-spec.md`'s "Bootstrap-from-code (Level 0/1)"
      section reviewed and refined (role-chain status line updated;
      validator findings appended) — not duplicated.

## Open questions

None blocking. All three of the audit's open questions
(Q-bootstrap-1/2/3) were already resolved by the bootstrap-implementer
with documented rationale; this pass found no reason to reopen them.

## Assumptions

- "Independently" in this role's brief means re-deriving each claim from
  the actual source/diff/test run, not re-reading the prior specs'
  prose and restating it — followed literally throughout this pass (see
  "Implementation notes" below for the concrete method used for each
  claim).
- The `Axiom` repository's large body of unrelated pre-existing
  uncommitted changes (confirmed via `git status`/`git diff --stat`) is
  out of this increment's remit; this pass verified INC-18's own diff in
  isolation by checking each touched file's diff individually rather
  than assuming the full working-tree diff represents INC-18's scope.

## Implementation notes

No source code was changed by this pass (validator-reviewer is a
read/verify role per the audit's collapsed-role recommendation). One
temporary vitest smoke-test file
(`apps/cli/tests/bootstrap-smoke-INC18-validator.test.ts`) was created,
run, and then deleted — it is not part of the shipped diff. This pass
also made two small clarifying edits to `Axiom.Spec/general-spec.md`
(see "General spec integration" below); no `Axiom.SDD` or `Axiom` files
were modified.

### 1. `writer.ts` refactor — independent diff read

`git diff -- packages/document-bootstrap/src/writer.ts` was read
directly (not inferred from the implementer's prose). Confirmed:

- The path-guard block (previously inlined `path.resolve` + `path.relative`
  + escape check) was replaced by a call to `resolveGuardedPath`, and the
  atomic-write block (previously inlined `mkdirSync`/`writeFileSync(.tmp)`/
  `renameSync`) was replaced by a call to `writeGuardedFile` — same order
  of operations, same error `kind`s (`path-escape`, `io-error`), same
  return shape.
- **Found a real, previously-unflagged behavioral difference**: before,
  `opts.targetPath` (when provided) was resolved via `path.resolve(opts.targetPath)`
  — i.e. a *relative* override would resolve against `process.cwd()`.
  After, `targetRel = opts.targetPath ?? DEFAULT_TARGET_REL` is passed to
  `resolveGuardedPath(projectRoot, targetRel)`, which resolves via
  `path.resolve(projectRoot, targetRel)` — a relative override now
  resolves against `projectRoot` instead. Verified with a direct Node
  repro (`path.resolve('rel/path')` vs `path.resolve(projectRoot,
  'rel/path')` produce different absolute paths for the same relative
  input).
- **This difference is dead code today, not a live regression**: grepped
  every call site and every test. The one production caller
  (`apps/cli/src/commands/configure.ts`) never sets `targetPath` (always
  uses the default). Every pre-existing test in `writer.test.ts` either
  omits `targetPath` or passes an **absolute** one
  (`path.join(tmpDir, '..', 'escape', 'copilot-instructions.md')`, itself
  absolute since `tmpDir` is absolute) — and for absolute inputs,
  `path.resolve(x)` and `path.resolve(projectRoot, x)` are provably
  identical (confirmed with a direct repro). So the "byte-identical
  behavior" claim holds for every path actually exercised in this
  codebase today; the difference only matters for a hypothetical future
  caller passing a relative `targetPath` override, which does not exist.
  Documented in `general-spec.md` for future awareness rather than left
  silently undocumented.
- Ran the pre-existing `writer.test.ts` suite standalone:
  `npx vitest run packages/document-bootstrap/tests/writer.test.ts` ->
  **4/4 passed**, confirming no observable regression for any exercised
  path.

### 2. Path-guard escape rejection — independent trace

Read `guarded-write.ts`'s `resolveGuardedPath` directly and reproduced
its logic in an isolated Node script against several inputs (not just
reading the test file's assertions):

| Input (`relativePath`) | Result |
|---|---|
| `../../etc/passwd` | rejected (`path-escape`) |
| `docs/a.md` | accepted |
| `/etc/passwd` (absolute, outside root) | rejected (`path-escape`) |
| `./../../etc/passwd` | rejected (`path-escape`) |
| `` (empty, i.e. the root itself) | rejected (`path-escape`) |
| `..evilfile.md` (legit filename starting with literal `..`, no actual escape) | **rejected — false positive** |

Confirmed the actual escape-rejection requirement holds: every path that
genuinely resolves outside `projectRoot` (via `..` traversal or an
absolute path elsewhere) is rejected before any I/O runs (`resolveGuardedPath`
is called before `mkdirSync`/`writeFileSync` in `writeGuardedFile`). Found
one minor imprecision: `relative.startsWith('..')` also rejects a
legitimate filename that happens to start with the literal characters
`..` (e.g. `..evilfile.md`) even though it does not escape `projectRoot`
— a false positive, not a false negative. This is safe (over-rejects,
never under-rejects) and not a security defect, but is not fully
precise; documented in `general-spec.md` as a known minor edge case, not
a blocking finding (no real caller in this codebase generates filenames
starting with literal `..`).

### 3. `TechnicalContextIndex.status?` — independent additive confirmation

Read `technical-context-index.ts` in full. Confirmed:

- `validateTechnicalContextIndex` only validates `status` when
  `parsed['status'] !== undefined` (line ~388-394); the `ok` return only
  spreads `status` into the result when present (line ~440-442) — the
  exact same conditional-presence pattern already used for the
  pre-existing `projectId?` field one block above. An index YAML that
  omits `status` parses identically to before this change.
- Ran `npx vitest run packages/technical-context` -> **29/29 passed**,
  including the three new tests added for `status` (absent/draft/invalid
  enum value).
- Grepped every consumer of `TechnicalContextIndex` in the codebase
  (`@axiom/mcp-tools/technical-context-handlers.ts`,
  `@axiom/doctor/checks.ts`) and confirmed by reading them that neither
  destructures/enumerates the interface's fields exhaustively — both
  read structurally (`index.mandatory`, `index.available`, etc.), so
  neither is affected by an additional optional field. Ran
  `npx vitest run packages/mcp-tools/tests/technical-context-handlers.test.ts`
  -> passed. (`packages/doctor/tests/checks.test.ts` had one unrelated,
  pre-existing cwd-sensitive failure — see "Unrelated failure noted
  during scoped runs" below — not caused by this field.)
- `Axiom/apps/cli/src/bootstrap-from-code/index-builder.ts`'s
  `buildTechnicalContextIndex` was confirmed to always set
  `status: 'draft'` (hardcoded literal, not conditional) on every
  generated index.

### 4. `analyzer.ts`/`drafter.ts`/`index-builder.ts` — independent review

Read all three files in full.

- **Mechanical analysis confirmed**: `detectStacks` only checks
  `fs.existsSync` on known manifest filenames and reads `package.json`
  for a `name`/`scripts` field verbatim (no interpretation);
  `buildRepoMap` lists top-level directory names only, mapping a small
  literal table of folder-name conventions to a `purposeGuess` (no
  recursion, no semantic inference); `detectCommands` copies
  `package.json#scripts` entries verbatim. `drafter.ts`'s
  `buildSummaryParagraph` assembles a sentence directly from these facts
  and explicitly states in its own output text that "this summary...
  does not reflect architectural or business understanding of the
  code" — confirming no fabricated claims about the analyzed codebase.
- **`AXIOM:DRAFT` banner confirmed unconditional**: every one of the 4
  documents (`overview`, `repo-map`, `stack`, `commands`) is prefixed
  with `AXIOM_DRAFT_MARKER` at both the `minimal` and `basic` level
  branches in `draftDocuments` — verified by reading every branch, not
  sampling one.
- **Referential integrity confirmed two ways**: (a) direct path
  arithmetic — reproduced in Node that `buildTechnicalContextIndex`'s
  `available[].path` (`'..' + '/' + doc.relativePath`, e.g.
  `../repo/overview.md`), when resolved relative to the index file's own
  directory (`<specScope>/technical-context/indexes/`), lands exactly on
  the same absolute path the document was written to
  (`<specScope>/technical-context/repo/overview.md`); (b) the end-to-end
  smoke test (see below) round-trips the generated index through the
  real `validateTechnicalContextIndex` and asserts every `available[].path`
  resolves to a file that is also present in `writtenDocumentPaths`.
- **`mandatory.*` confirmed empty**: `buildTechnicalContextIndex` hardcodes
  `mandatory: { always: [], whenTags: [] }` — a literal, not a computed
  value that could accidentally become non-empty.

### 5. End-to-end smoke test

The built CLI was confirmed independently broken, matching the
implementer's own note:

```
node apps/cli/dist/index.js bootstrap from-code --level minimal
-> Error: Cannot find module './_shared'
   (dist/commands/init.js, required eagerly at CLI startup)
```

This reproduces `general-spec.md`'s already-tracked "Known build-tooling
defect" (`packages/cli-commands`'s `tsconfig.json` cross-including
`apps/cli` sources) — confirmed unrelated to the bootstrap command
specifically (it breaks at CLI startup via `init.js`, before any command
dispatch, for effectively every command).

Per the implementer's own note that this doesn't affect vitest-based
tests, a temporary smoke-test file was written and run
(`apps/cli/tests/bootstrap-smoke-INC18-validator.test.ts`, deleted after
this pass — not part of the shipped diff) that:

- Ran `runBootstrapFromCode` with `repoPath` pointed at a **real
  subdirectory of this monorepo** (`packages/document-bootstrap`, which
  has a real `package.json`, `src/`, `tests/`, `dist/` — not a synthetic
  fixture), scoped `cwd` to a scratch project directory with a minimal
  `axiom.yaml` + spec scope.
- Asserted `exitCode === 0`, 4 documents written, every document exists
  on disk and contains `AXIOM:DRAFT`.
- Parsed the generated index YAML and ran it through the **real**
  `validateTechnicalContextIndex` (not just `yaml.load`) — confirmed
  `kind: 'ok'`, `status: 'draft'`, both `mandatory` arrays empty.
- Resolved every `available[].path` relative to the index file's own
  directory and confirmed each resolves to a file that both exists on
  disk and is present in `writtenDocumentPaths` — closing the
  referential-integrity loop end-to-end, not just in isolated unit
  tests.

Result: **passed** (`1 passed (1)`). This confirms the command works
end-to-end against a real codebase, beyond the implementer's own
synthetic-fixture unit/happy-path tests.

### 6. `EPERM` rename-race investigation

Independently investigated rather than accepting the implementer's
single observation, per this brief's explicit instruction (citing
INC-15's precedent where a similar claim turned out to be a real
regression):

- Confirmed via `grep` that `packages/versioning` has **zero**
  references to `document-bootstrap`, `technical-context`, or any
  `bootstrap-from-code`/`commands/bootstrap.ts` file — no code or
  dependency relationship exists between the two areas. The race (in
  `packages/versioning/src/checkpoints.ts`'s own `fs.promises.rename`
  call) is structurally unrelated to `guarded-write.ts`'s separate
  atomic-write implementation.
- Ran `packages/versioning/tests/checkpoints.test.ts` in isolation **4
  times** — 9/9 passed every time, no EPERM observed.
- Ran the **full monorepo suite 3 times** in immediate succession. All
  three runs landed at the exact same result: **12 failed files / 13
  failed tests / 1559 passed** — matching the implementer's
  post-implementation claim exactly, with **zero EPERM occurrences**
  across all three runs, and zero occurrences of `checkpoints.test.ts`
  in any of the three failing-file lists.
- Listed the 13 failing tests from a fresh run by name; confirmed none
  belong to `document-bootstrap`, `technical-context`, or any
  `bootstrap-from-code`/`bootstrap.test.ts` suite — they span
  `agents`, `skills`, `model-routing`, `doctor`, `project-resolution`,
  `telemetry`, `toolchain`, `tui`, and `apps/cli/tests/start.test.ts`,
  consistent with pre-existing, filesystem/cwd-state-sensitive
  integration tests unrelated to this increment.
- **Conclusion**: could not force-reproduce the EPERM race in 3
  full-suite attempts (expected for a genuinely non-deterministic
  Windows filesystem/antivirus-contention race — absence of reproduction
  is not proof of absence, but 3 clean runs plus zero code/dependency
  relationship is meaningfully stronger evidence than the implementer's
  single observation). Unlike INC-15's precedent, this pass found no
  evidence the claim is masking a real regression: the failure category
  (Windows `rename` under load) is a known class of flake distinct in
  kind from a logic regression, the affected file has no relationship to
  anything INC-18 touched, and repeated runs are stable. Treated as
  confirmed non-reproducible/unrelated, not dismissed without
  investigation.

### 7. Full validation — independent re-run

Followed the same validation discovery order as the prior specs
(`Axiom/package.json` scripts: `build`, `test`, `typecheck`).

```
npm run typecheck    -> clean (tsc -b, no output = success)
npm run build        -> clean (tsc -b, no output = success)
npm test  (run 1)     -> Test Files: 12 failed | 144 passed (156)
                         Tests:      13 failed | 1559 passed (1572)
npm test  (run 2)     -> Test Files: 12 failed | 144 passed (156)  (same)
npm test  (run 3)     -> Test Files: 12 failed | 144 passed (156)  (same)
```

Matches the implementer's claimed post-implementation result exactly,
independently, across 3 runs (not a single run). Also independently
re-ran every suite the implementer listed as new/touched:

```
npx vitest run packages/document-bootstrap                          -> 5 files, 42 tests passed
npx vitest run packages/technical-context                           -> 1 file, 29 tests passed
npx vitest run apps/cli/tests/bootstrap-from-code apps/cli/tests/bootstrap.test.ts
                                                                       -> 4 files, 36 tests passed
```

All match the implementer's own scoped-run claim (107 tests across 10
files).

**Unrelated failure noted during scoped runs**: running
`packages/doctor/tests/checks.test.ts` in isolation (scoped cwd, not the
full monorepo run) surfaced one failure
(`resolveProject(...).status` returned `'ambiguous'` instead of
`'resolved'|'not-found'`). Confirmed via `git diff --stat` that
`packages/doctor` has pre-existing uncommitted changes predating INC-18
(last commit touching `checks.ts` is `4ba8abe`, an earlier, unrelated
commit), and the failure is cwd-resolution-sensitive (does not
reproduce in the full-suite run, which uses the monorepo root as cwd).
Confirmed out of scope: no file this increment touched is referenced by
`packages/doctor`'s failing test.

## Result

Independently re-verified every claim in the bootstrap-implementer's
spec by reading diffs/code directly and running real commands, rather
than re-trusting prose:

1. `writer.ts`'s refactor is confirmed byte-identical for every path
   actually exercised in this codebase (production caller + full
   pre-existing test suite, 4/4 passing unchanged) — with one
   previously-undocumented, currently-dead-code semantic difference for
   a hypothetical relative-`targetPath` caller, now documented.
2. The path-guard genuinely rejects `../`-style and absolute
   escape attempts (traced directly, reproduced in isolation) — with one
   minor, safe-direction false-positive edge case found and documented.
3. `TechnicalContextIndex.status?` is confirmed genuinely additive by
   reading the exact validator branches and confirming no existing
   caller enumerates fields exhaustively; `bootstrap-from-code` always
   sets `status: 'draft'`.
4. `analyzer.ts`/`drafter.ts`/`index-builder.ts` are confirmed mechanical
   (no fabricated codebase claims), every draft carries `AXIOM:DRAFT`,
   referential integrity is confirmed by both direct arithmetic and an
   end-to-end run, and `mandatory.*` is confirmed always empty.
5. An end-to-end smoke test against a real monorepo subdirectory passed,
   validated through the real schema validator, confirming the command
   works beyond synthetic unit fixtures.
6. The `EPERM` rename race was independently investigated (not accepted
   on a single observation): confirmed structurally unrelated code, 4/4
   isolated passes, and 3/3 clean full-suite runs with zero EPERM
   occurrences — treated as confirmed non-reproducible/unrelated per
   this pass's own evidence, not the implementer's claim alone.
7. Full validation independently re-run 3 times:
   `typecheck`/`build` clean; `test` consistently 12 failed files / 13
   failed tests / 1559 passed, matching the implementer's claim exactly.

All of the audit's and implementer's acceptance criteria are
independently confirmed satisfied. No regressions, no fabricated claims,
no referential-integrity gaps found in the shipped output. Two minor,
non-blocking findings were documented (relative-`targetPath` dead-code
semantic change; `..`-prefixed-filename false-positive in the path
guard) for future awareness, neither rising to a defect requiring rework
of this increment's scope.

**INC-18 as a whole is closed** (see "General spec integration" below
and the closure rationale): all three roles (migration-engineer audit ->
bootstrap-implementer -> validator-reviewer) completed, acceptance
criteria met at each stage, real validation executed and independently
reconfirmed, and stable knowledge integrated into `general-spec.md`.

### INC-18 closure summary (audit + impl + validator, all three roles)

- **migration-engineer audit**: confirmed `writer.ts`'s path-guard +
  atomic-write primitive is reusable (as an extracted helper, not via
  `writeCopilotInstructions` directly); confirmed
  `TechnicalContextIndex` fitness; scoped exhaustiveness to Level 0/1
  only; collapsed the roadmap's 4-role sequence to 2
  (`bootstrap-implementer` + `validator-reviewer`); designed the CLI
  surface. Audit-only, no code changed.
- **bootstrap-implementer**: extracted `writeGuardedFile`/
  `resolveGuardedPath`; refactored `writeCopilotInstructions` to use
  them; added the additive `TechnicalContextIndex.status?` field;
  implemented mechanical Level 0/1 analysis, drafting (with
  `AXIOM:DRAFT` banner), and index-building (available-only, status:
  draft); shipped `axiom bootstrap from-code --level minimal|basic
  [--role <role>]`; resolved all three open questions
  (Q-bootstrap-1/2/3) with documented rationale; ran real validation.
- **validator-reviewer** (this spec): independently re-verified every
  claim above by reading diffs/code directly and running real commands
  (including a real end-to-end smoke test and a multi-run EPERM
  investigation), found two minor non-blocking issues, confirmed no
  regressions, and closed INC-18.

**Deferred items, named explicitly** (not implemented in INC-18, not
regressions — intentional scope boundaries per the audit's own
reasoning):

- **Level 2 (Standard), Level 3 (Deep), Level 4 (Exhaustive)
  exhaustiveness** — require genuine code-semantic/architectural
  understanding, explicitly deferred to a future increment if/when
  needed.
- **`mandatory.*` promotion tooling** (e.g. a possible `axiom bootstrap
  promote` or manual-edit workflow to flip `status: 'draft'` ->
  `'reviewed'` or populate `mandatory.always`/`mandatory.whenTags`) —
  deliberately not built; this is a human curation step by design
  (Q-bootstrap-1), not a mechanical gap.
- **`--dry-run` flag** — noted as a reasonable near-term follow-up by the
  implementer, not required by any concrete need yet (YAGNI).
- **Topology-based role/repo-kind auto-detection** — Level 0/1 uses an
  explicit `--role` flag instead of `@axiom/topology`'s
  `roleCodeRepositories`; deferred as materially larger, unnecessary
  work for this scope.
- **`canonical-agents-md.ts`'s inline path-guard/atomic-write
  duplicate** — not refactored to use the newly extracted
  `writeGuardedFile`, per "do not modify unrelated files"; flagged as a
  legitimate future cleanup.
- **Referential-integrity enforcement in the loader/validator itself**
  (`loadTechnicalContextIndex`/`validateTechnicalContextIndex` still do
  not check `path` existence on disk) — INC-18's own output avoids the
  gap by construction; the underlying gap in `@axiom/technical-context`
  (pre-existing since INC-10) remains unfixed.
- **Via B (`bootstrap-from-legacy-sdd`)** — explicitly out of scope for
  INC-18; this is **INC-19** in the parent roadmap, not started.
- **The two minor findings from this validator pass** (relative-
  `targetPath` dead-code semantic change in the writer refactor; `..`
  -prefixed-filename false-positive in the path guard) — both
  documented, neither blocking, neither has a live caller today.
- **Pre-existing, unrelated issues re-confirmed but not fixed** (out of
  INC-18's scope by definition): the `packages/cli-commands` dist-build
  defect (breaks the built CLI broadly, tracked in `general-spec.md`
  since before this increment); the `packages/doctor`/`resolveProject`
  cwd-sensitivity test failure; the pre-existing 12-failed-file test
  baseline (unrelated snapshot/timing/integration issues spanning
  `agents`, `skills`, `model-routing`, `telemetry`, `toolchain`, `tui`,
  `apps/cli/tests/start.test.ts`).

## Validation

Validation discovery order followed (`README` -> package scripts):
`Axiom/package.json` scripts `build` (`tsc -b`), `test` (`vitest run`),
`typecheck` (`tsc -b`) — same commands the prior two specs used, run
fresh, independently, multiple times where reproducibility mattered
(EPERM investigation, full-suite baseline).

```
npm run typecheck    -> clean
npm run build        -> clean
npm test  x3          -> Test Files: 12 failed | 144 passed (156) (all 3 runs identical)
                         Tests:      13 failed | 1559 passed (1572) (all 3 runs identical)
```

Scoped suites (all passing):

```
npx vitest run packages/document-bootstrap/tests/writer.test.ts     -> 4/4 (pre-existing suite, unmodified)
npx vitest run packages/document-bootstrap                          -> 5 files, 42 tests
npx vitest run packages/technical-context                           -> 1 file, 29 tests
npx vitest run apps/cli/tests/bootstrap-from-code apps/cli/tests/bootstrap.test.ts
                                                                       -> 4 files, 36 tests
npx vitest run packages/versioning/tests/checkpoints.test.ts (x4)    -> 9/9 every run, no EPERM
```

Temporary end-to-end smoke test (created, run, deleted; not part of the
shipped diff): `apps/cli/tests/bootstrap-smoke-INC18-validator.test.ts`
-> 1/1 passed, exercising `runBootstrapFromCode` against a real
monorepo subdirectory with the real `validateTechnicalContextIndex`.

Manual reproduction of the known dist-build defect (informational, not
part of pass/fail counts): `node apps/cli/dist/index.js bootstrap
from-code --level minimal` -> `Cannot find module './_shared'`,
confirming the implementer's claim independently.

## General spec integration

Reviewed the existing "Bootstrap-from-code (Level 0/1)" section in
`Axiom.Spec/general-spec.md` (added by the bootstrap-implementer).
Confirmed every factual claim in it against the code/diffs directly
during this pass; found it accurate. Made two small edits rather than
duplicating the section:

1. Updated the role-chain status line from "`validator-reviewer` pending
   as of this writing" to "all three roles closed," reflecting this
   pass's closure.
2. Appended a "Validator-reviewer findings" bullet documenting the two
   minor, non-blocking discoveries from this pass (the relative-
   `targetPath` dead-code semantic change in the `writer.ts` refactor,
   and the `..`-prefixed-filename false-positive in the path guard) —
   genuinely new, independently-discovered knowledge not present in
   either prior spec, appropriate for the consolidated spec per
   `AGENTS.md`'s documentation rules.

No other `general-spec.md` sections required changes.

## Next step recommendation

INC-18 is closed. Per the parent roadmap
(`INC-20260702-axiom-redesign-roadmap/README.md`, Phase F), the next
increment in sequence is **INC-19 — Reconcile bootstrap-from-legacy-SDD**
(Via B: migrate an existing legacy-SDD project's specs into Axiom
format), with subagent sequence `bootstrap-analyzer
(legacy-spec-reader) -> migration-engineer (increment-migrator,
bug-migrator, plan-migrator) -> docs-skills-writer (context-normalizer,
adr-extractor) -> cli-implementer (axiom-format-writer, routed through
Axiom commands) -> validator-reviewer`, depending on INC-06, INC-09/
INC-10, and INC-11 (all already closed per the roadmap). Recommend
starting with a migration-engineer/bootstrap-analyzer audit pass (mirror
of this INC-18 chain's own first step) before any implementation, since
no legacy-SDD migrator exists in the codebase today per the roadmap's
own audit note.
