# Increment: Confirm registry-wiring for `axiom init` (registry-engineer)

Status: closed
Date: 2026-07-02

## Goal

Confirm (and, only if a genuine narrow gap exists, fix) that `axiom init`
correctly wires into INC-01's repo-registration path (`addProjectV2`,
`~/.axiom/projects.yml`, `repos: { roleKey: { role, path } }`) for what
exists today: basic identity plus the single repo being initialized. This
is the `registry-engineer` role in INC-03's sequence (migration-engineer
-> docs-skills-writer -> **registry-engineer** -> cli-implementer ->
validator-reviewer).

**Explicitly out of scope for this increment** (per direct user
instruction, not a discovery made during this session): the Roles
(8-role model with per-role repo associations) and Tools/MCPs
(user-facing selection + `mcp.yml` manifest) dimensions the
migration-engineer audit flagged as the largest gaps. Both are deferred
to separate future increments — no design or implementation for either
was attempted here, per `Axiom.SDD/AGENTS.md`'s bootstrap limits ("no
complex metadata systems," "no MCP abstractions... unless explicitly
requested").

## Context

Parent roadmap: `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
Phase B ("Instalación de proyecto"), INC-03. Prior roles in this chain:

- migration-engineer audit
  (`Axiom.Spec/specs/increments/INC-20260702-installer-wizard-reconcile/README.md`):
  confirmed `init.ts` already calls `addProjectV2` (INC-01's cutover),
  registers exactly one repo (cwd) under hardcoded role `sdd`, and that
  `repo attach` already covers multi-repo growth with an explicit
  `--role` flag. Flagged Roles and Tools/MCPs as the two large gaps
  against the source docs.
- docs-skills-writer (`Axiom.Spec/specs/increments/INC-20260702-installer-wizard-reconcile-agents-md/README.md`):
  implemented the canonical repo-root `AGENTS.md` generator
  (`writeCanonicalAgentsMd`), wired additively into `init.ts` right
  after `axiom.yaml` is written. Confirmed baseline after that work:
  12 failed files / 13 failed tests / 1231 passed.

This increment re-read `init.ts` in its CURRENT live state (post
docs-skills-writer's edit) rather than trusting either prior spec's
description, per this task's explicit instruction.

## Scope

- Read `Axiom/apps/cli/src/commands/init.ts` in full (live state,
  including the `writeCanonicalAgentsMd` call already wired by
  docs-skills-writer).
- Confirm whether `init.ts` calls `addProjectV2` today, and whether the
  `repos` map it constructs matches the "reconciled installer" shape
  (`{ roleKey: { role, path } }`).
- Read `Axiom/packages/user-workspace/src/registry-types.ts`,
  `registry.ts` (`addProjectV2`, `findByRepoPathV2`), and
  `registry-id.ts` (`slugifyProjectId`) to verify the exact contract
  `init.ts` is expected to satisfy.
- Read `Axiom/apps/cli/src/commands/repo.ts` (`runRepoAttach`) as the
  reference implementation of the same `addProjectV2` call pattern, to
  compare role-key derivation between the two call sites.
- Check for a narrow identity-consistency gap: does `init.ts` derive an
  `id` distinct from `projectName` consistent with the registry entry,
  without touching the deferred `schemaVersion: 2` re-enablement?
- Run full validation (`typecheck`, `build`, `test`) and compare against
  the stated baseline.
- If, and only if, a genuine narrow gap is found in the repo-registration
  wiring itself, implement the minimal fix and add/update tests.

## Non-goals

- No Roles (8-role model, per-role repo associations) design or
  implementation. Deferred to a future increment.
- No Tools/MCPs (user-facing MCP selection, `mcp.yml` manifest) design
  or implementation. Deferred to a future increment.
- No `schemaVersion: 2` re-enablement in `init.ts` (tracked separately,
  e.g. `INC-20260702-init-schemaversion2-cutover`, per the
  migration-engineer audit's recommendation) — not touched here.
- No changes to `@axiom/install-profiles`' composition algorithm.
- No changes to `configure.ts`, `join.ts`, or the `AGENTS.md` generator
  introduced by docs-skills-writer (verified, not modified).
- No speculative multi-repo collection inside `init` itself (the
  migration-engineer audit's Finding 3 already established that `init`
  staying single-repo-first, with `repo attach` covering multi-repo
  growth, is correct by design — not re-litigated here).

## Acceptance criteria

- [x] `Axiom/apps/cli/src/commands/init.ts` read in full, live state,
      confirming exactly how/where it calls `addProjectV2`.
- [x] Confirmed whether a gap exists between `init.ts`'s current
      `addProjectV2` call and the "reconciled installer" target shape.
- [x] If no gap: explicitly stated as "confirm, not implement," backed by
      direct code reading and an existing/passing test that asserts the
      exact registry-entry shape (`id`, `name`,
      `repos.sdd.{role,path}`).
- [x] If a gap: minimal, narrow fix implemented, with rationale tied to
      what already exists (no invented metadata fields).
- [x] Identity-consistency check performed: `projectName` (v1
      `axiom.yaml`) vs. registry `id`/`name` fields — documented, with
      any residual edge case flagged (not silently fixed if out of
      narrow scope).
- [x] Roles and Tools/MCPs dimensions explicitly documented as deferred,
      out-of-scope future increments (not designed, not implemented).
- [x] `npm run typecheck`, `npm run build`, `npm test` (full monorepo)
      executed; results compared against the stated baseline (12 failed
      files / 13 failed tests / 1231 passed).
- [x] No files modified under `Axiom/` unless a genuine gap was found
      (none was — see Result).
- [x] Recorded in `Axiom.Spec` per `Axiom.SDD/AGENTS.md`.
- [x] Next-step recommendation given, including whether cli-implementer's
      slot in the roadmap sequence is still needed.

## Open questions

None blocking. One non-blocking observation is recorded under
"Identity-consistency check" below (a pre-existing, narrow edge case in
`slugifyProjectId` vs. `PROJECT_NAME_REGEX`, not something this
increment's scope authorizes fixing).

## Assumptions

- "Narrow wiring only," per the explicit user instruction, means: verify
  `init.ts`'s existing `addProjectV2` call against INC-01's target shape,
  and fix only if there's a genuine, small, non-speculative divergence —
  not add new questionnaire dimensions, roles, or tools/MCP concepts.
- The user's own instruction anticipated that this role might turn out
  to be "confirm, not implement" (explicitly authorized this outcome in
  the task) — this increment follows that guidance rather than inventing
  work to justify a code change.

## Implementation notes

### Files read in full

- `Axiom/apps/cli/src/commands/init.ts` (live state, current file)
- `Axiom/apps/cli/src/commands/repo.ts` (`runRepoAttach`, comparison
  reference)
- `Axiom/apps/cli/tests/init.test.ts` (Scenario 4: registry auto-registration
  tests)
- `Axiom/packages/user-workspace/src/registry-types.ts`
  (`ProjectEntryV2`, `RegistryRepoEntry`, `RepoRoleKey`)
- `Axiom/packages/user-workspace/src/registry.ts` (`addProjectV2`,
  `findByRepoPathV2`, `computeProjectStatus`)
- `Axiom/packages/user-workspace/src/registry-id.ts`
  (`slugifyProjectId`)
- `Axiom/packages/config-validation/src/schemas.ts`
  (`AxiomYamlSchemaV2#projectId`, for identity-consistency comparison
  only — not modified)

### Finding — `init.ts` already fully implements the reconciled
repo-registration wiring; no gap found

Direct inspection of the live `init.ts` (lines ~638-732) confirms:

- `init.ts` calls `addProjectV2(homeDir, { id, name, repos })` with:
  - `id: slugifyProjectId(result.projectName)` — a real slug-derived id,
    not a hardcoded or duplicated `name`.
  - `name: result.projectName`.
  - `repos: { [DEFAULT_REPO_ROLE]: { role: DEFAULT_REPO_ROLE, path:
    resolvedRoot } }` — exactly the `{ roleKey: { role, path } }` map
    shape `ProjectEntryV2`/`RegistryRepoEntry` require, with the role
    key and the `role` field kept consistent (both `DEFAULT_REPO_ROLE`,
    `'sdd'`), matching `RegistryRepoEntry`'s own documented
    denormalization rationale ("role: Denormalized copy of the map key,
    kept for callers that flatten the map to a list").
- This call site is structurally identical in pattern to `repo.ts`'s
  `runRepoAttach`, which also calls `addProjectV2` with a single-entry
  `repos` map keyed by an explicit or defaulted role — the two call
  sites are consistent with each other; there is no divergence in how
  role keys are derived or written between `init` and `repo attach`.
- `init.ts` also already handles the deduplication path
  (`findByRepoPathV2`) and the `legacy-registry-not-migrated`
  transition-window case, both of which are part of a "reconciled"
  installer's registration flow, not just the happy path.
- `apps/cli/tests/init.test.ts`'s "Scenario 4" (`auto-registra en el
  registry user-level`) already asserts the exact shape this task asked
  to confirm: `entry.id === 'my-init-project'`, `entry.name ===
  'my-init-project'`, `entry.repos.sdd?.role === 'sdd'`,
  `entry.repos.sdd?.path === tmpDir`. This test passes today, unchanged.
- `notifyRepoRegistered` (INC-02's event model) is called after a
  successful `addProjectV2`, matching `repo.ts`'s and `join.ts`'s same
  pattern (confirmed by the migration-engineer audit and re-confirmed
  here by reading the call site directly).

**Conclusion: this is a "confirm, not implement" outcome for
repo-registration wiring specifically.** INC-01's cli-implementer
cutover already did this work correctly and completely; there was no
genuine gap left for registry-engineer to close here. No code was
changed in `init.ts`, `repo.ts`, or any `@axiom/user-workspace` file.

### Identity-consistency check (item 5 of the task)

`init.ts`'s current v1-shape `axiom.yaml` emission (`buildAxiomYaml`)
does **not** persist a `projectId` or `id` field at all — v1's
`axiom.yaml` only has `project.name`. The only place an `id` is derived
is the registry call site itself (`id: slugifyProjectId(result.projectName)`),
which is consistent by construction: it's the same `projectName` string
used both to build `axiom.yaml`'s `project.name` and to derive the
registry `id`/`name`. There is no second, independent "identity" value
anywhere in `init.ts` that could drift from the registry entry — so
there is nothing to reconcile between manifest and registry today.

One narrow, pre-existing edge case worth flagging (not fixing, per
scope): `PROJECT_NAME_REGEX` (`/^[a-z0-9][a-z0-9-]{0,62}$/`) accepts
consecutive hyphens (e.g. `my--project` is a valid `projectName`), while
`slugifyProjectId` collapses consecutive hyphens to one
(`slugifyProjectId('my--project') === 'my-project'`). Verified directly:

```
node -e "..."
'my--project' -> PROJECT_NAME_REGEX.test => true
slugifyProjectId('my--project') => 'my-project' (not idempotent)
```

In this edge case, the registry `id` (`my-project`) would differ from
`axiom.yaml`'s `project.name` (`my--project`) by one collapsed hyphen.
This is:

- Pre-existing behavior from INC-01's schema-writer design
  (`slugifyProjectId`, `registry-id.ts`), not introduced or touched by
  any role in this INC-03 chain.
- Orthogonal to the `schemaVersion: 2` cutover (v1 `axiom.yaml` doesn't
  persist an `id` field to compare against at all; the divergence, if
  it ever matters, is purely between the registry's derived `id` and
  the human-readable `name`/`project.name`, which are allowed to differ
  by design — `id` has always been "a slug derived from name," not
  "the exact same string as name").
- Not a "narrow wiring" gap in the sense this task's scope authorizes
  fixing: fixing it would mean either tightening
  `PROJECT_NAME_REGEX` (a validation-rule change with its own blast
  radius across every `projectName`-consuming caller) or changing
  `slugifyProjectId`'s collapsing behavior (a `@axiom/user-workspace`
  package change affecting `repo attach` and `join` too) — both exceed
  "extend the installer to call INC-01's repo-registration path for
  what already exists," per the explicit scope decision for this
  increment.

Recommendation: leave as a documented, non-blocking observation. If it
ever becomes a real operator-facing confusion point, it belongs in a
small, dedicated follow-up touching `registry-id.ts` and/or
`PROJECT_NAME_REGEX` together, not folded into this "confirm" pass.

### Roles and Tools/MCPs — explicit deferral

Per the explicit user instruction for this increment, the following are
**not designed or implemented here**, and are recorded as deferred,
separate future increments:

- **Roles** (8-role model — PO/Analyst/Architect/Backend/Frontend/QA/
  DevOps/Orchestrator — with repo associations per role). The
  migration-engineer audit's gap-analysis table already documents this
  as a "large gap" against the current 2-value `functionalProfile` axis
  (`product-owner`/`builder`), which serves an unrelated purpose
  (capability/token-budget selection, not role-to-repo association).
  No new fields were added to `ProjectEntryV2`, `RegistryRepoEntry`, or
  `axiom.yaml` for this.
- **Tools/MCPs** (user-facing MCP selection + `mcp.yml` manifest). The
  migration-engineer audit's gap-analysis table documents today's
  `EXTERNAL_DEPS_BY_CAPABILITY` as a derived side-effect of capability
  selection, not a user-facing question, and confirms no `mcp.yml`
  manifest exists. No manifest schema, no MCP abstraction, and no new
  CLI surface were introduced for this.

Both remain exactly as scoped by the migration-engineer audit: separate,
future increments, not part of this "narrow wiring" pass.

## Validation

Validation discovery order followed `Axiom.SDD/AGENTS.md` (README ->
package scripts -> task runners -> test configs -> build configs ->
available scripts). The monorepo has `npm run build`, `npm run
typecheck`, and `npx vitest run` at the root — all three were run.

Commands executed and results:

- `npm run typecheck` (full monorepo, `tsc -b`) — clean, no errors.
- `npm run build` (full monorepo, `tsc -b`) — clean, no errors.
- `npx vitest run` (full monorepo) — **12 failed files / 13 failed
  tests / 1231 passed** (125 files, 1244 tests total).
- `npx vitest run apps/cli/tests/init.test.ts apps/cli/tests/repo.test.ts packages/user-workspace/tests/registry-v2.test.ts`
  — 3 files, 38 tests, all passed (no new tests added; these are the
  pre-existing tests that already assert the shape this increment
  verified).

### Baseline comparison

Stated baseline for this task (post docs-skills-writer): 12 failed
files / 13 failed tests / 1231 passed.

This run: 12 failed files / 13 failed tests / 1231 passed — **identical**.
No code was changed, so an identical result (not merely "no
regressions") is the expected and correct outcome. The failing files are
the same pre-existing, environment/real-repo-relative-path-dependent
tests already accounted for by the migration-engineer audit and
docs-skills-writer's increment (`model-routing`, `doctor`,
`agents`/`skills` catalogs, `tui`, `toolchain`, `telemetry`,
`project-resolution`, `apps/cli/tests/start.test.ts`).

No new tests were added because no code was changed — there was nothing
new to test. The existing `apps/cli/tests/init.test.ts` Scenario 4 tests
already cover the exact assertions this increment's acceptance criteria
required to confirm.

## Result

Confirmed, by direct reading of the live `Axiom/apps/cli/src/commands/init.ts`
and cross-referencing `Axiom/packages/user-workspace/src/registry.ts`/
`registry-types.ts`/`registry-id.ts` and `Axiom/apps/cli/src/commands/repo.ts`,
that `init.ts` already fully and correctly implements the reconciled
repo-registration wiring: it calls `addProjectV2` with a properly
slug-derived `id`, the matching `name`, and a `repos: { sdd: { role:
'sdd', path } }` map consistent with `ProjectEntryV2`'s contract and with
`repo.ts`'s own call-site pattern. This was already fully implemented
and tested by INC-01's cli-implementer cutover; no gap existed for
registry-engineer to close here. **No files were changed under
`Axiom/`.** A narrow, pre-existing, non-blocking identity edge case
(`slugifyProjectId`'s hyphen-collapsing vs. `PROJECT_NAME_REGEX`'s
consecutive-hyphen tolerance) was identified and documented, but is
explicitly out of this increment's narrow scope to fix. The Roles and
Tools/MCPs dimensions remain explicitly deferred to future increments,
as instructed. Full monorepo `typecheck`/`build`/`test` were run; the
test suite result is byte-identical to the stated baseline (12 failed
files / 13 failed tests / 1231 passed), which is the correct and
expected outcome for a no-code-change increment.

## General spec integration

No integration into `Axiom.Spec/general-spec.md` was performed — this
file does not exist in this repo (confirmed absent, consistent with
every prior increment in this chain's own note). No other stable,
consolidated-knowledge document exists to update. The one piece of
knowledge worth flagging for whoever eventually creates
`general-spec.md`: `axiom init`'s registry-wiring to `addProjectV2` is a
**closed, verified, complete** integration point as of this increment —
future roles extending `init`'s questionnaire (Roles, Tools/MCPs) can
build on top of the existing `repos: { roleKey: { role, path } }` call
without needing to re-verify or re-plumb the `addProjectV2` call itself,
only add new data to feed into it.

## Next step recommendation

**Recommend skipping cli-implementer and proceeding directly to
validator-reviewer to close INC-03's narrow-scope version.**

Rationale: the roadmap's planned INC-03 sequence
(migration-engineer -> docs-skills-writer -> registry-engineer ->
cli-implementer -> validator-reviewer) anticipated cli-implementer would
have new questionnaire/registry wiring to implement after
registry-engineer's step. This increment found that INC-01's
cli-implementer cutover already completed that wiring in full — there is
no remaining `init.ts`/registry implementation gap for a second
cli-implementer pass to close in this narrow-scope version of INC-03
(Roles and Tools/MCPs, the two dimensions that WOULD justify further
`init.ts`/CLI implementation work, are explicitly out of scope per the
user's instruction for this increment).

Concretely, the work remaining to close INC-03 (narrow-scope) is
verification/closure, not new implementation:

- validator-reviewer should re-confirm this chain's four artifacts
  (migration-engineer audit, docs-skills-writer's `AGENTS.md` generator,
  this registry-engineer confirmation, and the full monorepo
  test/build/typecheck state) are internally consistent and that no
  acceptance criterion across the chain was left unmet.
- If validator-reviewer agrees, INC-03 can close as "narrow-scope
  reconciliation delivered" (canonical `AGENTS.md` + confirmed registry
  wiring), with Roles, Tools/MCPs, and the `schemaVersion: 2` cutover
  explicitly filed as separate, later increments — not as unfinished
  work inside INC-03 itself.

If a future, broader INC-03-follow-up decides to tackle Roles or
Tools/MCPs, cli-implementer's slot in the roadmap is naturally where
that work would land — but that is new, separately-scoped work, not a
continuation of this increment's narrow wiring confirmation.
