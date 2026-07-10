# Increment: dogfooding workflow configs (P0-2)

Status: closed
Date: 2026-07-10

## Goal

Make `axiom-increment`, `axiom-bug`, `axiom-plan approve`,
`axiom-role`, and `axiom-qa-e2e` work against the real `Axiom` repo
(dogfooding its own SDD state machine), and make `axiom topology
show/validate` reflect the real 3-repo workspace, without breaking
any existing test or `@axiom/doctor` check. Also fix the pre-existing
TC-011 drift (stale `bundleHash` on `axiom-reviewer`).

## Context

`Axiom/axiom.config/workflows.yaml` and `Axiom/axiom.config/
topology.yaml` did not exist anywhere in the workspace (no file, no
fixture). Live, `node apps/cli/dist/index.js axiom-increment create
--id 9999 --slug x` failed with "No se encontró workflows.yaml en
…\axiom.config\workflows.yaml" (exit 1) — same for `axiom-bug`,
`axiom-plan approve`, `axiom-role`, `axiom-qa-e2e`. `topology show`
silently defaulted to `mode: single-repo`, hiding the real 3-repo
topology (`Axiom`, `Axiom.SDD`, `Axiom.Spec`).

Root cause: `workflows.yaml`/`topology.yaml` are canonical PRODUCT
data (identical shape across projects, like `profiles.yaml`), but
unlike `profiles.yaml` (which has a bundled `DEFAULT_PROFILES`
fallback in `@axiom/install-profiles`, added by
`BUG-20260703-configure-needs-bundled-profiles`), `workflows.yaml`
had NO bundled fallback — every `axiom-*` command's `loadXWorkflow
Config` hard-failed with `kind: 'workflows-yaml-missing'` when the
file was absent, with no project-agnostic default to fall back to.

Separately, `packages/doctor/tests/agents.test.ts` TC-011 ("7/7
entries consistentes") was failing against the real repo: the
`axiom-reviewer` catalog entry's `bundleHash` in
`axiom.config/agents-catalog.yaml` no longer matched the sha256 of
its source file (`axiom.spec/target-axiom-agents/axiom-reviewer.md`)
— a stale-hash drift unrelated to any of this increment's changes,
flagged by the brief as a bounded pre-existing fix to include.

## Scope

- New `DEFAULT_WORKFLOWS` bundled constant in `@axiom/workflow`
  (`packages/workflow/src/default-workflows.ts`), mirroring the
  `DEFAULT_PROFILES` pattern. Declares the 5 canonical workflows
  (`increment`, `bug`, `plan`, `role`, `qa-e2e`) derived
  authoritatively from the 5 CLI command files that consume them
  (`apps/cli/src/commands/axiom-{increment,bug,plan,role,qa-e2e}.ts`).
- Each of those 5 command files' `loadXWorkflowConfig` now falls
  back to the matching `DEFAULT_WORKFLOWS` entry ONLY when
  `axiom.config/workflows.yaml` is absent (`fs.existsSync` false). A
  PRESENT file — even if malformed — still goes through the
  unchanged read/parse/validate path and keeps its existing typed
  errors (`invalid-yaml`, `invalid-entry`, etc.). File always wins
  over the fallback.
- Shipped `Axiom/axiom.config/workflows.yaml` (the same 5 workflows,
  `schemaVersion: 1`) and `Axiom/axiom.config/topology.yaml`
  (`mode: multi-repo`, referencing the sibling `../Axiom.SDD` and
  `../Axiom.Spec` repos, with `Axiom` itself declared as a
  `roleCodeRepository` assigned to the `builder` profile from
  `profiles.yaml`).
- Two new `@axiom/doctor` checks (`packages/doctor/src/checks.ts`,
  category `workflow-config`): `TC-014` (`workflows.yaml` valid, or
  absent → pass; present+malformed → fail) and `TC-015`
  (`topology.yaml` valid, or absent → pass; present+malformed →
  fail). Registered in `packages/doctor/src/index.ts`'s
  `runDoctorChecks`. Unit tests in `packages/doctor/tests/
  workflow-config.test.ts`.
- Fixed the stale `bundleHash` for `axiom-reviewer` in
  `axiom.config/agents-catalog.yaml` (root cause: source file
  content had drifted from the catalogued hash; corrected the
  declared value to the actual sha256 of the current source file —
  no test logic was weakened).
- New regression tests: `packages/workflow/tests/default-workflows
  .test.ts` (validity + exact sync with the real committed
  `workflows.yaml`), plus a "DEFAULT_WORKFLOWS fallback" scenario
  added to each of `apps/cli/tests/axiom-{increment,bug,plan,role,
  qa-e2e}.test.ts`.

## Non-goals

- No migration/reconciliation of `topology.yaml`'s `assignments[]
  .roleId` from a functional profile (`builder`) to an
  implementation-role id — that mismatch is explicitly deferred to a
  later increment per the brief.
- No change to `@axiom/topology`'s validator or loader semantics —
  only new config content and a new doctor check that surfaces
  `loadTopology` errors that were previously silently absorbed into
  a `skip` by `runTopologyChecks` (TC-001/TC-002).
- No consolidation of the 5 duplicated inline
  `loadXWorkflowConfig`/`parseXEntry` implementations across the 5
  command files into the already-existing centralized
  `loadWorkflowsConfig` (`packages/workflow/src/workflows-loader
  .ts`, Spec 0028/E3) — that refactor is out of scope; this
  increment adds the smallest correct fallback to each existing
  function.
- No script/CLI command to regenerate `bundleHash` values
  automatically (unlike `packages/skills/src/refresh.ts`'s
  skills-lock refresh) — `@axiom/agents` already exposes a pure
  `computeAgentBundleHash` helper; wiring a CLI refresh command is a
  future consideration, not implemented here.

## Acceptance criteria

- [x] `npm run build` (`tsc -b`) is clean.
- [x] Live: `axiom-increment create --id 9999 --slug dogfood-probe
      --json` succeeds (exit 0) against the real `Axiom` repo with no
      `workflows.yaml` override.
- [x] Live: `axiom-bug create --id 9998 --slug dogfood-bug-probe
      --json` succeeds (exit 0).
- [x] Live: `axiom-plan approve --id PLAN-DOGFOOD-PROBE --json`
      succeeds (exit 0) — bonus verification beyond the two
      explicitly required probes.
- [x] Live: `topology validate` passes (exit 0, "✓ Topología válida
      (mode: multi-repo)"); `topology show` prints the real 3-repo
      topology with no error.
- [x] All live probe artifacts (`.axiom-state/`,
      `axiom.spec/increments/INC-…`, `axiom.spec/bugs/BUG-…`) were
      deleted after verification; the tree was left exactly as found.
- [x] `npx vitest run` on `packages/workflow`, `packages/doctor`,
      `apps/cli`, and the full workspace: zero failures (TC-011 now
      passes; no new failures).
- [x] TC-011 fixed at the root (corrected `bundleHash`, not a
      weakened assertion).

## Open questions

None blocking.

## Assumptions

- The 5 workflows' states/transitions/commands as hard-coded in the
  5 `apps/cli/src/commands/axiom-*.ts` files are the authoritative
  source of truth for both `DEFAULT_WORKFLOWS` and the shipped
  `workflows.yaml` — verified line-by-line against each file's
  `SUBCOMMAND_TO_TRANSITION` map (or, for `plan`, the single
  `TRANSITION_COMMAND` constant) plus each command's doc-comment
  description (e.g. `archive` and `plan-approve` both documented as
  "Requiere confirmation", mapped to `requiresApproval: true`).
- `topology.yaml`'s `roleCodeRepositories`/`assignments` intentionally
  use the `builder` functional-profile id (from `profiles.yaml`), not
  an implementation-role id (`backend`/`frontend`/`qa-e2e`), per the
  brief's explicit instruction that the profile-vs-role mismatch is
  handled by a later increment.
- `Axiom/axiom.spec/{increments,bugs}/` (lowercase, the artifact-store
  root inside the Axiom repo itself — distinct from the sibling
  `Axiom.Spec` repo) is where `axiom-increment`/`axiom-bug create`
  write folder-per-instance `metadata.yml`/`README.md`, unaffected by
  `topology.yaml`'s `specRepo` field (which is informational for
  role/topology tooling, not consumed by the artifact-store path).

## Implementation notes

### Fallback wiring (the primary fix)

Each of the 5 `loadXWorkflowConfig` functions had the exact same
shape: `if (!fs.existsSync(filePath)) return err({ kind:
'workflows-yaml-missing', ... })`. Changed that branch (only) to
look up `DEFAULT_WORKFLOWS.workflows.find(w => w.id === WORKFLOW_ID)`
and, if found, run it through the SAME existing `parseXEntry`
helper used for a real parsed YAML entry (cast via `as unknown as
Record<string, unknown>`, since `DEFAULT_WORKFLOWS`'s entries already
satisfy that shape). Everything below the `existsSync` check —
including the entire malformed-file error path — is untouched.

### Files changed (implementation, `Axiom` repo)

- `packages/workflow/src/default-workflows.ts` (new) — `DEFAULT_WORKFLOWS`.
- `packages/workflow/src/index.ts` — export `DEFAULT_WORKFLOWS`.
- `apps/cli/src/commands/axiom-increment.ts` — fallback wiring.
- `apps/cli/src/commands/axiom-bug.ts` — fallback wiring.
- `apps/cli/src/commands/axiom-plan.ts` — fallback wiring.
- `apps/cli/src/commands/axiom-role.ts` — fallback wiring.
- `apps/cli/src/commands/axiom-qa-e2e.ts` — fallback wiring (not
  explicitly named in the brief's "Required fix" list, but the brief's
  "Problem" section explicitly listed `axiom-qa-e2e` as failing live
  too; wired for consistency, closing the same gap).
- `axiom.config/workflows.yaml` (new) — the 5 real workflows.
- `axiom.config/topology.yaml` (new) — multi-repo topology.
- `axiom.config/agents-catalog.yaml` — corrected `axiom-reviewer`
  `bundleHash` (TC-011 root-cause fix).
- `packages/doctor/src/checks.ts` — `runWorkflowsYamlValidityCheck`
  (TC-014) + `runTopologyYamlValidityCheck` (TC-015), new category
  `workflow-config`.
- `packages/doctor/src/index.ts` — registered both in exports and in
  `runDoctorChecks`.
- Tests (new/extended): `packages/workflow/tests/default-workflows
  .test.ts`; `packages/doctor/tests/workflow-config.test.ts`; a
  "Bonus: DEFAULT_WORKFLOWS fallback" scenario appended to each of
  `apps/cli/tests/axiom-increment.test.ts`, `axiom-bug.test.ts`,
  `axiom-plan.test.ts`, `axiom-role.test.ts`, `axiom-qa-e2e.test.ts`.

## Validation

- `cd Axiom && npm run build` (`tsc -b`): clean, exit 0.
- `npx vitest run packages/workflow`: 8 files, 90 tests, all passed
  (includes the new `default-workflows.test.ts`, 5 tests, one of
  which does an exact deep-equal between `DEFAULT_WORKFLOWS` and the
  real committed `workflows.yaml` parsed via `loadWorkflowsConfig` —
  confirming the two stay in sync).
- `npx vitest run packages/doctor`: 18 files, 177 tests, all passed
  (includes `agents.test.ts` — TC-011's "smoke contra el repo Axiom
  real" test NOW PASSES with "7/7 entries consistentes"; includes
  the new `workflow-config.test.ts`, 10 tests for TC-014/TC-015).
- `npx vitest run apps/cli`: 65 files, 620 tests, all passed
  (includes the 5 new fallback-scenario tests, one per command).
- `npx vitest run` (full workspace): 207 files (207 passed), 2179
  tests (2179 passed). Baseline before this increment (from
  `INC-20260710-schema-reconciliation`): 205 files/2159 tests with 1
  known failure (TC-011). Delta: +2 files, +20 tests, **zero
  failures** (TC-011's failure is now fixed, not just unaffected).
- Live probes (from `Axiom/`, using the built
  `apps/cli/dist/index.js`, run AFTER `npm run build`):
  - `node apps/cli/dist/index.js axiom-increment create --id 9999
    --slug dogfood-probe --json` → exit 0, `state: "specifying"`,
    `vars.metadataId: "INC-20260710-104852-uy45wc"` (previously: exit
    1, "No se encontró workflows.yaml...").
  - `node apps/cli/dist/index.js axiom-bug create --id 9998 --slug
    dogfood-bug-probe --json` → exit 0, `state: "specifying"`,
    `vars.metadataId: "BUG-20260710-104902-5a05vx"`.
  - `node apps/cli/dist/index.js axiom-plan approve --id
    PLAN-DOGFOOD-PROBE --json` → exit 0, `state: "plan-approved"`
    (bonus probe, not explicitly required).
  - `node apps/cli/dist/index.js topology validate` → exit 0, "✓
    Topología válida (mode: multi-repo)." (previously would have
    reported `mode: single-repo`, the pre-file default).
  - `node apps/cli/dist/index.js topology show` → exit 0, prints
    `mode: multi-repo`, `sdd-repo: sdd-repo @ ../Axiom.SDD`,
    `spec-repo: spec-repo @ ../Axiom.Spec`, `role-code-repos: 1` (
    `axiom-product-repo @ .`), `assignments: 1` (`axiom-product-repo
    → builder (primary)`).
  - Cleanup: deleted `.axiom-state/` (contained
    `local/workflow-state.json` with the 3 probe workflow states:
    `increment`, `bug`, `plan`), `axiom.spec/increments/
    INC-20260710-104852-uy45wc/` (`metadata.yml` + `README.md`), and
    `axiom.spec/bugs/BUG-20260710-104902-5a05vx/` (`metadata.yml` +
    `README.md`) — then removed the now-empty
    `axiom.spec/increments/` and `axiom.spec/bugs/` parent
    directories themselves (they did not exist before the probes),
    restoring the tree to its exact pre-probe state.

## Result

Closed. All 5 dogfooding commands (`axiom-increment`, `axiom-bug`,
`axiom-plan approve`, `axiom-role`, `axiom-qa-e2e`) now work in ANY
install — including the `Axiom` repo itself, which previously had no
`workflows.yaml` at all — via the bundled `DEFAULT_WORKFLOWS`
fallback, while a present-but-malformed file still produces the
pre-existing typed error (no silent tolerance regression). The `Axiom`
repo also now ships its own real `workflows.yaml`/`topology.yaml`
(file wins over the fallback), so `topology show/validate` reflect
the actual 3-repo workspace. Two new doctor checks (TC-014, TC-015)
close the "present but invalid config" blind spot for both files. The
pre-existing, unrelated TC-011 drift (stale `axiom-reviewer`
`bundleHash`) was fixed at the root. Full workspace suite: 207
files/2179 tests, zero failures (up from 205/2159 with 1 known
failure at the prior increment's baseline).

## General spec integration

Integrated into `Axiom.Spec/context/architecture/
01-vision-general-y-capas.md`'s companion note (see the new
"Dogfooding: workflows.yaml y topology.yaml" addition below) — the
stable facts worth consolidating are:

1. `axiom.config/workflows.yaml` and `axiom.config/topology.yaml` are
   both OPTIONAL at the file level: `@axiom/workflow` ships a bundled
   `DEFAULT_WORKFLOWS` fallback (mirroring `@axiom/install-profiles`'s
   `DEFAULT_PROFILES`) and `@axiom/topology` already had
   `defaultSingleRepoManifest`/`defaultInstalledMultiRepoManifest`. A
   project never needs to author either file to use the SDD workflow
   commands or `topology show/validate` — but if it DOES ship one,
   that file wins and a malformed one is a hard error (not silently
   tolerated).
2. The `Axiom` repo itself now ships both files, declaring its real
   multi-repo topology (siblings `../Axiom.SDD`, `../Axiom.Spec`).
3. `@axiom/doctor` TC-014/TC-015 close the "present but invalid"
   blind spot for these two files specifically (complementing, not
   replacing, TC-001/TC-002's role-coverage checks).

No other file under `Axiom.Spec` needed changes; the numbered specs
(`00_Resumen_Ejecutivo.md` through `08_Glosario.md`) describe target
product behavior and were already consistent with this fix (they do
not currently claim these two config files are mandatory).
