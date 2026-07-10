# Increment: dynamic team roles (P0-3 + "roles no fijos")

Status: closed
Date: 2026-07-10

## Goal

Let the architect register an ARBITRARY number (1..N) of team/code
roles — backend, frontend, mobile, qa, devops, or any other name —
one by one, and have every wizard/CLI-generated `topology.yaml` pass
`axiom topology validate`. Today `validateTopology`'s `validRoleIds`
is hardcoded to `profiles.yaml#functionalProfiles[].id`
(`product-owner`/`builder` only), so ANY multi-repo topology emitted
by `runWorkspaceSetup`/`runRepoAdd` (which assign `roleId` = the
architect's chosen implementation-role id, e.g. `backend`) fails
validation with `unknown-role`.

## Context

Two distinct "role" axes exist in Axiom, and they were conflated by
the bug this increment fixes:

1. **Install profiles** (`axiom.config/profiles.yaml#functionalProfiles`):
   `product-owner`, `builder` + aliases (`analista`, `arquitecto`).
   The CAPABILITY axis — drives the profile triple
   (functionalProfile + overlay + adapterTarget) and many existing
   tests. Managed by `axiom roles add` (unchanged by this increment).
2. **Team/code roles** (what the user means by "roles"): backend,
   frontend, mobile, qa, devops… — the roles that OWN code repos.
   Before this increment these had NO first-class registry: the
   wizard (`apps/cli/src/commands/tui.ts`'s `parseRolesInput`/
   `sanitizeRoleName`) invented arbitrary implementation-role ids on
   the fly and wrote them straight into
   `topology.yaml#assignments[].roleId`, with nothing ever
   registering them anywhere `validateTopology` would recognize.

Verified bug: `validateTopology(manifest, validRoleIds)`
(`packages/topology/src/validate.ts`) requires every
`assignments[].roleId ∈ validRoleIds`, where the CLI caller
(`apps/cli/src/commands/topology.ts`'s `loadProfilesYamlForValidation`)
passed ONLY `functionalProfiles[].id` = `{product-owner, builder}`.
The wizard's `buildTopologyManifest`
(`apps/cli/src/commands/workspace-setup.ts`, and its inline twin in
`workspace-incremental.ts`'s `runRepoAdd`) writes
`assignments[].roleId` = the chosen implementation-role id (e.g.
`backend`/`frontend`/`qa-e2e`/any custom name) — never a value from
`{product-owner, builder}`. Result: EVERY wizard-generated
multi-repo topology failed `axiom topology validate` with
`assignment.roleId "<x>" no está en functionalProfiles[].id`. Also,
`loadProfilesYamlForValidation` returned `null` (→ empty
`validRoleIds`) whenever `profiles.yaml` was absent, which the
setup engine never scaffolded — making validation fail even harder
for a fresh project.

## Scope

**Decision D5** — introduce a first-class, dynamic team-role
registry, decoupled from install profiles:

1. `TopologyManifest` gains an optional `roles?: readonly RoleDef[]`
   field (`RoleDef = { id, description? }`) in
   `packages/topology/src/types.ts`. Additive/back-compat:
   `schemaVersion` stays `1`; the loader's shape-guard accepts the
   field when present and treats it as `[]` when absent.
   `defaultSingleRepoManifest` defaults `roles` to `[]`.
2. `validateTopology`'s signature is UNCHANGED (`ReadonlySet<string>`).
   Every caller now builds `validRoleIds` as the union of:
   `manifest.roles[].id` (team roles) ∪
   `profiles.yaml#functionalProfiles[].id` (install profiles) ∪
   `functionalProfiles[].activatesImplementationRoles`. Updated call
   sites: `apps/cli/src/commands/topology.ts` (`axiom topology
   validate`), `packages/doctor/src/checks.ts` (TC-001), and
   `packages/tui/src/driver.ts` (topology screen). The `unknown-role`
   finding message no longer hardcodes "functionalProfiles[].id".
   `topology.ts`'s `loadProfilesYamlForValidation` now falls back to
   `DEFAULT_PROFILES` (`@axiom/install-profiles`) instead of
   returning `null` when `profiles.yaml` is absent, so `validRoleIds`
   is never empty just because the file is missing.
3. New CLI surface in `apps/cli/src/commands/roles.ts`:
   - `axiom roles register <id> [--description <d>] [--repo <repoId>] [--path <p>]`
     — appends a `RoleDef` to `topology.yaml#roles` (idempotent). If
     `--repo` is given, also creates the assignment in the same step.
   - `axiom roles unregister <id> [--path <p>]` — removes it from
     `topology.yaml#roles`. BLOCKS (exit 1) if the role still has
     active assignments (does not cascade-delete — see Assumptions).
   - `roles list` now ALSO shows registered team roles from
     `topology.yaml#roles`, clearly separated from install profiles
     (`--json` emits `{ installProfiles, teamRoles }`).
   - `runRolesAssign`/`runRolesUnassign` resolve `--role` against
     EITHER a registered team role OR an install profile/alias (via
     the new `resolveTeamOrProfileRole` helper) — team role checked
     first, falls back to `resolveRoleId` (profiles.yaml) otherwise.
   - `roles add` (install-profile axis) is untouched.
4. Producers fixed: `buildTopologyManifest`
   (`workspace-setup.ts`) and the inline manifest-rebuild in
   `runRepoAdd` (`workspace-incremental.ts`) now (a) register each
   architect-chosen code role into `topology.yaml#roles` via the new
   shared `buildRoleDefs` helper (preserving any manually-registered
   roles with no repo yet), (b) keep emitting
   `assignments[].roleId` referencing those roles (unchanged
   convention), and (c) best-effort scaffold
   `axiom.config/profiles.yaml` (seeded with `DEFAULT_PROFILES`) via
   the new `scaffoldProfilesYamlIfMissing` helper, only if the file
   doesn't already exist.
5. `Axiom/axiom.config/topology.yaml` (this workspace's own manifest,
   shipped by INC-B) gets a `roles: [{id: builder, ...}]` entry to
   exercise the new registry, without changing its existing
   assignment; it still passes `axiom topology validate`.

## Non-goals

- No change to the install-profile axis (`roles add`,
  `resolveRoleId`, `roleAliases`) — team roles have NO aliasing.
- No cascading auto-unassign on `roles unregister` — see Assumptions
  for the explicit design choice (block instead of cascade).
- No migration tooling for existing real installs' `topology.yaml`
  (none ship a populated `roles` block yet other than this repo's
  own, updated as part of this increment).
- No change to `axiom role add`/`runRoleAdd` (singular `role`
  command in `workspace-incremental.ts`, which creates a NEW code
  repo end-to-end) — unrelated command, out of scope.

## Acceptance criteria

- [x] `packages/topology`: `TopologyManifest.roles?` is additive,
      back-compat, and defaults to `[]` where applicable.
- [x] `validateTopology`'s signature is unchanged; every known caller
      (`topology.ts`, `doctor/checks.ts` TC-001, `tui/driver.ts`)
      builds the broadened `validRoleIds` union.
- [x] `axiom roles register <id>` registers an ARBITRARY team role
      (1..N, one at a time), idempotently, optionally assigning a
      repo in the same call.
- [x] `axiom roles unregister <id>` removes a team role, blocking
      with a clear message if it still has active assignments.
- [x] `axiom roles list` shows both install profiles and team roles,
      clearly separated.
- [x] `axiom roles assign`/`unassign` resolve `--role` against team
      roles OR install profiles/aliases.
- [x] `buildTopologyManifest` (`workspace-setup.ts`) and
      `runRepoAdd`'s manifest rebuild (`workspace-incremental.ts`)
      register roles + scaffold `profiles.yaml`; the generated
      topology passes `validateTopology`.
- [x] `Axiom/axiom.config/topology.yaml` still passes
      `axiom topology validate`, live, and exercises `roles`.
- [x] `npm run build` exits 0.
- [x] Live end-to-end proof in a scratch temp dir: register 3
      arbitrary roles (backend/frontend/mobile) + assign, and
      register a SINGLE role (devops) — both pass
      `axiom topology validate` (proves 1..N).
- [x] Full `npx vitest run` stays green (zero failures), including
      updated/added tests for the legitimately-changed behavior
      (`profiles.yaml` absence no longer implies an empty
      `validRoleIds`).

## Open questions

None blocking.

## Assumptions

- `roles unregister` BLOCKS (exit 1) rather than cascading the
  removal of dependent `assignments[]` entries. Rationale: silently
  deleting assignments as a side effect of removing a role
  definition is more surprising/dangerous than requiring an explicit
  `roles unassign` first — consistent with the codebase's existing
  preference for explicit, narrow mutations (e.g. `roles add`'s
  duplicate-id rejection) over cascading writes.
- Team roles have no aliasing (unlike install profiles): `--role`
  passed to `roles register`/`assign`/`unassign` is matched verbatim
  against `topology.yaml#roles[].id`.
- `scaffoldProfilesYamlIfMissing` never overwrites an existing
  `axiom.config/profiles.yaml` — an architect who has already
  customized it (e.g. via `roles add`) keeps their content.
- The `DEFAULT_PROFILES` fallback added to
  `loadProfilesYamlForValidation` is scoped to `axiom topology
  validate` (the CLI command named in the brief). `doctor/checks.ts`
  and `tui/driver.ts` were extended to union in `manifest.roles` +
  `activatesImplementationRoles`, but were NOT given the same
  DEFAULT_PROFILES-when-missing fallback (out of scope; each already
  has its own established, unchanged behavior for a missing
  `profiles.yaml` — doctor skips TC-001/TC-002 entirely, tui treats
  it as an empty set with a visible load-error finding).

## Implementation notes

- `packages/topology/src/types.ts`: added `RoleDef` type and
  `TopologyManifest.roles?: readonly RoleDef[]`.
- `packages/topology/src/loader.ts`: shape-guard accepts optional
  `roles`; `defaultSingleRepoManifest`/`defaultInstalledMultiRepoManifest`
  default it to `[]`.
- `packages/topology/src/index.ts`: exports `RoleDef`.
- `packages/topology/src/validate.ts`: `unknown-role` message no
  longer hardcodes "functionalProfiles[].id"; doc comments updated
  to describe the union-building responsibility of callers.
- `apps/cli/src/commands/topology.ts`: `loadProfilesYamlForValidation`
  now returns `DEFAULT_PROFILES` instead of `null` on missing file;
  new `buildValidRoleIds(manifest, profiles)` helper replaces
  `validRoleIdsFromProfiles`.
- `apps/cli/src/commands/roles.ts`: new `runRolesRegister`/
  `runRolesUnregister` + CLI subcommands; `runRolesList` extended
  with `teamRoles`/`TeamRoleEntry` + `formatTeamRolesList`;
  `runRolesAssign`/`runRolesUnassign` refactored to use the new
  `resolveTeamOrProfileRole` helper (loads topology manifest first,
  checks team roles, falls back to `resolveRoleId`/profiles.yaml).
- `apps/cli/src/commands/workspace-setup.ts`: new exported
  `buildRoleDefs`/`scaffoldProfilesYamlIfMissing` helpers;
  `buildTopologyManifest` gains an `existingRoles` param and now
  populates `roles`; `runWorkspaceSetup` calls
  `scaffoldProfilesYamlIfMissing` (best-effort) right after writing
  `topology.yaml`.
- `apps/cli/src/commands/workspace-incremental.ts`: `runRepoAdd`'s
  manifest-rebuild block now derives `roles` via `buildRoleDefs`
  (merged with any prior manually-registered roles from the existing
  `topology.yaml`) and calls `scaffoldProfilesYamlIfMissing`.
- `packages/doctor/src/checks.ts`: `readProfilesForTopologyCheck`
  also collects `activatesImplementationRoles` into `validRoleIds`;
  new `addTeamRoleIds(validRoleIds, manifest)` helper unions in
  `manifest.roles[].id` at the TC-001 call site only (TC-002's
  alias-dangling check intentionally keeps using the narrower
  `profiles.validRoleIds`, since aliases only ever target install
  profile ids).
- `packages/tui/src/driver.ts`: `loadTopologyData` seeds
  `validRoleIds` from `manifest.roles[].id` and also unions in
  `activatesImplementationRoles` from `profiles.yaml`.
- `Axiom/axiom.config/topology.yaml`: added `roles: [{id: builder,
  description: ...}]`.

Tests added/updated (all in the same commits as the code they cover):
- `packages/topology/tests/topology.test.ts`: `roles[]` accepted by
  the loader (+ back-compat when absent); `defaultSingleRepoManifest`
  defaults `roles` to `[]`.
- `apps/cli/tests/topology.test.ts`: updated the "profiles.yaml
  absent" bonus scenario to reflect the new `DEFAULT_PROFILES`
  fallback (legitimate behavior change — `builder` now resolves OK
  without a project `profiles.yaml`; a genuinely unregistered id
  still produces `unknown-role`); added a new "Decision D5" describe
  block covering `manifest.roles[]` and `activatesImplementationRoles`
  resolution.

## Validation

- `cd Axiom && npm run build` → exit 0 (`tsc -b`, clean, no errors).
- `npx vitest run packages/topology` → 2 files, 30 tests, all pass.
- `npx vitest run packages/install-profiles` → 4 files, 49 tests, all
  pass (untouched by this increment; run as a regression guard since
  `apps/cli` now imports `DEFAULT_PROFILES` from it in two more
  places).
- `npx vitest run apps/cli` → 65 files, 623 tests, all pass.
- Full `npx vitest run` → **207 files, 2184 tests, all pass** (up
  from the pre-increment baseline of 207 files / 2179 tests — +5 new
  tests, zero failures, zero regressions).
- LIVE end-to-end proof in a scratch temp dir (created under the
  session scratchpad, removed afterwards):
  - Multi-role case (3 arbitrary roles: `backend`, `frontend`,
    `mobile`), one `roles register` call each (one used `--repo` for
    a one-step register+assign; the other two used separate
    `roles register` + `roles assign` calls) →
    `axiom topology validate --path <tmp>` → `✓ Topología válida
    (mode: multi-repo).` exit 0.
  - Single-role case (`devops`, no `profiles.yaml` present at all in
    that temp dir) → `roles register devops --repo only-repo` →
    `axiom topology validate --path <tmp>` → `✓ Topología válida
    (mode: multi-repo).` exit 0 (also proves the `DEFAULT_PROFILES`
    fallback works when the file is fully absent).
  - Idempotency: re-running `roles register devops` → `✓ ... ya
    estaba registrado ... Nada que hacer.` exit 0.
  - `roles unregister devops` while still assigned → blocked, exit 1,
    message names the affected repo. After `roles unassign` →
    `roles unregister devops` succeeds, exit 0.
- LIVE against the real `Axiom/` repo (`node apps/cli/dist/index.js`):
  - `roles list` → exit 0; shows the 2 install profiles + 2 aliases,
    AND a new "Roles de equipo" section listing `builder` (assigned
    to `axiom-product-repo`).
  - `topology validate` → exit 0, `✓ Topología válida (mode:
    multi-repo).`

## Result

Closed. The root bug (validator's `validRoleIds` hardcoded to the
install-profile axis, while every producer wrote team/code-role ids)
is fixed by introducing the team-role registry as a genuinely
separate, first-class axis (Decision D5) rather than papering over
it by widening `functionalProfiles` — matching the user's explicit
requirement that team roles be "no fijos" (not a fixed enum, 1..N,
one at a time). Both known wizard/incremental producers
(`runWorkspaceSetup`, `runRepoAdd`) are fixed at the source, so every
NEW multi-repo workspace generated from now on produces a
`topology.yaml` that passes validation out of the box, and the
architect has a first-class CLI surface
(`register`/`unregister`/`list`/`assign`/`unassign`) to manage team
roles directly. All 3 other known `validateTopology` call sites
(doctor TC-001, TUI topology screen) were updated for consistency.
Full test suite validated green with zero regressions; live
end-to-end proof covers both the 1-role and N-role cases plus
idempotency and the unregister-blocking safety rail.

## General spec integration

`Axiom.Spec/general-spec.md` does not exist in this repo's structure
(canonical stable knowledge instead lives in `specs/03_Modelo_Operativo_y_Datos.md`,
which already documents `TopologyManifest`/`schemaVersion: 1` in
detail) — used that file as the equivalent, per the increment rule
"if that path does not exist, use the equivalent... in the spec
repository structure". Integrated: a new subsection "Dos ejes de
'rol', desacoplados (Decision D5, INC-20260710-dynamic-team-roles)"
right after the existing `TopologyManifest` bullets, documenting the
two decoupled role axes (install profiles vs. team/code roles), the
`topology.yaml#roles` registry, how `validRoleIds` is now built as a
union by every `validateTopology` caller, the
`axiom roles register/unregister/list` CLI surface, and which
producers (`runWorkspaceSetup`, `runRepoAdd`) now populate `roles` +
scaffold `profiles.yaml`. Also updated the `TopologyManifest`
TypeScript snippet in that same file to show the new `roles?` field.
