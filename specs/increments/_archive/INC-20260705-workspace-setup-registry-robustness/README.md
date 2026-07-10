# Increment: Workspace setup registry robustness

Status: closed
Date: 2026-07-07

## Goal

Make `runWorkspaceSetup` (`apps/cli/src/commands/workspace-setup.ts`)
complete the FULL local install (adapters, `workspace.json`, SDD skills,
spec base, `.axiom/mcp.yml` + adapter MCP configs) even when user-level
registry registration fails or the user has an unmigrated legacy v1
registry (`~/.axiom/registry.json` present, `~/.axiom/projects.yml`
absent). Additionally, auto-migrate the legacy v1 registry to v2 when
that specific failure mode is detected, so the common case (an existing
Axiom user running the new multi-repo workspace setup for the first
time) self-heals instead of leaving a half-completed install.

## Context

A real multi-repo install came out HALF-COMPLETED: per-repo `axiom.yaml`,
`topology.yaml`, and local bindings were written, but the user-level
registry entry, `.axiom/mcp.yml`, all adapters, `workspace.json`, the SDD
skills, and the spec base were ALL missing.

Root cause, confirmed in code:

1. The user had a legacy `~/.axiom/registry.json` (v1) and no
   `projects.yml` (v2). `loadRegistryV2`
   (`packages/user-workspace/src/registry.ts`) returns
   `err(legacyNotMigratedError(...))` in exactly that situation
   (`projects.yml` absent + `registry.json` present). `saveRegistryV2`
   explicitly refuses to write/migrate `registry.json` ("that's `axiom
   upgrade`'s job") — and no such migration existed anywhere in the
   codebase.
2. `runWorkspaceSetup` calls `upsertProjectReposV2` (from the prior
   `INC-20260705-workspace-multirepo-setup-engine`). On failure
   (legacy-not-migrated, `ensureHomeDir` failure, or any upsert error)
   the engine EARLY-RETURNED with `registryRegistered:false`. The later
   best-effort steps — MCP (`writeWorkspaceMcpConfig`), adapters
   (`generateWorkspaceAdapters`), `workspace.json`, SDD skills
   (`scaffoldSddSkills`), spec base (`scaffoldSpecRepoBase`) — were all
   added by LATER siblings increments
   (`INC-20260705-workspace-mcp-generation`,
   `INC-20260705-workspace-adapters-multiselect`,
   `INC-20260705-workspace-sdd-skills`,
   `INC-20260705-workspace-spec-base`) AFTER those early-returns in the
   control flow, so a registration failure silently skipped every one of
   them, even though none of those local-scaffolding steps actually
   need the registry (only MCP's `validateMcpProjectConfig` does, and
   even that already degrades to a warning rather than throwing).

## Scope

- Restructure `runWorkspaceSetup` so registry registration is
  best-effort and non-blocking: a registration failure or
  `spec.register===false` records `registryRegistered:false` + a clear
  warning, but does NOT early-return. All local scaffolding steps
  (adapters, `workspace.json`, SDD skills, spec base, MCP config) always
  run afterward.
- Add `migrateLegacyRegistryV1ToV2(homeDir)` to `@axiom/user-workspace`:
  converts `registry.json` (v1) to `projects.yml` (v2), preserving the
  legacy file (renamed, not deleted). Wire it into `runWorkspaceSetup`:
  on a `legacy-registry-not-migrated` upsert failure, migrate once and
  retry the upsert.
- Focused unit tests for the migration helper and the non-blocking
  control flow.

## Non-goals

- No changes to `runInit` (single-repo `axiom init`) — it keeps its own
  existing legacy-not-migrated WARN handling untouched. The migration
  helper is additive infrastructure; `runInit` may adopt it in a future
  increment.
- No general-purpose `axiom upgrade` command (out of scope; the
  migration helper is invoked internally by `runWorkspaceSetup` only).
- No changes to `apps/cli`'s TUI wizard surface.
- No git operations of any kind.

## Acceptance criteria

- [x] `runWorkspaceSetup` no longer early-returns on registration
      failure or `register:false`; local scaffolding (adapters,
      `workspace.json`, SDD skills gated on control `created`, spec base
      gated on spec `created`) always executes regardless of
      `registryRegistered`.
- [x] MCP config (`.axiom/mcp.yml` + adapter `mcp.json`) is always
      attempted; when the project id does not resolve in the registry,
      `validateMcpProjectConfig` degrades to an `unknown-project`
      warning (mcp.yml is written regardless — validate-after-write,
      matching the pre-existing pattern), not a skip.
- [x] `migrateLegacyRegistryV1ToV2(homeDir)` converts each v1
      `ProjectEntry` to a v2 `ProjectEntryV2` (`repos` map keyed by the
      v1 entry's role, defaulting to `'sdd'` when absent), preserves
      `id`/`name`/`addedAt`/`lastUsedAt`, writes `projects.yml` via
      `saveRegistryV2`, and renames `registry.json` →
      `registry.json.migrated` (never deletes user data).
- [x] `runWorkspaceSetup` calls the migration once and retries
      `upsertProjectReposV2` when the initial upsert fails with
      `legacy-registry-not-migrated`; if migration or the retry still
      fails, execution falls back to the non-blocking warn-and-continue
      path (no throw).
- [x] `apps/cli/tests/workspace-setup.test.ts` covers: legacy-v1-registry
      auto-migration end-to-end (migrated entries + new project both
      present in `projects.yml`, `registry.json.migrated` exists, ALL
      local scaffolding produced); `register:false` still produces all
      local scaffolding; a simulated registration failure still produces
      all local scaffolding.
- [x] `packages/user-workspace/tests/migrate-legacy-registry.test.ts`
      unit-tests the migration helper in isolation (multi-entry mapping,
      legacy file preservation, idempotency when `projects.yml` already
      exists).
- [x] `npm run build` (tsc -b) clean from `Axiom/`.
- [x] `npx vitest run apps/cli packages/user-workspace packages/topology
      packages/adapters` — new and existing tests pass (pre-existing
      unrelated failures classified, not silently ignored).

## Open questions

None blocking. The task brief supplied fully closed design decisions
(non-blocking control flow, migration mapping rules, retry-once
semantics). Narrow implementation choices are recorded under
Assumptions.

## Assumptions

- The real v1 `ProjectEntry` shape (confirmed in
  `packages/user-workspace/src/registry-types.ts`) is `{ id, name,
  rootPath, functionalProfile, overlay, adapterTarget, addedAt,
  lastUsedAt, stale }` — a SINGLE repo per entry (`rootPath`), with no
  explicit "role" field. Since v1 has no role concept at all, every v1
  entry maps its one repo into the v2 `repos` map under the key `'sdd'`
  (the task brief's instruction "if v1 entries lack a role, default the
  repo role to `sdd`" applies unconditionally here, since v1 never had a
  role field to begin with).
- `stale`, `functionalProfile`, `overlay`, `adapterTarget` are dropped
  during migration (already documented in `registry-types.ts` as
  "deprecated-with-warning at migration time" for v2 — read
  `install-profile.json` instead). This is pre-existing, documented
  design intent, not a new decision made in this increment.
- Migration is invoked lazily and narrowly: only from inside
  `runWorkspaceSetup`, only on the specific
  `legacy-registry-not-migrated` cause, and only once per call (no retry
  loop). A standalone `axiom upgrade` CLI command remains unimplemented
  (pre-existing gap, out of scope here — the error message pointing
  users at `axiom upgrade` is technically still slightly aspirational,
  now with a partial in-code equivalent for the workspace-setup path
  only).
- If `migrateLegacyRegistryV1ToV2` itself fails (e.g. the rename fails
  due to file locks), the code proceeds using the freshly-created
  `projects.yml` in memory (migration already succeeded logically by
  the time the rename is attempted) — rename failure is a warning-level
  concern for user cleanup, not a reason to abort the migration or the
  workspace setup.
- `packages/skills/tests/catalog.test.ts > "carga el catálogo real del
  repo"` is a known pre-existing dogfood gap unrelated to this
  increment; it is excluded from the validation scope's pass/fail
  judgment per the task brief.

## Implementation notes

- `packages/user-workspace/src/migrate-legacy-registry.ts` (new):
  `migrateLegacyRegistryV1ToV2(homeDir): Result<ProjectsFile,
  UserWorkspaceError>`. Reads `registry.json` via the existing
  `loadRegistry` (v1 loader, already handles missing/corrupt files),
  short-circuits with `ok` (no-op) if `projects.yml` already exists
  (idempotency guard using `getPaths(homeDir).registryPath` +
  `fs.existsSync`), maps every `ProjectEntry` to a `ProjectEntryV2` with
  a one-entry `repos` map (`{ sdd: { role: 'sdd', path: rootPath } }`),
  writes via `saveRegistryV2`, and renames `registry.json` →
  `registry.json.migrated` (best-effort; a rename failure is folded into
  the returned warning-equivalent path — see Assumptions — not
  propagated as an `err`, since the v2 file was already durably written
  by that point). Barrel-exported from
  `packages/user-workspace/src/index.ts`.
- `apps/cli/src/commands/workspace-setup.ts`: replaced the three
  early-return blocks (`spec.register===false`, `ensureHomeDir` failure,
  `upsertProjectReposV2` failure) with warning-only branches that fall
  through to a single unified continuation. Registration now resolves
  to a `{ registered: boolean; effectiveProjectId: string }`-shaped
  local state computed up front (steps 1-2 unchanged; step 3 rewritten),
  and steps 4-8 (MCP, adapters, `workspace.json`, SDD skills, spec base)
  now run unconditionally after it, using `effectiveProjectId` (falls
  back to the pre-registration `projectId` when registration never
  completed) and reading `registryRegistered` only to decide the final
  return value.
- On `upsertProjectReposV2` failing with
  `cause.kind === 'legacy-registry-not-migrated'`, the engine now calls
  `migrateLegacyRegistryV1ToV2(homeDir)` once, and on success retries
  `upsertProjectReposV2` exactly once with the same input. Any
  remaining failure (migration itself, or the retried upsert) falls
  through to the same warn-and-continue path as any other registration
  failure.
- `writeWorkspaceMcpConfig` needed NO changes: it already writes
  `.axiom/mcp.yml` unconditionally and folds
  `validateMcpProjectConfig` issues (including `unknown-project`, the
  exact issue produced when a project id does not resolve) into
  `warnings` rather than throwing (validate-after-write, pre-existing
  pattern). The fix is entirely in `workspace-setup.ts`'s control flow:
  simply no longer skip calling `buildWorkspaceMcpServers` /
  `writeWorkspaceMcpConfig` when `registryRegistered` is `false`.

## Validation

From `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (tsc -b) — clean.
- `npx vitest run apps/cli packages/user-workspace packages/topology
  packages/adapters` — all passing except the pre-existing, out-of-scope
  `packages/skills/tests/catalog.test.ts > "carga el catálogo real del
  repo"` dogfood gap (unaffected by this increment's changes, not part
  of the filtered package scope run here since `packages/skills` was not
  in the requested filter — verified not newly broken).

See the executor's final report for verbatim command tails.

## Result

`runWorkspaceSetup` now completes ALL local file scaffolding
unconditionally, independent of registry registration outcome.
Registration failures (including an unmigrated legacy v1 registry) are
now self-healing via a one-shot `migrateLegacyRegistryV1ToV2` + retry,
and any remaining failure degrades to a warning instead of truncating
the install. New unit tests cover both the migration helper in isolation
and the end-to-end non-blocking control flow.

## General spec integration

Integrated into the canonical spec in the round-3 cross-increment pass
(covering this increment and the two sibling round-3
`INC-20260705-workspace-*` increments: wizard-ux-custom-roles and
code-repo-skills). Files updated:

- **03_Modelo_Operativo_y_Datos.md** (PRIMARY) — reworded the registry
  bullet to point at the new non-blocking scope; added the "Registro no
  bloqueante y auto-migración de registry v1→v2" subsection (registration
  non-blocking for ALL local scaffolding, `registryRegistered` reflects
  reality, MCP degrades to an `unknown-project` warning while `mcp.yml` is
  still written; `migrateLegacyRegistryV1ToV2` v1→v2 auto-migration
  invoked once + retry-once on `legacy-registry-not-migrated`, legacy file
  preserved as `registry.json.migrated`); fixed the round-2 gating
  paragraph's stale "tras un registro exitoso" phrasing for multi-adapter
  generation.
- **01_Requisitos_Funcionales.md** — added a robustness sub-bullet to
  RF-AXM-024 (install completes all local scaffolding even with a failed
  registration / legacy v1 registry / `register:false`; auto-migration +
  retry).
- **07_Gobierno_y_Seguridad.md** — added an ownership/data-safety
  paragraph to the write-scope/ownership section: a failed/legacy registry
  no longer silently aborts the install, and the v1→v2 migration never
  deletes user data (renames the legacy `registry.json` to
  `registry.json.migrated`).
- **08_Glosario.md** — added the term "migración de registry v1→v2
  (`registry.json.migrated`)".
