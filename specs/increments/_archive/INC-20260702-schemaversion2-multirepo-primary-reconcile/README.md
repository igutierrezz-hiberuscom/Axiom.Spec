# Increment: schemaVersion:2 emission + multi-repo-primary reconcile (audit)

Status: pending
Date: 2026-07-03

## Goal

Close the specific deferred thread named explicitly by two consecutive
increments (INC-20, INC-21) as blocking further Phase G work
(per-asset version tracking, per-repo change reporting): finish what
INC-01's own resolved decisions D1 and D3 asked for and never fully
implemented.

- **D3** (`axiom.yaml` schemaVersion 1 -> 2): the write side
  (`apps/cli/src/commands/init.ts`'s `buildAxiomYaml`) still emits v1
  only. The read side (`@axiom/project-resolution`'s `resolveProject`)
  became genuinely version-aware in
  `INC-20260702-registry-manifest-schema-v2-resolver-fix`, and
  `@axiom/doctor`'s MC-001/BC-001/BC-002 became genuinely version-aware
  in `INC-20260702-registry-manifest-schema-v2-validator` — but nobody
  went back to flip the write side, despite both increments judging (and
  in the validator-reviewer's case, closing INC-01 on the assumption)
  that it was safe to do so as an explicit, separately-verified
  follow-up.
- **D1** (multi-repo becomes primary/default; `single-repo` mode in
  `@axiom/topology` is deprecated as the default, kept only as explicit
  opt-in): `defaultSingleRepoManifest`'s silent fallback in
  `@axiom/topology/src/loader.ts` still has **zero** deprecation-warning
  logic — confirmed still true today, 21 increments after the decision
  was recorded.

This increment is **audit-only** (migration-engineer role): it does not
flip `init.ts`, does not implement the deprecation warning, and does not
touch any code. It produces the consumer-by-consumer safety check that
was explicitly skipped before (causing the original cli-implementer
revert), states a scope-split recommendation for the two pieces of work,
and hands off to the appropriate next role(s).

## Context

Parent chain: `INC-20260702-axiom-redesign-roadmap` (parent roadmap,
Q1/D1, Q-with-D3) -> `INC-20260702-registry-manifest-schema-v2`
(migration-engineer audit, records D1/D2/D3 as settled decisions) ->
`...-design` (schema-writer) -> `...-impl` (registry-engineer) ->
`...-cli` (cli-implementer: attempted the D3 cutover, reverted it after
finding `resolveProject` only read v1) -> `...-resolver-fix` (unplanned
follow-up: made `resolveProject` version-aware for both v1/v2, explicitly
judged the D3 cutover "now safe" but deliberately did not flip it,
recommending "a separately-verified follow-up step, including a real
`init` -> `join` -> `repo attach` -> `tui --topology` walkthrough") ->
`...-validator` (validator-reviewer: made `@axiom/doctor`'s MC-001/
BC-001/BC-002 version-aware, formally closed INC-01, again explicitly
left the D3 cutover as a named, unimplemented follow-up) -> INC-20
(`versioning-reconcile`, audit-only, names the still-v1 `init.ts` as the
concrete blocking fact for `@axiom/versioning`'s own `assets`-per-category
gap) -> INC-21 (`upgrade-reconcile`, audit-only, calls this "the second
consecutive increment blocked on the same root cause," explicitly
recommending this thread be picked up as its own increment before Phase G
continues) -> **this increment**.

This is a new chain (not a further `-reconcile-*` suffix on the resolver-
fix chain) per the user's own naming, since it reconciles both D1 and D3
together rather than continuing the six-role INC-01 subagent sequence.

## Scope

1. Re-confirm (not re-trust) that `resolveProject`'s v1/v2 dispatch,
   `@axiom/config-validation`'s `validateAxiomYamlContent`, and
   `@axiom/doctor`'s MC-001/BC-001/BC-002 are genuinely version-aware
   today, by reading the live source, not the closed specs' own claims.
2. Re-run a full consumer-by-consumer safety check of every place that
   reads `axiom.yaml`'s shape (extending, not just re-citing, the
   resolver-fix increment's 33-import-site consumer audit and the
   original migration-engineer audit's 31-file blast-radius list) to
   find anything that would silently degrade or break if `init.ts`
   started emitting `schemaVersion: 2`.
3. Confirm the current, still-unbuilt state of `defaultSingleRepoManifest`'s
   deprecation warning (D1's other half) in `@axiom/topology/src/loader.ts`
   and in the doctor `TC-001` check that reads it indirectly.
4. Decide and record whether the two pieces of work (D3 emission flip;
   D1 deprecation warning) should be done together in one increment or
   split into two, with rationale.
5. Write a brief for the next role(s) (cli-implementer for D3;
   schema-writer or a direct topology-implementer pass for D1) covering
   exact files, exact required verification steps, and exact known-safe
   vs. known-risky consumers.

## Non-goals

- No code changes of any kind. This is an audit/reconciliation increment
  only, per the task's explicit instruction.
- Do NOT flip `init.ts`'s `buildAxiomYaml` to emit `schemaVersion: 2`.
  That is the next role's job, informed by this increment's findings.
- Do NOT implement `defaultSingleRepoManifest`'s deprecation warning or
  `TC-001`'s corresponding `warn` branch. Same reasoning.
- Do NOT re-litigate D1/D2/D3 themselves, or Q3/Q4 from the parent
  roadmap (both remain open, untouched, consistent with every increment
  in the parent chain).
- No `@axiom/tui` package-internal driver changes (confirmed out of
  scope below — the driver already reads only the normalized
  `ProjectResolution` shape, not raw `axiom.yaml` fields).

## Acceptance criteria

- [x] The resolver-fix increment's claims about `resolveProject` were
      independently re-verified against the live `resolver.ts` source
      (not re-trusted), including confirming no increment since has
      regressed it.
- [x] `init.ts`'s current `buildAxiomYaml` was read fresh; confirmed it
      still emits v1 only, and the historical-revert comment is still
      present and accurate.
- [x] `@axiom/topology/src/loader.ts`'s `defaultSingleRepoManifest` and
      its silent-fallback call site (`loadTopology`) were read fresh;
      confirmed zero deprecation-warning logic exists anywhere in the
      function or its callers.
- [x] Every consumer identified by the original migration-engineer audit
      and the resolver-fix increment's own consumer audit was re-checked
      against the current source, not just re-cited.
- [x] At least one consumer-safety gap not previously flagged by any
      prior increment in this chain was found and documented, if one
      exists (see Implementation notes) — or the audit explicitly states
      none was found, with the search method shown.
- [x] A decision on splitting vs. combining the D3 emission flip and the
      D1 deprecation warning into one or two increments is stated with
      explicit rationale.
- [x] A next-role brief is written naming exact files, exact verification
      steps (including the real `init` -> `join` -> `repo attach` ->
      `tui --topology` walkthrough the resolver-fix increment itself
      asked for), and the consumer-safety findings from this increment.
- [x] Validation was performed per the documented fallback (no code
      changed, no test suite to run for an audit-only increment); a
      best-effort review of `npm run typecheck`/`build` status is not
      needed since zero files changed.

## Open questions

None blocking this increment's own closure. Carried forward, unresolved,
to the next implementation role(s):

- Should `init.ts`'s v2 cutover be unconditional, or gated behind a new
  flag/prompt (`--schema-version`), given `configure.ts`'s
  `readAxiomYamlProjectName` fallback (see finding below) would newly
  fail for v2-emitted projects lacking `axiom.spec/product.manifest.yaml`?
  This increment recommends fixing `readAxiomYamlProjectName` as part of
  the same follow-up work (small, same-shape fix as the MC-001/BC-001
  precedent), not gating `init.ts` behind a flag — but does not implement
  either.
- Q3 (`axiom.spec/` prefix survival) and Q4 (capability/provider/MCP
  folding) remain open at the parent-roadmap level, untouched here.

## Assumptions

- Inherited unchanged from the whole INC-01 chain: non-hard-removal
  posture for D1/D2/D3, no migration-script implementation in this
  increment or its immediate follow-ups.
- The real `Axiom/axiom.yaml` (the workspace-root manifest describing the
  3-repo `Axiom`/`Axiom.Spec`/`Axiom.SDD` topology, `schemaVersion: 1`,
  shape `product`/`repos`/`rules`/`artifact_id_policy`) is a **different,
  unrelated schema** from the `axiom.yaml` discussed throughout this
  chain (`AxiomYamlSchema`/`AxiomYamlSchemaV2` in `@axiom/config-
  validation`, the `project`/`scopes` or `projectId`/`name`/`paths`
  shape written by `axiom init` for scaffolded product projects). This
  is why `packages/doctor/tests/checks.test.ts`'s "repositorio Axiom
  real" integration test already fails today (pre-existing, unrelated,
  confirmed by every prior increment's `git stash` bisection) — it is
  not evidence of anything this increment's scope touches.

## Implementation notes (audit findings)

### 1. Resolver-fix re-confirmation (read fresh, not re-trusted)

`Axiom/packages/project-resolution/src/resolver.ts` (read in full):
`resolveProject` calls `validateAxiomYamlContent(rawContent)`
(`@axiom/config-validation`) and dispatches to `resolveFromV1`/
`resolveFromV2`/`resolveInvalid` on the discriminated result. Confirmed
line-by-line identical in substance to what the resolver-fix increment
described — no increment since has touched this file (grep for any
change history / re-read of the file's own header comments confirms it
still self-describes as "Version-aware desde
INC-20260702-registry-manifest-schema-v2-resolver-fix").

**This confirmation goes one step further than the task's own
instruction asked**: the resolver-fix increment's own closing note
flagged `@axiom/doctor`'s BC-001 as a known, unfixed degrade for v2
documents ("safe degrade, not a crash... worth a deliberate decision").
Re-reading `Axiom/packages/doctor/src/checks.ts`'s `runBoundaryChecks`
directly (not just the resolver-fix spec's account) shows this was
**already fixed** by the subsequent, closed `...-validator` increment:
`usesInstalledMultiRepo` now reads `mode` from both
`rawConfig.project.mode` (v1) and `rawConfig.mode` (v2). Likewise,
`runManifestChecks`'s MC-001 already implements the exact three-way
`pass`(v2)/`warn`(v1)/`fail`(invalid) branch the schema-writer design
specified. **Both of the two remaining doctor-side gaps the resolver-fix
increment left open are already closed** — this is new-to-this-audit
information, since the task's own framing (correctly) only asked about
the resolver-fix increment's claims, not the validator increment that
closed the gaps it flagged. `Axiom/packages/doctor/tests/checks.test.ts`
independently confirms this with passing MC-001/BC-001/BC-002 tests for
both v1 and v2 fixtures.

**Conclusion: yes, genuinely safe on the read side.** No regression
since the resolver-fix/validator increments; both are still exactly as
strong as documented, and one previously-flagged gap (BC-001) closed in
the interim.

### 2. `init.ts`'s current state (read fresh)

`Axiom/apps/cli/src/commands/init.ts`'s `buildAxiomYaml` (lines
240-270) still emits only the v1 shape (`project: { name, status, mode
}` + `scopes: {...}`) for both `layout` variants. The historical-revert
comment (lines 212-239) is still present, accurate, and specifically
cites `resolveProject`'s pre-fix v1-only read as the reason — i.e. the
comment is now **stale relative to the current resolver**, since the
blocker it describes no longer exists, but nobody has updated either the
comment or the emission. This is exactly the state INC-20/INC-21 both
independently found and flagged as the blocking root cause.

### 3. `defaultSingleRepoManifest` deprecation-warning status (read fresh)

`Axiom/packages/topology/src/loader.ts`: `defaultSingleRepoManifest`
(lines 45-65) is a pure builder with no logging, no warning, no
side-effect of any kind. Its only call site, `loadTopology` (lines
142-151), falls back to it silently whenever
`axiom.spec/config/topology.yaml` is absent and `axiom.yaml`'s
(v1-only, via `tryLoadTopologyHint`) `project.mode` is not
`'installed-multi-repo'`. **Confirmed: still completely unbuilt**, 21
increments after the migration-engineer audit
(`INC-20260702-registry-manifest-schema-v2`) named this exact function
and call site as "a concrete extension point for the later
validator-reviewer step, not something to build now" (in the context of
`@axiom/doctor`'s TC-001 check needing a corresponding `warn` branch).
`Axiom/packages/doctor/src/checks.ts`'s `runTopologyChecks`/TC-001 (read
fresh, lines 1181-1244) is still a strict binary `pass`/`fail` on
`validateTopology`'s result, with no branch at all for "resolved via
`defaultSingleRepoManifest` without an explicit opt-in." No increment
between the migration-engineer audit and today (including the
`...-validator` increment that fixed MC-001/BC-001, and every TUI/
topology-adjacent increment: `tui-shell-detection-reconcile`,
`tui-menu-promote-inventory-screens`) touched this function or check.

Additionally, `tryLoadTopologyHint` (`loader.ts` lines 95-126) — the
helper that reads `axiom.yaml` to decide between
`defaultSingleRepoManifest` and `defaultInstalledMultiRepoManifest` when
`topology.yaml` is absent — is itself **v1-only**: it reads
`project.name`/`project.mode` exclusively, with no `schemaVersion: 2`
branch. A v2-emitted `axiom.yaml` (no `.project` object) would make this
helper return `null` unconditionally, which `loadTopology` treats as "no
hint" and falls through to `defaultSingleRepoManifest` regardless of
whether the v2 document's top-level `mode` says
`installed-multi-repo`. **This is a second, previously-unflagged D1/D3
interaction**: re-enabling `schemaVersion: 2` emission without also
updating `tryLoadTopologyHint` would silently downgrade every
newly-`init`-ed `installed-multi-repo` v2 project's topology resolution
to the single-repo default — the exact fallback D1 wants deprecated,
triggered silently by the D3 cutover itself. This must be fixed
alongside (or before) the D3 emission flip, not treated as independent
work.

### 4. Consumer-by-consumer safety check (extends, does not just re-cite, prior audits)

Re-verified against live source, every file identified by the original
migration-engineer audit's 31-file blast-radius list and the
resolver-fix increment's 33-import-site consumer audit, plus a fresh
grep sweep for any `axiom.yaml`-shape-reading code not captured by
either prior list.

**Already version-aware (safe, confirmed by direct read):**

- `@axiom/project-resolution/src/resolver.ts` — `resolveProject` (see
  above).
- `@axiom/config-validation/src/validator.ts` — `validateAxiomYamlContent`
  (unchanged since registry-engineer's increment; still the shared
  version-dispatch primitive everything else should delegate to).
- `@axiom/doctor/src/checks.ts` — MC-001, BC-001, BC-002 (see above).
- `Axiom/apps/cli/src/commands/repo.ts`'s `readProjectNameFromYaml`
  (used by `axiom discover`/`runDiscover`, a **different** command from
  `repo attach`) — already has its own explicit, correct
  `schemaVersion === 2` branch (reads top-level `name`), independently
  implemented outside `resolveProject`. Comment cites
  `INC-20260702-registry-manifest-schema-v2-cli` by name. Confirmed
  correct by direct read; no change needed.
- Every consumer that only reads the normalized `ProjectResolution`
  shape (`name`/`rootPath`/`scopes`/`status`/`mode`) rather than raw
  `axiom.yaml` fields: `apps/cli/src/commands/{_shared,join,projects,
  toolchain,upgrade}.ts`, `apps/cli/src/index.ts`,
  `@axiom/tool-routing/*`, `@axiom/isolation/src/p0.ts`,
  `@axiom/tui/src/driver.ts` (grepped directly — only imports
  `resolveProject`/`ProjectResolution` and reads normalized fields, at
  lines 119-120, 965, 1791; no raw `rawConfig.project.*` access
  anywhere in the package). These are all safe by construction — same
  conclusion the resolver-fix increment reached, re-confirmed by direct
  grep against the current source rather than re-trusted.

**NOT version-aware — genuinely new findings, not previously flagged by
any prior increment in this chain:**

- **`Axiom/apps/cli/src/commands/configure.ts`'s
  `readAxiomYamlProjectName`** (lines 147-158): reads only
  `parsed?.project?.name`, no `schemaVersion` check at all. This is a
  **fourth, independent, ad hoc `axiom.yaml` parser** (distinct from
  `resolveProject`, `validateAxiomYamlContent`, and `repo.ts`'s own
  already-version-aware parser) that nobody made version-aware. It feeds
  `writeCopilotForTarget` (line 289) as a fallback-only source for the
  project name used by `@axiom/document-bootstrap`'s
  `resolveVariables`, which **throws** (`variables.ts` lines 74-83) if
  no project name is available from either `product.manifest.yaml` or
  this `axiom.yaml` fallback. `writer.ts` catches that throw and
  converts it to a `Result` `err`, but `configure.ts`'s own call site
  (lines 413-425) re-throws on any `writeCopilotForTarget` failure,
  which propagates out of `runConfigure` as a hard failure.
  **Concrete blast radius**: today, for a v1-emitted project with no
  `axiom.spec/product.manifest.yaml`, `axiom configure --target
  copilot-vscode` (or `github-copilot`) succeeds using the v1 fallback
  name. If `init.ts` starts emitting `schemaVersion: 2` without also
  fixing this function, the identical scenario for a newly-`init`-ed v2
  project would make `readAxiomYamlProjectName` return `undefined`
  (since it never checks for a top-level `name`/`schemaVersion: 2`),
  and if `product.manifest.yaml` is also absent, `axiom configure`
  would now **fail outright** for those two adapter targets — a real,
  user-facing regression, not a silent degrade. This is exactly the
  category of check the task described as "skipped before" (the
  original cli-implementer revert happened because a similar,
  single-consumer gap in a different package —
  `@axiom/project-resolution` — was found only after attempting the
  cutover, not before). Fixing this requires the same small, additive,
  narrowly-scoped shape of change as MC-001/BC-001's own fix: add a
  `schemaVersion === 2` branch reading the top-level `name` field,
  exactly mirroring `repo.ts`'s already-correct `readProjectNameFromYaml`
  (which could arguably be extracted into a small shared helper at this
  point, given it would then exist in two places — flagged as an
  optional simplification for whichever role does this fix, not
  mandated).
- **`@axiom/topology/src/loader.ts`'s `tryLoadTopologyHint`** (see
  Implementation notes §3 above): v1-only, would silently mis-resolve
  topology for a v2-emitted `installed-multi-repo` project.

**Confirmed NOT a concern (checked and ruled out):**

- `Axiom/apps/cli/src/commands/{axiom-plan,axiom-increment,axiom-bug,
  join,toolchain,gateway,mcp,axiom-qa-e2e,axiom-role,topology,roles,
  audit}.ts` were flagged by the initial broad grep (matching the
  substring `.project` for unrelated reasons — e.g. `PlanMetadata`
  fields, workflow `project` variable names) but contain **no**
  `axiom.yaml#project.name`/`project.mode` reads on direct inspection;
  false positives from the grep pattern, not real consumers.
- `@axiom/versioning`'s `ManagedState` (INC-20, confirmed audited) does
  not read `axiom.yaml` at all — a structurally separate persisted file
  (`.sdd/config/<projectName>/managed-state.json`). Unaffected either
  way by the D3 cutover.
- The real `Axiom/axiom.yaml` (workspace-root manifest) is a distinct,
  unrelated schema (see Assumptions) — not a consumer of
  `AxiomYamlSchema`/`AxiomYamlSchemaV2` at all, and not read by
  `resolveProject` in a way that matters for this thread (it already
  resolves to `ambiguous` today, pre-existing and unrelated, per every
  prior increment's own bisection).

### 5. Two-piece scope decision: split into two follow-up increments, sequenced

**Recommendation: split, not combine**, run sequentially (D3 first,
then D1), for these reasons:

1. **Different blast radii, different verification shapes.** D3's
   emission flip requires a live CLI walkthrough (`init` -> `join` ->
   `repo attach` -> `tui --topology`) plus the two newly-found
   consumer fixes above (`configure.ts`'s `readAxiomYamlProjectName`,
   `loader.ts`'s `tryLoadTopologyHint`) — all concentrated in
   `apps/cli` + `@axiom/topology`'s hint-reading path. D1's deprecation
   warning is a single, self-contained addition to
   `defaultSingleRepoManifest`'s call site plus a new TC-001 `warn`
   branch in `@axiom/doctor` — no CLI walkthrough needed, testable in
   isolation.
2. **D1's warning should fire based on the *post-D3* world, not
   before it.** The whole point of D1's deprecation warning is "warn
   when `single-repo` is in effect without an explicit opt-in, now that
   multi-repo is primary." If D1 is implemented before D3 lands, the
   warning would immediately fire for the overwhelming majority of
   existing (still-v1) projects, none of which had any way to
   "explicitly opt in" to v2/multi-repo yet — a noisy, premature
   warning with no actionable migration path attached. Landing D3 first
   (so `axiom upgrade`/fresh `init` gives users a real v2/multi-repo
   path to opt into) makes D1's warning meaningful the moment it ships,
   rather than immediately noisy.
3. **Different next roles.** D3 is a `cli-implementer` + small
   `@axiom/topology`-hint-reader fix, verified end-to-end via the CLI.
   D1 is closer to a `schema-writer`/`topology`-package change (deciding
   the exact warning mechanism: stderr log line vs. a new `DoctorCheck`
   `warn` only vs. both) plus a `@axiom/doctor` TC-001 update — a
   distinct design decision the task's own instruction did not ask this
   audit to make unilaterally.
4. **Both are still clearly related and should reference each other.**
   The second increment (D1) should explicitly link back to this one
   and confirm D3 has landed before implementing the warning, per point
   2. Neither should be scheduled as fully independent, unrelated work.

## Validation

`No validation command was found. Performed best-effort validation by inspecting changed files and checking consistency against the requested behavior.`

No code was changed by this increment (audit-only), so no build/test run
applies beyond the direct-source-reading verification already documented
in each numbered finding above (each claim was checked against live
`Axiom` source at the exact cited file/line, not re-quoted from a prior
closed spec without re-reading).

## Result

Confirmed, by direct re-reading of live source (not by re-trusting the
closed resolver-fix/validator specs' own accounts):

- `resolveProject`, `validateAxiomYamlContent`, and `@axiom/doctor`'s
  MC-001/BC-001/BC-002 are genuinely, currently version-aware. The one
  gap the resolver-fix increment left open (BC-001) was independently
  found already fixed by the subsequent validator-reviewer increment —
  good news not surfaced by the task's own framing, confirmed here.
- `init.ts` still emits v1 only; the historical-revert comment is stale
  (describes a since-fixed blocker) but still present.
- `defaultSingleRepoManifest`'s deprecation warning (D1's other half) is
  still completely unbuilt, exactly as INC-01's migration-engineer audit
  flagged it 21 increments ago — confirmed by direct read of
  `loader.ts` and `@axiom/doctor`'s TC-001, both unchanged since.
- Two genuinely new, previously-unflagged consumer-safety gaps were
  found that would silently or loudly break under a naive D3 cutover:
  `configure.ts`'s `readAxiomYamlProjectName` (v1-only, feeds a
  fallback chain that **throws** when both it and
  `product.manifest.yaml` are absent — a real hard-failure risk for
  `axiom configure --target copilot-vscode|github-copilot`) and
  `@axiom/topology/src/loader.ts`'s `tryLoadTopologyHint` (v1-only,
  would silently mis-resolve a v2 `installed-multi-repo` project's
  topology to the single-repo default). Both must be fixed as part of
  the D3 follow-up increment, not treated as separate, unrelated work —
  this is exactly the class of check the original cli-implementer
  increment skipped, causing its revert.
- Recommendation: split D3 (schemaVersion:2 emission flip + the two
  newly-found consumer fixes + full CLI walkthrough) and D1
  (`defaultSingleRepoManifest` deprecation warning + TC-001 warn branch)
  into two sequential follow-up increments, D3 first, with explicit
  rationale recorded above.

## General spec integration

**Deferred, not performed**, consistent with every increment in this
chain's own stated reasoning: `Axiom.Spec/general-spec.md` already
records (in its "Versioning" and other sections) that this exact thread
is the named blocker for `@axiom/versioning`'s `assets`-per-category gap
and for Phase G's per-repo change reporting. Adding a new section now,
before the D3/D1 follow-up work actually lands, would either restate
this audit's own findings out of place or lock in details (e.g. the
exact deprecation-warning mechanism) that the next role has not yet
designed. Once the D3 follow-up increment lands and is verified
end-to-end (the real CLI walkthrough), that is the correct point to
update `general-spec.md`'s existing "Versioning" section to note the
blocker is resolved — not before, and not as part of this audit-only
increment.

## Closure rationale

`Status: pending`, not `closed`, because:

- Goal, scope, and acceptance criteria are clear and this increment's
  own narrow audit-only scope is fully satisfied (every acceptance
  criterion above is checked and evidenced against live source).
- However, per `Axiom.SDD/AGENTS.md`'s closure rules, this increment
  produces **no implemented change** — it is explicitly an audit whose
  entire purpose is to hand off to the next implementation role(s). The
  actual D3 emission flip and D1 deprecation warning remain
  unimplemented, by explicit design (this increment's own non-goals),
  not by omission.
- This is consistent with the same posture the parent chain's own
  migration-engineer/audit-only increments took (e.g. INC-20's
  `versioning-reconcile`, INC-21's `upgrade-reconcile`): an audit
  increment with a clear, evidenced result and an explicit next-step
  handoff is correctly `pending`, not `closed`, because "changes were
  implemented, or no-code rationale is explicit" is satisfied by the
  no-code rationale being explicit and deliberate, but the overall
  thread (D1/D3) this increment exists to unblock is still open until
  the follow-up increment(s) actually implement it.

## Next step recommendation

**Immediate: launch a `cli-implementer` increment for D3** (the
schemaVersion:2 emission flip), scoped to:

1. Flip `Axiom/apps/cli/src/commands/init.ts`'s `buildAxiomYaml` to emit
   `schemaVersion: 2` (`AxiomYamlSchemaV2` shape:
   `projectId`/`name`/`repoId`/`role`/`mode`/`paths`), replacing the
   stale historical-revert comment with one describing why it is now
   safe (citing this increment and the resolver-fix/validator
   increments it re-confirmed).
2. Fix `Axiom/apps/cli/src/commands/configure.ts`'s
   `readAxiomYamlProjectName` to add the same `schemaVersion === 2`
   branch `repo.ts`'s `readProjectNameFromYaml` already has (reads
   top-level `name`) — same shape of fix as MC-001/BC-001's own
   precedent. Consider (optional, not mandated) extracting a single
   shared helper at this point, since the identical logic would then
   exist in two files.
3. Fix `Axiom/packages/topology/src/loader.ts`'s `tryLoadTopologyHint`
   to also read a `schemaVersion: 2` document's top-level `name`/`mode`
   fields (mirroring the `resolveFromV2` precedent in
   `@axiom/project-resolution`), so a v2-emitted `installed-multi-repo`
   project does not silently mis-resolve to
   `defaultSingleRepoManifest`.
4. Run the real, live CLI walkthrough the resolver-fix increment itself
   asked for: `axiom init` -> `axiom join` -> `axiom repo attach` ->
   `axiom tui --topology` (and, given the newly-found gap, also
   `axiom configure --target copilot-vscode` against a v2-emitted
   project with no `product.manifest.yaml`, to confirm the
   `configure.ts` fix actually closes the gap this increment found) —
   against the actual built CLI binary, not just unit tests.
5. Run `npm run typecheck`/`npm run build`/`npm test` (full monorepo and
   scoped to `apps/cli`, `@axiom/topology`, `@axiom/document-bootstrap`)
   and cross-check the pre-existing 12-file/13-test failure set
   documented by every prior increment in this chain, confirming no new
   regressions.

**Follow-up, after D3 lands and is verified: a `schema-writer` (or
direct `topology`-implementer) increment for D1**, scoped to:

1. Design and implement the deprecation-warning mechanism for
   `defaultSingleRepoManifest`'s silent fallback in
   `@axiom/topology/src/loader.ts`'s `loadTopology` (a stderr warning
   line, a new `TC-001`-adjacent `warn`-only `DoctorCheck`, or both —
   this increment deliberately leaves that mechanism choice to the next
   role, per its own non-goals).
2. Add the corresponding `warn` branch to `@axiom/doctor`'s TC-001
   (`Axiom/packages/doctor/src/checks.ts`'s `runTopologyChecks`),
   distinct from its current binary `pass`/`fail`, for "manifest
   resolved via `defaultSingleRepoManifest` without an explicit
   opt-in" — exactly the extension point the original migration-engineer
   audit named.
3. Confirm this fires meaningfully post-D3 (i.e., against a project that
   could have opted into `topology.yaml`/multi-repo but didn't), not
   pre-D3 (see this increment's scope-split rationale, point 2).
