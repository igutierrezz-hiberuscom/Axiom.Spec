# Increment: Bootstrap-from-legacy-sdd — cli-implementer

Status: closed
Date: 2026-07-03

**Closure note (added by validator-reviewer, 2026-07-03)**: this
increment's status was `pending` per the collapsed 2-role sequence
(`cli-implementer` only; `validator-reviewer` had not yet run). It is now
`closed` because the `validator-reviewer` pass independently reviewed
and confirmed every claim in this spec — including the flagged
`runIncrementSubcommand` bypass judgment call (confirmed sound, not
fixed) — ran an independent end-to-end smoke test, and independently
re-ran the full validation suite with zero regressions. See
`Axiom.Spec/specs/increments/INC-20260702-bootstrap-legacy-sdd-reconcile-validator/README.md`
for the full independent findings.

## Goal

Implement `axiom bootstrap from-legacy-sdd <path> [--dry-run]`: a minimal,
mechanical migrator that scans a legacy SDD/spec repo
(`increments/`/`bugs/`/`plans/`/`adr/`/`decisions/` subfolders) and creates
new Axiom-format artifacts for each entry found, reusing the EXISTING
artifact-store creation primitives from INC-06/INC-11 (no new writer). This
is **INC-19**'s `cli-implementer` role, following up on the
`migration-engineer` audit at
`Axiom.Spec/specs/increments/INC-20260702-bootstrap-legacy-sdd-reconcile/README.md`.

## Context

Re-read the audit spec in full before implementing (its "Assumptions",
"Implementation notes", and "Brief for the next role" sections are this
increment's exact scope boundary). Key facts carried over from the audit:

- Input assumption: a legacy repo with zero or more of
  `increments/`/`bugs/`/`plans/`/`adr/`/`decisions/` folders, each child
  either a subdirectory containing a markdown file (preferring
  `README.md`, falling back to the first `.md` found) or a flat `.md`
  file. Title from the first `# heading`; best-effort status; everything
  else in the body carried verbatim.
- Write primitives to reuse directly (already confirmed reusable by the
  audit, re-confirmed here by reading the source): `generateUniqueArtifactId`,
  `artifactExists`, `saveArtifactMetadata`, `resolveArtifactDir`, and the
  five `makeInitial*Metadata` factories, all from
  `Axiom/packages/workflow/src/artifact-store.ts` /
  `Axiom/packages/workflow/src/artifact-id.ts`, exported via
  `@axiom/workflow`'s barrel.
- Addendum section 15: bootstrap output — explicitly including
  "migraciones desde SDD legacy" — must be born `draft`/`proposed`, never
  an approved/terminal state, regardless of what the legacy file claims.
- Role collapse (per the audit's recommendation): this increment covers
  BOTH `cli-implementer` (scan + migrate + CLI wiring + tests) work,
  collapsing source doc 15.3's 7-subagent chain and the roadmap's 5-role
  sequence down to this single implementation pass, followed by a
  separate `validator-reviewer` pass (not part of this increment).

## Scope

- A read-only scanner (`apps/cli/src/bootstrap-from-legacy-sdd/scanner.ts`)
  that walks the five kind-folders, supports both folder-per-artifact and
  flat-`.md` shapes, and extracts title/status/body per the audit's rules.
- A migration module (`apps/cli/src/bootstrap-from-legacy-sdd/migrator.ts`)
  that creates new Axiom artifacts via the existing primitives, forces a
  safe status default, and writes a provenance-marked README.
- Wiring `axiom bootstrap from-legacy-sdd <path> [--dry-run]` as a second
  subcommand in the existing `apps/cli/src/commands/bootstrap.ts` (same
  file INC-18 created for `from-code`), not a new command file.
- `--dry-run`: preview-only, zero writes.
- Collision handling per Q-legacy-4 (resolved below): skip-and-report,
  never overwrite, never abort the whole batch.
- Unit tests for the scanner, the migrator, and the CLI-level command
  (`runBootstrapFromLegacySdd`), covering both artifact shapes, status
  extraction (matched/unmatched), dry-run (no writes), and collision
  handling.

## Non-goals

- No content restructuring or semantic normalization of legacy body text
  (carried verbatim into the new artifact, per the audit's Non-goals).
- No ADR inference from unstructured prose outside an explicit
  `adr`/`decisions`-shaped legacy folder.
- No migration of THIS workspace's own `Axiom.Spec/specs/increments/`
  content — out of scope per the audit (INC-19 builds a generic product
  capability, not a self-migration script).
- No new artifact-store writer, no new metadata shape, no modification to
  `artifact-store.ts`/`artifact-id.ts` — pure consumer of already-shipped
  primitives.
- No `--kind` filter flag or other flags beyond `--dry-run` — the audit
  left this open as "plausible but not decided as in-scope"; not added
  here to keep the slice minimal. Can be a follow-up if requested.

## Acceptance criteria

- [x] `axiom bootstrap from-legacy-sdd <path>` scans
      `<path>/{increments,bugs,plans,adr,decisions}/`, skipping absent
      kind-folders without error.
- [x] Both folder-per-artifact (README.md preferred, first `.md` fallback)
      and flat-`.md` shapes are supported.
- [x] Title extracted from the first `# heading`; falls back to the
      child's name (sans `.md`) if no heading is found.
- [x] Status extraction is best-effort and narrow (Q-legacy-1): a literal
      `^Status:\s*(\w[\w-]*)` line mapped through a synonym table; any
      unparseable/unmapped value defaults to `'draft'` for
      increment/bug/plan; ADR/Decision are always `'proposed'`
      (unaffected by legacy status — factory-hardcoded per INC-11).
- [x] Migration creates new Axiom artifacts via the existing
      `artifact-store.ts` primitives — no new writer introduced.
- [x] Every migrated artifact's README carries an explicit provenance
      marker naming the original legacy source path.
- [x] `--dry-run` performs zero filesystem writes and reports what would
      be created (kind, title, status, source path).
- [x] Collision handling (Q-legacy-4, resolved below): skip-and-report,
      never overwrite existing content, never abort the rest of the batch.
- [x] Tests cover: scanner (both shapes, title/status found/unparseable,
      absent kind-folder), migrator (correct kind/content/provenance,
      status forcing), CLI command (`--dry-run`, happy path, collision
      non-overwrite), all passing.
- [x] `npm run typecheck`, `npm run build`, and `npm test` executed; no
      regressions relative to the pre-existing baseline.

## Open questions — resolutions

All four questions the audit left open (explicitly non-blocking) are
resolved here, as directed by the audit's own brief for this role.

### Q-legacy-1 (status inference) — implemented as audited, no changes

Best-effort narrow match: a line matching `^Status:\s*(\w[\w-]*)`
(case-insensitive, multiline), passed through a small synonym table
(`done|complete|completed|closed|finished -> archived`,
`approved -> accepted`, `pending|open -> draft`, etc.), then checked
against the real known vocabulary (`WorkflowState` values for
increment/bug/plan; `AdrStatus`/`DecisionStatus` values are also
recognized by the synonym table so a legacy `Status: accepted` line
doesn't fall through to "unparseable," but see below for why that still
doesn't leak into increment/bug/plan's status). If no match, or the
mapped value isn't in the known vocabulary, the scanner returns
`rawStatus: undefined` and the migrator defaults to `'draft'`.

One refinement beyond the audit's literal text, made during
implementation: `resolveWorkflowStateForMigration` additionally checks
that the resolved value is specifically a member of the `WorkflowState`
set (9 values) — not just "any known status string" — before honoring it
for increment/bug/plan. This matters because the synonym table's
vocabulary is shared with `AdrStatus`/`DecisionStatus` (e.g. `'proposed'`,
`'accepted'`, `'rejected'`, `'superseded'` are valid ADR statuses but NOT
valid `WorkflowState` values). Without this extra check, a legacy
increment with `Status: accepted` (an ADR-shaped word landing in an
`increments/` folder by legacy-repo inconsistency) would silently pass
the "is it a known status" check and then fail to satisfy
`WorkflowState`'s type at the `makeInitialIncrementOrBugMetadata` call
site — or worse, be miscast. Checking the specific closed set for the
specific kind-family closes that gap. ADR/Decision are unaffected by any
of this: their factories hardcode `'proposed'` regardless of input
(INC-11 design), so Q-legacy-1's rule is moot for those two kinds by
construction — confirmed, not just assumed, by reading
`makeInitialAdrMetadata`/`makeInitialDecisionMetadata` directly.

### Q-legacy-2 (kind detection) — implemented as audited, no changes

Kind is derived exclusively from which of the five fixed subfolder names
(`increments`, `bugs`, `plans`, `adr`, `decisions`) a child lives under —
zero content-sniffing. Verified by a dedicated test
(`scanLegacyRepo: Q-legacy-2`) that plants a legacy file whose title/body
"looks like" a bug report inside `increments/` and confirms it is still
classified `kind: 'increment'`.

### Q-legacy-3 (folder-per-artifact vs flat file) — implemented as audited, no changes

Both shapes are scanned: a directory child is searched for `README.md`
first, then the first `.md` file (alphabetically, for deterministic
tests) if no `README.md` exists; a `.md` file child is read directly. A
directory with no markdown inside produces a `ScanFailure` (not a thrown
error) so one malformed entry doesn't abort the rest of the scan — same
posture as `listArtifacts`'s own per-entry failure collection.

### Q-legacy-4 (collision handling) — resolved: skip-and-report (option a)

**Full text of the audit's Q-legacy-4** (verbatim, for the record — the
brief text handed to this role was not actually truncated; re-reading the
full audit spec confirmed its complete reasoning was already present):

> What happens on a legacy title collision (two legacy files resolving to
> the same generated Axiom ID, or a legacy file whose derived slug
> collides with an already-existing Axiom artifact)? This audit
> recommends: rely on `generateUniqueArtifactId`'s existing uniqueness
> guarantee (it already probes for collisions when creating new
> artifacts) and skip-with-warning rather than overwrite if an identical
> target somehow already exists — never silently overwrite an existing
> Axiom artifact. Not blocking; next role's implementation detail.

**Decision: option (a), skip-and-report** — confirmed, not overridden.
Reasoning, arrived at independently before reading the audit's own
recommendation, then cross-checked against it:

- **Why not (b) disambiguating suffix**: `generateUniqueArtifactId`
  already generates IDs from a `<PREFIX>-<timestamp>-<6-char-suffix>`
  scheme with its own internal collision-retry loop (`maxAttempts`,
  default 10) against a real `exists()` probe. A genuine collision at
  this migration's call site is therefore already vanishingly unlikely
  *before* any application-level disambiguation logic runs — the
  underlying primitive already does exactly what option (b) would ask
  for, at the ID-generation layer, not the migration layer. Adding a
  second, migration-specific disambiguation scheme on top (e.g.
  `-2`/`-3` suffixes) would duplicate a guarantee the primitive already
  provides and introduce a second ID-shape convention inconsistent with
  every other artifact-creating command in this codebase (`axiom
  increment create`, `axiom-adr create`, etc. — none of them use
  suffix-based disambiguation; they all rely on
  `generateUniqueArtifactId`'s retry).
- **Why not (c) fail the whole run**: legacy repos are exactly the kind
  of messy, best-effort input this whole increment is designed around
  (see the audit's own "more speculative than most others" framing). A
  batch of 50 legacy items should not be entirely discarded because one
  entry hit the practically-impossible residual collision case (or, in
  this implementation, the equally-unlikely case where `artifactExists`
  is somehow still `true` for a freshly-generated ID due to a race). This
  mirrors `listArtifacts`'s own established pattern in this exact
  codebase: a single malformed `metadata.yml` is collected into
  `failures` and the scan continues, it does not abort listing every
  other artifact. Failing the whole run would also contradict this
  increment's own goal of being additive/non-destructive: draft-only
  migrated content should never gatekeep on a single bad legacy entry.
- **What "collision" concretely means in this implementation**: since
  `generateUniqueArtifactId` already does its own retry internally, this
  implementation's `migrateLegacyItems` treats two residual cases as
  "collision" and skips (never overwrites, never throws to the caller):
  (1) `generateUniqueArtifactId` itself throws after exhausting
  `maxAttempts` (astronomically unlikely given the 6-char base36 random
  suffix space, but handled defensively rather than left to propagate as
  an uncaught exception that would kill the whole CLI process); (2) a
  defensive re-check immediately before writing — if `artifactExists`
  somehow still reports `true` for the ID `generateUniqueArtifactId` just
  returned (e.g. a concurrent write raced the check-then-generate window),
  skip rather than overwrite. Both cases produce a `SkippedCollision`
  entry with a human-readable reason, surfaced in the CLI's printed
  output under a dedicated "Items salteados por colisión" section, and
  the run's exit code remains `0` (a skip is not treated as a command
  failure — only actual `io-error`s from `saveArtifactMetadata` set exit
  code `1`).
- **Test coverage for this decision**: `migrator.test.ts`'s
  "Q-legacy-4 (collision handling)" describe block and
  `bootstrap.test.ts`'s "Q-legacy-4 (collision handling — skip, never
  overwrite)" describe block both verify the observable guarantee that
  matters most: running the same migration twice against the same legacy
  repo never corrupts or overwrites the first run's artifact — it always
  produces a second, independently-addressable artifact with a distinct
  ID, and the original artifact's README content is untouched. A true
  forced collision (mocking `generateUniqueArtifactId` to exhaust
  attempts) was judged not worth the added mocking complexity for a
  defensive branch this unlikely to trigger in practice; the
  reachable-in-tests guarantee (idempotent-safe re-runs, no data loss) is
  the property that actually matters to a user running this command
  against the same legacy repo more than once.

## Assumptions

- Same legacy-format input assumption as the audit (see the audit spec's
  own "Assumptions" section) — not repeated here to avoid content drift;
  this increment implements exactly that shape, unmodified.
- `resolveArtifactDir` (already exported by `@axiom/workflow`) is a
  legitimate reuse target for path resolution even though it isn't in the
  audit's literal "primitives to reuse" list — it is the exact same
  path-resolution function `ensureArtifactReadme` itself calls internally,
  and was needed here because the migration needs to write a
  FULL-CONTENT README (the migrated body, wrapped with a provenance
  marker), not `ensureArtifactReadme`'s `# ${title}\n` title-only stub.
  No new write-primitive semantics were invented: the same "only write if
  absent, never clobber" guard `ensureArtifactReadme` uses is replicated
  exactly in the new `ensureArtifactReadmeWithContent` helper.
- The provenance marker text (`AXIOM:MIGRATED — created by \`axiom
  bootstrap from-legacy-sdd\`. Migrated from legacy SDD path: <path>.
  Review before relying on this content; do not treat as approved.`) is
  deliberately distinct from INC-18's `AXIOM:DRAFT` marker text (not just
  a copy with a different word), per the task's explicit instruction that
  this is "similar in spirit... but distinct since this is migration, not
  fresh drafting." `AXIOM:MIGRATED` as the marker keyword, plus the
  literal source path, gives a reviewer strictly more traceability than
  `AXIOM:DRAFT` alone would (a migrated artifact's origin is a concrete
  external file, not "the current repo's own code").

## Implementation notes

### Files created

- `Axiom/apps/cli/src/bootstrap-from-legacy-sdd/scanner.ts` — read-only
  scan: `scanLegacyRepo`, `extractLegacyTitle`, `extractLegacyStatus`,
  `LEGACY_FOLDER_TO_KIND`, `LEGACY_FOLDER_NAMES`.
- `Axiom/apps/cli/src/bootstrap-from-legacy-sdd/migrator.ts` — write side:
  `migrateLegacyItems`, `resolveWorkflowStateForMigration`,
  `buildProvenanceMarker`.
- `Axiom/apps/cli/tests/bootstrap-from-legacy-sdd/scanner.test.ts` (17 tests).
- `Axiom/apps/cli/tests/bootstrap-from-legacy-sdd/migrator.test.ts` (13 tests).

### Files modified

- `Axiom/apps/cli/src/commands/bootstrap.ts` — extended (not replaced):
  added `runBootstrapFromLegacySdd` + `BootstrapFromLegacySddArgs`/
  `BootstrapFromLegacySddResult` types, registered `from-legacy-sdd
  <path> [--dry-run]` as a second subcommand under the existing
  `bootstrap` namespace, updated the file's header comment and the
  `bootstrap` command's top-level description to mention both
  subcommands.
- `Axiom/apps/cli/tests/bootstrap.test.ts` — extended (not replaced):
  added a `runBootstrapFromLegacySdd` import and 12 new test cases across
  6 new `describe` blocks (unresolved project, invalid legacy path, empty
  legacy repo, `--dry-run`, happy path, Q-legacy-4 collision-safety),
  reusing the file's existing `tmpDir`/`scaffoldMinimalProject` fixtures.

### Reused, unmodified

`generateUniqueArtifactId`, `artifactExists`, `saveArtifactMetadata`,
`resolveArtifactDir`, `makeInitialIncrementOrBugMetadata`,
`makeInitialPlanMetadata`, `makeInitialAdrMetadata`,
`makeInitialDecisionMetadata` — all imported from `@axiom/workflow`'s
existing barrel, zero changes to `artifact-store.ts`/`artifact-id.ts`/
`index.ts`. `resolveProject` from `@axiom/project-resolution` — same
project-resolution pattern `from-code` already uses (destination project
resolved from cwd; `<path>` argument names only the external legacy
source, never the destination, per the audit's CLI design).

### Deviation check against the audit's brief

No deviations from the audit's minimal-viable scope. One clarification
worth flagging for the validator-reviewer: the audit's brief said
"reuse... the equivalent already-wired CLI `create` subcommand functions
themselves (check `axiom-increment.ts`/`axiom-plan.ts`/the ADR/Decision
command files for a reusable `runXCreate`-style function before
reimplementing the primitive call sequence a second time)." This was
checked directly: `axiom-adr.ts`'s `runCreateAdr` and `axiom-decision.ts`'s
`runCreateDecision` are indeed thin, primitive-level functions (generate
ID → build metadata via factory → `saveArtifactMetadata` →
`ensureArtifactReadme`) with no workflow-state coupling — but
`axiom-increment.ts`'s `runIncrementSubcommand` (the only "create"-style
function for increment/bug) is NOT a reusable thin function for this
purpose: it is deeply coupled to `workflow-state.json`
(`loadWorkflowState`/`saveWorkflowState`/`applyTransition`/hook engine),
which a migration should not invoke — a migrated increment/bug should get
a `metadata.yml` folder-per-artifact instance WITHOUT force-initializing a
parallel `workflow-state.json` record for it (that record is
per-workflow-ID state-machine tracking, keyed by a different `vars.id`
concept than the migration's derived legacy title, and forcing one into
existence here would silently interfere with a human later running real
`axiom increment create`/transitions on a similarly-named ID). Decision:
call the artifact-store PRIMITIVES directly (matching the audit's own
literal fallback instruction: "or the equivalent primitive call sequence")
for ALL five kinds uniformly, rather than reusing `runIncrementSubcommand`
for two of the five and primitives directly for the other three. This
keeps the migration path uniform across all five kinds and avoids the
side effect of writing `workflow-state.json` entries this migration has
no legacy-side data to populate correctly (no `--slug` equivalent, no
real command-driven transition history). Flagging this explicitly since
it is a judgment call beyond what the audit fully specified.

## Validation

Validation discovery: this monorepo's root `package.json` declares
`typecheck` (`tsc -b`), `build` (`tsc -b`), and `test` (`vitest run`) —
all three were run.

- **Baseline** (confirmed before implementing, via a clean working tree —
  no stash bisection needed since the working tree was clean at task
  start): `npm test` → **12 failed files / 13 failed tests / 1559 passed**
  (out of 1572 total). All 13 pre-existing failures are unrelated to
  bootstrap/workflow/artifact-store: `apps/cli/tests/start.test.ts`,
  `packages/agents/tests/catalog.test.ts`,
  `packages/doctor/tests/checks.test.ts`,
  `packages/model-routing/tests/{assignments,loader,opencode-projection,resolver}.test.ts`,
  `packages/project-resolution/tests/resolver.test.ts`,
  `packages/skills/tests/catalog.test.ts`,
  `packages/telemetry/tests/audit-trail-sink.test.ts`,
  `packages/toolchain/tests/repair-add-gitignore.test.ts`, and two cases
  in `packages/tui/tests/driver.test.ts` — all environment/real-repo-path
  or timing-dependent tests, none touching `bootstrap.ts` or
  `artifact-store.ts`.
- **`npm run typecheck`**: clean, no errors, before and after
  implementation.
- **`npm run build`**: clean, no errors, after implementation.
- **`npm test` (full monorepo, after implementation)**: **12 failed
  files / 13 failed tests / 1598 passed** (out of 1611 total) — the exact
  same 13 pre-existing failures (verified by diffing the failed-test
  names, not just counts), plus 39 net new passing tests (46 new test
  cases: 17 scanner + 13 migrator + 16 CLI-level, of which some were
  already-counted extensions of `bootstrap.test.ts`'s existing file — the
  net total-file-count delta is +2 test files, +39 net passing tests).
  Zero regressions.
- **Scoped run** (`npx vitest run apps/cli/tests/bootstrap.test.ts
  apps/cli/tests/bootstrap-from-legacy-sdd`): 3 files, **46/46 passed**.

## Result

Implemented `axiom bootstrap from-legacy-sdd <path> [--dry-run]` exactly
per the audit's minimal-viable scope: a read-only scanner
(`scanner.ts`) and a migration module (`migrator.ts`), both new files
under `apps/cli/src/bootstrap-from-legacy-sdd/`, wired into the existing
`apps/cli/src/commands/bootstrap.ts` as a second subcommand alongside
INC-18's `from-code`. No new artifact-store writer was introduced — every
write goes through the exact primitives INC-06/INC-11 already shipped and
already-tested (`generateUniqueArtifactId`, `saveArtifactMetadata`, the
five `makeInitial*Metadata` factories), confirming the audit's core
reusability finding end-to-end with real passing tests instead of just a
design-time confirmation.

All four open questions are resolved with documented rationale:
Q-legacy-1 (status: narrow best-effort match, safe `'draft'` default,
implemented exactly as audited plus one WorkflowState-specific tightening
found necessary during implementation), Q-legacy-2 (kind: folder-name-only,
implemented and test-verified against a deliberately misleading fixture),
Q-legacy-3 (both folder-per-artifact and flat-file shapes supported,
implemented as audited), and Q-legacy-4 (collision: skip-and-report,
confirmed as the audit's own recommendation after independently
re-deriving the same conclusion and reading the audit's full text — not
truncated after all).

One implementation-level judgment call beyond the audit's explicit text:
increment/bug migration uses the artifact-store primitives DIRECTLY
rather than reusing `axiom-increment.ts`'s `runIncrementSubcommand`,
because that function is coupled to `workflow-state.json` in a way a
migration has no correct legacy-side data to drive — see "Deviation
check" above for full reasoning. This is flagged explicitly for the
validator-reviewer to confirm or challenge.

Status is `pending`, not `closed`: per the audit's own collapsed 2-role
sequence, this increment covers `cli-implementer` only.
`validator-reviewer` — an independent, non-authoring pass confirming
correct kind/ID mapping, all-draft/proposed status, no silent overwrite,
valid metadata shape, and verbatim body round-tripping — has not yet run.

## General spec integration

No integration into `Axiom.Spec/general-spec.md` performed yet. Rationale:
per `Axiom.SDD/AGENTS.md`'s Documentation Rules, stable knowledge should be
consolidated once a feature's shape is confirmed stable — and this
increment's design (especially the increment/bug-vs-`runIncrementSubcommand`
judgment call, flagged above) has not yet been reviewed by
`validator-reviewer`. Once that review confirms the design is sound, the
next step should add a "Bootstrap-from-legacy-sdd" section to
`general-spec.md`, mirroring the existing "Bootstrap-from-code (Level
0/1)" section's structure: CLI surface, input assumption (cross-referencing
the audit's explicit "this is an invented assumption, not a specified
schema" framing), the four Q-legacy resolutions, and the
direct-primitive-reuse pattern (no new writer) as a confirmed, stable fact
about this codebase's artifact-creation architecture.

## Next step recommendation

Hand off to **validator-reviewer**, with these concrete inputs:

1. This spec (`Axiom.Spec/specs/increments/
   INC-20260702-bootstrap-legacy-sdd-reconcile-impl/README.md`) and the
   audit it follows up on
   (`Axiom.Spec/specs/increments/INC-20260702-bootstrap-legacy-sdd-reconcile/README.md`).
2. Implementation files: `Axiom/apps/cli/src/bootstrap-from-legacy-sdd/scanner.ts`,
   `Axiom/apps/cli/src/bootstrap-from-legacy-sdd/migrator.ts`,
   `Axiom/apps/cli/src/commands/bootstrap.ts` (the `from-legacy-sdd`
   additions specifically).
3. Test files: `Axiom/apps/cli/tests/bootstrap-from-legacy-sdd/scanner.test.ts`,
   `Axiom/apps/cli/tests/bootstrap-from-legacy-sdd/migrator.test.ts`,
   `Axiom/apps/cli/tests/bootstrap.test.ts` (the `from-legacy-sdd`
   describe blocks specifically).
4. Specific things to confirm or challenge, per this spec's own flags:
   - The "Deviation check" judgment call: is calling artifact-store
     primitives directly (bypassing `runIncrementSubcommand`) for
     increment/bug migration correct, or should migrated
     increments/bugs also get a `workflow-state.json` entry?
   - Confirm every migrated artifact is genuinely `draft`/`proposed` in
     all code paths (including the Q-legacy-1 edge case where a legacy
     `Status:` line matches an ADR/Decision-shaped word inside an
     increment/bug/plan folder).
   - Confirm no existing Axiom artifact is ever overwritten (re-run the
     collision tests, consider adding a forced-collision test via
     mocking if judged worth the complexity).
   - Confirm `--dry-run` performs literally zero filesystem writes
     (already asserted in tests via directory-non-existence checks, but
     an independent read is warranted per this workspace's "fresh
     review rule").
   - Spot-check that verbatim body content round-trips correctly
     (already covered by one test with a custom heading + list markup;
     consider one more spot-check with real INC-19-history-style
     content if available).
