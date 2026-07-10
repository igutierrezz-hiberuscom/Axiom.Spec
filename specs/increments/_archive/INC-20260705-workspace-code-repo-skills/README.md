# Increment: Autoskills en los repos de código

Status: closed
Date: 2026-07-07

## Goal

When `runWorkspaceSetup` creates a brand-new role/code repo
(`created === true`), install THAT role's skills baseline INTO the code
repo itself — not only into the control/SDD repo. Each freshly created
role code repo ends up with its own scoped skills baseline (catalog +
materialized skills + one `axiom.config/skills-index/<roleId>.yaml`
matching its own functional role), reusing the exact same bundled seed
and `@axiom/skills` machinery already used for the control repo, with
no duplication.

## Context

`apps/cli/src/commands/workspace-skills.ts` already implements
`scaffoldSddSkills({controlRepoPath, projectId, functionalRoleIds,
homeDirOverride?})` (`INC-20260705-workspace-sdd-skills`), which — only
when the control/SDD repo is freshly `created` — writes a bundled seed
`axiom.config/skills-catalog.yaml` (+ seed source `.md` files under
`axiom.spec/target-axiom-skills/<id>.md`), materializes
`.opencode/agents/<id>/SKILL.md` + `.axiom-state/<projectId>/
skills-pending.json` via `@axiom/skills`'s `applySkillSet`, and writes
one `axiom.config/skills-index/<role>.yaml` per functional role
declared in the workspace. This step runs ONLY for the control repo —
role/code repos are explicitly untouched ("Sólo el repo de control
recibe skills").

`apps/cli/src/commands/workspace-setup.ts`'s `runWorkspaceSetup`
already tracks, per repo, whether it was freshly `created` in this call
(`repoResults`), and already gates both `scaffoldSddSkills` (control)
and `scaffoldSpecRepoBase` (spec) on their own repo's `created` flag —
`INC-20260705-workspace-setup-registry-robustness` made these
best-effort steps run unconditionally with respect to registry
outcome, but the per-repo `created` gating itself is unchanged.

A sibling increment (`INC-20260705-workspace-wizard-ux-custom-roles`)
made functional role ids fully arbitrary free text (any name, any
count) — role ids are not a closed enum of
backend/frontend/qa-e2e, so this increment's `roleId` parameter must
accept any sanitized string.

## Scope

- `apps/cli/src/commands/workspace-skills.ts`:
  - Factor the seed-writing/catalog-building logic that
    `scaffoldSddSkills` already uses (`writeSeedSourcesAndBuildCatalog`,
    `buildRoleIndex`, `renderRoleIndexYaml`, `atomicWrite`) so it can be
    reused, unchanged in content, by a new function — no duplication of
    the bundled seed sources or catalog shape.
  - New `scaffoldCodeRepoSkills({repoPath, projectId, roleId,
    homeDirOverride?}): Promise<{filesCreated: string[]; warnings:
    string[]}>`: writes, into `repoPath` (a role/code repo):
    - `axiom.config/skills-catalog.yaml` (same bundled seed pool as the
      control repo) + its seed source `.md` files under
      `axiom.spec/target-axiom-skills/<id>.md`.
    - materializes the seed's desired skill set via
      `loadSkillRegistry` + `applySkillSet(registry, repoPath,
      projectId, SEED_DESIRED_IDS)` → `.opencode/agents/<id>/SKILL.md`
      + `.axiom-state/<projectId>/skills-pending.json`.
    - ONE `axiom.config/skills-index/<roleId>.yaml` scoped to this
      repo's own role (`role: roleId`, `repoKinds: ['role']`),
      validated with `validateSkillsRoleIndex` (pre and post write);
      validation issues become warnings, the file is written anyway.
    - Best-effort: any internal failure becomes a warning entry, never
      throws.
- `apps/cli/src/commands/workspace-setup.ts` (`runWorkspaceSetup`):
  after the existing control-repo skills step, for EACH repo with
  `kind === 'role'` whose own `created === true` (from `repoResults`),
  call `scaffoldCodeRepoSkills({repoPath: repo.path, projectId:
  effectiveProjectId, roleId: repo.functionalRoleId ?? repo.roleKey,
  homeDirOverride: spec.homeDirOverride})`, best-effort, appending
  `filesCreated`/`warnings`. Role repos that already existed
  (`created === false`) are skipped — never clobbered. Control/spec
  repos are not touched by this new step.
- Tests: extend `apps/cli/tests/workspace-skills.test.ts` for
  `scaffoldCodeRepoSkills`, and `apps/cli/tests/workspace-setup.test.ts`
  for the wired end-to-end behavior.

## Non-goals

- No changes to `scaffoldSddSkills`'s own behavior or its control-repo
  gating (unchanged, additive-only increment).
- No changes to `runInit`/`axiom init` (single-repo path).
- No changes to the spec-repo base scaffolding
  (`scaffoldSpecRepoBase`).
- No assumption of a closed role enum (backend/frontend/qa-e2e) — any
  sanitized role id string must work identically.
- No `Axiom.Spec/specs/00-08` integration in this pass (orchestrator
  performs the final cross-increment integration pass, per precedent of
  sibling `INC-20260705-workspace-*` increments).

## Acceptance criteria

- [x] `scaffoldCodeRepoSkills` on a tmp code repo with an ARBITRARY role
      id (e.g. `data`) produces: `axiom.config/skills-catalog.yaml`
      parseable by `loadSkillsCatalog` as `kind: 'ok'`; at least one
      materialized `.opencode/agents/<id>/SKILL.md`;
      `.axiom-state/<projectId>/skills-pending.json`; and
      `axiom.config/skills-index/data.yaml` accepted by
      `loadSkillsRoleIndex`/`validateSkillsRoleIndex` as `kind: 'ok'`
      with `role === 'data'`.
- [x] Re-running `scaffoldCodeRepoSkills` on the same repo is idempotent
      (no throw).
- [x] `runWorkspaceSetup` on a spec with two freshly-created role repos
      (`backend` and an arbitrary `data`) ends with EACH role repo
      having its own skills-catalog + `skills-index/<role>.yaml` +
      materialized skills, scoped to that repo's own role.
- [x] `runWorkspaceSetup` on a spec where a role repo's path pre-exists
      (`create: false`) skips code-repo skills scaffolding for that
      repo entirely (no `axiom.config/skills-catalog.yaml` written by
      this step).
- [x] The control repo's own `scaffoldSddSkills` behavior is unchanged
      (still control-repo-only, still gated on the control repo's own
      `created` flag).
- [x] The new code-repo skills step never fails the overall
      `runWorkspaceSetup` call, even if `@axiom/skills` throws
      internally.
- [x] Existing tests keep passing (the known pre-existing
      `packages/skills/tests/catalog.test.ts` dogfood failure is out of
      scope and unrelated).
- [x] `npm run build` (tsc -b) clean from `Axiom/`.
- [x] Focused vitest run (`apps/cli`, `packages/skills`,
      `packages/user-workspace`, `packages/topology`) green (new +
      pre-existing, aside from the known unrelated failure).

## Open questions

None blocking — the increment prompt supplied fully closed design
decisions (function shape, reuse strategy, wiring gate, arbitrary role
id handling). Narrow implementation choices are recorded under
Assumptions.

## Assumptions

- `scaffoldCodeRepoSkills` reuses the exact same bundled seed (5 ids:
  3 canonical Axiom skills + the 2 `DEFAULT_DESIRED_SKILLS`) as
  `scaffoldSddSkills` — the catalog is the same installable pool
  regardless of whether the target repo is the control repo or a code
  repo; only the per-repo `skills-index` differs (one file, scoped to
  that repo's own `roleId`, vs. one file per functional role in the
  control repo).
- The per-role-repo `skills-index/<roleId>.yaml` uses the same
  mandatory/available split as `buildRoleIndex` in the control-repo
  path (`axiom-sdd-orchestrator` + `axiom-context-persistence`
  mandatory; the rest available with role-derived tags) — no new
  curation logic invented for this increment.
- `roleId` passed to `scaffoldCodeRepoSkills` is
  `repo.functionalRoleId ?? repo.roleKey`, matching the exact same
  fallback already used elsewhere in `workspace-setup.ts`
  (`buildTopologyManifest`, `scaffoldSddSkills`'s call site).
- `projectId` passed to `scaffoldCodeRepoSkills` from
  `runWorkspaceSetup` is `effectiveProjectId` (the same id used for the
  control repo's `scaffoldSddSkills` call and for `workspace.json`),
  so `.axiom-state/<effectiveProjectId>/skills-pending.json` is
  consistent across every repo of the same workspace.
- Gate is strictly on each role repo's own `created` flag (from
  `repoResults`, computed by `writeOneRepo`), not on whether the
  catalog file already exists in that repo — mirrors the exact
  precedent of the control-repo gate in `scaffoldSddSkills`'s call
  site.
- `repoKinds: ['role']` (not `['backend']`/etc.) on the per-code-repo
  skills-index, since `repoKinds` scopes by structural repo KIND
  (`WorkspaceRepoKind`), and the `role` field itself already carries
  the specific functional role id — consistent with
  `SkillsRoleIndex.repoKinds`'s documented purpose in
  `packages/skills/src/role-index.ts`.

## Implementation notes

- `apps/cli/src/commands/workspace-skills.ts`:
  - Extracted `writeCodeRepoSkillsBaseline(repoPath, projectId,
    filesCreated, warnings)` — the shared core (write seed sources +
    catalog, `loadSkillRegistry` + `applySkillSet`) used by BOTH
    `scaffoldSddSkills` (control repo) and the new
    `scaffoldCodeRepoSkills` (role repo), so the bundled seed content
    and materialization logic exist in exactly one place.
  - `scaffoldSddSkills` now calls this shared helper, then still does
    its own per-functional-role loop (`buildRoleIndex` for EACH role in
    `functionalRoleIds`) — unchanged output for the control-repo path.
  - New `scaffoldCodeRepoSkills({repoPath, projectId, roleId,
    homeDirOverride})`: calls the same shared helper for
    seed+materialization, then writes exactly ONE
    `axiom.config/skills-index/<roleId>.yaml` via the same
    `buildRoleIndex`/`renderRoleIndexYaml`/`validateSkillsRoleIndex`/
    `loadSkillsRoleIndex` pre/post-write validation pattern already
    used by `scaffoldSddSkills`'s per-role loop — reused verbatim (no
    parallel implementation), just called with a single `roleId`
    instead of iterated over `functionalRoleIds`. Whole function body
    wrapped in try/catch — any thrown error becomes a warning, never
    propagates.
  - `buildRoleIndex(role)` is reused unmodified (already pure, already
    accepts an arbitrary string `role`).
- `apps/cli/src/commands/workspace-setup.ts`: after the existing
  control-repo skills best-effort block (step 7), a new best-effort
  block iterates `roleRepos`, looks up each one's own `created` flag
  from `repoResults` (matched by `topologyId`), and — only when
  `true` — calls `scaffoldCodeRepoSkills` with that repo's own `path`,
  `effectiveProjectId`, and `roleId: repo.functionalRoleId ??
  repo.roleKey`. Appends `filesCreated`/`warnings` per repo; a thrown
  error from one repo's scaffolding is caught per-iteration so one
  repo's failure doesn't skip the others.
- No enum introduced anywhere — `roleId` stays a plain `string`,
  matching `WorkspaceRepoSpec.functionalRoleId`/`roleKey`'s existing
  open-string contract.

## Validation

From `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (tsc -b) — clean, no output (success).
- `npx vitest run apps/cli packages/skills packages/user-workspace packages/topology`:

  ```
  Test Files  1 failed | 67 passed (68)
       Tests  1 failed | 651 passed (652)
  ```

  The 1 failing test is pre-existing and unrelated (same one already
  documented by the sibling `INC-20260705-workspace-sdd-skills`
  increment): `packages/skills/tests/catalog.test.ts` > "carga el
  catálogo real del repo (axiom.config/skills-catalog.yaml)" expects the
  Axiom product repo's OWN root `axiom.config/skills-catalog.yaml` (7
  official skills) to exist; it does not — a pre-existing gap unrelated
  to and untouched by this increment. All new tests are green:
  `apps/cli/tests/workspace-skills.test.ts` grew from 8 to 14 tests
  (added `scaffoldCodeRepoSkills` coverage: seed catalog reuse with an
  arbitrary role, materialization, single scoped skills-index,
  idempotency, best-effort resilience); `apps/cli/tests/
  workspace-setup.test.ts` grew from 14 to 16 tests (new Scenario (i):
  both `backend` and an arbitrary `data` role repo receive their own
  skills baseline; a pre-existing role repo with `create: false` is
  skipped while its sibling still receives scaffolding). One
  pre-existing assertion in Scenario (f) was updated (not weakened) to
  reflect the new, intended behavior: the `backend` role repo now DOES
  receive its own `skills-catalog.yaml` (previously asserted `false`,
  which was the exact behavior this increment changes by design); the
  spec repo continues to receive none.

## Result

Implemented `scaffoldCodeRepoSkills` (`apps/cli/src/commands/
workspace-skills.ts`), reusing the exact same bundled seed catalog and
`@axiom/skills` materialization engine as `scaffoldSddSkills` via a new
shared internal helper (`writeCodeRepoSkillsBaseline`) plus the
existing `buildRoleIndex`/`renderRoleIndexYaml` role-index logic
(extracted into `writeRoleSkillsIndex`, reused by both the control
repo's per-functional-role loop and the new single-role code-repo
call) — zero duplication of seed content or catalog shape. Wired into
`runWorkspaceSetup` as a new best-effort step (7b) that iterates every
`kind: 'role'` repo, gates strictly on that repo's own `created` flag
(from `repoResults`, same pattern as the existing control/spec gates),
and calls `scaffoldCodeRepoSkills` per freshly-created role repo with
an arbitrary `roleId` (`functionalRoleId ?? roleKey`) — a failure in
one role repo's scaffolding is caught per-iteration and never blocks
the others or the overall setup. `scaffoldSddSkills`'s own behavior and
gating are unchanged. Build and focused test suite are both green
aside from the one pre-existing, unrelated, out-of-scope failure.

## General spec integration

Integrated into the canonical spec in the round-3 cross-increment pass
(covering this increment and the two sibling round-3
`INC-20260705-workspace-*` increments: setup-registry-robustness and
wizard-ux-custom-roles). Files updated:

- **06_Integraciones_y_Capacidades.md** (PRIMARY) — added the "Skills
  también scaffoldeadas en cada repo de código recién creado" subsection,
  superseding the "sólo el repo de control recibe skills" restriction:
  skills are now scaffolded into BOTH the control/SDD repo AND each
  freshly-created code repo via `scaffoldCodeRepoSkills` (same bundled
  seed + machinery, no duplication; one role-scoped
  `skills-index/<roleId>.yaml` per code repo; per-repo best-effort, gated
  on each role repo's own `created`).
- **03_Modelo_Operativo_y_Datos.md** — added the "Autoskills por repo de
  código" subsection documenting the per-code-repo data shapes
  (`skills-catalog.yaml` + materialized `.opencode/agents/<id>/SKILL.md`
  + `skills-pending.json` + one `skills-index/<roleId>.yaml` scoped to
  that repo's own, possibly-custom, role).
- **01_Requisitos_Funcionales.md** — added a per-code-repo-skills
  sub-bullet to RF-AXM-024.
- **07_Gobierno_y_Seguridad.md** — noted the code-repo autoskills inherit
  the same best-effort / created-gated / no-clobber ownership posture.
- **08_Glosario.md** — added the term "autoskills de repo de código".
