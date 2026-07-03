# Increment: Dogfooding boundary enforcement — check design (INC-23, cross-cutting)

Status: closed
Date: 2026-07-02

Closed by `INC-20260702-dogfooding-boundary-reconcile-final-validator`
(final validator-reviewer pass), which confirms the full 4-role chain
(audit -> this design -> implementation -> final review) is complete and
that the implementation matches this design contract with no deviation
beyond the two documented, accepted simplifications. See that
increment's "INC-23 closure summary" for the consolidated closure
rationale.

## Goal

validator-reviewer pass for INC-23 (roadmap's cross-cutting section),
second role in the chain: migration-engineer (done, audit) ->
**validator-reviewer (this increment, design)** -> registry-engineer
(implements it next) -> validator-reviewer (final review). Resolve the 3
open questions left by the migration-engineer audit and produce the exact
`DoctorCheck` contract — id, category, scan scope, pass/fail/warn/skip
conditions, evidence string format — ready for registry-engineer to
implement `checks.ts` without re-deciding anything. No code is written in
this increment.

## Context

Predecessor audit (read in full before this design):
`Axiom.Spec/specs/increments/INC-20260702-dogfooding-boundary-reconcile/README.md`.

Key facts carried over from that audit, re-verified independently in this
pass (not re-trusted blindly):

- `runBoundaryChecks` (BC-001/002/003, `Axiom/packages/doctor/src/checks.ts`
  lines 92-174) operates exclusively on `resolution.scopes`, a map derived
  from one project's one `axiom.yml`. Confirmed no existing check reads a
  second repo's filesystem location or role. This design step does not
  re-litigate that finding.
- `Axiom` (this workspace's own product repo) currently has zero
  physical/import/filesystem dependency on `Axiom.SDD`/`Axiom.Spec` —
  confirmed by the audit's grep passes. Not re-verified here (out of
  scope for a design-only pass); carried forward as-is.
- Roadmap: `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`,
  cross-cutting section, INC-23.
- Addendum source (re-read directly in this pass, not re-trusted from the
  prior summary):
  `C:\Users\igutierrezz\Downloads\axiom_decisiones_sesion_addendum_revision.md`,
  section 2.

## Scope

- Resolve Q-dogfood-1, Q-dogfood-2, Q-dogfood-3 (see "Open questions" in
  the predecessor audit) with explicit rationale, each independently
  re-verified against source in this pass (not merely accepting the
  audit's non-binding recommendation).
- Produce the exact `DoctorCheck` implementation contract: check id,
  category, the minimal sufficient scan logic, pass/fail/warn/skip
  conditions, and the evidence string format, consistent with every
  existing check's shape in `checks.ts`.
- Hand off a concrete, decision-free brief for registry-engineer.

## Non-goals

- No implementation of `checks.ts` — that is registry-engineer's task
  next, per the roadmap's own subagent sequence.
- No re-audit of whether `Axiom` violates the dogfooding rule today (the
  predecessor audit already answered this: no violation found).
- No re-litigation of BC-001/002/003's existing behavior.
- No change to `runDoctorChecks`'s signature (this design explicitly
  avoids requiring one — see Q-dogfood-2 resolution).

## Acceptance criteria

- [x] Q-dogfood-1 resolved with explicit rationale, re-checked against
      live `checks.ts` category prefixes (not just repeating the audit's
      recommendation).
- [x] Q-dogfood-2 resolved by directly re-reading
      `Axiom/packages/topology/src/types.ts` and `loader.ts` in this pass,
      confirming `TopologyManifest`'s `sddRepo`/`specRepo`/
      `roleCodeRepositories` resolve to filesystem paths via
      `resolveRepoPath` without needing `homeDir` or the registry.
- [x] Q-dogfood-3 resolved by directly re-reading addendum §2 in this
      pass, confirming the one-directional reading.
- [x] Exact `DoctorCheck` contract recorded: id, category, description
      wording, scan scope (what is and is not scanned, with rationale for
      the cutoff), pass/fail/warn/skip conditions, evidence string format.
- [x] Brief for registry-engineer is concrete enough to implement without
      further design decisions.
- [x] No code changed in `Axiom.SDD` or `Axiom` — design-only.

## Open questions

None outstanding — all 3 inherited open questions are resolved below.

## Assumptions

- `readTopologyManifest(resolution)` (private helper in `checks.ts`,
  already used by `TC-001`/`TC-003`) is reusable as-is by the new check:
  it already handles `resolution.status !== 'resolved'` -> `null` and
  `loadTopology` failure -> `null`. The new check adds its own additional
  skip condition on top (`manifest.mode === 'single-repo'`), which
  `readTopologyManifest` does not itself apply.
- `RepoRef.ref` resolution must go through `resolveRepoPath` (from
  `@axiom/topology`, already exported) rather than reimplementing path
  logic, to stay consistent with how `WS-001` and every topology-aware
  check resolves repo paths (local binding override first, then
  absolute-looks-like check, then join with `projectRoot`).

## Implementation notes

### Q-dogfood-1 — category naming: own category, not a `boundaries` extension

**Resolution: new category, id prefix `DF-001`.** Confirmed, not just
carried over from the audit's non-binding recommendation.

Rationale, re-verified against live source in this pass:

- Read `Axiom/packages/doctor/src/checks.ts` category constants in full
  (`CATEGORY_BOUNDARIES`, `CATEGORY_POLICIES`, `CATEGORY_MANIFESTS`,
  `CATEGORY_ISOLATION`, `CATEGORY_CAPABILITY_MODEL`,
  `CATEGORY_ARTIFACT_INDEX`, `CATEGORY_WRITE_SCOPE`, plus
  `CATEGORY_TOPOLOGY` and `CATEGORY_TOOLCHAIN` further down the file) and
  every check id prefix in use: `BC-`, `PC-`, `WS-`, `TC-` (topology
  checks TC-001..003 AND toolchain checks TC-004..008 share the `TC-`
  prefix across two different categories — an existing precedent that
  prefix reuse across categories already happens, but not one this design
  should add to).
- `write-scope` (`WS-001`) is the direct precedent cited by the audit: it
  is conceptually adjacent to `boundaries` (both are about what belongs
  where) but was given its own category and its own prefix rather than
  becoming `BC-004`. The dogfooding check is at least as distinct from
  `boundaries` as write-scope is: `boundaries` checks intra-manifest scope
  presence/separation; the new check checks cross-repo filesystem/dependency
  entanglement. Different data source (`TopologyManifest` vs.
  `resolution.scopes`), different failure mode (a real dependency edge vs.
  a missing/mismarked scope), different consumer-facing concept.
  `BC-004` would overload `boundaries` with a check that shares no
  helper, no data source, and no failure semantics with BC-001/002/003.
- A new category also avoids the `TC-` collision risk: if this check
  were folded into `topology` (also plausible, since it reads
  `TopologyManifest`) it would become `TC-009`, but `topology` already
  mixes topology-coverage and QA-lane-coherence concerns (TC-001/002/003)
  with toolchain concerns (TC-004/008) under the same `TC-` prefix
  umbrella in practice — adding a third unrelated concern (cross-repo
  dependency boundary) to that already-overloaded prefix would make
  `TC-` even less meaningful as a category signal. A fresh `DF-`
  category/prefix has zero collision with any existing id prefix
  (`BC-`, `PC-`, `WS-`, `TC-`, and the others confirmed above).

**Decision**: category constant `CATEGORY_DOGFOODING = 'dogfooding'`,
check id `DF-001`.

### Q-dogfood-2 — data source: `@axiom/topology`, not registry v2 + `homeDir`

**Resolution: confirmed, option (b), `@axiom/topology`.** Re-verified by
a fresh read of both `types.ts` and `loader.ts` in this pass (not
re-trusted from the audit's citation alone).

Confirmed directly from `Axiom/packages/topology/src/types.ts`:

```ts
export interface RepoRef {
  readonly id: string;
  readonly ref: string; // path relative to projectRoot, or absolute URI
  readonly description?: string;
}

export interface TopologyManifest {
  readonly schemaVersion: 1;
  readonly mode: 'single-repo' | 'multi-repo';
  readonly sddRepo: RepoRef;
  readonly specRepo: RepoRef;
  readonly roleCodeRepositories: readonly RepoRef[];
  readonly assignments: readonly RoleAssignment[];
  readonly qaLane?: 'inline' | 'parallel';
}
```

Confirmed directly from `Axiom/packages/topology/src/loader.ts`:

- `loadTopology(projectRoot: string): Result<TopologyManifest, TopologyError>`
  takes only `projectRoot` — no `homeDir`. `resolution.rootPath` is
  already available on every `ProjectResolution` passed into every
  `@axiom/doctor` check today (confirmed: `ProjectResolution.rootPath: string`
  in `Axiom/packages/project-resolution/src/resolver.ts`).
- `resolveRepoPath(repoRef: RepoRef, localBindings: LocalBindings, projectRoot: string): string`
  is a pure function, already exported, that resolves any `RepoRef` to an
  absolute filesystem path: local binding override first, then
  `looksAbsolute(ref)` (absolute path or URI scheme), then
  `path.join(projectRoot, ref)`. This is genuinely sufficient to turn
  `sddRepo`/`specRepo`/each `roleCodeRepositories[]` entry into a real,
  comparable filesystem path — no registry, no `homeDir`, no
  `getProjectV2` call needed.
- `loadLocalBindings(projectRoot: string)` (also `homeDir`-free) supplies
  the `LocalBindings` argument `resolveRepoPath` needs — reads
  `<projectRoot>/.sdd/local/topology-bindings.yaml`, defaults to
  `{ schemaVersion: 1, localPaths: {} }` if absent.
- `runDoctorChecks(resolution: ProjectResolution): DoctorReport`
  (`Axiom/packages/doctor/src/index.ts`) confirmed unchanged: no
  `homeDir` parameter exists or is threaded to any check today. The
  `@axiom/topology` path requires zero changes to this signature.
- Precedent already in production: `WS-001` (`runWriteScopeChecks`,
  `checks.ts` line ~2669) already calls `loadTopology(resolution.rootPath)`
  directly inside a doctor check with no `homeDir` — this is not a novel
  pattern, it is the established one.

**Decision**: use `@axiom/topology`'s `loadTopology` +
`loadLocalBindings` + `resolveRepoPath`, exactly as `WS-001` already
does for `loadTopology`. Reject the registry-v2 (`homeDir`-threading)
alternative — confirmed unnecessary, not just "preferred."

### Q-dogfood-3 — one-directional only

**Resolution: confirmed one-directional.** Re-read addendum §2 verbatim
in this pass, in full, independent of the audit's summary.

The five implications listed in §2 are:

1. "`axiom/` no debe contener los agentes/skills/prompts usados para
   crear Axiom salvo que sean parte explícita del producto instalable."
   — constrains `axiom/` (code role) only.
2. "`axiom.sdd/` no debe considerarse runtime del producto." — a
   framing statement about `axiom.sdd`'s role, not a constraint on what
   `axiom.sdd` may contain or reference.
3. "`axiom.spec/` no debe copiarse dentro del producto." — again
   constrains the product (code role): spec must not be copied INTO the
   product, not "code must not be copied into spec."
4. "El instalador de Axiom no debe depender físicamente de `axiom.sdd`
   ni de `axiom.spec`." — explicitly names the installer (code role) as
   the subject that must not depend on sdd/spec. No mirror clause exists
   for "the installer of axiom.sdd/axiom.spec must not depend on axiom".
5. "Axiom debe poder instalarse en proyectos de terceros sin arrastrar
   la factoría privada usada para construir Axiom." — again: the
   installable product (code) must not drag the factory (sdd/spec) along.

All five implications are phrased with the `code` role as the sole
constrained subject and `sdd`/`spec` as the thing that must not leak
into it. None of the five states or implies a reverse constraint (sdd or
spec repos must not reference/contain code). This is also consistent
with ordinary practice already confirmed in the predecessor audit's own
finding: `Axiom.Spec` and `Axiom.SDD` legitimately reference `Axiom`
constructs for context (e.g. checks.ts comments citing spec paths,
`Axiom/axiom.yaml`'s own `repos.spec`/`repos.sdd` `workspaceRoot`
fields, which describe the topology without being consumed at runtime)
— a bidirectional check would flag this normal, expected direction as a
false violation.

**Decision**: the check is strictly one-directional: it flags only
`code`-role repos (`roleCodeRepositories`) containing a
file/dependency reference resolving into `sddRepo` or `specRepo`'s
filesystem path. It does NOT flag `sddRepo`/`specRepo` referencing
`roleCodeRepositories` paths — that direction is expected and out of
scope, not merely unpenalized.

### `DoctorCheck` contract — final design for registry-engineer

**Check id**: `DF-001`
**Category constant**: `CATEGORY_DOGFOODING = 'dogfooding'`
**Description string** (for the `description` field, matching the
existing Spanish-language convention used by every other check's
`description` in `checks.ts`): `"El repo de código no depende físicamente del repo sdd ni del repo spec (dogfooding boundary)"`

**Function name** (for registry-engineer, non-binding on exact name but
consistent with the file's existing `run*Checks` naming convention):
`runDogfoodingBoundaryChecks(resolution: ProjectResolution): DoctorCheck[]`

**Inputs**: only `resolution: ProjectResolution` (specifically
`resolution.status` and `resolution.rootPath`) — the same single
parameter every other check function in `checks.ts` receives. No
signature change to `runDoctorChecks` needed (confirmed under
Q-dogfood-2).

**Step-by-step algorithm**:

1. If `resolution.status !== 'resolved'`: return
   `[skip('DF-001', CATEGORY_DOGFOODING, description, 'No se puede verificar: proyecto en estado "${resolution.status}".')]`
   — matches the exact skip wording pattern used by `BC-001`/`WS-001` for
   unresolved projects.
2. Call `loadTopology(resolution.rootPath)`. If `!result.ok`: return
   `[skip('DF-001', CATEGORY_DOGFOODING, description, 'No se pudo cargar topology.yaml: ${result.error.message}.')]`
   — matches `WS-001`'s exact wording pattern for a topology load
   failure (fail-open-to-skip, not fail, on malformed/unreadable input).
3. If `manifest.mode === 'single-repo'`: return
   `[skip('DF-001', CATEGORY_DOGFOODING, description, 'Proyecto en modo single-repo: sdd, spec y code coinciden en el mismo repo; el boundary de dogfooding no aplica.')]`
   — the concern is structurally moot when there is only one repo (this
   is the "mirrors WS-001's own skip-when-inapplicable posture" case the
   audit called for).
4. If `manifest.roleCodeRepositories.length === 0`: return
   `[skip('DF-001', CATEGORY_DOGFOODING, description, 'No hay roleCodeRepositories declarados en topology.yaml; no hay repo de código que verificar.')]`
   — multi-repo mode but no code repos registered yet (e.g. mid-bootstrap
   project) is also structurally moot, not a failure.
5. Call `loadLocalBindings(resolution.rootPath)`. If `!result.ok`,
   treat as empty bindings (`{ schemaVersion: 1, localPaths: {} }`) rather
   than skip — local bindings are optional/per-user and their absence or
   corruption should not block the check when the shared manifest already
   loaded successfully. (Note for registry-engineer: this one intentional
   deviation from "any load failure skips" is because `loadLocalBindings`
   itself already returns an empty-bindings `ok` result when the file is
   simply absent — an `err` here means the file exists but is malformed,
   which should degrade to "no overrides" rather than abort the whole
   check, since the manifest's own `ref` values are still usable.)
6. Resolve `sddPath = resolveRepoPath(manifest.sddRepo, bindings, resolution.rootPath)`
   and `specPath = resolveRepoPath(manifest.specRepo, bindings, resolution.rootPath)`.
7. For each `codeRepo` in `manifest.roleCodeRepositories`:
   a. Resolve `codePath = resolveRepoPath(codeRepo, bindings, resolution.rootPath)`.
   b. If `!fs.existsSync(codePath)` or `!fs.statSync(codePath).isDirectory()`:
      skip this repo entry silently (do not fail — an unresolvable/not-yet-cloned
      code repo is an availability gap, not a dogfooding violation; this
      mirrors every other check's fail-open posture on missing filesystem
      state).
   c. Run the scan (see "Scan scope" below) against `codePath`, looking
      for any reference resolving inside `sddPath` or `specPath`.
8. Aggregate: if any violation was found across any code repo, return a
   single `fail('DF-001', CATEGORY_DOGFOODING, description, evidence)`
   with `evidence` listing every violating file/dependency and which
   repo+role boundary it crosses (format below) — one aggregated `DF-001`
   result, not one per code repo, matching the convention that most
   checks in this file return exactly one `DoctorCheck` per id for a
   given run (e.g. `BC-003`, `WS-001`) rather than fan out per sub-target.
9. If no violation was found across any code repo, return
   `[pass('DF-001', CATEGORY_DOGFOODING, description, evidence)]` with
   evidence naming which code repos were scanned and against which
   sdd/spec paths (format below), so a passing run is auditable, not
   silent.

**Scan scope — minimal sufficient scan (recommended, not a full AST
parse)**:

The scan runs against each resolved `codePath` and checks for references
landing inside `sddPath` or `specPath`. Two passes, both cheap and
bounded — no full AST parsing, consistent with the audit's own
recommendation and with every existing doctor check's preference for
structural/textual checks over deep static analysis:

1. **`package.json` dependency scan** (primary, highest-signal pass):
   read `codePath/package.json` (and, if a workspace/monorepo, every
   workspace member's `package.json` the repo declares via its own
   `workspaces` field — bounded to that repo's own workspace glob, not a
   recursive `node_modules` walk). For every value under
   `dependencies`/`devDependencies`/`optionalDependencies` that is a
   local reference (`file:`, `link:`, or a bare relative path per npm's
   local-path dependency syntax), resolve it relative to the
   `package.json`'s own directory and check whether the resolved absolute
   path is equal to, or nested inside, `sddPath` or `specPath`. This
   mirrors the audit's own section-2 method (root `package.json`
   `dependencies`/`workspaces` inspection) and is the single
   highest-confidence signal: a real physical dependency, not a string
   coincidence.
2. **Source-level path-literal grep** (secondary, bounded pass): grep
   source files under `codePath` (respecting the repo's own
   `.gitignore`/excluding `node_modules`, `dist`, `build`, and other
   already-known generated-path globs — reuse
   `aggregateKnownGeneratedPathGlobs` from `./write-scope`, already
   imported in this file for `WS-001`, to avoid scanning generated
   output) for `require(...)`/`import ... from ...` string literals, or
   bare string literals matching a relative (`../`) or absolute path
   pattern, that — once resolved relative to the file's own directory —
   land inside `sddPath` or `specPath`. This is a plain-text/regex path
   resolution, NOT a full AST parse: match `require\(\s*['"]([^'"]+)['"]\s*\)`
   and `from\s+['"]([^'"]+)['"]` textually, resolve each captured string
   as a relative path, and only act on captures that already look like a
   path traversal (contain `/` or `\`, or start with `.`) — this
   deliberately excludes bare package-name imports (e.g. `import x from
   'lodash'`), which can never resolve into a sibling repo's absolute
   filesystem path by construction, keeping the pass cheap and low
   false-positive.

**Explicitly out of scope for the scan** (recommendation the audit asked
for — the minimal sufficient scan, not the most thorough possible one):
no full AST parsing/type-checking of source files, no resolution through
bundler alias configs (`tsconfig.json` `paths`, webpack aliases, etc.),
no transitive `node_modules` dependency graph walk. Rationale: a doctor
check runs frequently and must stay fast and simple to reason about;
`package.json` + literal `require`/`import` path scanning already covers
the two realistic ways a physical dependency on a sibling repo would be
introduced (declared package dependency, or a hand-written relative
import reaching across repo boundaries) — anything requiring alias
resolution or transitive graph walking to detect would already be a far
more deliberate, harder-to-miss-in-review violation, and is explicitly
deferred as a future enhancement rather than blocking this check's first
version.

**Pass/fail/warn/skip conditions summary**:

| Condition | Status |
|---|---|
| `resolution.status !== 'resolved'` | `skip` |
| `loadTopology` returns `err` | `skip` |
| `manifest.mode === 'single-repo'` | `skip` |
| `manifest.roleCodeRepositories.length === 0` | `skip` |
| A `roleCodeRepositories[]` entry's resolved path does not exist on disk | silently excluded from the scan (not a per-repo skip result — see step 7b) |
| At least one `package.json` dependency or source-level path literal in any code repo resolves inside `sddPath` or `specPath` | `fail` |
| No such reference found in any scanned code repo | `pass` |

No `warn` outcome is defined for this check: unlike `BC-001`/`BC-002`
(which warn on an absent-but-optional scope), a dogfooding boundary
violation is a real physical dependency that either exists or does not —
there is no "partially acceptable" middle state analogous to those
checks' semantics, so `warn` is intentionally not used here.

**Evidence string format**:

- **On `fail`**: one aggregated string, comma-or-semicolon-joined per
  violation, each entry formatted as:
  `"${codeRepo.id}: ${relativeOffendingPath} → depende de ${boundaryRoleName} (${resolvedBoundaryPath})"`
  where `boundaryRoleName` is literally `sdd` or `spec` (whichever
  boundary was crossed) and `relativeOffendingPath` is the offending
  file or dependency spec's path relative to `codePath` (e.g.
  `package.json#dependencies.axiom-sdd-utils` for a `package.json` hit,
  or `src/foo.ts:12` for a source-literal hit). Example:
  `"code-repo: package.json#dependencies.axiom-sdd-utils → depende de sdd (C:\repos\Axiom Workspace\Axiom.SDD); code-repo: src/bootstrap.ts:44 → depende de spec (C:\repos\Axiom Workspace\Axiom.Spec)"`
- **On `pass`**: a positive confirmation string naming what was checked,
  e.g.:
  `"${N} repo(s) de código verificados sin dependencia física hacia sdd (${sddPath}) ni spec (${specPath})."`
- **On `skip`**: the exact reason strings specified in the algorithm
  steps above (already given verbatim per step 1-4).

This format is consistent with every existing check's evidence style in
`checks.ts` (Spanish-language, states the concrete finding or the
concrete reason for skipping, never generic).

## Validation

`No validation command was found. Performed best-effort validation by
inspecting changed files and checking consistency against the requested
behavior.`

This increment is design-only; no code was written or changed under
`Axiom/packages/` or `Axiom.SDD/`. Best-effort validation performed:

- Re-read `Axiom/packages/topology/src/types.ts` and `loader.ts` in full
  in this pass (not re-trusted from the predecessor audit's citation) to
  confirm `TopologyManifest`'s exact fields, `resolveRepoPath`'s exact
  resolution order, and `loadTopology`/`loadLocalBindings`'s exact
  signatures (both `homeDir`-free).
- Re-read `Axiom/packages/doctor/src/checks.ts` in full for: every
  category constant and id prefix in use (to confirm `DF-` has no
  collision), `WS-001`'s full implementation (as the direct structural
  precedent for a topology-aware check with no `runDoctorChecks`
  signature change), `readTopologyManifest`'s exact behavior (confirmed
  reusable, with one caveat noted in Assumptions), and `runTopologyChecks`/
  `TC-001`/`TC-003`'s skip/pass/fail wording conventions.
- Re-read `Axiom/packages/doctor/src/index.ts` in full to confirm
  `runDoctorChecks(resolution: ProjectResolution)`'s signature is
  unchanged and still takes no `homeDir` parameter.
- Re-read `Axiom/packages/project-resolution/src/resolver.ts` to confirm
  `ProjectResolution.rootPath: string` is a real, existing field already
  threaded to every check.
- Re-read the addendum source document, section 2, verbatim, in full, in
  this pass, to independently confirm the one-directional reading rather
  than trusting the predecessor audit's characterization of it.

## Result

All 3 open questions are resolved:

- **Q-dogfood-1**: new category `dogfooding`, check id `DF-001` (not a
  `boundaries`/`BC-004` extension) — confirmed against live category/id
  prefixes in `checks.ts`, following the `write-scope`/`WS-001` precedent.
- **Q-dogfood-2**: confirmed `@axiom/topology`-based design
  (`loadTopology` + `loadLocalBindings` + `resolveRepoPath`, all
  `homeDir`-free) over the registry-v2 alternative — verified directly
  against `types.ts`/`loader.ts` source, with `WS-001` as existing
  production precedent for the same pattern. No `runDoctorChecks`
  signature change is needed.
- **Q-dogfood-3**: confirmed one-directional — `code`-role repos must
  not depend on `sdd`/`spec`-role repos; the reverse direction is
  expected and explicitly out of scope. Verified against a fresh,
  independent re-read of addendum §2's five implications, all of which
  name the product/installer (code role) as the sole constrained party.

The exact `DoctorCheck` contract (`DF-001`, `dogfooding` category,
two-pass minimal scan: `package.json` dependency values +
`require`/`import` path-literal grep excluding generated-path globs,
skip-heavy fail-open posture on missing/inapplicable input, single
aggregated pass/fail result, evidence string formats for all three
terminal states) is fully specified above, ready for registry-engineer
to implement without further design decisions.

## General spec integration

No integration into `Axiom.Spec/general-spec.md` yet. Per this
increment's own predecessor audit and the established convention across
this roadmap's other multi-role increments (e.g.
`INC-20260702-write-scope-validation-reconcile`), integration happens
once the chain closes with real, implemented and verified code — not at
the design step. This increment's status remains `pending` for that
reason (see Closure rationale below), not `closed`.

## Next step recommendation

Hand off to **registry-engineer** with these exact inputs (no further
design decisions required):

1. Implement `runDogfoodingBoundaryChecks(resolution: ProjectResolution): DoctorCheck[]`
   in `Axiom/packages/doctor/src/checks.ts`, following the "DoctorCheck
   contract" section above verbatim: check id `DF-001`, new category
   constant `CATEGORY_DOGFOODING = 'dogfooding'`, the described
   description string, the exact step-by-step algorithm (skip conditions
   1-4, per-repo scan, aggregated fail/pass), the two-pass scan scope
   (`package.json` dependency values + bounded `require`/`import`
   path-literal grep excluding generated-path globs via the existing
   `aggregateKnownGeneratedPathGlobs` helper from `./write-scope`), and
   the evidence string formats for `fail`/`pass`/`skip`.
2. Wire `runDogfoodingBoundaryChecks(resolution)` into
   `runDoctorChecks` in `Axiom/packages/doctor/src/index.ts` (add to the
   `checks` array and to both the import list and the re-export list,
   matching every other check's wiring pattern already in that file) and
   export `CATEGORY_DOGFOODING` alongside the file's other
   `CATEGORY_*` re-exports.
3. Add unit tests covering: single-repo mode (skip), multi-repo mode with
   no `roleCodeRepositories` (skip), a passing multi-repo case with no
   cross-repo references, a failing case via a `package.json` local
   dependency pointing into `sddPath`/`specPath`, and a failing case via
   a source-level relative import literal pointing into
   `sddPath`/`specPath` — mirroring the existing test structure used for
   `WS-001`/`TC-001` in this package's test suite.
4. Run the existing validation commands confirmed in the predecessor
   audit (`Axiom/package.json`: `npm run build`, `npm test`, `npm run
   doctor`, `npm run typecheck`) after implementation.

After registry-engineer implements and validates, hand off to a final
**validator-reviewer** pass to confirm the implementation against this
design and against the real `Axiom`/`Axiom.SDD`/`Axiom.Spec` workspace as
a live (currently-passing, per the predecessor audit's "no violation
found" finding) test case — and, at that point, integrate the "Dogfooding
boundary check" section into `Axiom.Spec/general-spec.md` documenting the
check's final id/category, the role-parameterized design, and the
baseline finding, per this increment's own "General spec integration"
note above.

## Closure rationale

Status is `pending`, not `closed`: this increment is a design-only step
in a 4-role chain. Per `Axiom.SDD/AGENTS.md`'s closure rules, "changes
were implemented, or no-code rationale is explicit" is satisfied
(no-code is explicit and intentional here), and validation/review against
acceptance criteria are both done — but the chain's own convention
(confirmed by the predecessor audit and by
`INC-20260702-write-scope-validation-reconcile`'s precedent) is to
integrate stable knowledge into `general-spec.md` only once the chain
closes with real implemented code, which has not happened yet. Marking
this `closed` now would be premature given registry-engineer's
implementation and the final validator-reviewer pass are still pending.
