# Increment: Reconcile bootstrap-from-code — implementation (Level 0/1)

Status: closed (independently re-verified and closed by `INC-20260702-bootstrap-from-code-reconcile-validator`)
Date: 2026-07-02

## Goal

Implement the `bootstrap-implementer` role's brief from the
migration-engineer audit
(`Axiom.Spec/specs/increments/INC-20260702-bootstrap-from-code-reconcile/README.md`):
ship `axiom bootstrap from-code --level minimal|basic`, a Level 0
(Minimal)/Level 1 (Basic) mechanical analysis+draft+write+index pipeline
that reads an existing codebase and drafts technical-context documents
plus a `TechnicalContextIndex` YAML referencing them — always in
draft/review state, never auto-approved (addendum section 15). This is
INC-18 of the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
Phase F), continuing directly after the audit increment above.

## Context

The audit (read in full before this implementation) confirmed:

- `Axiom/packages/document-bootstrap/src/writer.ts`'s path-guard +
  atomic-write steps are pure/generic and reusable; `writeCopilotInstructions`
  itself is not (hardwired to one file/renderer/variable shape). The
  correct reuse is extracting the two pure blocks into a small shared
  helper, not calling `writeCopilotInstructions` directly.
- `@axiom/technical-context`'s `TechnicalContextIndex`/
  `loadTechnicalContextIndex` (INC-10) are a good fit for bootstrap
  output, with a pre-existing gap (no path-existence validation between
  the index and the documents it references) the bootstrap step must
  handle by construction (write both in the same pass, keep paths
  consistent).
- Only Level 0 (Minimal) and Level 1 (Basic) are in scope — Level 2
  (Standard), Level 3 (Deep), Level 4 (Exhaustive) are explicitly
  deferred (they require genuine code-semantic judgment, out of scope
  for this pass).
- The roadmap's 4-role sequence (`bootstrap-analyzer -> docs-skills-writer
  -> index-engineer -> validator-reviewer`) collapses to 2 roles for this
  scope: `bootstrap-implementer` (this increment, analysis + draft +
  write + index in one pass) + `validator-reviewer` (unchanged, separate,
  next).
- CLI surface: `axiom bootstrap from-code --level minimal|basic`
  (namespace `bootstrap`, verb `from-code`, default level `minimal`).

## Scope

- Extract `writer.ts`'s path-guard + atomic-write primitive into a small,
  genuinely reusable exported function (`writeGuardedFile`), used by both
  the existing `writeCopilotInstructions` (refactored, no behavior
  change) and the new bootstrap writer.
- Implement Level 0/1 mechanical analysis: stack/package-manager
  detection (`package.json`/`pyproject.toml`/`requirements.txt`/
  `Gemfile`/`go.mod`/`Cargo.toml`), a top-level repo map with
  best-effort purpose guesses from known folder-name conventions,
  declared commands (`package.json#scripts`), and a Level-1 summary
  paragraph assembled mechanically from these facts.
- Draft markdown documents (overview, repo-map, stack, commands) via the
  extracted writer helper, carrying an `AXIOM:DRAFT` banner
  (Q-bootstrap-2), under `<specScope>/technical-context/<role>/*.md`
  (INC-10's existing path convention).
- Generate a `TechnicalContextIndex` YAML per role at
  `<specScope>/technical-context/indexes/<role>.index.yml`, populating
  `available` only (Q-bootstrap-1), `status: 'draft'` (Q-bootstrap-3).
- Implement `axiom bootstrap from-code --level minimal|basic [--role
  <role>]`, wired into `apps/cli`, printing a review-requirement
  reminder on completion.
- Resolve the three open questions from the audit (Q-bootstrap-1
  through Q-bootstrap-3), each with documented rationale.
- Tests for the extracted writer helper, the analysis/draft/index logic,
  and the CLI command's happy path.

## Non-goals

- No Level 2 (Standard), Level 3 (Deep), or Level 4 (Exhaustive)
  exhaustiveness — explicitly deferred per the audit.
- No topology-based role/repo-kind auto-detection (`@axiom/topology`
  `roleCodeRepositories` mapping) — Level 0/1 uses a single, explicit
  `--role` flag (default `'repo'`) instead of inferring roles from
  topology, to keep the analysis mechanical and avoid adding topology
  resolution complexity not required by this scope.
- No `axiom bootstrap promote` (or any other mechanism to flip
  `status: 'draft'` to `'reviewed'` or populate `mandatory.*`) — that is
  a human curation step explicitly deferred per Q-bootstrap-1's
  rationale and the audit's brief.
- No fix to `@axiom/technical-context`'s pre-existing referential-
  integrity gap (index `path` fields are not checked against the
  filesystem by the loader/validator) — this increment avoids the gap by
  construction (writing documents and index in the same pass with
  consistent paths), but does not change the loader/validator itself.
- No Via B (`bootstrap-from-legacy-sdd`) — that remains INC-19.
- No new standalone package (e.g. `@axiom/bootstrap-from-code`) — the
  analysis/draft/index-builder logic lives inside `apps/cli` as CLI-only
  glue modules (`apps/cli/src/bootstrap-from-code/*`), since there is no
  other consumer today and creating a new package would be speculative
  infrastructure per `Axiom.SDD/AGENTS.md`'s "Explicit Bootstrap Limits."

## Acceptance criteria

- [x] `writeGuardedFile(projectRoot, relativePath, content)` extracted
      into `@axiom/document-bootstrap/src/guarded-write.ts`, exported
      from the package barrel.
- [x] `writeCopilotInstructions` refactored to use the extracted
      path-guard (`resolveGuardedPath`) and atomic-write
      (`writeGuardedFile`) helpers, with the pre-existing
      `writer.test.ts` suite (4 tests) passing unchanged (confirms no
      behavior change).
- [x] Level 0/1 mechanical analysis implemented (`apps/cli/src/
      bootstrap-from-code/analyzer.ts`): stack detection, top-level repo
      map with purpose guesses, declared commands.
- [x] Drafted markdown documents carry an explicit `AXIOM:DRAFT` marker
      (Q-bootstrap-2), written via the extracted writer helper.
- [x] `TechnicalContextIndex` YAML generated per role, `available`-only
      (Q-bootstrap-1: `mandatory.always`/`mandatory.whenTags` always
      empty arrays), `status: 'draft'` (Q-bootstrap-3).
- [x] `axiom bootstrap from-code --level minimal|basic [--role <role>]`
      implemented and wired into `apps/cli/src/index.ts`, printing the
      addendum-15 review-requirement reminder on completion.
- [x] Tests written: extracted writer helper (path-guard rejection,
      atomic write, overwrite), Level 0/1 analysis logic (stack/repo-map/
      commands), drafter (banner presence, minimal vs. basic content),
      index-builder (available-only, status:draft, shape validates),
      CLI command happy path (including project-not-resolved and
      no-spec-scope error paths).
- [x] Real validation executed: `npm run typecheck`, `npm run build`,
      `npm test` (full monorepo + scoped to touched packages), baseline
      re-confirmed via isolated re-run of the one non-deterministic
      failure.
- [ ] `validator-reviewer` pass (next role, separate increment/pass) —
      not part of this increment's closure per the audit's own role
      sequencing.

## Open questions — resolutions

### Q-bootstrap-1: `available`-only vs. `mandatory.*` population

**Resolution: `available`-only. `mandatory.always`/`mandatory.whenTags`
are always emitted as empty arrays.**

Confirmed by the audit and re-confirmed during implementation: declaring
a document "mandatory reading" for a role is a human curation judgment
(it changes what an agent is forced to read before acting), not a
mechanical fact derivable from folder names or manifest contents. A
bootstrap tool populating `mandatory.*` unilaterally would silently
change agent behavior without human sign-off, contradicting addendum
section 15's "never auto-approved" requirement in spirit even if the
*document content* itself is correctly marked draft. `mandatory.*`
population is deferred to a future explicit promotion step (e.g. a
possible `axiom bootstrap promote` or manual edit of the generated
index) — out of scope for this increment (see "Non-goals").

### Q-bootstrap-2: in-file draft marker

**Resolution: yes — every drafted markdown document is prefixed with:**

```html
<!-- AXIOM:DRAFT — generated by `axiom bootstrap from-code`. Review before relying on this content; do not treat as approved. -->
```

Rationale for the exact text: it (a) names the generating command
explicitly, so a reader knows how to regenerate/inspect the source, (b)
states the action required ("review before relying on this content"),
matching the CLI's own completion reminder verbatim in spirit, and (c)
explicitly disclaims approval, closing the defense-in-depth gap the
audit flagged (a document copied out of its index context still carries
the warning). It deliberately does **not** reuse `document-bootstrap`'s
literal `AXIOM:GENERATED`/`TEAM:CUSTOM` marker syntax or regexes: those
markers are tied to that package's preserve/merge machinery
(`classifyAndPreserve`), which bootstrap-drafted documents do not use in
this first pass (per the audit's `writer.ts` finding — merge/preserve is
a valuable *future* opt-in mode for re-running bootstrap against
human-edited files, not needed for a first pass writing new files). The
marker constant (`AXIOM_DRAFT_MARKER`) is defined locally in
`apps/cli/src/bootstrap-from-code/drafter.ts` as a plain string, with no
import dependency between `document-bootstrap` and the CLI's bootstrap
module beyond the shared `writeGuardedFile` primitive.

### Q-bootstrap-3: where draft/review state lives for the index

**Resolution: option (a) from the audit — added an optional
`status?: 'draft' | 'reviewed'` field to `TechnicalContextIndex`.**

Before committing to this, the actual schema/validation code
(`Axiom/packages/technical-context/src/technical-context-index.ts`) was
read in full to confirm the audit's "if cheap to extend additively"
condition. Confirmed cheap: the file already has an established,
literal pattern for adding an optional field (`projectId?: string`,
present since INC-10) — the same 3-part change was replicated for
`status`:

1. Add `readonly status?: TechnicalContextIndexStatus` to the
   `TechnicalContextIndex` interface (new `TechnicalContextIndexStatus =
   'draft' | 'reviewed'` type alias).
2. Add one validation branch in `validateTechnicalContextIndex` (closed
   two-value enum check), mirroring the existing `projectId`
   non-empty-string check immediately above it.
3. Add one conditional spread in the `ok` return value, mirroring the
   existing `...(parsed['projectId'] !== undefined ? {...} : {})`
   pattern immediately above it.

No changes were needed to `loadTechnicalContextIndex`,
`resolveMandatoryDocuments`, `technicalContextIndexPath`, or any
existing caller (`@axiom/mcp-tools`'s `technical-context-handlers.ts`,
`@axiom/doctor`'s checks) — all read `TechnicalContextIndex` structurally
and do not enumerate its fields exhaustively. This is confirmed
additive: every existing fixture/test that omits `status` continues to
parse identically (added a regression test asserting `status` is
`undefined` for the pre-existing worked fixture). Total diff: ~15 lines
across the interface, validator, and barrel export, plus tests — matching
the audit's "cheap to extend additively" expectation. Options (b)
(out-of-band `.review-status.yml`) and (c) (rely on git) were not needed.

## Assumptions

- `--role` (default `'repo'`) stands in for the topology-based
  role/repo-kind resolution the audit left open ("Target repo/scope
  resolution... is the next role's task"). A single explicit flag,
  defaulting to a generic name, was chosen over inferring roles from
  `@axiom/topology`'s `roleCodeRepositories` because Level 0/1 analysis
  is scoped to "one mechanical pass over one repo" — topology-based
  multi-repo role inference is a materially different, larger piece of
  work not required by the audit's Level 0/1 scope, and would risk
  scope creep into topics (multi-repo role mapping) the audit explicitly
  left to a "next role's task" without mandating it be solved now.
- The repo analyzed defaults to the resolved project's product root
  (`resolution.rootPath`), overridable via a `repoPath` argument on the
  underlying `runBootstrapFromCode` function (no CLI flag exposed for it
  yet — YAGNI until a concrete multi-repo bootstrap need arises; the
  function signature already supports it for a future CLI flag with no
  further refactor).
- `document-bootstrap`'s `canonical-agents-md.ts` (which independently
  re-implements the same path-guard + atomic-write pattern inline,
  predating this increment) was deliberately **not** refactored to use
  the newly extracted `writeGuardedFile` helper — the audit's brief
  scoped the extraction specifically to `writer.ts`, and refactoring an
  unrelated, already-working, already-tested file is out of scope per
  `Axiom.SDD/AGENTS.md`'s "do not modify unrelated files." Flagged here
  as a legitimate future cleanup, not implemented now.

## Implementation notes

### Files changed

**`Axiom/packages/document-bootstrap/`** (writer extraction):

- `src/guarded-write.ts` (new) — `resolveGuardedPath` and
  `writeGuardedFile`, the extracted path-guard + atomic-write primitive.
- `src/writer.ts` — refactored `writeCopilotInstructions` to call the
  extracted helpers instead of inlining the same logic; same error
  shapes, same order of operations, no behavior change.
- `src/types.ts` — added `GuardedWriteError`/`GuardedWriteResult` types.
- `src/index.ts` — barrel exports for the new helper + types.
- `tests/guarded-write.test.ts` (new) — 7 tests: cold write, atomic
  write (no `.tmp` residual), overwrite, path-guard rejection (`..`
  escape and absolute-outside-root), `resolveGuardedPath` direct tests.

**`Axiom/packages/technical-context/`** (Q-bootstrap-3 schema extension):

- `src/technical-context-index.ts` — added `TechnicalContextIndexStatus`
  type, `status?` field on `TechnicalContextIndex`, one validation
  branch, one conditional spread in the parse result.
- `src/index.ts` — barrel export for `TechnicalContextIndexStatus`.
- `tests/technical-context-index.test.ts` — 3 new tests: `status`
  absent (pre-existing fixture unaffected), `status: draft` parses,
  `status` with an invalid value is rejected.

**`Axiom/apps/cli/`** (Level 0/1 analysis + CLI command):

- `src/bootstrap-from-code/analyzer.ts` (new) — `detectStacks`,
  `buildRepoMap`, `detectCommands`, `analyzeRepo`. Pure, filesystem-read
  only, no AST parsing.
- `src/bootstrap-from-code/drafter.ts` (new) — `draftDocuments`,
  `AXIOM_DRAFT_MARKER`. Turns a `RepoAnalysis` into 4 markdown documents
  (overview, repo-map, stack, commands) per level.
- `src/bootstrap-from-code/index-builder.ts` (new) —
  `buildTechnicalContextIndex`. Turns drafted documents into a
  `TechnicalContextIndex` value (available-only, status: draft).
- `src/commands/bootstrap.ts` (new) — `runBootstrapFromCode` (pure,
  testable) + `registerBootstrap` (commander wiring). Resolves the
  project and spec scope (same `specScope` resolution pattern as
  `@axiom/doctor`'s `checks.ts`), orchestrates analyze -> draft -> write
  -> index, prints the review reminder.
- `src/index.ts` — registered `registerBootstrap(program)`.
- `package.json` — added `@axiom/technical-context` dependency (was
  missing entirely; needed for `technicalContextIndexPath` and
  `TechnicalContextIndex` types).
- `tsconfig.json` — added `@axiom/technical-context` to `paths` and
  `references` (same gap: the package existed but was never wired into
  `apps/cli`'s TypeScript project references).
- `tests/bootstrap-from-code/analyzer.test.ts` (new) — 17 tests.
- `tests/bootstrap-from-code/drafter.test.ts` (new) — 8 tests.
- `tests/bootstrap-from-code/index-builder.test.ts` (new) — 4 tests.
- `tests/bootstrap.test.ts` (new) — 7 tests: not-resolved, no-spec-scope,
  minimal happy path (documents + index shape, paths), basic happy path
  (mechanical content from a real `package.json`), review reminder text.

### CLI surface implemented

```
axiom bootstrap from-code --level minimal|basic [--role <role>]
```

- `bootstrap` namespace, `from-code` verb, matching the audit's design.
- `--level` closed to `minimal|basic` (rejects any other value with a
  clear error referencing the audit's Level 2+ deferral). Default:
  `minimal`.
- `--role` (not in the audit's original flag list; added during
  implementation per its own "resolving the exact flag list... is this
  role's own implementation task" instruction) names the document
  set/index (`<role>.index.yml`, `technical-context/<role>/*.md`).
  Default: `'repo'`.
- No `--dry-run` flag implemented yet — the audit noted it as
  "recommended... but not decided as in-scope," and no concrete
  requirement forced it in for this pass; flagged as a reasonable
  near-term follow-up, not implemented now (YAGNI).

### Referential-integrity-by-construction

Per the audit's flagged gap (the index loader/validator does not check
that `available[].path` resolves to a real file), `runBootstrapFromCode`
writes every drafted document AND the index in the same function call,
deriving `available[].path` directly from the same `DraftedDocument`
values used to write the files (`buildTechnicalContextIndex`'s
`documentPathPrefix` parameter is the only manually-kept-consistent
piece — a single `'..'` literal matching INC-10's one-level-up
convention). If a future document set changes its relative path
convention, this is the one place that would need to change in lockstep;
documented here for the validator-reviewer's benefit.

## Validation

Validation discovery order followed: checked `package.json` scripts at
the repo root (`Axiom/package.json`) — `build` (`tsc -b`), `test`
(`vitest run`), `typecheck` (`tsc -b`) all exist and were run for real.

**Baseline confirmed before any change** (per the brief's request to
double-check, given INC-15's validator previously found a real
regression):

```
npm run build       → clean (tsc -b, no output = success)
npm test             → Test Files: 12 failed | 139 passed (151)
                       Tests:      13 failed | 1513 passed (1526)
```

The 12 failing files are pre-existing, unrelated snapshot/timing issues
(TUI ANSI-art snapshot mismatches, a gateway-state drift test) —
confirmed by reading representative failure output; none touch
`document-bootstrap`, `technical-context`, or `apps/cli`'s bootstrap
surface.

**Post-implementation:**

```
npm run typecheck    → clean (tsc -b, no output = success)
npm run build        → clean (tsc -b, no output = success)
npm test              → Test Files: 12 failed | 144 passed (156)
                        Tests:      13 failed | 1559 passed (1572)
```

The failing-file/test counts match the baseline exactly (12/13). One
run produced 13 failed files / 14 failed tests due to a Windows-specific
`EPERM: operation not permitted, rename ...` race in
`packages/versioning/tests/checkpoints.test.ts` (a pre-existing,
non-deterministic file-lock/antivirus-contention issue under full-suite
parallel load, unrelated to any file touched by this increment) — this
test was re-run in isolation and passed cleanly (9/9), and a subsequent
full-suite re-run returned to the exact 12/13 baseline, confirming the
extra failure was transient flakiness, not a regression introduced by
this increment.

Scoped test runs (all passing, all new/touched suites):

```
npx vitest run packages/document-bootstrap                        → 5 files, 42 tests passed
npx vitest run packages/technical-context                          → 1 file, 29 tests passed
npx vitest run apps/cli/tests/bootstrap-from-code apps/cli/tests/bootstrap.test.ts
                                                                     → 4 files, 36 tests passed
```

Net new tests added: 7 (guarded-write) + 3 (technical-context status) +
36 (bootstrap-from-code + CLI) = 46, consistent with the baseline-to-
post-implementation passed-test delta (1513 → 1559, net +46).

**Manual CLI smoke test (informational, not part of the pass/fail
count):** ran `node apps/cli/dist/index.js bootstrap from-code --level
basic --role backend` against a scratch project. It failed with
`Cannot find module './_shared'` inside `dist/commands/init.js` — this
is the pre-existing, already-documented "Known build-tooling defect"
in `general-spec.md` (`packages/cli-commands/tsconfig.json` cross-includes
source files that should belong to `apps/cli`'s own composite build,
breaking the built CLI for a broad set of commands including baseline
ones like `init`, confirmed unrelated to any specific feature increment).
Reproduced the same failure on the pre-existing `axiom index rebuild`
command to confirm it is not specific to `bootstrap.ts`. Per that
section's own note ("Does not affect vitest-based tests"), the real
test harness (`npx vitest run apps/cli/tests/bootstrap.test.ts`)
exercises `runBootstrapFromCode` end-to-end against a real filesystem
and passes; this is the validation channel this repository actually
relies on for `apps/cli` today.

## Result

Implemented the `bootstrap-implementer` role's full brief: extracted
`writer.ts`'s path-guard + atomic-write primitive into
`writeGuardedFile`/`resolveGuardedPath` (`@axiom/document-bootstrap`),
refactored `writeCopilotInstructions` to use it with zero behavior
change (confirmed via its pre-existing, unmodified test suite still
passing). Implemented Level 0 (Minimal)/Level 1 (Basic) mechanical
analysis (stack detection, repo map, declared commands) as pure,
testable modules under `apps/cli/src/bootstrap-from-code/`. Drafted
markdown documents carry an `AXIOM:DRAFT` banner (Q-bootstrap-2).
Generated `TechnicalContextIndex` YAML populates `available` only
(Q-bootstrap-1) with `status: 'draft'` (Q-bootstrap-3, implemented as a
confirmed-cheap additive schema extension to
`@axiom/technical-context`). Shipped `axiom bootstrap from-code --level
minimal|basic [--role <role>]`, wired into `apps/cli`, printing the
addendum-15 review reminder on every successful run. All three open
questions from the audit were resolved with documented rationale before
or during implementation, per the audit's own brief ordering. Real
validation (`typecheck`, `build`, `test`) was executed against a
freshly-confirmed baseline; the only test-count deviation observed
across multiple runs was a one-off, unrelated, Windows-specific
filesystem race, independently confirmed non-reproducible and unrelated
to any file this increment touched.

Status is `pending`, not `closed`: per the audit's own role sequencing
and this repository's closure rules, a `validator-reviewer` pass — an
independent, fresh-context check that draft/review marking is present
and correct, the generated `TechnicalContextIndex` validates against
`validateTechnicalContextIndex`, every `available[].path` resolves to an
actually-written file, and no `mandatory.*` entries were populated —
has not yet run. This increment's own acceptance criteria are otherwise
fully met (implementation done, validated, reviewed against intent), but
the audit's explicit hand-off condition ("continue with
`validator-reviewer`... After bootstrap-implementer") has not been
satisfied yet, so closure is deferred to that pass rather than declared
here.

## General spec integration

Added a new "Bootstrap-from-code (Level 0/1)" section to
`Axiom.Spec/general-spec.md`, mirroring the existing "Technical-context
index" section's structure: summarizes the shipped CLI surface
(`axiom bootstrap from-code --level minimal|basic [--role <role>]`), the
`available`-only/`status: draft`/`AXIOM:DRAFT`-marker invariants, the
`writeGuardedFile` primitive's existence and intended reuse for future
generators, and a cross-reference to both this increment and the
migration-engineer audit's scope-minimization reasoning (Level 0/1 only,
collapsed roles, Level 2+ deferred) so a future reader understands why
the shipped surface deliberately does not implement source doc 15.3's
literal 7-subagent chain.

## Next step recommendation

Hand off to **validator-reviewer** with these exact inputs:

- This increment's spec (file you are reading).
- The audit increment:
  `Axiom.Spec/specs/increments/INC-20260702-bootstrap-from-code-reconcile/README.md`.
- Files to review: `Axiom/packages/document-bootstrap/src/guarded-write.ts`,
  `Axiom/packages/document-bootstrap/src/writer.ts`,
  `Axiom/packages/technical-context/src/technical-context-index.ts`,
  `Axiom/apps/cli/src/bootstrap-from-code/*.ts`,
  `Axiom/apps/cli/src/commands/bootstrap.ts`.
- Specific checks per the audit's brief item 8: (a) draft/review marking
  present and correct (`AXIOM:DRAFT` banner in every drafted document,
  `status: 'draft'` in every generated index), (b) generated
  `TechnicalContextIndex` output validates against
  `validateTechnicalContextIndex` (can be spot-checked by running
  `axiom bootstrap from-code` against a scratch project and loading the
  result), (c) every `available[].path` resolves to an actually-written
  file (referential integrity, for this command's own output only — not
  a general fix to the loader/validator), (d) no `mandatory.*` entries
  were populated (both arrays empty in generated output).
- After validator-reviewer, this increment (and INC-18 overall, pending
  the parent roadmap's own tracking) can be considered for `closed`
  status per this repository's closure rules.
