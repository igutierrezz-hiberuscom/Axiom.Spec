# Increment: Reconcile increment/bug/plan commands (validator-reviewer)

Status: closed
Date: 2026-07-02

## Goal

Execute the **validator-reviewer** step of INC-06 of the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`),
independently re-verifying the migration-engineer audit, schema-writer
design, and cli-implementer implementation for INC-06, then close INC-06
as a whole if all acceptance criteria are met.

## Context

Parent chain (all read in full before this review):

1. migration-engineer audit —
   `Axiom.Spec/specs/increments/INC-20260702-increment-bug-plan-commands-reconcile/README.md`.
2. schema-writer design —
   `Axiom.Spec/specs/increments/INC-20260702-increment-bug-plan-commands-reconcile-design/README.md`.
3. cli-implementer implementation —
   `Axiom.Spec/specs/increments/INC-20260702-increment-bug-plan-commands-reconcile-impl/README.md`.

This document does not re-litigate any of the three prior steps' own
decisions; it independently re-derives the same conclusions (or flags a
divergence) from the actual code and test run, per this role's mandate
("validator," not "re-implementer").

## Scope

- Independent code review of `packages/workflow/src/artifact-id.ts` and
  `artifact-store.ts` (format, collision logic, termination bound, schema
  fidelity to the design).
- Independent trace of the bidirectional link resolution in
  `apps/cli/src/commands/artifact-metadata-cli.ts` and its three call
  sites (`axiom-increment.ts`, `axiom-bug.ts`, `axiom-plan.ts`).
- `git diff` on `@axiom/workflow`'s core files
  (`state-machine.ts`, `state-store.ts`, `hooks.ts`, `branch-naming.ts`,
  `types.ts`, `workflows-loader.ts`) and `axiom-role.ts` to confirm the
  "unmodified" claim directly, not by trusting the prior spec's report.
- Re-analysis of `axiom-role.ts`'s `checkPlanIsApproved` gate against the
  new multi-plan-folder reality.
- Independent full-suite validation run (`npm run typecheck`, `npm run
  build`, `npm test`) plus a `git stash`/`git stash pop` bisection to
  confirm the pre-existing failure baseline.
- INC-06-as-a-whole closure decision, naming any deferred items from the
  migration-engineer's original command-surface gap table that remain
  unaddressed.

## Non-goals

- No new code changes. This is a review-and-validate pass; any bug found
  would be reported here, not silently patched, unless trivial and
  explicitly noted (none were found — see Result).
- No re-opening of the schema-writer's design decisions (folder layout,
  `metadata.yml` shape, state-machine-untouched decision) — those were
  independently re-confirmed against the code, not re-debated.
- No implementation of the deferred items identified below (`update`/
  `status` commands, `workflow-state.json` migration, multi-repo
  `allowedWriteScope` enforcement) — these are named as tracked
  follow-ups, not implemented in this increment.

## Acceptance criteria

- [x] `artifact-id.ts`'s ID format independently confirmed against
      addendum 11's exact example.
- [x] Collision check-and-retry logic independently confirmed correct,
      with an explicit termination-bound check (not just "present").
- [x] `artifact-store.ts`'s `metadata.yml` schema and atomic write logic
      independently confirmed against the schema-writer's exact designed
      shape, field by field.
- [x] Bidirectional link resolution independently traced through code
      (not just re-read from the impl spec's prose), including the
      missing-target-metadata warning path.
- [x] `git diff` run directly on `@axiom/workflow`'s core state-machine
      files and `axiom-role.ts` to confirm zero modification.
- [x] `axiom-role.ts`'s `checkPlanIsApproved` gate re-analyzed against
      the new multi-plan-folder reality, with an explicit regression/
      no-regression conclusion and reasoning.
- [x] Full monorepo validation independently re-run
      (`typecheck`/`build`/`test`), with a stash-based bisection
      confirming the pre-existing baseline.
- [x] INC-06-as-a-whole closure assessed, with deferred items named
      explicitly (not silently dropped).

## Open questions

None blocking closure. See "Deferred items" in Result/Closure summary
below for tracked (not blocking) follow-up items.

## Assumptions

- The pre-existing uncommitted, unrelated working-tree changes in
  `Axiom` (`apps/cli/src/commands/tui.ts`,
  `packages/filesystem-truth/**`, `packages/installer/src/registry.ts`,
  `packages/project-resolution/**`, `packages/user-workspace/**`) are
  out of scope for this review — they predate and are unrelated to
  INC-06 (confirmed by `git diff --stat` showing no overlap with any
  file INC-06's chain claims to touch, and by cli-implementer's own
  scope section never mentioning them). They were left untouched and
  unreviewed here, consistent with "do not modify/review unrelated
  files."

## Implementation notes

### 1. `artifact-id.ts` — independent review

Read `Axiom/packages/workflow/src/artifact-id.ts` in full.

**Format**: `generateArtifactId` produces
`${ARTIFACT_ID_PREFIX[kind]}-${yyyy}${mm}${dd}-${hh}${min}${ss}-${randomSuffix()}`.
For `kind='increment'` this yields e.g. `INC-20260702-183012-a7f3c9` —
byte-for-byte the same shape as addendum 11's worked example (prefix,
`-`, 8-digit UTC date, `-`, 6-digit UTC time, `-`, 6-char suffix).
Confirmed correct: all six numeric fields are `padStart`-padded to their
documented width (`yyyy` to 4, all others to 2), so the format cannot
produce a short/malformed segment even for single-digit months/days/
hours/minutes/seconds. `randomSuffix`'s default
(`Math.random().toString(36).slice(2, 8).padEnd(6, '0')`) always yields
exactly 6 base36 characters (padded on the right if `toString(36)`
produces fewer than 6 post-slice characters, which can happen for small
random values) — correct, not just "present."

**Collision check-and-retry — independently verified, not just present**:
`generateUniqueArtifactId(kind, exists, options)` loops
`for (let attempt = 0; attempt < maxAttempts; attempt += 1)`, calling
`generateArtifactId(kind, options)` fresh on every iteration (note: this
re-reads `options.now`/`options.randomSuffix` each time, so if the
caller injects a fixed `now`/`randomSuffix` for deterministic tests, the
loop would spin without ever changing its candidate — this is expected
test behavior, not a runtime bug, since production callers use the
default `Math.random()`-based `randomSuffix`, which differs per call by
construction). If `!exists(id)`, returns immediately. If all
`maxAttempts` (default 10) iterations collide, the loop naturally
terminates and throws an `Error` with a clear message — **this is a
bounded loop, not a potential infinite spin**: `attempt` is a plain
incrementing counter compared against a fixed, finite `maxAttempts`,
with no path that resets or extends it. The only way this function could
loop forever is if `maxAttempts` were `Infinity` (impossible via the
type signature, which is `number` with a numeric default) or if a caller
passed `Number.POSITIVE_INFINITY` explicitly (an intentional misuse, not
a design flaw). Edge case checked: `maxAttempts: 0` would skip the loop
body entirely and throw immediately — correct fail-fast behavior, not a
hang. **Conclusion: correct, bounded, no infinite-loop risk under any
non-adversarial input.**

### 2. `artifact-store.ts` — independent schema review

Read `Axiom/packages/workflow/src/artifact-store.ts` in full and
compared field-by-field against the schema-writer design's three worked
YAML examples (shared base, increment/bug `links`, plan `links` +
`targetRepos`/`taskType`/`allowedWriteScope`).

**Shared base** (`BaseArtifactMetadata`): `id`, `kind`, `title`,
`status` (typed as `WorkflowState`, imported from `./types` — exactly
the design's "reuse `WorkflowState`, don't invent a second vocabulary"
decision), `createdAt`, `updatedAt`, `externalRefs`. All six fields
present, none renamed, none added beyond the design. `ExternalRef`
(`provider`, `type`, `id`, `url?`) matches addendum 11's shape verbatim,
including the deliberate "always string, even if provider's native ID
is numeric" convention (typed `readonly id: string`, no numeric union).

**Increment/bug** (`IncrementMetadata`, `BugMetadata`): base +
`links: { planId?: string | null }` — matches the design's `links:
{ planId }` exactly, same field name for both kinds as designed.

**Plan** (`PlanMetadata`): base + `links: { incrementId?, bugId? }` +
`targetRepos: ReadonlyArray<string>` + `taskType: string` +
`allowedWriteScope: ReadonlyArray<{ repo: string; paths:
ReadonlyArray<string> }>` — matches the design's plan-only fields
exactly, same names, same shapes, `allowedWriteScope` entries carrying
`repo`/`paths` exactly as addendum 9's worked example.

**No field dropped, renamed, or added beyond the design.** The only
things not explicitly in the design's code sample are implementation
plumbing the design's prose implied but didn't spell out as code:
`ArtifactStoreError` (a `Result`-style discriminated union consistent
with `state-store.ts`/`workflows-loader.ts`'s existing error-handling
convention — not a schema field, an internal error type), and the
runtime shape-validation functions (`parseBaseFields`, `isExternalRef`,
`isAllowedWriteScopeEntry`) that reject malformed YAML at load time —
both are implementation necessities for a YAML *reader*, not scope
creep on the persisted schema itself.

**Atomic write** (`saveArtifactMetadata`): `fs.mkdirSync(artifactDir,
{ recursive: true })` → `yaml.dump(...)` → `fs.writeFileSync(tmpPath,
...)` → `fs.renameSync(tmpPath, filePath)`, wrapped in try/catch
returning `err({ kind: 'io-error', ... })` on failure. This is the exact
tmp+rename pattern both the design and impl spec claim, independently
confirmed by reading the function body (not by trusting either prior
spec's description). `ensureArtifactReadme` checks
`fs.existsSync(filePath)` before writing and returns early — confirmed
it genuinely never overwrites an existing `README.md`.

**Conclusion: schema and atomic write logic confirmed correct and
faithful to the design, with no undocumented additions.**

### 3. Bidirectional link resolution — independently traced

Read `Axiom/apps/cli/src/commands/artifact-metadata-cli.ts`'s
`runLinkCommand` in full and traced its call sites in
`axiom-increment.ts` (registers `link-plan`, `ownLinkField: 'planId'`,
`reverseLinkField: undefined` — see note below), `axiom-bug.ts` (same
shape), and `axiom-plan.ts` (registers `link-increment` and `link-bug`,
each with `reverseLinkField: 'planId'`).

Traced execution for `axiom increment link-plan --id INC-x --target-id
PLAN-y` where `PLAN-y`'s `metadata.yml` does **not** exist (the exact
scenario requested for conceptual testing):

1. `loadArtifactMetadata(projectRoot, 'increment', 'INC-x')` — assume
   this succeeds (increment already created).
2. Own metadata updated in-memory (`links.planId = 'PLAN-y'`,
   `updatedAt` bumped) and saved via `saveArtifactMetadata` — **this
   write happens unconditionally, before the target is ever touched.**
3. Because `axiom-increment.ts` registers this subcommand via
   `addLinkPlanSubcommand(cmd, { ..., reverseLinkField: undefined })`
   (increment/bug's registration omits `reverseLinkField` — the reverse
   direction is instead registered from the *plan* side, as
   `plan link-increment`/`plan link-bug`, avoiding a double-write race
   where both `increment link-plan` and `plan link-increment` would
   each try to update both sides), `args.reverseLinkField !==
   undefined` is `false`, so the target-side block is skipped entirely
   and the command returns `exitCode: 0` with no warning.
4. Conversely, tracing `axiom plan link-increment --id PLAN-y --target-id
   INC-x` where `INC-x` exists but `PLAN-y` does not yet exist (plan
   metadata not created): step 2 (own side, on `plan`) fails at
   `loadArtifactMetadata` returning `ok(undefined)` (not an error, just
   "not found"), which triggers the **already-existing** "No existe
   metadata.yml... Corré '... create' primero." early return with
   `exitCode: 1` — this is the *own*-side-missing path, not the
   target-missing path.
5. Tracing the actual target-missing warning path: `axiom plan
   link-increment --id PLAN-y --target-id INC-missing` where `PLAN-y`
   exists but `INC-missing` does not: own side (`plan`) loads fine, own
   side is updated and saved successfully (`links.incrementId =
   'INC-missing'`), THEN `reverseLinkField` is `'planId'` (plan's
   registration DOES set `reverseLinkField`), so
   `loadArtifactMetadata(projectRoot, 'increment', 'INC-missing')` runs,
   returns `ok(undefined)` (`targetLoad.ok === true` but
   `targetLoad.value === undefined`), which hits the `else` branch:
   `warning = " (warning: increment 'INC-missing' no tiene metadata.yml
   todavía; sólo se actualizó el lado de plan)"`. Final result:
   `exitCode: 0` (own side succeeded), message includes the non-fatal
   warning appended. **This exactly matches the impl spec's claimed
   behavior; independently confirmed by tracing the actual branch
   logic, not by re-reading the impl spec's prose.**

**One nuance worth flagging (not a bug, a design asymmetry worth
naming)**: because only the `plan`-side registrations set
`reverseLinkField`, calling `increment link-plan` or `bug link-plan`
alone never attempts (or warns about) the reverse write — the
increment/bug side is genuinely one-directional at that specific call
site. Full bidirectionality is achieved only if the `plan`-side
`link-increment`/`link-bug` command is also run (or was already run).
This is consistent with the impl spec's own stated rationale ("avoiding
a two-write race" is not stated, but avoiding *duplicate* reverse writes
if both directions were symmetric is a sound reason) and does not
contradict the impl spec's claim of "bidirectional, best-effort" — the
bidirectionality is achieved by the *pair* of commands together
(`increment link-plan` + `plan link-increment`), with the actual
reverse-write mechanics living only on the `plan`-side call. This is a
reasonable, working design, not a defect — flagged here only so it's
explicit rather than assumed, since the impl spec's prose reads as if
either single command would fully sync both sides, which is not quite
literally true per-call (it is true per-pair, and the natural CLI usage
pattern of running the plan-side command achieves it).

**Conclusion: link resolution code confirmed to work as described,
including the missing-target-metadata warning path, with one asymmetry
noted above for completeness (not a bug).**

### 4. `@axiom/workflow` core files — `git diff` confirmed unchanged

Ran `git diff --stat` directly (not trusting the impl spec's "confirmed
by inspection" claim) against:

```
packages/workflow/src/state-machine.ts
packages/workflow/src/state-store.ts
packages/workflow/src/hooks.ts
packages/workflow/src/branch-naming.ts
packages/workflow/src/types.ts
packages/workflow/src/workflows-loader.ts
apps/cli/src/commands/axiom-role.ts
```

**Zero diff output for all seven files** — confirmed byte-for-byte
unchanged. `packages/workflow/src/index.ts` (the barrel) does have a
diff, but independently reviewed and confirmed it is purely additive:
32 new lines, all `export { ... } from './artifact-id'` /
`'./artifact-store'` statements; no existing export line was touched,
removed, or reordered.

### 5. `axiom-role.ts`'s `checkPlanIsApproved` gate — re-analysis

Re-read `checkPlanIsApproved` (`axiom-role.ts` lines 257-298) and its
call site (`runRoleSubcommand`, gating the `start` subcommand) in full.

**Finding: this is NOT a regression.** The gate calls
`loadWorkflowState(projectRoot, PLAN_WORKFLOW_ID)` — reading
`workflow-state.json`'s single shared `plan` record via
`@axiom/workflow`'s existing, completely untouched `state-store.ts`
mechanism. It checks `planRecord.state === 'plan-approved'`. This
mechanism has **no dependency on plan-folder existence at all** — it
never looked at a filesystem folder before INC-06, and it still doesn't
after INC-06, because `axiom-plan.ts`'s `approve` command still calls
`applyTransition`/`saveWorkflowState` at the exact same call sites with
the exact same arguments as before (independently confirmed by reading
`runPlanApprove`'s body: `applyTransition(config, currentRecord.state as
WorkflowState, TRANSITION_COMMAND)` at line 301, `saveWorkflowState(args.
projectRoot, newRecord)` at line 339 — neither touched by this
increment).

The scenario the task description asked to consider — "what happens now
that multiple plans can exist as folders" — does not change this gate's
behavior at all, because the gate's semantics were **already** "the
most recently approved plan according to `workflow-state.json`'s single
shared record," not "is there a plan folder that is approved." Multiple
plan *folders* (the new `metadata.yml` instances) can coexist freely,
but `workflow-state.json`'s `plan` key still holds exactly one record —
whichever plan's `approve` call most recently ran — exactly as it did
before this increment (this was already the migration-engineer audit's
own finding: "this only supports one *active* plan at a time — approving
a second plan overwrites the first plan's record"). INC-06 did not
change that pre-existing single-active-plan limitation; it added a
parallel, multi-instance metadata layer *alongside* it without altering
the gate's data source.

**Conclusion, matching option (b) framed in the task**: the existing
gate's semantics were already scoped to something orthogonal to
plan-folder existence (`workflow-state.json`'s single active-plan-id
mechanism), and that mechanism is genuinely untouched by this
increment. No fix is needed. This matches both the schema-writer
design's explicit statement ("this means `axiom-role.ts`'s existing
`checkPlanIsApproved` gate... keeps working unmodified... it simply
becomes explicit that it checks 'the most recently active plan'") and
the cli-implementer's claim, and is independently re-derived here from
the actual code rather than accepted on either prior spec's word.

## Validation

Real validation executed (monorepo root, `Axiom/package.json` scripts):

```
npm run typecheck  → clean (tsc -b, no errors)
npm run build      → clean (tsc -b, no errors)
npm test           → Test Files  12 failed | 120 passed (132)
                      Tests       13 failed | 1298 passed (1311)
```

Failed test files/tests (independently listed from this run's own
output, not copied from the impl spec):

```
apps/cli/tests/start.test.ts (1)
packages/agents/tests/catalog.test.ts (1)
packages/doctor/tests/checks.test.ts (1)
packages/model-routing/tests/assignments.test.ts (1)
packages/model-routing/tests/loader.test.ts (1)
packages/model-routing/tests/opencode-projection.test.ts (1)
packages/model-routing/tests/resolver.test.ts (1)
packages/project-resolution/tests/resolver.test.ts (1)
packages/skills/tests/catalog.test.ts (1)
packages/telemetry/tests/audit-trail-sink.test.ts (1)
packages/toolchain/tests/repair-add-gitignore.test.ts (1)
packages/tui/tests/driver.test.ts (2)
```

12 files, 13 tests — matches the impl spec's claimed baseline exactly.
None touch `axiom-increment.ts`, `axiom-bug.ts`, `axiom-plan.ts`,
`axiom-role.ts`, `artifact-metadata-cli.ts`, or any file under
`packages/workflow/`.

**Bisection performed for independent confidence** (not just trusting
the impl spec's before/after numbers): stashed out exactly the INC-06
file set (`artifact-id.ts`, `artifact-store.ts`, `index.ts`,
`workflow/package.json`, `artifact-metadata-cli.ts`, the three modified
CLI command files, and all five new test files) via `git stash push -u
-- <paths>`, leaving the pre-existing unrelated dirty files
(`tui.ts`, `filesystem-truth/**`, etc.) untouched in the working tree.

```
With INC-06 stashed out:
  npm run typecheck → clean
  npm test          → Test Files  12 failed | 114 passed (126)
                       Tests       13 failed | 1254 passed (1267)
```

This is byte-for-byte the same baseline the impl spec claimed
(`12/13/1254`) and the same 12 files by name. `git stash pop` restored
the working tree to its exact prior state (`git status --short` line
count verified identical before/after: 29 lines both times).

**Conclusion: the +44 test delta (1254 → 1298) is fully attributable to
INC-06's new tests, with zero regressions and zero pre-existing tests
silently dropped, independently re-confirmed via bisection, not just
recomputed arithmetic on the impl spec's reported numbers.**

## Result

Independent re-verification of all five review items completed, with no
bugs found requiring a fix:

1. **`generateArtifactId`/`generateUniqueArtifactId`**: format confirmed
   byte-for-byte matching addendum 11's example; collision retry loop
   confirmed correctly bounded (finite `maxAttempts`, throws on
   exhaustion, no infinite-loop path under any non-adversarial input).
2. **`artifact-store.ts`**: `metadata.yml` schema confirmed field-for-
   field identical to the schema-writer's design (no drops, renames, or
   undocumented additions); atomic tmp+rename write confirmed correct
   by reading the function body directly.
3. **Bidirectional link resolution**: traced through the actual code
   for both the missing-target-metadata warning path (confirmed working
   exactly as claimed) and the own-side-missing early-return path.
   Flagged one non-bug nuance: full bidirectionality is achieved by the
   *pair* of increment/bug-side and plan-side commands together, not by
   either single command call in isolation (reverse-write logic lives
   only on the `reverseLinkField`-bearing registration, which is the
   plan-side registrations only) — documented for clarity, not a defect.
4. **Core `@axiom/workflow` files + `axiom-role.ts`**: `git diff`
   confirmed zero changes to `state-machine.ts`, `state-store.ts`,
   `hooks.ts`, `branch-naming.ts`, `types.ts`, `workflows-loader.ts`,
   and `axiom-role.ts`. `index.ts`'s diff confirmed purely additive.
5. **`axiom-role.ts`'s plan-approved gate**: re-analyzed and confirmed
   NOT a regression — the gate was already scoped to
   `workflow-state.json`'s single shared `plan` record (a mechanism
   orthogonal to plan-folder existence), which remains completely
   unmodified. No fix needed.
6. **Full validation**: `typecheck`/`build` clean; `npm test` shows
   1298 passed / 13 failed / 12 failed files, independently re-run and
   bisected via `git stash` to confirm the pre-existing 1254/13/12
   baseline, with the working tree fully restored afterward.

## General spec integration

`Axiom.Spec/general-spec.md` does not exist in this repo (confirmed by
all three prior specs in this chain; the closest equivalents are
`Axiom.Spec/specs/00_Resumen_Ejecutivo.md` through `08_Glosario.md`,
none of which were modified by this review). This validator-reviewer
step is the natural point in the chain where the prior specs deferred
general-spec integration ("consolidating it into a stable, reusable
general spec now, before... review, would risk documenting a shape that
changes" — schema-writer; "this implementation has not yet passed a
validator-reviewer pass" — cli-implementer). Since this review confirms
the design and implementation are correct with no changes required, the
following stable knowledge is now safe to consolidate, but is
**deliberately not written into a new `general-spec.md` file in this
increment**, per `Axiom.SDD/AGENTS.md`'s own instruction to keep
`general-spec.md` "clean, consolidated, and reusable" and this
repository's explicit choice (mirrored by all three prior specs) to use
the `specs/00_*.md`-`08_*.md` documents as the closest existing
equivalent rather than inventing a new file speculatively. Recommended
stable-knowledge summary for whoever next touches those documents (not
executed here, to avoid modifying files outside this increment's
established scope without a dedicated integration pass):

- Increments/bugs/plans are now folder-per-instance
  (`axiom.spec/{increments,bugs,plans}/<ID>/{README.md,metadata.yml}`),
  with system-generated stable IDs (`generateArtifactId`) replacing
  free-text caller-supplied IDs for this new layer.
- `metadata.yml`'s `status` mirrors `@axiom/workflow`'s `WorkflowState`
  but is a separate, per-instance concept; `workflow-state.json` remains
  the single-active-instance-per-workflow-ID mechanism, unchanged, and
  is what `axiom-role.ts`'s plan-approved gate reads.
- `link-plan`/`link-increment`/`link-bug` are bidirectional as a pair of
  commands (own-side write is unconditional; reverse-side write is
  best-effort with a non-fatal warning if the target's `metadata.yml`
  does not yet exist).

## Closure summary — INC-06 as a whole

All four subagent steps of INC-06 (migration-engineer audit →
schema-writer design → cli-implementer implementation →
validator-reviewer, this document) are complete. Closure rule checklist
(`Axiom.SDD/AGENTS.md`):

- Goal/expected behavior: clear across all four steps.
- Acceptance criteria: exist and are checked off in all four specs.
- Changes implemented: yes (`artifact-id.ts`, `artifact-store.ts`, three
  CLI command files, `artifact-metadata-cli.ts`, 5 new test files, 44
  new tests), independently re-verified in this document.
- Available validation executed: yes, both by cli-implementer and
  independently re-run + bisected in this document.
- Review against intent and acceptance criteria: done in this document.
- Stable knowledge integration: assessed above (recommended summary
  given; not written into a file, consistent with all three prior
  specs' own reasoning and this repo's not-yet-created
  `general-spec.md`).
- Result documented clearly: yes, across all four specs.

**INC-06 is closed.**

### Deferred items (explicitly named, not silently dropped)

From the migration-engineer's original command-surface gap table
(`INC-20260702-increment-bug-plan-commands-reconcile/README.md`,
"Confirmed exact current command surface" table), the following gaps
were identified but **not** addressed by cli-implementer, and are not
addressed here either — named explicitly as tracked follow-ups for a
future increment, not silently dropped:

1. **`update`/`status` commands** (source doc 6.1's assumed generic
   names) do not exist on any of `axiom-increment`, `axiom-bug`, or
   `axiom-plan` today. The existing state-transition subcommands
   (`refine`, `specify`, `change`, `plan`, `plan-approve`, `verify`,
   `archive` for increment; `fix-plan`, `verify`, `archive` for bug) and
   the new `create`/`approve`/`link-*`/`external-ref` subcommands for
   plan cover the functional need (creating, transitioning, linking, and
   annotating artifacts), but no command under the literal name
   `update` or `status` was added, and none was in cli-implementer's
   scope (the task brief that produced the impl spec scoped `plan
   create`, linking commands, and `externalRefs` — not a rename/add of
   generic `update`/`status` verbs). Confirmed independently in this
   review by listing every registered subcommand
   (`.command(...)` call) on all three files.
2. **`axiom.yaml` `scopes.spec.path`/`paths.spec.path` override
   honoring** for the new artifact-store folder resolution — the
   cli-implementer's own Assumptions section states `<specPath>`
   hardcodes `axiom.spec` (matching the four CLI files' existing,
   equally-hardcoded `defaultWorkflowsYamlPath()` behavior), and
   explicitly flags this as a known gap for "a future increment" if
   real scope-path honoring is ever added.
3. **`workflow-state.json` migration for pre-existing in-flight
   increments/bugs/plans** (schema-writer's open question 1) — not
   auto-migrated; running a transition command on a pre-existing
   instance with no `vars.metadataId` is a silent metadata-sync no-op,
   by explicit design default, not a blocking gap, but worth tracking
   if any project actually has such in-flight state today.
4. **`axiom validate changes` / `allowedWriteScope` enforcement** — the
   field exists on plan `metadata.yml` (addendum 9's shape), but no
   validator/command consumes it yet; explicitly out of scope per both
   the design and impl specs, tracked here as a genuine future
   increment (likely INC-14 per the parent roadmap's own dependency
   listing on INC-06).

None of these four items block INC-06's closure — they were each
explicitly scoped out by the prior steps with stated rationale, not
overlooked, and are named here so they are not lost from the roadmap's
tracking.

## Next step recommendation

INC-06 closes cleanly. Per the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
Phase C), the next dependent increment is:

**INC-07 — Reconcile derived caches / index rebuild
(`@axiom/doctor` + any existing index builder)**, which the roadmap
lists as depending on INC-06 and whose own next step is a
migration-engineer audit ("audit whether any package already does cache
rebuild; report back before assuming additive") before any schema or
implementation work begins — following the same lightweight,
audit-first subagent sequence used throughout this chain.
