# Increment: Reconcile migration engine and upgrade commands

Status: closed
Date: 2026-07-02

## Goal

Audit `axiom upgrade`'s current output format against addendum §8's rule
("Axiom debe mostrar qué cambios se han generado en cada repo y
recomendar commits separados por repositorio") and the roadmap's INC-21
hypothesis, then decide honestly whether per-repo change reporting is
buildable now or premature.

## Context

Roadmap: `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
Phase G, INC-21. Prior increment in the same phase, INC-20
(`INC-20260702-versioning-reconcile`), audited `@axiom/versioning`'s
persisted shape and closed with a no-code rationale, naming the same
blocking dependency this increment also runs into: `axiom.yaml
schemaVersion: 2` was attempted and reverted in
`INC-20260702-registry-manifest-schema-v2-cli` (`Status: pending`,
unresolved since INC-01). See `Axiom.Spec/general-spec.md`'s
"Versioning" section for the full prior finding.

Addendum §8 (`axiom_decisiones_sesion_addendum_revision.md`) asks for a
post-operation summary shaped like:

```text
Cambios detectados:
- spec repo: N ficheros creados/modificados
- sdd repo: N ficheros creados/modificados
- backend repo: N ficheros creados/modificados
- frontend repo: N ficheros creados/modificados
```

The roadmap's own hypothesis for INC-21 states `axiom upgrade` already
satisfies source doc §16.4's detect→diff→apply→validate flow "almost
verbatim" via `--dry-run`/`--from-checkpoint`/`--target-version`/
`--no-sync`/`--no-doctor`, and flags per-repo reporting as the one
open diff, tentatively deferred until multi-repo becomes primary.

## Scope

- Read `Axiom/packages/versioning/src/upgrade.ts` and
  `Axiom/apps/cli/src/commands/upgrade.ts` in full; confirm the exact
  shape of `axiom upgrade`'s current output (dry-run and real-run).
- Check whether any changed-file list (not just a version-transition
  summary) is already surfaced anywhere in the upgrade path, and compare
  against the closest sibling command (`axiom sync`) for precedent.
- Assess whether per-repo change reporting is buildable now, given the
  confirmed single-repo-default reality, or whether it is blocked on the
  same `schemaVersion: 2`/multi-repo-primary thread INC-20 already
  flagged.
- Separately assess whether a smaller, non-speculative, single-repo-scoped
  improvement (listing changed file paths, not just a repo-plural
  breakdown) is warranted now, distinct from "per-repo" reporting.
- Record the finding; implement only if a genuinely small, low-risk,
  well-covered change is identified.

## Non-goals

- No multi-repo change-reporting implementation (blocked, see below).
- No re-litigation of Q1/Q2/Q5 (roadmap's open architecture questions) —
  this increment only confirms they remain the blocking dependency, it
  does not attempt to resolve them.
- No modification of `axiom sync`'s own output format, even though it is
  discussed here for comparison — out of scope for this increment.
- No rewrite of `upgrade.ts`'s checkpoint/rollback mechanism (INC-20
  already confirmed it real and working; not this increment's concern).

## Acceptance criteria

- [x] `upgrade.ts`'s current output format is confirmed by direct read
      (not inferred from the roadmap's hypothesis alone).
- [x] The premature-multi-repo-reporting question is answered with
      concrete evidence, not restated as a generic hedge.
- [x] A smaller single-repo-scoped improvement is explicitly considered;
      if not implemented, the reason is stated in code-level terms (test
      coverage, build-tooling risk), not just "still blocked."
- [x] If the second-consecutive-blocked-increment pattern (INC-20,
      INC-21) is confirmed, it is named explicitly with a recommendation
      to the orchestrator/user.
- [x] Available validation was executed and its result recorded.
- [x] Closure status set with explicit rationale either way.

## Open questions

None new. Inherits the roadmap's Q1/Q2/Q5 (still open, tracked at the
roadmap level, not restated in full here) and INC-20's already-flagged
blocker (`axiom.yaml schemaVersion: 2`, `INC-20260702-registry-manifest-
schema-v2-cli`, `Status: pending`).

## Assumptions

- "Per-repo" in addendum §8 means literally plural repos (spec/sdd/
  backend/frontend as separate change-count lines) — a single-repo
  product upgrade has exactly one line to show, which is not the
  addendum's intended shape, not merely a degenerate case of it.
- `axiom sync`'s existing `generatedFilesCount` output (a count, not a
  path list) is treated as the closest existing precedent in this
  codebase for "post-operation change summary," not as evidence that
  file-path listing is already solved elsewhere and out of scope here.

## Implementation notes

### 1. `upgrade.ts`'s current output format (confirmed by direct read)

`Axiom/packages/versioning/src/upgrade.ts`:

- `UpgradePlan` (dry-run): `fromVersion`, `toVersion`, `migrations`
  (`Migration[]`, each with `from`/`to`/`description`, no file-level
  detail), `touchedFiles` (`string[]` — the paths the pre-upgrade
  checkpoint will cover: `init.json`, `install-profile.json`,
  `managed-state.json`), `isNoOp`.
- `UpgradeResult` (real run): `fromVersion`, `toVersion`,
  `appliedMigrations` (`Migration[]`), `checkpointId`, `syncRun`
  (boolean), `doctorRun` (boolean), `followUps` (`string[]`, always `[]`
  today — no code path populates it).

`Axiom/apps/cli/src/commands/upgrade.ts`'s `formatPlan`/`formatResult`
render exactly those fields as text — confirmed identical to
`Axiom/docs/cli/upgrade.md`'s own documented sample output. Neither
function lists any file as "changed" — `touchedFiles` describes what the
checkpoint *covers* (a fixed, hardcoded 3-path list from
`computeTouchedFiles`, unrelated to what a migration actually modifies),
and `appliedMigrations` names migration version pairs, never file paths.
**`axiom upgrade` today shows zero file-level change information, per
package or otherwise.**

### 2. Per-repo reporting: premature, confirmed by evidence, not by hedge

Confirmed facts (not inference):

- `Axiom/packages/topology/src/types.ts`'s `TopologyManifest.mode`
  defaults to `'single-repo'` (`defaultSingleRepoManifest`) — `sddRepo`
  and `specRepo` both resolve to `projectRoot`, `roleCodeRepositories` is
  empty. This is the MVP default per `Axiom.Spec/general-spec.md`'s
  "Project topology model" section, itself confirmed across 8+ closed
  increments.
- `axiom upgrade`'s own code path (`upgrade.ts`, `checkpoints.ts`) has no
  topology awareness at all — it operates on a single `rootPath` and a
  single `projectName`, with no iteration over `roleCodeRepositories` or
  `assignments`.
- The one place in the monorepo that already does real per-repo
  changed-file enumeration is `Axiom/packages/doctor/src/write-scope.ts`
  (`buildRepoChangeSets`, INC-08), which requires a `TopologyManifest`
  and an *active, approved plan* (`EXPECTED_PLAN_STATUSES`) to make sense
  of "which repo does this changed path belong to." `axiom upgrade` has
  neither of those inputs and gating it on an active plan would be a
  scope invention this increment was not asked to make.
- Building "per-repo" (plural) reporting today would mean either (a)
  fabricating a fake single-entry breakdown for the one repo that always
  exists in the default mode — which does not test or validate anything
  addendum §8 actually cares about (multiple repos), or (b) wiring
  `upgrade.ts` to `@axiom/topology` prematurely, ahead of INC-01 actually
  making multi-repo primary, which the roadmap explicitly names as Q1
  (blocking, still unresolved).

**Conclusion: per-repo change reporting is premature, confirmed, not
merely restated from the roadmap's hypothesis.** It remains correctly
sequenced as "after INC-01, INC-20" in the roadmap's own dependency
list.

### 3. Smaller single-repo-scoped improvement: considered, not implemented

Checked whether `axiom upgrade`'s *current*, single-repo-scoped output
already lists changed files (the core spirit of addendum §8 minus the
plural-repo part) — it does not (see #1). Checked the closest sibling
command for precedent: `axiom sync`
(`Axiom/apps/cli/src/commands/sync.ts`) also does **not** list file
paths — its final `process.stdout.write` block
(`registerSync`) only prints `generatedFilesCount` (a number), even
though each adapter's underlying result already carries
`writtenFiles: string[]` internally (`unwrapAdapterResult`'s generic
constraint) — that array is summed into a count and discarded before
reaching stdout. So this gap is not `upgrade`-specific; it is a
consistent pattern across both commands in this codebase today.

The clear, low-risk candidate would be: expose `checkpoint.files`
(`CheckpointRecord.files`, already a `string[]` returned by
`createCheckpoint`) or the git-diff-based `runGitDiff`
(`Axiom/packages/doctor/src/write-scope.ts`, already single-repo-capable
if called with just `rootPath` and no topology) in `formatResult`.

**Decision: not implemented in this increment**, for two concrete,
code-level reasons (not a generic "let's be careful" hedge):

1. **Zero existing test coverage for the file being changed.**
   `Axiom/apps/cli/src/commands/upgrade.ts` has no dedicated test file —
   only `Axiom/packages/versioning/tests/upgrade.test.ts` covers
   `previewUpgrade`/`executeUpgrade` (the underlying runtime, confirmed
   by `Glob` search: no `apps/cli/**/*upgrade*.test.ts` exists). Any
   change to `formatResult`/`runUpgrade` in the CLI wrapper would need
   new tests written from scratch, which is materially more than a
   "small tweak" and risks scope creep beyond what this audit increment
   was asked to do.
2. **Confirmed pre-existing build-tooling defect affects exactly this
   file.** `Axiom.Spec/general-spec.md`'s "Known build-tooling defect"
   section documents that `Axiom/packages/cli-commands/tsconfig.json`
   cross-includes `apps/cli/src/commands/upgrade.ts` (named explicitly,
   among 6 files) into a second composite TypeScript project, breaking
   `apps/cli`'s own `dist/` emission for this file. Editing this file
   compounds a known, already-flagged, unrelated defect rather than
   working around it — the correct sequencing is to fix the build defect
   first (its own dedicated increment, not scoped here), then extend
   this file with test coverage attached.

Making a source change here without tests and inside a known-broken
build boundary would violate `Axiom.SDD/AGENTS.md`'s "keep changes
small, focused, and validated" rule more than deferring it does. This is
recorded as a concrete, ready-to-pick-up follow-up (not silently
dropped): once the build-tooling defect is fixed, add a test file for
`apps/cli/src/commands/upgrade.ts` and extend `formatResult` to print
`checkpoint.files` (already collected, zero new computation needed).

### 4. Second consecutive increment blocked on the same root cause

This is the **second** increment in a row (INC-20, then this one,
INC-21) that reaches the same architectural blocker: `axiom.yaml
schemaVersion: 2` / multi-repo-primary was attempted in
`INC-20260702-registry-manifest-schema-v2-cli` during INC-01's
cli-implementer step, reverted, and left `Status: pending` — flagged
repeatedly since (roadmap Q1, INC-20's close, now this increment) but
never picked back up as its own increment. Phase G's remaining
increments (INC-22, "Configure/upgrade/repair operations") depend on
INC-21 and INC-15, both of which either are or will be gated on the same
multi-repo-primary transition. Continuing further into Phase G without
resolving this thread risks a third, fourth, and further consecutive
audit-only closure with no forward progress.

**Recommendation to the user/orchestrator**: pick up the deferred
`axiom.yaml schemaVersion: 2` / multi-repo-primary re-enablement as its
own explicit increment (resuming `INC-20260702-registry-manifest-
schema-v2-cli` or opening a fresh INC-01 follow-up) before continuing
further into Phase G (INC-22) or any other increment whose diff depends
on Q1's resolution.

## Validation

`vitest run packages/versioning` (36 tests, 4 files) — all passing,
confirming the pre-existing state is unchanged by this audit-only
increment:

```text
✓ packages/versioning/tests/migrations.test.ts (7 tests)
✓ packages/versioning/tests/managed-state.test.ts (8 tests)
✓ packages/versioning/tests/checkpoints.test.ts (9 tests)
✓ packages/versioning/tests/upgrade.test.ts (12 tests)
Test Files  4 passed (4)
     Tests  36 passed (36)
```

No `apps/cli`-level test exists for `upgrade.ts` to run (see
Implementation notes §3) — this is itself a confirmed finding, not a
gap in this increment's own validation.

## Result

Confirmed `axiom upgrade`'s current output is a version-transition
summary only (no file-level change information, per-repo or otherwise).
Confirmed per-repo change reporting (addendum §8) is genuinely premature
today — blocked on the same unresolved `schemaVersion: 2`/multi-repo-
primary thread INC-20 already flagged, not a new blocker. Considered a
smaller single-repo-scoped alternative (listing changed file paths) and
found it technically feasible (the data already exists in
`CheckpointRecord.files`/`runGitDiff`) but decided against implementing
it now due to zero existing test coverage for the CLI wrapper file and a
confirmed, already-documented build-tooling defect affecting that exact
file — implementing it now would mean writing tests from scratch inside
a known-broken build boundary, which is a larger and riskier change than
this audit increment's scope. No code was changed in `Axiom` or
`Axiom.SDD`.

## General spec integration

Added a short cross-reference to `Axiom.Spec/general-spec.md`'s
"Versioning" section confirming `axiom upgrade`'s output format and
explicitly naming this as the second consecutive increment blocked on
the `schemaVersion: 2`/multi-repo-primary thread, so the pattern is
visible at the consolidated-knowledge level, not only inside this
increment's own file.

## Next step recommendation

Do not proceed to INC-22 (Phase G's next increment) yet. Instead, open a
dedicated increment to resolve the deferred `axiom.yaml schemaVersion: 2`
/ multi-repo-primary re-enablement (Q1), since it is now confirmed to
block two consecutive increments in this phase and will continue to
block INC-22 and any future per-repo-reporting work. Once that lands,
revisit this increment's §3 follow-up (add test coverage for
`apps/cli/src/commands/upgrade.ts`, then surface `checkpoint.files` in
`formatResult`) as a small, low-risk, single-repo-scoped addition.
