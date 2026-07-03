# Increment: Project-scoped MCP isolation + per-adapter MCP config — audit

Status: pending
Date: 2026-07-02

## Goal

Audit the roadmap's INC-15 hypothesis (`@axiom/isolation` is "a working
superset of addendum 17, scoped to sub-paths of one repo root") directly
against the current, post-INC-01/13/14 code, and determine precisely how
much of INC-15's original scope is already satisfied vs. genuinely new
work. Design the `mcp.yml` schema (source doc §14.2, addendum §13) as the
one confirmed-new artifact, and hand off a scoped brief for the next role.
This increment is audit-only — no code is written.

## Context

Roadmap: `Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/
README.md`, Phase E, INC-15. Depends on INC-13 (MCP tool registration,
closed) and INC-01 (registry v2, closed) per the roadmap's own dependency
line.

Source docs re-read directly for this audit:

- `C:\Users\igutierrezz\Downloads\axiom_decisiones_sesion_prompt_implementacion.md`
  §14 (project-scoped tools/MCPs: global vs. project split, `mcp.yml`
  example, §14.3 isolation rule "si el usuario entra en KVP25 no debe ver
  MCPs de KVP10").
- `C:\Users\igutierrezz\Downloads\axiom_decisiones_sesion_addendum_revision.md`
  §13 (project-scoped MCP config per adapter: canonical `mcp.yml` in the
  project, `adapters/<tool>/mcp.json` generated per adapter) and §17
  (security/isolation reinforcements: no cross-project MCP exposure, no
  cache mixing, cache namespaced by `projectId`, validate `targetRepoId`
  belongs to `projectId`, validate `specRepo`/`sddRepo` belong to the same
  registered project).

`Axiom.Spec/general-spec.md`'s "MCP tool registration" section is the
canonical summary of INC-13/14's landed shape and was read first for fast
context, per this workspace's convention.

## Scope

- Re-read `Axiom/packages/isolation/src/{p0.ts,index.ts}` in full and
  every call site of its exports across the monorepo to confirm or refute
  the roadmap's "sub-paths of one repo root" claim.
- Re-read `Axiom/packages/user-workspace/src/{registry.ts,
  registry-types.ts}` (registry v2) to confirm whether
  `getProjectV2`/`listProjectsV2` already answer addendum §17's two
  explicit validation asks ("`targetRepoId` belongs to `projectId`",
  "`specRepo`/`sddRepo` belong to the same registered project") without
  new isolation logic.
- Grep the full `Axiom/packages/**` tree for any persistent cache
  mechanism, to confirm or refute whether "namespace caches by
  `projectId`" (addendum §17) is a live concern today.
- Re-read `Axiom/packages/mcp-tools/src/*` (all 8 files, including the
  flagship composite handler) to confirm whether tool dispatch takes a
  multi-project-in-one-call shape (which would need isolation
  enforcement) or a single already-resolved project/repo context per call.
- Grep for any existing `mcp.yml`/`mcp.json`/`mcp-manifest.*` artifact
  anywhere in the monorepo, to confirm both that source doc §14.2's
  `mcp.yml` does not exist yet AND to surface any adjacent/colliding
  existing concept before designing a new schema.
- Design `mcp.yml`'s schema (source doc §14.2 shape:
  `schemaVersion, projectId, servers: [{id, type, scope, targetRepo?,
  enabled}]`), reconciled against whatever adjacent existing artifact the
  grep above surfaces.
- Produce the isolation-already-satisfied-vs-new-work assessment and a
  brief for the next role.

## Non-goals

- No implementation of `mcp.yml` reading/writing, no adapter-specific
  generated config (`adapters/opencode/mcp.json`, etc.), no `axiom.spec/
  .axiom/mcp.yml` file written to any real project in this increment.
- No change to `@axiom/isolation`'s existing exports or to
  `runIsolationChecks` in `@axiom/doctor` — this increment only documents
  what would need to change and whether that change is currently needed.
- No re-litigation of INC-13/INC-14's already-closed MCP tool-registration
  design (`sdd`/`spec` -> `axiom-gateway` provider mapping, 17 capability
  ids) — treated as fixed background.
- No decision here on the pre-existing, unrelated `mcp-manifest.yaml` /
  `axiom mcp list|validate` command (spec 0024) beyond noting its
  existence and shape for schema-design purposes — reconciling or merging
  it with `mcp.yml` is out of scope unless a future role is explicitly
  asked to do so.

## Acceptance criteria

- [x] `@axiom/isolation`'s actual behavior (not the roadmap's inherited
      hypothesis) is confirmed directly from source, including every
      production call site of each exported function.
- [x] Addendum §17's two explicit validation asks are checked against
      registry v2's existing read functions, with a concrete yes/no
      answer for each.
- [x] The "namespace caches by `projectId`" concern is checked against
      the actual codebase (does a persistent cache exist at all today)
      before being treated as in-scope work.
- [x] `@axiom/mcp-tools`'s dispatch shape is confirmed as single-project-
      per-call or multi-project-per-call, with evidence.
- [x] `mcp.yml`'s schema is designed to the level of detail needed for an
      implementer to write a loader/validator without further design
      iteration, reconciled against any pre-existing adjacent artifact.
- [x] An honest assessment separates "already satisfied by INC-01/13/14"
      from "genuinely new work," each claim backed by a specific file/
      function reference, not by re-stating the roadmap's own hypothesis.
- [x] Recorded in `Axiom.Spec` per `Axiom.SDD/AGENTS.md`; no code changed
      in `Axiom.SDD` or `Axiom`.

## Open questions

- **Q-mcp-1**: Should `mcp.yml` (source doc §14.2, new) be reconciled
  with, or kept fully separate from, the pre-existing `mcp-manifest.yaml`
  / `axiom mcp list|validate` command (spec 0024, `McpEntry {id,
  displayName, capabilities, installMode, projectBinding, readonly}`)?
  The two describe overlapping but non-identical concepts (a project's
  set of usable MCP-ish things) with different shapes and different
  purposes (0024's manifest feeds `mcp list|validate`'s human-facing
  catalog view; §14.2's `mcp.yml` is meant to be the source adapters
  generate `mcp.json` from). Not blocking for this audit's own
  acceptance criteria, but blocking before an implementer writes
  `mcp.yml`'s loader, because two independently-evolving "list of MCPs
  for this project" files inside the same project risk becoming a
  drift/consistency problem addendum §17 itself warns against in spirit
  (no cross-project mixing) even though addendum §17 was written about
  cross-*project* mixing, not cross-*artifact* mixing within one project.
- **Q-mcp-2**: The roadmap's dependency line names INC-01 as a
  prerequisite for INC-15 ("once INC-01's repo model exists"). INC-01
  closed and shipped registry v2's `repos: {roleKey: {role, path}}` map
  (confirmed below), but `@axiom/isolation`'s `buildIsolationContext`
  still takes a single `ProjectResolution` (one `rootPath`), not a
  registry-v2 `ProjectEntryV2` (multiple repo paths). Extending
  `buildIsolationContext` to accept a multi-repo `ProjectEntryV2` was the
  roadmap's literal INC-15 scope ("what changes when repos are no longer
  sub-paths of one root") — this audit found **no live production code
  path today that needs that extension** (see Assessment below). Is
  building that extension anyway, ahead of a concrete consumer, in scope
  for this chain, or does `AGENTS.md`'s "do not create speculative
  architecture" rule mean it stays deferred until a real multi-repo-aware
  MCP dispatch call site exists?

## Assumptions

- "Genuinely new work" is scoped strictly to what source doc §14/addendum
  §13 explicitly ask for (`mcp.yml` + generated per-adapter configs), not
  to speculative extensions of `@axiom/isolation` that no current call
  site needs, per `Axiom.SDD/AGENTS.md`'s explicit bootstrap limits.
- The next role, if any, should implement `mcp.yml` loading/validation and
  per-adapter config generation directly, without an intermediate
  schema-writer hand-off, because this audit already produces a
  sufficiently detailed schema (see below) — consistent with this
  chain's established precedent of skipping unneeded role-hops when an
  audit's own output already covers the next role's design need (see
  INC-07, INC-11 precedent of collapsing roles when a role's output makes
  the next one trivial).

## Implementation notes

Audit-only. No files were changed under `Axiom/packages/`, `Axiom/apps/`,
or `Axiom.SDD/` for this increment.

### 1. `@axiom/isolation` — does it assume one repo root, or does it already handle registry v2?

Read in full: `Axiom/packages/isolation/src/p0.ts`,
`Axiom/packages/isolation/src/index.ts`, and grepped every reference to
`buildIsolationContext`, `pathsAreIsolated`, `checkMcpAllowed`,
`DEFAULT_ALLOWED_MCP_SERVERS`, `IsolationContext` across the whole
`Axiom/` tree.

**Confirmed: the roadmap's hypothesis is accurate, not stale.**
`buildIsolationContext(resolution: ProjectResolution, allowedMcpServers?)`
takes a single `ProjectResolution` (one `name`, one `rootPath` — the type
from `@axiom/project-resolution`, unrelated to registry v2's
`ProjectEntryV2`). `buildProjectScopedPaths(projectName, rootPath)`
derives every project-scoped path (`memory`, `mcpConfig`, `config`,
`outputs`) as a sub-path of that one `rootPath`, joined under `.sdd/`:

```ts
const sddRoot = path.join(rootPath, '.sdd');
memory: path.join(sddRoot, 'memory', projectName), // etc.
```

There is no code path anywhere in `@axiom/isolation` that accepts a
registry-v2 `ProjectEntryV2` (`repos: Record<RepoRoleKey,
RegistryRepoEntry>`, i.e. multiple independent repo paths). The package
has zero import of `@axiom/user-workspace` (confirmed: no
`registry`/`ProjectEntryV2` reference anywhere in `packages/isolation/`).
This is architecturally consistent with the package predating INC-01
(spec 0014 P0, single-repo-topology era) and never having been revisited
since INC-01 landed registry v2 — INC-01/13/14 did not touch
`@axiom/isolation` at all (confirmed: `@axiom/isolation` does not appear
as a dependency of `@axiom/mcp-tools`, `@axiom/user-workspace`, or any
INC-01/13/14 deliverable).

**Only production call site**: `runIsolationChecks(resolution:
ProjectResolution)` in `Axiom/packages/doctor/src/checks.ts` (lines
288-...), which itself takes a single `ProjectResolution` and only checks
(a) IC-001: project-scoped paths contain the project name, (b) IC-002:
`backend-mcp`/`frontend-mcp` are blocked. Both checks operate entirely
within one resolved project/root — there is no multi-repo or
cross-project comparison happening in the one place this package is
actually wired into the product today.

**`pathsAreIsolated` (the literal cross-project collision check named by
the roadmap) has zero production call sites** — grepped across the whole
monorepo, it appears only in its own definition (`p0.ts`), its own
re-export (`index.ts`), and its own unit test
(`packages/isolation/tests/p0.test.ts`). No doctor check, no CLI command,
no MCP tool handler calls it. It is dead code from the standpoint of the
shipped product, not a "working superset ... in production" as the
roadmap's phrasing could be read to imply — it exists as a tested,
correct pure function, but nothing calls it to actually prevent a
cross-project collision today.

**Conclusion for point 1**: the roadmap's premise holds precisely as
stated — `@axiom/isolation` genuinely assumes one repo root with
sub-paths, and this has not changed post-INC-01. However, the practical
implication is narrower than "INC-15 must extend `buildIsolationContext`
for multi-repo": since nothing calls `buildIsolationContext` with
anything other than a single-root `ProjectResolution`, and nothing calls
`pathsAreIsolated` at all, there is no live bug or live isolation gap
today — only a latent one that would appear if/when some future MCP
transport or dispatch path needs to reason about isolation across a
registry-v2 multi-repo project. See Q-mcp-2.

### 2. Does registry v2 already answer addendum §17's two explicit asks?

Read in full: `Axiom/packages/user-workspace/src/{registry.ts,
registry-types.ts}` (already read for `general-spec.md`'s "Registry v2"
section context; re-confirmed here against the two specific addendum §17
sentences).

Addendum §17's two asks, verbatim:

> Validar que `targetRepoId` pertenece al `projectId`.
> Validar que `specRepo` y `sddRepo` pertenecen al mismo proyecto
> registrado.

**Confirmed: both are already trivially answerable from data
`getProjectV2`/`listProjectsV2` already return — no new isolation logic
needed.**

- `getProjectV2(homeDir, projectId)` returns `ProjectEntryV2WithStatus`
  whose `repos: Record<RepoRoleKey, RegistryRepoEntryWithStatus>` is
  keyed by role (`sdd`, `spec`, or any code-repo role key). "Does
  `targetRepoId` belong to `projectId`" reduces to: call `getProjectV2`,
  check `targetRepoId in result.repos` (or, if `targetRepoId` denotes a
  role-key rather than an opaque id — see note below — check the role key
  is present). This is a one-line membership check against data the
  registry already returns; it requires no new field, no new function,
  and no change to `@axiom/user-workspace`.
- "specRepo and sddRepo belong to the same registered project" reduces to
  the same call: both `repos.spec` and `repos.sdd` come from the *same*
  `ProjectEntryV2` returned by one `getProjectV2(projectId)` call — by
  construction, if both keys resolve, they are already guaranteed to
  belong to the same registered project, because there is no code path
  that lets a caller assemble a `spec` repo from one project's entry and
  an `sdd` repo from another's. The only way this check could fail is if
  a caller manually threads two independently-looked-up repo paths from
  two different `getProjectV2` calls without going through one project
  entry — a caller-side discipline concern, not a missing registry
  capability.

**Caveat on `targetRepoId` naming**: source doc §8.2's
`get_implementation_context(input: { projectId, planId, targetRepoId?,
... })` and the already-shipped
`GetImplementationContextInput.targetRepoId` in
`Axiom/packages/mcp-tools/src/implementation-context-handler.ts` (line
48) documents `targetRepoId` as "Role key (e.g. `'target'`) used to
disambiguate `repositories.target`" — i.e. in the shipped code today,
`targetRepoId` is actually a **role key** into the `repos` map (matching
`RepoRoleKey`), not a separately-namespaced global repo id. This makes
the addendum's "belongs to projectId" check even more direct: it is a
map-key-presence check (`targetRepoId in project.repos`) with no
ambiguity about cross-project ids, because role keys are scoped inside
one project's `repos` map by construction — there is no global repo-id
space where a KVP25 role key could accidentally collide with a KVP10 one.

**Conclusion for point 2**: no new isolation logic is needed for either
addendum §17 ask. If a future increment wants this validation
*enforced* (not just "answerable"), that enforcement is a thin guard
clause inside whatever handler accepts a `targetRepoId`/`projectId` pair
(e.g. `get_implementation_context`'s existing `selectRepositories`
logic), not a new package or a `@axiom/isolation` extension.

### 3. Does a real cache-namespacing-by-`projectId` concern exist today?

Grepped `Axiom/packages/**/*.ts` (excluding `dist/`/build artifacts) for
`cache`, `.axiom/cache`, `CacheEntry`, `writeCache`, `readCache`.

**Confirmed: no persistent cache mechanism exists anywhere in the
codebase today.** The only hits are:
`Axiom/packages/workflow/src/write-scope.ts` and
`Axiom/packages/doctor/src/write-scope.ts`, both of which reference
`.axiom/cache/**` only as a **glob pattern to detect tampering** with a
generated-cache path that write-scope validation should flag if
modified — i.e. code that defends against a cache that might exist in
the future, not code that reads or writes an actual cache today. This is
consistent with `general-spec.md`'s already-documented "Folder-per-
artifact convention" section: *"No persistent `.axiom/cache/*.index.json`
file exists; `axiom index rebuild`/`validate` are cache-less, direct-scan
wrappers... deferred until either a measured performance need or a
concrete consumer... requires one"* (INC-07's explicit, already-recorded
decision).

**Conclusion for point 3**: "namespace caches by `projectId`" (addendum
§17) is confirmed moot/premature today — there is nothing to namespace.
This is not a gap to fix now; it is a design constraint to remember
*if and when* a persistent cache is eventually built (the future cache's
key/path scheme should include `projectId` from day one), consistent
with INC-07's already-recorded deferral rationale.

### 4. Does `@axiom/mcp-tools` dispatch take a `projectId` needing isolation enforcement, or a single resolved project per call?

Read in full: all 8 files in `Axiom/packages/mcp-tools/src/` (`types.ts`,
`registry-handlers.ts`, `artifact-handlers.ts`, `skills-handlers.ts`,
`technical-context-handlers.ts`, `cli-backed-handlers.ts`, `registry.ts`,
`index.ts`, plus `implementation-context-handler.ts`).

**Confirmed: every handler operates on a single, already-scoped
project/repo context per call — there is no multi-project-in-one-call
code path today.**

- `getProjectRegistry(input: {homeDir, includeStale?})` returns the
  *entire* registry (all projects) — this is the one handler that returns
  cross-project data, but it is a read-only listing operation
  (`sdd.projectRegistryRead`, analogous to `axiom projects list`), not a
  per-project MCP/tool access path. It does not leak one project's
  MCP/tool configuration into another's context; it lists project
  *identities*, which is the registry's whole purpose (source doc §3.2's
  own `projects.yml` example lists all projects by design).
  `getProjectRepos(input: {homeDir, projectId})` (the next handler down)
  already takes a single `projectId` and returns only that project's
  `repos` map.
- Every other handler (`getPlan`, `getIncrement`, `getBug`,
  `getAdrIndex`, `getDecisionIndex`, `getSkillIndex`,
  `getSkillCatalog`, `getSkill`, `getTechnicalContextIndex`,
  `getRecommendedContext`, `getAllowedWriteScope`,
  `runIndexRebuild`/`runIndexValidate`/`runValidateChanges`) takes an
  already-resolved `rootPath`/`specRoot`/`projectRoot` as input — a
  single filesystem location the caller has already picked, with no
  `projectId` parameter to cross-reference against a second project.
- `get_implementation_context` (the flagship composite,
  `buildImplementationContext`) takes `{homeDir, projectId, planId,
  targetRepoId?, role?, contextBudget?}` — it does take a `projectId`,
  but resolves it through exactly one `getProjectV2(homeDir, projectId)`
  call, then threads that one project's `repos` map through every sibling
  handler call it composes. There is no branch anywhere in
  `buildImplementationContext` that reads a *second* project's data
  within the same call.

**Conclusion for point 4**: cross-project leakage is structurally
impossible in `@axiom/mcp-tools` today, precisely because no handler's
call signature admits two different `projectId`s in one invocation. This
is "isolation by construction," matching the audit brief's own framing —
not because an isolation layer actively blocks it, but because the
call-path shape has nowhere for a second project to enter. This further
narrows what INC-15 needs to build: there is no live tool-dispatch bug to
fix, only the `mcp.yml` config artifact itself, which is a prerequisite
for *which* MCP servers get exposed per project in the first place (a
concern one level below tool dispatch — it is about adapter/runtime
config, not about `@axiom/mcp-tools`' Node-level function calls).

### 5. `mcp.yml` schema design

Grepped the full `Axiom/` tree for `mcp.yml`, `mcp.json`, and
`mcp-manifest` before designing anything.

**Confirmed: `mcp.yml` (source doc §14.2's exact shape) does not exist
anywhere in the codebase.** However, grep surfaced a **pre-existing,
unrelated artifact that must be reconciled with or explicitly kept apart
from it**: `mcp-manifest.yaml`, consumed by `axiom mcp list|validate`
(`Axiom/apps/cli/src/commands/mcp.ts`, spec 0024 "Lote A", predating this
redesign roadmap chain entirely). Its shape:

```ts
interface McpEntry {
  id: string;
  displayName: string;
  capabilities: readonly string[];
  installMode: 'project-scoped' | 'user-scoped';
  projectBinding: 'required' | 'optional';
  readonly: boolean;
}
interface McpManifest {
  schemaVersion: 1;
  mcps: readonly McpEntry[];
}
```

This is a real, shipped, tested artifact (`apps/cli/tests/mcp.test.ts`)
with a genuinely different shape and purpose from source doc §14.2's
`mcp.yml`: `McpManifest` describes *capabilities* an MCP entry exposes
and whether `sdd`/`spec` are required-bound for `axiom mcp validate`'s
pass/fail gate; it has no `type`/`scope`/`targetRepo` fields and no
per-adapter generation story. **This increment does not merge or replace
`mcp-manifest.yaml`** (out of scope, flagged as Q-mcp-1 above) — the
schema below is designed as an additive, separate artifact, matching
source doc §14.2's own file path (`project.spec/.axiom/mcp.yml`, distinct
from wherever `mcp-manifest.yaml` currently resolves).

**Proposed `mcp.yml` schema** (source doc §14.2 verbatim shape, typed for
an implementer):

```ts
// project.spec/.axiom/mcp.yml
interface McpProjectConfig {
  schemaVersion: 1;
  projectId: string; // must match a registry-v2 project id (getProjectV2 key)
  servers: readonly McpServerEntry[];
}

type McpServerType = 'axiom' | 'serena' | 'integration' | string; // open-ended, mirrors RepoRoleKey's open-endedness rationale
type McpServerScope = 'project' | 'repo';

interface McpServerEntry {
  id: string; // unique within this file
  type: McpServerType;
  scope: McpServerScope;
  /** Required when scope === 'repo'; must be a role key present in this
   *  project's registry-v2 `repos` map (getProjectV2(projectId).repos).
   *  Omitted/ignored when scope === 'project'. */
  targetRepo?: string;
  enabled: boolean;
}
```

Validation rules an implementer's loader/validator should enforce
(derived directly from source doc §14.2's example plus this audit's
findings in points 1-4, not invented beyond them):

1. `schemaVersion` must equal `1` (closed literal, same pattern as every
   other Axiom schema in this codebase — `ProjectsFile`, `axiom.yml`,
   etc.).
2. `projectId` must resolve via `getProjectV2(homeDir, projectId)` to a
   real, registered project (reuses registry v2 read path from point 2 —
   no new lookup mechanism).
3. Every `servers[].id` must be unique within the file.
4. `servers[].scope === 'repo'` requires `targetRepo` to be present AND
   to be a key in `getProjectV2(projectId).repos` (this is precisely
   addendum §17's "validate `targetRepoId` belongs to `projectId`" ask,
   applied at config-validation time instead of at tool-call time —
   confirmed as a one-line membership check per point 2, no new logic).
5. `servers[].scope === 'project'` should reject a present `targetRepo`
   (a project-scoped server naming a specific repo is a contradiction;
   flag as a validation error, not silently ignore it).
6. No two `servers[]` entries should declare the same `(type, targetRepo)`
   pair with both `enabled: true` (avoids ambiguous double-registration
   of the same physical MCP server for the same repo — not in source
   doc's text but a reasonable, minimal guard consistent with "don't let
   config get into a state doctor can't explain").

**Isolation guarantee this schema gives "for free"**: because
`mcp.yml` lives at `project.spec/.axiom/mcp.yml` (one file per project,
inside that project's own spec repo) rather than in any global/shared
location, addendum §14.3's rule ("si el usuario entra en KVP25 no debe
ver MCPs de KVP10") is satisfied by simple physical/filesystem scoping,
the same way registry v2 already scopes `repos` per project. No new
`@axiom/isolation` code is needed to enforce this — reading the wrong
project's `mcp.yml` would require the caller to have already resolved
the wrong project's `specRepo` path, which point 2's registry-v2
lookup path does not allow to happen silently.

**Per-adapter generation** (addendum §13's explicit ask): the pattern
already exists for a structurally identical problem —
`GENERATED_FILES_BY_TARGET` in `@axiom/installer` plus each
`packages/adapters/*/src/generator.ts` (6 real adapters: opencode,
claude-code, github-copilot, vscode, cursor, litellm) already generate
tool-specific files from one canonical source
(`ResolvedInstallProfile`/`axiom.yml`), with atomic writes and (for
opencode/claude-code) `AXIOM:GENERATED`/`TEAM:CUSTOM` marker-block
preservation. `mcp.yml` -> `adapters/<tool>/mcp.json` should follow the
exact same generator pattern per adapter (one new generator function per
adapter package, reusing the existing atomic-write/marker-preservation
helpers where the target format supports it), not a new generation
mechanism. This is additive work with a clear, already-proven template to
copy — genuinely new code, but architecturally low-risk because the
pattern is proven 6 times over already.

## Validation

`No validation command was found. Performed best-effort validation by
inspecting changed files and checking consistency against the requested
behavior.`

Note: `Axiom/package.json` defines real validation commands (`npm run
build`, `npm test`, `npm run doctor`, `npm run typecheck`), but this
increment made no code changes to validate — running them would only
assert pre-existing state, not anything produced by this audit.

Best-effort validation performed for this audit:

- Directly read `Axiom/packages/isolation/src/{p0.ts,index.ts}` in full,
  not inferred from the roadmap's own description.
- Grepped every reference to `buildIsolationContext`, `pathsAreIsolated`,
  `checkMcpAllowed`, `DEFAULT_ALLOWED_MCP_SERVERS` across the whole
  `Axiom/` tree to confirm production call sites (only
  `@axiom/doctor`'s `runIsolationChecks`) vs. dead exports
  (`pathsAreIsolated`, confirmed zero non-test call sites).
- Directly read `Axiom/packages/user-workspace/src/{registry.ts,
  registry-types.ts}` in full to confirm registry v2's `repos` map shape
  and its `getProjectV2`/`listProjectsV2` read functions.
- Grepped the full `Axiom/packages/**/*.ts` tree (excluding `dist/`) for
  `cache`, `CacheEntry`, `.axiom/cache` to confirm no persistent cache
  mechanism exists, consistent with `general-spec.md`'s already-recorded
  INC-07 deferral.
- Read all 8 files under `Axiom/packages/mcp-tools/src/` in full,
  confirming every handler's input signature takes a single resolved
  project/repo context, never two.
- Grepped the full `Axiom/` tree for `mcp.yml`, `mcp.json`,
  `mcp-manifest` — found zero `mcp.yml`/`mcp.json` (confirms genuinely
  new), but found the pre-existing, unrelated `mcp-manifest.yaml` /
  `axiom mcp list|validate` (spec 0024) artifact, read in full
  (`Axiom/apps/cli/src/commands/mcp.ts`) to avoid designing a colliding
  or duplicate schema.

## Result

The roadmap's INC-15 hypothesis about `@axiom/isolation` is confirmed
accurate (not stale) on its narrow technical claim: `buildIsolationContext`/
`buildProjectScopedPaths` do genuinely assume one repo root with
sub-paths, unchanged since spec 0014 P0 and untouched by INC-01/13/14.
However, the *practical* isolation-audit scope INC-15 originally implied
("extend `buildIsolationContext` for multi-repo") turns out to be
premature/speculative once the actual call graph is traced: the one
production call site (`@axiom/doctor`'s `runIsolationChecks`) and the one
cross-project collision function (`pathsAreIsolated`) both have no live
multi-repo consumer today, and `@axiom/mcp-tools`'s entire handler surface
already structurally prevents cross-project leakage by never accepting
two `projectId`s in one call. Addendum §17's two explicit validation asks
("`targetRepoId` belongs to `projectId`", "`specRepo`/`sddRepo` belong to
the same project") are both already trivially answerable from registry
v2's existing `getProjectV2`/`listProjectsV2` data — no new isolation
package logic needed, only a thin guard clause at whichever call site
later wants it enforced. The "namespace caches by `projectId`" concern is
confirmed moot today because no persistent cache exists anywhere in the
codebase (consistent with INC-07's already-recorded deferral).

The one genuinely new, concretely-scoped, addendum-mandated artifact is
`mcp.yml` (source doc §14.2 / addendum §13): it does not exist anywhere
in the codebase, and its schema is fully designed above, reconciled
against the pre-existing, unrelated `mcp-manifest.yaml` (spec 0024)
artifact this audit surfaced and flagged (Q-mcp-1) rather than silently
overwrote or ignored. Per-adapter generated configs
(`adapters/<tool>/mcp.json`) have a proven, six-times-shipped generator
pattern to copy from `@axiom/installer`/`packages/adapters/*`.

Net assessment: **INC-15's original scope is roughly 60-70% already
satisfied by INC-01/13/14's landed work** (isolation-by-construction via
narrow call signatures, registry-v2 data sufficient for both addendum
§17 asks, no live cache to namespace) and **30-40% genuinely new**
(`mcp.yml` schema + loader/validator + per-adapter generated config),
with the multi-repo `@axiom/isolation` extension the roadmap originally
anticipated demoted from "needed now" to "documented future
consideration, blocked on a concrete consumer" (Q-mcp-2).

## General spec integration

Not yet integrated — this increment is `pending`, not `closed`, per
`Axiom.SDD/AGENTS.md`'s closure rules (Q-mcp-1 is a real open question
blocking the next role's schema-loader design, and no code was written to
document as "landed"). Once a follow-up role implements `mcp.yml` +
per-adapter generation and this chain closes, `general-spec.md` should
gain a new "`mcp.yml` project-scoped MCP config" section analogous to the
existing "MCP tool registration" section, explicitly cross-referencing
and distinguishing it from the pre-existing `mcp-manifest.yaml` / `axiom
mcp list|validate` artifact this audit surfaced, so a future reader does
not mistake the two for the same thing.

## Next step recommendation

Skip the `schema-writer` role. This audit's own §5 output already
provides a complete, implementer-ready `mcp.yml` schema (types + six
concrete validation rules), consistent with this chain's established
precedent (INC-07, INC-11) of collapsing a planned role hop when an
earlier role's output already satisfies the next role's design need —
introducing a separate schema-writer pass here would just restate this
section without adding information.

Recommended next role: an **implementer** (the roadmap's
`adapter-engineer`, but its first task should be the `mcp.yml`
loader/validator in `@axiom/user-workspace` or a new small package, not
adapter work yet) should:

1. Resolve Q-mcp-1 first (explicit decision: keep `mcp.yml` and
   `mcp-manifest.yaml` fully separate, or design a reconciliation) —
   this blocks the loader's own scope.
2. Implement `mcp.yml` load/validate following this audit's §5 schema and
   six validation rules verbatim.
3. Implement per-adapter `mcp.json` generation for at least one adapter
   (opencode or claude-code, both of which already have marker-block
   preservation infra) by copying the existing
   `GENERATED_FILES_BY_TARGET`/`generator.ts` pattern, not inventing a
   new one.
4. Defer the `@axiom/isolation` multi-repo extension (Q-mcp-2) explicitly
   in the implementer's own spec, rather than silently dropping it or
   silently building it unrequested.
5. Route through `validator-reviewer` afterward, per the roadmap's
   original sequence, extending `@axiom/doctor` with a new check category
   for `mcp.yml` validity following the existing `pass/fail/warn/skip` +
   evidence convention (mirroring `TC-012`/`TC-013`'s skip-when-absent
   posture, since `mcp.yml` is project-optional, not mandatory).
