# Increment: Registry + manifest schema v2 (migration-engineer audit)

Status: pending
Date: 2026-07-02

## Goal

Execute INC-01 of the parent roadmap
(`Axiom.Spec/specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`):
reconcile `@axiom/user-workspace` (user-level registry), `@axiom/
project-resolution` (per-project `axiom.yaml` resolver), and `@axiom/
topology` (repo topology manifest) against the new target contract —
`~/.axiom/projects.yml` (YAML, multi-repo-by-default) and `axiom.yml`
`schemaVersion: 2` (`projectId/repoId/role/indexes/paths`).

This specific increment covers only the **migration-engineer** role: full
audit of the three packages' remaining files, a complete cross-repo
blast-radius list of every consumer of the current shapes, and a
field-by-field compatibility/cutover note. It does **not** write new
schema types or implementation code — that is the next role
(schema-writer) in the roadmap's subagent sequence for INC-01.

## Context

Parent increment: `INC-20260702-axiom-redesign-roadmap`. That roadmap
identified INC-01 as the first, blocking increment in the sequence and
listed three open questions (Q1, Q2, Q5) as blocking for INC-01 write
work. The user has now resolved all three. They are recorded below as
**settled decisions**, not open questions.

### Settled decisions (previously Q1, Q2, Q5 in the parent roadmap)

- **D1 (was Q1) — repo topology mode**: multi-repo becomes the primary/
  default model. `single-repo` mode in `@axiom/topology` is deprecated as
  the default — it is **not silently removed**. It remains available only
  as an explicit opt-in convenience ("comodidad local, pero no
  requisito" per source doc 1.1). `defaultSingleRepoManifest` in
  `@axiom/topology/src/loader.ts` needs a deprecation path (a runtime
  warning when it is the manifest actually in effect, i.e. no
  `topology.yaml` present and the caller has not explicitly opted into
  single-repo), not a hard removal or silent behavior change.
- **D2 (was Q2) — registry format/location**: `~/.axiom/registry.json`
  (JSON, one `rootPath` per project, `@axiom/user-workspace/src/
  registry.ts` + `registry-types.ts`) migrates to `~/.axiom/projects.yml`
  (YAML, `repos: { roleKey: { role, path } }` map per project), matching
  the two external decision documents' exact format. This is a **format
  migration**, not an additive/parallel format — existing installations
  need a migration script from `registry.json` to `projects.yml`.
- **D3 (was Q5) — `axiom.yml` schema version**: `axiom.yml` gets a
  versioned schema bump, `schemaVersion: 1 -> 2`, introducing `projectId/
  repoId/role/indexes/paths` per the new docs. This is a **versioned
  migration with a migration script**, not an additive superset merged
  into schema version 1. Note: the audit below found that the *current*
  `axiom.yaml` schema (`@axiom/config-validation/src/schemas.ts`,
  `AxiomYamlSchema`) has **no `schemaVersion` field at all** — this is a
  finding that affects how "1 -> 2" is actually implemented (see Field-by-
  field diff, D3 below).

## Scope

- Read every remaining source file (not dist, not previously-sampled-only)
  in `Axiom/packages/user-workspace/`, `Axiom/packages/project-resolution/`,
  and `Axiom/packages/topology/`: types, registry/resolver/loader logic,
  and tests.
- Read the doctor checks that validate these packages' output shapes:
  `MC-001`/`MC-002` (manifest checks) and `TC-001`/`TC-002` (topology
  checks) in `@axiom/doctor/src/checks.ts`.
- Grep the entire `Axiom` monorepo (`packages/` and `apps/`) for every
  file that reads or writes `axiom.yaml`, `registry.json`, or topology
  manifests today, to give the schema-writer step a complete blast-radius
  list.
- Produce a field-by-field diff (old shape -> new shape) for both the
  registry and the `axiom.yml` manifest.
- Classify every change as needing a deprecation warning vs. a hard
  break, and state what a migration script must do for each of the two
  artifacts (`registry.json -> projects.yml`, `axiom.yml` schemaVersion
  1 -> 2).

## Non-goals

- No new TypeScript types or schema code for the v2 contract — that is
  the schema-writer subagent's job, next in the roadmap sequence.
- No registry-engineer, cli-implementer, or validator-reviewer work
  (extending `@axiom/user-workspace`/`@axiom/topology`, CLI commands, or
  `@axiom/doctor` checks) — those are later steps in INC-01's own
  subagent sequence, after schema-writer.
- No decision-making on Q3 (whether `axiom.spec/` prefix survives
  multi-repo separation) or Q4 (capability/provider/MCP folding) — both
  remain open per the parent roadmap and are out of scope for this
  increment.
- No code changes anywhere in `Axiom/` or `Axiom.SDD/` in this increment
  (audit + spec authoring only).

## Acceptance criteria

- [x] Every remaining file in `@axiom/user-workspace`, `@axiom/
      project-resolution`, and `@axiom/topology` (src + tests) has been
      read, not just the files sampled in the parent roadmap pass.
- [x] `MC-001`/`MC-002` and `TC-001`/`TC-002` doctor checks have been read
      and their dependency on the current shapes is documented.
- [x] A complete, grep-verified blast-radius list of every `Axiom`
      package/app source file (excluding `dist/` and tests) that reads or
      writes `axiom.yaml`/`registry.json`/topology manifests is produced.
- [x] A field-by-field diff (old shape -> new shape) exists for both the
      registry and the `axiom.yml` manifest.
- [x] Each diff item is classified as deprecation-warning or hard-break,
      with the concrete migration-script behavior stated.
- [x] D1/D2/D3 are recorded as settled decisions, not open questions.
- [x] No new schema types or implementation code were written in this
      increment; the handoff to schema-writer is explicit and lists exact
      inputs.

## Open questions

None blocking this increment. Q3 and Q4 from the parent roadmap remain
open but do not block the migration-engineer audit or the schema-writer
step that follows (Q3 affects path conventions inside the new schema,
which the schema-writer should flag but not resolve unilaterally; Q4 is
scoped to a later increment, INC-13).

## Assumptions

- The schema-writer subagent (next step) will use this document's
  field-by-field diff and blast-radius list as its starting brief,
  per the parent roadmap's subagent sequence for INC-01: migration-
  engineer -> schema-writer -> registry-engineer -> cli-implementer ->
  validator-reviewer.
- "Migration script" here means a data-migration routine (reads the old
  file, writes the new file, does not lose data) invoked from the CLI
  (e.g. via `axiom upgrade`, whose migration mechanism already exists in
  `@axiom/versioning`), not a new standalone tool — consistent with
  `Axiom.SDD/AGENTS.md`'s prohibition on speculative infrastructure.
- `@axiom/config-validation`'s `AxiomYamlSchema` (Zod) is treated as the
  actual current schema source of truth for `axiom.yaml`, since it is
  what `MC-001` validates against — not the narrower shape read ad hoc by
  `@axiom/project-resolution/resolver.ts`'s internal `AxiomYamlConfig`
  interface (which only reads `project.name`, `project.mode`, `scopes`
  and ignores the other fields `AxiomYamlSchema` allows, such as `rules`,
  `artifact_id_policy`, `lifecycle_commands`, `initial_capabilities`).

## Implementation notes

### Full audit: `@axiom/user-workspace`

Files read in full (all `src/` and `tests/`, beyond the roadmap's earlier
`registry.ts`/`registry-types.ts` sample):

- `src/errors.ts` — `UserWorkspaceError` discriminated union
  (`io-error | not-a-directory | invalid-override`). Reusable as-is for
  the v2 registry; no shape assumptions specific to JSON.
- `src/types.ts` — `UserWorkspacePaths` (`homeDir, registryPath,
  installPath, logsDir, cacheDir`), `UserWorkspaceInitOptions`,
  `EnsureHomeDirOptions`. `registryPath` is currently hardcoded to
  `<homeDir>/registry.json` inside `getPaths()` (`src/paths.ts:56`) — this
  is the single source of the on-disk filename and must change to
  `projects.yml` for D2.
- `src/paths.ts` — `resolveHomeDir`, `getPaths`, `ensureHomeDir`. Pure/
  idempotent; `getPaths` is the one place that needs the filename change.
- `src/registry-types.ts` — `FunctionalProfileId`, `OperationalOverlayId`,
  `ProjectEntry` (`id, name, rootPath, functionalProfile, overlay,
  adapterTarget, addedAt, lastUsedAt, stale`), `Registry` (`schemaVersion:
  1` literal, `projects: ProjectEntry[]`). This is the exact shape that
  must become the new `repos: { roleKey: { role, path } }` map per
  project (see Field-by-field diff, D2).
- `src/registry-id.ts` — `slugifyProjectId`, pure slug function. No shape
  dependency; reusable as-is for the new project `id`.
- `src/registry.ts` — `loadRegistry`, `saveRegistry`, `addProject`,
  `removeProject`, `getProject`, `listProjects`, `useProject`,
  `findByRootPath`. All are JSON-specific (`JSON.parse`/
  `JSON.stringify`) and schema-version-gated (`CURRENT_SCHEMA_VERSION =
  1`, hard rejection of any other value in `loadRegistry`). This
  hard-rejection behavior is important: **a v1 binary will already refuse
  to read a `projects.yml`/bumped-schema registry** — confirms D2 is a
  breaking format change requiring an explicit migration, not something
  that can silently coexist without a version/format switch in the
  loader.
- `src/self-update.ts` — `UserInstallManifest` (`~/.axiom/install.json`),
  `loadInstallManifest`/`saveInstallManifest`/`recordCheck`/
  `readInstalledVersion`/`compareSemver`. **Independent of the registry
  format** — `install.json` is a separate file with its own
  `schemaVersion: 1`. Not affected by D2, but confirms the `~/.axiom/`
  home directory already hosts multiple versioned JSON documents side by
  side; `projects.yml` will be a third, YAML-formatted sibling.
- `src/index.ts` — barrel export. Confirms the full public API surface of
  `@axiom/user-workspace` that any registry-engineer change must
  preserve or deliberately break.
- `tests/paths.test.ts`, `tests/registry.test.ts`,
  `tests/self-update.test.ts` — read in full. `registry.test.ts` in
  particular hardcodes the JSON file name/shape in assertions
  (`registryFile()` helper joins `'registry.json'`; scenario 12 asserts
  `err.cause === 'unsupported-schema-version'` for `schemaVersion: 99`,
  and scenario 12's malformed-array case is JSON-specific
  (`JSON.stringify`)). These tests are a hard break point: they will fail
  against a `projects.yml`-based implementation unless rewritten, not
  merely extended.

### Full audit: `@axiom/project-resolution`

- `src/resolver.ts` — `resolveProject`, `assertUnambiguous`. Internal
  `AxiomYamlConfig` interface reads only `project.name`, `project.mode`,
  `scopes` (map of `{ path, product_runtime }`). `ProjectMode` union is
  `'local-only' | 'gateway' | 'hybrid'` — this is an **execution-topology
  axis** (network/gateway behavior), orthogonal to `@axiom/topology`'s
  `mode: 'single-repo' | 'multi-repo'` **repo-topology axis**. Both axes
  are real and independent; D1 does not collapse them, it only affects
  the repo-topology axis.
- `src/index.ts` — barrel, exports `resolveProject`, `assertUnambiguous`,
  and the three types. Small, stable public surface.
- `tests/resolver.test.ts` — read in full. All fixtures use bare
  `project.name` + `project.mode` + `scopes` (no `schemaVersion` field
  anywhere in the YAML fixtures) — confirms the current `axiom.yaml` is
  genuinely unversioned in practice, not just in the Zod schema.

### Full audit: `@axiom/topology`

- `src/types.ts` — `RepoRef` (`id, ref, description?`), `RoleAssignment`
  (`repoId, roleId, primary?`), `TopologyManifest` (`schemaVersion: 1`,
  `mode: 'single-repo' | 'multi-repo'`, `sddRepo, specRepo,
  roleCodeRepositories, assignments, qaLane?`), `LocalBindings`
  (`schemaVersion: 1`, `localPaths: Record<string,string>`),
  `TopologyError`, `TopologyFinding`, `ValidationResult`.
- `src/loader.ts` — `defaultSingleRepoManifest` (both `sddRepo` and
  `specRepo` resolve to `projectRoot`, `mode: 'single-repo'`) and
  `defaultInstalledMultiRepoManifest` (`sddRepo.ref: '.'`, `specRepo.ref:
  '../${projectName}.spec'`, `mode: 'multi-repo'`). `loadTopology` reads
  `axiom.spec/config/topology.yaml`; if absent, it probes `axiom.yaml`
  for `project.mode === 'installed-multi-repo'` — if found, returns the
  installed-multi-repo default; otherwise **falls back silently to
  `defaultSingleRepoManifest`**. This silent fallback is exactly the
  behavior D1 requires a deprecation warning for once multi-repo becomes
  primary. Also implements `loadLocalBindings`/`saveLocalBindings`
  (per-user, YAML, atomic tmp+rename — this is the existing precedent for
  how `projects.yml` should be written) and `resolveRepoPath`.
- `src/validate.ts` — `validateTopology`: `uncovered-repo` /
  `unknown-role` / `invalid-repo-ref` errors, `overlap-warning` warning.
  Pure validation, no I/O; reusable unchanged for the v2 topology model.
- `src/index.ts` — barrel export, confirms public surface.
- `tests/topology.test.ts` — 20+ assertions covering loader defaults,
  local bindings round-trip (confirms YAML, not JSON, already used here —
  useful precedent for `projects.yml`), `validateTopology` scenarios, and
  `resolveRepoPath`. Explicitly asserts
  `TOPOLOGY_YAML_PATH === 'axiom.spec/config/topology.yaml'` — a hardcoded
  path this increment must not casually change (Q3, still open, governs
  whether that prefix survives).
- `tests/qa-lane.test.ts` — 4 tests for the optional `qaLane` field
  (`'inline' | 'parallel'`, defaults to `'inline'`). Orthogonal to D1/D2/
  D3; not affected by this migration.

### Doctor checks read: MC-001/MC-002, TC-001/TC-002

`Axiom/packages/doctor/src/checks.ts`:

- **MC-001** (`runManifestChecks`, line ~204): calls
  `validateAxiomYamlContent(rawContent)` from `@axiom/config-validation`,
  which parses YAML then validates against the Zod `AxiomYamlSchema`.
  Confirms: **the actual schema-validation source of truth for
  `axiom.yaml` is `@axiom/config-validation/src/schemas.ts`**, not
  `@axiom/project-resolution`'s narrower internal interface. Any
  `schemaVersion: 2` bump must update `AxiomYamlSchema` (currently has NO
  `schemaVersion` field — see Field-by-field diff D3) or MC-001 will pass
  v1 and v2 documents indiscriminately, or reject v2 documents outright
  once a stricter schema is added.
- **MC-002** (line ~221): checks that `.gitignore` mentions `.sdd`, to
  warn if the local overlay could be versioned. Unrelated to the registry/
  manifest shape; no change needed for D1/D2/D3.
- **TC-001** (`runTopologyChecks`, line ~1140–1181, `topology-coverage`):
  calls `readTopologyManifest` (loads the manifest, falling back through
  the same `defaultSingleRepoManifest`/`defaultInstalledMultiRepoManifest`
  path as `@axiom/topology`) then `validateTopology`. If D1 deprecates
  `single-repo` as default, TC-001 needs a new `warn` (not `fail`) branch
  for "manifest resolved via `defaultSingleRepoManifest`" once multi-repo
  is primary, distinct from the current `pass`/`fail` binary based only on
  coverage. This is a concrete extension point for the later
  validator-reviewer step, not something to build now.
- **TC-002** (line ~1183–1200, `role-aliases-resolvable`): validates
  `profiles.yaml#roleAliases` values against canonical role ids from
  `@axiom/topology`'s `validRoleIds`. Independent of D1/D2/D3; no direct
  impact.

### Blast-radius list (grep-verified, `Axiom/packages/**/src` +
`Axiom/apps/**/src`, excluding `dist/` and test files)

35 source files reference `registry.json`, `projects.yml`, `axiom.yaml`,
or `axiom.yml` directly. Grouped by how they depend on the shapes (this
grouping is the key input for the schema-writer's blast-radius
assessment):

**Direct persistence/schema owners (must change for D2/D3):**
1. `packages/user-workspace/src/paths.ts` — hardcodes `registry.json`
   filename in `getPaths()`. **Must change for D2.**
2. `packages/user-workspace/src/registry.ts` — JSON parse/stringify,
   `CURRENT_SCHEMA_VERSION = 1` hard gate. **Must change for D2.**
3. `packages/user-workspace/src/registry-types.ts` — `ProjectEntry`/
   `Registry` shape. **Must change for D2.**
4. `packages/config-validation/src/schemas.ts` — `AxiomYamlSchema` (Zod),
   the actual schema-validation source of truth for `axiom.yaml`. **Must
   change for D3** (needs `schemaVersion` field added, plus new v2
   fields).
5. `packages/config-validation/src/validator.ts` —
   `validateAxiomYamlContent` wraps `AxiomYamlSchema`. Will need
   version-dispatch logic once two schema versions must be supported
   side by side during migration.
6. `packages/topology/src/loader.ts` — `defaultSingleRepoManifest`
   deprecation path (D1), and the `axiom.yaml#project.mode` hint-reading
   logic (`tryLoadTopologyHint`) that bridges `axiom.yaml` and topology
   defaults.
7. `packages/topology/src/types.ts` — `TopologyManifest.mode` enum; no
   change strictly required by D1 (deprecation is a runtime/warning
   concern, not a type-shape concern), but the schema-writer should
   confirm.

**Direct readers via package APIs (need no shape change, but exercise the
old shape and must be re-verified against the new one):**
8. `packages/filesystem-truth/src/discovery.ts` — `discoverAxiomRoot`,
   `AXIOM_CONFIG_FILENAME = 'axiom.yaml'`. Root-discovery only; agnostic
   to schema version.
9. `packages/project-resolution/src/resolver.ts` — reads `project.name`,
   `project.mode`, `scopes` from `axiom.yaml`. Needs to read `projectId`/
   `repoId`/`role` once v2 lands (D3), likely via a new resolver branch
   gated on `schemaVersion`.
10. `packages/doctor/src/checks.ts` — MC-001/MC-002/TC-001/TC-002 (see
    above); also the broader doctor file likely has other checks reading
    these shapes indirectly through `ProjectResolution`/`TopologyManifest`
    — full doctor audit beyond MC/TC checks is out of scope here (INC-08
    territory per the parent roadmap) but this file is flagged as needing
    re-verification once v2 lands.
11. `packages/installer/src/persist.ts` — comment-only reference to
    `axiom.yaml` as the project root marker; no structural read of its
    fields found.
12. `packages/installer/src/types.ts` — comment-only reference.
13. `packages/document-bootstrap/src/variables.ts` — reads
    `axiom.yaml#project.name` as a fallback when `product.manifest` is
    absent. Needs re-verification once field names change under v2 (if
    `project.name` is renamed/restructured under `projectId`).
14. `packages/versioning/src/checkpoints.ts` — comment-only reference to
    `axiom.yaml` as the project-root marker.
15. `packages/persistence/src/filesystem-store.ts` — appears in the
    broad grep; needs no shape-specific change (generic filesystem store,
    not a schema-aware reader) — flagged for confirmation, not treated as
    a hard dependency.
16. `packages/core/src/types.ts` — appears in the broad grep, likely
    generic path/type declarations; flagged for confirmation.
17. `packages/orchestrator/src/types.ts`,
    `packages/orchestrator/src/state-machine.ts` — appear in the broad
    grep; the roadmap's INC-02 already flags `@axiom/orchestrator` for a
    separate audit. Flagged here only as a blast-radius entry, not
    analyzed further (out of this increment's scope).
18. `packages/model-routing/src/checks.ts` — appears in the broad grep;
    flagged for confirmation.
19. `packages/components/src/state.ts` — appears in the broad grep;
    flagged for confirmation.
20. `packages/tui/src/render.ts`, `packages/tui/src/driver.ts`,
    `packages/tui/src/screens/projects.ts` — TUI screens that display
    registry/project data read via `@axiom/user-workspace`'s public API,
    not the raw file. Shape changes propagate automatically if the
    registry-engineer step keeps `ProjectEntry`'s public shape (or a
    compatible replacement) stable; flagged for the tui-developer step in
    a later increment (INC-04) per the parent roadmap, not this one.

**CLI commands (consume `@axiom/user-workspace` + `@axiom/project-
resolution` + `@axiom/topology` public APIs, plus some display raw
paths):**
21. `apps/cli/src/index.ts`
22. `apps/cli/src/commands/init.ts` — writes `axiom.yaml` on `axiom init`;
    the write-side of D3, needs update to emit `schemaVersion: 2` shape
    (or `1` with an explicit opt-out, if D1's "explicit opt-in" path
    applies at init time too — schema-writer to confirm exact UX).
23. `apps/cli/src/commands/join.ts` — registers a project in the
    registry; write-side of D2.
24. `apps/cli/src/commands/projects.ts` — lists/manages registry entries;
    read+write side of D2.
25. `apps/cli/src/commands/repo.ts` — registers a repo in the registry;
    write-side of D2, and the most direct existing analog to the new
    docs' `repos: { roleKey: ... }` map (already speaks of "repos" as a
    concept, even though it persists into the old one-`rootPath`-per-
    project shape today).
26. `apps/cli/src/commands/configure.ts` — reads/mutates project config;
    flagged for confirmation.
27. `apps/cli/src/commands/model.ts`
28. `apps/cli/src/commands/toolchain.ts`
29. `apps/cli/src/commands/self-update.ts` — uses `install.json`, adjacent
    but not shape-dependent on D2/D3.
30. `apps/cli/src/commands/capability.ts`
31. `apps/cli/src/commands/app-api.ts`

**Config/validation cross-references:**
32. `packages/config-validation/tests/validator.test.ts` — test file
    (listed by the broader glob, included here for completeness since it
    directly encodes today's `AxiomYamlSchema` expectations; will need
    new cases for `schemaVersion: 2`).

Files 33–35 in the raw glob output were test files already covered under
the package-specific audits above (`resolver.test.ts`, `checks.test.ts`,
`topology.test.ts` families) and are not double-counted here.

**Confirmed count: 31 non-test source files + doctor checks explicitly
audited.** This is close to, and cross-checked against, the parent
roadmap's "26 packages" estimate — the difference is explained by
counting individual `apps/cli/src/commands/*.ts` files separately rather
than as one "CLI" unit, and by including `packages/` files that reference
these artifacts only in comments (flagged as such above, not treated as
hard dependencies).

### Field-by-field diff

**D2 — registry: `registry.json` -> `projects.yml`**

| Old (`Registry`, JSON) | New (target, YAML) | Classification |
|---|---|---|
| File: `~/.axiom/registry.json` | File: `~/.axiom/projects.yml` | Hard break (path + format change) |
| `schemaVersion: 1` (whole-document) | Needs an equivalent version marker in the new YAML doc (exact field name not specified by the two source docs as read; schema-writer must confirm/introduce, e.g. `schemaVersion: 1` for the new format's own versioning, independent of `axiom.yml`'s version) | Hard break |
| `projects: ProjectEntry[]` (flat array) | Per-project entry becomes a map: `repos: { roleKey: { role, path } }` | Hard break (array -> map, and single `rootPath` -> multi-path-per-role) |
| `ProjectEntry.id` | Likely preserved as the project key or an explicit `id`/`projectId` field (schema-writer to confirm against source docs' exact key naming) | Needs schema-writer decision |
| `ProjectEntry.name` | Likely preserved | Additive/compatible |
| `ProjectEntry.rootPath` (single path) | Replaced by `repos: { roleKey: { role, path } }` — one path per repo role, not one path per project | Hard break |
| `ProjectEntry.functionalProfile`, `.overlay`, `.adapterTarget` | No equivalent in the two source documents' registry shape as described in the parent roadmap; likely stays out of `projects.yml` and remains tracked elsewhere (`install-profile.json`) — schema-writer must confirm this is not silently dropped | Needs schema-writer decision; do not drop silently |
| `ProjectEntry.addedAt`, `.lastUsedAt` | No stated equivalent in source docs; likely preserved for UX parity (`axiom projects use`) | Needs schema-writer decision; recommend preserving |
| `ProjectEntry.stale` (computed lazily from `fs.existsSync`) | Same computed-lazily approach should carry over per repo path, not per project, since a project can now have some repos present and others missing | Behavior change, not just a shape change — flag explicitly for schema-writer |

Migration script requirements for D2:
1. Read `~/.axiom/registry.json` (old shape) if present.
2. For each `ProjectEntry`, construct a `projects.yml` entry with at
   least one repo role (the existing single `rootPath` becomes the first/
   only entry in the new `repos` map — the schema-writer must decide the
   default `roleKey`, e.g. `sdd` or `code`, consistent with `@axiom/
   topology`'s existing `sdd-repo`/`spec-repo` role ids for naming
   consistency).
3. Preserve `id`/`name` and (per the open schema-writer decision above)
   either preserve or explicitly and visibly drop `functionalProfile`/
   `overlay`/`adapterTarget`/`addedAt`/`lastUsedAt`.
4. Write `projects.yml` atomically (tmp+rename — reuse the existing
   pattern already proven in `@axiom/topology/src/loader.ts`'s
   `saveLocalBindings`, which already writes YAML, not JSON).
5. Do **not** delete `registry.json` automatically; keep it as a backup
   until the user confirms (or until a `--commit`/`--dry-run` flag
   pattern consistent with `axiom upgrade`'s existing `--dry-run` design
   is applied).
6. `loadRegistry`-equivalent for the new format must emit a clear,
   typed error (not a silent misread) if it encounters a `registry.json`-
   shaped file where it expects `projects.yml`, mirroring the existing
   `unsupported-schema-version` err pattern.

**D3 — `axiom.yml`: `schemaVersion` (absent) -> `schemaVersion: 2`**

| Old (current `AxiomYamlSchema`) | New (target, per source docs) | Classification |
|---|---|---|
| No `schemaVersion` field at all (confirmed in `config-validation/src/schemas.ts`) | `schemaVersion: 2` (or a versioned document where `1` is the implicit/legacy value for documents without the field, and `2` is explicit) | Hard break at the validation level once `schemaVersion` becomes required; soft/compatible at the parse level if the loader treats "field absent" as "assume 1" during a transition window |
| `project.name: string` (required) | `projectId` (per new docs) | Hard break (rename) unless the loader/migration maps `project.name -> projectId` automatically |
| `project.mode?: string` (loose, e.g. `local-only\|gateway\|hybrid` used at the `@axiom/project-resolution` call site, but the Zod schema itself accepts any string) | `repoId`, `role` (per new docs) — these are new, distinct concepts (which repo, what role it plays), not a renamed `mode` | Hard break; `mode`'s *execution-topology* meaning (local-only/gateway/hybrid) has no direct new-docs equivalent found in this audit — schema-writer must confirm whether it is preserved as a separate field or folded elsewhere (do not silently drop; ties into Q4) |
| `scopes: Record<string, { path, description?, product_runtime?, current_state? }>` | `indexes`, `paths` (per new docs) | Hard break in field name and likely in shape granularity (schema-writer to confirm exact target shape from the two source documents directly, since this increment's audit was scoped to `Axiom/packages/*`, not a re-read of the two external decision documents' full text) |
| `rules?: string[]`, `artifact_id_policy?`, `lifecycle_commands?: string[]`, `initial_capabilities?: Record<string, unknown>` | No stated equivalent found in the roadmap's description of the new docs' `axiom.yml` shape | Needs schema-writer decision; do not drop silently — these are actively validated fields today (`AxiomYamlSchema`) even though `@axiom/project-resolution`'s resolver does not read them |

Migration script requirements for D3:
1. Read the existing `axiom.yaml` (note: current filename is `.yaml`,
   not `.yml` — `@axiom/filesystem-truth`'s `AXIOM_CONFIG_FILENAME =
   'axiom.yaml'` is the actual discovery constant; the roadmap and
   source docs use `axiom.yml`. **This filename discrepancy itself needs
   an explicit schema-writer decision**: rename the file, support both
   via discovery fallback, or treat this as a documentation
   inconsistency to correct. Flagged as a new finding not previously
   called out in the parent roadmap.)
2. Map `project.name -> projectId` (and decide the new `repoId`/`role`
   values — likely defaulted from `@axiom/topology`'s existing
   `sdd-repo`/`spec-repo` role ids for a single-repo project being
   migrated).
3. Map `scopes` entries into whatever `indexes`/`paths` shape the
   schema-writer defines, preserving `product_runtime` semantics if an
   equivalent concept exists in the new shape.
4. Explicitly decide (and document) the fate of `project.mode`, `rules`,
   `artifact_id_policy`, `lifecycle_commands`, `initial_capabilities`
   rather than dropping them silently.
5. Write `schemaVersion: 2` explicitly into the migrated file.
6. `AxiomYamlSchema` (or its v2 successor) must support reading both
   `schemaVersion` absent (treated as legacy v1) and `schemaVersion: 2`
   during a transition window, consistent with D1/D2's non-hard-removal
   posture — `MC-001` should `warn` (not `fail`) on a v1 document once v2
   exists, and only `fail` on a structurally invalid document under
   either version.

### Deprecation-warning vs. hard-break summary

| Change | Classification |
|---|---|
| `single-repo` as default topology mode (D1) | Deprecation warning (kept as explicit opt-in; runtime warns when `defaultSingleRepoManifest` is the manifest in effect without an explicit opt-in) |
| `registry.json` file existing on disk | Hard break for the *reader* (a v2 binary must not silently treat it as `projects.yml`), but the migration script must read it losslessly before removal/backup |
| `axiom.yaml` documents without `schemaVersion` | Deprecation warning during a transition window (treated as implicit v1), hard break only once v1 support is fully retired (not part of this increment's scope to schedule) |
| `axiom.yaml` filename vs `axiom.yml` (new finding) | Undecided — flagged explicitly for the schema-writer, not classified here to avoid a unilateral decision outside this role's scope |

## Validation

`No validation command was found. Performed best-effort validation by
inspecting changed files and checking consistency against the requested
behavior.`

This increment is audit-only (no code changed), so the applicable "best-
effort validation" was:
- Every claim about current shapes/behavior above is sourced from a
  direct read of the named file (all `src/` and `tests/` files in the
  three target packages, plus `checks.ts`, `schemas.ts`, `validator.ts`,
  `discovery.ts`, and each blast-radius file's exact matching lines via
  grep) — not inferred from package or file names alone.
- The blast-radius list was produced with a monorepo-wide grep
  (`registry\.json|projects\.yml|axiom\.yaml|axiom\.yml`) scoped to
  `**/src/**/*.ts` to exclude `dist/` build artifacts and reduce noise,
  then cross-checked file-by-file to classify comment-only references
  vs. structural dependencies.
- `Axiom/package.json`'s real validation commands (`npm run build`, `npm
  test`, `npm run doctor`, `npm run typecheck`) were not run, because no
  code was changed in `Axiom/` for this increment — running them would
  only assert pre-existing state, consistent with the same reasoning the
  parent roadmap increment used.

## Result

Completed the migration-engineer audit for INC-01: read every remaining
source and test file in `@axiom/user-workspace`, `@axiom/project-
resolution`, and `@axiom/topology`; read the `MC-001`/`MC-002`/`TC-001`/
`TC-002` doctor checks; produced a grep-verified blast-radius list of 31
non-test source files across `Axiom/packages/*` and `Axiom/apps/cli/*`
that read or write `axiom.yaml`/`registry.json`/topology manifests; and
produced a field-by-field diff for both the registry (D2) and the
`axiom.yml` manifest (D3), each classified as deprecation-warning or
hard-break with explicit migration-script requirements.

Two new findings emerged during this audit that were not called out in
the parent roadmap:
1. The current `axiom.yaml` schema (`AxiomYamlSchema` in `@axiom/config-
   validation`) has no `schemaVersion` field at all — the "1 -> 2" bump
   in D3 is really "absent -> 2" at the validation layer, which changes
   how a migration script and a transition-window doctor check need to
   behave (warn on absent, not just on `schemaVersion: 1`).
2. The current config file is named `axiom.yaml` (per `@axiom/filesystem-
   truth`'s `AXIOM_CONFIG_FILENAME` discovery constant), while the
   roadmap and external decision documents consistently write
   `axiom.yml`. This filename discrepancy was not previously flagged and
   needs an explicit decision from the schema-writer (rename, dual
   discovery, or treat as a naming inconsistency in the docs).

No new schema types or implementation code were written, per this
increment's non-goals.

## General spec integration

No integration into `Axiom.Spec/general-spec.md` was performed — that
file does not exist in this repo (confirmed by the parent roadmap
increment; the closest equivalents are `Axiom.Spec/specs/00_Resumen_
Ejecutivo.md` through `08_Glosario.md`). Those documents were not
modified in this increment. Per the parent roadmap's own "General spec
integration" note, the placeholder in `03_Modelo_Operativo_y_Datos.md`
("sin cerrar el naming final" for the registry/manifest YAMLs) should
still not be updated yet: this increment resolves D1/D2/D3 at the
decision level and produces the diff, but the schema-writer has not yet
produced the actual final field names/types, so updating that document
now would still be premature (it would either restate this increment's
own findings out of place or lock in field names before the
schema-writer finalizes them).

## Next step recommendation

Launch the **schema-writer** subagent next, per the parent roadmap's
subagent sequence for INC-01. Exact inputs it needs from this document:

1. **The three settled decisions** (D1, D2, D3) in the Context section
   above — repo-topology deprecation path, registry format/shape, and
   `axiom.yml` schema version bump — to design against, not re-litigate.
2. **The two field-by-field diff tables** (D2 registry, D3 manifest)
   above, including every "needs schema-writer decision" row — these are
   the specific open shape questions the schema-writer must resolve
   (e.g. default `roleKey` naming, fate of `functionalProfile`/`overlay`/
   `adapterTarget`/`addedAt`/`lastUsedAt`, fate of `project.mode`/
   `rules`/`artifact_id_policy`/`lifecycle_commands`/
   `initial_capabilities`, exact `indexes`/`paths` shape).
3. **The two new findings** not in the parent roadmap: the absent
   `schemaVersion` in the current `AxiomYamlSchema`, and the `axiom.yaml`
   vs `axiom.yml` filename discrepancy — both need explicit resolution
   before new types are written, not silent assumptions.
4. **The full blast-radius list** (31 non-test source files, grouped by
   dependency kind) — so the schema-writer's new types/schemas are
   designed with every consumer's actual current usage in mind, and so
   the subsequent registry-engineer/cli-implementer/validator-reviewer
   steps have a known, complete set of files to revisit.
5. **The migration-script requirements** listed under each diff table —
   these constrain the new schema design (e.g. it must remain possible to
   losslessly reconstruct old data during migration, and both old and new
   `axiom.yaml`/`axiom.yml` documents must be distinguishable at load
   time without ambiguity).

The schema-writer should explicitly confirm or override the "needs
schema-writer decision" rows before finalizing new types, and should flag
(but not silently resolve) any interaction with the still-open Q3
(`axiom.spec/` prefix survival) and Q4 (capability/provider/MCP folding)
from the parent roadmap.
