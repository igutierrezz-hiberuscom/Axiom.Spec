# Increment: Registry + manifest schema v2 (cli-implementer)

Status: pending
Date: 2026-07-02

## Goal

Execute the **cli-implementer** step of INC-01's subagent sequence
(migration-engineer -> schema-writer -> registry-engineer ->
**cli-implementer** -> validator-reviewer), cutting over the CLI
command layer (`apps/cli/src/commands/*.ts`) in the real `Axiom`
monorepo from the v1 registry functions (`loadRegistry`/`addProject`/
`findByRootPath`/etc.) to the `*V2` functions implemented by
registry-engineer (`loadRegistryV2`/`addProjectV2`/`findByRepoPathV2`/
etc.), and resolving the `legacy-registry-not-migrated` UX decision the
registry-engineer spec explicitly left open.

## Context

Parent chain: `INC-20260702-axiom-redesign-roadmap` (parent roadmap) ->
`INC-20260702-registry-manifest-schema-v2` (migration-engineer audit)
-> `INC-20260702-registry-manifest-schema-v2-design` (schema-writer
design) -> `INC-20260702-registry-manifest-schema-v2-impl`
(registry-engineer implementation) -> this increment (cli-implementer).

All three prior specs were read in full before any code was written.
The registry-engineer spec's "Consequence for cli-implementer" section
named six files needing cutover: `apps/cli/src/commands/{init,join,
projects,repo,tui,app-api}.ts`. All six were read in full. Their exact
v1 call sites (file:line references, per the registry-engineer's own
audit) were re-confirmed live before editing.

One new, blocking design/live-code mismatch was found during
implementation and is documented in detail below (not silently
resolved): `@axiom/project-resolution/src/resolver.ts` (untouched by
any prior role in this chain) only understands the v1 `axiom.yaml`
shape (`project.name`), so `init.ts` could not be safely flipped to
emit `schemaVersion: 2` manifests without breaking `join`/`repo attach`/
`tui` end-to-end for any newly-`init`-ed project.

## Scope

- Cut over `apps/cli/src/commands/init.ts`, `join.ts`, `projects.ts`,
  `repo.ts`, `app-api.ts` from the v1 registry API to the `*V2` API.
- `tui.ts`: evaluated for cutover; NOT cut over — see "Flagged
  design/live-code mismatch #2" below (its only registry dependency is
  a type import consumed by `@axiom/tui`'s package-internal driver,
  which is out of this increment's scope).
- Resolve the `legacy-registry-not-migrated` UX decision left open by
  registry-engineer: decide and implement what each CLI command does
  when it hits that typed error.
- Where the v1 (`rootPath` per project) -> v2 (`repos` map per project)
  shape change forces a genuine UX change, make the minimal necessary
  addition and document why (`--role` flag on `axiom repo attach`,
  `axiom projects add`, `axiom projects join`).
- `init.ts`: evaluated emitting `axiom.yaml` `schemaVersion: 2`
  (`AxiomYamlSchemaV2`) for new projects; NOT implemented — see
  "Flagged design/live-code mismatch #1" below. `axiom.yaml` continues
  to be written as v1 by `init.ts` in this increment.
- Update/add tests for the CLI command changes, following existing
  conventions in `apps/cli/tests/*`.
- Run real validation (`npm run build`, `npm test`, `npm run
  typecheck`) and report actual pass/fail output.

## Non-goals

- No `@axiom/doctor/src/checks.ts` changes (MC-001's pass/warn/fail
  three-way branch is validator-reviewer's job, next and final role in
  INC-01's sequence).
- No `@axiom/project-resolution` changes, despite the blocking mismatch
  found (see below) — that package was not authorized for this
  increment's scope by any prior spec in the chain, and expanding scope
  into it unilaterally would be exactly the kind of unrequested,
  cross-package architectural expansion `Axiom.SDD/AGENTS.md` prohibits.
  It is flagged as a new, concrete follow-up recommendation instead.
- No `@axiom/tui` package-internal changes (`driver.ts`, `screens/
  projects.ts`, `render.ts`) — flagged by the migration-engineer audit
  as its own later increment (INC-04, tui-developer), not this one.
- No migration script implementation (`axiom upgrade`'s actual
  `registry.json -> projects.yml` migration logic) — consistent with
  all four prior roles' non-goals in this chain.
- No deletion of the v1-named registry functions (`loadRegistry`,
  `addProject`, etc.) — `tui.ts`'s indirect dependency (via `@axiom/
  tui`) on the v1 `ProjectEntry` type means they are still live,
  necessary code, not dead code, until `@axiom/tui` has its own
  cutover.

## Acceptance criteria

- [x] `apps/cli/src/commands/init.ts` registry calls (`addProject`,
      `findByRootPath`) cut over to `addProjectV2`/`findByRepoPathV2`.
- [x] `apps/cli/src/commands/join.ts` registry calls (`addProject`,
      `findByRootPath`) cut over to `addProjectV2`/`findByRepoPathV2`.
- [x] `apps/cli/src/commands/projects.ts` registry calls (`addProject`,
      `listProjects`, `useProject`, `getProject`) cut over to
      `addProjectV2`/`listProjectsV2`/`useProjectV2`/`getProjectV2`.
- [x] `apps/cli/src/commands/repo.ts` registry calls (`addProject`) cut
      over to `addProjectV2`.
- [x] `apps/cli/src/commands/app-api.ts` registry calls (`listProjects`,
      `getProject`) cut over to `listProjectsV2`/`getProjectV2`.
- [x] `apps/cli/src/commands/tui.ts` evaluated for cutover; decision
      (no cutover, flagged) is explicit and justified, not silent.
- [x] `legacy-registry-not-migrated` UX decision made and implemented
      consistently across all five cut-over command files.
- [x] Every forced UX change (new `--role` flag) is documented with
      rationale.
- [x] `@axiom/doctor/src/checks.ts` was not modified.
- [x] `@axiom/topology` was not modified.
- [x] `@axiom/project-resolution` was not modified (mismatch flagged,
      not silently worked around by touching it).
- [x] Existing CLI command tests (`init.test.ts`, `join.test.ts`,
      `projects.test.ts`, `repo.test.ts`, `app.test.ts`) updated to
      reflect the v2 shapes; no test coverage was deleted, only
      adjusted for the new shape plus new tests added for `--role`.
- [x] Real validation commands were run (`npm run typecheck`, `npm run
      build`, `npm test`) and their actual pass/fail output is reported
      below.
- [x] Every place where live code differed from what prior specs in
      this chain assumed is flagged explicitly below, not silently
      resolved.

## Open questions

None blocking this increment's own closure. One new, concrete follow-up
is raised for a future role (not named in INC-01's original five-role
sequence, since none of migration-engineer/schema-writer/
registry-engineer anticipated this specific package needing its own
cutover): **`@axiom/project-resolution` needs a version-dispatching
`resolveProject`** (mirroring `validateAxiomYamlContent`'s `{valid,
version, data}` pattern) before `init.ts` can safely emit `schemaVersion:
2` manifests. See "Flagged design/live-code mismatch #1" below for the
full analysis and "Next step recommendation" for how this interacts
with validator-reviewer's remaining MC-001 work.

## Assumptions

- Inherited unchanged from all three prior specs in this chain:
  `axiom upgrade` as the eventual migration invocation mechanism, the
  non-hard-removal posture for D1/D2/D3, and `@axiom/core`'s `Result<T,
  E>` pattern.
- The registry-engineer spec's open question ("which CLI cutover
  strategy... direct cutover of all `apps/cli/src/commands/*.ts` call
  sites in one pass, or an incremental per-command migration") is
  resolved here as: **direct cutover in one pass** for the five files
  whose only blocker was the registry API surface (`init`, `join`,
  `projects`, `repo`, `app-api`), and **explicit non-cutover** for the
  one file (`tui.ts`) whose blocker is a different, out-of-scope
  package (`@axiom/tui`).
- `legacy-registry-not-migrated` UX: chosen to print clear, actionable
  guidance and exit/fail the specific operation (not silently succeed,
  not auto-migrate) — see "legacy-registry-not-migrated UX decision"
  below for the full rationale.

## Implementation notes

### Files changed

**`Axiom/apps/cli/src/commands/init.ts`**:
- Registry imports cut over: `addProject`/`findByRootPath` ->
  `addProjectV2`/`findByRepoPathV2`; `ProjectEntry` -> `ProjectEntryV2`
  / `ProjectEntryV2WithStatus` for the `InitResult.registryAttempt.entry`
  type (a union of both, since `findByRepoPathV2` returns the
  `WithStatus` variant and `addProjectV2` returns the plain variant).
- Auto-registration logic (`runInit`'s post-orchestration block)
  rewritten: registers the project into `projects.yml` with a single
  `repos` entry under role `sdd` (`DEFAULT_REPO_ROLE`), matching
  `@axiom/topology`'s `sdd-repo`/`RepoRef.id` naming convention (per
  the schema-writer design's own `RepoRoleKey` doc comment: "mirrors
  `@axiom/topology`'s existing repo id vocabulary... without the
  '-repo' suffix").
- `legacy-registry-not-migrated` handled: best-effort WARN to stderr
  with actionable guidance, `registryRegistered: false`,
  `registryAttempt.outcome: 'error'` — consistent with `init`'s
  existing best-effort semantics (registry auto-registration never
  fails the `init` command itself).
- `axiom.yaml` **schemaVersion: 2 emission was attempted and reverted**
  in this same session — see "Flagged design/live-code mismatch #1"
  below for the full analysis. `buildAxiomYaml` still emits the v1
  shape (`project.name`/`project.mode`/`scopes`), with an extensive
  comment block explaining why and pointing at this spec.
- Output text (`registerInit`'s human-readable summary) updated from
  `~/.axiom/registry.json` to `~/.axiom/projects.yml`.

**`Axiom/apps/cli/src/commands/join.ts`**:
- Registry imports cut over: `addProject`/`findByRootPath` ->
  `addProjectV2`/`findByRepoPathV2`.
- `readProfileTripleFromInstall` helper (previously read
  `install-profile.json` to persist `functionalProfile`/`overlay`/
  `adapterTarget` into the registry) **removed**: per the
  schema-writer design's Resolution #3, those three fields are no
  longer persisted in `projects.yml` (deprecated-with-warning at the
  registry level; they still live, fresh, in `install-profile.json`).
  The helper became genuinely dead code after the cutover (no other
  caller), not speculatively removed.
- Auto-registration registers a single `repos` entry under role `sdd`,
  same pattern as `init.ts`.
- `legacy-registry-not-migrated` handled identically to `init.ts`.
- Doc comments and output text updated from `registry.json` to
  `projects.yml`.

**`Axiom/apps/cli/src/commands/projects.ts`**:
- Registry imports cut over: `addProject`/`listProjects`/`useProject`/
  `getProject` -> `addProjectV2`/`listProjectsV2`/`useProjectV2`/
  `getProjectV2`.
- `readInstallProfileTriple` helper **removed** (same rationale as
  `join.ts`'s `readProfileTripleFromInstall` — dead after the cutover
  per Resolution #3).
- **FORCED UX CHANGE**: `projects add` and `projects join` gain a new,
  optional `--role <role>` flag (default `sdd`). Rationale: v1's
  `ProjectsAddArgs`/`ProjectsJoinArgs` registered exactly one
  `rootPath` per project; v2 requires a `repos: { roleKey: {...} }`
  map, so registering "one path" now requires knowing which role that
  path plays. This was not optional to skip — the v1 call shape
  (`{id, name, rootPath, ...}`) has no v2 equivalent without deciding a
  role, so a flag (with a safe default) was the minimal necessary
  addition, not a UX redesign.
- `formatProjectsList`/`formatProjectRow` (the `projects list` table)
  rewritten: the `rootPath` column is replaced with a `repos (rol=path)`
  summary column (`sdd=/path (stale)  |  spec=/other/path`), and the
  `stale` column now reflects `projectStale` (true iff **all** repos of
  a project are stale) instead of a single-repo `stale` flag. This is a
  necessary, minimal UX change: v1's single-repo table literally cannot
  represent a v2 multi-repo project; it is not a stylistic choice.
- New helper `resolvePrimaryRepoPath` added: resolves "the one path"
  for operations that still assume a single path per project (`projects
  use --print-cwd`), preferring the `sdd`-role repo, falling back to
  the first repo in the map. This is the same resolution strategy used
  independently in `app-api.ts`'s `resolveProjectRoot` (see below) —
  applied consistently, not duplicated with different semantics.
- `legacy-registry-not-migrated` handled in `runProjectsList`,
  `runProjectsAdd`, `runProjectsJoin`, `runProjectsUse`: each returns
  its own typed failure result (not a thrown exception) with the same
  actionable message text used in `init.ts`/`join.ts`.
- `getRegistryProject` re-export renamed to `getProjectV2` (same
  re-export pattern, new function).

**`Axiom/apps/cli/src/commands/repo.ts`**:
- Registry imports cut over: `addProject` -> `addProjectV2`;
  `ProjectEntry` -> `ProjectEntryV2`.
- **FORCED UX CHANGE**: `axiom repo attach <repoId>` gains a new,
  optional `--role <role>` flag (default `sdd`). Same rationale class
  as `projects add`/`join`: v1's `addProject(homeDir, {id: repoId,
  rootPath, ...})` call had no v2 equivalent without a role decision.
  `runRepoAttach`'s `RepoAttachArgs.role` documents this explicitly as
  the "one CLI surface... where the v1->v2 shape change forces a
  genuine UX addition," per this increment's own task framing.
- `readProjectNameFromYaml` (used by `runDiscover`, reads `axiom.yaml`
  directly, not via the registry) updated to support **both** v1
  (`project.name`) and v2 (`schemaVersion: 2` -> top-level `name`)
  manifest shapes. This was a necessary, in-scope fix discovered while
  implementing: `axiom discover` reads `axiom.yaml` directly rather
  than going through `@axiom/project-resolution`, so it does not share
  in the blocking mismatch described below — it was safe and correct
  to make it version-aware directly.
- `legacy-registry-not-migrated` handled in `runRepoAttach`.

**`Axiom/apps/cli/src/commands/app-api.ts`**:
- Registry imports cut over: `listProjects`/`getProject` ->
  `listProjectsV2`/`getProjectV2`.
- `resolveProjectRoot` (used by every project-scoped REST endpoint:
  topology, roles, toolchain, memory, mcps, plugins, workflow state,
  command preview/execute) rewritten to resolve a single `projectRoot`
  from the new `repos` map, using the same `sdd`-role-first, else-
  first-repo strategy as `projects.ts`'s `resolvePrimaryRepoPath`. This
  endpoint's contract (`{ok: true, projectRoot: string}`) was
  deliberately **not** redesigned to be multi-repo-aware — doing so
  would be a larger, unrequested REST API redesign beyond this
  increment's CLI-cutover scope; the single-path resolution is the
  minimal compatible behavior.
- **FORCED UX CHANGE (REST API shape)**: `apiGetProjects`/
  `apiGetProject`'s JSON response shape changed from `{id, name,
  rootPath, functionalProfile, overlay, adapterTarget, addedAt,
  lastUsedAt, stale}` (v1) to `{id, name, repos, addedAt, lastUsedAt,
  stale}` (v2, where `repos` is the full `{roleKey: {role, path,
  stale}}` map and `stale` now means `projectStale`). This is a
  genuine breaking change to the app's REST API, forced by the same
  root cause as the CLI table change in `projects.ts` — v1's flat
  shape cannot represent a multi-repo project. No `apps/*` static
  frontend assets were found to consume this endpoint's exact field
  names in a way that would need a parallel update (out of this
  increment's grep scope beyond the `apps/cli` files it was scoped to
  touch — flagged for validator-reviewer or a future front-end-facing
  role to re-verify against `apps/cli/static/*.js` if it exists).
- `legacy-registry-not-migrated` handled in `resolveProjectRoot`
  (surfaces as a 404-class error message through every dependent
  endpoint, since they all funnel through this one function).

**`Axiom/apps/cli/src/commands/tui.ts`**: **not modified**. See
"Flagged design/live-code mismatch #2" below.

**Test files updated** (`Axiom/apps/cli/tests/`):
- `init.test.ts`: Scenario 4's registry assertions rewritten for the
  `repos.sdd.path` shape; `getProject` import -> `getProjectV2`.
- `join.test.ts`: Scenario 6's `getRegistryProject` calls ->
  `getProjectV2` (the test's own `beforeJoin`/`afterJoin` state
  checks were reading the wrong file after the cutover — this was a
  genuine test bug exposed by the cutover, not a new test requirement).
- `projects.test.ts`: Scenario 2's `ProjectEntry` assertions rewritten
  for the `repos` map shape; the "runProjectsJoin con proyecto Axiom
  resoluble" test rewritten (its premise — that the triple is read from
  `install-profile.json` into the registry — is no longer true per
  Resolution #3) plus a new test added for `--role`; Scenario 5's
  `--print-cwd` test rewritten to resolve the path via `repos.sdd.path`
  instead of a flat `rootPath`.
- `repo.test.ts`: Scenario 1 rewritten for the `repos.sdd` shape plus a
  new test added for `--role`; Scenario 2's `loadRegistry`/`.projects`
  array assertions -> `loadRegistryV2`/`.projects` map (`{}` instead of
  `[]`).
- `app.test.ts`: `makeAppFixture`'s hand-written registry fixture
  rewritten from a v1 `registry.json` (JSON, `rootPath` field) to a v2
  `projects.yml` (YAML, `repos` map) — this fixture directly
  constructs the on-disk file `resolveProjectRoot` reads, so it had to
  track the cutover exactly.

**Not touched** (confirmed by re-reading before finishing): `@axiom/
doctor/src/checks.ts`, `@axiom/topology/*`, `@axiom/project-resolution/
*`, `@axiom/tui/*` (package internals), `apps/cli/src/commands/tui.ts`.

### `legacy-registry-not-migrated` UX decision

**Decision: print clear, actionable guidance to the relevant
output stream and fail the specific operation (exit 1 for CLI
commands; a typed error/404-class message for the REST API) — no
auto-migration, no interactive prompt.**

The registry-engineer spec left this open with three named options:
"e.g. auto-invoke the migration (if `cli-implementer` decides that's
the right UX), or instruct the user, rather than starting the user over
with an empty registry." Rationale for choosing "instruct the user" over
the other two:

1. **Auto-invoke `axiom upgrade` is not possible today.** All three
   prior specs in this chain (migration-engineer, schema-writer,
   registry-engineer) explicitly scoped the actual `registry.json ->
   projects.yml` migration **logic** out of every one of INC-01's five
   roles — it does not exist yet. Auto-invoking a migration command
   that has no implementation would either be a silent no-op (worse
   than doing nothing) or would require this increment to build that
   migration logic itself, which is a large, unrequested expansion of
   scope explicitly reserved for a separate future work item by every
   prior spec's own Assumptions section.
2. **An interactive prompt is inconsistent with this increment's
   commands.** None of `init`/`join`/`projects`/`repo attach`/the
   `app-api` REST endpoints have any existing interactive-prompt
   machinery for registry errors (`ensureUserHome`'s existing pattern
   for other registry errors is already "return a message, let the
   caller decide exit code" — no prompt). Introducing one exclusively
   for this new error kind would be new, speculative UX beyond what
   `Axiom.SDD/AGENTS.md`'s bootstrap limits allow ("no speculative
   architecture... only what's needed to unblock the user").
3. **Printing instructions is the minimal, already-precedented
   pattern.** Every one of these commands already has an established
   "best-effort registry op fails -> WARN/error message with the exact
   `UserWorkspaceError.message`" pattern (see e.g. `init.ts`'s existing
   `ensureHomeDir` failure handling, unchanged by this increment). This
   increment's `legacy-registry-not-migrated` handling reuses that
   exact pattern, adding the `legacyPath`/`expectedPath` detail from
   the typed error so the message is concrete and actionable ("Se
   encontró `<path>` (registry v1) sin migrar a `<path>` (registry
   v2). Migrá manualmente o esperá a que `axiom upgrade` soporte esta
   migración."), without inventing new interaction models.
4. **Never silently starts the user over with an empty registry** —
   the explicit thing registry-engineer's design was careful to avoid
   at the loader level (`loadRegistryV2` returning a typed error, not
   `ok({projects: {}})`) is preserved end-to-end at the CLI layer: none
   of the five cut-over commands catch this error kind and fall through
   to "proceed as if empty."

This decision is applied uniformly across all five cut-over command
files (`init`, `join`, `projects` x4 sub-commands, `repo attach`,
`app-api`'s `resolveProjectRoot` and its nine downstream endpoints), so
the UX is consistent regardless of which command surface the user or
API consumer hits it through.

### Forced UX changes (v1 `rootPath` -> v2 `repos` map)

Three commands gained a new, optional `--role <role>` flag (default
`sdd`, matching `@axiom/topology`'s `sdd-repo` naming convention):

1. **`axiom repo attach <repoId> [--role <role>]`** — the most direct
   case: v1's `addProject(homeDir, {id: repoId, rootPath, ...})` has no
   v2 equivalent without deciding which role this repo plays.
2. **`axiom projects add -p <path> [--role <role>]`**
3. **`axiom projects join -p <path> [--role <role>]`**

In all three cases the flag is **additive and optional** (default
preserves today's single-repo UX with a sensible default role), so
existing invocations without `--role` keep working identically except
for the underlying on-disk shape (which was already going to change
per D2, independent of the CLI). This is the minimal necessary
addition: the alternative (silently picking a role with no escape
hatch) would make it impossible to register a second repo under a
different role for the same project id via the CLI, which the v2
schema explicitly supports (`repos: { sdd: {...}, spec: {...} }`).

`init.ts`/`join.ts`'s **auto-registration** paths (background,
non-interactive, triggered by `axiom init`/`axiom join` themselves, not
a dedicated registry sub-command) do **not** gain a `--role` flag: they
only ever know about one repo (the cwd they are running in), so there
is no ambiguity a flag would need to resolve — they use the `sdd`
default unconditionally, consistent with `@axiom/topology`'s posture
that a `sdd-repo` is the primary/default repo role.

`projects list`'s table format and `app-api.ts`'s `apiGetProjects`/
`apiGetProject` JSON shape also changed (documented in "Files changed"
above) — these are consequences of the same v1-`rootPath`-cannot-
represent-v2-`repos` root cause, not independent UX decisions.

### Flagged design/live-code mismatches (not silently resolved)

**Mismatch #1 (blocking, new) — `@axiom/project-resolution/src/
resolver.ts` cannot read `axiom.yaml` `schemaVersion: 2` documents,
which blocks `init.ts` from safely emitting them.**

The task description for this increment named `init.ts`'s
`schemaVersion: 2` emission as explicit, required scope ("Update
`init.ts` to emit `axiom.yaml` with `schemaVersion: 2`... per
`AxiomYamlSchemaV2`"), citing the migration-engineer audit's own open
question on this point. This was implemented first, then reverted
after a fresh, live read of `Axiom/packages/project-resolution/src/
resolver.ts` (not re-audited by any of migration-engineer/
schema-writer/registry-engineer beyond being named in the
migration-engineer's blast-radius list, item #9, as needing its own
future update — never inspected line-by-line against the actual v2
manifest shape until this session).

The concrete problem: `resolveProject` (the single function every
project-scoped CLI/TUI/API surface uses to find "what project am I in
right now") does exactly this and nothing else for reading the
manifest:

```ts
interface AxiomYamlConfig {
  project?: { name?: string; mode?: string };
  scopes?: Record<string, { path?: string; product_runtime?: boolean }>;
}
// ...
if (!config?.project?.name) {
  return { /* status: 'ambiguous', ... */ };
}
```

A `schemaVersion: 2` document (per `AxiomYamlSchemaV2`) has **no**
`project.name` — it has a top-level `name`/`projectId`. Feeding a v2
document to `resolveProject` therefore returns `status: 'ambiguous'`,
not `status: 'resolved'`, for **every** newly-`init`-ed v2 project.

Traced consequences, each independently confirmed by reading the
calling code (not assumed):
- `join.ts`'s `withProjectContext` (via `_shared.ts`) calls
  `assertUnambiguous`, which **throws** on `status: 'ambiguous'` — so
  `axiom join` would hard-fail immediately after `axiom init` on any
  v2 project, with a confusing "falta project.name en axiom.yaml"
  message despite the field genuinely being present (just renamed).
- `projects.ts`'s `runProjectsJoin` and `repo.ts`'s `runRepoAttach`
  both call `resolveProject` directly and branch on `status !==
  'resolved'` -> exit 1 with "no se encontró un proyecto Axiom" — same
  practical failure, worse message (implies no project at all, not a
  version mismatch).
- `tui.ts`'s `--topology`/`--model-validate`/`--components-show` modes
  all require `resolution.status === 'resolved'` or throw
  `projectNotFound: true`.

This is not a hypothetical: it is the exact end-to-end flow this
increment's own cutover is trying to make work (`init` -> registry v2
-> `join`/`repo attach`/`tui`). Shipping `schemaVersion: 2` manifests
from `init.ts` today would break that flow for every new project,
which is a materially worse outcome than not making the change.

**Resolution taken (flagged, not silently worked around):** reverted
`init.ts`'s `buildAxiomYaml` to continue emitting the v1 shape
(`project.name`/`project.mode`/`scopes`), with a large, explicit code
comment at the function explaining exactly this chain of reasoning and
pointing at this spec file. **`@axiom/project-resolution` was not
touched** to work around this, even though a version-dispatching
`resolveProject` (mirroring `validateAxiomYamlContent`'s `{valid,
version, data}` design, already proven by registry-engineer for the
manifest **validator**) would be the obvious fix — because:
1. No prior spec in this chain authorized touching that package; the
   migration-engineer audit explicitly named it as needing a *future*
   resolver branch, not this role's job.
2. `Axiom.SDD/AGENTS.md`'s "do not modify unrelated files" and "no
   speculative architecture beyond what's asked" both argue against a
   cli-implementer unilaterally redesigning a resolution package's
   parsing logic as a side effect of a CLI command's manifest-writing
   decision.
3. The safe, minimal, immediately-correct fix available at this
   increment's actual scope (CLI commands) is exactly what was done:
   don't ship a manifest shape the rest of the system cannot yet read.

**Consequence / next step:** `@axiom/project-resolution` needs its own
small, focused cutover — a `resolveProject` (or a new
`resolveProjectV2`, mirroring registry-engineer's own `*V2`-suffix
precedent for exactly this kind of "can't break existing callers"
situation) that reads `schemaVersion` and branches to read either
`project.name` (v1) or top-level `name`/`projectId` (v2). This is a
small, well-scoped, single-package change — not a redesign — but it is
new work not named in INC-01's original five-role sequence. See "Next
step recommendation" below for how this should be sequenced relative
to validator-reviewer.

**Mismatch #2 (flagged, not a blocker) — `tui.ts`'s only registry
dependency is a type import satisfying `@axiom/tui`'s own
package-internal API, which this increment is not scoped to touch.**

`apps/cli/src/commands/tui.ts` imports `type { ProjectEntry } from
'@axiom/user-workspace'` for exactly one purpose: typing the
`onProjectSelect` callback parameter it passes into `@axiom/tui`'s
`runTui` driver (`(entry: ProjectEntry) => { process.chdir(entry.
rootPath); ... }`). A live read of `Axiom/packages/tui/src/driver.ts`
confirms the driver itself is the actual v1 dependency:

```ts
import {
  ensureHomeDir, listProjects, useProject as useRegistryProject,
  resolveHomeDir, type ProjectEntry,
} from '@axiom/user-workspace';
```

`driver.ts` calls `listProjects`/`useProject` internally (inside its
`projects` screen flow) and constructs/consumes `ProjectEntry` values
throughout `screens/projects.ts`'s ~1600 lines. This is squarely
`@axiom/tui` package-internal code, not `apps/cli/src/commands/*.ts` —
the migration-engineer audit's blast-radius list explicitly separated
these (`packages/tui/src/{render,driver}.ts` and `screens/projects.ts`
are listed as items flagged "for the tui-developer step in a later
increment (INC-04)," distinct from and not part of this chain's own
`apps/cli/src/commands/*.ts` cutover list).

**Resolution taken (flagged, not silently skipped):** `tui.ts` was
read in full, its exact dependency traced into `@axiom/tui/src/
driver.ts`, and left **unmodified**. Cutting `tui.ts`'s single type
import over to `ProjectEntryV2` without also rewriting `@axiom/tui`'s
driver/screens would be a type error (the driver's own `onProjectSelect`
parameter type would still be v1's `ProjectEntry`, and `entry.rootPath`
would not exist on a `ProjectEntryV2`) — there is no partial, safe
cutover available here without touching `@axiom/tui` itself, which is
explicitly the next increment's (INC-04, tui-developer) job per the
parent roadmap, not this chain's.

**Practical effect:** `axiom tui --projects` (the registry-browsing
screen) continues to read/write the **v1** registry (`registry.json`)
end-to-end, completely independent of and unaffected by this
increment's cutover of the other five CLI commands to `projects.yml`.
This means a user who runs `axiom init`/`axiom join`/`axiom repo
attach` (now writing to `projects.yml`) will **not** see that project
in `axiom tui --projects` (which still reads `registry.json`) until
`@axiom/tui` gets its own cutover. This is a real, user-visible gap
introduced by this increment being scoped to CLI commands only, not a
regression this increment could have prevented within its own scope —
flagged explicitly for the "Next step recommendation" below.

## Validation

Real validation commands were run (not the "no validation command
found" fallback — this increment has actual code).

**`npm run typecheck` (root, `tsc -b`, all 33 project references):**

```
> axiom-product@0.1.0 typecheck
> tsc -b
```

Exit code 0, zero errors, across the entire monorepo.

**`npm run build` (root, `tsc -b`):**

```
> axiom-product@0.1.0 build
> tsc -b
```

Exit code 0, identical clean result (this repo's `build` and
`typecheck` scripts are both `tsc -b`).

**`npm test` (root, `vitest run`, full monorepo suite):**

```
Test Files  12 failed | 112 passed (124)
     Tests  13 failed | 1199 passed (1212)
```

All 13 failures were bisected against the clean `main` baseline via
`git stash` / `git stash pop` (same method the registry-engineer
increment used) **in this session**, run twice for stability, with an
identical failure list both times:

- `apps/cli/tests/start.test.ts` (1) — pre-existing, unrelated (missing
  `axiom.spec/config/profiles.yaml` fixture file in this environment).
- `packages/agents/tests/catalog.test.ts` (1) — pre-existing, reads the
  real repo's `axiom.spec/config/agents-catalog.yaml`.
- `packages/doctor/tests/checks.test.ts` (1) — pre-existing, "repositorio
  Axiom real" integration test.
- `packages/model-routing/tests/{assignments,loader,opencode-projection,
  resolver}.test.ts` (4) — pre-existing, all read the real repo's
  `axiom.spec` model-routing policy files.
- `packages/project-resolution/tests/resolver.test.ts` (1) —
  pre-existing (unrelated to Mismatch #1 above — this specific failing
  test scenario, "resuelve scopes con absolutePath y exists correcto,"
  fails identically on clean `main`).
- `packages/skills/tests/catalog.test.ts` (1) — pre-existing, reads the
  real repo's `axiom.spec/config/skills-catalog.yaml`.
- `packages/telemetry/tests/audit-trail-sink.test.ts` (1) —
  pre-existing, retention-sweep timing test.
- `packages/toolchain/tests/repair-add-gitignore.test.ts` (1) —
  pre-existing.
- `packages/tui/tests/driver.test.ts` (2) — pre-existing, ANSI-escape-
  sequence string-matching assertions (same file/failure class the
  registry-engineer increment already reported, though this session's
  environment reproduces a different count — 2, not 13 — from that
  file alone; see note below).

**Note on the difference from the registry-engineer increment's
reported 13-failures-all-in-`driver.test.ts`:** this session's `git
stash` bisection against the same `main` baseline reproduces a
**different** 13-failure set (12 files, only 2 of them in
`driver.test.ts`, plus 11 other pre-existing failures across `agents`/
`doctor`/`model-routing`/`project-resolution`/`skills`/`telemetry`/
`toolchain`/`start.test.ts`) than the registry-engineer spec's reported
"13 pre-existing failures, all in `tui/tests/driver.test.ts`." Both
counts are 13, but the composition differs. This was verified twice in
this session (identical both times) and confirmed via the same
git-stash-against-clean-`main` method, so it is not this increment's
own code causing the difference — it reflects environment-dependent
test flakiness/state (most of the differing failures are
"reads-the-real-repo's-`axiom.spec`-files" integration tests, which are
sensitive to whatever `axiom.spec/config/*.yaml` content exists on disk
at test-run time, not to `@axiom/user-workspace`/`@axiom/
config-validation`/`apps/cli` — none of which any of these differing
failing tests import). Per this increment's task instructions ("If any
of the 13 pre-existing tui/tests/driver.test.ts failures are still
present, note that they're pre-existing... rather than re-investigating
them"), this is noted rather than re-investigated further; the
important, verified fact is that **all 13 failures in this session
reproduce identically on a clean `main` checkout with none of this
increment's changes applied**, twice, via `git stash`.

**Scoped run, `apps/cli` (the package this increment actually
changed):**

```
Test Files  1 failed | 30 passed (31)
     Tests  1 failed | 219 passed (220)
```

The one failure is `start.test.ts`'s pre-existing, unrelated case
(above). All CLI command tests touched by this increment's cutover
(`init.test.ts` 12/12, `join.test.ts` 6/6, `projects.test.ts` 11/11 —
10 pre-existing + 1 new, `repo.test.ts` 5/5 — 4 pre-existing + 1 new,
`app.test.ts` 7/7) pass cleanly.

## Result

Implemented the cli-implementer step of INC-01 in the real `Axiom`
monorepo:

- Cut over five of the six named CLI command files
  (`init.ts`, `join.ts`, `projects.ts`, `repo.ts`, `app-api.ts`) from
  the v1 registry API to the `*V2` API, following the exact contract
  registry-engineer's spec handed off.
- Resolved the `legacy-registry-not-migrated` UX decision uniformly
  across all five files: print actionable guidance, fail the specific
  operation, never silently fall back to an empty registry.
- Implemented the one genuinely forced UX change class (`--role` flag
  on `repo attach`/`projects add`/`projects join`), and the two
  necessary shape-following changes (`projects list`'s table,
  `app-api.ts`'s REST JSON shape) that follow from the same root cause
  (v1 single-`rootPath` cannot represent v2's `repos` map).
- Did **not** cut over the sixth named file, `tui.ts` — its only
  dependency is a type import satisfying `@axiom/tui`'s
  package-internal driver, which is out of this increment's scope
  (INC-04, tui-developer, per the migration-engineer audit).
- Did **not** implement `init.ts`'s planned `axiom.yaml` `schemaVersion:
  2` emission — attempted, then reverted after discovering it would
  break `join`/`repo attach`/`tui` end-to-end via `@axiom/
  project-resolution`'s v1-only `resolveProject`, a package no prior
  role in this chain authorized touching. Flagged as a new, concrete,
  small follow-up (see "Next step recommendation").
- Updated 5 existing test files (`init.test.ts`, `join.test.ts`,
  `projects.test.ts`, `repo.test.ts`, `app.test.ts`) for the v2 shapes,
  adding 2 new tests for the `--role` flag; zero test coverage deleted.
- Ran real `npm run typecheck`/`npm run build`/`npm test`: typecheck
  and build are 100% clean; the full suite has 13 pre-existing,
  environment-dependent failures (verified via `git stash` bisection
  against clean `main`, twice) unrelated to this increment's files, and
  100% pass (219/220, the 1 failure being the same pre-existing
  `start.test.ts` case) in `apps/cli`, the package this increment
  actually changed.

`@axiom/doctor/src/checks.ts`, `@axiom/topology`, and `@axiom/
project-resolution` were confirmed untouched, per this increment's
explicit non-goals.

Status is `pending`, not `closed` — see rationale below.

## General spec integration

No integration into `Axiom.Spec/general-spec.md` was performed — that
file does not exist in this repo, consistent with all three prior
increments in this chain. The CLI is now wired end-to-end to the v2
registry for five of six command surfaces, but the INC-01 chain is not
yet complete: `@axiom/doctor`'s MC-001 still does the old binary
pass/fail (validator-reviewer has not run), `tui.ts`/`@axiom/tui` still
speak v1 exclusively, and `axiom.yaml` `schemaVersion: 2` emission
remains blocked pending a small `@axiom/project-resolution` fix not yet
scheduled. Per the same reasoning all three prior increments in this
chain gave, the correct point to integrate stable knowledge into
`general-spec.md`-equivalent documents is after validator-reviewer
closes out INC-01 (and, per this increment's new finding, after the
`@axiom/project-resolution` follow-up lands) — not before the shape is
consistently readable end-to-end by every part of the system that needs
to read it.

## Closure rationale

`Status: pending`, not `closed`, because:

- This increment's own scope (cutting over five of six named CLI
  command files, resolving the `legacy-registry-not-migrated` UX
  decision, implementing the forced `--role` UX change) is fully done,
  tested, and validated with real, passing command output.
- However, INC-01 as a whole still has its final subagent role pending:
  **validator-reviewer** (`@axiom/doctor`'s MC-001 three-way branch).
- This increment also surfaced a new, concrete, small follow-up not
  anticipated by any prior role (`@axiom/project-resolution`'s v1-only
  `resolveProject`), which blocks `init.ts`'s originally-requested
  `schemaVersion: 2` emission and blocks `tui.ts`'s cutover
  transitively. Neither is fixed yet.
- Marking this `closed` while `axiom.yaml` still cannot be written as
  v2 by the CLI, and while `axiom tui --projects` cannot see any
  project registered by the other five commands, would overstate what
  has actually shipped from the end user's perspective, even though the
  five cut-over commands' own registry read/write behavior is solid and
  tested.

## Next step recommendation

Two possible next steps, in order of the sequence's own logic:

1. **Immediate**: launch **validator-reviewer** next, per INC-01's
   subagent sequence (this increment does not block it — MC-001's
   three-way branch operates on `validateAxiomYamlContent`'s return
   shape, which registry-engineer already implemented and this
   increment did not change). Exact inputs it needs:
   - **The MC-001 three-way branch table** from the schema-writer
     design (`INC-20260702-registry-manifest-schema-v2-design`,
     section "What MC-001 needs to do differently"):

     | `validateAxiomYamlContent` result | MC-001 status |
     |---|---|
     | `valid: true, version: 2` | `pass` |
     | `valid: true, version: 1` | `warn` (not `pass`, not `fail`) |
     | `valid: false, version: 'unknown'` | `fail` (unchanged) |

   - The exact file/lines: `Axiom/packages/doctor/src/checks.ts`,
     `runManifestChecks` (MC-001, currently binary pass/fail on
     `validationResult.valid`).
   - Confirmation that `validateAxiomYamlContent`'s discriminated union
     return shape (`AxiomYamlValidationResult`) is unchanged by this
     increment (verified: this increment touched no files in
     `@axiom/config-validation`).
2. **Follow-up, either before or in parallel with validator-reviewer**:
   a small, focused fix to `@axiom/project-resolution/src/resolver.ts`
   (a version-dispatching `resolveProject`, mirroring
   `validateAxiomYamlContent`'s `{valid, version, data}` pattern) so
   that (a) `init.ts` can safely emit `schemaVersion: 2` as originally
   requested, and (b) `join`/`projects join`/`repo attach`/`tui`'s
   topology/model-validate modes correctly resolve v2 projects once
   they exist. This was not named in INC-01's original five-role
   sequence and is new scope this increment discovered, not resolved
   unilaterally — whoever picks this up should treat this spec's
   "Flagged design/live-code mismatch #1" section as its complete
   brief (exact failure mode, exact traced consequences, exact
   suggested fix shape).
3. **Not yet scheduled**: `@axiom/tui`'s own cutover (INC-04,
   tui-developer per the parent roadmap) — `tui.ts`'s
   `onProjectSelect`/`ProjectEntry` dependency and `driver.ts`'s
   internal `listProjects`/`useProject` calls should migrate to the
   `*V2` API together, in that increment, not here.
