# Increment: Dogfooding boundary enforcement reconcile (INC-23, cross-cutting)

Status: closed
Date: 2026-07-02

Closed by `INC-20260702-dogfooding-boundary-reconcile-final-validator`
(final validator-reviewer pass), which confirms the full 4-role chain
(this audit -> design -> implementation -> final review) is complete.
See that increment's "INC-23 closure summary" for the consolidated
closure rationale.

## Goal

Migration-engineer audit for INC-23 (roadmap's cross-cutting section):
confirm exactly what `@axiom/doctor`'s `runBoundaryChecks` (BC-001/BC-002/
BC-003, spec 0013) checks today, confirm whether the `Axiom` product repo
itself currently violates addendum §2's dogfooding rule ("Axiom se
desarrolla con Axiom, pero Axiom no contiene su propia factoría interna
como parte del producto instalable"), and design a genuinely generalizable
doctor check for this rule — one that any Axiom-managed project could run,
not one hardcoded to this workspace's specific `Axiom`/`Axiom.SDD`/
`Axiom.Spec` repo names. This increment is audit-and-design only: no code
is written (per the roadmap's subagent sequence, the next roles —
validator-reviewer then registry-engineer — do the design finalization and
implementation).

## Context

Roadmap: `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/
README.md`, cross-cutting section, INC-23. INC-01 through INC-22 are
closed. Addendum source doc (read directly):
`C:\Users\igutierrezz\Downloads\axiom_decisiones_sesion_addendum_revision.md`,
section 2, verbatim rule:

```text
Axiom se desarrolla con Axiom, pero Axiom no contiene su propia factoría
interna como parte del producto instalable.
```

Implications listed in section 2:
- `axiom/` must not contain the agents/skills/prompts used to create Axiom
  unless explicitly part of the installable product.
- `axiom.sdd/` must not be considered product runtime.
- `axiom.spec/` must not be copied inside the product.
- Axiom's installer must not physically depend on `axiom.sdd` or
  `axiom.spec`.
- Axiom must be installable into third-party projects without dragging
  along the private factory used to build Axiom itself.

Roadmap's stated hypothesis (INC-23 entry): `@axiom/doctor`'s
`runBoundaryChecks` (BC-001/BC-002/BC-003, spec 0013) is the closest
existing analog, but scoped to "spec/product-runtime scopes separated
within one `axiom.yml`," not "the `Axiom` product repo must not depend on
`Axiom.SDD`/`Axiom.Spec`." Planned subagent sequence: migration-engineer
(this pass, confirm BC-001/002/003 extension points) -> validator-reviewer
(defines the new check) -> registry-engineer (implements it) ->
validator-reviewer (final review).

Dependencies per roadmap: INC-08 (write-scope validation pattern, closed),
INC-01 (registry v2 / topology model, closed).

## Scope

- Read `Axiom/packages/doctor/src/checks.ts`'s `runBoundaryChecks` in full
  and confirm exactly what BC-001/BC-002/BC-003 check.
- Grep the entire `Axiom` package (source, tests, `package.json`,
  workspace config, the repo-root `axiom.yaml`) for any reference —
  literal string, relative path, or dependency — to this workspace's
  sibling `Axiom.SDD`/`Axiom.Spec` repos, to confirm or refute whether the
  dogfooding boundary is already respected in practice.
- Identify the design tension between BC-001/002/003 (single-repo,
  in-manifest scope separation) and a genuine cross-repo/cross-workspace
  dogfooding check, and explain why the latter cannot hardcode
  `Axiom.SDD`/`Axiom.Spec` as magic strings without breaking
  `@axiom/doctor`'s "runs on any user's Axiom-managed project" design
  intent.
- Propose the generalized check design: a check keyed on the **role**
  (`code` vs `sdd` vs `spec`) as resolved through INC-01's registry v2
  (`ProjectEntryV2.repos: Record<RepoRoleKey, RegistryRepoEntry>`) or
  `@axiom/topology`'s `TopologyManifest` (`sddRepo`/`specRepo`/
  `roleCodeRepositories`), not on this workspace's specific repo names.
- Confirm this design is buildable today against the real, landed INC-01
  data shapes (cite the actual field names/types read directly from
  source, not assumed from the roadmap's summary).
- Hand off a concrete brief for the next role (validator-reviewer) to
  finalize the check's exact id, category, pass/fail/warn/skip semantics,
  and evidence strings, consistent with every other `@axiom/doctor` check
  in `checks.ts`.

## Non-goals

- No code changes to `@axiom/doctor` or any other package in this
  increment — that is registry-engineer's task, after validator-reviewer
  finalizes the design.
- No re-litigation of BC-001/002/003's existing behavior; they are
  confirmed and left untouched.
- No attempt to hardcode this workspace's own repo names
  (`Axiom`/`Axiom.SDD`/`Axiom.Spec`) into `@axiom/doctor` as a one-off —
  explicitly rejected by this audit as inconsistent with `@axiom/doctor`
  being a tool that ships to third-party users' projects.
- No decision on the check's final numeric id (`BC-004` vs a new
  category) — left for validator-reviewer, though this audit records the
  natural candidate and precedent for that decision.

## Acceptance criteria

- [x] BC-001/BC-002/BC-003's exact current behavior is confirmed by direct
      read of `Axiom/packages/doctor/src/checks.ts`, not inferred from the
      roadmap's summary.
- [x] A concrete, evidenced answer exists for whether `Axiom` (the
      product repo in this workspace) currently violates the dogfooding
      rule — with the specific file(s) found, or an explicit "zero
      matches" confirmation with the search patterns used.
- [x] The design tension (generic doctor check vs. workspace-specific
      magic strings) is stated explicitly, not glossed over.
- [x] A concrete, buildable generic-check design is proposed, citing the
      real INC-01 data shapes (registry v2 `ProjectEntryV2`/
      `RegistryRepoEntry`, `@axiom/topology`'s `TopologyManifest`) by
      actual field name.
- [x] A brief for the next role (validator-reviewer) is recorded.
- [x] Recorded in `Axiom.Spec` per `Axiom.SDD/AGENTS.md`; no code changed
      in `Axiom.SDD` or `Axiom`.

## Open questions

- **Q-dogfood-1 (for validator-reviewer)**: Should the new check be a
  numbered extension of the existing `boundaries` category (e.g.
  `BC-004`), or its own category (e.g. `DF-001`, "dogfooding")? Precedent
  both ways exists in `checks.ts` (e.g. `write-scope` got its own category
  `WS-001` rather than being folded into `boundaries`, even though it is
  conceptually adjacent). This audit does not resolve it — flagged as the
  first concrete decision for the next role.
- **Q-dogfood-2 (for validator-reviewer)**: `runDoctorChecks(resolution:
  ProjectResolution)` (`Axiom/packages/doctor/src/index.ts`) does not
  thread `homeDir` to any check today — the general-spec's own "MCP
  project config" section already flags this exact gap for a future
  `mcp.yml` check. A registry-v2-aware dogfooding check needs `homeDir` to
  resolve `~/.axiom/projects.yml` (`getProjectV2` needs it). This
  increment does not change `runDoctorChecks`'s signature; the next role
  must decide whether to (a) add `homeDir` as a new parameter (breaking
  every existing call site) or (b) read the registry via
  `@axiom/topology`'s `TopologyManifest` instead, which resolves repo
  paths without needing `homeDir` (see "Recommended check design" below
  for why (b) is the safer default).
- **Q-dogfood-3 (for validator-reviewer)**: should the check also flag a
  reverse-direction violation — i.e. the `sdd`-role or `spec`-role repo
  containing a copy of the `code`-role repo's runtime — or is the rule
  strictly one-directional (`code` must not depend on `sdd`/`spec`, but
  `sdd`/`spec` referencing `code` paths for context-reading purposes is
  expected and fine)? This audit's reading of addendum §2 is that the
  rule is one-directional (the product must not embed its own factory),
  but this is worth an explicit confirmation before implementation.

## Assumptions

- The roadmap's own framing of BC-001/002/003 as "the closest existing
  analog, scoped to one repo" is the starting hypothesis to confirm, which
  this audit does confirm as accurate (see Implementation notes).
- INC-01's registry v2 (`ProjectEntryV2.repos`) and `@axiom/topology`'s
  `TopologyManifest` are both real, landed, and usable today — confirmed
  by direct read of `Axiom/packages/user-workspace/src/registry-types.ts`
  and `Axiom/packages/topology/src/types.ts` in this pass, not assumed
  from `general-spec.md`'s summary alone.
- This workspace's own `Axiom`/`Axiom.SDD`/`Axiom.Spec` 3-repo layout is
  used here only as the **motivating example** to prove the generic check
  design is meaningful — the check itself must not hardcode these names.

## Implementation notes

Audit-only. No files were changed under `Axiom/packages/`, `Axiom/apps/`,
or `Axiom.SDD/` for this increment.

### 1. BC-001/BC-002/BC-003, confirmed by direct read

Read in full: `Axiom/packages/doctor/src/checks.ts`, lines 92-174
(`runBoundaryChecks`), plus its call site in
`Axiom/packages/doctor/src/index.ts` (`runDoctorChecks`).

`runBoundaryChecks(resolution: ProjectResolution): DoctorCheck[]` operates
entirely on ONE `axiom.yml`'s resolved `scopes: Record<string, ScopeInfo>`
(`@axiom/project-resolution`'s `ProjectResolution`), where each `ScopeInfo`
carries `isProductRuntime: boolean` and `exists: boolean`. Confirmed
checks:

- **BC-001** ("Scope de especificación presente y existente"): `pass` if
  at least one declared scope has `isProductRuntime === false` AND
  `exists === true`; otherwise `warn`. Purely a presence check for a
  non-runtime (documentation/spec) scope inside the same manifest.
- **BC-002** ("Superficie runtime declarada en la configuración del
  proyecto"): `pass` if at least one scope has `isProductRuntime === true`,
  OR the project declares `mode: 'installed-multi-repo'` (checked via a
  `project.mode`/top-level `mode` fallback, confirmed live for both
  `axiom.yml` schema v1 and v2 per the `INC-20260702-
  schemaversion2-multirepo-primary-reconcile-validator` increment); `warn`
  otherwise.
- **BC-003** ("Los scopes runtime se tratan como runtime, no como fuente
  documental"): `pass` unless a scope simultaneously has
  `isProductRuntime === true` in a way that also fails its own
  `isProductRuntime === false` check on the same object (this is a
  same-object contradiction guard, not a cross-scope comparison) — `fail`
  if a runtime-marked scope is found "mixing roles."

**Confirmed scope of these three checks**: all three operate exclusively
on `resolution.scopes`, a map derived from ONE project's ONE `axiom.yml`
(`project-resolution`'s `resolver.ts`). None of the three reads any other
repo's filesystem location, any registry entry, or any topology manifest.
The roadmap's characterization — "scopes inside one repo," not "the
product repo must not depend on sibling repos" — is accurate, confirmed
by direct source read, not merely re-trusted.

### 2. Does `Axiom` (this workspace's product repo) violate the dogfooding rule today?

**No violation found.** Searched `Axiom/` (all source, tests,
`package.json`, workspace config) for any reference to the sibling
`Axiom.SDD`/`Axiom.Spec` repos, using multiple pattern passes:

- Exact literal `Axiom\.SDD` / `Axiom\.Spec` (case-sensitive): 4 matches,
  all comments citing where the *spec document* for a past increment
  lives (e.g. `// Spec: Axiom.Spec/specs/increments/
  INC-20260702-skills-role-index-reconcile*` in `checks.ts`) — these are
  human-readable provenance comments, not imports, not file reads, not
  path construction.
- Exact filesystem path literal `C:\repos\Axiom Workspace` /
  `../Axiom.SDD` / `../Axiom.Spec` (any relative traversal into the
  sibling repos): **zero matches** anywhere in `Axiom/`.
- `Axiom/package.json` (root): `dependencies` is absent,
  `devDependencies` are `@types/node`/`typescript`/`vitest` only,
  `workspaces` is `["apps/*", "packages/*", "packages/*/*"]` — no
  reference to any path outside the `Axiom` repo itself.
- `Axiom/axiom.yaml` (the repo-root manifest, distinct from the
  per-project `axiom.yml` that `init.ts`/`configure.ts`/
  `resolveProject` generate/read for *installed* third-party projects):
  contains literal `workspaceRoot: Axiom.Spec` and `workspaceRoot:
  Axiom.SDD` fields under `repos.spec`/`repos.sdd`, describing this
  workspace's own 3-repo topology in human-readable form. Confirmed by
  direct grep across every package and app that this file's content
  (`product.id: axiom`, `repos.*.workspaceRoot`, `cli_commands_available`,
  etc.) is **never parsed, read, or imported by any runtime code** — no
  package under `Axiom/packages/*` or `Axiom/apps/*` references
  `workspaceRoot`, `axiom-runtime`, `axiom-sdd`, `axiom-spec`, or
  `mvp-runtime-implantado` (the literal strings from this file). It is
  descriptive documentation of the dogfooding topology, not a
  runtime-loaded configuration input. This is architecturally correct
  under addendum §2's own terms: the file *documents* the separation, it
  does not create a dependency.

**Conclusion**: `Axiom`'s own product code has zero physical, import, or
filesystem-path dependency on `Axiom.SDD` or `Axiom.Spec`. The dogfooding
boundary is already respected in practice. This increment finds no
violation to fix — its real work is the check *design*, not a remediation.

### 3. The design tension, stated explicitly

BC-001/002/003 operate on data that exists inside a single, already-loaded
`axiom.yml` (`ProjectResolution.scopes`). A genuine cross-repo dogfooding
check is a different kind of check by construction: it must know that
MULTIPLE repos exist, where each one is on disk, and which role
(`code`/`sdd`/`spec`) each plays — none of which a single `axiom.yml`'s
`scopes` map encodes today (`scopes` are sub-paths inside ONE repo, not
references to independent repos at arbitrary filesystem locations, per
`general-spec.md`'s own "Project topology model" section).

The naive implementation — hardcode the strings `"Axiom.SDD"` and
`"Axiom.Spec"` into `@axiom/doctor` and grep for them — would be wrong for
a concrete, structural reason, not just a style preference:
`@axiom/doctor` is shipped as part of the **installable Axiom product**
(`Axiom/packages/doctor`) and runs `axiom doctor` inside **any** third
party's Axiom-managed project. A third-party project's sdd-role and
spec-role repos will almost never be named `Axiom.SDD`/`Axiom.Spec` — they
will have the project's own naming (e.g. `kvp25-sdd`/`kvp25-spec`, per the
addendum §9 example already in `general-spec.md`'s "Write-scope
validation" section). Hardcoding this workspace's specific names would
make the check silently useless for every real user project, which
directly contradicts addendum §2's own final implication: "Axiom debe
poder instalarse en proyectos de terceros sin arrastrar la factoría
privada usada para construir Axiom" — a check that only works for Axiom's
own repos would itself be a symptom of the same anti-pattern this rule
exists to prevent (product logic entangled with the private factory used
to build it).

### 4. Recommended generic check design

The check must be phrased as: **"does the `code`-role repo contain any
file or dependency pointing at the filesystem location of the registered
`sdd`-role or `spec`-role repo?"** — parameterized entirely by role, never
by name. This is directly buildable today using INC-01's landed data
shapes, confirmed by direct read of both packages in this pass:

- `@axiom/topology`'s `TopologyManifest`
  (`Axiom/packages/topology/src/types.ts`): `sddRepo: RepoRef`,
  `specRepo: RepoRef`, `roleCodeRepositories: readonly RepoRef[]`, each
  `RepoRef` carrying `{ id: string; ref: string }` (`ref` is either a path
  relative to `projectRoot` or an absolute path/URI). This is the
  **preferred** data source for the new check because it requires only
  `resolution.rootPath` (already threaded to every `@axiom/doctor` check
  via `ProjectResolution`) plus `loadTopology`/`validateTopology` (already
  imported in `checks.ts`) — no signature change to `runDoctorChecks` or
  any new `homeDir` parameter is needed.
- `@axiom/user-workspace`'s registry v2
  (`Axiom/packages/user-workspace/src/registry-types.ts`):
  `ProjectEntryV2.repos: Readonly<Record<RepoRoleKey, RegistryRepoEntry>>`,
  each `RegistryRepoEntry` carrying `{ role: RepoRoleKey; path: string }`
  (an absolute filesystem path). This is a valid **alternative** data
  source with the same role-keyed shape, but it requires `homeDir` to call
  `getProjectV2`/`listProjects`, which `runDoctorChecks` does not thread
  today (the same threading gap `general-spec.md` already flags for a
  future `mcp.yml` check) — using it would force a breaking signature
  change to `runDoctorChecks(resolution, homeDir?)` across every existing
  call site. Recommendation: prefer `@axiom/topology` for this reason
  alone, unless the next role has a concrete reason multi-repo detection
  must go through the user-level registry instead of the project-local
  topology manifest.

Concrete check algorithm (for validator-reviewer/registry-engineer to
finalize the exact id/category/wording against):

1. Load the project's `TopologyManifest` via the existing
   `loadTopology(resolution.rootPath)`. If `mode === 'single-repo'` or the
   manifest is absent/defaulted, `skip` — there is only one repo, so a
   cross-repo dogfooding violation is structurally impossible (mirrors
   `WS-001`'s own skip-when-inapplicable posture).
2. Resolve `sddRepo.ref` and `specRepo.ref` to absolute filesystem paths
   (relative to `resolution.rootPath` when `ref` is a relative path,
   verbatim when `ref` is already absolute — the same resolution
   `@axiom/topology`'s own loader already performs for
   `LocalBindings`/`topology-bindings.yaml` overrides).
3. For each `RepoRef` in `roleCodeRepositories` (the `code`-role repos),
   resolve its own absolute path, then scan that repo's own
   dependency/import surface (at minimum: `package.json`
   `dependencies`/`devDependencies` values that resolve to a local
   `file:`/relative path landing inside the sdd/spec repo's resolved
   path; a source-level grep for path literals that resolve inside those
   same two paths — mirroring this audit's own two-pass method in section
   2 above) for any reference landing inside the resolved `sddRepo`/
   `specRepo` paths.
4. `pass` if no such reference is found; `fail` with an evidence string
   naming the specific file/dependency and which boundary it crosses if
   one is found; `skip` (not `fail`) if the topology manifest cannot be
   loaded/parsed, consistent with every other `@axiom/doctor` check's
   fail-open-to-skip posture on missing/malformed input.

This design satisfies the "reusable, not a one-off" requirement stated in
the task: it never mentions `Axiom`, `Axiom.SDD`, or `Axiom.Spec`
anywhere in its logic — those three names are only this workspace's
specific instance data (`sddRepo.ref`/`specRepo.ref`'s actual values when
`axiom doctor` runs inside this workspace), read from the same
`TopologyManifest` every other topology-aware check already consumes.

## Validation

`No validation command was found. Performed best-effort validation by
inspecting changed files and checking consistency against the requested
behavior.`

Note: `Axiom/package.json` does define real validation commands (`npm run
build`, `npm test`, `npm run doctor`, `npm run typecheck`); they were not
run here because this increment made no code changes to validate — running
the existing test suite would only assert pre-existing state, not anything
produced by this audit-and-design increment.

Best-effort validation performed for this increment:
- Directly read `Axiom/packages/doctor/src/checks.ts` (BC-001/002/003,
  lines 92-174) and `Axiom/packages/doctor/src/index.ts`
  (`runDoctorChecks`'s full check list and signature) in full.
- Directly read `Axiom/packages/topology/src/types.ts` and
  `Axiom/packages/user-workspace/src/registry-types.ts` in full to confirm
  the exact field names cited in the recommended design actually exist
  (not inferred from `general-spec.md`'s summary alone).
- Ran multiple grep passes across the entire `Axiom` repo (case-sensitive
  exact literal, case-insensitive broad, exact path-traversal literal) to
  confirm the "no violation found" conclusion in section 2, including a
  deliberate false-positive check (the first broad case-insensitive pass
  matched unrelated substrings inside `sdd`-containing identifiers; this
  was caught and corrected with a tighter follow-up pass before drawing a
  conclusion).
- Read `Axiom/axiom.yaml`, `Axiom/package.json`, and `Axiom/README.md` in
  full to confirm the repo-root manifest's fields are never consumed by
  any runtime code path.

## Result

BC-001/BC-002/BC-003 are confirmed, by direct read, to check only
in-manifest scope separation (`ProjectResolution.scopes`, one `axiom.yml`)
— the roadmap's characterization was accurate. `Axiom`'s own product code
currently has **zero** physical/import/filesystem dependency on
`Axiom.SDD` or `Axiom.Spec` — the dogfooding rule is already respected in
practice, so this increment is not a remediation, it is a check-design
increment. The real design tension (generic doctor check vs. this
workspace's specific repo names) is real and load-bearing: a naive
hardcoded check would be structurally wrong for a product meant to install
into third-party projects. The recommended fix is a role-parameterized
check built on INC-01's already-landed `@axiom/topology`
`TopologyManifest` (`sddRepo`/`specRepo`/`roleCodeRepositories`), which
requires no `runDoctorChecks` signature change, over the alternative
registry-v2-based design, which would require threading `homeDir` through
every existing doctor call site.

## General spec integration

No integration into `Axiom.Spec/general-spec.md` yet. This increment is
audit-and-design only, one step in a 4-role chain (migration-engineer ->
validator-reviewer -> registry-engineer -> validator-reviewer); the
existing convention across this roadmap's other multi-role increments
(e.g. `INC-20260702-write-scope-validation-reconcile`) integrates into
`general-spec.md` once the chain closes with real, implemented code, not
at the first audit step. Once registry-engineer implements the check and
the final validator-reviewer pass confirms it, that closing increment
should add a "Dogfooding boundary check" section to `general-spec.md`
documenting the check's final id/category, the role-parameterized design,
and this audit's "no violation found today" baseline finding.

## Next step recommendation

Hand off to **validator-reviewer** with this brief: (1) resolve
Q-dogfood-1 (new category `DF-001` vs. `boundaries` extension `BC-004` —
this audit's non-binding recommendation is a new `dogfooding` category,
consistent with `write-scope` getting its own category despite being
conceptually adjacent to `boundaries`), (2) resolve Q-dogfood-2 by
confirming the `@axiom/topology`-based design (no `runDoctorChecks`
signature change) over the registry-v2-based alternative, (3) resolve
Q-dogfood-3 (one-directional vs. bidirectional check), (4) write the exact
`DoctorCheck` id/description/pass-fail-warn-skip wording and evidence
string format, consistent with every existing check in `checks.ts`. Then
hand off to **registry-engineer** to implement `runDogfoodingBoundaryChecks`
(or equivalent name) following section 4's algorithm above, wired into
`runDoctorChecks`. Then a final **validator-reviewer** pass confirms the
implementation against this design and the real `Axiom`/`Axiom.SDD`/
`Axiom.Spec` workspace as a live (currently-passing, per section 2's
finding) test case.
