# Increment: Bootstrap-from-legacy-sdd — validator-reviewer

Status: closed
Date: 2026-07-03

## Goal

Independently review and validate `axiom bootstrap from-legacy-sdd
<path> [--dry-run]` (implemented by the `cli-implementer` role at
`Axiom.Spec/specs/increments/INC-20260702-bootstrap-legacy-sdd-reconcile-impl/README.md`),
by directly reading code (not re-trusting claims), independently
running an end-to-end smoke test against a realistic fixture, and
independently re-running the full validation suite. This is the final
role of **INC-19** of the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
Phase F), following the `migration-engineer` audit and the
`cli-implementer` implementation.

## Context

Read both prior specs in full before reviewing:

- `Axiom.Spec/specs/increments/INC-20260702-bootstrap-legacy-sdd-reconcile/README.md`
  (migration-engineer audit: input-format assumption, reusability
  confirmation, collapsed-role recommendation, CLI surface design,
  Q-legacy-1..4 recommendations).
- `Axiom.Spec/specs/increments/INC-20260702-bootstrap-legacy-sdd-reconcile-impl/README.md`
  (cli-implementer: scanner/migrator implementation, Q-legacy-1..4
  resolutions, and one explicitly flagged judgment call — bypassing
  `axiom-increment.ts`'s `runIncrementSubcommand` in favor of calling
  artifact-store primitives directly for increment/bug migration).

Implementation files reviewed directly: `Axiom/apps/cli/src/
bootstrap-from-legacy-sdd/{scanner,migrator}.ts`, `Axiom/apps/cli/src/
commands/bootstrap.ts` (the `from-legacy-sdd` additions),
`Axiom/apps/cli/src/commands/axiom-increment.ts` (`runIncrementSubcommand`,
`syncIncrementMetadata`), `Axiom/packages/workflow/src/{state-store,
artifact-store}.ts`.

## Scope

- Independently review the bypass-of-`runIncrementSubcommand` judgment
  call by reading `runIncrementSubcommand` and the state-store/
  artifact-store design comments directly.
- Independently trace the scanner's folder-vs-flat-file detection and
  status-extraction logic against concrete fixture cases.
- Independently verify the Q-legacy-1 tightening (WorkflowState-only
  vocabulary for increment/bug/plan) by reading the actual validation
  logic.
- Independently verify `--dry-run` performs zero filesystem writes by
  tracing the code path.
- Independently verify the collision-skip-and-report behavior (Q-legacy-4)
  reports clearly, never silently.
- Run a real end-to-end smoke test against a realistic legacy-repo
  fixture (folder-shape and flat-file-shape, multiple kinds, a
  misleading-content fixture for Q-legacy-2, a malformed entry, and a
  double-run collision-safety check).
- Re-run `npm run typecheck`, `npm run build`, `npm test` independently
  and compare against both the pre-INC-19 baseline and the
  implementer's reported post-implementation numbers.
- Assess INC-19 as a whole for closure, naming deferred items
  explicitly.
- Integrate a "Bootstrap-from-legacy-sdd" section into
  `Axiom.Spec/general-spec.md`.

## Non-goals

- No new implementation scope beyond what the audit/implementer already
  defined (content restructuring, ADR inference from prose, `--kind`
  filter flag — all explicitly deferred by the prior two roles, reaffirmed
  here, not revisited).
- No fix attempted for the pre-existing, unrelated `dist/` build defect
  or the 13 pre-existing failing tests (confirmed independently
  unrelated — see "Validation" below).
- No code changes to `scanner.ts`/`migrator.ts`/`bootstrap.ts` — this
  role found the implementation correct as-is (see "Result").

## Acceptance criteria

- [x] The `runIncrementSubcommand` bypass judgment call is independently
      reviewed with concrete evidence (not just re-trusting the
      implementer's reasoning) and either fixed or explicitly confirmed
      sound.
- [x] Scanner folder-vs-flat-file detection and status extraction are
      traced against at least 3 concrete fixture cases and confirmed
      correct.
- [x] The Q-legacy-1 `WorkflowState`-only tightening is confirmed
      correctly implemented by reading the actual code path (not the
      spec's description of it).
- [x] `--dry-run` is confirmed to perform zero filesystem writes by
      code-path tracing, not just by trusting existing tests.
- [x] Collision-skip-and-report (Q-legacy-4) is confirmed to report
      clearly to the user, never silently swallowed.
- [x] A real end-to-end smoke test was run against a realistic fixture
      (not just unit-level mocks) and passed.
- [x] `npm run typecheck`, `npm run build`, `npm test` were independently
      re-run; results compared against baseline and the implementer's
      claim.
- [x] INC-19 as a whole is assessed for closure, with deferred items
      named explicitly.
- [x] `general-spec.md` integration performed (or explicit no-op
      rationale given).

## Open questions

None outstanding — all four Q-legacy items were resolved by the
`cli-implementer` role and are independently re-confirmed correct below.

## Assumptions

Same input-format assumption as the audit and implementer (not
repeated here — see the audit spec's "Assumptions" section). This role
does not revisit or challenge that assumption; it validates the
implementation against it.

## Implementation notes

### 1. Bypass-of-`runIncrementSubcommand` judgment call — independently reviewed, CONFIRMED SOUND

Read `Axiom/apps/cli/src/commands/axiom-increment.ts`'s
`runIncrementSubcommand` (lines 314-480) and `syncIncrementMetadata`
(lines 493-519) directly, plus `Axiom/packages/workflow/src/
state-store.ts` and the header comment of `Axiom/packages/workflow/src/
artifact-store.ts` (lines 1-15), to determine whether the bypass causes
a migrated increment/bug to behave inconsistently with a
normally-created one later.

**Finding: the bypass is correct. `runIncrementSubcommand` does one
thing a migration must NOT replicate, and `metadata.yml` is the only
data a migrated artifact needs — confirmed by direct evidence, not
assumption.**

Concrete evidence:

- `artifact-store.ts`'s own header comment states the design explicitly:
  *"Este módulo es ADITIVO: no reemplaza `state-store.ts` (state machine
  runtime, `workflow-state.json`), que queda completamente sin
  modificar... `metadata.yml` es la identidad/metadata por-instancia
  (id, title, status, externalRefs, links); el state machine sigue
  operando per-workflow-ID, sin conocimiento de instancias."* This is
  INC-06's own original design, confirmed by reading the source file
  directly (not the task framing's paraphrase of it) — `metadata.yml`
  and `workflow-state.json` are two genuinely parallel, independent
  stores by design, not one layered on the other.
- `workflow-state.json` (`state-store.ts`, `WorkflowStateFile.workflows:
  Record<WorkflowId, WorkflowStateRecord>`) is keyed by `WorkflowId`
  (`'increment' | 'bug' | 'plan' | 'role' | 'qa-e2e'`) — **one singleton
  record per workflow TYPE, not one per artifact instance.** The file's
  own header comment states this design constraint outright: *"El
  caller (CLI) corre un solo command por invocación... el riesgo de race
  es despreciable"* (single-current-workflow assumption). This means
  `workflow-state.json` structurally CANNOT represent "N migrated
  increments" — there is exactly one `vars`/`state` slot for the whole
  `'increment'` workflow type at any time. Populating it during a bulk
  migration of, say, 20 legacy increments would require picking exactly
  one of them to occupy that global slot — silently overwriting/
  discarding the state for the other 19, or clobbering whatever real,
  human-driven increment is currently mid-workflow. This is not a
  missing feature; force-initializing `workflow-state.json` during
  migration would be actively wrong.
- `runIncrementSubcommand`'s `metadata.yml` sync path
  (`syncIncrementMetadata`, step 8.5) is driven by `vars.metadataId`, a
  value generated once during `create`/init (step 6.5) and cached inside
  the SAME singleton `workflow-state.json` record. A migrated artifact
  has no `vars.id`/`vars.slug` from a real `axiom increment create`
  invocation to seed that record with, and forcing one into existence
  per migrated item would multiply the exact singleton-collision problem
  above by N.
- What a migrated increment/bug DOES get via the direct-primitive path
  (`generateUniqueArtifactId` + `makeInitialIncrementOrBugMetadata` +
  `saveArtifactMetadata` + the migrator's own
  `ensureArtifactReadmeWithContent`) is a `metadata.yml` folder-per-
  artifact instance — the exact same shape and exact same write
  primitives `syncIncrementMetadata` itself calls
  (`makeInitialIncrementOrBugMetadata`/`saveArtifactMetadata`/
  `ensureArtifactReadme`, verified identical function names by reading
  both call sites side by side). This is a real, first-class
  `metadata.yml` instance, indistinguishable on disk from one created
  via `syncIncrementMetadata`'s own `create`/init branch.
- **Does a migrated increment later behave inconsistently under normal
  `axiom increment` commands?** No — because `axiom-increment.ts`'s 8
  subcommands (`create`/`refine`/`specify`/.../`archive`) operate
  exclusively on the SINGLETON `workflow-state.json` record for the
  `'increment'` workflow type, driven by `vars.id`/`vars.slug`, not by
  iterating or looking up `metadata.yml` instances by ID. A migrated
  artifact's `metadata.yml` sits alongside, unaffected, exactly like an
  ADR/Decision artifact today (which likewise never gets a
  `workflow-state.json` entry — `axiom-adr create`/`axiom-decision
  create` call the same direct-primitive path this migration uses, with
  zero `workflow-state.json` involvement, and no inconsistency has ever
  been reported for those two kinds). A human who later wants to drive a
  migrated increment through real workflow transitions would run `axiom
  increment create --id <x> --slug <y>` for a NEW singleton-workflow
  session and separately reconcile it with the migrated `metadata.yml`
  if desired — this is a pre-existing, orthogonal manual step, not
  something this migration could have automated correctly anyway (there
  is no legacy-side `--slug` equivalent to seed it with, as the
  implementer's spec already noted).
- **Conclusion**: the bypass does not skip anything a migrated artifact
  needs. It correctly avoids fabricating fictitious
  `workflow-state.json` state for artifacts that were never driven
  through the `increment`/`bug` workflow via real CLI commands, and
  avoids the singleton-collision hazard bulk migration would otherwise
  cause. No fix required. This CONFIRMS (does not merely accept) the
  implementer's own reasoning in the impl spec's "Deviation check"
  section, using direct evidence not cited there (`state-store.ts`'s
  singleton-key structure and its own header comment) in addition to the
  evidence the implementer already gave.

### 2. Scanner folder-vs-flat-file detection and status extraction — traced against 3+ concrete cases

Read `scanner.ts` directly (not the impl spec's description of it) and
traced these cases by hand against the source:

- **Folder-shape with parseable status**: a child that is a directory
  containing `README.md` with a `# heading` and a `Status: done` line.
  Trace: `scanLegacyRepo` sees `entry.isDirectory()` true (line 214),
  calls `findMarkdownInDirectory` (line 216) which finds `README.md` at
  line 161 (`fs.existsSync` + `isFile()` check) and returns its path
  without falling through to the `.md`-glob fallback. `extractLegacyTitle`
  matches the `# heading` (line 119). `extractLegacyStatus` matches
  `^Status:\s*(\w[\w-]*)` (line 109), lowercases `done`, maps via
  `STATUS_SYNONYMS['done'] = 'archived'` (line 74), confirms `'archived'`
  is in `KNOWN_STATUS_VALUES` (line 55), returns `'archived'`. Then in
  `migrator.ts`, `resolveWorkflowStateForMigration('archived')` checks
  `WORKFLOW_STATES.has('archived')` (line 69, present) and returns
  `'archived'` as the final `WorkflowState`. **Confirmed correct** —
  independently re-verified end-to-end by the smoke test below (the
  `INC-001-old-feature` fixture), which asserts an `archived`-status
  `metadata.yml` is actually written.
- **Flat-file-shape with unparseable status defaulting to draft**: a
  child that is a `.md` file directly under `bugs/` with a `# heading`
  but no `Status:` line at all. Trace: `entry.isFile() &&
  childName.toLowerCase().endsWith('.md')` is true (line 233),
  `sourceFilePath` is the child path directly (no directory search).
  `extractLegacyStatus`'s regex `exec` returns `null` (no match anywhere
  in the body), function returns `undefined` at line 110. In
  `migrator.ts`, `resolveWorkflowStateForMigration(undefined)` fails the
  `rawStatus !== undefined` check (line 83) and falls through to the
  literal `return 'draft'` (line 86). **Confirmed correct** —
  independently re-verified by the smoke test's flat-file bug fixture,
  which asserts `status === 'draft'` on the resulting `metadata.yml`.
- **Folder-name-only kind detection (Q-legacy-2)**: a child under
  `increments/` whose title/body text reads like a bug report (e.g.
  `# Bug: this looks like a bug report but lives in increments/`). Trace:
  `scanLegacyRepo`'s outer loop iterates `LEGACY_FOLDER_NAMES` (line 194)
  and derives `kind` exclusively from `LEGACY_FOLDER_TO_KIND[folderName]`
  (line 195) BEFORE reading any file content — content is read only
  afterward (line 242) purely to extract `title`/`rawStatus`/`body`, and
  is never consulted to influence `kind`. There is no branch anywhere in
  `scanLegacyRepo` that reads `body` and reassigns `kind`. **Confirmed
  correct by static trace** and independently re-verified by the smoke
  test's `INC-003-looks-like-bug` fixture, which asserts the resulting
  `metadata.yml`'s `kind` is `'increment'` despite the misleading
  content.
- **Bonus case traced: malformed entry (directory with no markdown
  inside)** does not abort the scan. `findMarkdownInDirectory` returns
  `undefined` when no `README.md` and no `.md` glob match exists (line
  169); the caller (line 225) pushes a `ScanFailure` and `continue`s the
  loop rather than throwing. **Confirmed correct by static trace** and
  independently re-verified by the smoke test's `INC-999-empty-dir`
  fixture, which asserts the run still completes with `exitCode: 0` and
  the other 3 real items still migrate, with the failure surfaced in
  `result.lines` (`No se pudo leer ...`).

### 3. Q-legacy-1 tightening — independently verified correct

Read `migrator.ts`'s `resolveWorkflowStateForMigration` (lines 82-87)
and `WORKFLOW_STATES` (lines 61-71) directly, side by side with
`scanner.ts`'s `KNOWN_STATUS_VALUES` (lines 48-62).

**Finding: the tightening is real and necessary, confirmed by direct
set comparison, not just re-trusting the impl spec's prose.**
`scanner.ts`'s `KNOWN_STATUS_VALUES` (13 entries) is the UNION of
`WorkflowState`'s 9 values (`draft`, `specifying`, `planned`,
`plan-approved`, `in-progress`, `verifying`, `archived`, `failed`,
`cancelled`) plus `AdrStatus`/`DecisionStatus`'s 4 values (`proposed`,
`accepted`, `superseded`, `rejected`) — confirmed by literal enumeration
of both constants. `migrator.ts`'s `WORKFLOW_STATES` set independently
re-declares exactly the 9 `WorkflowState` values (not importing/reusing
`scanner.ts`'s superset), so `resolveWorkflowStateForMigration` correctly
rejects a legacy `Status: accepted`/`Status: proposed`/`Status:
rejected`/`Status: superseded` line found inside an `increments/`/`bugs/`/
`plans/` folder (an ADR-shaped word landing in the wrong kind-folder by
legacy-repo inconsistency) and falls back to `'draft'`, rather than
incorrectly assigning an ADR-only status string to an
`IncrementOrBugMetadata.status: WorkflowState` field where it would
violate the type or, worse, silently coerce. **This is the exact gap the
impl spec claims to close — independently confirmed closed by reading
both constants directly, not by trusting the claim.**

### 4. `--dry-run` zero-writes — independently verified by code-path trace, not just trusting tests

Read `runBootstrapFromLegacySdd` in `bootstrap.ts` directly (lines
281-381). The `dryRun` branch (line 327, `if (dryRun) { ... return
{...} }`) executes and RETURNS before `migrateLegacyItems` is ever
called (line 346) — `migrateLegacyItems` is the ONLY function in the
entire call graph that invokes any of `saveArtifactMetadata`/
`ensureArtifactReadmeWithContent`/`fs.writeFileSync`/`fs.mkdirSync`
(confirmed by reading `migrator.ts` in full: the only write calls in the
module are inside `migrateLegacyItems` and
`ensureArtifactReadmeWithContent`, both unreachable from the `dryRun`
branch by construction — not merely "tested to not write," but
structurally unreachable). `scanLegacyRepo` (the only function the
`dryRun` branch does call) is read-only by its own module header comment
and by direct inspection — its only `fs` calls are `readdirSync`,
`existsSync`, `statSync`, `readFileSync`, none of which mutate.
**Confirmed by static reachability analysis, not by trusting the
existing directory-non-existence test assertions** — independently
re-verified end-to-end by the smoke test's dry-run case, which snapshots
`axiom.spec/`'s directory listing before and after a dry-run call and
asserts byte-for-byte equality, plus asserts `increments/`/`bugs/`
subdirectories were never created.

### 5. Collision-skip-and-report (Q-legacy-4) — independently verified to report clearly, never silently

Read `runBootstrapFromLegacySdd`'s post-migration reporting block
(`bootstrap.ts` lines 355-360) directly:

```
if (migrationResult.skipped.length > 0) {
  lines.push(`  Items salteados por colisión (${migrationResult.skipped.length}):`);
  for (const skipped of migrationResult.skipped) {
    lines.push(`    - [${skipped.kind}] ${skipped.childName}: ${skipped.reason}`);
  }
}
```

**Confirmed: a skip is always surfaced in `result.lines` (the same array
printed to `process.stdout` by `registerBootstrap`'s action handler,
line 449-451) with the kind, the original legacy child name, and a
human-readable reason string — never swallowed into a silent no-op.**
The exit code remains `0` for a skip (only actual `saveArtifactMetadata`
`io-error`s in `migrationResult.errors` set exit code `1`, line 379) —
this is a deliberate, correct distinction: a skip is a safe, expected
outcome of the collision-avoidance design, not a command failure,
consistent with `listArtifacts`'s own established "one bad entry doesn't
fail the whole operation" posture elsewhere in this codebase. Both
`migrator.test.ts`'s and `bootstrap.test.ts`'s existing "Q-legacy-4"
describe blocks were also independently re-run (see "Validation" below)
and independently re-read; they assert the observable guarantee that
matters (idempotent-safe re-runs, no data loss), which this role's own
smoke test (see below) also independently re-derives without relying on
those existing tests' mocks.

### 6. End-to-end smoke test — independently run, PASSED

The built CLI was confirmed independently broken, reproducing the exact
same known defect INC-18's validator already found:

```
node apps/cli/dist/index.js bootstrap from-legacy-sdd --help
-> Error: Cannot find module './_shared'
   (dist/commands/init.js, required eagerly at CLI startup)
```

This is the same pre-existing, unrelated `packages/cli-commands`
dist-build defect already tracked in `general-spec.md` since before
INC-18 — it breaks the built CLI at startup for effectively every
command, not specific to `from-legacy-sdd`.

Per the same posture INC-18's validator used, a temporary vitest smoke
test (`apps/cli/tests/bootstrap-smoke-INC19-validator.test.ts`, created,
run, then deleted — not part of the shipped diff) was written and run
against a **realistic legacy-repo fixture** built from scratch
(`fs.mkdtempSync`-based, not reusing any existing test fixture
verbatim), containing:

1. `increments/INC-001-old-feature/README.md` — folder-shape, `#
   heading` title, `Status: done` line (tests Q-legacy-1's synonym
   mapping to `archived`).
2. `bugs/BUG-002-flat-bug.md` — flat-file shape, `# heading` title, no
   status line at all (tests the unparseable-status-defaults-to-draft
   path).
3. `increments/INC-003-looks-like-bug/README.md` — folder-shape,
   deliberately misleading title (`# Bug: this looks like a bug report
   but lives in increments/`) to independently re-test Q-legacy-2's
   folder-name-only kind detection with a fresh fixture (not reusing the
   implementer's own test fixture).

Four independent test cases were run against this fixture:

- **Dry-run zero-writes**: snapshotted `axiom.spec/`'s directory listing
  before/after a `--dry-run` call, asserted byte-identical, and asserted
  `increments/`/`bugs/` subdirectories were never created. **PASSED.**
- **Real migration, all 3 fixtures**: asserted `createdCount === 3`,
  `skippedCount === 0`, `errorCount === 0`; read every resulting
  `metadata.yml` directly via `yaml.load` (not just trusting
  `result.lines`' text) and confirmed: `INC-001` maps to `kind:
  'increment'`, `status: 'archived'`; the flat-file bug maps to `kind:
  'bug'`, `status: 'draft'`, `title: 'Flat Bug Report'`; the
  misleading-content fixture maps to `kind: 'increment'` despite its
  bug-like title (Q-legacy-2 independently re-confirmed); every
  generated `README.md` contains the `AXIOM:MIGRATED` provenance marker
  and the review-requirement sentence. **PASSED.**
- **Double-run collision-safety**: ran the full migration twice against
  the same legacy fixture; asserted the second run also succeeds
  (`createdCount: 3` again, fresh IDs), asserted the total artifact
  count under `increments/` is exactly 4 (2 real increments-fixtures ×
  2 runs, none overwritten), and asserted the FIRST run's `README.md`
  content is byte-identical before and after the second run (never
  clobbered). **PASSED.**
- **Malformed-entry resilience**: added an empty directory
  (`increments/INC-999-empty-dir`, no markdown inside) to the fixture
  and re-ran; asserted the run still completes with `createdCount: 3`
  (the other 3 real items unaffected) and the failure is surfaced in
  `result.lines` (`No se pudo leer ...`). **PASSED.**

All 4/4 smoke-test cases passed on first corrected run (one initial
assertion-count arithmetic error in this role's own test, not the
implementation, was caught and fixed before reporting this result).

## Validation

Validation discovery: same as the implementer's — this monorepo's root
`package.json` declares `typecheck` (`tsc -b`), `build` (`tsc -b`), and
`test` (`vitest run`). All three independently re-run by this role.

- **`npm run typecheck`**: clean, no errors.
- **`npm run build`**: clean, no errors.
- **`npm test` (full monorepo, independently re-run)**: **12 failed
  files / 13 failed tests / 1598 passed** (out of 1611 total) —
  independently confirmed to be the EXACT SAME 13 pre-existing failures
  as both the documented pre-INC-19 baseline (12 failed files / 13
  failed tests / 1559 passed) and the implementer's own reported
  post-implementation count, by diffing the actual failed-test name list
  (not just the counts): `apps/cli/tests/start.test.ts`,
  `packages/agents/tests/catalog.test.ts`,
  `packages/doctor/tests/checks.test.ts`,
  `packages/model-routing/tests/{assignments,loader,
  opencode-projection,resolver}.test.ts`,
  `packages/project-resolution/tests/resolver.test.ts`,
  `packages/skills/tests/catalog.test.ts`,
  `packages/telemetry/tests/audit-trail-sink.test.ts`,
  `packages/toolchain/tests/repair-add-gitignore.test.ts`, and two cases
  in `packages/tui/tests/driver.test.ts` — none touch `bootstrap.ts`,
  `artifact-store.ts`, or `bootstrap-from-legacy-sdd/`. **Zero
  regressions, independently confirmed** (not re-trusted from the
  implementer's report).
- **Scoped run** (`npx vitest run apps/cli/tests/bootstrap-from-legacy-sdd
  apps/cli/tests/bootstrap.test.ts`): 3 files, **46/46 passed**,
  independently re-run and matching the implementer's claim exactly.
- **Temporary end-to-end smoke test** (created, run, deleted; not part
  of the shipped diff): `apps/cli/tests/
  bootstrap-smoke-INC19-validator.test.ts` -> **4/4 passed**, exercising
  `runBootstrapFromLegacySdd` against a fresh, realistic legacy-repo
  fixture built for this validation pass (not reusing the implementer's
  own fixtures).
- **Manual reproduction of the known dist-build defect** (informational,
  not part of pass/fail counts): `node apps/cli/dist/index.js bootstrap
  from-legacy-sdd --help` -> `Cannot find module './_shared'`,
  independently confirming this pre-existing, unrelated defect is still
  present and still blocks the built CLI for every command, not just
  `from-legacy-sdd`.

## Result

Independently reviewed and confirmed CORRECT, with no code changes
required:

1. **Bypass-of-`runIncrementSubcommand` judgment call: CONFIRMED SOUND**,
   using direct evidence beyond what the implementer's own spec cited —
   `workflow-state.json` is a per-`WorkflowId` SINGLETON store (one
   record per workflow type, not per artifact instance, confirmed by
   reading `state-store.ts`'s own type shape and header comment), and
   `artifact-store.ts`'s own design comment explicitly states
   `metadata.yml` and `workflow-state.json` are parallel, independent
   stores by design (INC-06's original intent). Force-populating
   `workflow-state.json` during a bulk migration would require
   collapsing N migrated artifacts into one global slot — an active
   correctness hazard, not a missing feature. No fix needed.
2. Scanner folder-vs-flat-file detection and status extraction:
   independently traced against 4 concrete cases (folder+parseable
   status, flat-file+unparseable status, folder-name-only kind detection
   with a misleading fixture, and malformed-entry resilience) — all
   confirmed correct by direct source-code trace, then independently
   re-verified end-to-end via a real smoke test.
3. Q-legacy-1's `WorkflowState`-only tightening: independently confirmed
   correct by direct set comparison of `scanner.ts`'s
   `KNOWN_STATUS_VALUES` (13-value superset) against `migrator.ts`'s
   independently-declared `WORKFLOW_STATES` (9-value subset) — the
   tightening is real and necessary, not cosmetic.
4. `--dry-run`: independently confirmed to perform literally zero
   filesystem writes by static reachability analysis of the call graph
   (the `dryRun` branch returns before `migrateLegacyItems` — the only
   function with write calls — is ever invoked), then independently
   re-verified via a directory-snapshot smoke test.
5. Collision-skip-and-report (Q-legacy-4): independently confirmed to
   always surface skips in the printed output with kind/name/reason,
   never silently swallowed, with exit code correctly staying `0` for a
   skip (only real I/O errors set exit `1`).
6. End-to-end smoke test against a fresh, realistic fixture (not reusing
   the implementer's own fixtures): 4/4 passed, covering dry-run,
   real migration with content/status/kind verification, double-run
   collision-safety, and malformed-entry resilience.
7. Full validation suite independently re-run: typecheck clean, build
   clean, `npm test` -> 12 failed files / 13 failed tests / 1598 passed
   (1611 total) — confirmed byte-for-byte identical failing-test list to
   both the documented pre-INC-19 baseline and the implementer's own
   report. Zero regressions, independently confirmed.

**INC-19 (as a whole) closure assessment: CLOSED.** All of INC-19's own
overarching goals (per the audit spec's framing) are met:

- A minimal, mechanical `axiom bootstrap from-legacy-sdd <path>
  [--dry-run]` command exists, migrating increments/bugs/plans/adr/
  decisions from a legacy folder-based SDD repo into new Axiom artifacts.
- No new artifact-store writer was introduced — confirmed end-to-end,
  not just at design time.
- Every migrated artifact is forced into a safe draft/proposed status by
  default, per addendum section 15's explicit requirement — confirmed
  independently across all fixture cases.
- All four open questions (Q-legacy-1 through Q-legacy-4) are resolved
  with documented rationale, and independently re-verified correct by
  this role via direct code trace (not just re-reading the prior roles'
  prose).
- The one flagged judgment call (bypass of `runIncrementSubcommand`) was
  independently reviewed with concrete evidence and confirmed sound.
- Real validation (typecheck/build/test) was executed independently,
  twice now (implementer + validator), with zero regressions both times.

**Deferred items, named explicitly** (not gaps in INC-19's own scope —
explicitly out of scope per the audit's Non-goals, reaffirmed here):

- **Content restructuring/normalization**: legacy body content is
  carried verbatim into the new artifact's README, not restructured into
  goal/scope/acceptance-criteria sections. Would require semantic
  understanding of arbitrary legacy prose — explicitly deferred to a
  future increment if ever requested.
- **ADR inference from unstructured legacy prose**: only migrates
  ADR/Decision content that already lives in an explicitly-named legacy
  `adr`/`decisions` folder. No NLP-based extraction of implicit
  decisions from prose elsewhere in a legacy repo.
- **`--kind` filter flag**: not implemented; the audit flagged this as
  "plausible but not decided as in-scope," and the implementer
  deliberately kept the slice minimal. A follow-up increment could add
  it if a real need arises (e.g. very large legacy repos where partial
  migration passes are useful).
- **No self-migration of this workspace's own `Axiom.Spec/specs/
  increments/`/`specs/bugs/` content**: explicitly out of scope by the
  audit's own framing (INC-19 builds a generic product capability, not a
  one-off self-migration script) — not revisited by this role.
- **Two pre-existing, unrelated issues re-confirmed but not fixed** (out
  of INC-19's scope by definition, and out of scope for this validator
  pass as well): the `packages/cli-commands` dist-build defect (breaks
  the built CLI broadly, tracked in `general-spec.md` since before
  INC-18) and the pre-existing 13-failing-test baseline (all
  environment/real-repo-path/timing-dependent, none related to
  bootstrap/workflow/artifact-store).

## General spec integration

Added a new "Bootstrap-from-legacy-sdd" section to
`Axiom.Spec/general-spec.md`, mirroring the existing "Bootstrap-from-code
(Level 0/1)" section's structure and depth: CLI surface, input
assumption (cross-referencing the audit's explicit "this is an invented
assumption, not a specified schema" framing), the four Q-legacy
resolutions with their rationale, the direct-primitive-reuse pattern (no
new writer), and the confirmed state-store/artifact-store separation
finding from this validation pass (now elevated to stable,
cross-increment knowledge since it generalizes beyond this one command —
it explains why ADR/Decision creation and this migration both correctly
skip `workflow-state.json`).

**This also completes Phase F (Bootstrap documental) as a whole** —
both INC-18 (`bootstrap-from-code`) and INC-19 (`bootstrap-from-legacy-sdd`)
are now closed, each having gone through the full
audit -> implement -> validate sequence with independent verification at
each step and zero unresolved regressions.

## Next step recommendation

Phase F is complete. Proceed to **Phase G — Versionado y upgrades**,
starting with **INC-20 — Reconcile version tracking
(`@axiom/versioning`)**, per the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
line 661): audit the existing `@axiom/versioning` package
(`managed-state.ts`, `checkpoints.ts`, `migrations.ts`, `upgrade.ts`,
`version.ts`) before designing any new scope, following the same
audit-first posture INC-18/INC-19 both used.
