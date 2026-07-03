# Increment: Reconcile `axiom doctor` extension + write-scope validation (migration-engineer audit)

Status: pending
Date: 2026-07-02

## Goal

Execute the **migration-engineer** step of INC-08 of the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
Phase C): confirm whether any `allowedWriteScope`-equivalent **validation**
exists in `Axiom` today, now that INC-06 has already landed the
`allowedWriteScope` **schema** field on plan `metadata.yml`
(`Axiom/packages/workflow/src/artifact-store.ts`). This document is
audit-only: no code was changed in `Axiom`, `Axiom.SDD`, or elsewhere.

## Context

Parent chain read in full before this audit:

1. Parent roadmap —
   `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`
   (INC-08 entry, Phase C).
2. INC-06 impl/validator closure —
   `Axiom.Spec/specs/increments/INC-20260702-increment-bug-plan-commands-reconcile-impl/README.md`,
   `...-validator/README.md`.
3. INC-07 audit (pattern precedent for this document's structure) —
   `Axiom.Spec/specs/increments/INC-20260702-index-rebuild-reconcile/README.md`.

External decision document re-read directly for this audit (not from
memory/summary): `axiom_decisiones_sesion_addendum_revision.md` §9
("Scope de escritura y validación de cambios") — exact rule:

> Todo plan debe poder declarar `targetRepos` y `allowedWriteScope`. Axiom
> debe validar cambios reales contra ese scope.

Example metadata shape (verbatim, already landed by INC-06):

```yaml
targetRepos:
  - kvp25-back
taskType: backend-implementation
allowedWriteScope:
  - repo: kvp25-back
    paths:
      - src/**
      - tests/**
  - repo: kvp25-spec
    paths:
      - plans/PLAN-20260702-001/**
```

Command named explicitly by the addendum:

```bash
axiom validate changes --project kvp25 --plan PLAN-20260702-001
```

Must detect:

- files modified outside scope;
- repos not declared;
- changes in `sdd` not requested;
- manual changes to generated caches;
- inconsistent metadata.

**Correction to the parent roadmap's original framing carried into this
audit** (per the user's explicit instruction): the roadmap's INC-08 entry
was written before INC-06 landed and states `allowedWriteScope` "does not
appear among the audited checks... this is a genuinely new check category
to add." That framing is now stale on the schema side — `allowedWriteScope`
already exists verbatim on `PlanMetadata` (confirmed below, Finding 1). What
remains genuinely missing is (a) any logic that reads real filesystem
changes and compares them against `allowedWriteScope`, and (b) any
`@axiom/doctor` check category or CLI command that exposes that
comparison.

## Scope

- Re-read `Axiom/packages/workflow/src/artifact-store.ts` directly to
  confirm the exact shape of `allowedWriteScope` and `AllowedWriteScopeEntry`
  as landed by INC-06, and judge whether that shape is sufficient for
  addendum §9's validation needs or needs more structure.
- Grep the whole `Axiom` monorepo for any existing "diff changed files"
  or "compare against git" utility (git library dependency, `execSync`/
  `spawnSync` shelling out to `git`, or an in-house file-change-tracking
  mechanism) that could be reused.
- Grep `@axiom/doctor/src/checks.ts` and `governance-checks.ts` for any
  existing check category adjacent to write-scope/diff validation.
- Grep all CLI command registrations (`apps/cli/src/commands/*.ts`,
  `apps/cli/src/index.ts`) to confirm no `axiom validate changes` (or
  `axiom validate` of any kind) command exists today.
- Design-check (not implement) whether "changes in `sdd` not requested" and
  "manual changes to generated caches" detection require new tracking state
  (e.g. recording which files a specific `axiom plan implement`-style run
  touched) or can be derived purely from `git diff` + `allowedWriteScope` +
  a known list of generated/cache paths.
- Recommend whether this belongs as a new `@axiom/doctor` check category, a
  dedicated `axiom validate changes` CLI command, or a single underlying
  function exposed both ways — and recommend the minimal implementation
  that satisfies both surfaces without duplicating logic.
- Produce a brief for the next role, including whether a schema-writer step
  is still needed at all (it may not be, since INC-06 already landed the
  schema) or whether the roadmap's subagent sequence should skip directly
  to a validator-reviewer/cli-implementer step.

## Non-goals

- No code implementation in this increment (audit-only, migration-engineer
  step).
- No re-litigation of the `allowedWriteScope` schema shape itself unless a
  concrete gap is found — INC-06's landed shape is treated as the starting
  point, not as something to redesign speculatively.
- No design of glob-matching implementation details (which library, which
  matching semantics) beyond confirming whether one is needed at all — that
  is the next role's job if this audit concludes new matching logic is
  required.
- No implementation of `axiom validate changes` or any new doctor check —
  deferred to the next role per this audit's recommendation.
- No design of a general "track which files a workflow run touched" system
  unless this audit concludes the simpler git-diff-based approach is
  genuinely insufficient.

## Acceptance criteria

- [x] Confirmed by direct re-read of `artifact-store.ts` whether
      `allowedWriteScope`'s shape (`[{repo, paths}]`) is sufficient for
      addendum §9's validation needs, or whether real validation logic
      needs more structure (e.g. glob support, exact-path matching only).
- [x] Confirmed by grep whether any existing "diff changed files" / git-diff
      utility exists anywhere in the `Axiom` monorepo.
- [x] Confirmed by grep that no `axiom validate changes` command and no
      write-scope doctor check category exist today.
- [x] Design-checked whether "outside scope" / "changes in sdd not
      requested" / "manual changes to generated caches" detection requires
      new tracking state, or can be derived from git-diff + scope +
      known-cache-paths, with a concrete recommendation either way.
- [x] Recommended whether this is a new doctor check, a new CLi command, or
      shared logic exposed both ways, with a minimal-implementation
      rationale.
- [x] Recorded in `Axiom.Spec`; no code changed in `Axiom` or `Axiom.SDD`.

## Open questions

None blocking. See "Recommendation" below.

## Assumptions

- "Real changes" in addendum §9's sense means the working tree's
  uncommitted (and/or last-commit) diff relative to some baseline (most
  likely the plan's approval point or a `git diff` against a base ref),
  not a live filesystem-watcher. This audit does not attempt to pin down
  the exact baseline ref — that is a design decision for the next role,
  flagged explicitly in the Recommendation section rather than assumed.
- `<specPath>` continues to resolve to `<projectRoot>/axiom.spec` per
  INC-06's established convention; this audit does not revisit that
  resolution mechanism.
- Each target repo in `targetRepos`/`allowedWriteScope` is assumed to be a
  git repository reachable from the resolved project topology (per
  `@axiom/topology`); validating scope for a non-git-tracked repo is out of
  scope until a concrete need surfaces.

## Implementation notes

### Finding 1 — `allowedWriteScope`'s schema shape is confirmed exactly as the roadmap's stale hypothesis assumed it might be revised, but INC-06 already landed it

Direct re-read of `Axiom/packages/workflow/src/artifact-store.ts` (full
file) confirms:

```ts
export interface AllowedWriteScopeEntry {
  readonly repo: string;
  readonly paths: ReadonlyArray<string>;
}

export interface PlanMetadata extends BaseArtifactMetadata {
  readonly kind: 'plan';
  readonly links: PlanLinks;
  readonly targetRepos: ReadonlyArray<string>;
  readonly taskType: string;
  readonly allowedWriteScope: ReadonlyArray<AllowedWriteScopeEntry>;
}
```

This is verbatim the shape from addendum §9's example
(`repo: kvp25-back`, `paths: [src/**, tests/**]`). `parseMetadata` already
validates it structurally (`isAllowedWriteScopeEntry`: `repo` must be a
string, `paths` must be an array of strings), and
`makeInitialPlanMetadata` already supports constructing it. `targetRepos`
and `taskType` are landed alongside it, also verbatim from the addendum's
example. **This part of the roadmap's INC-08 framing ("schema-writer
needed for allowedWriteScope on plan metadata") is confirmed stale**: the
schema-writer task described in the roadmap is already done, by INC-06,
not by this increment.

**Sufficiency judgment**: the `[{repo, paths}]` shape itself is sufficient
to *declare* scope. It is not, by itself, sufficient to *validate* scope,
because `paths` entries are glob-like strings (`src/**`, `plans/PLAN-.../**`)
and nothing in `artifact-store.ts` — nor anywhere else in the monorepo
(confirmed in Finding 2) — implements glob-pattern matching against those
strings. The schema does not need a *shape* change (no new fields, no
restructuring); it needs a **consumer** that interprets `paths` as glob
patterns and matches a list of real changed-file paths against them. This
is a new small function, not a schema change — `paths: string[]` already
carries enough information for a glob matcher to consume directly.

### Finding 2 — No git-diff or file-change-tracking utility exists anywhere in the monorepo

Grepped `Axiom/packages/*` and `Axiom/apps/*` (source files only, excluding
`dist/`/`tsconfig.tsbuildinfo` build artifacts) for `git diff`,
`--name-only`, `simple-git`, `isomorphic-git`, `execSync`/`spawnSync`
combined with `git`, and `child_process`: **zero real source-code matches.**
The only files matching `child_process`-adjacent substrings were stale
`.tsbuildinfo`/`dist` build-cache artifacts (coincidental substring hits,
not actual usage) plus `packages/workflow/src/hooks.ts`, which on direct
read contains no git invocation of any kind.

Also confirmed via `package.json` grep across the root and every
`packages/*/package.json`/`apps/*/package.json`: **no `simple-git`,
`isomorphic-git`, `glob`, `minimatch`, `micromatch`, or `picomatch`
dependency exists anywhere in the monorepo.** This means:

- There is no existing utility to enumerate "files changed since X" at all
  — not via a git library, not via shelling out to the `git` CLI.
- There is no existing glob-matching library to interpret
  `allowedWriteScope[].paths` entries like `src/**` against real file
  paths. Any implementation must either add a minimal glob dependency or
  hand-roll a narrow matcher (recommendation below favors a small,
  dependency-free matcher given AGENTS.md's minimalism, since the pattern
  vocabulary needed — `dir/**`, `*.ext`, exact paths — is narrow).

`@axiom/doctor`'s existing checks were also inspected directly
(`checks.ts`, `governance-checks.ts`) for anything git-diff-adjacent:
every check reads static files (`axiom.yaml`, `topology.yaml`,
`providers.yaml`, `metadata.yml`, etc.) via `@axiom/filesystem-truth`'s
`fileExists`/`directoryExists`/`readFileContent` helpers — none of them
inspect git state, working-tree diffs, or commit history in any form.

**Conclusion: confirmed — no git-diff utility and no glob-matching utility
exist anywhere in `Axiom` today.** Both would be new, small, additive
pieces if this increment proceeds.

### Finding 3 — No `axiom validate changes` command and no write-scope doctor check category exist today

Grepped every CLI command registration
(`apps/cli/src/index.ts` + all 34 files under `apps/cli/src/commands/`)
for `validate`, `scope`, `write-scope`, `writeScope` (case-insensitive):
24 files matched, but on inspection every hit is either (a) an unrelated
use of the English word "validate" in a comment/doc-string describing an
existing check (e.g. `validateAxiomYamlContent`, `validateTopology`,
`validateToolchain` — schema/shape validators, not change-scope
validators), or (b) `index-cmd.ts`'s `axiom index validate` (metadata.yml
parse-validity, from INC-07, unrelated to write-scope). **No command named
`validate changes`, `scope`, or `write-scope` exists**, and no
`--project`/`--plan`-shaped command resembling the addendum's exact
invocation (`axiom validate changes --project X --plan PLAN-ID`) exists
anywhere.

`@axiom/doctor/src/index.ts`'s `runDoctorChecks` aggregates 19 check-runner
functions today (more than the roadmap's original "9+" estimate — it now
also includes `governance`, `toolchain` (P0/P1/no-user-install),
`mcp-bindings-coherence`, `no-cross-project-memory`,
`adapter-runtime-coverage`, `skills-catalog-coverage`,
`agents-catalog-coverage`, and `artifact-index` (`IX-001`, from INC-07),
none of which existed when the roadmap's INC-08 entry was drafted). None of
these 19 categories reads `allowedWriteScope`, compares it against real
file changes, or does anything write-scope-adjacent. **Confirmed: no
`WS-00x`-equivalent category exists.**

### Finding 4 — Design-check: "outside scope" detection is derivable from git-diff + scope + known-cache-paths; "changes in sdd not requested" needs one additional signal, not new tracking state

Addendum §9 lists five things `axiom validate changes` must detect. Judged
each against whether it needs new state (tracking which files a specific
workflow run touched) or can be derived from `git diff` (working tree vs. a
baseline) + the plan's `allowedWriteScope` + a known list of
generated/cache paths:

1. **Files modified outside scope** — derivable purely from `git diff
   --name-only <baseline>` (or working-tree status) per declared repo,
   matched against that repo's `allowedWriteScope[].paths` globs. No new
   state needed: the plan's `metadata.yml` already declares the scope,
   and git already tracks what changed. This is the core case and it is
   fully covered by the simple approach.
2. **Repos not declared** — derivable the same way: enumerate repos that
   have any diff at all (via `@axiom/topology`'s already-resolved repo
   list) and flag any with changes that are not present in `targetRepos`/
   `allowedWriteScope`. No new state needed.
3. **Changes in `sdd` not requested** — derivable if `targetRepos`/
   `allowedWriteScope` simply omits the `sdd`-role repo for a given plan
   (the common case, since most plans target code/spec repos, not the
   factory itself). If the `sdd` repo has a diff and is absent from
   `allowedWriteScope`, that is indistinguishable from case 1/2 above —
   it is not actually a distinct detection mechanism, it is the same
   scope-comparison applied to a repo that happens to have the `sdd` role
   per `@axiom/topology`'s manifest. **No new tracking state needed**;
   this only needs the validator to resolve each repo's role (already
   available from topology) so it can label the finding "unexpected sdd
   change" instead of a generic "undeclared repo" message — a labeling
   nicety, not new infrastructure.
4. **Manual changes to generated caches** — this is the one item that
   needs a small additional input beyond git-diff + scope: a **known list
   of generated/cache paths** (e.g. `.axiom/cache/**`, adapter-generated
   files per `GENERATED_FILES_BY_TARGET` from `packages/adapters/*`,
   `install-profile.json`). This list already exists in scattered form
   today (each adapter package already declares
   `GENERATED_FILES_BY_TARGET`; `document-bootstrap` has path-guard logic
   for its own outputs) — it is not new tracking state in the sense of
   "record what a workflow run touched," it is a **static, declarative
   list of known-generated paths** that already exists per-package and
   would need to be aggregated, not invented. This is the simpler,
   AGENTS.md-aligned approach: reuse each package's existing
   generated-files declarations rather than building a new run-tracking
   ledger.
5. **Inconsistent metadata** — derivable from `artifact-store.ts`'s
   existing `loadArtifactMetadata`/`parseMetadata` shape validation (already
   used by `IX-001`) plus a cross-check that `targetRepos` and
   `allowedWriteScope[].repo` values are internally consistent (every
   `allowedWriteScope` entry's `repo` should appear in `targetRepos`, and
   vice versa is a judgment call for the next role). No new state needed —
   this is a pure metadata-shape/cross-field check, same pattern as
   `IX-001`.

**Conclusion: the simpler, git-diff-based approach is sufficient for all
five detection cases.** None of them requires building new state to track
which files a specific `axiom plan implement`-style workflow run touched.
The only non-trivial addition beyond raw `git diff` + `allowedWriteScope`
comparison is aggregating each adapter/generator package's already-declared
generated-file paths into one known-cache-paths list for case 4 — itself a
small, additive, non-speculative piece (the underlying data already exists
per-package; nothing new is invented, only aggregated). Recommend the
simpler approach; it is genuinely sufficient and matches
`Axiom.SDD/AGENTS.md`'s minimalism (no new run-tracking ledger, no new
persistent state file).

### Finding 5 — Recommend one underlying function exposed through both a new doctor check and a new CLI command, following the `index-cmd.ts` precedent

The roadmap's INC-08 framing (and this audit's own scope) asks which
surface this belongs on: a new `@axiom/doctor` check (on-demand via
`axiom doctor`, checking current git-diff state against the
currently-approved plan's scope) vs. a dedicated `axiom validate changes`
command (which the addendum names explicitly). These are not competing
options — the addendum's own command name (`axiom validate changes
--project X --plan PLAN-ID`) requires a **specific plan** as input, which
`axiom doctor` (a whole-project health sweep with no per-plan argument
today) does not naturally accept. But both surfaces need the same
underlying comparison logic (diff a repo's working tree against a scope,
report violations with the same `pass/fail/warn/skip` + evidence shape
doctor already uses everywhere).

`apps/cli/src/commands/index-cmd.ts` (INC-07) is the direct precedent for
this exact shape: a thin CLI command built on a shared primitive function
(`listArtifacts`, from `@axiom/workflow`) with a `run*`/`register*` split,
no new package, no cache file. The equivalent shape here:

- A single new exported function (likely in `@axiom/workflow` next to
  `artifact-store.ts`, since it needs `PlanMetadata`/`AllowedWriteScopeEntry`
  directly, or in a small new file within an existing package — not a new
  `@axiom/*` package) that takes a plan's `allowedWriteScope` +
  a per-repo list of changed file paths (already computed by the caller via
  git) and returns a structured list of violations (outside-scope files,
  undeclared repos, unexpected `sdd` changes, generated-cache tampering,
  metadata inconsistencies).
- `axiom validate changes --project X --plan PLAN-ID` (new CLI command,
  addendum's literal name) calls this function directly: resolves the
  plan's `metadata.yml`, shells out to `git diff`/`git status` per declared
  repo (or all topology-known repos, to also catch undeclared-repo
  changes), and reports violations in a human-readable CLI format.
- A new `@axiom/doctor` check category (e.g. `WS-001`) calls the *same*
  function against the currently-approved/most-recent plan (if the project
  resolution can identify one unambiguously) as part of the general
  `axiom doctor` sweep, using the exact `pass/fail/warn/skip` + evidence
  convention already established in `checks.ts` — `skip` cleanly if no
  plan is currently approved/active, matching every other check's
  established skip condition (e.g. `runGatewayStateChecks`' skip-when-
  uninitialized pattern).

This avoids duplicating the diff-vs-scope comparison logic in two places
and matches `index-cmd.ts`'s already-established "thin CLI wrapper over a
shared primitive, also consumable by doctor" pattern — not a new
convention.

## Validation

`No validation command was found. Performed best-effort validation by
inspecting changed files and checking consistency against the requested
behavior.`

This increment made no code changes to validate. Best-effort validation
performed:

- Re-read `Axiom/packages/workflow/src/artifact-store.ts` in full (514
  lines) to confirm `AllowedWriteScopeEntry`/`PlanMetadata`'s exact landed
  shape rather than trusting the roadmap's or INC-06's prose description.
- Grepped `Axiom/packages/*` and `Axiom/apps/*` source files (excluding
  `dist/`/`tsconfig.tsbuildinfo` build artifacts) for git-diff and
  glob-matching library usage; confirmed zero real hits via both source
  grep and `package.json` dependency grep across the root and every
  package/app manifest.
- Read `Axiom/packages/doctor/src/index.ts` in full to get the authoritative,
  current list of 19 aggregated check-runner functions (superseding the
  roadmap's original "9+" estimate) and confirmed none is write-scope
  adjacent.
- Read `Axiom/packages/doctor/src/checks.ts`'s `runArtifactIndexChecks`
  (`IX-001`, from INC-07) directly, since it is the closest existing
  precedent for a doctor check that reads `metadata.yml`/plan-adjacent
  data via `@axiom/workflow`'s `listArtifacts`.
- Grepped all 34 files under `apps/cli/src/commands/` plus
  `apps/cli/src/index.ts` for `validate`/`scope`/`write-scope`/`writeScope`
  and inspected every match directly to rule out false positives (existing
  schema validators unrelated to write-scope) before concluding no such
  command exists.
- Read `Axiom/apps/cli/src/commands/index-cmd.ts` in full as the direct
  structural precedent recommended for the next role's implementation
  (`run*`/`register*` split, shared primitive, no new package).

## Result

**Confirmed the user's stated correction to the parent roadmap**:
`allowedWriteScope` (and `targetRepos`, `taskType`) already exist on
`PlanMetadata` exactly as addendum §9's example specifies, landed by
INC-06. The roadmap's original "schema-writer needed for allowedWriteScope
on plan metadata" framing is stale — that work is done.

**What is genuinely still missing**, confirmed by direct grep rather than
assumed:

1. No git-diff/file-change-enumeration utility exists anywhere in `Axiom`.
2. No glob-matching library exists anywhere in `Axiom` (needed to interpret
   `allowedWriteScope[].paths` entries like `src/**` against real file
   paths).
3. No `axiom validate changes` command, and no `WS-00x`-equivalent
   `@axiom/doctor` check category, exist today.

**Design-check conclusion**: all five detection cases from addendum §9
("files modified outside scope," "repos not declared," "changes in `sdd`
not requested," "manual changes to generated caches," "inconsistent
metadata") are derivable from `git diff` + the plan's already-landed
`allowedWriteScope` + topology's repo-role resolution + an aggregation of
each generator package's already-declared generated-file paths. **None of
them requires new persistent state to track which files a specific
workflow run touched.** The simpler, git-diff-based approach is
recommended and is sufficient.

**Surface recommendation**: one shared comparison function, exposed both as
a new `axiom validate changes --project X --plan PLAN-ID` CLI command
(addendum's literal name, plan-scoped) and a new `@axiom/doctor` `WS-001`
check category (project-wide sweep against the current/most-recent plan),
following `index-cmd.ts`'s established "thin CLI wrapper + doctor-consumable
shared primitive" pattern from INC-07 — not a new convention, and not two
independent implementations of the same comparison.

## General spec integration

No integration into a `general-spec.md` was performed — that file does not
exist in this repo (confirmed by every prior spec in the INC-06/INC-07
chain; the closest equivalents are `Axiom.Spec/specs/00_Resumen_
Ejecutivo.md` through `08_Glosario.md`, not modified here). This is an
audit-only step whose recommendation may still be adjusted by the next
role; consolidating stable knowledge before implementation lands would risk
documenting a shape that changes.

## Closure rationale

Per `Axiom.SDD/AGENTS.md`'s closure rules, this increment is set to
`pending`, not `closed`, because:

- Changes were not implemented (by design — this is the migration-engineer
  audit step; audit-only completion is a valid outcome per the roadmap's
  own subagent sequence, but the roadmap's overall INC-08 goal is not yet
  achieved).
- The recommendation above proposes concrete follow-up work (a shared
  diff-vs-scope comparison function, a new CLI command, and a new doctor
  check category); until that work is implemented and validated, INC-08 as
  a whole is not done.

This mirrors the established chain pattern from INC-06/INC-07
(migration-engineer audit -> next role -> validator-reviewer close), not a
deviation from it.

## Next step recommendation

**Skip the schema-writer role** — it is not needed. INC-06 already landed
the exact schema addendum §9 specifies (Finding 1); re-invoking
schema-writer here would mean editing a schema that already matches spec,
which is unneeded churn per `Axiom.SDD/AGENTS.md`'s minimalism and mirrors
INC-05's precedent of not inventing a role when the audit shows no work for
it.

Proceed directly to **cli-implementer** (paired with **validator-reviewer**
at the end), per the roadmap's own alternate phrasing
("cli-implementer/validator-reviewer"), with this concrete brief derived
from the findings above:

1. Implement a small, dependency-minimal glob matcher (or add a minimal,
   well-known glob library only if hand-rolling proves awkward for the
   `**`/`*` vocabulary actually used) to interpret `allowedWriteScope[].paths`
   entries against real file paths. Keep this narrowly scoped to what
   `allowedWriteScope` examples actually use (directory-prefix + `**`,
   simple `*.ext` patterns) — do not build a general-purpose glob engine.
2. Implement a shared comparison function (in `@axiom/workflow`, alongside
   `artifact-store.ts`, since it consumes `PlanMetadata` directly) that
   takes a plan's `allowedWriteScope`/`targetRepos` plus a per-repo list of
   changed file paths, and returns structured violations covering all five
   addendum §9 cases (Finding 4). The caller is responsible for producing
   the changed-file list via `git diff`/`git status` (shelling out, since no
   git library exists — confirmed in Finding 2) per repo resolved through
   `@axiom/topology`.
3. Implement `axiom validate changes --project <project> --plan <plan-id>`
   as a new CLI command (pattern-match `index-cmd.ts`'s `run*`/`register*`
   split), calling the shared function from step 2 and reporting violations
   in human-readable form with a non-zero exit code on any violation.
4. Add a new `@axiom/doctor` check category `WS-001` (category
   `write-scope`) that calls the same shared function against the
   current/most-recent approved plan (skip cleanly, per every other check's
   established skip convention, if no plan can be unambiguously resolved),
   following the exact `pass/fail/warn/skip` + evidence shape from
   `checks.ts`.
5. For "manual changes to generated caches" (case 4), aggregate the
   already-declared generated-file path lists from `packages/adapters/*`'s
   `GENERATED_FILES_BY_TARGET` (and any other package with an equivalent
   declarative generated-files list, e.g. `document-bootstrap`) rather than
   inventing a new run-tracking mechanism.
6. Hand off to **validator-reviewer** afterward, following the same
   independent-re-verification pattern used to close INC-06/INC-07
   (re-read diffs directly, re-run `npm run typecheck`/`build`/`test`, do
   not trust this audit's or the implementer's prose without independently
   confirming against the actual code).
