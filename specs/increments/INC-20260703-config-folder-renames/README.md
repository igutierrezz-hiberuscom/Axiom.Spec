# Increment: config/state folder renames (`.sdd` → `.axiom-state`, `axiom.spec/config` → `axiom.config`)

Status: closed
Date: 2026-07-03

## Goal

Rename two generated config/state folders that `axiom init` scaffolds,
because their current names are misleading:

- `.sdd/` (runtime state directory, e.g. `init.json`,
  `install-profile.json`, `.sdd/local/` overlay) has an unclear name —
  it does not obviously read as "Axiom's runtime state".
- `axiom.spec/config/` (generated CONFIG files: `topology.yaml`,
  `profiles.yaml`, `providers.yaml`, etc.) is nested one level too
  deep and is easily confused with SPEC CONTENT — it is config, not
  specs.

There are no real Axiom projects yet (all test/dogfood data), so no
migration or backward-compatibility path is needed. The old names are
broken freely, project-wide.

## Context

This is Etapa 3 of the init/config redesign, explicitly deferred from
`INC-20260703-tui-init-wizard` (see that increment's Non-goals: "No
folder/path renames (`.sdd` → `.axiom-state`, `axiom.spec/config` →
`axiom.config`, etc.) — explicitly out of scope, deferred to a
separate increment").

Two independent renames were required:

1. **`.sdd` → `.axiom-state`** — the canonical constant
   `LOCAL_OVERLAY_DIRNAME` lives in
   `packages/filesystem-truth/src/discovery.ts` (re-exported via
   `packages/core/src/paths.ts`), but dozens of call sites across
   `apps/cli`, `packages/persistence`, `packages/orchestrator`,
   `packages/workflow`, `packages/installer`, `packages/doctor`,
   `packages/isolation`, `packages/memory`, `packages/model-routing`,
   `packages/versioning`, `packages/telemetry`, `packages/tui`, and
   their tests hardcoded the literal `'.sdd'` instead of importing the
   constant.
2. **`axiom.spec/config/` → `axiom.config/`** — collapses the
   redundant `config` subfolder. This is a folder MOVE (config moves
   out from under the spec scope to become a sibling of `axiom.spec/`
   at the project root), not a simple string substitution: several
   doctor checks and CLI commands derived the old config path by
   joining `'config'` onto the resolved **spec scope's absolute
   path** (`specScope.absolutePath`), which no longer makes sense once
   config is a project-root-level concern independent of whether a
   spec scope/repo exists at all.

## Scope

### Rename 1: `.sdd` → `.axiom-state`

- Changed `LOCAL_OVERLAY_DIRNAME` from `'.sdd'` to `'.axiom-state'` in
  `packages/filesystem-truth/src/discovery.ts` (single source of
  truth, re-exported via `packages/filesystem-truth/src/index.ts` and
  `packages/core/src/paths.ts`).
- Swept every hardcoded `'.sdd'` / `` `.sdd` `` / `".sdd"` /
  `.sdd/`-path-segment literal across `apps/cli`, and packages
  `persistence`, `orchestrator`, `workflow`, `installer`, `doctor`,
  `isolation`, `memory`, `model-routing`, `versioning`, `telemetry`,
  `tui`, `components`, `skills`, and their tests (155 files touched
  for this rename alone).
- Updated `apps/cli/src/commands/init.ts`'s generated `.gitignore`
  template (`.sdd/local/` → `.axiom-state/local/`) and doc comments.
- Updated `docs/cli/*.md` real-output examples that embedded literal
  Windows paths (`...\.sdd\...` → `...\.axiom-state\...`).
- **Explicitly NOT touched**: the unrelated `sdd` **repo-role**
  concept (`RepoRole` enum value `'sdd'`, `manifest.sddRepo`,
  `project.repos.sdd`, `mandatory.sddSkills`, `paths.sddRoot`,
  `repositories.sdd`). This is a different, pre-existing concept (the
  "spec/sdd repository" role in a multi-repo topology) that happens to
  share the `sdd` substring — verified case by case during the sweep
  and left untouched.

### Rename 2: `axiom.spec/config/` → `axiom.config/`

- Introduced `AXIOM_CONFIG_DIRNAME = 'axiom.config'` in
  `packages/filesystem-truth/src/discovery.ts`, re-exported via
  `packages/filesystem-truth/src/index.ts` and
  `packages/core/src/paths.ts`.
- Exact path remapping applied everywhere (never a naive
  `s/axiom.spec/axiom.config/`):
  - `axiom.spec/config/topology.yaml` → `axiom.config/topology.yaml`
  - `axiom.spec/config/profiles.yaml` → `axiom.config/profiles.yaml`
  - `axiom.spec/config/providers.yaml` → `axiom.config/providers.yaml`
  - `axiom.spec/config/capabilities.yaml` → `axiom.config/capabilities.yaml`
  - `axiom.spec/config/integrations.yaml` → `axiom.config/integrations.yaml`
  - `axiom.spec/config/toolchain.yaml` → `axiom.config/toolchain.yaml`
  - `axiom.spec/config/model-routing-policy.yaml` → `axiom.config/model-routing-policy.yaml`
  - `axiom.spec/config/mcp-manifest.yaml` → `axiom.config/mcp-manifest.yaml`
  - `axiom.spec/config/skills-catalog.yaml` → `axiom.config/skills-catalog.yaml`
  - `axiom.spec/config/agents-catalog.yaml` → `axiom.config/agents-catalog.yaml`
  - `axiom.spec/config/scaffolding-contract.yaml` → `axiom.config/scaffolding-contract.yaml`
  - `axiom.spec/config/skills-index/<role>.yaml` → `axiom.config/skills-index/<role>.yaml`
  - `axiom.spec/config/telemetry-sinks.yaml`, `policy-as-code.yaml`,
    `command-protocol.yaml`, `onboarding.yaml`, `local-overlay-policy.yaml`
    (documentation-only mentions in `docs/configuration/files/*.md`) →
    equivalent `axiom.config/...`
- Updated `apps/cli/src/commands/init.ts`'s scaffold to write
  `axiom.config/topology.yaml` (was `axiom.spec/config/topology.yaml`).
- Fixed several call sites that built the old path DYNAMICALLY by
  joining `'config'` onto a resolved spec-scope path (not a literal
  string, so the initial literal-string sweep missed them). All were
  changed to resolve `axiom.config/` from the **project root**
  instead, and the now-obsolete "spec scope must exist" guard was
  removed where it existed only to gate config-file reads:
  - `packages/doctor/src/checks.ts` — 8 helper functions
    (`runPolicyChecks`, `runCapabilityModelChecks`,
    `runInstallProfileChecks`, `readTopologyManifest`,
    `readProfilesForTopologyCheck`, `readOptionalStructuralCapabilities`,
    `readToolchainManifest`, `readToolchainCatalogue`,
    `readMcpManifest`, `readSkillsCatalog`, `readAgentsCatalog`,
    `resolveSkillsRoleIndexDir`, `collectGatewayDigests`,
    `runGatewayStateChecks`) — all now resolve `axiom.config/` from
    `resolution.rootPath` and no longer gate on `specScope.exists`.
  - `packages/capability-model/src/loader.ts` — `loadCapabilityModel`'s
    parameter renamed `specScopePath` → `projectRootPath` (it now
    resolves `<projectRootPath>/axiom.config/{capabilities,providers}.yaml`
    instead of `<specScopePath>/config/...`); callers updated in
    `apps/cli/src/commands/gateway.ts`, `configure.ts`, `start.ts`.
  - `apps/cli/src/commands/gateway.ts` — removed
    `resolveSpecScopePath`; `specScopePath`-named parameters/locals
    renamed to `projectRootPath` throughout
    (`readCapabilityModelState`, `readProviderIdsFallback`,
    `collectGatewayInputDigests`, `buildGatewayStateRecord`). The
    persisted `GatewayStateRecord.validation.specScopePath` JSON field
    name was kept unchanged (avoids an unrelated schema-name churn on
    an already-persisted artifact) but its value is now the project
    root path.
  - `apps/cli/src/commands/configure.ts`, `topology.ts`,
    `qa-archive-gate.ts`; `packages/model-routing/src/checks.ts`,
    `opencode-projection.ts`; `packages/tui/src/driver.ts` — each had
    one multi-line `path.join(root, 'axiom.spec', 'config', '<file>')`
    call fixed to `path.join(root, 'axiom.config', '<file>')`.
- Test fixtures/scaffolds updated to write to `axiom.config/` instead
  of `axiom.spec/config/`: `packages/capability-model/tests/loader.test.ts`,
  `packages/mcp-tools/tests/capability-routing-roundtrip.test.ts`,
  `packages/skills/tests/apply.test.ts` (`scaffoldWithCatalog`),
  `apps/cli/tests/roles.test.ts` (`readProfilesYaml`), and the many
  doctor/CLI test files whose scaffolds already used `path.join(root,
  'axiom.config', ...)`-style construction correctly.
- Moved the physical fixture file
  `packages/skills/tests/fixtures/axiom.spec/config/skills-index/backend-developer.yaml`
  to `packages/skills/tests/fixtures/axiom.config/skills-index/backend-developer.yaml`
  (confirmed unused directly by any test — informational fixture only)
  and updated the prose in the sibling
  `axiom.spec/target-axiom-skills/catalogs/backend-developer.md` that
  referenced the old path in its text.
- **Explicitly NOT touched** (spec-content paths, a different
  concern): `axiom.spec/plans/`, `axiom.spec/specs/`,
  `axiom.spec/scripts/`, `axiom.spec/target-axiom-skills/`,
  `axiom.spec/target-axiom-agents/`, `axiom.spec/templates/`,
  `axiom.spec/technical-context/`, `axiom.spec/product.manifest.yaml`.

## Non-goals

- No migration/back-compat shims for the old `.sdd`/`axiom.spec/config`
  names — there are no real Axiom projects yet.
- No implementation of the config-duplication consolidation described
  below (analysis only in this increment).
- No changes to the TUI init wizard's behavior beyond mechanical
  path-literal renames (the wizard flow itself, added in
  `INC-20260703-tui-init-wizard`, is unchanged).
- No renames of spec-content paths (`axiom.spec/plans|specs|scripts|
  target-axiom-skills|target-axiom-agents|templates|technical-context`).

## Acceptance criteria

- [x] `LOCAL_OVERLAY_DIRNAME` constant (`@axiom/filesystem-truth`,
      re-exported via `@axiom/core`) equals `'.axiom-state'`.
- [x] `AXIOM_CONFIG_DIRNAME` constant (`@axiom/filesystem-truth`,
      re-exported via `@axiom/core`) equals `'axiom.config'`, newly
      introduced.
- [x] No remaining literal `.sdd` path-segment references anywhere in
      `Axiom/` source, tests, or docs (verified by repo-wide grep;
      only the unrelated `sdd`-repo-role identifiers remain, by
      design).
- [x] No remaining `axiom.spec/config` path references anywhere in
      `Axiom/` source, tests, or docs (verified by repo-wide grep,
      including `path.join(...)` multi-arg and dynamic-variable
      constructions, not just literal-string matches).
- [x] `axiom init`'s generated `.gitignore` references
      `.axiom-state/local/`; its scaffold writes
      `axiom.config/topology.yaml`.
- [x] `npm run build` (tsc -b) is clean.
- [x] `npx vitest run` (full suite) has the exact same failing-test
      set as the pre-rename baseline (11 pre-existing failures,
      unrelated to this increment — see Validation), and no new
      failures.

## Open questions

None blocking.

## Assumptions

- The pre-existing baseline (captured via `git stash` before this
  increment's changes) already had **11** failing tests, not 1 — the
  previous increment's report only called out the single
  `start.test.ts` Scenario 4 failure it happened to notice from a
  partial run. The other 10 are pre-existing, unrelated gaps (this
  dogfooding `Axiom/` repo does not have a real
  `axiom.config`/`axiom.spec/config` scaffold at its own root, so any
  test that does real end-to-end integration against `process.cwd()`
  or a `path.resolve(__dirname, '..', ..., '..')`-derived repo root
  fails looking for files that were never seeded there). This
  increment's closure bar is "same failing set as baseline," not
  "only the one previously-documented failure."
- `GatewayStateRecord.validation.specScopePath`'s JSON field NAME was
  deliberately kept as-is (only its semantic value changed, from the
  spec-scope path to the project-root path) to avoid an unrelated
  persisted-schema rename inside this folder-rename increment.

## Implementation notes

See Scope above for the full file-by-file breakdown. Key structural
insight: the initial literal-string sweep (`'axiom.spec/config'` →
`'axiom.config'`, `'.sdd'` → `'.axiom-state'`) caught the majority of
occurrences, but several call sites built the old paths dynamically —
joining `'config'` onto a resolved **spec scope** path/variable rather
than using the literal string `'axiom.spec/config'`. Because
`axiom.config/` is now a project-root-level sibling of `axiom.spec/`
rather than nested inside it, these call sites required a structural
fix (resolve from `resolution.rootPath` / `projectRoot` instead of
`specScope.absolutePath`), not just a text substitution. This also
made several `specScope.exists` guards in `packages/doctor/src/checks.ts`
obsolete (they existed only to gate config-file reads that no longer
depend on the spec scope existing) — those guards were removed.

## Validation

`npm run build` (tsc -b) from `Axiom/` — clean, no errors.

`npx vitest run` (full suite) — verbatim tail:

```
 Test Files  11 failed | 152 passed (163)
      Tests  11 failed | 1661 passed (1672)
```

The 11 failures are byte-for-byte the same test IDs as a baseline run
captured via `git stash` immediately before this increment's changes
(only the failure *labels* differ for two tests whose descriptions
literally embed the renamed path, e.g. "carga el catálogo real del
repo (axiom.config/agents-catalog.yaml)" vs. the old
"...(axiom.spec/config/agents-catalog.yaml)" — same underlying gap,
same test, same root cause, renamed text). Full list:

- `apps/cli/tests/start.test.ts` — Scenario 4 (explicitly called out
  in the increment brief as the one pre-existing/allowed failure)
- `packages/agents/tests/catalog.test.ts` — real-repo catalog
  integration test
- `packages/doctor/tests/checks.test.ts` — "repositorio Axiom real"
  integration test
- `packages/model-routing/tests/assignments.test.ts` — real-policy
  integration test
- `packages/model-routing/tests/loader.test.ts` — real-policy
  integration test
- `packages/model-routing/tests/opencode-projection.test.ts` — 4a,
  real-policy integration test
- `packages/model-routing/tests/resolver.test.ts` — real-policy
  integration test
- `packages/project-resolution/tests/resolver.test.ts` — v1 scopes
  resolution test (unrelated to this rename)
- `packages/skills/tests/catalog.test.ts` — real-repo catalog
  integration test
- `packages/telemetry/tests/audit-trail-sink.test.ts` — retention
  sweep timing test (unrelated to this rename)
- `packages/toolchain/tests/repair-add-gitignore.test.ts` — unrelated
  to this rename

No validation command discovery was needed beyond `README.md` /
`package.json` scripts (`npm run build`, `npx vitest run`), consistent
with `Axiom.SDD/AGENTS.md`'s validation discovery order.

## Result

Both renames implemented project-wide and verified clean:

- `.sdd` → `.axiom-state` (canonical constant + ~155 files swept).
- `axiom.spec/config/` → `axiom.config/` (new canonical constant,
  exact path remapping, including structural fixes to ~20 call sites
  that derived the old path dynamically from the spec scope rather
  than a literal string).

Build is clean; full test suite has the identical failing-test set as
the pre-rename baseline (11 pre-existing, unrelated failures) — no
regressions introduced, no failures fixed beyond the increment's
scope (per instructions, only failures caused by this rename were to
be fixed).

## General spec integration

Updated `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md`: generated
project config now lives in `axiom.config/` (was `axiom.spec/config/`)
and runtime state in `.axiom-state/` (was `.sdd/`).

Updated `Axiom.Spec/specs/08_Glosario.md`: renamed/redefined the two
terms accordingly.

## Config duplication analysis (documentation only — no dedup implemented)

While sweeping the rename, three instances of the same piece of
project data being persisted in more than one place were identified.
This is analysis only; **no data flow or field was changed in this
increment**. Recommended as a follow-up increment:

1. **Repo map duplicated** — the map of repos/scopes for a project is
   declared both in `axiom.yaml#paths` (or legacy `#scopes`) and in
   `axiom.config/topology.yaml` (`roleCodeRepositories`,
   `assignments`). Both describe overlapping repo/role information
   with different shapes and different write paths (`axiom init`
   writes `axiom.yaml#paths`; `axiom roles assign`/topology tooling
   writes `topology.yaml`).
2. **`projectName`/id duplicated** — the project's name/id is stored
   in `axiom.yaml` (`project.name` / `projectId`) AND independently in
   `.axiom-state/<projectName>/init.json` (`projectName` field). The
   `.axiom-state/<projectName>/` directory segment itself also encodes
   the same name a third time (as a path component).
3. **Overlay/target duplicated** — the resolved operational
   overlay/adapter-target pair is stored in
   `.axiom-state/<projectName>/init.json` (`profileTriple.
   operationalOverlay`/`adapterTarget`) AND again in
   `.axiom-state/<projectName>/install-profile.json` (the resolved
   `ResolvedInstallProfile`, which is derived FROM `init.json` but
   re-persists its own copy of the overlay/target-dependent fields).

Recommendation: a future increment should evaluate whether
`topology.yaml` can become the single source of truth for the repo
map (with `axiom.yaml#paths` reduced to "this repo's own scopes
only"), and whether `install-profile.json` should reference
`init.json` by pointer/hash instead of re-persisting derived fields —
without breaking `axiom doctor`'s existing drift-detection checks
(several of which currently rely on comparing the two files
independently, e.g. `GW-001`'s config-hash drift check). Out of scope
here; documented per `AGENTS.md`'s "document as a future
consideration, do not implement now" rule.
