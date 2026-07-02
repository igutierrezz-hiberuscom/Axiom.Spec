# Increment: Registry + manifest schema v2 (schema-writer design)

Status: pending
Date: 2026-07-02

## Goal

Execute the **schema-writer** step of INC-01's subagent sequence
(migration-engineer -> **schema-writer** -> registry-engineer ->
cli-implementer -> validator-reviewer), following on from
`Axiom.Spec/specs/increments/INC-20260702-registry-manifest-schema-v2/README.md`
(the migration-engineer audit). This increment resolves every field marked
"needs schema-writer decision" in that audit, resolves the `axiom.yaml`
vs `axiom.yml` filename discrepancy, and produces the exact new
TypeScript types/schemas for `~/.axiom/projects.yml` (registry v2) and
`axiom.yml` `schemaVersion: 2`, plus the transition-window/loader/MC-001
behavior design. It does **not** implement any of this in
`@axiom/user-workspace`, `@axiom/topology`, or `@axiom/config-validation` —
that is the next role, registry-engineer.

## Context

Parent chain: `INC-20260702-axiom-redesign-roadmap` (parent roadmap) ->
`INC-20260702-registry-manifest-schema-v2` (migration-engineer audit,
status `pending`) -> this increment (schema-writer design).

The migration-engineer audit is treated as authoritative input and is not
re-litigated here except where it explicitly delegated a decision to
schema-writer. Two categories of open items were handed off:

1. Nine field-by-field "needs schema-writer decision" rows (five in the
   registry diff, four in the manifest diff — see migration-engineer
   README's Field-by-field diff tables, D2 and D3).
2. One filename discrepancy (`axiom.yaml` vs `axiom.yml`), explicitly
   assigned to schema-writer's discretion.

All resolutions below are made against the actual current source (read in
full for this increment): `Axiom/packages/user-workspace/src/{paths.ts,
registry.ts,registry-types.ts}`, `Axiom/packages/config-validation/src/
{schemas.ts,validator.ts}`, `Axiom/packages/filesystem-truth/src/
discovery.ts`, `Axiom/packages/topology/src/types.ts`, and
`Axiom/packages/doctor/src/checks.ts` (MC-001).

## Scope

- Resolve all 9 "needs schema-writer decision" fields from the
  migration-engineer audit, each as preserved / renamed / moved / or
  deprecated-with-warning, with rationale.
- Resolve the `axiom.yaml` vs `axiom.yml` filename question.
- Design the full `projects.yml` (registry v2) TypeScript shape.
- Design the full `axiom.yml`/`axiom.yaml` `schemaVersion: 2` TypeScript
  shape.
- Design the transition-window coexistence behavior: what a loader does
  on an old-shape document, and what MC-001 must do differently.
- Hand off an unambiguous field-level contract to registry-engineer.

## Non-goals

- No implementation of these types in `@axiom/user-workspace`,
  `@axiom/topology`, or `@axiom/config-validation` (registry-engineer's
  job, next).
- No CLI command changes (`cli-implementer`'s job).
- No `@axiom/doctor` check code changes (`validator-reviewer`'s job) —
  this increment specifies *what* MC-001 must do differently, not the
  patched `checks.ts`.
- No resolution of Q3 (`axiom.spec/` prefix survival) or Q4
  (capability/provider/MCP folding) — both remain open per the parent
  roadmap; flagged where they touch this design, not resolved here.
- No migration-script code (the actual `axiom upgrade` migration step) —
  only its exact required behavior, per the audit's own scoping of
  "migration script requirements" as design constraints, not code.

## Acceptance criteria

- [x] Each of the 9 "needs schema-writer decision" fields has an explicit
      resolution (preserved / renamed / moved / deprecated-with-warning)
      and a stated reason.
- [x] The `axiom.yaml`/`axiom.yml` filename question has an explicit
      decision with rationale grounded in the actual blast radius
      (`AXIOM_CONFIG_FILENAME` and its consumers).
- [x] A complete TypeScript shape for `projects.yml` (registry v2) exists,
      including the `repos` map, carried-over project fields, and a file
      version marker.
- [x] A complete TypeScript shape for `axiom.yml`/`axiom.yaml`
      `schemaVersion: 2` exists, including `projectId`, `repoId`, `role`,
      `indexes`, `paths`, and the resolved legacy fields.
- [x] The transition-window design states exactly what a loader does on
      an old-shape document (per artifact) and what MC-001 does
      differently under v1-absent vs v2 vs structurally invalid input.
- [x] No implementation code was written in `Axiom/packages/*` for this
      increment.
- [x] Next step (registry-engineer) is named with its exact required
      inputs.

## Open questions

None blocking this increment's own closure. Two items are explicitly
flagged as still open at the parent-roadmap level, not resolved here,
and registry-engineer must not silently resolve them either:

- **Q3** (`axiom.spec/` prefix survival) interacts with the new `paths`
  field design below (see "Resolution — `scopes` -> `indexes`/`paths`").
  This design keeps `paths` value-agnostic (plain relative-path strings)
  specifically so Q3's outcome does not require a schema change later,
  only a convention change in what values are written.
- **Q4** (capability/provider/MCP folding) is untouched; `rules`,
  `artifact_id_policy`, `lifecycle_commands`, `initial_capabilities`
  dispositions below do not presuppose Q4's outcome.

## Assumptions

- "Migration script" scope, `axiom upgrade` as the invocation mechanism,
  and the non-hard-removal posture (D1/D2/D3) are inherited unchanged
  from the migration-engineer audit's own Assumptions section.
- Registry-engineer will implement exactly the shapes below; any
  deviation found necessary during implementation should come back as a
  spec update to this file, not a silent divergence.
- `@axiom/core`'s `Result<T, E>` pattern (already used by
  `@axiom/user-workspace` and `@axiom/topology`) is reused for all new
  loader function signatures below, consistent with existing code style;
  this is not a new architectural choice, just continuity.

## Implementation notes

### Resolution of the 9 "needs schema-writer decision" fields

**Registry (D2) — `registry.json` -> `projects.yml`**

1. **`ProjectEntry.id`** (row: "Likely preserved as the project key or an
   explicit `id`/`projectId` field").
   **Resolution: preserved, and used as the map key.** In `projects.yml`,
   the top-level `projects` map is keyed by this same slug id (produced by
   the existing, reusable `slugifyProjectId` in
   `@axiom/user-workspace/src/registry-id.ts`). An explicit `id` field is
   *also* kept inside each entry (denormalized) so callers that already
   destructure a `ProjectEntry`-like object out of a list (e.g.
   `listProjects()`'s current return shape) do not need a call-site
   rewrite from "read `.id`" to "read the map key" — the v2 loader can
   still return `{ id, ...rest }[]` if it flattens the map for API
   compatibility (registry-engineer's exact return-shape decision, but
   the source-of-truth on disk has both the key and the field). No data
   loss, no rename.

2. **`ProjectEntry.name`** — audit already classified as "Likely
   preserved" / "Additive/compatible". **Resolution: preserved as-is**,
   field name and meaning unchanged.

3. **`ProjectEntry.functionalProfile`, `.overlay`, `.adapterTarget`**
   (row: "No equivalent in the two source documents' registry shape...
   likely stays out of `projects.yml`... must confirm this is not
   silently dropped").
   **Resolution: deprecated-with-warning at the registry level, not
   dropped.** These three fields are a *snapshot* of
   `install-profile.json` taken at `addProject` time (per the current
   code's own doc comment: "Se persiste como referencia inmutable"). They
   are orthogonal to the registry's actual job (mapping project id ->
   repo paths) and duplicate data that already lives, and is kept fresh,
   in each project's own `install-profile.json`. Reason to not carry them
   into `projects.yml`: the new registry shape is multi-repo
   (`repos: {...}` per project) and a single project-level
   `functionalProfile`/`overlay`/`adapterTarget` snapshot does not
   obviously belong to any one repo in that map — keeping it at the
   project level would be fine schema-wise, but it re-introduces exactly
   the kind of staleness the current code comment already flags as a
   known limitation ("un cambio posterior en el install-profile.json...
   NO se refleja aquí"). Decision: **remove from the persisted v2 shape**,
   and have the v1->v2 migration script emit an explicit,
   user-visible warning per project ("functionalProfile/overlay/
   adapterTarget for project X were not carried into projects.yml; read
   them from install-profile.json instead") rather than silently
   dropping them. This is a deprecation-with-warning at the *migration*
   level (the field disappears from the new format, with an explicit,
   logged reason), not a silent loss — consistent with D2's "not an
   additive/parallel format" framing while still honoring the audit's
   "do not drop silently" instruction. Any future consumer that needs a
   fast list-view of a project's profile can already read
   `install-profile.json` directly; no new indirection is introduced.

4. **`ProjectEntry.addedAt`, `.lastUsedAt`** (row: "No stated equivalent
   in source docs; likely preserved for UX parity").
   **Resolution: preserved as-is**, same field names, same ISO-8601
   string convention (`AD-04` in the current `registry-types.ts`). These
   support `axiom projects use` UX (recency sorting, "last used" display)
   and have no staleness problem like functionalProfile/overlay/
   adapterTarget do — they are registry-native facts about the registry
   entry itself, not a snapshot of another file. Carried over unchanged
   at the project level (not per-repo — "when was this project last used"
   is a project-level fact, not a per-repo one).

5. **`ProjectEntry.stale`** (flagged as "Behavior change, not just a
   shape change").
   **Resolution: moved, from project-level to per-repo-level, and
   remains a computed/lazy (non-persisted) field**, exactly as the audit
   recommended. `stale` is not part of the on-disk `projects.yml` shape
   at all (same as today — it's computed at read time by
   `refreshStaleness`, never serialized). The v2 loader computes
   `stale: boolean` per entry in the `repos` map (`fs.existsSync(repo.path)`),
   and a derived, also-computed `projectStale: boolean` (true iff *all*
   repos for that project are stale) can be exposed at the project level
   for callers that only care about the coarse case (e.g. `axiom
   projects list` default view). Both are computed, not persisted.

**Manifest (D3) — `axiom.yaml`: absent `schemaVersion` -> `schemaVersion: 2`**

6. **`project.name` -> `projectId`** — audit already classified this as
   "Hard break (rename) unless the loader/migration maps
   `project.name -> projectId` automatically".
   **Resolution: renamed, with automatic migration mapping.** The v2
   loader/migration script sets `projectId = slugifyProjectId(project.name)`
   (reusing the same slug function already used by the registry, for
   consistency between a project's registry key and its manifest
   `projectId` — this was not explicitly required by the source docs but
   removes an entire class of "registry id != manifest projectId" drift
   bugs at no cost). `project.name` itself is **not** dropped: v2 keeps a
   `name` field (human-readable, mutable) alongside `projectId` (slug,
   stable identity) — see full shape below. This mirrors the existing
   registry's own `id`/`name` split, another point of continuity rather
   than an invented distinction.

7. **`project.mode` (execution-topology: `local-only|gateway|hybrid`)**
   (row: "these are new, distinct concepts... schema-writer must confirm
   whether it is preserved as a separate field or folded elsewhere; do
   not silently drop; ties into Q4").
   **Resolution: preserved as-is, as a separate top-level field, not
   folded into `role`.** The migration-engineer audit already
   established (Full audit section) that `project.mode` (execution
   topology: local-only/gateway/hybrid, read by
   `@axiom/project-resolution/resolver.ts`) and `@axiom/topology`'s
   `mode` (repo topology: single-repo/multi-repo) are two independent
   axes. The new v2 `role` field (per repo) is a *third*, also
   independent axis: "what role does this repo play in the project"
   (e.g. `sdd`, `spec`, `code`). None of the three collapse into each
   other. Folding `project.mode` into `role` would conflate execution
   topology with repo role, which is exactly the kind of unrequested
   architectural conflation `Axiom.SDD/AGENTS.md` says to avoid. Decision:
   keep `project.mode?: string` verbatim (same loose typing as today —
   the current Zod schema deliberately accepts any string to stay
   decoupled from a runtime enum) at the project level in v2. This is
   explicitly flagged for Q4, not resolved by Q4's outcome: if Q4 later
   folds capability/provider concerns into role/tooling config, that is
   an *additional* change on top of this field's continued existence, not
   a replacement of it.

8. **`scopes: Record<string, {path, description?, product_runtime?,
   current_state?}>` -> `indexes`, `paths`** (row: "Hard break in field
   name and likely in shape granularity... schema-writer to confirm exact
   target shape").
   **Resolution: split into two fields per repo, `paths` (renamed,
   direct successor of `scopes`) and `indexes` (new, additive), not a
   single merged concept.** Rationale, grounded in what `scopes` actually
   is today: each `scopes` entry names a sub-path inside the repo
   (`path`), optionally flags it as the product runtime
   (`product_runtime`), and gives it a free-text description/state. Under
   v2, a repo is no longer necessarily one thing with internal
   sub-scopes — it has one `role` (the repo's job: sdd/spec/code) and,
   within it, a `paths` map that is the direct, renamed continuation of
   `scopes` (same shape: name -> relative path, `product_runtime` and
   `description` preserved verbatim, see full shape below). `indexes` is
   **new** and additive — it is not a `scopes` renaming; the parent
   roadmap describes indexes as a "derived vs curated" concept (source
   doc's registry model) that has no existing-code equivalent audited so
   far (this matches the migration-engineer audit's own finding: no
   dedicated index package exists, flagged for INC-07/INC-10, later
   increments). Decision: `indexes` is declared in the v2 manifest shape
   as an **optional, empty-by-default** field (`indexes?: Record<string,
   IndexRef>`) so the schema is forward-compatible with INC-07/INC-10
   without this increment inventing index *semantics* it has no mandate
   to design (that is explicitly out of scope per the parent roadmap's
   own phase sequencing — Phase C, after Phase A). `current_state` from
   the old `ScopeConfigSchema` is preserved verbatim inside the new
   `paths` entries (same free-text field, same optionality) since it
   carries information (a path's current state) that has no other home
   in this design.

9. **`rules?`, `artifact_id_policy?`, `lifecycle_commands?`,
   `initial_capabilities?`** (row: "No stated equivalent found... do not
   drop silently — these are actively validated fields today").
   **Resolution: preserved as-is, verbatim, unchanged shape, at the
   top level of the v2 manifest.** These four fields are validated by
   `AxiomYamlSchema` today but are not read by `@axiom/project-resolution`
   (per the migration-engineer audit's own Assumptions section, which
   explicitly names this gap). That means: (a) nothing in the new docs
   contradicts their existence — they were simply never in scope for the
   redesign's source documents, and (b) some other part of the system
   (not audited in this pass; `lifecycle_commands` overlaps in name with
   `@axiom/install-profiles`' `InstallProfilesYamlSchema.lifecycleCommands`,
   and `artifact_id_policy`/`initial_capabilities` may back other
   consumers not yet identified) may depend on them being present. Given
   the explicit "do not drop silently" instruction and no evidence they
   are dead fields, the correct, minimal-risk move is to **carry them
   forward unchanged** rather than invent a migration or a new home for
   them. This is deliberately conservative: it is easier to formally
   deprecate an actively-preserved field later (with evidence of non-use)
   than to un-drop a field that turned out to matter.

### Resolution: `axiom.yaml` vs `axiom.yml` filename

**Decision: keep `axiom.yaml`. Treat the docs' `axiom.yml` as a
documentation inconsistency to correct, not the code.**

Rationale, weighed against the actual blast radius:

- `@axiom/filesystem-truth/src/discovery.ts` exports
  `AXIOM_CONFIG_FILENAME = 'axiom.yaml'` as the **single source of
  truth** for the discovery filename — `discoverAxiomRoot` is the only
  place that probes the filesystem for this file, and every other
  package/CLI command that needs the project root goes through
  `discoverAxiomRoot`/`ProjectResolution`, not a hardcoded filename of
  its own (confirmed by the migration-engineer audit's blast-radius list:
  entries 9, 11, 12, 13, 14 all read the resolved root/content via
  `@axiom/project-resolution` or comment-only references, not a second
  hardcoded filename string).
- A rename to `axiom.yml` would require touching this one constant *and*
  every already-installed project's on-disk file (a forced rename
  migration for zero functional benefit — the two spellings are
  semantically identical, `.yaml` is the more explicit/conventional
  extension, and YAML tooling treats both identically).
- A dual-discovery fallback (probe both `axiom.yaml` and `axiom.yml`)
  would introduce two valid filenames for the same concept permanently,
  which is exactly the kind of ambiguity `Axiom.SDD/AGENTS.md`'s
  "no speculative architecture" and "no complex metadata systems" limits
  argue against, for a problem that has a much simpler fix: the
  decision documents and the parent roadmap simply used the shorter,
  common shorthand `axiom.yml` when writing prose, while the actual
  implementation has always used `.yaml`. Nothing in the two source
  decision documents' *behavior* requirements depends on the literal
  three-vs-four-letter extension.
- Net effect: **zero code change** for this specific question.
  `AXIOM_CONFIG_FILENAME` stays `'axiom.yaml'`. Everywhere this design
  document and the registry-engineer/cli-implementer/validator-reviewer
  steps say "the manifest file", they mean the file at
  `AXIOM_CONFIG_FILENAME`, i.e. `axiom.yaml`. Future prose in
  `Axiom.Spec` and the parent roadmap should say `axiom.yaml` going
  forward; this increment does not go back and edit the parent roadmap's
  or migration-engineer audit's already-written prose (their historical
  record stays as written), but every *new* document from this point on
  in the INC-01 chain uses `axiom.yaml`.

### New type design 1 — `~/.axiom/projects.yml` (registry v2)

File path: `path.join(homeDir, 'projects.yml')` (replaces
`path.join(homeDir, 'registry.json')` in `@axiom/user-workspace/src/
paths.ts`'s `getPaths()`; `registryPath` field name in
`UserWorkspacePaths` can stay, now pointing at the new file — an internal
rename of what the field *points to*, not its own key, is a
registry-engineer implementation choice, not a schema concern here).

```ts
// packages/user-workspace/src/registry-types.ts (v2 replacement)

/** Schema version of the projects.yml document itself (distinct from
 *  axiom.yml's own schemaVersion — these are two independently
 *  versioned artifacts). */
export type ProjectsFileSchemaVersion = 2;

/** Canonical role key for a repo inside a project's `repos` map.
 *  Mirrors @axiom/topology's existing repo id vocabulary
 *  ('sdd-repo' | 'spec-repo' style) for naming consistency, expressed
 *  without the '-repo' suffix since it is already implied by the
 *  `repos` map key position. Open-ended string, not a closed union,
 *  because `roleCodeRepositories` in @axiom/topology already allows
 *  arbitrary role-code repo ids beyond sdd/spec — the registry must not
 *  be stricter than the topology model it mirrors. */
export type RepoRoleKey = string;

/** One repo entry inside a project's `repos` map. */
export interface RegistryRepoEntry {
  /** Canonical role this repo plays for the project
   *  (e.g. 'sdd', 'spec', 'code', or a role-code repo id).
   *  Denormalized copy of the map key, kept for callers that flatten
   *  the map to a list (mirrors ProjectEntry.id's denormalization
   *  rationale below). */
  readonly role: RepoRoleKey;
  /** Absolute filesystem path to this repo on the current machine. */
  readonly path: string;
}

/** A single project entry in projects.yml. Keyed by `id` (slug) in the
 *  parent `projects` map; `id` is also denormalized inside the entry
 *  (see Resolution #1 above). */
export interface ProjectEntryV2 {
  /** Identifier, same slug produced by slugifyProjectId (unchanged
   *  from v1; reused as the projects map key). */
  readonly id: string;
  /** Human-readable project name (from axiom.yaml `project.name`,
   *  or `projectId`'s un-slugified source — see manifest design). */
  readonly name: string;
  /** Map of role key -> repo entry. Replaces v1's single `rootPath`.
   *  Must contain at least one entry (enforced by registry-engineer's
   *  write path, not by this type alone). */
  readonly repos: Readonly<Record<RepoRoleKey, RegistryRepoEntry>>;
  /** ISO-8601 timestamp of registration (ProjectEntry.addedAt,
   *  preserved verbatim per Resolution #4). */
  readonly addedAt: string;
  /** ISO-8601 timestamp of last `axiom projects use` (preserved
   *  verbatim per Resolution #4). */
  readonly lastUsedAt: string;
  // NOTE: functionalProfile / overlay / adapterTarget intentionally
  // absent per Resolution #3 — read install-profile.json instead.
  // NOTE: `stale` intentionally absent — computed at read time per repo
  // (RegistryRepoEntryWithStatus below), never persisted.
}

/** Document persisted at ~/.axiom/projects.yml. */
export interface ProjectsFile {
  readonly schemaVersion: ProjectsFileSchemaVersion;
  readonly projects: Readonly<Record<string /* id */, ProjectEntryV2>>;
}

// ─── Read-time computed shapes (never persisted) ──────────────────────

/** RegistryRepoEntry plus the lazily-computed staleness flag. */
export interface RegistryRepoEntryWithStatus extends RegistryRepoEntry {
  readonly stale: boolean; // fs.existsSync(path) === false
}

/** ProjectEntryV2 plus computed per-repo status and a project-level
 *  rollup, returned by the v2 loadRegistry/listProjects functions. */
export interface ProjectEntryV2WithStatus extends Omit<ProjectEntryV2, 'repos'> {
  readonly repos: Readonly<Record<RepoRoleKey, RegistryRepoEntryWithStatus>>;
  /** true iff every repo in `repos` is stale. Convenience rollup for
   *  coarse list views (e.g. default `axiom projects list`). */
  readonly projectStale: boolean;
}
```

Notes on this design:

- `schemaVersion: 2` for `projects.yml` itself is a **new, independent**
  version marker from `axiom.yml`'s `schemaVersion: 2` — they happen to
  share the numeral by coincidence of both being "the second shape,"
  not because they are coupled. This is the "exact field name... schema-
  writer must confirm/introduce" item the audit flagged; `schemaVersion`
  (same key name as v1's registry, same key name as `axiom.yml`) is
  chosen for consistency with every other versioned document in this
  codebase (`Registry.schemaVersion`, `TopologyManifest.schemaVersion`,
  `LocalBindings.schemaVersion`, `UserInstallManifest.schemaVersion`) —
  introducing a differently-named version key here (e.g.
  `projectsFileVersion`) would break an otherwise-universal convention
  for no benefit.
- `projects` changed from `Registry.projects: ProjectEntry[]` (array) to
  `ProjectsFile.projects: Record<id, ProjectEntryV2>` (map) — this
  directly follows D2's own "`repos: { roleKey: ... }` map" pattern
  applied one level up, and removes the need for O(n) linear scans by id
  that `getProject`/`useProject`/`removeProject` do today.
- Atomic tmp+rename write pattern is preserved unchanged (already proven
  for YAML by `@axiom/topology/src/loader.ts`'s `saveLocalBindings`, per
  the audit's own note).

### New type design 2 — `axiom.yaml` `schemaVersion: 2`

```ts
// packages/config-validation/src/schemas.ts (v2 addition, alongside v1)

import { z } from 'zod';

// ─── v2 schema ────────────────────────────────────────────────────────

/** Direct rename-successor of v1's ScopeConfigSchema; same shape,
 *  now scoped per-repo inside `paths` rather than per-project. */
const AxiomPathEntrySchemaV2 = z.object({
  path: z.string({ required_error: 'El campo path es obligatorio en cada entrada de paths.' }),
  description: z.string().optional(),
  product_runtime: z.boolean().optional(),
  current_state: z.string().optional(),
});

/** New, additive. Empty/absent by default. Semantics (derived vs
 *  curated) are NOT designed in this increment — see Non-goals; this
 *  is a forward-compatible placeholder shape only: a named reference
 *  to *something* index-like, kept intentionally minimal. */
const AxiomIndexRefSchemaV2 = z.object({
  path: z.string(),
  description: z.string().optional(),
});

const ArtifactIdPolicySchemaV2 = z.object({
  timezone: z.string(),
  hash_length: z.number().int().positive(),
  formats: z.record(z.string()),
}); // unchanged from v1, preserved per Resolution #9

export const AxiomYamlSchemaV2 = z.object({
  schemaVersion: z.literal(2),

  /** Stable slug identity of the project (see Resolution #6). Reuses
   *  the same slugifyProjectId function as the registry. */
  projectId: z.string({ required_error: 'projectId es obligatorio en schemaVersion 2.' }),
  /** Human-readable project name (successor of v1 project.name; kept
   *  distinct from projectId per Resolution #6). */
  name: z.string({ required_error: 'name es obligatorio en schemaVersion 2.' }),
  /** Identity of THIS repo within the project's topology
   *  (matches @axiom/topology's RepoRef.id vocabulary). */
  repoId: z.string({ required_error: 'repoId es obligatorio en schemaVersion 2.' }),
  /** This repo's role in the project (e.g. 'sdd', 'spec', 'code').
   *  Matches @axiom/topology's RoleAssignment.roleId vocabulary. */
  role: z.string({ required_error: 'role es obligatorio en schemaVersion 2.' }),

  /** Execution-topology mode, preserved verbatim per Resolution #7.
   *  Independent axis from `role` (repo topology) and from
   *  @axiom/topology's TopologyManifest.mode (single-repo|multi-repo). */
  mode: z.string().optional(),

  /** Renamed successor of v1 `scopes` (Resolution #8). Same entry
   *  shape, same semantics, scoped to sub-paths of THIS repo. */
  paths: z.record(AxiomPathEntrySchemaV2).optional(),
  /** New, additive (Resolution #8). Semantics deferred to a later
   *  increment (INC-07/INC-10); this schema only reserves the field. */
  indexes: z.record(AxiomIndexRefSchemaV2).optional(),

  /** Preserved verbatim from v1 (Resolution #9). */
  rules: z.array(z.string()).optional(),
  artifact_id_policy: ArtifactIdPolicySchemaV2.optional(),
  lifecycle_commands: z.array(z.string()).optional(),
  initial_capabilities: z.record(z.unknown()).optional(),

  /** Preserved verbatim from v1 (not in the "needs decision" list, but
   *  present in AxiomYamlSchema today and not addressed by D3 — kept
   *  for parity, no reason found to drop). */
  status: z.string().optional(),
  product_implementation_status: z.string().optional(),
});

export type AxiomYamlConfigV2 = z.infer<typeof AxiomYamlSchemaV2>;
```

Field-origin cross-reference (every v1 field accounted for):

| v1 field | v2 field | Resolution # |
|---|---|---|
| *(absent)* | `schemaVersion: 2` | new, required in v2 |
| `project.name` | `projectId` (slug) + `name` (human-readable) | #6 |
| `project.status` | `status` (top-level, unchanged) | preserved, unaddressed by D3, carried for parity |
| `project.product_implementation_status` | `product_implementation_status` (top-level, unchanged) | preserved, unaddressed by D3, carried for parity |
| `project.mode` | `mode` (top-level, unchanged) | #7 |
| *(absent)* | `repoId` | new, required in v2 |
| *(absent)* | `role` | new, required in v2 |
| `scopes` | `paths` (renamed, same shape) | #8 |
| *(absent)* | `indexes` (new, optional, semantics deferred) | #8 |
| `rules` | `rules` (unchanged) | #9 |
| `artifact_id_policy` | `artifact_id_policy` (unchanged) | #9 |
| `lifecycle_commands` | `lifecycle_commands` (unchanged) | #9 |
| `initial_capabilities` | `initial_capabilities` (unchanged) | #9 |

### Transition-window coexistence design

Per D1/D2/D3's shared non-hard-removal posture, both artifacts must
support reading their predecessor shape during a transition window. The
two artifacts differ in *how* that is possible, and the design below is
deliberately different for each, for reasons stated inline.

**`projects.yml` vs `registry.json` — hard file-format switch, no
in-place coexistence.**

This mirrors D2's own classification ("Hard break for the *reader*... a
v2 binary must not silently treat it as `projects.yml`"). There is no
"single loader reads either shape" design here, because the two are
different files at different paths (`registry.json` vs `projects.yml`),
not the same file with a version field:

- `loadRegistry(homeDir)` (v2) reads `projects.yml` only. If
  `projects.yml` is absent **and** `registry.json` is present, it returns
  a distinguishable, typed result — not a silent empty registry — e.g.
  `err({ kind: 'legacy-registry-not-migrated', path: registryJsonPath,
  message: 'Found registry.json (v1) but no projects.yml (v2). Run
  `axiom upgrade` to migrate.' })`. This gives the CLI layer a clear
  branch to either auto-invoke the migration (if `cli-implementer`
  decides that's the right UX) or instruct the user, rather than starting
  the user over with an empty registry.
- If **neither** file is present, behavior is unchanged from today:
  empty registry, `ok({ schemaVersion: 2, projects: {} })` (normal
  first-install case).
- If `projects.yml` is present but has `schemaVersion !== 2`, this is a
  **future**-version case (hard rejection, same posture as v1's own
  `unsupported-schema-version` gate) — not a legacy case, since 2 is the
  first version of this filename/shape.
- `registry.json` is never deleted automatically by the loader (only the
  migration script may move it aside, per the audit's migration-script
  requirement #5); the loader's job is only to detect and report its
  presence, not to consume it.

**`axiom.yaml` `schemaVersion` absent (v1) vs `schemaVersion: 2` —
same file, same filename, coexisting via a version-dispatch parse.**

This is the opposite shape of the registry case: same file path
(`axiom.yaml`, decision above), same discovery mechanism
(`discoverAxiomRoot`/`AXIOM_CONFIG_FILENAME`), two possible document
shapes distinguished only by the presence/value of `schemaVersion`. This
is exactly the "same document, absent -> 2" situation the audit
identified as the real shape of D3 (not "1 -> 2").

`validateAxiomYamlContent` (in `@axiom/config-validation/src/
validator.ts`) becomes version-dispatching:

```ts
// packages/config-validation/src/validator.ts (v2 dispatch design)

export type AxiomYamlValidationResult =
  | { valid: true; version: 1; data: AxiomYamlConfig }   // legacy, schemaVersion absent
  | { valid: true; version: 2; data: AxiomYamlConfigV2 } // schemaVersion: 2
  | { valid: false; version: 'unknown'; errors: ValidationError[] };

export function validateAxiomYamlContent(rawYaml: string): AxiomYamlValidationResult {
  const parsed = /* yaml.load, same as today */;
  const obj = parsed as { schemaVersion?: unknown } | null;

  if (obj && typeof obj === 'object' && obj.schemaVersion === 2) {
    const result = AxiomYamlSchemaV2.safeParse(parsed);
    return result.success
      ? { valid: true, version: 2, data: result.data }
      : { valid: false, version: 'unknown', errors: /* map issues, same as today's validateWithSchema */ };
  }

  // schemaVersion absent, or any other non-2 value: attempt legacy (v1) parse.
  // This treats "schemaVersion absent" and "schemaVersion: 1" identically,
  // per the audit's finding that v1 never had the field at all.
  const legacyResult = AxiomYamlSchema.safeParse(parsed);
  return legacyResult.success
    ? { valid: true, version: 1, data: legacyResult.data }
    : { valid: false, version: 'unknown', errors: /* mapped issues */ };
}
```

Key property: a document is only `valid: false` when it fails **both**
schemas — i.e. it is structurally invalid under either version, not
merely "old". This is the direct implementation of D3's stated posture
("`MC-001` should `warn` (not `fail`) on a v1 document once v2 exists,
and only `fail` on a structurally invalid document under either
version").

**What MC-001 needs to do differently:**

Today (`packages/doctor/src/checks.ts`, `runManifestChecks`), MC-001 is a
binary pass/fail on `validationResult.valid`. Under v2 coexistence, its
required behavior becomes a three-way branch on the new
`AxiomYamlValidationResult.version`/`valid`:

| `validateAxiomYamlContent` result | MC-001 status | Evidence text (design intent, not final copy) |
|---|---|---|
| `valid: true, version: 2` | `pass` | "axiom.yaml is valid under schemaVersion 2." |
| `valid: true, version: 1` | `warn` (not `pass`, not `fail`) | "axiom.yaml has no schemaVersion (legacy v1 format); valid under the legacy schema. Run `axiom upgrade` to migrate to schemaVersion 2." |
| `valid: false, version: 'unknown'` | `fail` (unchanged from today) | Same error-joining format as today (`errors.map(...).join('; ')`), since this means invalid under both schemas. |

This is a **new required warn branch**, not present in today's binary
pass/fail `DoctorCheck` construction — MC-001's existing early-return on
unreadable file content (`fail` if `readFileContent` returns falsy) is
untouched, only the branch after a successful read/parse changes from
one `pass`/`fail` decision to the three-way table above. This is
precisely the "concrete extension point... not something to build now"
the migration-engineer audit flagged for TC-001's analogous
`defaultSingleRepoManifest` warning — the same shape of change, applied
to MC-001. Implementing the actual `checks.ts` edit is
**validator-reviewer's** job (next-next), not this increment's; this
table is its exact required contract.

## Validation

`No validation command was found. Performed best-effort validation by
inspecting changed files and checking consistency against the requested
behavior.`

This increment is design/spec-only (no implementation code changed in
`Axiom/packages/*` or anywhere else). Best-effort validation performed:

- Every resolution above is checked against a direct re-read of the
  actual current source this session (`user-workspace/src/{paths.ts,
  registry.ts,registry-types.ts}`, `config-validation/src/{schemas.ts,
  validator.ts}`, `filesystem-truth/src/discovery.ts`,
  `topology/src/types.ts`, `doctor/src/checks.ts` lines ~196-234 covering
  MC-001/MC-002), not inferred from the migration-engineer audit's prose
  alone — confirming the audit's field descriptions match the live code
  before designing against them.
- Every new type/schema was cross-checked for field-name collisions or
  contradictions against sibling packages' existing conventions
  (`schemaVersion` key naming across `Registry`, `TopologyManifest`,
  `LocalBindings`, `UserInstallManifest`; `RepoRef.id`/`RoleAssignment
  .roleId` vocabulary in `@axiom/topology`) so registry-engineer inherits
  a internally-consistent contract, not just an internally-consistent
  one.
- The full v1 field-origin cross-reference table (manifest) was built by
  enumerating every key in the live `AxiomYamlSchema` (`schemas.ts`) and
  confirming each has an explicit v2 destination — no field was
  silently omitted from the mapping.
- `Axiom/package.json`'s real validation commands (`npm run build`, `npm
  test`, `npm run doctor`, `npm run typecheck`) were not run, because no
  code was changed in `Axiom/` for this increment; running them would
  only assert pre-existing state.

## Result

Resolved all 9 "needs schema-writer decision" fields from the
migration-engineer audit (5 registry, 4 manifest), each classified as
preserved / renamed / moved / deprecated-with-warning with explicit
rationale grounded in a re-read of the live source. Resolved the
`axiom.yaml` vs `axiom.yml` filename discrepancy: **keep `axiom.yaml`**,
correct the docs, no code or file rename required — justified against
`AXIOM_CONFIG_FILENAME`'s actual single-source-of-truth status and its
consumer list from the migration-engineer's blast-radius audit. Produced
full TypeScript/Zod designs for both `~/.axiom/projects.yml`
(`ProjectsFile`/`ProjectEntryV2`/`RegistryRepoEntry`, keyed map with a
denormalized `id`/`role`, `functionalProfile`/`overlay`/`adapterTarget`
deprecated-with-warning at migration time, `addedAt`/`lastUsedAt`
preserved, `stale` moved to per-repo/computed) and `axiom.yaml`
`schemaVersion: 2` (`AxiomYamlSchemaV2`: `projectId`/`name` split from
`project.name`, new required `repoId`/`role`, `mode` preserved as an
independent axis, `scopes` renamed to `paths` with `indexes` added
additively and semantics deferred, `rules`/`artifact_id_policy`/
`lifecycle_commands`/`initial_capabilities` preserved verbatim).
Designed transition-window coexistence for both artifacts: a hard,
distinguishable-error file-format switch for the registry
(`registry.json` presence without `projects.yml` yields a typed
"not migrated" error, not a silent empty registry), and a
version-dispatching parse for the manifest (`schemaVersion` absent is
treated identically to `schemaVersion: 1`, valid under either schema
routes to `pass`(v2)/`warn`(legacy v1)/`fail`(invalid under both) —
giving MC-001 its exact required three-way branch, up from today's
binary pass/fail).

No implementation code was written in `Axiom/packages/*`, per this
increment's non-goals. Status is `pending`, not `closed` (see Closure
rationale below).

## General spec integration

No integration into `Axiom.Spec/general-spec.md` was performed — that
file does not exist in this repo, consistent with both prior increments
in this chain. Per the same reasoning the migration-engineer audit gave:
`Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md`'s "sin cerrar el naming
final" placeholder should still not be updated yet. This increment
finalizes the field-level *design* (types/schemas), but registry-engineer
has not yet implemented or validated it against real code (tests,
`axiom doctor`, `axiom upgrade` dry-run) — updating the canonical spec
doc now would lock in a design that has not yet survived contact with
implementation. The correct point to integrate stable knowledge into
`general-spec.md`-equivalent documents is after validator-reviewer closes
out the full INC-01 chain (migration-engineer -> schema-writer ->
registry-engineer -> cli-implementer -> validator-reviewer), when the
shape is both designed and proven, not merely designed.

## Next step recommendation

Launch **registry-engineer** next, per INC-01's subagent sequence. Exact
inputs it needs from this document:

1. **The two full type/schema designs** above (`ProjectsFile`/
   `ProjectEntryV2`/`RegistryRepoEntry` for `projects.yml`;
   `AxiomYamlSchemaV2` for `axiom.yaml`) as the literal contract to
   implement in `@axiom/user-workspace` (`registry-types.ts`,
   `registry.ts`, `paths.ts`) and `@axiom/config-validation`
   (`schemas.ts`, `validator.ts`) respectively — not to redesign.
2. **The 9 field resolutions** with their stated rationale, so
   registry-engineer implements the deprecation warnings (functionalProfile/
   overlay/adapterTarget at migration time) and straight carries-over
   (addedAt/lastUsedAt, rules/artifact_id_policy/lifecycle_commands/
   initial_capabilities, mode) exactly as specified, without
   re-deciding them.
3. **The filename decision** (keep `axiom.yaml`, zero rename) — so
   registry-engineer does not touch `AXIOM_CONFIG_FILENAME` and instead
   focuses only on the schema-version-dispatch logic inside the same
   file.
4. **The transition-window design** (registry: hard-break with typed
   "not migrated" error; manifest: version-dispatching parse returning
   `{valid, version, data}` / `{valid: false, version: 'unknown',
   errors}`) as the exact loader/validator contract, including the
   `AxiomYamlValidationResult` discriminated union shape.
5. **The MC-001 three-way branch table** (pass v2 / warn legacy-v1 /
   fail invalid-under-both) — registry-engineer does not implement
   `checks.ts` itself (that is validator-reviewer's job) but must keep
   `validateAxiomYamlContent`'s new return shape stable enough for
   validator-reviewer to consume it exactly as designed.
6. **The full blast-radius list** (31 non-test source files) from the
   migration-engineer audit, unchanged, as the set of files
   registry-engineer must revisit once the new shapes exist, plus this
   document's field-origin cross-reference table as the checklist for
   "did every old field get an explicit new-shape destination."

Registry-engineer should flag, not silently resolve, any point where the
live code turns out to differ from what this design assumed (all designs
above were checked against a fresh read this session, but registry-
engineer's implementation pass is the point where any residual mismatch
would surface concretely, e.g. inside test fixtures not read in this
pass).
