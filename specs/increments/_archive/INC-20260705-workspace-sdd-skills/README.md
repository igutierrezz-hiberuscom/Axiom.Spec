# Increment: Workspace SDD skills baseline

Status: closed
Date: 2026-07-05

## Goal

When `runWorkspaceSetup` creates a brand-new control/SDD repo (`created ===
true`), scaffold an Axiom SKILLS baseline into it using the real
`@axiom/skills` API: a seeded `axiom.config/skills-catalog.yaml`, the
matching materialized `.opencode/agents/<id>/SKILL.md` files +
`skills-pending.json`, and one `axiom.config/skills-index/<role>.yaml`
per functional role declared for the workspace. If the control repo
already existed, skip entirely (never clobber). Only the control/SDD
repo gets skills — skills are an SDD-repo concern, not a spec/role-repo
concern.

## Context

Builds on `INC-20260705-workspace-multirepo-setup-engine`
(`runWorkspaceSetup` in `apps/cli/src/commands/workspace-setup.ts`),
`INC-20260705-workspace-mcp-generation`, and
`INC-20260705-workspace-adapters-multiselect`. `@axiom/skills`
(`packages/skills`) already implements the full runtime API
(`loadSkillsCatalog`, `loadSkillRegistry`, `applySkillSet`,
`materializeSkillSet`, `loadSkillsRoleIndex`/`validateSkillsRoleIndex`)
but the product does not bundle any canonical catalog or skill source
content anywhere — it only exists as test fixtures
(`packages/skills/tests/apply.test.ts`'s `scaffoldWithCatalog`,
`packages/doctor/tests/skills.test.ts`) and as the shape documented in
`Axiom.Spec/templates/product-skill-template.md`. `packages/doctor`'s
TC-010 smoke test even references 7 canonical ids
(`axiom-sdd-orchestrator`, `axiom-install-profile`,
`axiom-capability-router`, `axiom-context-persistence`,
`axiom-token-optimization`, `axiom-code-intelligence`,
`axiom-telemetry`) that are expected to exist in the real Axiom repo's
own catalog — but that catalog does not exist yet, so the smoke test
short-circuits on `resolution.status !== 'resolved'` in this repo's
current state. This increment fills the "freshly bootstrapped SDD repo
has no skills at all" gap for the workspace-setup engine, without
attempting to build the full official 7-skill catalog for the product
itself.

## Scope

- New `apps/cli/src/commands/workspace-skills.ts`:
  - `scaffoldSddSkills(args)` — best-effort, never throws. Writes a
    small bundled seed catalog (`axiom.config/skills-catalog.yaml`)
    plus each seeded skill's source content (bundled as TS string
    constants, not asset files — `tsc -b` does not copy non-TS assets
    to `dist`), calls `loadSkillRegistry` + `applySkillSet` to
    materialize `.opencode/agents/<id>/SKILL.md` and
    `.axiom-state/<projectId>/skills-pending.json`, and writes one
    `axiom.config/skills-index/<role>.yaml` per functional role
    (validated with `validateSkillsRoleIndex` before/after write,
    warnings surfaced, never blocking).
  - Bundled seed catalog: 5 ids — `axiom-sdd-orchestrator`,
    `axiom-context-persistence`, `axiom-capability-router`, plus the
    two `DEFAULT_DESIRED_SKILLS` ids (`serena-this-project`,
    `context7-this-project`) so a plain `applyDefaultSkillSet` call
    would also resolve against this catalog. `bundleHash` computed via
    the real exported `computeSkillBundleHash`, so the catalog is
    internally consistent (byte-exact hash of the bundled source
    content).
- Wire into `runWorkspaceSetup` (`workspace-setup.ts`): after the
  existing best-effort steps, if the control repo's `created === true`,
  call `scaffoldSddSkills` with the control repo path, `projectId`, and
  the functional role ids of every `kind: 'role'` repo in the spec.
  Append `filesCreated`/`warnings`. If the control repo was not freshly
  created, skip silently (no clobbering, no noise).

## Non-goals

- No changes to `axiom init`/`runInit` (single-repo path) or to `axiom
  skills` CLI behavior.
- No changes to the SDD workflow `.claude/skills/axiom-*` command
  skills (bootstrap-harness concern, separate from the product skills
  catalog/materialization covered here).
- No attempt to reproduce Axiom's full/official skill library — a
  small representative canonical seed is a deliberate, bounded
  decision, not a placeholder for future work tracked here.
- No integration into `Axiom.Spec/specs/00-08` in this pass (deferred
  to the orchestrator's cross-increment integration pass, matching the
  precedent of the sibling `INC-20260705-workspace-*` increments).

## Acceptance criteria

- [x] `scaffoldSddSkills` on a tmp control repo produces: a
      `skills-catalog.yaml` that `loadSkillsCatalog` parses as `kind:
      'ok'`; a materialized `SKILL.md` under
      `.opencode/agents/<id>/SKILL.md` for each of the 5 seeded/desired
      skills; `.axiom-state/<projectId>/skills-pending.json`; and one
      `axiom.config/skills-index/<role>.yaml` per passed role that
      `loadSkillsRoleIndex`/`validateSkillsRoleIndex` accept as `kind:
      'ok'`.
- [x] Re-running `scaffoldSddSkills` on the same repo is idempotent (no
      throw, no unnecessary rewrite of byte-identical output).
- [x] `runWorkspaceSetup` on a spec whose control repo does not exist
      yet (fresh tmp dir, `create: true`) ends up with the skills
      catalog + materialized skills + `skills-index/<role>.yaml` for
      every role repo, in the control repo only (never in spec/role
      repos).
- [x] `runWorkspaceSetup` on a spec whose control repo path already
      exists (`create: false`) skips skills scaffolding entirely — no
      `axiom.config/skills-catalog.yaml` is written by this step.
- [x] The best-effort skills step never fails the overall
      `runWorkspaceSetup` call, even if `@axiom/skills` throws
      internally.
- [x] Existing tests keep passing.
- [x] `npm run build` (tsc -b) clean from `Axiom/`.
- [x] Focused vitest run (`apps/cli`, `packages/skills`,
      `packages/user-workspace`, `packages/topology`) green (new +
      pre-existing).

## Open questions

None blocking — the increment prompt supplied fully closed design
decisions (module shape, seed catalog composition, wiring gate,
bundling strategy). Narrow implementation choices are recorded under
Assumptions.

## Assumptions

- The seed catalog's 5 ids are a deliberately small representative
  set, not the eventual "official 7" referenced by
  `packages/doctor/tests/skills.test.ts`'s smoke test — that smoke test
  targets the Axiom repo's OWN `axiom.config/skills-catalog.yaml` (a
  product-level artifact, out of scope here), not a scaffolded
  workspace repo's catalog.
- Seed skill source bodies are minimal but valid Markdown following the
  `product-skill-template.md` narrative shape closely enough to be
  genuine skill content (goal + when-to-use + brief guidance), not
  placeholder lorem ipsum — while staying short since they are bundled
  as TS string constants compiled into the CLI.
- Per-role `skills-index/<role>.yaml` mandatory/available split: the
  role-scoped orchestrator skill (`axiom-sdd-orchestrator`) and
  `axiom-context-persistence` are marked `mandatory` for every role
  (baseline SDD hygiene); `axiom-capability-router` plus the two
  `DEFAULT_DESIRED_SKILLS` ids are marked `available` with simple tags
  derived from the role id. This is a bounded, generic default — richer
  per-role curation (like the real `backend-developer.md` catalog
  fixture) is future work, not required by this increment's acceptance
  criteria.
- `scaffoldSddSkills` is called with `functionalRoleIds` derived as
  `role.functionalRoleId ?? role.roleKey` for every `kind: 'role'` repo
  in `spec.repos`, mirroring the same fallback already used elsewhere
  in `workspace-setup.ts` (`buildTopologyManifest`'s
  `roleId: r.functionalRoleId ?? r.roleKey`).
- Gate is strictly on the control repo's own `created` flag (computed
  by `writeOneRepo` earlier in the same `runWorkspaceSetup` call), not
  on whether the catalog file already exists — matches the brief's
  explicit "if control repo already existed, SKIP" instruction, and
  avoids a second class of no-clobber logic duplicating what
  `writeOneRepo` already decided.

## Implementation notes

- New file `apps/cli/src/commands/workspace-skills.ts`:
  - `SEED_SKILL_SOURCES: Record<string, string>` — bundled Markdown
    body per seeded id (3 canonical Axiom ids +
    `DEFAULT_DESIRED_SKILLS` re-exported from `@axiom/skills`).
  - `buildSeedCatalogYaml(rootPath)` — writes each seed source under
    `axiom.spec/target-axiom-skills/<id>.md` inside the control repo
    (mirrors the path convention used by the test fixtures and
    `product-skill-template.md`'s narrative), computes
    `computeSkillBundleHash` per source, and renders
    `axiom.config/skills-catalog.yaml` (schemaVersion 1).
  - `buildRoleIndexYaml(role, seedIds)` — renders a
    `SkillsRoleIndex`-shaped YAML document for one role.
  - `scaffoldSddSkills({ controlRepoPath, projectId, functionalRoleIds,
    homeDirOverride })` — orchestrates: write seed sources + catalog →
    `loadSkillRegistry` + `applySkillSet(registry, controlRepoPath,
    projectId, SEED_DESIRED_IDS)` → for each role, build + write
    `skills-index/<role>.yaml`, validate with
    `validateSkillsRoleIndex`, warn (don't throw) on any
    inconsistency. The whole function body is wrapped so any thrown
    error becomes a single warning entry and an empty
    `filesCreated`/partial result, never propagating.
- `workspace-setup.ts`: after the existing adapters/workspace.json
  best-effort blocks, a new best-effort block checks
  `repoResults.find(r => r.topologyId === control.topologyId)?.created`
  (the `created` flag already computed for the control repo earlier in
  the same call) and, only if `true`, calls `scaffoldSddSkills` with
  `functionalRoleIds` derived from `roleRepos`. Appends
  `filesCreated`/`warnings` from the result.
- No enum duplication: role ids come straight from
  `WorkspaceRepoSpec.functionalRoleId`/`roleKey`, no new closed union
  introduced.
- `apps/cli/tsconfig.json` already has a path mapping + project
  reference for `@axiom/skills` (used today by `commands/skills.ts`),
  so no build wiring change was needed for the new import.

## Validation

From `C:\repos\Axiom Workspace\Axiom`:

- `npm run build` (tsc -b) — clean, no errors.
- `npx vitest run apps/cli packages/skills packages/user-workspace packages/topology`:

  ```
  Test Files  1 failed | 65 passed (66)
       Tests  1 failed | 613 passed (614)
  ```

  The 1 failing test is pre-existing and unrelated:
  `packages/skills/tests/catalog.test.ts` > "carga el catálogo real del
  repo (axiom.config/skills-catalog.yaml)" expects the Axiom product
  repo's OWN root `axiom.config/skills-catalog.yaml` (7 official skills)
  to exist; it does not (a pre-existing gap from
  `INC-20260703-config-folder-renames`, confirmed via `git diff` showing
  this test's path was already rewritten from
  `axiom.spec/config/skills-catalog.yaml` to `axiom.config/skills-catalog.yaml`
  in a prior, already-committed-to-working-tree change — nothing in this
  increment touches that file or the Axiom repo's own root
  `axiom.config/`). All new tests are green:
  `apps/cli/tests/workspace-skills.test.ts` (8 tests covering
  catalog/materialization/role-index/idempotency/best-effort resilience)
  and 2 new scenarios appended to `apps/cli/tests/workspace-setup.test.ts`
  (Scenario f: fresh control repo gets skills scaffolding; pre-existing
  control repo path skips it) — 17/17 passing in isolation. Every
  pre-existing test in the focused scope still passes.

## Result

Implemented `scaffoldSddSkills` (`apps/cli/src/commands/
workspace-skills.ts`) using the real `@axiom/skills` public API
end-to-end (`loadSkillRegistry`, `applySkillSet`,
`loadSkillsRoleIndex`/`validateSkillsRoleIndex`,
`computeSkillBundleHash`) with a small bundled seed catalog (5 ids) and
matching seed source content, all as TS constants (no non-TS asset
files, compatible with `tsc -b`'s output). Wired as a best-effort final
step of `runWorkspaceSetup`, gated strictly on the control repo's own
freshly-created flag, producing one `skills-index/<role>.yaml` per
functional role repo declared in the workspace spec. Spec/role repos
are never touched by this step. Build and focused test suite are both
green.

## General spec integration

Integrated into the canonical spec in the round-2 cross-increment pass
(covering this increment and the three sibling round-2
`INC-20260705-workspace-*` increments). Files updated:

- **06_Integraciones_y_Capacidades.md** — added the "Baseline de Axiom
  SKILLS scaffoldeada en el repo SDD recién creado" subsection: bundled
  seed catalog (5 ids, `computeSkillBundleHash`), `applySkillSet`
  materialization of `.opencode/agents/<id>/SKILL.md` +
  `skills-pending.json`, one `skills-index/<role>.yaml` per functional
  role, control-repo-only + `created`-gated + best-effort.
- **03_Modelo_Operativo_y_Datos.md** — documented the scaffolded shapes
  (`axiom.config/skills-catalog.yaml`, `.opencode/agents/<id>/SKILL.md`,
  `skills-pending.json`, `axiom.config/skills-index/<role>.yaml`) and the
  control-repo `created`-gating semantics in the new "Artefactos
  adicionales del setup de workspace" subsection.
- **01_Requisitos_Funcionales.md** — extended RF-AXM-024 with the
  "freshly-created SDD repo gets an Axiom skills baseline" sub-point.
- **08_Glosario.md** — added "skills-catalog / skills-index".
- **07_Gobierno_y_Seguridad.md** — noted the `created`-gating +
  best-effort no-clobber safety of the skills step in the ownership
  section.
